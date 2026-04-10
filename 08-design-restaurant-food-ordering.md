# 8. Design Restaurant Food Ordering & Ratings

**Typical round:** 60 minutes · **Focus:** many-to-many item↔restaurant, order + rating aggregates, top restaurants queries.

---

## Correct interview flow (this document)

Same order as **README**. Do **not** open with **MenuEntry** / **denormalized aggregates** until **rating rules** and **API surface** are clear.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“What do we support: **restaurants**, **menu items** at a restaurant, **place order**, **rate order**? Is rating **per order** once? **Scale** (1–5)? **Tie-break** for top lists? **Normalize** food names for ‘best for X’? **Double rating** same order?”

**Pause.** Then:

“**Invariant**: **one rating per order**; **aggregates** reflect rolled-up ratings. I’ll do **FR/NFR**, list **APIs** (catalog, order, rate, top queries), **then** add **indexes** for fast leaderboards.”

### Flow — cover in this order

1. **Open / clarify** — as above + table.  
2. **FR / NFR** — section below.  
3. **Invariant** — single rate per order; consistent aggregates.  
4. **Entities** — restaurant, menu entry, order, service **after** API.  
5. **APIs on board** — add restaurant/entry, place, rate, top global, top for food.  
6. **Data structures** — maps + **per-food / per-restaurant** running sums.  
7. **Code** — **`rate_order`**, **one top-k**.  
8. **Edge cases** — unknown order, duplicate rate.  
9. **Complexity** — rate O(1); top O(R log R) sort.  
10. **Scale / tests** — CQRS; two restaurants same dish.

### Natural phrases

- “I **denormalize aggregates** so ‘top restaurants’ doesn’t rescan every order on every query.”  
- “For ‘**best for veg burger**’, I use the same pattern but **scoped** to restaurants that serve that normalized key.”

### APIs on the board

Place + rate + two query shapes (global vs per food).

### Must-code (2–3)

1. **`rate_order`**  
2. **`top_restaurants`** or **`top_restaurants_for_food`**  
3. Optional **`add_menu_entry`** to show index maintenance.

### Closing

“If reads dominate, I’d serve **materialized leaderboards** from a cache and treat ratings as **append-only events** for replay and audits.”

### Mental checklist

Clarified · FR/NFR · APIs before aggregates · `rate_order` + top · Edges · Tests.

---

## After alignment

> “Orderable unit is **MenuEntry** (food at a restaurant). I’ll **denormalize** running averages per restaurant and a **secondary index** keyed by **normalized food name** for ‘best for X’.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Rating scale (1–5)? | Validation |
| Rate **order** vs **restaurant**? | Problem says orders → attribute to restaurant |
| **Weighted** by recency? | v2 |
| Case sensitivity of food names? | Normalize keys |
| Need **menu search**? | Out of scope unless asked |

---

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** on **MenuEntry** vs food name; ratings → aggregates.

### Functional requirements (FR)

- **Register** restaurants and **menu entries** (food at a restaurant).
- **Place order** (line items reference **menu entry** ids at that restaurant).
- **Rate order** (once); **query** top restaurants globally and **top for a food name**.

**What you can say:** “**FRs**: catalog, order, rate order, **two ranking** queries.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | One rating per order; aggregates consistent | “**Sum/count** updated with each rate.” |
| **Performance** | Top queries without full order scan | “**Denormalized** restaurant + per-food aggregates.” |
| **Data quality** | Normalized food key for “same dish” | “**Lowercase / trim** index key.” |

**What you can say:** “**NFRs**: fast leaderboards via **pre-aggregation**.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`Restaurant`**, **`MenuEntry`**, **`Order`** | Catalog + order lines + optional rating. |
| **`FoodOrderingService`** | Stores entities; **running aggregates** per restaurant and per (food, restaurant). |

**Relationships:** Order **references** restaurant + entries; rating **updates** aggregate indexes.

**What you can say:** “**Entities**: **menu entry** is the orderable unit; **service** owns aggregates.”

### API design

| Method | Purpose |
|--------|---------|
| `add_restaurant`, `add_menu_entry` | Catalog. |
| `place_order(order)` | Validate entries belong to restaurant. |
| `rate_order(order_id, rating)` | Once; roll up stats. |
| `top_restaurants(k)`, `top_restaurants_for_food(food, k)` | Queries. |

**What you can say:** “**API**: small **write** surface; **two read** patterns for rankings.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code** (see **README**).

---

## Approach

**Entities**

- `Restaurant(id, name, …)`
- `MenuEntry(id, restaurant_id, food_name, price)` — unique per (restaurant, food) instance
- `Order(id, user_id, restaurant_id, lines, rating: Optional[float])`

**Aggregates (denormalized for fast queries)**

- `restaurant_stats[rest_id] → (sum_rating, count)` updated on `rate_order`
- **Inverted index** `food_name_normalized → {rest_id → (sum, count)}` for “best for veg burger”

**Why denormalize:** O(1) update on rate; O(R) scan per food for top unless you keep a heap per food (mention as extension).

---

## Complexity

- `rate_order`: O(1) aggregate update + O(1) per food index bucket
- `top_restaurants_global(k)`: O(N log k) heap or O(N log N) sort
- `top_restaurants_for_food(food, k)`: O(number of restaurants serving food)

---

## Reference implementation (Python)

