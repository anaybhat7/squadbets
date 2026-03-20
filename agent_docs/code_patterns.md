# Code Patterns & Best Practices

## Quick Reference

Use this document to ensure code consistency across the SquadBets MVP. Refer to it when building components, functions, and API endpoints.

---

## Component Architecture (Base44)

### Pattern 1: Screen Template

```javascript
// screens/SquadDashboardScreen.js
import React, { useState, useEffect } from 'react';
import { supabase } from '../lib/supabase';

export default function SquadDashboardScreen({ squadId }) {
  const [events, setEvents] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Fetch events for this squad (RLS enforced)
    const fetchEvents = async () => {
      try {
        setLoading(true);
        const { data, error: dbError } = await supabase
          .from('events')
          .select('*')
          .eq('squad_id', squadId); // CRITICAL: Always filter by squad_id
        
        if (dbError) throw dbError;
        setEvents(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchEvents();
  }, [squadId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div className="error">Error: {error}</div>;

  return (
    <div className="squad-dashboard">
      <h1>Squad Dashboard</h1>
      <button onClick={() => navigateToCreateEvent(squadId)}>Create Event</button>
      <div className="events-list">
        {events.map(event => (
          <EventCard key={event.id} event={event} />
        ))}
      </div>
    </div>
  );
}
```

### Pattern 2: Form Component

```javascript
// components/CreateEventForm.js
import React, { useState } from 'react';
import { supabase } from '../lib/supabase';

export default function CreateEventForm({ squadId, onSuccess }) {
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!title.trim()) {
      alert('Title is required');
      return;
    }

    try {
      setLoading(true);
      
      // Insert event
      const { data, error } = await supabase
        .from('events')
        .insert([
          {
            squad_id: squadId,
            title: title.trim(),
            description: description.trim(),
            status: 'OPEN',
            current_prob: 0.5,
            creator_id: (await supabase.auth.getUser()).data.user.id
          }
        ]);

      if (error) throw error;

      // Success feedback
      alert('Event created!');
      onSuccess(data[0]); // Pass new event to parent
      
      // Reset form
      setTitle('');
      setDescription('');
    } catch (err) {
      alert(`Error: ${err.message}`);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Event title (e.g., 'Will it rain tomorrow?')"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        disabled={loading}
        required
      />
      <textarea
        placeholder="Description (optional)"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
        disabled={loading}
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Event'}
      </button>
    </form>
  );
}
```

### Pattern 3: Slider Component (Prediction)

```javascript
// components/PredictionSlider.js
import React, { useState, useEffect } from 'react';
import { supabase } from '../lib/supabase';

export default function PredictionSlider({ eventId, squadId }) {
  const [prediction, setPrediction] = useState(50);
  const [submitted, setSubmitted] = useState(false);
  const [loading, setLoading] = useState(false);
  const [isEventOpen, setIsEventOpen] = useState(true);

  // Check if event is still open
  useEffect(() => {
    const checkEventStatus = async () => {
      const { data, error } = await supabase
        .from('events')
        .select('status')
        .eq('id', eventId)
        .single();
      
      if (data?.status !== 'OPEN') {
        setIsEventOpen(false);
      }
    };
    checkEventStatus();
  }, [eventId]);

  const handleSubmit = async () => {
    if (!isEventOpen) {
      alert('Event is closed. Cannot submit predictions.');
      return;
    }

    try {
      setLoading(true);
      const predictedProb = prediction / 100; // Convert percent to decimal
      
      // Upsert prediction (insert or update)
      const user = (await supabase.auth.getUser()).data.user;
      
      const { error } = await supabase
        .from('predictions')
        .upsert(
          {
            event_id: eventId,
            user_id: user.id,
            predicted_prob: predictedProb,
            updated_at: new Date()
          },
          { onConflict: 'event_id,user_id' }
        );

      if (error) throw error;

      setSubmitted(true);
      
      // Trigger LMSR recalculation
      await updateGroupProbability(eventId);
      
      // UI feedback
      alert(`Prediction submitted: ${prediction}%`);
    } catch (err) {
      alert(`Error: ${err.message}`);
    } finally {
      setLoading(false);
    }
  };

  if (!isEventOpen) {
    return <div className="closed-message">This event is closed. Cannot predict.</div>;
  }

  return (
    <div className="prediction-slider">
      <label>Your Prediction: <strong>{prediction}%</strong></label>
      <input
        type="range"
        min="0"
        max="100"
        value={prediction}
        onChange={(e) => {
          setPrediction(parseInt(e.target.value));
          setSubmitted(false); // Allow re-submission
        }}
        disabled={loading}
      />
      <button onClick={handleSubmit} disabled={loading}>
        {loading ? 'Submitting...' : submitted ? 'Update Prediction' : 'Submit Prediction'}
      </button>
    </div>
  );
}

// Helper function
async function updateGroupProbability(eventId) {
  // Call your Edge Function or backend API
  // This recalculates current_prob based on all predictions
}
```

