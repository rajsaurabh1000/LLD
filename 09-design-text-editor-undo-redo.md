# 9. Design Text Editor (Rows, Insert/Delete, Undo / Redo)

**Typical round:** 45–60 minutes · **Focus:** command pattern, two stacks, row invariants.

---

## Correct interview flow (this document)

Same order as **README**. Do **not** open with **Command pattern** / **two stacks** until **document rules** (rows, no `\n`, delete semantics) are clear.

---

## Interview script & checklist (human speaking)

### Opening (clarify-first)

“Document is **rows** of strings—**no newlines inside** a cell? **0-based** row/col? If **delete** goes past end of line—**clamp** or **error**? Is a **cursor** in scope or only explicit ops? **Undo/redo**—unbounded stacks OK?”

**Pause.** Then:

“**Invariant**: **undo** truly reverses **apply**; **new edit clears redo**. I’ll state **FR/NFR**, write **insert/delete/undo/redo** on the board, **then** use **commands** and **two stacks**.”

### Flow — cover in this order

1. **Open / clarify** — as above + table.  
2. **FR / NFR** — section below.  
3. **Invariant** — reversible edits; redo cleared on new edit.  
4. **Entities** — document, commands, editor **after** API.  
5. **APIs on board** — `insert`, `delete`, `undo`, `redo`.  
6. **Data structures** — `list[str]` + **undo/redo stacks** of commands.  
7. **Code** — **insert/delete** + **undo/redo**.  
8. **Edge cases** — OOR index, empty undo.  
9. **Complexity** — O(len) splice per line.  
10. **Scale / tests** — rope if huge; undo chain test.

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

Clarified · FR/NFR · APIs before Command pattern · Two stacks · Code · Tests.

---

## After alignment

> “Each edit is a **command** with **apply** and **revert**; **undo** and **redo** stacks; **new mutation clears redo**.”

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

## FR, NFR, core entities & API (say this for SDE2)

Spend **1–2 minutes** on **command pattern** and stack rules; then implement.

### Functional requirements (FR)

- **Insert** text at **(row, col)** on one line (no `\n` in inserted text if that’s the rule).
- **Delete** **length** characters at **(row, col)**.
- **Undo** last applied edit; **redo** last undone edit; **new edit clears redo**.

**What you can say:** “**FRs**: per-row insert/delete plus **undo/redo** with standard stack semantics.”

### Non-functional requirements (NFR)

| Area | Typical NFR | One line |
|------|-------------|----------|
| **Correctness** | Document always valid rows; undo truly reverses | “Each command implements **apply** and **revert**.” |
| **Performance** | String ops O(length) per line | “**Rope** only if they care about huge lines.” |
| **Memory** | Bounded undo depth (if they ask) | “Optional cap on stack size.” |

**What you can say:** “**NFRs**: reversible edits, honest cost of string concat.”

### Core entities

| Entity | Responsibility |
|--------|----------------|
| **`DocumentState`** | `list[str]` lines. |
| **`Command`** (ABC) | `apply(doc)`, `revert(doc)` — `InsertCommand`, `DeleteCommand`. |
| **`TextEditor`** | Holds doc + **undo** / **redo** stacks. |

**Relationships:** Editor **applies** commands to document; stacks **store** command history.

**What you can say:** “**Entities**: **document**, **command** objects, **editor** with two stacks.”

### API design

| Method | Purpose |
|--------|---------|
| `insert(row, col, text)`, `delete(row, col, length)` | Mutations (push undo, clear redo). |
| `undo() -> bool`, `redo() -> bool` | Return false if nothing to do. |

**What you can say:** “**Public API** is four operations; commands stay **internal**.”

### Order in the interview

**Clarify → FR / NFR → invariant → entities → API → DS + code** (see **README**).

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
