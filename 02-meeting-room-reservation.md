# 2. Design Meeting Room Reservation System

**Typical round:** 45–60 minutes · **Focus:** interval overlap detection, cancellation, clean APIs.

---

## Interview opener

> “I’ll keep a **fixed set of rooms**. Each room holds a **sorted list of bookings** by start time. Booking succeeds only if the new interval **doesn’t overlap** any existing one in that room. I’ll confirm interval semantics—**half-open** `[start, end)` is easiest for adjacency (`end == other.start` allowed).”

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
