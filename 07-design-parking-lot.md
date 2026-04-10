# 7. Design Parking Lot (Floors, Rows, 2W / 4W)

**Typical round:** 45–60 minutes · **Focus:** spot allocation strategy, data layout, thread safety story.

---

## Interview script & checklist (human speaking)

### Opening

“I’ll model **floors** and a **grid** of spots, each spot typed for **2W or 4W**. When you say **nearest**, I’ll interpret that as **lowest floor, then row-major order** unless you want a different metric. Invariant: **a spot holds at most one vehicle**; we return a **ticket** so `unpark` is O(1) lookup.”

### Flow — cover in this order

1. **Clarify** — nearest rule, vehicle types, full lot behavior, concurrent park.  
2. **Invariants** — one occupant per spot; ticket maps to exactly one spot.  
3. **Entities** — lot, `SpotId`, free **deques** per policy, ticket store.  
4. **APIs** — `park(vehicle_kind) -> Optional[Ticket]`, `unpark(ticket_id) -> bool`.  
5. **Data structures** — **deque per floor** (or per floor+kind) pre-filled in scan order; map `spot → kind` for returning to correct free pool.  
6. **Code** — **`park`** and **`unpark`**.  
7. **Edge cases** — full lot, wrong ticket, double unpark.  
8. **Complexity** — O(1) park/unpark with pre-bucketed free lists.  
9. **Scale** — **per-floor locks**; striping if needed.  
10. **Testing** — fill and fail; unpark frees correct type queue.

### Natural phrases

- “I’m using **deques** so ‘nearest’ is just **pop front** from the precomputed order.”  
- “On **unpark**, I need to know the spot’s **kind** so I push back to the **right** free queue.”

### APIs on the board

`park` · `unpark` · optional `status`.

### Must-code (2–3)

1. **`park`**  
2. **`unpark`**  
3. Optional **init** that builds free queues (explain verbally if low on time).

### Closing

“If contention is high, I’d **lock per floor** instead of the whole lot, with a fixed **lock ordering** on floors to avoid deadlock.”

### Mental checklist

Nearest policy · Two deques or queues · Ticket map · Code · Full lot · Concurrency · Tests.

---

## Interview opener

> “I’ll model a **parking lot** with floors; each floor has spots in a **row × col** grid. Spots are typed for **two-wheeler** or **four-wheeler**. `park(vehicle)` finds the **nearest** available compatible spot—I'll define nearest as **lowest floor, then row, then column** unless they want a different distance metric. `unpark` frees by **ticket id**.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Multiple lots or one? | Scope |
| **Nearest** definition? | Algorithm |
| **Full** behavior — queue vs reject? | API |
| One vehicle per spot? | Yes typically |
| **Concurrency** | Lock lot or striping |

---

## Approach

- `ParkingSpot` identified by `(floor, row, col)` with a **SpotKind** (TWO vs FOUR).
- **Free lists:** `deque` per floor for TWO spots and per floor for FOUR spots, **pre-filled in row-major order** so `popleft()` gives deterministic “nearest”.
- **Ticket** maps `ticket_id → (spot, vehicle_kind)`; **unpark** returns spot to the correct deque using a side map `spot → SpotKind`.

**Why deques:** O(1) allocate/release at the chosen policy order.

---

## Complexity

- Init: O(F × R × C)
- Park / unpark: O(1) with pre-bucketed free lists

---

## Reference implementation (Python)

