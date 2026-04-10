# Uber SDE2 LLD Interview Prep (Python)

This folder contains **one markdown guide per classic low-level design (LLD) problem**, written in a **~1-hour interview style**: what to clarify first, how to narrate your approach, trade-offs, optimizations, and **Python reference implementations with detailed inline comments**.

Use these as study notes or upload the whole folder to GitHub for reference.

Each topic file now starts with **Interview script & checklist (human speaking)**—phrases you can say out loud, flow order (clarify → invariants → APIs → DS → code → edges → complexity → scale/concurrency → testing), must-code functions, a strong closing line, and a **10-second mental checklist** before you stop. The rest of each file is deeper design, trade-offs, and commented Python.

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