```python
"""
Restaurant ordering + ratings.

Model:
- MenuEntry ties a human-readable food_name to a specific restaurant.
- Order stores which restaurant and which menu entry ids were ordered.
- Rating is attached to an order and rolls up to:
  - restaurant-level running average
  - per-food-name index: for each normalized food name, track stats per restaurant

Interview simplifications:
- In-memory only, single-threaded Lock for demo concurrency.
- top_k uses full sort for clarity; mention heap for large N.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from threading import Lock
from typing import Optional


def _norm_food(name: str) -> str:
    """Normalize for index keys — lower/strip; extend with slug rules if asked."""
    return name.strip().lower()


@dataclass
class MenuEntry:
    entry_id: str
    restaurant_id: str
    food_name: str
    price_cents: int


@dataclass
class Order:
    order_id: str
    user_id: str
    restaurant_id: str
    # menu_entry_id -> qty
    lines: dict[str, int]
    rating: Optional[float] = None


@dataclass
class _Agg:
    """Running sum/count for incremental average."""

    sum_r: float = 0.0
    count: int = 0

    def mean(self) -> float:
        return self.sum_r / self.count if self.count else 0.0


class FoodOrderingService:
    def __init__(self) -> None:
        self._restaurants: dict[str, str] = {}  # id -> name
        self._menu: dict[str, MenuEntry] = {}  # entry_id -> entry
        self._orders: dict[str, Order] = {}

        self._restaurant_rating = dict[str, _Agg]()
        # food_key -> restaurant_id -> Agg for restaurants that serve that food
        self._food_restaurant_rating: dict[str, dict[str, _Agg]] = {}
        self._lock = Lock()

    def add_restaurant(self, restaurant_id: str, name: str) -> None:
        with self._lock:
            self._restaurants[restaurant_id] = name
            self._restaurant_rating.setdefault(restaurant_id, _Agg())

    def add_menu_entry(self, entry: MenuEntry) -> None:
        """
        Register a dish at a restaurant. Builds food→restaurant index for later queries.

        Why index at insert time:
        - Speeds up 'who serves veg burger' without scanning all entries each query.
        """
        with self._lock:
            if entry.restaurant_id not in self._restaurants:
                raise ValueError("Unknown restaurant")
            self._menu[entry.entry_id] = entry
            key = _norm_food(entry.food_name)
            self._food_restaurant_rating.setdefault(key, {}).setdefault(
                entry.restaurant_id, _Agg()
            )

    def place_order(self, order: Order) -> None:
        """Persist order without rating yet."""
        with self._lock:
            if order.restaurant_id not in self._restaurants:
                raise ValueError("Unknown restaurant")
            for eid in order.lines:
                if eid not in self._menu:
                    raise ValueError(f"Unknown menu entry {eid}")
                if self._menu[eid].restaurant_id != order.restaurant_id:
                    raise ValueError("Menu entry not at this restaurant")
            self._orders[order.order_id] = order

    def rate_order(self, order_id: str, rating: float, scale_max: float = 5.0) -> None:
        """
        Attach rating to order and update aggregates.

        Invariants:
        - One rating per order (reject if already set unless interviewer allows edit).
        - Clamp/validate rating range.
        """
        if rating < 0 or rating > scale_max:
            raise ValueError("Rating out of range")

        with self._lock:
            order = self._orders.get(order_id)
            if order is None:
                raise ValueError("Unknown order")
            if order.rating is not None:
                raise ValueError("Already rated")

            order.rating = rating

            # Update restaurant-level aggregate
            ra = self._restaurant_rating.setdefault(order.restaurant_id, _Agg())
            ra.sum_r += rating
            ra.count += 1

            # Update per-food aggregates for each distinct food on the order
            seen_food_keys: set[str] = set()
            for eid, _qty in order.lines.items():
                entry = self._menu[eid]
                fk = _norm_food(entry.food_name)
                if fk in seen_food_keys:
                    continue
                seen_food_keys.add(fk)
                bucket = self._food_restaurant_rating.setdefault(fk, {})
                fr = bucket.setdefault(order.restaurant_id, _Agg())
                fr.sum_r += rating
                fr.count += 1

    def top_restaurants(self, k: int) -> list[tuple[str, float]]:
        """
        Top-k restaurants by average rating.

        Tie-break: higher count then lexicographic id (deterministic).
        Complexity O(R log R) — mention heap O(R log k) as optimization.
        """
        with self._lock:
            scored: list[tuple[str, float, int]] = []
            for rid, agg in self._restaurant_rating.items():
                if agg.count == 0:
                    continue
                scored.append((rid, agg.mean(), agg.count))
            scored.sort(key=lambda t: (-t[1], -t[2], t[0]))
            return [(rid, m) for rid, m, _c in scored[:k]]

    def top_restaurants_for_food(self, food_name: str, k: int) -> list[tuple[str, float]]:
        """Best average rating among restaurants that serve this food name."""
        key = _norm_food(food_name)
        with self._lock:
            bucket = self._food_restaurant_rating.get(key, {})
            scored = [(rid, agg.mean(), agg.count) for rid, agg in bucket.items() if agg.count]
            scored.sort(key=lambda t: (-t[1], -t[2], t[0]))
            return [(rid, m) for rid, m, _c in scored[:k]]
```

---

## Optimizations

- **Heap** per global leaderboard and per food key for O(log k) top maintenance.
- **Time-decay** ratings with sliding window buckets.
- **Idempotent** `rate_order` with revision history if product needs edits.

---

## Pitfalls

- Rating an order **twice** — must guard.
- Same food name different strings — **normalize**.
- Attributing rating to **every food line** once per order vs per item — clarify with interviewer (above: **once per distinct food** on order).
