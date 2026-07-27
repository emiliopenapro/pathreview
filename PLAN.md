## Solution plan

**Issue:** [#153 — Faithfulness checker crashes when a context chunk has `text: None`](https://github.com/ascherj/pathreview/issues/153)

### Understand

The RAG faithfulness checker (`rag/evaluator/faithfulness_checker.py`) scores how well
generated feedback is supported by retrieved context. In `check()` it concatenates chunk
text with:

```python
context_text = " ".join([chunk.get("text", "") for chunk in context_chunks])
```

**Root cause:** `dict.get(key, default)` returns the default only when the key is *absent*.
A chunk shaped like `{"text": None}` has the key present with value `None`, so `.get()`
returns `None` — not `""`. `" ".join([None])` then raises
`TypeError: sequence item 0: expected str instance, NoneType found`.

**Expected vs. actual:**
- *Expected:* a `None`-text chunk contributes no context and `check()` returns a valid
  `float` in `[0.0, 1.0]` — the same graceful behavior a *missing* `text` key already gets
  (`test_missing_text_key_in_chunk` passes today).
- *Actual:* `check()` raises `TypeError` and the entire faithfulness check crashes.

### Map

- **`rag/evaluator/faithfulness_checker.py`** — `FaithfulnessChecker.check()`, the join at
  **line 34**. This is the only file I expect to change.
- **`tests/unit/test_faithfulness_checker.py`** — `test_none_context_chunk_text` already
  asserts the correct behavior (returns a float in range). No change needed; it is the
  Definition of Done.

### Plan

1. In `check()`, coalesce a `None` text value to an empty string:
   `context_text = " ".join([(chunk.get("text") or "") for chunk in context_chunks])`.
2. Run `pytest tests/unit/test_faithfulness_checker.py -m unit` and confirm
   `test_none_context_chunk_text` flips **FAIL → PASS**.
3. Confirm the 18 currently-passing tests in that file still pass (no new failures).
4. Run `make check` (lint + format + typecheck) to satisfy contribution standards.
5. Commit on branch `fix/153-faithfulness-none-context-text` as
   `fix(rag): handle None text in faithfulness context chunks`.

### Inputs & outputs

- **Input:** `context_chunks: list[dict]` where a chunk's `"text"` may be a string, absent,
  or `None`.
- **Output:** unchanged public contract — a `float` faithfulness score in `[0.0, 1.0]`. A
  `None`-text chunk contributes `""`, so it behaves identically to a missing `text` key.
- No change to the method signature, return type, claim extraction, or scoring logic.

### Risks & unknowns

- **KNOWN TRAP — pre-existing unrelated failures.** Running the test file today yields
  **4 failures, not 1**. Besides `test_none_context_chunk_text`, these fail:
  `test_partial_support_returns_middle_score`, `test_multiple_context_chunks`,
  `test_multiple_claims_varying_support`. They fail because `_is_supported()` requires
  **≥2** overlapping tokens, so single-keyword matches score `0.0`. That is a *different*
  bug — **issue #152** ("Faithfulness checker can never mark short claims as supported") —
  and is **out of scope** here. Risk: a naive "all unit tests green" check would flag these.
  *Mitigation:* my Definition of Done is FAIL→PASS on `test_none_context_chunk_text` with
  **no new** failures; the three #152 failures are documented as pre-existing.
- `chunk.get("text") or ""` also maps a legitimate `""` to `""` (no-op) and any falsy
  non-string (e.g. `0`) to `""`. Acceptable — `text` is expected to be a string.
- Full-suite `make test-unit` may import heavier modules than this file needs; if the
  Python 3.14 environment cannot install all `[dev]` deps, verify via the targeted file run
  (only `pytest` + `structlog` are required for this test).

### Edge cases

- `{"text": None}` → contributes `""`, no crash (the target case).
- `{}` (missing key) → already `""`; must stay working.
- `{"text": ""}` → `""`, unchanged.
- `[{"text": "Python"}, {"text": None}]` → only the valid chunk contributes.
- All chunks `None`/empty → context is `""`, no claim supported → score `0.0` (valid float).
