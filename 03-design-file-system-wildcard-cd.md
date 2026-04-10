# 3. Design File System (`mkdir`, `pwd`, `cd` with `*`)

**Typical round:** 45–60 minutes · **Focus:** tree structure, path parsing, wildcard semantics.

---

## Interview opener

> “I’ll model directories as a **tree** of nodes; each node has a name and children map. `pwd` tracks the **current node** from root. `mkdir` creates missing segments. For `cd`, I’ll parse absolute vs relative paths; the only special piece is `*` as a **segment wildcard** matching **exactly one** child name—I’ll confirm whether `*` can match multiple characters across segments or only ‘any single directory name’ in one segment.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| Is `*` one **segment** only (e.g. `/a/*/c`)? | Classic problem |
| Multiple matches — **error**, **first**, or **all**? | Usually deterministic pick or fail |
| Only directories? | Yes for `cd` |
| Root path `/` behavior | Parent of root stays root or error |

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
