Technical Design Document

# TechDesign-SquadBets-MVP.md

## 1. System Architecture

To keep costs at $0 and ensure the math is "perfect," we will use a **Serverless Managed Stack**. This avoids the "wrong tech choice" trap by using industry-standard tools that the AI can easily code for.

| Component | Recommendation | Why? | Cost |
| :--- | :--- | :--- | :--- |
| **Builder/Frontend** | **Base44 (or FlutterFlow)** | Best for "AI-written" mobile apps that need custom logic. | Free Tier |
| **Backend/Auth** | **Supabase** | Industrial-strength PostgreSQL. Handles private squads perfectly. | Free Tier |
| **Math Engine** | **Edge Functions** | Small scripts that run the LMSR and Brier math instantly. | Free Tier |
| **Deployment** | **Vercel / PWA** | Quickest way to get an "app" on a phone without App Store delays. | Free |

## 2. The "Math-First" Implementation

Since you are an ORIE student, we won't let the AI guess the formulas. We will "hard-code" these logic blocks into the prompt instructions:

### A. The LMSR (Group Signal)
Instead of a simple average, we use the **Logarithmic Market Scoring Rule**. This ensures that as people move the slider, the "Squad Probability" reacts dynamically.
* **Formula:** $P_i = \frac{e^{q_i/b}}{\sum e^{q_j/b}}$
* *Implementation:* The AI will create a function `calculateNewProb` that updates the global squad signal every time a user saves their slider position.

### B. The Brier Score (Accuracy)
This is your "Proof of Intelligence" metric. 
* **Formula:** $BS = \frac{1}{N} \sum (f_t - o_t)^2$
* *Implementation:* When an event is resolved ($1$ for Yes, $0$ for No), the system calculates $(prediction - outcome)^2$. A score of $0$ is a "Perfect God," $1$ is "Perfectly Wrong."



## 3. Database Schema (The "Source of Truth")

The AI needs a clear map to prevent bugs. We will instruct the AI to build these tables:

* **`profiles`**: Stores `user_id`, `display_name`, and `total_brier_points`.
* **`squads`**: Stores `squad_name` and a `join_code`. 
* **`squad_members`**: Links users to squads (Private visibility check).
* **`events`**: The actual bet (e.g., "Will the speech be > 5 mins?"). Stores `status` (Open/Closed) and `final_outcome`.
* **`predictions`**: Stores one row per `user_id` per `event_id` with the `probability_value` (0.0 to 1.0).

## 4. Privacy & Security (Squad Isolation)

To address your "wrong tech choice" worry, we use **Row Level Security (RLS)**. 
* **Rule:** A user can only `SELECT` from the `events` table if their `user_id` is present in the `squad_members` list for that event's `squad_id`. 
* **The Reveal:** Individual `predictions` remain hidden to others until the `event.status` changes to 'Closed'. This prevents "herd bias."

## 5. 30-Day Build Roadmap

| Phase | Focus | Goal |
| :--- | :--- | :--- |
| **Week 1** | **The Skeleton** | Setup Supabase + Auth + Join Squad via code. |
| **Week 2** | **The Engine** | Implement the 0-100% Slider + LMSR update logic. |
| **Week 3** | **The Result** | Event Resolution + Brier Score calculation + Leaderboard. |
| **Week 4** | **The Vibe** | Add "Vibe Memos" + "How it Works" tab + Mobile Testing. |

## 6. AI Assistance Strategy

To prevent the "lacking direction" bug you felt before, use this specific prompting pattern in Base44:

1.  **"Context First":** Always start a session by telling the AI: *"I am building SquadBets, a private prediction market. We are using Brier scores $(p-o)^2$ for accuracy. Do not guess the math."*
2.  **"One Feature at a Time":** Don't ask for the whole app. Ask for: *"Build the logic for the private join code today."*
3.  **"Explain the Code":** Ask: *"Explain how the Brier score is being calculated in the database trigger so I can verify it matches my ORIE project requirements."*

---

### **Critical Review Questions**
1.  **Budget Check:** This stack is 100% free for up to ~500 users. No surprise bills.
2.  **Scalability:** Supabase/PostgreSQL is what "Pro" apps use. You won't have to rebuild.

**Does this technical path feel right to you, or should we simplify the LMSR math for the MVP?**
