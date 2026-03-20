# REVIEW-CHECKLIST.md — SquadBets MVP Quality Gates

Use this checklist BEFORE moving to the next phase. Each phase must pass all gates.

---

## Phase 1 Review: Skeleton (Auth + Squad Setup)

- [ ] **Google Auth works**
  - [ ] Successful login redirects to app dashboard
  - [ ] User profile saved to database
  - [ ] Logout works and clears session
  
- [ ] **Squad creation works**
  - [ ] Creator can generate a unique invite code
  - [ ] Invite code is visible to creator
  - [ ] Invite code can be copied/shared
  
- [ ] **Squad joining works**
  - [ ] User can enter valid invite code and join
  - [ ] Invalid code shows error message
  - [ ] After joining, user appears in squad_members table
  - [ ] User can see they're part of the squad
  
- [ ] **User profiles work**
  - [ ] Display name is editable
  - [ ] Display name shows on squad members list
  - [ ] Avatar/initial is visible (if implemented)
  
- [ ] **Privacy check**
  - [ ] Users can only see squads they joined (check in Supabase)
  - [ ] Another user can't join someone else's squad without code
  
- [ ] **Mobile test**
  - [ ] All screens render correctly on phone
  - [ ] Touch interactions work (buttons, inputs, forms)
  - [ ] Navigation is intuitive on mobile

**GATE:** All boxes checked? ✅ → Ready for Phase 2

**Issues found?** → Document and fix before moving on

---

## Phase 2 Review: Engine (Predictions + LMSR)

- [ ] **Event creation works**
  - [ ] Members can create Yes/No events
  - [ ] Event title and description save correctly
  - [ ] Events appear in squad feed
  - [ ] Event shows "OPEN" status
  
- [ ] **Prediction form works**
  - [ ] Slider ranges 0-100% correctly
  - [ ] User can submit ONE prediction per event
  - [ ] Submitted prediction shows on their screen
  - [ ] Submission timestamp recorded
  
- [ ] **Prediction editing works**
  - [ ] User can move slider and update prediction (before close)
  - [ ] Updated prediction overwrites old one (no duplicates)
  - [ ] Editing after event closes shows error
  
- [ ] **LMSR calculation works**
  - [ ] Group probability updates in real-time as users predict
  - [ ] Group probability visible to members while OPEN
  - [ ] Individual predictions hidden (only average shown)
  - [ ] Manual calculation matches UI display
    - *Example: If 3 users predict 60%, 70%, 50%, group avg should be 60%*
  
- [ ] **Privacy check**
  - [ ] User A can't see User B's prediction while event OPEN
  - [ ] Only "Group Average: 60%" is visible
  - [ ] Count of predictions visible? (optional, helps engagement)
  
- [ ] **Edge cases handled**
  - [ ] New event starts at 50% (no predictions yet)
  - [ ] What happens if only 1 person predicts? (shows their %)
  - [ ] Decimal precision is correct (0-1, not 0-100)
  
- [ ] **Mobile test**
  - [ ] Slider works smoothly on mobile
  - [ ] Keyboard doesn't break layout
  - [ ] Feed scrolling smooth

**GATE:** All boxes checked? ✅ → Ready for Phase 3

**Issues found?** → Fix before moving on

---

## Phase 3 Review: Results (Brier Scores + Leaderboard)

- [ ] **Event resolution works**
  - [ ] Creator can mark event as Yes/No (or Void?)
  - [ ] Status changes to "CLOSED"
  - [ ] Predictions are no longer editable
  
- [ ] **Brier score calculation**
  - [ ] For each predictor: `(prediction - outcome)²` calculated
  - [ ] Scores stored in database
  - [ ] Manual check: predict 0.8, outcome 1.0 → score should be 0.04
  - [ ] Manual check: predict 0.3, outcome 0 → score should be 0.09
  
- [ ] **Leaderboard works**
  - [ ] Shows all squad members with their average Brier score
  - [ ] Sorted: **lowest score at TOP** (lowest = best)
  - [ ] Score updates after event resolves
  - [ ] Multiple events: average is across all resolved events
  - [ ] Leaderboard shows in real-time
  
- [ ] **Prediction reveal**
  - [ ] After close: all predictions visible with usernames
  - [ ] Shows: "User X predicted 75%, event was Yes"
  - [ ] Brier score shown for each prediction
  
- [ ] **Edge cases handled**
  - [ ] User with no predictions: excluded from leaderboard? (decide & implement)
  - [ ] New users: start at 0.5 avg? (decide & implement)
  - [ ] Void events: how do they affect scores? (document decision)
  
- [ ] **Mobile test**
  - [ ] Leaderboard displays well on mobile
  - [ ] Can swipe/scroll to see full predictions
  - [ ] Numbers read clearly (not truncated)

**GATE:** All boxes checked? ✅ → Ready for Phase 4

**Issues found?** → Fix before moving on

---

## Phase 4 Review: Polish (UI + Docs + Mobile)

- [ ] **"How it Works" tab**
  - [ ] Explains LMSR in simple terms
  - [ ] Explains Brier score in simple terms
  - [ ] Explains privacy rules
  - [ ] Accessible to first-time users
  
- [ ] **Final mobile test**
  - [ ] All pages load fast (< 2 sec)
  - [ ] All buttons/links clickable
  - [ ] No layout shifts or broken elements
  - [ ] Text readable (not too small)
  - [ ] Touch targets large enough (≥ 48px)
  
- [ ] **Performance**
  - [ ] App loads within 3 seconds
  - [ ] Leaderboard renders smoothly with 10+ users
  - [ ] No console errors in browser
  
- [ ] **Usability test (Ask a friend)**
  - [ ] Can they create a squad? Yes / No
  - [ ] Can they join a squad? Yes / No
  - [ ] Can they understand how to predict? Yes / No
  - [ ] Do they understand the leaderboard? Yes / No
  - [ ] Any confusing moments? (document)
  
- [ ] **Security spot-check**
  - [ ] Can User A access User B's private squad? No ✓
  - [ ] Can User A see predictions before event closes? No ✓
  - [ ] RLS rules enforced in Supabase? Yes ✓
  
- [ ] **Deployment ready**
  - [ ] All code committed to repo
  - [ ] No secrets/API keys in code
  - [ ] Supabase RLS policies finalized
  - [ ] Base44 app published/deployed

**GATE:** All boxes checked? ✅ → LAUNCH MVP 🚀

**Issues found?** → Fix and re-test

---

## Post-Launch Monitoring

- [ ] Monitor app for errors (check browser console)
- [ ] Collect user feedback
- [ ] Track Brier score calculations (spot-check manually)
- [ ] Note ideas for Version 2 (multi-outcome markets, etc.)

---

**Template Version:** 1.0  
**Project:** SquadBets MVP  
**Last Updated:** [Date]
