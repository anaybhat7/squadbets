# Testing Strategy

## Overview

Testing is critical for SquadBets because the **math must be exact**. A Brier calculation error that goes unnoticed could invalidate the entire leaderboard. This guide ensures every feature works and every formula is verified before launch.

---

## Testing Pyramid

```
Top:     Manual Testing (you)
        E2E Tests (Base44 + Supabase)
Mid:    Unit Tests (Edge Functions)
Bottom: Database Tests (SQL)
```

---

## Unit Tests (Edge Functions)

### Test: LMSR / Group Probability Calculation

**Tool:** Node.js test framework (Jest recommended)

```javascript
// functions/__tests__/calculateGroupProb.test.ts
import { calculateGroupProb } from '../calculateGroupProb';

describe('calculateGroupProb', () => {
  test('Average of [0.6, 0.7, 0.5] should be 0.6333', async () => {
    const eventId = 'test-event-1';
    // Mock predictions: 0.6, 0.7, 0.5
    const result = await calculateGroupProb(eventId);
    expect(result).toBeCloseTo(0.6333, 2);
  });

  test('Single prediction: should return that prediction', async () => {
    const eventId = 'test-event-2';
    // Mock 1 prediction: 0.8
    const result = await calculateGroupProb(eventId);
    expect(result).toBeCloseTo(0.8, 4);
  });

  test('No predictions: should default to 0.5', async () => {
    const eventId = 'test-event-3';
    const result = await calculateGroupProb(eventId);
    expect(result).toBe(0.5);
  });
});
```

**Run:**
```bash
npm test -- calculateGroupProb.test.ts
```

---

### Test: Brier Score Calculation

**Tool:** Jest

```javascript
// functions/__tests__/calculateBrierScores.test.ts
import { calculateBrierScores } from '../calculateBrierScores';

describe('calculateBrierScores', () => {
  test('Predict 0.8, outcome Yes (1): score = 0.04', async () => {
    const eventId = 'test-event-4';
    // Mock: user predicted 0.8, event outcome is 1
    await calculateBrierScores(eventId);
    const scores = await getSavedScores(eventId);
    expect(scores[0]).toBeCloseTo(0.04, 4);
  });

  test('Predict 0.3, outcome No (0): score = 0.09', async () => {
    const eventId = 'test-event-5';
    // Mock: user predicted 0.3, event outcome is 0
    await calculateBrierScores(eventId);
    const scores = await getSavedScores(eventId);
    expect(scores[0]).toBeCloseTo(0.09, 4);
  });

  test('Predict 1.0, outcome Yes (1): score = 0 (perfect)', async () => {
    const eventId = 'test-event-6';
    await calculateBrierScores(eventId);
    const scores = await getSavedScores(eventId);
    expect(scores[0]).toBe(0);
  });

  test('Predict 0.0, outcome No (0): score = 0 (perfect)', async () => {
    const eventId = 'test-event-7';
    await calculateBrierScores(eventId);
    const scores = await getSavedScores(eventId);
    expect(scores[0]).toBe(0);
  });

  test('Average Brier across 3 events', async () => {
    // Predictions: 0.04, 0.09, 0.25 for 3 events
    // Average should be (0.04 + 0.09 + 0.25) / 3 = 0.1267
    const avgScore = (0.04 + 0.09 + 0.25) / 3;
    expect(avgScore).toBeCloseTo(0.1267, 4);
  });
});
```

---

## SQL / Database Tests

### Test: Row-Level Security (RLS)

**Tool:** Supabase query editor (manual, then automate if scaling)

```sql
-- Scenario 1: User A tries to see User B's squad
SELECT * FROM events WHERE squad_id = 'squad-xyz';
-- Result: Should return ZERO events if User A is not member of squad-xyz
-- Expected: []

-- Scenario 2: User A queries after joining squad
INSERT INTO squad_members (squad_id, user_id) VALUES ('squad-xyz', 'user-a');
SELECT * FROM events WHERE squad_id = 'squad-xyz';
-- Result: Should now return events if squad exists
-- Expected: [event1, event2, ...]

-- Scenario 3: Prediction visibility (event OPEN)
SELECT * FROM predictions WHERE event_id = 'event-123';
-- Result: Should return only AGGREGATE (count, average), not individual predictions
-- Expected if event.status = 'OPEN': hidden (or error)
-- Expected if event.status = 'CLOSED': all predictions visible

-- Scenario 4: Prediction visibility (event CLOSED)
UPDATE events SET status = 'CLOSED' WHERE id = 'event-123';
SELECT * FROM predictions WHERE event_id = 'event-123';
-- Result: Should now return all predictions with user names
-- Expected: [user-a: 0.6, user-b: 0.75, ...]
```

