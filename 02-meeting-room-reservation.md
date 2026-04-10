# 2. Design Meeting Room Reservation System

**Typical round:** 45–60 minutes · **Focus:** interval overlap detection, cancellation, clean APIs.

---

## Correct interview flow (this document)

Same order as **README** (master table): **questions → clarifying table → FR/NFR → invariant → entities → API → data structures → code → edges/complexity → scale/tests → close**.  
Do **not** open with “sorted list + bisect” until **after** you’ve stated **FR/NFR** and **API** (sections below).

---

## Interview script & checklist (human speaking)

### Opening (clarify-first — first 30–45 seconds)

“Before I pick a structure: is the **room list fixed** up front? How should we represent **time** (epoch, minutes, dates)? For **overlap**, I’d like **half-open** `[start, end)` so back-to-back slots don’t conflict—is that OK, or do you want **closed** intervals? How does **cancel** work—by **`booking_id`** only? **Who generates** the booking id? Any need to **list** bookings per room? **Multi-threaded**?”

**Pause.** Then one line of intent (still **no** sorted list yet):

“Once aligned, my **invariant** is: **no two bookings in the same room overlap in time**. Next I’ll state **FR/NFR**, put **book/cancel** on the board, **then** choose how to store intervals.”

### Flow — cover in this order

1. **Open / clarify** — as above; use **Clarifying questions** table.  
2. **FR / NFR** — **FR, NFR, core entities & API** section (say aloud, 1–2 min).  
3. **Invariant** — no overlap per room; reject invalid ranges.  
4. **Entities** — scheduler + per-room schedule (on board **after** API names).  
5. **APIs on the board** — `book`, `cancel` (signatures **before** internal DS).  
6. **Data structures** — **now** justify: naive scan vs **sorted by start** + neighbor check + `bisect`; optional `booking_id → room`.  
7. **Code** — **`book`**, **`cancel`**.  
8. **Edge cases** — `end <= start`, duplicate id, boundary touch with half-open.  
9. **Complexity** — find O(log n), list insert O(n)—honest; tree if they need strict log n.  
10. **Scale / concurrency / testing** — room shard, DB constraint, locks; adjacent intervals in tests.

### Natural phrases

- “Half-open intervals make **adjacency** easy: `[10,20)` and `[20,30)` don’t overlap.”  
- “I only need to check the **interval before and after** the insertion point.”  
- “If cancel is hot, I’d add a **`booking_id → room`** map so I don’t scan every slot.”

### APIs on the board

`book(...) -> bool` · `cancel(...) -> bool` · optional `list_bookings(room_id)`.

### Must-code (2–3)

1. **`book`** — validate range, neighbor overlap, sorted insert.  
2. **`cancel`** — find by id, remove.  
3. Optional: **`_overlaps_neighbors`** extracted.

### Closing

“This enforces **exclusive use of a room** over time; in production I’d persist with a **uniqueness constraint** on non-overlapping ranges and handle **retries** with idempotent `booking_id`.”

### Mental checklist

Clarified · FR/NFR · Invariant · Entities · APIs **before** DS · DS + `book` + `cancel` · Edges · Complexity · Concurrency/DB · Tests.

---

## After alignment (optional — you may say this before DS)

> “I’ll keep bookings **per room** in **start order** so overlap checks only need **neighbors** at insert time—I'll use binary search to find the slot.”

(Only **after** steps 1–5 above; this is **not** your first sentence.)

---

## Clarifying questions

| Question | Why |
|----------|-----|
| **Time units** — minutes since midnight, epoch, datetimes? | Comparison and tests |
| **Overlap rule** — touch at boundary allowed? | Defines `end <= other_start` OK |
| **IDs** — who generates `booking_id`? | Return value vs input |
| **Cancel** — by id only, or by interval? | Lookup structure |
| **List bookings** for a room? | Extra API |
| **Concurrency** | Lock per room vs global |

---

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** naming these after clarification; then data structures and code.

### Functional requirements (FR)

- **Book** a time range for a given **room** if it does not **overlap** existing bookings in that room.
- **Cancel** an existing booking (by **booking_id** and room, or as agreed).
- (Optional if in scope) **List** bookings for a room.

**What you can say:** “**FRs**: book and cancel with **no overlap per room**; I’ll confirm list/cancel shape.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | No double-book same room/time | “**Invariant**: intervals for one room never overlap.” |
| **Performance** | Fast book/cancel for typical n per room | “I’ll use **sorted intervals** and only check **neighbors**.” |
| **Concurrency** | Safe parallel books | “**Per-room lock** or start single-threaded.” |
| **Consistency** | `booking_id` unique / idempotent retries | “Same `booking_id` retry shouldn’t double-insert.” |

**What you can say:** “**NFRs**: correctness first, predictable book/cancel, thread safety if needed.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`MeetingRoomScheduler`** (or `BookingService`) | Known rooms; `book` / `cancel`; owns per-room state. |
| **Per-room schedule** | Sorted list of `(start, end, booking_id)`—no separate `Booking` class required in interview code. |

