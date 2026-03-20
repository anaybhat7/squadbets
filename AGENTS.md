# AGENTS.md — SquadBets MVP Agent Instructions

## 🎯 Project Context

- **Product:** SquadBets (MVP)
- **Vision:** A private prediction market where friend groups compete on forecast accuracy using Brier scores
- **Tech Stack:** Base44 (frontend), Supabase (backend + auth), Edge Functions (LMSR math)
- **Timeline:** 2 weeks to MVP launch
- **Your Role:** Vibe-coder — AI handles complexity, you guide and test
- **Current Phase:** Phase 1 - Skeleton (Auth + Squad Setup)

## 📋 How I Should Think

1. **Understand Intent First:** Before I code, I'll confirm I understand what you're building
2. **Ask If Unsure:** Missing critical info? I'll ask ONE specific clarifying question
3. **Plan Before Coding:** I propose an approach, you approve, then I implement
4. **Verify After Changes:** Test features and verify they match requirements
5. **Explain Trade-offs:** When recommending something, I mention alternatives

## 🚀 Build Phases

### Phase 1: The Skeleton (Week 1)
- [ ] Google Auth login integration
- [ ] "Create/Join Squad" screen with invite codes
- [ ] Squad profile setup (optional)
- **Milestone:** Users can log in and join private squads

### Phase 2: The Engine (Week 1-2)
- [ ] Squad feed showing events
- [ ] Create event (Yes/No questions only)
- [ ] 0-100% prediction slider (one prediction per user per event)
- [ ] LMSR calculation for group probability
- **Milestone:** Real-time squad probability updates

### Phase 3: The Result (Week 2)
- [ ] Event resolution (mark as Yes/No/Void)
- [ ] Brier score calculation: `(prediction - outcome)²`
- [ ] Squad leaderboard (sorted by average Brier score, lowest = best)
- **Milestone:** Rankings visible after event closes

### Phase 4: The Polish (Week 2)
- [ ] "How it Works" educational tab
- [ ] Vibe memos (optional)
- [ ] Mobile testing
- **Milestone:** Ready for production

## 🔑 Critical Requirements (NON-NEGOTIABLE)

### Math Precision
```
LMSR Formula: Pᵢ = e^(qᵢ/b) / Σ(e^(qⱼ/b))
Brier Score: BS = (prediction - outcome)²
- Perfect prediction (1.0 → outcome 1): BS = 0
- Wrong prediction (0.0 → outcome 1): BS = 1
```

### Privacy Rules
- Users can ONLY see events from squads they've joined
- While event is OPEN: hide individual predictions, show only group average
- After event CLOSES: reveal all predictions with names
- Prevent "herd bias" by hiding live predictions

### Squad Isolation
- One prediction per user per event (edit allowed before close)
- Predictions can be edited if event status is "OPEN"
- Each squad completely isolated from others

## 🗄️ Database Schema

| Table | Key Columns | Purpose |
|-------|------------|---------|
| **profiles** | `user_id`, `display_name`, `global_brier_avg` | User identity & global score |
| **squads** | `id`, `name`, `invite_code`, `creator_id` | Squad containers |
| **squad_members** | `squad_id`, `user_id`, `squad_brier_score` | Local ranking |
| **events** | `id`, `squad_id`, `title`, `status` (Open/Closed), `final_outcome` | The bets |
| **predictions** | `id`, `event_id`, `user_id`, `predicted_prob` (0-1) | One per user per event |

## 🎮 Building in Base44

**First Prompt:**
> "I'm building SquadBets: a private prediction market where friends compete on forecast accuracy. Start by setting up Google Auth login and a 'Create/Join Squad' screen. Squads must be private, joinable only via invite code."

**Next Prompts (one at a time):**
- "Build the Squad Feed showing all events. Members can create Yes/No events with a 0-100% slider for predictions."
- "Implement the LMSR calculation — when users submit predictions, update the group's average probability."
- "Add event resolution: creator marks event as Yes (1) or No (0), then calculate Brier scores for all predictors."
- "Build the leaderboard: rank users by average Brier score (lowest = best). Update it in real-time."

## ✅ Quality Gates

- [ ] Test each feature before moving to next phase
- [ ] Row-Level Security (RLS) enforced in Supabase
- [ ] No individual predictions visible while event is OPEN
- [ ] Brier score calculation verified (test with manual calc)
- [ ] Mobile responsiveness tested on phone
- [ ] All users isolated to their own squads

## 🛠️ Key Commands (Supabase Backend)

```sql
-- Check RLS is enforced
SELECT * FROM events WHERE squad_id = [squad_id];

-- Calculate Brier scores for an event
SELECT 
  user_id, 
  predicted_prob,
  POWER((predicted_prob - final_outcome), 2) as brier_score
FROM predictions
WHERE event_id = [event_id];

-- Squad leaderboard
SELECT 
  u.display_name, 
  AVG(POWER((p.predicted_prob - e.final_outcome), 2)) as avg_brier_score
FROM predictions p
JOIN predictions_events e ON p.event_id = e.id
JOIN profiles u ON p.user_id = u.id
WHERE e.squad_id = [squad_id]
GROUP BY u.display_name
ORDER BY avg_brier_score ASC;
```

## ⚠️ What NOT To Do

- Do NOT delete events without explicit confirmation
- Do NOT allow multiple predictions per user per event (edit-in-place only)
- Do NOT show individual predictions while event is OPEN
- Do NOT skip Brier score verification before leaderboard goes live
- Do NOT deploy without testing mobile responsiveness
- Do NOT change the LMSR or Brier formulas without approval

## 🚨 Troubleshooting

**If Base44 seems confused:**
- Start with: "Read AGENTS.md completely, then confirm you understand the LMSR and Brier formulas"

**If users can see other squads' events:**
- Check Supabase RLS: every query must filter by `squad_id` and verify the user is a member

**If Brier scores don't match manual calculation:**
- Verify the formula: `(prediction - outcome)²`, not absolute difference
- Check that predictions are stored as decimals (0-1), not percentages (0-100)

**If math seems off:**
- Have me explain the exact SQL or function that calculates it
- We'll verify it matches the ORIE requirements

---

## 📚 Additional Resources

- **Supabase Docs:** supabase.com/docs
- **Base44 Docs:** base44.io/docs
- **LMSR Reference:** A user-friendly explanation: "LMSR dynamically adjusts group probability based on the sum of all predictions"
- **Brier Score:** https://en.wikipedia.org/wiki/Brier_score

---

**Status:** Ready for Phase 1. Let me know when you want to start building!
