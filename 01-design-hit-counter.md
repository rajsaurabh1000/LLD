# 1. Design Hit Counter

**Typical round:** 45–60 minutes · **Focus:** time-window aggregation, per-key state, optional concurrency.

---

## Interview script & checklist (human speaking)

Use this **out loud** in the room. The sections below are backup if they go deeper.

### Opening (first 20–30 seconds)

“Before I design, let me **clarify**: are hits **per page** or one global counter? Is the window **fixed** (e.g. five minutes) or **parameterized**? Should I assume **multi-threaded** callers, and can I **inject time** for tests? Once we agree, I’ll state one **invariant**: **`get_hits` never counts events outside the sliding window** relative to `now`.”

### Flow — cover in this order (SDE2 signal)

1. **Clarify** — page vs global, window, threading, clock.  
2. **Invariants** — stale buckets contribute zero; each bucket maps to one calendar minute.  
3. **Entities** — a service holding `page_id → counter`, and a per-page counter object.  
4. **APIs on the board** — `record_hit`, `get_hits` (signatures + what “now” means).  
5. **Data structures** — **minute ring buffer** per page; justify vs storing every timestamp.  
6. **Code** — **`record_hit`** and **`get_hits`** fully (rotation / expiry is the hard part).  
7. **Edge cases** — new page, window of 1, boundary minutes, optional “clock backward.”  
8. **Complexity** — “Update is O(1) per bucket slot; query is O(window) unless we maintain a running sum.”  
9. **Scale / concurrency** — shard by page; **lock per page**; Redis / streams if distributed.  
10. **Testing** — inject `now`; cases at minute boundaries.

### Natural phrases (sound human, not memorized)

- “Let me **play back** the time model so we don’t bake in the wrong semantics.”  
- “I’m optimizing for **bounded memory**: O(window) per page, not O(number of hits).”  
- “If you need **second-level** accuracy, I’d switch to second buckets or a deque of timestamps—same idea, different trade-off.”  
- “Under load I’d avoid one **global lock**; **per-page** isolation is the first lever.”

### APIs to write on the board

- `record_hit(page_id)`  
- `get_hits(page_id)` — and whether `now` is implicit or passed in.

### Must-code in the round (2–3)

1. **`record_hit`** (bucket index + expire stale).  
2. **`get_hits`** (sum valid buckets after rotation).  
3. Optional: **`_expire_stale` / rotate** if they want it factored.

### Edge cases — mention aloud

Empty page; first hit; window length 1; high cardinality of pages → **evict idle** counters if they bring it up.

### Strong closing

“This fits **in-memory exact counting**; at Uber scale I’d **shard by page**, use **per-page concurrency control**, and only then talk **approximate** structures if the product allows.”

### Mental checklist (10 seconds before you stop)

Clarified · Invariants · Entities · APIs · DS + why · Code (2–3) · Edges · Complexity · Concurrency/sharding · Testing.

---

## Elite human-driven script (full narrative)

Use this when you want a **slower, senior cadence**: you take the room, show trade-off thinking, narrate while coding, and check in with the interviewer. Same facts as above—just **more dialogue**.

### 1. Opening — take control

“Before I jump into design, let me **quickly clarify** a couple of things so we align on expectations.”

**Pause** — let them answer.

Then:

“Are we tracking hits **per page** or **globally**? Is the window **fixed** at five minutes or **configurable**? And should I assume **multi-threaded** callers, or start **single-threaded** and add locking if you want?”

(Optional add-on:) “For tests, is it okay if I **inject `now`** instead of only using wall clock?”

### 2. Set direction — show ownership

After they respond, **lock the scope** in one sentence:

“Great — I’ll design this as a **per-page** hit counter with a **sliding** time window. My goal is to keep **both** operations efficient and **memory bounded**.”

That signals intent without jumping to code.

### 3. Thought process — don’t jump straight to buckets

“Let me start with a **naive** approach first.”

**Brute force (say it, then reject it):**

“We could **store every timestamp** and on each query **filter** to the last N minutes.”

**Pause**, then:

“That doesn’t scale for what we care about: **query** becomes O(number of hits) and **memory** grows with traffic. So I want to **aggregate** instead of retaining every event.”

### 4. Transition — confident pivot

“So I’ll optimize by **aggregating into fixed-size time buckets**. Concretely, I’ll use **one-minute buckets** and a **circular buffer** so we **reuse** a fixed number of slots instead of growing without bound.”

**Winning line (memory story):**

“Instead of storing every hit, I only store **counts per minute** for the last *W* minutes.”

### 5. Why this (interviewers like the “because”)

“This gives us **bounded memory** per page—O(window), not O(hits)—**O(1)** work per `record_hit` for updating one bucket, and **predictable** performance for the hot path.”

