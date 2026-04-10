# 3. Design File System (`mkdir`, `pwd`, `cd` with `*`)

**Typical round:** 45–60 minutes · **Focus:** tree structure, path parsing, wildcard semantics.

---

## Correct interview flow (this document)

Same order as **README**. Do **not** open with “I’ll use a **tree**…” until **after** wildcard and path rules are agreed; then **FR/NFR → API →** choose tree + parent pointers.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“We’re building **mkdir**, **pwd**, **cd** with a special `*` segment. Before I choose a representation: is `*` exactly **one path segment** matching **one** child name? If several children could match, do we **fail**, pick **deterministically**, or **search**? How does **`..`** behave at **root**? Should **mkdir** create **intermediate** dirs (`mkdir -p`)? **Only directories**—no files?”

**Pause.** Then:

“**Invariant**: **`cwd` always points to a valid directory**. Next I’ll do **FR/NFR**, put the **three commands** on the board, **then** I’ll use a **tree** with **parent pointers** for `pwd` and segment resolution for `cd`.”

### Flow — cover in this order

1. **Open / clarify** — as above + **Clarifying questions** table.  
2. **FR / NFR** — section below.  
3. **Invariant** — valid `cwd`; deterministic `*` if required.  
4. **Entities** — `DirNode`, shell (`root`, `cwd`) **after** API.  
5. **APIs on board** — `mkdir`, `pwd`, `cd`.  
6. **Data structures** — **now** tree + `dict` children + **parent** for `pwd`.  
7. **Code** — **`mkdir`**, **`cd`** (`*` backtracking), **`pwd`**.  
8. **Edge cases** — empty path, `/`, ambiguous `*`, missing segment.  
9. **Complexity** — `pwd` O(depth); honest cost for `*`.  
10. **Scale / tests** — in-memory; `..`, `/a/b`, `*`.

### Natural phrases

- “I’ll **tokenize** the path and resolve segment by segment.”  
- “For `*`, I’ll **try children in sorted order** so behavior is deterministic—that’s easy to explain and test.”  
- “`pwd` is **walk up to root** with parent pointers, then reverse.”

### APIs on the board

`mkdir` · `pwd` · `cd` → bool.

### Must-code (2–3)

1. **`mkdir`**  
2. **`cd`** (including `*` handling)  
3. **`pwd`** or `_resolve_parts` if they want the core recursive piece.

### Closing

“This matches an **in-memory shell**; if we added **rename** or **symlinks**, I’d revisit how `pwd` and `cd` resolve and whether we need a **visited set** for cycles.”

### Mental checklist

Wildcard agreed · FR/NFR · APIs before tree · Tree + `mkdir` / `cd` / `pwd` · Edges · Complexity · Tests.

---

## After alignment (before coding — structure teaser)

> “I’ll represent dirs as a **tree** with **parent pointers** so `pwd` walks up to root; children in a **map** for fast lookup by segment.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Is `*` one **segment** only (e.g. `/a/*/c`)? | Classic problem |
| Multiple matches — **error**, **first**, or **all**? | Usually deterministic pick or fail |
| Only directories? | Yes for `cd` |
| Root path `/` behavior | Parent of root stays root or error |

---

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** naming these after clarification; then tree + path logic.

### Functional requirements (FR)

- **`mkdir`** — create directory path (create parents as agreed, e.g. `mkdir -p` style).
- **`pwd`** — print absolute path of current working directory.
- **`cd`** — change cwd; support `.`, `..`, absolute/relative paths, and segment **`*`** per spec.

**What you can say:** “**FRs**: **mkdir**, **pwd**, **cd** with the wildcard rules we just aligned on.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | `cwd` always valid; paths resolve deterministically | “**Invariant**: cwd is always a real node; `*` behavior is **defined**.” |
| **Performance** | `pwd` and single-segment `cd` cheap | “Tree walk is **O(depth)**; worst-case `*` is branchy—I’ll say that.” |
| **Memory** | In-memory tree only | “No disk; children in a **map** per node.” |

**What you can say:** “**NFRs**: correct resolution, predictable cost for normal paths, honest complexity for wildcards.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`DirNode`** | `name`, `parent`, `children: dict[str, DirNode]`. |
| **`FileSystemShell`** | `root`, `cwd`; implements `mkdir` / `pwd` / `cd`. |

**Relationships:** Shell **references** root and cwd; nodes form a **tree** via parent + children.

**What you can say:** “**Entities**: **directory nodes** in a tree, plus a **shell** that tracks cwd.”

### API design

| Method | Purpose |
|--------|---------|
| `mkdir(path: str) -> None` | Create path. |
| `pwd() -> str` | Return `/` or `/a/b/...`. |
| `cd(path: str) -> bool` | Return **False** if path invalid / ambiguous per spec. |

