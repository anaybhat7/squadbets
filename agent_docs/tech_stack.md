# Tech Stack & Tools

## Overview
SquadBets MVP uses a **serverless, zero-cost stack** optimized for rapid AI-assisted development and precise mathematical calculations.

### Technology Choices

| Component | Technology | Version | Why This Choice |
|-----------|-----------|---------|-----------------|
| **Frontend/Builder** | Base44 | Latest | AI-friendly no-code builder; rapid prototyping without deployment headaches |
| **Backend/Database** | Supabase (PostgreSQL) | Managed | Row-Level Security (RLS) for squad privacy; free tier supports MVP |
| **Authentication** | Supabase Auth (Google OAuth) | Built-in | Simple integration; no backend coding required |
| **Math Engine** | Supabase Edge Functions | Serverless | Auto-scaling for LMSR and Brier calculations; free tier |
| **Deployment** | Vercel / PWA | - | Instant deploy; works on mobile without App Store approval |
| **Styling** | Base44 built-in design system | - | Pre-built mobile components; consistency guaranteed |

## Database (Supabase PostgreSQL)

### Schema Definition

```sql
-- Users' global profile
CREATE TABLE profiles (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id),
  display_name TEXT NOT NULL,
  avatar_url TEXT,
  global_brier_avg DECIMAL(5,4) DEFAULT 0.5,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Squad containers (private groups)
CREATE TABLE squads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  invite_code TEXT UNIQUE NOT NULL, -- 6-char alphanumeric
  creator_id UUID NOT NULL REFERENCES profiles(user_id),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Squad membership + local ranking
CREATE TABLE squad_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  squad_id UUID NOT NULL REFERENCES squads(id),
  user_id UUID NOT NULL REFERENCES profiles(user_id),
  squad_brier_score DECIMAL(5,4), -- average Brier score in this squad
  joined_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(squad_id, user_id)
);

-- Events (the bets)
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  squad_id UUID NOT NULL REFERENCES squads(id),
  creator_id UUID NOT NULL REFERENCES profiles(user_id),
  title TEXT NOT NULL,
  description TEXT,
  status TEXT DEFAULT 'OPEN' CHECK (status IN ('OPEN', 'CLOSED', 'VOID')),
  current_prob DECIMAL(5,4) DEFAULT 0.5, -- group probability (LMSR result)
  final_outcome DECIMAL(1,0), -- 1 for Yes, 0 for No, NULL if VOID
  created_at TIMESTAMP DEFAULT NOW(),
  closed_at TIMESTAMP
);

-- Predictions (one per user per event)
CREATE TABLE predictions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id),
  user_id UUID NOT NULL REFERENCES profiles(user_id),
  predicted_prob DECIMAL(5,4) NOT NULL, -- 0.0 to 1.0
  brier_score DECIMAL(5,4), -- calculated after event closes
  submitted_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(event_id, user_id) -- one prediction per user per event
);

-- Audit log (optional, for debugging)
CREATE TABLE event_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  squad_id UUID REFERENCES squads(id),
  event_description TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Row-Level Security (RLS) Policies

```sql
-- Profiles: Users see their own profile + squad members' profiles
CREATE POLICY "profiles_self_access" ON profiles
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "profiles_squad_members_access" ON profiles
  FOR SELECT USING (
    user_id IN (
      SELECT user_id FROM squad_members 
      WHERE squad_id IN (
        SELECT squad_id FROM squad_members 
        WHERE user_id = auth.uid()
      )
    )
  );

-- Squads: Users see squads they joined
CREATE POLICY "squads_member_access" ON squads
  FOR SELECT USING (
    id IN (
      SELECT squad_id FROM squad_members 
      WHERE user_id = auth.uid()
    )
  );

-- Events: Users see events from squads they joined
CREATE POLICY "events_member_access" ON events
  FOR SELECT USING (
    squad_id IN (
      SELECT squad_id FROM squad_members 
      WHERE user_id = auth.uid()
    )
  );

-- Predictions: Basic access
CREATE POLICY "predictions_member_access" ON predictions
  FOR SELECT USING (
    event_id IN (
      SELECT events.id FROM events
      JOIN squad_members ON events.squad_id = squad_members.squad_id
      WHERE squad_members.user_id = auth.uid()
    )
  );

-- Users can create predictions for their own event only
CREATE POLICY "predictions_create_own" ON predictions
  FOR INSERT WITH CHECK (user_id = auth.uid());

-- Users can update only their own predictions (if event still OPEN)
CREATE POLICY "predictions_update_own_open" ON predictions
  FOR UPDATE USING (
    user_id = auth.uid() AND
    event_id IN (
      SELECT id FROM events WHERE status = 'OPEN'
    )
  );
```

## Edge Functions (Serverless Math)

### LMSR Calculation

```javascript
// `functions/calculateGroupProb.ts`
// Triggered: When a prediction is created or updated

