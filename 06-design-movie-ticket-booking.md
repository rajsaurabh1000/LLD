# 6. Design Movie Ticket Booking (BookMyShow-style)

**Typical round:** 60 minutes · **Focus:** indexing by city/movie/cinema, seat hold/book concurrency story.

---

## Correct interview flow (this document)

Same order as **README**. Do **not** open with **denormalized indexes** or **per-show lock** until **scope**, **FR/NFR**, and **public API** are clear.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“What’s in scope: **add cinema/show**, **discovery** (cinemas in a city for a movie, shows at a cinema for a movie), and **book seats**? How are **seats** identified? Is booking **all-or-nothing** for a set? **Payment / holds with TTL** in scope or out? **Cancel** seats? **Concurrency** expectations?”

**Pause.** Then:

“**Invariant**: a **seat can’t be sold twice** for the same show; **discovery indexes** must stay **consistent** with the catalog. I’ll state **FR/NFR**, write **add/list/book** APIs, **then** decide **indexes** and **locking**.”

### Flow — cover in this order

1. **Open / clarify** — as above + table.  
2. **FR / NFR** — section below.  
3. **Invariant** — no double booking; index consistency.  
4. **Entities** — `Show`, service **after** API.  
5. **APIs on board** — `add_show`, `list_cinemas`, `list_shows`, `book`.  
6. **Data structures** — per-show **seat set** + **denormalized** discovery maps.  
7. **Code** — **`book`**, **`add_show`** (indexes).  
8. **Edge cases** — partial book failure, unknown show.  
9. **Complexity** — book O(k); list O(results).  
10. **Scale / tests** — per-show lock; concurrent last seat.

### Natural phrases

- “I’m **denormalizing** for read paths: writes to shows are rarer than ‘browse by city and movie’.”  
- “**All-or-nothing** seat booking is one transaction under one lock—simple to reason about.”  
- “If we add **holds**, I’d store `held_until` and sweep expired holds before confirm.”

### APIs on the board

Discovery APIs + `book(show_id, seats)`.

### Must-code (2–3)

1. **`book`** (atomic subset check + remove).  
2. **`add_show`** (create show + update both indexes).  
3. Optional one **list** method.

### Closing

“At scale the **show shard** is the bottleneck; I’d **pin inventory to one writer**, cache **listings**, and use **idempotent** `book` with a request id for retries.”

### Mental checklist

Clarified · FR/NFR · APIs before indexes · `book` atomicity · Concurrency · Tests.

---

## After alignment

> “I’ll **denormalize** `(city, movie) → cinemas` and `(cinema, movie) → shows` for reads, and keep **available seats** on each **show** with a **lock** for atomic **book**.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Seat model: **SeatId** string vs matrix? | Data structure |
| **Concurrent** booking same seat? | Lock per show or per seat set |
| **Refund / cancel**? | Release seats |
| Same movie **multiple shows** same day? | Show is first-class entity |
| Pricing in scope? | Often out of scope for LLD |

---

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** on catalog vs inventory; then indexes + atomic **book**.

### Functional requirements (FR)

- **Add** cinema / show (as scoped): register shows with **city**, **cinema**, **movie**, **time**, **seats**.
- **Discover**: cinemas in a **city** showing a given **movie**; shows at a **cinema** for a **movie**.
- **Book** one or more **seats** for a **show** — **all-or-nothing** if agreed.

**What you can say:** “**FRs**: catalog + **discovery** queries + **atomic seat booking**.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | No double booking same seat/show | “**Invariant**: seat set mutation is **atomic** under concurrency.” |
| **Performance** | Fast browse; book bounded by seat count | “**Denormalized indexes** for discovery.” |
| **Concurrency** | Many users booking same show | “**Lock per show** (or finer) for inventory.” |
| **Availability** | Idempotent **book** with request id (if they ask) | “Retries shouldn’t double-charge seats.” |

**What you can say:** “**NFRs**: consistency on seats first; scalable reads second.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`Show`** | `show_id`, cinema, city, movie, time, **available_seats**; lock for booking. |
| **`MovieBookingService`** | Registry of shows; **indexes** `(city,movie)→cinemas`, `(cinema,movie)→shows`; `book`. |

**Relationships:** Service **owns** shows and **maintains** query indexes when shows are added.

**What you can say:** “**Entities**: **Show** as inventory unit; **service** owns catalog and indexes.”

### API design

