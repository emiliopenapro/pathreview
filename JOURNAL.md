# Contribution Journal — PathReview

## Selected Issue

- **Issue:** [#153 — Faithfulness checker crashes when a context chunk has `text: None`](https://github.com/ascherj/pathreview/issues/153)
- **Tier:** Tier 1 (`good first issue`)
- **Labels:** `bug`, `rag`, `tier-1`, `good first issue`
- **Affected file:** `rag/evaluator/faithfulness_checker.py`
- **Related test:** `tests/unit/test_faithfulness_checker.py::test_none_context_chunk_text` (currently failing)

---

## Problem Summary (in my own words)

The RAG faithfulness checker scores how well a piece of generated feedback is
supported by the retrieved context chunks. Inside its `check()` method it builds
one big context string by joining the `"text"` field of every chunk:

```python
context_text = " ".join([chunk.get("text", "") for chunk in context_chunks])
```

The intent of `chunk.get("text", "")` is "use the chunk's text, or an empty
string if there isn't any." But `dict.get(key, default)` only falls back to the
default when the key is **missing** — not when the key is present with a value of
`None`. So a chunk shaped like `{"text": None}` returns `None`, and
`" ".join([None])` raises:

```
TypeError: sequence item 0: expected str instance, NoneType found
```

In short: the code defends against a *missing* `text` key but not against a
`text` key that is explicitly `None`, so a single null chunk crashes the whole
faithfulness check instead of being handled gracefully.

### Reproduction

```python
from rag.evaluator.faithfulness_checker import FaithfulnessChecker

FaithfulnessChecker().check("Knows Python.", [{"text": None}])
# TypeError: sequence item 0: expected str instance, NoneType found
```

### Root cause

`dict.get("text", "")` returns `None` (not `""`) when the key exists with a
`None` value. The default argument only applies to *absent* keys. The join then
receives a `None` element and raises `TypeError`.

### Expected behavior

The existing failing test defines "done": the checker should degrade gracefully
and still return a valid score rather than crashing.

```python
score = checker.check("Has Python skills", [{"text": None}])
assert isinstance(score, float)
assert 0.0 <= score <= 1.0
```

A `None`-text chunk should simply contribute no context (an empty string), the
same as a missing key would.

---

## Scope Reasoning (issue-fit checklist)

Why this issue is well-scoped for a first contribution:

- **Single file, single logical change.** The fix lives entirely in
  `rag/evaluator/faithfulness_checker.py`; no cross-module coordination.
- **The Definition of Done already exists.** A failing unit test
  (`test_none_context_chunk_text`) ships with the issue, so success is objective:
  make it pass without breaking the rest of the suite.
- **No infrastructure required.** The fix is pure Python logic verifiable with
  `pytest tests/unit -v -m unit` (~30s). It does **not** need Docker, Postgres,
  Redis, or the vector DB — which keeps local setup minimal and avoids the
  multi-service environment overhead.
- **Unambiguous reproduction.** The bug is a hard crash with a one-line repro, so
  there's no guessing about whether it's actually broken.
- **Low blast radius.** Coalescing `None` to `""` changes behavior only for the
  previously-crashing input; well-formed chunks are unaffected.

What's intentionally **out of scope** for this contribution:

- Guarding against chunks that aren't dicts, or a `text` value that is some other
  non-string, non-`None` type (e.g. an int). The issue is specifically about
  `text: None`.
- Any change to scoring, claim extraction, stop-word filtering, or the
  `_is_supported()` logic.

---

## Planned Approach (high level — implementation is next phase)

Treat a `None` text value the same as a missing one by coalescing to an empty
string:

```python
context_text = " ".join([(chunk.get("text") or "") for chunk in context_chunks])
```

`chunk.get("text") or ""` handles all three cases — key present with a string,
key missing, and key present but `None` — collapsing the last two to `""`.

**Verification plan:**
1. Reproduce the crash on `main` first (confirm the `TypeError`).
2. Apply the fix on branch `fix/153-faithfulness-none-context-text`.
3. Run `make test-unit` and confirm `test_none_context_chunk_text` passes and no
   other unit tests regress.

---

## Contribution Metadata (per `docs/CONTRIBUTING.md`)

- **Branch:** `fix/153-faithfulness-none-context-text`
- **Commit (Conventional Commits):** `fix(rag): handle None text in faithfulness context chunks`