---

## Edge Function Patterns

### Pattern 1: LMSR Calculator

```typescript
// functions/calculateGroupProb.ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
);

export async function calculateGroupProb(eventId: string) {
  try {
    // Fetch all predictions for this event
    const { data: predictions, error } = await supabase
      .from('predictions')
      .select('predicted_prob')
      .eq('event_id', eventId);

    if (error || !predictions || predictions.length === 0) {
      return 0.5; // Default if no predictions
    }

    // Option 1: Simple Average (MVP)
    const sum = predictions.reduce((acc, p) => acc + p.predicted_prob, 0);
    const avgProb = sum / predictions.length;

    // Option 2: TODO - True LMSR (Phase 2)
    // const b = 100; // liquidity parameter
    // const lmsrProb = Math.exp(totalYes / b) / (Math.exp(totalYes / b) + Math.exp(totalNo / b));

    // Update event.current_prob
    const { error: updateError } = await supabase
      .from('events')
      .update({ current_prob: avgProb })
      .eq('id', eventId);

    if (updateError) throw updateError;

    return avgProb;
  } catch (err) {
    console.error('calculateGroupProb error:', err);
    throw err;
  }
}

// Deno entry point (for POST /functions/v1/calculateGroupProb)
Deno.serve(async (req) => {
  if (req.method === 'POST') {
    const { eventId } = await req.json();
    const result = await calculateGroupProb(eventId);
    return new Response(JSON.stringify({ groupProb: result }), {
      headers: { 'Content-Type': 'application/json' }
    });
  }
  return new Response('Method not allowed', { status: 405 });
});
```

### Pattern 2: Brier Score Calculator

```typescript
// functions/calculateBrierScores.ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
);

export async function calculateBrierScores(eventId: string) {
  try {
    // Fetch event outcome
    const { data: event, error: eventError } = await supabase
      .from('events')
      .select('final_outcome, squad_id')
      .eq('id', eventId)
      .single();

    if (eventError || !event) throw eventError;

    const outcome = event.final_outcome; // 0 or 1

    // Fetch all predictions
    const { data: predictions, error: predError } = await supabase
      .from('predictions')
      .select('id, user_id, predicted_prob')
      .eq('event_id', eventId);

    if (predError) throw predError;

    // Calculate Brier for each prediction
    for (const pred of predictions) {
      const brierScore = Math.pow(pred.predicted_prob - outcome, 2);

      // Update prediction with Brier score
      await supabase
        .from('predictions')
        .update({ brier_score: brierScore })
        .eq('id', pred.id);
    }

    // Update squad_members.squad_brier_score (average across all resolved events)
    await updateSquadLeaderboard(event.squad_id);

    return { brierScoresUpdated: predictions.length };
  } catch (err) {
    console.error('calculateBrierScores error:', err);
    throw err;
  }
}

async function updateSquadLeaderboard(squadId: string) {
  // SQL: Calculate average Brier for each user in squad
  const query = `
    SELECT 
      user_id,
      AVG(COALESCE(brier_score, 0.5)) as avg_brier
    FROM predictions p
    JOIN events e ON p.event_id = e.id
    WHERE e.squad_id = '${squadId}' AND e.status = 'CLOSED'
    GROUP BY user_id
  `;

  const { data: results, error } = await supabase
    .rpc('get_squad_brier_scores', { squad_id: squadId });

  if (error) throw error;

  // Update squad_members table
  for (const result of results) {
    await supabase
      .from('squad_members')
      .update({ squad_brier_score: result.avg_brier })
      .eq('squad_id', squadId)
      .eq('user_id', result.user_id);
  }
}

Deno.serve(async (req) => {
  if (req.method === 'POST') {
    const { eventId } = await req.json();
    const result = await calculateBrierScores(eventId);
    return new Response(JSON.stringify(result), {
      headers: { 'Content-Type': 'application/json' }
    });
  }
  return new Response('Method not allowed', { status: 405 });
});
```

