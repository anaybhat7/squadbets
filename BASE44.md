# BASE44.md — Base44 Configuration for SquadBets

## Project Context

- **App:** SquadBets (MVP)
- **Stack:** Base44 (frontend builder) + Supabase (backend)
- **Tech Level:** Vibe-coder (AI does heavy lifting, you guide)
- **Timeline:** 2 weeks to MVP

---

## How to Use Base44 with This Project

### Phase 1: Skeleton (Days 1-2)

**Setup Steps:**

1. **Create a new Base44 project**
   ```bash
   base44 create squabbets-mvp
   cd squabbets-mvp
   ```

2. **Connect to Supabase**
   - Go to [supabase.com](https://supabase.com) → Create new project
   - Copy API URL and Anon Key
   - In Base44 dashboard: Project Settings → Database → Paste Supabase credentials

3. **Set up Google OAuth**
   - In Base44: Auth & Security → Add Provider → Google OAuth
   - Follow the wizard to register your app with Google

4. **Run the database schema**
   - Get the SQL file from AGENTS.md or `agent_docs/tech_stack.md`
   - In Supabase: SQL Editor → New Query → Paste schema → Execute

5. **Enable Row-Level Security (RLS)**
   - In Supabase: Security → Policies
   - For each table: Enable RLS, add the policies from `agent_docs/tech_stack.md`

**Your First Prompt to Base44:**
> "I'm building SquadBets: a private prediction market for friend groups. Start by setting up:
>
> 1. Google OAuth login screen
> 2. After login, show a dashboard with 'Create Squad' and 'Join Squad' buttons
> 3. Create Squad modal: User enters squad name, I generate a unique 6-character invite code
> 4. Join Squad modal: User enters invite code, system looks it up and adds them to squad_members table
> 5. After joining, show squad members list with their display names
>
> Use Supabase for the database. Make sure everything is privacy-protected using Row-Level Security."

---

### Phase 2: Predictions (Days 3-5)

**Components to Build:**

1. **Squad Feed Screen**
   ```
   Your Prompt:
   "Build the Squad Feed showing all events in a squad.
   
   - Show a list of events with title, description, status (OPEN/CLOSED), and current group probability
   - At the bottom, add a 'Create Event' button that opens a modal
   - Events should be sortable by OPEN first, then CLOSED
   - Click an event to open details screen"
   ```

2. **Event Creation Modal**
   ```
   Your Prompt:
   "Build the Create Event form:
   
   - Input field for event title (required)
   - Textarea for description (optional)
   - 'Create' button that saves to events table
   - After creation, close modal and refresh the squad feed
   - Event should default to status='OPEN' and current_prob=0.5"
   ```

3. **Prediction Slider Screen**
   ```
   Your Prompt:
   "Build the Prediction screen for an event:
   
   - Show event title and description
   - Show group probability (current_prob from database)
   - Show a 0-100% slider for prediction
   - 'Submit Prediction' button
   - After clicking: save to predictions table (one per user per event)
   - If user predicts again, update their existing prediction, don't create a new one
   - Show a confirmation: 'Your prediction: 75%'
   - While event is OPEN, don't show other users' predictions"
   ```

4. **Call Edge Functions**
   ```
   Your Prompt:
   "When a prediction is submitted, trigger the 'calculateGroupProb' Edge Function:
   
   - Extract the event_id
   - Call the function via HTTPS: POST /functions/v1/calculateGroupProb
   - Pass: { eventId: '...' }
   - After the function returns, refetch the event to get the updated current_prob
   - Update the UI with the new group probability
   
   See agent_docs/tech_stack.md for the function code."
   ```

---

### Phase 3: Leaderboard (Days 6-7)

**Components to Build:**

1. **Leaderboard Screen**
   ```
   Your Prompt:
   "Build the Leaderboard screen for a squad:
   
   - Query the squad_members table and calculate average Brier score for each user
   - Sort by avg_brier_score ascending (lowest = best)
   - Show columns: Rank | Name | Avg Brier Score
   - Update in real-time after an event is resolved
   - Make it look clean and readable"
   ```

2. **Event Resolution**
   ```
   Your Prompt:
   "Add a 'Resolve Event' button (for event creator only):
   
   - Show a modal: 'What happened? [Yes] [No] [Void]'
   - User clicks one option
   - Update events table: status='CLOSED', final_outcome=1 (for Yes) or 0 (for No)
   - Trigger the 'calculateBrierScores' Edge Function
   - After calculation, refresh and show leaderboard
   
   See agent_docs/tech_stack.md for the function code."
   ```

3. **Prediction Reveals**
   ```
   Your Prompt:
   "When event.status='CLOSED':
   
   - Show all predictions with user names and Brier scores
   - Format: 'User Name: 75% | Score: 0.0625'
   - Sort by user name
   - Hide predictoin names until this point"
   ```

---

### Phase 4: Polish (Days 8-10)

**Components to Build:**

1. **"How it Works" Tab**
   ```
   Your Prompt:
   "Create a 'How it Works' educational screen:
   
   Sections:
   1. What is Brier Score? (Explain: (prediction - outcome)^2, range 0-1, lower is better)
   2. How does the group probability work? (Simple average of all predictions)
   3. Privacy: Why can't I see others' predictions while event is OPEN?
   4. How is the leaderboard ranked?
   
   Include visual examples with numbers."
   ```

2. **Mobile Testing**
   ```bash
   # Test on phone
   base44 dev    # Start dev server
   # Open http://[your-ip]:3000 on phone
   
   # Check:
   # - All buttons are tap-able (48px minimum)
   # - Text is readable (not tiny)
   # - Slider works smoothly
   # - No horizontal scroll
   # - Forms are mobile-friendly
   ```

3. **Deployment**
   ```bash
   base44 deploy
   # This deploys to Vercel (free tier)
   # Get live URL and share with test users
   ```

---

## Key Base44 Prompting Tips

### DO:
✅ **Be specific about data sources**
- "Fetch from the 'events' table WHERE squad_id = currentSquad"
- "Save to the 'predictions' table with UPSERT logic"

✅ **Mention privacy requirements**
- "Enforce RLS so users only see their own squads"
- "Hide individual predictions while event is OPEN"

✅ **Include formula details**
- "Calculate Brier score as (predicted_prob - final_outcome)^2"
- "Average all predictions for group probability"

✅ **Ask for explanations**
- "How is the Brier score being calculated in your code?"
- "Explain the RLS policy that prevents data leakage"

### DON'T:
❌ **Ask for the whole app at once**
- Instead of: "Build SquadBets"
- Do this: "Build the login and squad creation screens"

❌ **Vague instructions**
- Instead of: "Make a prediction screen"
- Do this: "Build a slider (0-100%), a submit button, and fetch current_prob from events table"

❌ **Skip the math explanation**
- Instead of: "Calculate scores somehow"
- Do this: "Use this exact formula: (prediction - outcome)^2. Store as DECIMAL(5,4)"

❌ **Assume Base44 knows your project**
- Always include context: "Remember, predictions are stored as 0-1 decimals, not 0-100"

---

## Debugging Base44 Issues

### Issue: "RLS policy blocking my queries"

**Solution:**
```sql
-- In Supabase, run this to verify RLS is working:
SET ROLE authenticated; -- Pretend to be a logged-in user
SELECT * FROM events; -- This should be filtered by RLS

-- If you get ERROR, RLS policy is correct (expected for non-members)
-- If you get results, RLS policy needs adjustment
```

### Issue: "Edge Function not called"

**Check:**
1. Is the function deployed? (Supabase dashboard → Edge Functions → Check status)
2. Is the function name correct? (Check URL: `/functions/v1/[function-name]`)
3. Is the request payload correct? (POST with JSON body)

```bash
# Manual test:
curl -X POST https://[supabase-url]/functions/v1/calculateGroupProb \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer [SUPABASE_ANON_KEY]" \
  -d '{"eventId":"test-event-123"}'
```

### Issue: "Predictions update not shown in UI"

**Solution:**
- After submitting, refetch the event/predictions
- Or use real-time subscriptions (see Base44 docs)

```javascript
// After prediction submit:
await supabase.from('events').select('*').eq('id', eventId);
// Update UI with new current_prob
```

---

## Base44 + Supabase Integration Checklist

- [ ] Supabase project created and credentials saved
- [ ] Database schema deployed (copy from tech_stack.md)
- [ ] RLS policies enabled and tested
- [ ] Google OAuth flow works (login → dashboard)
- [ ] Create Squad flow saves to database
- [ ] Join Squad flow checks invite code
- [ ] Create Event flow saves to events table
- [ ] Prediction slider submits to predictions table
- [ ] LMSR Edge Function triggers and updates current_prob
- [ ] Event resolution triggers Brier score calculation
- [ ] Leaderboard fetches and sorts correctly
- [ ] Mobile responsive (tested on phone)
- [ ] Deployed to Vercel (live URL works)

---

## Commands Cheat Sheet

```bash
# Development
base44 dev              # Start local dev server (http://localhost:3000)
base44 build            # Build for production

# Deployment
base44 deploy           # Deploy to Vercel (free)
base44 logs             # View live server logs

# Database
base44 db:list          # List connected databases
base44 db:shell         # Open database shell

# Testing
base44 test             # Run your tests (if set up)
```

---

## When to Ask for Help

**Ask AI (me):**
- "Build this Base44 screen with these inputs and buttons"
- "I got this error: [error]. What's wrong?"
- "How do I fetch from Supabase in Base44?"

**Ask StackOverflow / Base44 Docs:**
- Base44-specific questions: base44.io/docs
- Supabase questions: supabase.com/docs
- JavaScript issues: stackoverflow.com

**Ask You (Tech Lead):**
- "Does this match the Brier score formula?"
- "Is this privacy-safe?"
- "Should we ship this or iterate?"

---

## Success Metrics for Phase 1+

| Phase | Metric | Status |
|-------|--------|--------|
| **Phase 1** | Login works, squads created | ⬜ |
| **Phase 2** | Predictions submitted, group prob updates | ⬜ |
| **Phase 3** | Brier scores calculated correctly | ⬜ |
| **Phase 4** | Deployed, mobile-tested, users can access | ⬜ |

---

**Last Updated:** Today  
**Next Step:** Open Base44, create new project, and follow Phase 1 instructions above  
**Questions?** Review AGENTS.md or agent_docs/ files