If they push on query cost: “`get_hits` sums at most **W** buckets, so it’s **O(window)**; with a **fixed** window that’s **constant** in big-O terms relative to traffic.”

### 6. Structure — two-part design

“I’ll split it into **two** pieces: a **page-level** counter that owns the ring buffer, and a **service** that maps `page_id` to that counter so callers have one entry point.”

### 7. While coding — narrate out loud

Start with:

“I’ll start with the **page-level** counter.”

As you write, short **running commentary** (don’t read every line):

| What you’re writing | What you can say |
|---------------------|------------------|
| Arrays for counts + minute labels | “I’m storing **hit counts** and the **calendar minute** each slot represents, so I know what’s stale.” |
| `current_minute = floor(now / 60)` | “I normalize time to **minute granularity** for the bucket key.” |
| Index `minute % window` | “I map the minute into the ring with **modulo** so I reuse the same array.” |
| Reset stale bucket | “If this slot is for an **old** minute, I **reset** it before I add new hits.” |
| Increment | “Then I **increment** the count for **this** minute.” |
| Query loop | “On query I **sum only** buckets whose minute falls **inside** the window relative to `now`.” |

### 8. Keep the interviewer engaged (senior signal)

Occasionally:

“Does this approach make sense so far?”  
“Would you like me to go **deeper** on expiry, or move to the **service** layer?”

Use **sparingly**—two check-ins per phase is enough.

### 9. Edge cases — proactive

“Let me quickly call out **edge cases**: **new page**—I’ll use a map or `defaultdict` so the counter appears on first hit; **window of one**—still works; **very old** timestamps—those buckets are **outside** the window and won’t be counted. If **clock goes backward**, I’d align with you on whether we **ignore** or **still accept** hits.”

### 10. Complexity — short and confident

“**Update** is O(1) for touching one bucket. **Query** is O(window)—and with a **fixed** window size, that’s **constant** relative to traffic volume.”

### 11. Tie-back to threading (if they asked)

“If we’re multi-threaded, I’d **lock per page** rather than one global lock so different pages don’t contend.”

---

## How to open the interview (30 seconds)

> “I’ll model this as a service that records hits **per page** and answers **how many hits in the last N minutes** (or last 5 minutes if fixed). I’ll use a **fixed-size time bucket** structure per page so we don’t store every event, and I’ll ask whether we need thread safety and what time resolution we should use.”

---

## Clarifying questions (say these out loud)

| Question | Why it matters |
|----------|----------------|
| Per **URL / page id** or global counter only? | Drives `Map[page] → counter`. |
| Window: **last 5 minutes** fixed or **parameter N**? | Buffer size and API shape. |
| Time source: **wall clock**, **injected timestamps**, or **monotonic** for tests? | Testability and bucket rotation. |
| **Throughput** and **cardinality** of pages? | Eviction policy for cold pages vs memory blow-up. |
| **Multi-threaded**? | Locking strategy vs lock-free (often one lock per page is enough). |

---

## Requirements snapshot (after alignment)

- `record_hit(page_id)` — count one visit.
- `get_hits(page_id, window_minutes)` or fixed window — sum hits in sliding window.
- Optional: `get_hits_inclusive` semantics — define `[start, end)` vs inclusive minutes with interviewer.

---

## Approaches and why

| Approach | Pros | Cons |
|----------|------|------|
| **Store every timestamp** (list per page) | Exact, simple | Memory heavy; query is O(k) prune |
| **Minute ring buffer** (buckets) | O(1) record, O(window) query, bounded memory | Minute granularity; sub-minute needs finer buckets |
| **Redis-style** with external store | Distributed | Overkill for pure LLD unless asked |

**Pick:** minute (or second) **ring buffer per page** for the classic coding variant.

---

## Core design

- **`HitCounterService`**: maps `page_id → PageHitCounter`, optional global lock or per-counter lock.
- **`PageHitCounter`**: arrays `hits[i]` and `minute[i]` for each slot in the ring; lazy cleanup on read/write.

**Invariants**

- Each bucket either **empty** (`minute[i] == -1`) or holds hits for **exactly** that calendar minute.
- On any operation, **expire** buckets outside the window relative to “current minute”.

---

## Complexity

- **Time:** `record_hit` O(1) amortized rotation work spread across calls; `get_hits` O(window) per query if you sum all slots (can optimize to O(1) with total cache if interviewer allows and you maintain it).
- **Space:** O(number_of_pages × window) for bucket arrays.

---

## Optimizations (mention in interview)

- **Per-page lock** instead of one global lock on hot paths.
- **Second-level buckets** if window is short and they want precision.
- **Evict idle pages** (LRU) if page cardinality is huge.
- **Approximate counting** (HyperLogLog) only if they explicitly relax exact counts.

---

## Reference implementation (Python, heavily commented)

