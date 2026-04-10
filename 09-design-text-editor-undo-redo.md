# 9. Design Text Editor (Rows, Insert/Delete, Undo / Redo)

**Typical round:** 45–60 minutes · **Focus:** command pattern, two stacks, row invariants.

---

## Interview script & checklist (human speaking)

### Opening

“The document is **rows of strings** with **no embedded newlines**. I’ll use the **Command pattern**: each edit knows how to **apply** and **undo** itself. **Undo** and **redo** are two stacks; after a new edit, I **clear redo**—that’s standard editor behavior. I’ll confirm delete **clamps** past end-of-line vs throws, and that indices are **0-based** as in the spec.”

### Flow — cover in this order

1. **Clarify** — Unicode code points vs bytes (Python `str`), clamping rules, cursor in scope or not.  
2. **Invariants** — after any operation, `lines` is a list of strings with no `\n`; undo restores prior state.  
3. **Entities** — `DocumentState`, `Command` interface, `TextEditor` with stacks.  
4. **APIs** — `insert(row, col, text)`, `delete(row, col, length)`, `undo()`, `redo()`.  
5. **Data structures** — `list[str]`; **two stacks** of commands.  
6. **Code** — **`InsertCommand` / `DeleteCommand`** (or inline) + **`undo`/`redo`**.  
7. **Edge cases** — out-of-range row/col, delete length 0, undo empty.  
8. **Complexity** — O(len) string splice per op on a line; rope if huge lines.  
9. **Scale** — usually local; mention **operation log** for persistence if asked.  
10. **Testing** — insert then undo; delete capture for redo path.

### Natural phrases

- “**Delete** needs to **remember what it removed** so undo can put it back.”  
- “Any **new edit** after undo **invalidates** the redo branch—that’s what users expect.”

### APIs on the board

`insert` · `delete` · `undo` · `redo`.

### Must-code (2–3)

1. **`insert` + undo path** (or `InsertCommand`)  
2. **`delete` + undo path** (snapshot removed substring)  
3. **`undo` and `redo`** stack discipline.

### Closing

“If files get huge, I’d swap the line storage for a **rope** or **piece table**; the **command stacks** stay the same.”

### Mental checklist

Command pattern · Two stacks · Clear redo on edit · APIs · Code · Edges · Complexity · Tests.

---

## Interview opener

> “The document is **list of lines** (strings without embedded newlines). Operations are **per row**: insert characters at column, delete a run of columns. I’ll use the **Command pattern** with **undo** and **redo** stacks; each command implements `execute` and `undo` mutating the document. I’ll confirm whether **cursor** is in scope or only operations given explicitly.”

---

## Clarifying questions

| Question | Why |
|----------|-----|
| UTF-8 codepoints vs bytes? | Python str is Unicode codepoints — state that |
| Delete past end? | Clamp or error |
| Max rows / line length? | Memory |
| Undo across **no-ops**? | Stack policy |
| **Batch** macro undo? | Optional `CompositeCommand` |

---

## Approach

- **Document:** `list[str]` — each row grows/shrinks.
- **Command interface:** `apply(doc)`, `revert(doc)`.
- **Stacks:** `undo_stack`, `redo_stack`; successful `apply` pushes to undo and **clears redo** (standard).

**Commands**

- `InsertText(row, col, text)`
- `DeleteRange(row, col, length)` — snapshot deleted substring for undo

---

## Complexity

- Each op: O(len(text)) for insert/delete due to string splice; mention **rope** if they want O(log n) for huge lines.

---

## Reference implementation (Python)

