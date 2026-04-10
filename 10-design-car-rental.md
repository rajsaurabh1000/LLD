# 10. Design Car Rental (Full-Day Bookings, Trip Cost)

**Typical round:** 45–60 minutes · **Focus:** booking calendar per car, pricing rules, checkout.

---

## Correct interview flow (this document)

Same order as **README**. Do **not** open with **ordinals** / **half-open** until **date semantics**, **pricing**, and **late fee** rules are agreed.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“**Full-day** bookings only? Are pickup/return **inclusive** dates on the UI—how do we avoid **off-by-one**? **Late fee** per day after agreed return? **Early return**—refund or full charge? **Extras** at checkout? **Same car**—**no overlap**? **Who** creates `booking_id`?”

**Pause.** Then:

“**Invariant**: **no overlapping bookings per car**; **checkout** matches agreed **pricing rules**. I’ll do **FR/NFR**, **`register` / `book` / `complete_trip`** on the board, **then** map dates to a clear interval model and implement overlap + pricing.”

### Flow — cover in this order

1. **Open / clarify** — as above + table.  
2. **FR / NFR** — section below.  
3. **Invariant** — non-overlap; checkout = rules + release car.  
4. **Entities** — car, service, bookings **after** API.  
5. **APIs on board** — `register_car`, `book`, `complete_trip`.  
6. **Data structures** — **ordinals + half-open** intervals per car + booking map.  
7. **Code** — **`book`**, **`complete_trip`**.  
8. **Edge cases** — bad dates, late days, unknown id.  
9. **Complexity** — book insert; complete O(1) with map.  
10. **Scale / tests** — pricing strategy; adjacent bookings.

### Natural phrases

- “I store **half-open ordinals** so overlap logic is identical to **room booking**.”  
- “**Late days** are ‘**actual return ordinal minus last included day**’—I’ll write that carefully on the board.”

### APIs on the board

`register_car` · `book` · `complete_trip` (cents or dollars—pick one).

### Must-code (2–3)

1. **`book`**  
2. **`complete_trip`**  
3. Optional **`cancel`** if they want lifecycle.

### Closing

“Production-wise I’d separate **pricing rules** from scheduling, add **idempotent** `book` with a client request id, and persist with a **constraint** preventing overlapping ranges per car.”

### Mental checklist

Clarified · FR/NFR · APIs before ordinals · `book` + `complete_trip` · Late fee · Tests.

---

## After alignment

> “I’ll store intervals as **half-open day ordinals** internally so overlap matches **meeting-room** logic; **complete_trip** computes base + late + extras and **frees** the car.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Date representation — `date` only? | No partial days |
| **Extend** rental vs new booking? | API |
| **Deposit** held? | Payment out of scope often |
| **Different car classes** pricing? | Strategy pattern |
| **Cancel** policy? | Refund rules |

---

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** on **calendar / ordinals** and pricing rules; then book + checkout.

### Functional requirements (FR)

- **Register** cars with **daily rate** (and late fee if in scope).
- **Book** car for **inclusive/exclusive** date range as agreed — **no overlap** on same car.
- **Complete trip**: compute **total charge** (base days × rate + **late** + extras) and **release** car.

**What you can say:** “**FRs**: book full-day blocks, **checkout** with pricing, no double booking.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | Non-overlap per car; fee matches rules | “**Half-open ordinals** same as meeting rooms.” |
| **Performance** | Fast book + checkout | “Sorted intervals + **booking_id** map.” |
| **Extensibility** | Pricing rules change | “Optional **PricingEngine** / strategy.” |

**What you can say:** “**NFRs**: scheduling correctness, clear pricing story, easy to extend rules.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`Car`** | id, `daily_rate_cents`, `late_fee_per_day_cents`, … |
| **`CarRentalService`** | Cars; per-car **sorted bookings**; `booking_id → booking` for checkout. |

**Relationships:** Service **owns** cars; each car **owns** non-overlapping rental intervals.

**What you can say:** “**Entities**: **car** with rates, **service** with schedules and active bookings map.”

### API design

| Method | Purpose |
|--------|---------|
| `register_car(car)` | Add fleet member. |
| `book(booking_id, car_id, user_id, pickup, return_inclusive) -> bool` | Align date semantics with interviewer. |
| `complete_trip(booking_id, actual_return, extras_cents) -> Optional[int]` | Total charge; remove booking. |

**What you can say:** “**API**: **register**, **book**, **complete_trip**—pricing encapsulated in checkout.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code** (see **README**).

---

## Approach

- `Car(car_id, daily_rate_cents, ...)`
- `RentalBooking(car_id, user_id, start_date, end_date_exclusive, booking_id)`
- **Overlap check** on `[start, end)` dates using ordinal integers `date.toordinal()`.
- **`complete_trip(booking_id, actual_return_date, extras)`** computes:

  `base = daily_rate * booked_days`

  `late = max(0, actual_return - agreed_end_exclusive) * late_fee_per_day`

  plus extras.

**Why end exclusive:** simplifies overlap (`new_start < existing_end and new_end > existing_start`).

---

## Complexity

- Book: O(n) per car list insert (same as other interval problems).
- Checkout: O(1) lookup if `booking_id → booking` map.

