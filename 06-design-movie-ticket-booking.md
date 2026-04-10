# 6. Design Movie Ticket Booking (BookMyShow-style)

**Typical round:** 60 minutes · **Focus:** indexing by city/movie/cinema, seat hold/book concurrency story.

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
