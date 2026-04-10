# 5. Design Train Platform Management System

**Typical round:** 45–60 minutes · **Focus:** exclusive resource assignment over time, query “train at platform at time t”.

---

## Correct interview flow (this document)

Same order as **README**. Same cadence as **meeting rooms**: **clarify time + overlap + queries** → **FR/NFR → API** → **then** sorted intervals / bisect.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“Is it **one train per platform at a time**? How do we represent **time**, and are intervals **half-open** OK for back-to-back assignments? Can the **same train** be scheduled on **two platforms** at once—if yes I need a rule. What **queries** matter—**point in time** only, or ranges? **Cancel/reschedule**? **Concurrency**?”

**Pause.** Then:

“**Invariant**: **no overlapping assignments on the same platform**. I’ll do **FR/NFR**, put **assign** and **query** on the board, **then** use the same **interval** idea as room booking.”

### Flow — cover in this order

1. **Open / clarify** — as above + table.  
2. **FR / NFR** — section below.  
3. **Invariant** — exclusive platform use over time; valid ranges.  
4. **Entities** — manager + per-platform timeline **after** API.  
5. **APIs on board** — `assign`, `query_train_at_platform`, optional `cancel`.  
6. **Data structures** — sorted intervals + **bisect** (say **after** API).  
7. **Code** — **`assign`**, **`query_train_at_platform`**.  
8. **Edge cases** — zero length, boundary `t`, unknown platform.  
9. **Complexity** — assign insert cost; query O(log n).  
10. **Scale / tests** — audit; adjacent intervals.

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

Clarified · FR/NFR · APIs before intervals · `assign` + query · Edges · Complexity.

---

## After alignment

> “Per platform I’ll keep **non-overlapping intervals sorted by start** and use **binary search** for insert and for ‘who is at P at time t’.”

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

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** naming these; same shape as **meeting rooms**, different nouns.

### Functional requirements (FR)

- **Assign** a **train** to a **platform** for a time range if the platform has **no overlap**.
- **Query** which train (if any) occupies a platform at **time t**.
- (Optional) **Cancel** / reschedule assignment.

**What you can say:** “**FRs**: assign train to platform without overlap; **point-in-time query** at minimum.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | At most one train per platform per instant | “Same **non-overlap invariant** as room booking.” |
| **Performance** | Fast assign + query for modest n per platform | “**Sorted intervals** + **bisect**.” |
| **Audit / ops** | Who changed what (if they ask) | “Out of core LLD unless required.” |

**What you can say:** “**NFRs**: scheduling correctness, efficient lookup, extensible to persistence.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`TrainPlatformManager`** | Fixed platforms; `assign`, `query`, optional `cancel`. |
| **Per-platform timeline** | Sorted `(start, end, train_id)` intervals. |
| **`Train`** | Often just **train_id**; optional value object `Assignment`. |

**Relationships:** Manager **owns** platforms; each platform **owns** its interval list.

**What you can say:** “**Entities**: **station service** + **per-platform schedule**; train as id.”

### API design

| Method | Purpose |
|--------|---------|
| `assign(train_id, platform_id, start, end) -> bool` | False on conflict or bad input. |
| `query_train_at_platform(platform_id, t) -> Optional[str]` | Train id or none. |
| Optional `cancel_assignment(...)` | Lifecycle. |

**What you can say:** “**API**: **assign** and **query** are the core; cancel if in scope.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code → complexity** (see **README**).

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