**Relationships:** Scheduler **owns** many rooms; each room **owns** a non-overlapping set of intervals.

**What you can say:** “**Entities**: one **scheduler**, each **room** holds an ordered list of bookings.”

### API design

| Method | Purpose |
|--------|---------|
| `book(room_id, start, end, booking_id) -> bool` | Success only if no overlap and valid range. |
| `cancel(room_id, booking_id) -> bool` | Remove if present. |
| Optional `list_bookings(room_id)` | Read model / debug. |

**What you can say:** “**Public API** stays small: **book** and **cancel**; overlap is enforced inside.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code → complexity** (see **README** master table).

---

## Approach

- **Per room:** maintain `list[tuple[start, end, booking_id]]` **sorted by start**.
- **Book:** binary search for insertion point; check **previous** interval’s `end > start` and **next** interval’s `start < end`.
- **Cancel:** linear scan by `booking_id` (O(n) per room); optimize with side `dict[booking_id → room]` if they ask.

**Why not a tree map only?** Sorted list + `bisect` is standard in Python interviews; `sortedcontainers` is usually not allowed.

---

## Complexity

- **Book:** O(n) worst-case insert in list; O(log n) locate + O(n) shift — say honestly “list is O(n) insert”; if they need O(log n) updates, discuss **balanced BST** or **interval tree** (overkill unless huge n).

---

## Optimizations

- **`booking_id → (room, start)`** map for O(1) cancel locate, then remove from list.
- **Timeline index** if queries are “all meetings today” (separate index).

---

## Reference implementation (Python)

```python
"""
Meeting room scheduler: no overlapping bookings per room.

Key interview choices:
- Intervals are half-open: [start, end) — so [1,3) and [3,5) do NOT overlap.
- Each room keeps a list sorted by start time for easy neighbor checks.
- bisect.insort maintains order; insert is O(n) due to list shifting (call out in interview).
"""

from __future__ import annotations

import bisect
from dataclasses import dataclass
from threading import Lock
from typing import Optional


@dataclass(frozen=True)
class Booking:
    """Immutable record returned to callers / used in logs."""

    booking_id: str
    room_id: str
    start: int
    end: int


class MeetingRoomScheduler:
    """
    rooms[room_id] -> list of (start, end, booking_id), sorted by start.
    """

    def __init__(self, room_ids: list[str]) -> None:
        # Fixed universe of rooms from problem statement
        self._rooms: dict[str, list[tuple[int, int, str]]] = {r: [] for r in room_ids}
        self._lock = Lock()

    def _overlaps_neighbors(
        self, intervals: list[tuple[int, int, str]], start: int, end: int
    ) -> bool:
        """
        `intervals` is sorted by start. Find where `start` would be inserted.

        Overlap with previous: prev.end > start (half-open: touching at point is OK)
        Overlap with next: next.start < end
        """
        # Tuple ordering compares start first — sufficient for bisect key here
        pos = bisect.bisect_left(intervals, (start, -10**18, ""))

        if pos < len(intervals):
            ns, ne, _ = intervals[pos]
            # If new interval ends after next starts, they intersect
            if start < ne and end > ns:
                return True

        if pos > 0:
            ps, pe, _ = intervals[pos - 1]
            if start < pe and end > ps:
                return True

        return False

    def book(self, room_id: str, start: int, end: int, booking_id: str) -> bool:
        """
        Try to add [start, end). Returns False if invalid room, bad range, or overlap.

        Implementation note:
        - Validate end > start first (common edge case).
        - Thread-safe: one global lock for interview simplicity; production = per-room locks.
        """
        if end <= start:
            return False

        with self._lock:
            if room_id not in self._rooms:
                return False
            intervals = self._rooms[room_id]

            if self._overlaps_neighbors(intervals, start, end):
                return False

            # Insert keeping sort order by start
            bisect.insort(intervals, (start, end, booking_id))
            return True

    def cancel(self, room_id: str, booking_id: str) -> bool:
        """Remove first booking with matching id in that room. O(n) scan."""
        with self._lock:
            if room_id not in self._rooms:
                return False
            intervals = self._rooms[room_id]
            for i, t in enumerate(intervals):
                _, _, bid = t
                if bid == booking_id:
                    intervals.pop(i)
                    return True
            return False

    def list_bookings(self, room_id: str) -> list[Booking]:
        """Optional API: snapshot for debugging / UI."""
        with self._lock:
            if room_id not in self._rooms:
                return []
            return [
                Booking(booking_id=bid, room_id=room_id, start=s, end=e)
                for s, e, bid in self._rooms[room_id]
            ]
```

---

## What to say about overlap check

Walk through example on the board: intervals `[10,20)`, `[30,40)`. Booking `[20,30)` — with half-open, **no overlap**. Booking `[19,31)` — overlaps both.

---

## Extensions

- **Recurring meetings** — expand to instances or store RRULE + materialize.
- **Find free slot** — scan merged gaps.
- **Multi-room booking** — two-phase commit or book all-or-nothing (harder).
