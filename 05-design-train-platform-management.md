# 5. Design Train Platform Management System

**Typical round:** 45–60 minutes · **Focus:** exclusive resource assignment over time, query “train at platform at time t”.

---

## Interview script & checklist (human speaking)

### Opening

“This is structurally like **room booking**, but the resource is a **platform** and the thing scheduled is a **train**. I’ll confirm: **only one train per platform at a time**, intervals **half-open** unless you say otherwise, and whether the same train can appear on **two platforms**—if yes, I’d add a **second invariant** on the train side. Core invariant: **no overlapping assignments on the same platform**.”

### Flow — cover in this order

1. **Clarify** — time model, reschedule rules, query types (point-in-time vs range), concurrency.  
2. **Invariants** — non-overlapping intervals per platform; valid `start < end`.  
3. **Entities** — station/service, platform → sorted assignments `(start, end, train_id)`.  
4. **APIs** — `assign(train, platform, start, end) -> bool`, `query_train_at_platform(platform, t) -> optional train`, optional `cancel`.  
5. **Data structures** — sorted intervals + **bisect** for insert and point query.  
6. **Code** — **`assign`** (overlap) and **`query_train_at_platform`**.  
7. **Edge cases** — zero length, boundary `t`, unknown platform.  
8. **Complexity** — assign O(n) list insert; query O(log n) locate + O(1) check.  
9. **Scale** — few platforms per station; persistence + audit; optional “train can’t be two places” index.  
10. **Testing** — adjacent intervals, query inside vs outside.

### Natural phrases

- “I’ll reuse the **two-neighbor overlap check** from meeting rooms—it’s the same math.”  
- “For ‘**who is at platform P at time t**’, I need the **last interval with start ≤ t** and then check **t < end** for half-open.”

### APIs on the board

`assign` · `query_train_at_platform` · optional `cancel_assignment`.

### Must-code (2–3)

1. **`assign`**  
2. **`query_train_at_platform`**  
3. Optional **`cancel`** if they care about lifecycle.

### Closing

“For **extensibility**, I’d introduce `TimeRange` and `Assignment` types and keep persistence behind an interface—same domain rules, swappable storage.”

### Mental checklist

Same as meeting rooms mentally · Platform exclusivity · Point query · Code · Edges · Complexity.

---

## Interview opener

> “A **platform** can host **at most one train** at a time. Trains have **arrival and departure** times. I’ll model assignments as **intervals per platform** (like meeting rooms) and support queries: which train is at platform P at time T, and optionally list assignments in a window. I’ll confirm whether times are **half-open** so back-to-back trains are allowed.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Train IDs unique? | Keys |
| Can same train **move** platforms? | Update vs new assignment |
| **Overlap** definition at exact boundaries? | `[arr, dep)` vs closed |
| Need **conflict detection** if double-book platform? | Core invariant |
| Query types: point-in-time, range, “free platforms at t”? | Index design |

---

## Approach

- **`Platform`:** sorted list of assignments `(start, end, train_id)` non-overlapping.
- **`assign(train_id, platform_id, start, end)`:** same overlap check as meeting room.
- **`query_platform_at(platform_id, t)`:** binary search for interval containing `t` (if `[start,end)` then `start <= t < end`).

**Extensibility:** introduce `Schedule` interface; later swap in DB or interval tree.

---

## Complexity

- Assign: O(n) list insert in interview implementation.
- Query: O(log n) locate + O(1) check neighbor.

---

## Reference implementation (Python)

```python
"""
Train platform manager: one train at a time per platform.

Assignments are half-open intervals [start, end) on an integer or comparable timeline.
This mirrors the meeting-room pattern: sorted by start, check neighbors on insert.
"""

from __future__ import annotations

import bisect
from dataclasses import dataclass
from threading import Lock
from typing import Optional


@dataclass(frozen=True)
class Assignment:
    train_id: str
    platform_id: str
    start: int
    end: int


class TrainPlatformManager:
    def __init__(self, platform_ids: list[str]) -> None:
        self._platforms: dict[str, list[tuple[int, int, str]]] = {
            p: [] for p in platform_ids
        }
        self._lock = Lock()

    def assign(
        self, train_id: str, platform_id: str, start: int, end: int
    ) -> bool:
        """
        Schedule train on platform from start to end (half-open).

        Returns False if platform unknown, invalid range, or overlap.
        """
        if end <= start:
            return False
        with self._lock:
            if platform_id not in self._platforms:
                return False
            iv = self._platforms[platform_id]
            pos = bisect.bisect_left(iv, (start, -10**18, ""))

            # Overlap with next?
            if pos < len(iv):
                ns, ne, _ = iv[pos]
                if start < ne and end > ns:
                    return False
            # Overlap with previous?
            if pos > 0:
                ps, pe, _ = iv[pos - 1]
                if start < pe and end > ps:
                    return False

            bisect.insort(iv, (start, end, train_id))
            return True

    def query_train_at_platform(self, platform_id: str, t: int) -> Optional[str]:
        """
        Return train_id occupying platform at time t, or None if idle.

        Strategy:
        - Find rightmost interval with start <= t using bisect_right - 1.
        - Check if t < end for half-open containment.
        """
        with self._lock:
            if platform_id not in self._platforms:
                return None
            iv = self._platforms[platform_id]
            pos = bisect.bisect_right(iv, (t, 10**18, "")) - 1
            if pos < 0:
                return None
            s, e, tid = iv[pos]
            if s <= t < e:
                return tid
            return None

    def cancel_assignment(self, platform_id: str, train_id: str, start: int) -> bool:
        """
        Optional: cancel by (platform, train, start) — identifies unique slot if no duplicates.
        """
        with self._lock:
            if platform_id not in self._platforms:
                return False
            iv = self._platforms[platform_id]
            for i, (s, e, tid) in enumerate(iv):
                if tid == train_id and s == start:
                    iv.pop(i)
                    return True
            return False
```

---

## Extensions

- **Delay / reschedule** — remove + re-insert with same checks.
- **Train cannot be two places** — secondary index `train_id → active assignments` and validate.
- **Visualization** — merge intervals export.

---

## OOP story for “clean, extensible”

- `TimeRange`, `Train`, `Platform`, `Station` (holds platforms), `AssignmentService` (rules).  
For 1 hour, **one class** with clear methods is often enough if you **name** the extension points.