---

## API Response Patterns

### Success Response

```json
{
  "status": "success",
  "data": {
    "event_id": "abc-123",
    "title": "Will it rain?",
    "current_prob": 0.675
  }
}
```

### Error Response

```json
{
  "status": "error",
  "error": "Event not found",
  "code": "EVENT_NOT_FOUND"
}
```

---

## Privacy Patterns (RLS)

### CRITICAL: Always check squad_id

**❌ Bad:**
```javascript
const { data } = await supabase
  .from('events')
  .select('*'); // VULNERABLE: No filtering!
```

**✅ Good:**
```javascript
const { data } = await supabase
  .from('events')
  .select('*')
  .eq('squad_id', squadId); // User can only see their own squad's events
```

### CRITICAL: Verify user is in squad before any action

```javascript
const isUserInSquad = async (userId, squadId) => {
  const { data, error } = await supabase
    .from('squad_members')
    .select('*')
    .eq('squad_id', squadId)
    .eq('user_id', userId)
    .single();
  
  return !!data && !error;
};

// Use in every endpoint:
if (!await isUserInSquad(userId, squadId)) {
  throw new Error('Unauthorized');
}
```

---

## Naming Conventions

| Category | Convention | Example |
|----------|-----------|---------|
| Database Tables | `snake_case`, plural | `squad_members`, `predictions` |
| Columns | `snake_case` | `predicted_prob`, `current_prob` |
| React Components | `PascalCase` | `CreateEventForm`, `PredictionSlider` |
| Functions | `camelCase` | `calculateGroupProb`, `updateSquadLeaderboard` |
| Constants | `SCREAMING_SNAKE_CASE` | `PREDICTION_MIN = 0`, `PREDICTION_MAX = 1` |
| URLs / Routes | `/kebab-case` | `/api/predictions/submit`, `/squads/:id/leaderboard` |
| CSS Classes | `.kebab-case` | `.squad-dashboard`, `.prediction-slider` |

---

## Error Handling Pattern

```javascript
const safeAction = async (action, errorMessage) => {
  try {
    const result = await action();
    return { success: true, data: result };
  } catch (err) {
    console.error(errorMessage, err);
    return { success: false, error: err.message };
  }
};

// Usage:
const { success, data, error } = await safeAction(
  () => updatePrediction(eventId, prediction),
  'Failed to update prediction'
);

if (!success) {
  showToast(`Error: ${error}`, 'error');
} else {
  showToast('Prediction updated!', 'success');
}
```

---

## Debugging Tips

### Check RLS Policies
```sql
-- In Supabase dashboard
SELECT * FROM information_schema.tables WHERE table_schema = 'public';
SELECT * FROM pg_policies WHERE tablename IN ('events', 'predictions', 'squad_members');
```

### Verify Brier Calculation
```sql
-- Manual spot-check
SELECT 
  user_id, 
  predicted_prob, 
  final_outcome,
  POWER((predicted_prob - final_outcome), 2) as calculated_brier,
  brier_score as stored_brier
FROM predictions p
JOIN events e ON p.event_id = e.id
WHERE e.id = 'event-abc-123';
-- Look for mismatches between calculated_brier and stored_brier
```

### Monitor Edge Function Logs
```bash
# In Supabase dashboard → Edge Functions → Select function → Logs
# Look for errors, timeouts, or unexpected behavior
```

---

**Last Updated:** Today  
**Owner:** AI Builder + You (math verification)