### Test: Leaderboard Calculation

```sql
-- Manual calculation of squad leaderboard
SELECT 
  u.display_name,
  AVG(POWER((p.predicted_prob - e.final_outcome), 2)) as avg_brier_score
FROM predictions p
JOIN events e ON p.event_id = e.id
JOIN squad_members sm ON sm.user_id = p.user_id AND sm.squad_id = e.squad_id
JOIN profiles u ON p.user_id = u.id
WHERE e.squad_id = 'squad-abc'
GROUP BY u.display_name
ORDER BY avg_brier_score ASC;

-- Expected output:
-- Alice | 0.1200
-- Bob   | 0.2500
-- You   | 0.3400
```

---

## E2E Tests (Base44 + Browser)

### Test: Full User Journey (Manual First, Automate Later)

**Tool:** Browser + Playwright (or Puppeteer)

```javascript
// e2e/squabbets.spec.ts
import { test, expect } from '@playwright/test';

test.describe('SquadBets E2E', () => {
  
  test('User 1 creates squad, User 2 joins, both predict', async ({ browser }) => {
    // User 1: Create squad
    const user1 = await browser.newContext();
    const page1 = await user1.newPage();
    await page1.goto('https://squabbets.vercel.app');
    
    // Google login (mock or use test account)
    await page1.click('text=Login with Google');
    await page1.fill('input[type="email"]', 'user1@test.com');
    await page1.click('text=Next');
    // ... complete OAuth flow
    
    // Create squad
    await page1.click('text=Create Squad');
    await page1.fill('input[name="name"]', 'Test Squad');
    const inviteCode = await page1.locator('.invite-code').textContent();
    expect(inviteCode).toMatch(/^[A-Z0-9]{6}$/);
    
    // User 2: Join squad
    const user2 = await browser.newContext();
    const page2 = await user2.newPage();
    await page2.goto('https://squabbets.vercel.app');
    // ... login as user2
    await page2.click('text=Join Squad');
    await page2.fill('input[name="invite"]', inviteCode);
    await page2.click('text=Join');
    
    // User 1: Create event
    await page1.click('text=Create Event');
    await page1.fill('input[name="title"]', 'Will it rain?');
    await page1.click('text=Create');
    
    // Both predict
    await page1.locator('.slider').fill('75'); // User 1 predicts 75%
    await page1.click('text=Submit');
    
    await page2.locator('.slider').fill('60'); // User 2 predicts 60%
    await page2.click('text=Submit');
    
    // Check group average
    const avgText = await page1.locator('.group-average').textContent();
    expect(avgText).toContain('67.5%'); // (75 + 60) / 2
  });
  
  test('Event resolves, Brier scores calculated', async ({ browser }) => {
    // ... setup event with predictions
    
    // Resolve event
    await page.click('text=Resolve');
    await page.click('text=Yes');
    
    // Check Brier scores
    const predictions = await page.locator('.prediction-item').all();
    expect(predictions[0]).toContain('User 1: 75% | Score: 0.06');
  });
});
```

**Run:**
```bash
npm run test:e2e
```

---

## Manual Testing Checklist

### Phase 1: Skeleton

- [ ] **Login**
  - [ ] Click "Login with Google"
  - [ ] Redirects to Google OAuth
  - [ ] Can sign in with test account
  - [ ] Redirects back to app
  - [ ] Display name visible in header

- [ ] **Create Squad**
  - [ ] Click "Create Squad" button
  - [ ] Modal opens with name field
  - [ ] Type "Test Squad"
  - [ ] Click "Create"
  - [ ] Invite code appears (6 alphanumeric chars)
  - [ ] Copy button works
  - [ ] Squad appears in "My Squads" list

