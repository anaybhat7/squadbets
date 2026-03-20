# Project Brief (Persistent)

## Vision

SquadBets is a **private prediction market** where friend groups compete on forecast accuracy. Instead of winning money, you compete for bragging rights using **Brier scores** — a statistical measure of prediction accuracy. Built for ORIE students and friends who love data, decision-making, and a little friendly competition.

## Core Values

1. **Precision First:** Math is not negotiable. LMSR and Brier calculations must be exact.
2. **Privacy by Default:** Users can only see their squad's data; no data leakage between groups.
3. **Simplicity in MVP:** One prediction per event (Yes/No only); no complex multi-outcome logic yet.
4. **Mobile-First:** App should feel natural on a phone, not like a web page squeezed down.

## Success Metrics (MVP)

- Users can create/join at least 1 squad each
- At least 5 events created in a squad
- All Brier scores calculated correctly (manual verification)
- No data leaks between squads (verified by QA)
- Mobile responsiveness verified on iOS and Android

## Conventions & Patterns

### Naming Conventions

**Database Tables:** `snake_case`, singular for lookup tables (`profiles`), plural for transactional (`predictions`)
Example: `squad_members`, `event_predictions` (NO - use `predictions`)

**API Endpoints:** RESTful, `/resource/:id/action`
Example: `GET /squads/me`, `POST /events/:id/resolve`

**Component Names (Base44 screens):** PascalCase, descriptive
Example: `CreateSquadScreen`, `PredictionSlider`, `LeaderboardTable`

**Variables:** camelCase in frontend, snake_case in database
Example: `const predictedProb = 0.65;` (JS) → `predicted_prob DECIMAL(5,4)` (SQL)

### Architecture Patterns

**Data Flow:**
1. User action in Base44 → API call to Supabase
2. Supabase triggers Edge Function (LMSR update)
3. Base44 polls for updated `current_prob`
4. UI re-renders with new group signal

**Error Handling:**
```javascript
// Base44 pattern:
try {
  const response = await supabase.from('events').insert({...});
  if (response.error) throw response.error;
  return response.data;
} catch (error) {
  console.error('Event creation failed:', error);
  showToast('Failed to create event', 'error');
  return null;
}
```

**Privacy Pattern:**
Every query to Supabase must include a `WHERE squad_id = ?` clause.
Base44 context must validate `auth.uid() IN squad_members` before showing squad data.

### Key Commands

```bash
# Development
base44 dev          # Start local dev server

# Testing (local)
curl -X GET "https://[supabase-url]/rest/v1/events?squad_id=eq.abc123" \
  -H "Authorization: Bearer [SUPABASE_ANON_KEY]"

# Deployment
base44 deploy       # Deploy to Vercel

# Database inspection
# Go to Supabase dashboard → SQL editor → Run queries

# View Edge Function logs
# Supabase dashboard → Edge Functions → Select function → Logs
```

### Quality Gates

**Before Committing:**
- No console errors in browser
- All RLS policies working (manually tested)
- Brier score calculation matches manual math
- Mobile responsiveness verified

**Before Phase Transition:**
- All REVIEW-CHECKLIST items checked ✅
- One feature fully tested (end-to-end)
- No technical debt blocking next phase

### When to Ask an AI

✅ **Good AI questions:**
- "Build the Base44 component for event creation with a date picker"
- "Write the Edge Function that calculates LMSR; here's the formula..."
- "Explain the RLS policy that prevents User A from seeing User B's squad"

❌ **Bad AI questions:**
- "Build the whole app" (too big; break into features)
- "Make the math work" (specify the exact formula)
- "Fix this error" (show the exact error message)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    Base44 (Frontend)                 │
│  [Auth] → [Squad Screen] → [Event Screen] → [Leaderboard]  │
└──────────────┬──────────────────────────────────────┘
               │ HTTP/REST
               ↓
┌──────────────────────────────────────────────────────┐
│             Supabase (Backend + Database)            │
│  - Auth (Google OAuth)                                │
│  - PostgreSQL (RLS enforced)                          │
│  - Edge Functions (LMSR & Brier calculations)         │
└──────────────────────────────────────────────────────┘
```

## Decision Log

### Decision: Simple Average vs. True LMSR (Phase 2)
- **Choice:** Start with simple average of predictions
- **Reasoning:** For 5-10 people, simpler is faster to debug; LMSR adds complexity without much benefit for small groups
- **Revisit:** Phase 2 can upgrade if group probability feels "flat"

### Decision: One Prediction Per Event (Phase 1)
- **Choice:** Users can edit their prediction before event closes, not duplicate
- **Reasoning:** Keeps Brier calculations clean; no "averaging multiple predictions"
- **Revisit:** Phase 2 might allow "second chance" bets

### Decision: Private Squads Only (Phase 1)
- **Choice:** No "global" leaderboard; each squad is isolated
- **Reasoning:** Matches the "friend group" use case; simpler privacy model
- **Revisit:** Phase 3 could add optional squad discovery

## Tech Lead Contact

- **Build Expert:** AI (Base44 builder)
- **Math Verification:** You (ORIE background)
- **QA & Testing:** You (mobile focus)

## Update Cadence

- **Weekly:** Review AGENTS.md for any phase changes
- **Per Feature:** Update this brief if new conventions emerge
- **Pre-Launch:** Final review of all architecture decisions

---

**Project Status:** 🟢 In Planning Phase  
**Last Sync:** Today  
**Next Review:** Day 3 of build (after Phase 1 complete)