---

## Reference implementation (Python)

```python
"""
Car rental: full-day bookings per car, priced at trip end.

Overlap detection uses integer day ordinals for half-open [start, end).

Pricing (example rules — narrate in interview):
- Booked period: [pickup_date, return_date) where return_date is the first day car is NOT needed
  (e.g. pickup Mon, return next Mon => 7 days if they count full week; ALIGN WITH INTERVIEWER).
- Base cost = num_booked_days * daily_rate
- If actual_return_date > agreed return_date (calendar day), charge late_fee_per_day per late day

This file uses:
- pickup_date inclusive, agreed_return_date inclusive as STORED,
  then converts to half-open internally for easier overlap math.
"""

from __future__ import annotations

import bisect
from dataclasses import dataclass
from datetime import date
from threading import Lock
from typing import Optional


@dataclass
class Car:
    car_id: str
    daily_rate_cents: int
    late_fee_per_day_cents: int = 0


@dataclass
class RentalBooking:
    booking_id: str
    car_id: str
    user_id: str
    # Half-open in ORDINAL space: [start_ord, end_ord)
    start_ord: int
    end_ord: int


class CarRentalService:
    def __init__(self) -> None:
        self._cars: dict[str, Car] = {}
        # car_id -> sorted list of (start_ord, end_ord, booking_id)
        self._bookings: dict[str, list[tuple[int, int, str]]] = {}
        self._booking_map: dict[str, RentalBooking] = {}
        self._lock = Lock()

    def register_car(self, car: Car) -> None:
        with self._lock:
            self._cars[car.car_id] = car
            self._bookings.setdefault(car.car_id, [])

    def _overlap(self, intervals: list[tuple[int, int, str]], s: int, e: int) -> bool:
        pos = bisect.bisect_left(intervals, (s, -10**18, ""))
        if pos < len(intervals):
            ns, ne, _ = intervals[pos]
            if s < ne and e > ns:
                return True
        if pos > 0:
            ps, pe, _ = intervals[pos - 1]
            if s < pe and e > ps:
                return True
        return False

    def book(
        self,
        booking_id: str,
        car_id: str,
        user_id: str,
        pickup: date,
        return_date_inclusive: date,
    ) -> bool:
        """
        Book car from pickup through return_date_inclusive (both inclusive calendar days).

        Convert to half-open ordinals:
        - start_ord = pickup.toordinal()
        - end_ord = return_date_inclusive.toordinal() + 1
        So number of days = end_ord - start_ord.

        Why +1 on end:
        - Makes interval math match other problems (meeting rooms).
        """
        if return_date_inclusive < pickup:
            return False

        start_ord = pickup.toordinal()
        end_ord = return_date_inclusive.toordinal() + 1

        with self._lock:
            if car_id not in self._cars:
                return False
            iv = self._bookings[car_id]
            if self._overlap(iv, start_ord, end_ord):
                return False
            bisect.insort(iv, (start_ord, end_ord, booking_id))
            self._booking_map[booking_id] = RentalBooking(
                booking_id=booking_id,
                car_id=car_id,
                user_id=user_id,
                start_ord=start_ord,
                end_ord=end_ord,
            )
            return True

    def complete_trip(
        self,
        booking_id: str,
        actual_return: date,
        extras_cents: int = 0,
    ) -> Optional[int]:
        """
        Finalize rental and return total charge in cents.

        Rules implemented (explicitly discuss with interviewer):
        - booked_days = end_ord - start_ord (from booking)
        - base = booked_days * daily_rate
        - late_days = max(0, actual_return.toordinal() - (end_ord - 1))
          Here agreed last day ORDINAL is end_ord - 1 (inclusive return date).
        - late charge = late_days * late_fee_per_day_cents
        - add extras (tolls, cleaning)

        Returns None if booking unknown.
        """
        with self._lock:
            b = self._booking_map.pop(booking_id, None)
            if b is None:
                return None
            car = self._cars[b.car_id]
            booked_days = b.end_ord - b.start_ord
            base = booked_days * car.daily_rate_cents

            # Agreed inclusive last day ordinal:
            agreed_last_ord = b.end_ord - 1
            late_days = max(0, actual_return.toordinal() - agreed_last_ord)
            late = late_days * car.late_fee_per_day_cents

            # Remove from car schedule (trip complete — car available again)
            iv = self._bookings[b.car_id]
            for i, t in enumerate(iv):
                if t[2] == booking_id:
                    iv.pop(i)
                    break

            return base + late + extras_cents
```

---

## Optimizations & extensions

- **Dynamic pricing** — `PricingEngine` interface (weekend surge).
- **One-way rental** — different drop-off location fee table.
- **Inventory pool** — “any compact car” maps to fleet class with free car selection.

---

## Pitfalls

- **Inclusive vs exclusive** end date — must be consistent in `book` and `complete_trip`.
- Removing booking from car’s interval list on complete — don’t leave ghost blocks.

---

## Testing angles

- Book adjacent days on same car — should succeed with half-open model.
- Return **early** — usually no refund unless interviewer says (implementation above charges full booked period).
- Return **late** — late fee ticks per calendar day late.