- [ ] **Join Squad**
  - [ ] Open new browser, log in as User 2
  - [ ] Click "Join Squad" button
  - [ ] Modal opens with invite code field
  - [ ] Paste invite code
  - [ ] Click "Join"
  - [ ] Squad now visible in User 2's squad list

- [ ] **Mobile Responsive**
  - [ ] Open app on iPhone 12 (or emulator)
  - [ ] Portrait mode: all buttons clickable
  - [ ] Text readable (not tiny)
  - [ ] No horizontal scroll
  - [ ] Forms are mobile-friendly (big input fields)

### Phase 2: Predictions

- [ ] **Create Event**
  - [ ] In squad, click "Create Event"
  - [ ] Modal opens with title + description
  - [ ] Type "Will the Sixers beat the Celtics?"
  - [ ] Click "Create"
  - [ ] Event appears in squad feed with "OPEN" status
  - [ ] Event shows current_prob = 50% (default)

- [ ] **Predict (One per User)**
  - [ ] Click event
  - [ ] Move slider to 75%
  - [ ] Click "Predict"
  - [ ] Your prediction shows on screen
  - [ ] Try to move slider again → moves your existing prediction (edit, not new)
  - [ ] Submit again → no duplicate created

- [ ] **Group Probability Updates**
  - [ ] As User 1 predicts 75%, see group_prob updates
  - [ ] Switch to User 2's browser, predict 60%
  - [ ] Refresh User 1's page → sees group_prob = 67.5%
  - [ ] Matches manual calc: (75 + 60) / 2 = 67.5 ✓

- [ ] **Individual Predictions Hidden (While OPEN)**
  - [ ] As User 1, you see your own prediction (75%)
  - [ ] You see "Group: 67.5%"
  - [ ] You do NOT see "User 2: 60%"
  - [ ] Only the count: "2 members predicted"

### Phase 3: Leaderboard

- [ ] **Resolve Event**
  - [ ] User 1 clicks "Resolve Event"
  - [ ] Modal asks "Did it happen: Yes / No"
  - [ ] Select "No" (Celtics won)
  - [ ] Status changes to "CLOSED"
  - [ ] Brier scores calculated and displayed

- [ ] **Brier Scores Correct**
  - [ ] User 1 (predicted 75%, outcome 0): score = (0.75 - 0)² = 0.5625 ✓
  - [ ] User 2 (predicted 60%, outcome 0): score = (0.60 - 0)² = 0.36 ✓
  - [ ] User 2 is ranked higher (lower score)

- [ ] **Leaderboard Displays**
  - [ ] Click "Leaderboard" tab
  - [ ] User 2 at top (0.36 avg)
  - [ ] User 1 below (0.5625 avg)
  - [ ] Clicking names shows prediction history (if implemented)

- [ ] **Predictions Revealed**
  - [ ] After event closes, see all predictions:
    - "User 1: 75% | Score: 0.5625"
    - "User 2: 60% | Score: 0.36"

---

## Pre-Commit Hooks (Optional, Recommended)

Set up Git hooks to auto-run tests before committing:

```bash
# .husky/pre-commit
#!/bin/sh
npm run test:unit
npm run lint
npm run format
```

Install:
```bash
npx husky install
npx husky add .husky/pre-commit "npm test"
```

---

## Verification Loop (Daily Check)

**After each feature pushed to staging:**

1. [ ] Run `npm test:unit` → All tests pass
2. [ ] Run `npm test:e2e` → All scenarios pass
3. [ ] Manual smoke test (fresh login + create event)
4. [ ] Mobile test (open on phone, click through)
5. [ ] Database spot-check (run manual SQL queries above)

If any step fails:
- Document the issue
- Fix the code
- Re-run tests
- Only then merge to main

---

## Budget for Testing

- Jest/Playwright setup: One-time setup (~1 hour)
- Unit tests: ~2 hours to write all 10+ tests
- E2E tests: ~3 hours to automate
- Manual testing: Daily (~15 min)

**Total effort:** ~6 hours upfront, then 15 min/day during build

---

## References

- **Jest Docs:** jestjs.io
- **Playwright Docs:** playwright.dev
- **Supabase Testing:** supabase.com/docs/guides/testing
- **Math verification:** Hand-calc examples in this document

---

**Testing Status:** Ready for Phase 1  
**Last Updated:** Today  
**Owner:** You (math verification) + AI Builder (code tests)
