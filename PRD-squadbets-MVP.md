Those are important refinements. Restricting users to **one prediction per event** is critical for the integrity of the Brier score, and having a "How it Works" tab will be essential for onboarded friends who might find the math intimidating at first.

Regarding **multi-outcome markets**: given your 2-week timeline and the complexity of building a custom UI for "allocate 100% across X options," I recommend moving that to **Version 2**. Focusing on perfect **Binary (Yes/No)** markets for the MVP ensures your "Prediction Loop" is bulletproof before adding complexity.

## Step 1: Final Verification Echo

> **Product:** **SquadBets** (MVP)
> **Updated Constraints:**
> 1. **One Shot:** Users get exactly one prediction per event (to keep Brier scores "clean").
> 2. **Editability:** Predictions can be edited as long as the market is still **OPEN**.
> 3. **Educational Tab:** A dedicated screen explaining LMSR, Brier Scores, and app features.
> 4. **Scope Control:** Custom multi-outcome markets moved to **Version 2** to ensure a 2-week launch.
> **Success Metric:** High engagement in binary markets (Yes/No) across multiple private squads.



---

## Step 2: The Technical Design Document (Part 3)

Since you are a **Vibe-coder** using **Base44**, this document is designed to be fed directly into your AI builder to set up your database and logic.

### **Technical Design: SquadBets MVP**

#### **1. Database Schema (The "Skeleton")**
Tell your AI builder to set up these specific tables:

| Table | Columns | Purpose |
| :--- | :--- | :--- |
| **Users** | `id`, `email`, `display_name`, `avatar_url`, `global_brier_avg` | Basic profile & global ranking. |
| **Squads** | `id`, `name`, `invite_code`, `creator_id`, `created_at` | Private group containers. |
| **Squad_Members**| `squad_id`, `user_id`, `squad_brier_score` | Links users to squads; tracks local ranking. |
| **Events** | `id`, `squad_id`, `title`, `description`, `status` (Open/Closed), `current_prob`, `resolution` (T/F) | The "Bet." `current_prob` starts at 50%. |
| **Predictions** | `id`, `event_id`, `user_id`, `predicted_prob` (0.0-1.0), `timestamp` | One row per user per event. Overwrite on edit. |

#### **2. The Math Logic (The "Brain")**
Copy these formulas for your AI to implement in the backend logic:

* **LMSR (Group Probability):** `New_Prob = (Exp(Total_Yes / b)) / (Exp(Total_Yes / b) + Exp(Total_No / b))`
    *(Note: For MVP, a simplified "Weighted Average of Predictions" is often more stable for small friend groups than a true LMSR liquidity pool—ask the AI to recommend the best "Group Signal" logic for 5-10 people).*
* **Brier Score (The Ranking):**
    `Score = (Prediction - Outcome)^2`
    *(Example: If I predict 80% (0.8) and it happens (1.0), my score is (0.8 - 1.0)² = 0.04. **Lower is better.**)*

#### **3. Privacy Rules (The "Shield")**
- **Row-Level Security:** A user can *only* see `Events` and `Predictions` if their `user_id` exists in the `Squad_Members` table for that `squad_id`.
- **The "Reveal" Logic:** If `Event.status == 'Open'`, query returns only `Count(Predictions)` and `Average(predicted_prob)`. If `Event.status == 'Closed'`, query returns all `Predictions` with `user_id` attached.

---

## **Part 4: The Build Prompts (Your "Vibe-Code" Toolbox)**

When you open **Base44**, use these prompts to start building:

**Prompt 1 (Setup):**
> "I want to build a mobile-first app called SquadBets. Start by creating a Google Auth login and a 'Create/Join Squad' screen. Squads must be private and accessible only via an invite code. Users should have a simple profile with a display name."

**Prompt 2 (The Event Loop):**
> "Create a screen for a 'Squad Feed.' Members can create 'Events' (Yes/No questions). When an event is created, show a 0-100% slider. A user can submit one prediction. If they move the slider again, it updates their existing prediction. While the event is 'Open,' hide individual names and only show the group's average probability."

**Prompt 3 (The Reveal & Math):**
> "Add a 'Resolve Event' button for the creator. When an event is marked 'Yes' (1) or 'No' (0), calculate a Brier Score for every user who predicted: (Prediction - Outcome)^2. Update a Squad Leaderboard where users are ranked by their average Brier Score (lowest score at the top)."

---

**Session continuity reminder:** Save a short summary of this logic (especially the Brier formula and Privacy rules) and reuse it if you start a new chat in Base44.

**I’ve given you the research, the PRD, and the Technical Prompts. Is there anything else you need to start your 2-week sprint for SquadBets?**