| Method | Purpose |
|--------|---------|
| `add_show(...)` | Create show + update indexes. |
| `list_cinemas(city_id, movie_id)` | Discovery. |
| `list_shows(cinema_id, movie_id)` | Discovery. |
| `book(show_id, seats) -> bool` | Atomic reserve if all seats free. |

**What you can say:** “**API**: two **list** paths plus **book**; everything else internal.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code** (see **README**).

---

## Approach

**Entities**

- `City` → `cinemas: dict[cinema_id, Cinema]`
- `Cinema` → `screens`, metadata
- `Show` → `movie_id`, `start_time`, `available_seats: set[SeatId]` (or dict row→set)
- **Indexes (denormalized)** for queries:
  - `(city_id, movie_id) → set[cinema_id]` updated when show is added
  - `(cinema_id, movie_id) → list[show_id]`

**Book seats:** under lock for that show, check subset of `available_seats`, then remove.

---

## Complexity

- List cinemas: O(1) index lookup + O(C) list if stored as set.
- List shows: O(S) shows for that pair.
- Book: O(k) for k seats.

---

## Reference implementation (Python)

```python
"""
Simplified BookMyShow-style model: catalog + per-show seat inventory.

Focus for interview:
- Denormalized indexes for "cinemas in city with movie" and "shows at cinema for movie"
- Atomic seat booking with a per-show lock

Out of scope unless asked: payments, holds with TTL, waitlists, dynamic pricing.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from threading import Lock
from typing import Optional


@dataclass
class Show:
    show_id: str
    cinema_id: str
    city_id: str
    movie_id: str
    start_time: int
    # Set of seat identifiers, e.g. "A1", "A2"
    available_seats: set[str] = field(default_factory=set)
    _lock: Lock = field(default_factory=Lock, repr=False)

    def book_seats(self, seats: set[str]) -> bool:
        """
        Try to book all seats atomically. Returns False if any seat missing.

        Why lock per show:
        - Two users booking concurrently must not both succeed on same seat.
        - Coarser lock is OK for LLD; finer locks per seat add complexity.
        """
        with self._lock:
            if not seats.issubset(self.available_seats):
                return False
            self.available_seats -= seats
            return True

    def release_seats(self, seats: set[str]) -> None:
        """Cancel booking / refund — add seats back."""
        with self._lock:
            self.available_seats |= seats


class MovieBookingService:
    def __init__(self) -> None:
        self._shows: dict[str, Show] = {}
        # (city, movie) -> cinema ids that have at least one show for that movie
        self._city_movie_cinemas: dict[tuple[str, str], set[str]] = {}
        # (cinema, movie) -> show ids
        self._cinema_movie_shows: dict[tuple[str, str], list[str]] = {}
        self._catalog_lock = Lock()

    def add_show(
        self,
        show_id: str,
        city_id: str,
        cinema_id: str,
        movie_id: str,
        start_time: int,
        seats: set[str],
    ) -> None:
        """
        Register a new show and refresh query indexes.

        Interview tip: say you're denormalizing for read-heavy query paths.
        """
        show = Show(
            show_id=show_id,
            cinema_id=cinema_id,
            city_id=city_id,
            movie_id=movie_id,
            start_time=start_time,
            available_seats=set(seats),
        )
        with self._catalog_lock:
            self._shows[show_id] = show
            self._city_movie_cinemas.setdefault((city_id, movie_id), set()).add(
                cinema_id
            )
            self._cinema_movie_shows.setdefault((cinema_id, movie_id), []).append(
                show_id
            )

    def list_cinemas_with_movie(self, city_id: str, movie_id: str) -> list[str]:
        with self._catalog_lock:
            return sorted(self._city_movie_cinemas.get((city_id, movie_id), set()))

    def list_shows(self, cinema_id: str, movie_id: str) -> list[Show]:
        """Return show objects (or DTOs) for cinema+movie."""
        with self._catalog_lock:
            ids = list(self._cinema_movie_shows.get((cinema_id, movie_id), []))
            return [self._shows[i] for i in ids if i in self._shows]

    def book(self, show_id: str, seats: set[str]) -> bool:
        show = self._shows.get(show_id)
        if show is None:
            return False
        return show.book_seats(seats)
```

---

## Optimizations

- **TTL holds:** `reserved_until` map + lazy cleanup on book.
- **Index cleanup** when shows end (TTL job) — avoid stale cinemas in index.
- **Read replicas** for listing; **single writer** for booking.

---

## Pitfalls

- Forgetting to update **both** indexes when adding/removing shows.
- **Non-atomic** check-then-set on seats → double booking under concurrency.
