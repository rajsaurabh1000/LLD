# 4. Design Leaderboard (Fantasy Sports, Top-K by Team Score)

**Typical round:** 45–60 minutes · **Focus:** incremental score updates, reverse index player → teams, Top-K query.

---

## Interview opener

> “Each user has **one team** of players. User score = **sum of current player points**. When a player’s points change, I need to **update every user** who owns that player. For Top-K users, I’ll keep a **heap** or **sorted structure** of scores; with many updates, a **lazy** or **tree map** approach works. In Python I’ll use `heapq` with **lazy deletion** or rescan if K is small.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| **Unique** users and players? | IDs |
| Player points **delta** or set? | Update API |
| **Top-K** — ties broken how? | Stable ordering by user_id |
| K **fixed** or per query? | API |
| Need **rank of one user**? | Fenwick / order-stat if they ask |

---

## Approach

1. **Structures**
   - `user_teams[user_id] → set[player_id]`
   - `player_score[player_id] → int`
   - `user_score[user_id] → int` (denormalized sum for O(1) read)
2. **Update player:** for each user containing player, `old = user_score[u]`; adjust; push change into ranking structure.
3. **Top-K:** 
   - **Small K, frequent updates:** max-heap of size N is messy on score change; better **SortedList** (if allowed) keyed by `(-score, user_id)`.
   - **Interview-safe:** `heapq.nlargest(K, ...)` on each query is O(N log K) — state O(N) users acceptable for small N; for large N discuss **indexed tree** or **bucket by score** if scores bounded.

**Say explicitly:** “For coding in 30 minutes I’ll implement Top-K as **sorting a snapshot** O(N log N) or **heap** O(N log K); production would use a balanced BST or score buckets.”

---

## Reference implementation (Python)

```python
"""
Leaderboard: users have fixed teams of players; user score = sum(player points).

This version optimizes for clarity + correct incremental updates:
- player_containers: reverse index from player -> users who drafted them
- user_score: denormalized total for O(1) access

Top-K:
- Simple approach: sort users by (-score, user_id) each query — O(U log U).
- Documented alternative: maintain heap with lazy updates for heavy read load.
"""

from __future__ import annotations

from collections import defaultdict
from threading import Lock
from typing import DefaultDict, Iterable


class Leaderboard:
    def __init__(self) -> None:
        # player_id -> current points
        self._player_score: dict[str, int] = {}
        # user_id -> set of player_ids on their team
        self._user_team: dict[str, set[str]] = {}
        # user_id -> cached total score
        self._user_score: dict[str, int] = {}
        # player_id -> users who have this player (reverse index for fast propagation)
        self._player_to_users: DefaultDict[str, set[str]] = defaultdict(set)
        self._lock = Lock()

    def register_user_team(self, user_id: str, player_ids: Iterable[str]) -> None:
        """
        One team per user (problem statement). If called twice, define behavior:
        here we replace the team and rebuild reverse index contributions.
        """
        player_ids = set(player_ids)
        with self._lock:
            # Remove old reverse-index edges
            if user_id in self._user_team:
                for p in self._user_team[user_id]:
                    self._player_to_users[p].discard(user_id)
                    if not self._player_to_users[p]:
                        del self._player_to_users[p]

            self._user_team[user_id] = set(player_ids)
            total = 0
            for p in player_ids:
                pts = self._player_score.get(p, 0)
                total += pts
                self._player_to_users[p].add(user_id)
            self._user_score[user_id] = total

    def update_player_points(self, player_id: str, new_points: int) -> None:
        """
        Set player's absolute points and push delta to all affected users.

        Time: O(number of users who own this player) — expected small vs total users.
        """
        with self._lock:
            old = self._player_score.get(player_id, 0)
            delta = new_points - old
            self._player_score[player_id] = new_points

            if player_id not in self._player_to_users:
                return

            for uid in list(self._player_to_users[player_id]):
                self._user_score[uid] = self._user_score.get(uid, 0) + delta

    def top_k(self, k: int) -> list[tuple[str, int]]:
        """
        Return up to k users with highest scores.
        Tie-break: lexicographic user_id ascending for deterministic output.

        Complexity: O(U log U) — acceptable for interview if U is modest.
        Mention: for huge U and small K with hot updates, use treap / sorted map / buckets.
        """
        if k <= 0:
            return []
        with self._lock:
            # Sort by (-score, user_id) so higher score wins; ties broken by user_id
            ranked = sorted(
                self._user_score.items(),
                key=lambda t: (-t[1], t[0]),
            )
            return ranked[:k]
```

---

## Optimizations (discussion)

- **Score buckets** if scores are small integers: O(1) bump user between buckets.
- **Heap with lazy delete:** store `(score, user_id)`; on update, push new tuple; pop stale until top is current (compare with `user_score` map).
- **Sharding** by user for parallel updates; **eventual consistency** for display-only leaderboard.

---

## Pitfall to mention

Updating only `user_score` without **reverse index** makes `update_player_points` O(users) scan — bad at scale.