```python
"""
Hit counter with a sliding window in MINUTE granularity.

Interview notes:
- We use "epoch minute" = floor(epoch_seconds / 60) as the bucket key.
- Ring index = minute % window_minutes (works because we only care about
  the last `window_minutes` distinct minute labels relative to "now").
- We lazy-expire: on each read/write, zero out buckets older than the window.
"""

from __future__ import annotations

import time
from collections import defaultdict
from threading import Lock
from typing import DefaultDict


class PageHitCounter:
    """
    Thread-safe counter for a single page.

    Internal state:
    - _window: how many past minutes we retain (e.g. 5 for "last 5 minutes")
    - _buckets[i]: hit count for the minute stored in _minute_at[i]
    - _minute_at[i]: which "epoch minute" that slot refers to, or -1 if unused
    """

    def __init__(self, window_minutes: int) -> None:
        if window_minutes <= 0:
            raise ValueError("window_minutes must be positive")
        self._window = window_minutes
        # Parallel arrays implementing a ring buffer keyed by minute % window
        self._buckets: list[int] = [0] * window_minutes
        self._minute_at: list[int] = [-1] * window_minutes
        # Protects all reads/writes on this page's buckets (fine-grained locking)
        self._lock = Lock()

    def _expire_stale(self, current_minute: int) -> None:
        """
        Invalidate any bucket whose minute is too far in the past.

        We compare using integer minutes so "current_minute - stored_minute >= window"
        means the stored bucket is entirely outside the sliding window.
        """
        for i in range(self._window):
            stored = self._minute_at[i]
            if stored >= 0 and current_minute - stored >= self._window:
                self._minute_at[i] = -1
                self._buckets[i] = 0

    def record_hit(self, now_epoch_s: float | None = None) -> None:
        """
        Record one hit at time `now_epoch_s` (defaults to real time).

        Steps:
        1. Convert to "current minute" (integer).
        2. Expire stale buckets so we don't sum ancient data.
        3. Map current minute to ring index; if slot holds a different minute,
           reset it (collision after 300+ minutes for window=300 — impossible
           if window matches query window; if window is huge, discuss with interviewer).
        4. Increment count for that minute.
        """
        now = time.time() if now_epoch_s is None else now_epoch_s
        current_minute = int(now // 60)

        with self._lock:
            self._expire_stale(current_minute)
            idx = current_minute % self._window
            # If this ring slot was used for an older minute, reclaim it
            if self._minute_at[idx] != current_minute:
                self._minute_at[idx] = current_minute
                self._buckets[idx] = 0
            self._buckets[idx] += 1

    def get_hits(self, now_epoch_s: float | None = None) -> int:
        """
        Return total hits in the last `_window` minutes ending at `now`.

        Sum only buckets whose stored minute is within [now - window, now).
        Using strict half-open interpretation on the minute axis matches
        typical "last 5 minutes" expectations once aligned with interviewer.
        """
        now = time.time() if now_epoch_s is None else now_epoch_s
        current_minute = int(now // 60)

        with self._lock:
            self._expire_stale(current_minute)
            total = 0
            for i in range(self._window):
                m = self._minute_at[i]
                if m < 0:
                    continue
                # Bucket is valid if its minute is not older than window-1 minutes ago
                if current_minute - m < self._window:
                    total += self._buckets[i]
            return total


class HitCounterService:
    """
    Facade over many pages. Uses defaultdict so new pages appear on first hit.

    Locking strategy:
    - We take a short lock on the map only to fetch/create the PageHitCounter.
    - Alternatively, single global lock for entire service (simpler but less scalable).

    Here: lock the outer dict briefly, then delegate to per-page locks inside
    PageHitCounter. For creation races, defaultdict + lock around first access
    is acceptable in interview; production might use ConcurrentHashMap patterns.
    """

    def __init__(self, window_minutes: int) -> None:
        self._window = window_minutes
        self._pages: DefaultDict[str, PageHitCounter] = defaultdict(
            lambda: PageHitCounter(window_minutes)
        )
        self._map_lock = Lock()

    def record_hit(self, page_id: str, now_epoch_s: float | None = None) -> None:
        """Thread-safe: synchronize page creation; per-page counter has its own lock."""
        with self._map_lock:
            counter = self._pages[page_id]
        counter.record_hit(now_epoch_s)

    def get_hits(self, page_id: str, now_epoch_s: float | None = None) -> int:
        with self._map_lock:
            counter = self._pages[page_id]
        return counter.get_hits(now_epoch_s)
```

---

## Follow-up questions interviewers like

- What if the window is **24 hours** and traffic is huge? (Consider hour buckets + minute buckets hybrid, or approximate structures.)
- How do you **test** time? (Inject `now` or use a clock interface.)
- **Distributed** hits? (Sharding by page_id, Redis with TTL, or stream processing.)