export async function calculateGroupProb(eventId: string) {
  // Fetch all predictions for this event
  const predictions = await supabase
    .from('predictions')
    .select('predicted_prob')
    .eq('event_id', eventId);
  
  // Option 1: Simple Average (recommended for MVP with small groups)
  const avgProb = predictions.data.reduce((sum, p) => sum + p.predicted_prob, 0) 
                  / predictions.data.length;
  
  // Option 2: True LMSR (if we need more sophistication)
  // const b = 100; // liquidity parameter
  // const totalYes = predictions.data.reduce((sum, p) => sum + p.predicted_prob * 100, 0);
  // const totalNo = predictions.data.reduce((sum, p) => sum + (1 - p.predicted_prob) * 100, 0);
  // const lmsrProb = Math.exp(totalYes / b) / (Math.exp(totalYes / b) + Math.exp(totalNo / b));
  
  // Update event.current_prob
  await supabase
    .from('events')
    .update({ current_prob: avgProb })
    .eq('id', eventId);
  
  return avgProb;
}
```

### Brier Score Calculation

```javascript
// `functions/calculateBrierScores.ts`
// Triggered: When event.status changes to 'CLOSED'

export async function calculateBrierScores(eventId: string) {
  // Fetch event outcome
  const event = await supabase
    .from('events')
    .select('final_outcome')
    .eq('id', eventId)
    .single();
  
  const outcome = event.data.final_outcome; // 0 or 1
  
  // Fetch all predictions for this event
  const predictions = await supabase
    .from('predictions')
    .select('id, predicted_prob')
    .eq('event_id', eventId);
  
  // Calculate Brier score for each
  for (const pred of predictions.data) {
    const brierScore = Math.pow(pred.predicted_prob - outcome, 2);
    
    await supabase
      .from('predictions')
      .update({ brier_score: brierScore })
      .eq('id', pred.id);
  }
  
  // Update squad_members.squad_brier_score (average across all events)
  // ... (SQL to calculate running average)
}
```

## API Endpoints (Base44 Backend Calls)

### Authentication
- **Login:** `POST /auth/login` → Google OAuth flow
- **Logout:** `POST /auth/logout` → Clear session
- **Current User:** `GET /auth/me` → Return `auth.uid()` + profile

### Squads
- **Create Squad:** `POST /squads` → Params: `name` → Returns: `invite_code`
- **List My Squads:** `GET /squads/me` → Returns: squads user joined
- **Join Squad:** `POST /squads/join` → Params: `invite_code` → Updates: `squad_members`

### Events
- **Create Event:** `POST /events` → Params: `squad_id, title, description` → Status: OPEN
- **List Events:** `GET /events/:squad_id` → Returns: all events in squad
- **Update Event Status:** `PUT /events/:id/status` → Params: `new_status` (CLOSED/VOID)
- **Resolve Event:** `PUT /events/:id/resolve` → Params: `final_outcome` (0/1) → Triggers Brier calculation

### Predictions
- **Create/Update Prediction:** `POST /predictions` → Params: `event_id, predicted_prob` → Triggers LMSR update
- **Get My Predictions:** `GET /predictions?event_id=X` → Returns: user's prediction only (privacy check)
- **Get All Predictions (if event CLOSED):** `GET /events/:id/predictions` → Returns: all predictions + names + Brier scores

### Leaderboard
- **Get Squad Leaderboard:** `GET /squads/:id/leaderboard` → Returns: sorted members by `squad_brier_score`

## Setup Commands (Base44 Quickstart)

```bash
# 1. Create Base44 project
base44 create squabbets-mvp

# 2. Connect to Supabase
# - Create Supabase project at supabase.com
# - Copy API key and URL
# - Paste into Base44 project settings

# 3. Run the schema setup (run SQL in Supabase dashboard)
# - Copy entire schema.sql file above
# - Paste into Supabase SQL editor
# - Execute

# 4. Enable RLS in Supabase dashboard
# - For each table: Settings → Enable RLS
# - Attach the policies above

# 5. Set up Edge Functions
# - Create functions/calculateGroupProb.ts
# - Create functions/calculateBrierScores.ts
# - Deploy to Supabase

# 6. Deploy Base44 app
base44 deploy
```

## Key Integrations Checklist

- [ ] Google OAuth configured in Supabase Auth
- [ ] Supabase RLS policies enabled and tested
- [ ] Edge Functions deployed and callable from Base44
- [ ] CORS configured to allow Base44 frontend → Supabase backend
- [ ] Free tier quotas understood (see Supabase pricing)
- [ ] Error logging set up (check Base44 logs)

## Notes on Architecture

1. **Why Serverless?** No servers to manage; pricing is per-use. For 10 users, cost is $0.
2. **Why RLS?** Ensures privacy at the database layer (not just application layer). Can't accidentally expose another squad's data.
3. **Why Edge Functions for Math?** Runs LMSR and Brier calculations instantly; no latency from app server.
4. **Why Simple Average first?** True LMSR is elegant but needs a "liquidity parameter (b)"; simple average is more intuitive for a friend group of 10.

---

**Last Updated:** Today  
**Tech Lead:** [You]  
**Status:** Ready for implementation