**What you can say:** “**Public API** is exactly the three commands; path parsing stays **private**.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → tree + implementation** (same as **README** table).

---

## Approach

- **Node:** `name`, `parent`, `children: dict[str, Node]`.
- **Current:** pointer to node; `pwd` prints path by walking up with parent pointers (or maintain path stack).
- **`mkdir`:** walk/create from root.
- **`cd`:** tokenize by `/`; for each segment, if `*` then **enumerate children** and try each candidate (DFS/backtracking) or if only one child exists pick it; if ambiguous, **fail** per spec.

**Optimization:** cache nothing for interview unless asked; **path string** from parent chain is O(depth) per `pwd`.

---

## Wildcard `*` algorithm (common spec)

For segment `*`:

1. If current node has **exactly one** child directory → optional auto-match (only if interviewer says).
2. Otherwise **try each child** name in sorted order for deterministic behavior when multiple paths exist (interviewer-dependent).
3. **Backtrack** if later segments fail.

---

## Reference implementation (Python)

```python
"""
In-memory hierarchical file system with mkdir, pwd, and cd.

Wildcard:
- A path segment equal to '*' matches any single direct child directory name.
- If multiple children exist, we try names in sorted order and use the first
  full path that allows the remainder of the path to resolve (backtracking).
  Adjust if your interviewer wants "fail if ambiguous" without search.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Optional


@dataclass
class DirNode:
    """Directory node; files are out of scope for this problem variant."""

    name: str
    parent: Optional[DirNode] = None
    # child_name -> subdirectory node
    children: dict[str, DirNode] = field(default_factory=dict)


class FileSystemShell:
    def __init__(self) -> None:
        # Root has empty name; path printing treats root specially
        self._root = DirNode(name="")
        self._cwd = self._root

    # ------------------------------------------------------------------
    # mkdir: create all intermediate directories (like mkdir -p semantics
    # unless interviewer says fail if parent missing)
    # ------------------------------------------------------------------
    def mkdir(self, path: str) -> None:
        """
        Absolute path like '/a/b/c' — leading slash optional in some specs;
        here we normalize to parts.
        """
        node = self._root
        parts = self._split_path(path)
        for part in parts:
            if part not in node.children:
                child = DirNode(name=part, parent=node)
                node.children[part] = child
            node = node.children[part]

    def pwd(self) -> str:
        """Return absolute path of cwd. Root is '/'."""
        if self._cwd is self._root:
            return "/"
        parts: list[str] = []
        cur: Optional[DirNode] = self._cwd
        while cur is not None and cur is not self._root:
            parts.append(cur.name)
            cur = cur.parent
        parts.reverse()
        return "/" + "/".join(parts)

    def cd(self, path: str) -> bool:
        """
        Change directory. Returns False if path cannot be resolved.

        Supports:
        - Absolute paths starting with '/'
        - Relative paths from cwd
        - '.' and '..' (parent of root stays root)
        - '*' wildcard per segment (see module docstring)
        """
        start = self._root if path.startswith("/") else self._cwd
        parts = self._split_path(path)
        resolved = self._resolve_parts(start, parts, 0)
        if resolved is None:
            return False
        self._cwd = resolved
        return True

    def _split_path(self, path: str) -> list[str]:
        """Split on '/' and drop empty segments from leading/trailing slashes."""
        raw = path.split("/")
        return [p for p in raw if p != "" and p != "."]

    def _resolve_parts(
        self, node: DirNode, parts: list[str], index: int
    ) -> Optional[DirNode]:
        """
        Recursively resolve path parts from `node`.

        For '..', move to parent (root's parent is root).
        For '*', try each subdirectory name in sorted order (deterministic).
        """
        if index == len(parts):
            return node

        token = parts[index]

        if token == "..":
            parent = node.parent if node.parent is not None else self._root
            return self._resolve_parts(parent, parts, index + 1)

        if token != "*":
            if token not in node.children:
                return None
            return self._resolve_parts(node.children[token], parts, index + 1)

        # Wildcard segment: branch over all child directories
        for name in sorted(node.children.keys()):
            child = node.children[name]
            got = self._resolve_parts(child, parts, index + 1)
            if got is not None:
                return got
        return None
```

---

## Optimizations & trade-offs

- **Path cache** on each node for O(1) `pwd` — more memory, must invalidate on move/rename (if added).
- **Ambiguity:** some interviewers want `cd` to **fail** if `*` matches more than one **valid completion**; then try all but succeed only if **exactly one** success.

---

## Testing angles

- `mkdir /a/b`, `cd /a`, `pwd` → `/a`
- `cd ..` from `/a/b` → `/a`
- `mkdir /x/y`, `cd /x/*/y` resolves if unique
- Two siblings `a/1/c` and `a/2/c` — behavior of `cd /a/*/c` (define with interviewer)