```python
"""
Row-based text editor with undo/redo.

Design:
- DocumentState holds lines: list[str]
- Each operation is an object with apply() and revert()
- Undo stack stores executed commands; redo stack stores undone commands

Clearing redo:
- Any new user edit after undo should drop redo history (standard editor behavior).
"""

from __future__ import annotations

from abc import ABC, abstractmethod
from threading import Lock
from typing import List


class DocumentState:
    """Mutable model; commands operate on this structure."""

    def __init__(self) -> None:
        # Starts with zero rows per problem statement
        self.lines: List[str] = []

    def _ensure_row(self, row: int) -> None:
        """Grow trailing empty rows if needed so lines[row] exists."""
        while len(self.lines) <= row:
            self.lines.append("")

    def insert(self, row: int, col: int, text: str) -> None:
        """Insert `text` at (row, col); pad rows with empty strings."""
        if "\n" in text:
            raise ValueError("Newlines not allowed inside inserted text")
        self._ensure_row(row)
        line = self.lines[row]
        if col < 0 or col > len(line):
            raise IndexError("Column out of range")
        self.lines[row] = line[:col] + text + line[col:]

    def delete(self, row: int, col: int, length: int) -> str:
        """
        Delete `length` chars starting at col; return deleted substring for undo capture.

        If length extends past EOL, clamp (interviewer may prefer strict error).
        """
        self._ensure_row(row)
        line = self.lines[row]
        if col < 0 or col > len(line):
            raise IndexError("Column out of range")
        end = min(len(line), col + length)
        removed = line[col:end]
        self.lines[row] = line[:col] + line[end:]
        return removed


class Command(ABC):
    @abstractmethod
    def apply(self, doc: DocumentState) -> None:
        raise NotImplementedError

    @abstractmethod
    def revert(self, doc: DocumentState) -> None:
        raise NotImplementedError


class InsertCommand(Command):
    def __init__(self, row: int, col: int, text: str) -> None:
        self.row = row
        self.col = col
        self.text = text

    def apply(self, doc: DocumentState) -> None:
        doc.insert(self.row, self.col, self.text)

    def revert(self, doc: DocumentState) -> None:
        # Undo insert == delete same length at same position
        doc.delete(self.row, self.col, len(self.text))


class DeleteCommand(Command):
    def __init__(self, row: int, col: int, length: int) -> None:
        self.row = row
        self.col = col
        self.length = length
        self._removed: str = ""

    def apply(self, doc: DocumentState) -> None:
        self._removed = doc.delete(self.row, self.col, self.length)

    def revert(self, doc: DocumentState) -> None:
        # Undo delete == re-insert removed text
        if self._removed:
            doc.insert(self.row, self.col, self._removed)


class TextEditor:
    def __init__(self) -> None:
        self._doc = DocumentState()
        self._undo: list[Command] = []
        self._redo: list[Command] = []
        self._lock = Lock()

    def _run(self, cmd: Command) -> None:
        """Execute command and push to undo; clear redo per new edit."""
        cmd.apply(self._doc)
        self._undo.append(cmd)
        self._redo.clear()

    def insert(self, row: int, col: int, text: str) -> None:
        with self._lock:
            self._run(InsertCommand(row, col, text))

    def delete(self, row: int, col: int, length: int) -> None:
        with self._lock:
            self._run(DeleteCommand(row, col, length))

    def undo(self) -> bool:
        """
        Pop last command, revert it, push to redo. Returns False if nothing to undo.
        """
        with self._lock:
            if not self._undo:
                return False
            cmd = self._undo.pop()
            cmd.revert(self._doc)
            self._redo.append(cmd)
            return True

    def redo(self) -> bool:
        """Inverse of undo."""
        with self._lock:
            if not self._redo:
                return False
            cmd = self._redo.pop()
            cmd.apply(self._doc)
            self._undo.append(cmd)
            return True

    def get_lines(self) -> list[str]:
        """Snapshot for display/tests (copy to avoid external mutation)."""
        with self._lock:
            return list(self._doc.lines)
```

---

## Optimizations

- **Rope / piece table** for large files O(log n) edits.
- **Compound command** for “replace selection” = delete + insert one undo step.
- **Persistent data structures** for unlimited undo with sharing (advanced).

---

## Pitfalls

- **Off-by-one** on column after insert/delete.
- Forgetting to **clear redo** on new edits.
- `DeleteCommand.apply` called twice without reset — store removed text only once (pattern above captures on first apply).
