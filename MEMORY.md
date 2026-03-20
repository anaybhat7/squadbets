# MEMORY.md — SquadBets MVP Working Notes

## 🏗️ Active Phase & Goal

**Current Phase:** Phase 1 - Skeleton (Auth + Squad Setup)
**Goal:** Get users authenticated and able to create/join squads via invite codes
**Est. Completion:** Day 2-3 of sprint

## 📊 Project Facts

- **Product:** SquadBets (Private prediction market)
- **Users:** Small friend groups (5-10 people per squad)
- **Math Core:** LMSR for group probability + Brier scores for rankings
- **Platform:** Base44 frontend + Supabase backend
- **Constraint:** 2-week MVP launch, $0 budget

## 🎯 Key Formulas (Copy Exactly)

**LMSR (Group Probability):**
```
P = e^(q/b) / (e^(sum_yes/b) + e^(sum_no/b))
```
For MVP with small groups, simplified: Average of all predictions as group signal
(Ask AI which is more stable for 10-person group)

**Brier Score (Accuracy):**
```
Score = (Prediction - Outcome)²
0 = Perfect, 1 = Perfectly wrong, 0.5 = 50/50 guess
```

## 🔐 Privacy Rules (CRITICAL)

1. **Squad Isolation:** User can only see events from squads they joined
2. **Prediction Hiding:** While event is OPEN, hide individual predictions (show only group average)
3. **One Prediction:** Each user gets ONE prediction per event (edit allowed before close)
4. **Reveal on Close:** When event marked as Closed, show all predictions with usernames

## 📱 UI/UX Vibe

- Mobile-first (test on phone constantly)
- Clean, minimal interface
- Educational for prediction market newcomers
- "How it Works" tab explaining Brier scores

## ✅ Phase 1 Scope

Must-Have:
- [ ] Google Auth
- [ ] Create Squad (invite code generation)
- [ ] Join Squad (enter invite code)
- [ ] User profile (display name)

Nice-to-Have:
- [ ] Squad settings/admin
- [ ] Invite friends via email

## ⚠️ Common Pitfalls

- DO NOT show individual predictions while event is OPEN (causes herd bias)
- DO NOT allow multiple predictions per event (users can edit, not duplicate)
- DO NOT forget Row-Level Security (RLS) in Supabase
- DO NOT deploy without mobile testing

## 🔗 Session Notes

**Session 1: Setup (Today)**
- Created AGENTS.md and agent_docs
- Ready to start Phase 1

**Session 2: [To be filled in]**
- [Notes here]

---

*Last Updated: Today*
*Next Step: Open Base44 and start Phase 1 prompt*
