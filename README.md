# Uber SDE2 LLD Interview Prep (Python)

This folder contains **one markdown guide per classic low-level design (LLD) problem**, written in a **~1-hour interview style**: what to clarify first, how to narrate your approach, trade-offs, optimizations, and **Python reference implementations with detailed inline comments**.

Use these as study notes or upload the whole folder to GitHub for reference.

## Correct interview flow (every question — same order)

Use this **sequence** in the room; each topic file maps its sections to it.

| Step | What you do | Anti-pattern |
|------|-------------|----------------|
| **1. Open** | **Questions only**: scope, semantics, threading, ambiguous rules. **Pause** for answers. | Leading with “I’ll use a heap / tree / sorted list / buckets…” before alignment. |
| **2. Clarify** | Use the **Clarifying questions** table; add problem-specific probes. | Assuming global vs per-key, time model, or overlap rules. |
| **3. FR + NFR** | State **functional** and **non-functional** expectations in 1–2 minutes (**FR, NFR…** section). | Jumping straight to code with no requirements label. |
| **4. Invariant** | One sentence: what must **never** break. | Skipping—interviewer can’t see your safety rail. |
| **5. Entities** | 2–4 nouns, who **owns** what. | A giant class diagram with no story. |
| **6. API** | Write **public methods** on the board (signatures + returns). | Private helpers as the “API.” |
| **7. Data structures** | Optional **naive** approach, then **your** pick with **why**. | DS with no tie to operations. |
| **8. Code** | **2–3** methods that enforce the invariant. | Only boxes, no working logic. |
| **9. Edges + complexity** | Boundaries, failures, time/space. | “It works” with no Big-O. |
| **10. Scale / concurrency / tests** | If time or if they steer you there. | Buzzwords with no link to your API. |
| **11. Close** | One trade-off recap. | Stopping mid-invariant. |

Each topic file: **Interview script** (spoken checklist) → **Clarifying questions** → **FR, NFR, core entities & API** → **Approach / design / code**. Openings are **clarify-first** unless noted.

## Contents

| # | Topic | File |
|---|--------|------|
| 1 | Hit Counter | [01-design-hit-counter.md](./01-design-hit-counter.md) |
| 2 | Meeting Room Reservation | [02-meeting-room-reservation.md](./02-meeting-room-reservation.md) |
| 3 | File System (`cd` with `*`) | [03-design-file-system-wildcard-cd.md](./03-design-file-system-wildcard-cd.md) |
| 4 | Leaderboard (Top-K) | [04-design-leaderboard.md](./04-design-leaderboard.md) |
| 5 | Train Platform Management | [05-design-train-platform-management.md](./05-design-train-platform-management.md) |
| 6 | Movie Ticket Booking | [06-design-movie-ticket-booking.md](./06-design-movie-ticket-booking.md) |
| 7 | Parking Lot | [07-design-parking-lot.md](./07-design-parking-lot.md) |
| 8 | Restaurant Food Ordering | [08-design-restaurant-food-ordering.md](./08-design-restaurant-food-ordering.md) |
| 9 | Text Editor (Undo / Redo) | [09-design-text-editor-undo-redo.md](./09-design-text-editor-undo-redo.md) |
| 10 | Car Rental | [10-design-car-rental.md](./10-design-car-rental.md) |

## How to practice (60 minutes)

1. **0–10 min:** Ask clarifying questions; agree on scope (single-threaded vs concurrent, scale assumptions).
2. **10–25 min:** Public API + 3–5 core classes; state invariants (e.g., no overlapping bookings).
3. **25–45 min:** Implement **2–3 critical methods** that enforce those invariants.
4. **45–55 min:** Complexity, extensions, failure modes.
5. **Last 5 min:** Recap trade-offs.

All code is **illustrative** for interviews—adapt naming and threading to the problem statement you get.
