# 6. Design Movie Ticket Booking (BookMyShow-style)

**Typical round:** 60 minutes · **Focus:** indexing by city/movie/cinema, seat hold/book concurrency story.

---

## Interview script & checklist (human speaking)

### Opening

“I’ll separate **catalog** data from **inventory** per show. Users need **discovery** queries—cinemas in a city showing a movie, shows at a cinema for a movie—and **booking**, where the hard part is **no double booking** under concurrency. I’ll ask about **seat identifiers**, whether booking is **all-or-nothing** for a set of seats, and if we need **holds with TTL** or keep scope to immediate commit.”

### Flow — cover in this order

1. **Clarify** — seat model, payment/hold out of scope?, cancel, add cinema/show APIs.  
2. **Invariants** — a seat can’t be sold twice for the same show; indexes stay consistent with catalog.  
3. **Entities** — `Show` (availability + lock), service with **denormalized indexes** `(city, movie)→cinemas`, `(cinema, movie)→shows`.  
4. **APIs** — `add_show`, `list_cinemas(city, movie)`, `list_shows(cinema, movie)`, `book(show, seats) -> bool`.  
5. **Data structures** — `set` of seats per show; inverted indexes for queries.  
6. **Code** — **`book`** under **per-show lock**; **`add_show`** updating indexes.  
7. **Edge cases** — partial failure (reject if any seat taken), unknown show, empty seat set.  
8. **Complexity** — book O(k) seats; listing O(result size).  
9. **Scale** — show as contention unit; read replicas for listings; idempotent booking token.  
10. **Testing** — concurrent two buyers, same last seat.

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

Indexes + inventory · Invariant on seats · APIs · `book` atomicity · Concurrency story · Tests.

---

## Interview opener

> “I’ll separate **catalog** (cities, cinemas, screens, shows) from **booking** (seat sets per show). Users need: add cinema/show, **list cinemas in a city** showing a movie, **list shows at a cinema** for a movie, and **book seats** atomically without double booking. I’ll confirm seat layout (rows/cols vs seat ids) and whether we need **hold + expiry**.”

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