```python
"""
Parking lot: multiple floors, 2D grid, two vehicle types.

Allocation:
- Row-major order within each floor for each spot kind.
- park() takes first available from lowest floor that still has a deque head.

Thread safety:
- One lot-level lock for interview clarity; striping per floor if contention is topic.
"""

from __future__ import annotations

from collections import deque
from dataclasses import dataclass
from enum import Enum
from itertools import count
from threading import Lock
from typing import Deque, Optional


class VehicleKind(str, Enum):
    TWO_WHEELER = "2W"
    FOUR_WHEELER = "4W"


class SpotKind(str, Enum):
    TWO = "TWO"
    FOUR = "FOUR"


@dataclass(frozen=True)
class SpotId:
    """Immutable identifier for a physical spot."""

    floor: int
    row: int
    col: int


@dataclass
class Ticket:
    ticket_id: str
    spot: SpotId
    vehicle_kind: VehicleKind


class ParkingLot:
    def __init__(
        self,
        floors: int,
        rows: int,
        cols: int,
        spot_kind_at: dict[tuple[int, int, int], SpotKind],
    ) -> None:
        """
        spot_kind_at maps (floor, row, col) -> TWO or FOUR for every cell.

        Why pass explicit map:
        - Interview flexibility (irregular layouts, blocked cells omitted from map).
        - If map is complete grid, caller builds it in loops.
        """
        self._floors = floors
        self._rows = rows
        self._cols = cols

        # Occupancy: None means free; otherwise stores ticket_id string
        self._occupant: dict[SpotId, Optional[str]] = {}
        # ticket_id -> Ticket object for unpark
        self._tickets: dict[str, Ticket] = {}
        # Remember physical kind to return spot to correct free-queue on unpark
        self._spot_kind: dict[SpotId, SpotKind] = {}

        # Per-floor queues of FREE spots in row-major insertion order
        self._free_two: list[Deque[SpotId]] = [deque() for _ in range(floors)]
        self._free_four: list[Deque[SpotId]] = [deque() for _ in range(floors)]

        self._lock = Lock()
        self._ticket_seq = count(1)

        for f in range(floors):
            for r in range(rows):
                for c in range(cols):
                    key = (f, r, c)
                    if key not in spot_kind_at:
                        continue
                    sid = SpotId(f, r, c)
                    kind = spot_kind_at[key]
                    self._spot_kind[sid] = kind
                    self._occupant[sid] = None
                    if kind == SpotKind.TWO:
                        self._free_two[f].append(sid)
                    elif kind == SpotKind.FOUR:
                        self._free_four[f].append(sid)
                    else:
                        raise ValueError("Unknown spot kind")

    def _take_first_available(self, per_floor: list[Deque[SpotId]]) -> Optional[SpotId]:
        """
        Scan floors from 0 upward; first non-empty deque wins.

        This encodes 'nearest' as smallest floor index, then row/col via enqueue order.
        """
        for f in range(self._floors):
            if per_floor[f]:
                return per_floor[f].popleft()
        return None

    def _release_spot(self, spot: SpotId) -> None:
        """Push back to the correct free deque (call only while holding lock)."""
        kind = self._spot_kind[spot]
        if kind == SpotKind.TWO:
            self._free_two[spot.floor].append(spot)
        else:
            self._free_four[spot.floor].append(spot)

    def park(self, vehicle: VehicleKind) -> Optional[Ticket]:
        """
        Assign a compatible spot. Returns None if fully occupied for that type.

        Steps:
        1. Choose deque family (2W vs 4W).
        2. Pop first free SpotId in global row-major policy.
        3. Mark occupant, issue monotonic ticket id.
        """
        with self._lock:
            if vehicle == VehicleKind.TWO_WHEELER:
                spot = self._take_first_available(self._free_two)
            else:
                spot = self._take_first_available(self._free_four)

            if spot is None:
                return None

            tid = f"T-{next(self._ticket_seq)}"
            self._occupant[spot] = tid
            ticket = Ticket(ticket_id=tid, spot=spot, vehicle_kind=vehicle)
            self._tickets[tid] = ticket
            return ticket

    def unpark(self, ticket_id: str) -> bool:
        """
        Free a spot given ticket. Idempotent failure returns False.

        Why we need _spot_kind:
        - On unpark we must push the spot back into TWO or FOUR queue correctly.
        """
        with self._lock:
            ticket = self._tickets.pop(ticket_id, None)
            if ticket is None:
                return False

            # Sanity: ticket should match occupant map
            if self._occupant.get(ticket.spot) != ticket_id:
                # In production log error; interview: restore ticket for debug
                self._tickets[ticket_id] = ticket
                return False

            self._occupant[ticket.spot] = None
            self._release_spot(ticket.spot)
            return True
```

---

## Optimizations

- **Per-floor locks** for high concurrency (watch deadlock ordering floor id).
- **Compact / large vehicle** rules (compact spot can take motorcycle) — extend `park` to try multiple deques in priority order.
- **Electric charging spots** — extra attribute filter.

---

## Testing checklist

- Park 2W only on TWO spots; 4W only on FOUR spots.
- Fill lot → `park` returns `None`.
- `unpark` then `park` reuses freed spot in correct queue order.
