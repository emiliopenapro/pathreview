# Contribution Journal — PathReview

## Selected Issue

- **Issue:** [#153 — Faithfulness checker crashes when a context chunk has `text: None`](https://github.com/ascherj/pathreview/issues/153)
- **Tier:** Tier 1 (`good first issue`)
- **Labels:** `bug`, `rag`, `tier-1`, `good first issue`
- **Affected file:** `rag/evaluator/faithfulness_checker.py`
- **Related test:** `tests/unit/test_faithfulness_checker.py::test_none_context_chunk_text` (currently failing)

---

## Problem Summary 

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

### Reproduction confirmed (2026-07-20)

Reproduced locally against the fork's `main` before making any code change.
Environment was intentionally minimal — the module only imports `re` (stdlib)
and `structlog`, so no Docker, database, or heavy AI dependencies were needed:

```
python -m venv .venv
.venv\Scripts\python -m pip install structlog
.venv\Scripts\python -c "from rag.evaluator.faithfulness_checker import FaithfulnessChecker; print(FaithfulnessChecker().check('Knows Python.', [{'text': None}]))"
```

Observed crash (as expected):

```
File "rag/evaluator/faithfulness_checker.py", line 34, in check
    context_text = " ".join([
        chunk.get("text", "") for chunk in context_chunks
    ])
TypeError: sequence item 0: expected str instance, NoneType found
```

The `TypeError` originates at line 34, exactly where `chunk.get("text", "")`
returns `None` for a present-but-null `text` key.

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

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/emiliopenapro/pathreview/commit/d703768

**Reproduction summary:** Ran the project's existing unit test
`test_none_context_chunk_text` in a minimal `pytest` + `structlog` environment; it fails
with `TypeError: sequence item 0: expected str instance, NoneType found` at
`rag/evaluator/faithfulness_checker.py:34`, confirming the reported crash on `{"text": None}`.

**PLAN.md link:** https://github.com/emiliopenapro/pathreview/blob/fix/153-faithfulness-none-context-text/PLAN.md

**Walkthrough video (recommended):** _not recorded (optional / ungraded)_

**Blockers or open questions:** None blocking. Note: 3 other tests in the file fail from a
separate bug (issue #152, the ≥2-overlap threshold) — pre-existing and out of scope for #153.

### Reproduction via the failing unit test (2026-07-26)

Reproduced the bug by running PathReview's own unit test for this checker, in a minimal
`pytest` + `structlog` virtualenv — no Docker and no full `[dev]` install needed, since the
module imports only `re` (stdlib) and `structlog`:

```
python -m venv .venv
.venv\Scripts\python -m pip install pytest structlog
.venv\Scripts\python -m pytest tests/unit/test_faithfulness_checker.py -v -m unit
```

Result: **4 failed, 18 passed.** The target test fails with exactly the documented crash:

```
tests/unit/test_faithfulness_checker.py::TestFaithfulnessChecker::test_none_context_chunk_text FAILED
...
E   TypeError: sequence item 0: expected str instance, NoneType found
rag\evaluator\faithfulness_checker.py:34: TypeError
```

**Important finding — 3 of the 4 failures are NOT #153.**
`test_partial_support_returns_middle_score`, `test_multiple_context_chunks`, and
`test_multiple_claims_varying_support` fail independently because `_is_supported()` requires
≥2 overlapping tokens, so single-keyword matches score `0.0`. That is a *separate* bug —
issue #152 ("Faithfulness checker can never mark short claims as supported") — and is **out
of scope** for this contribution. Only `test_none_context_chunk_text` belongs to #153. This
shapes the Definition of Done (see `PLAN.md` → Risks): fix the one test without introducing
new failures, and leave the pre-existing #152 failures alone.

---

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:** Implemented the fix from PLAN.md on a clean PR branch cut from
`upstream/main` (so the PR diff contains only the change, not my course docs). The one-line
fix in `rag/evaluator/faithfulness_checker.py` coalesces a `None`/absent `text` to `""`
(`chunk.get("text") or ""`). Added two edge-case unit tests (mixed valid+`None`, all-`None`).
Sub-tasks 1–3 from PLAN.md are done; the target `test_none_context_chunk_text` now passes.

**Next steps:** Run the full self-review (`ruff`/`black`/`mypy`/`pytest`), write the PR
description against the repo template, and open the PR.

**Blockers:** None. (`make` isn't available on Windows, so I ran the underlying
`ruff`/`black`/`mypy`/`pytest` commands directly — equivalent to the `make` targets.)

---

### Check-in 2 (end of week)

**PR link:** <PASTE YOUR PR URL HERE AFTER OPENING IT>

**Branch:** `fix/153-handle-none-context-text` (clean PR branch; course docs live on
`fix/153-faithfulness-none-context-text`)

**What you built:** `FaithfulnessChecker.check()` no longer crashes on a context chunk with
`{"text": None}`. The context is now built with `(chunk.get("text") or "")`, so a null chunk
contributes an empty string — identical to how a missing `text` key was already handled — and
`check()` returns a valid score instead of raising `TypeError`.

**Tests added or updated:** `tests/unit/test_faithfulness_checker.py` — added
`test_mixed_valid_and_none_chunk_text` and `test_all_none_context_chunks`; the pre-existing
`test_none_context_chunk_text` now passes.

**Self-review confirmation:**
- [x] make check passes — *no new failures vs. baseline (repo has 182 ruff / 52 black / 5 mypy
  pre-existing errors, all unrelated and documented in the PR)*
- [x] make test-unit passes — *unit suite went 53 failed/375 passed → 52 failed/378 passed:
  one failure fixed, two new passing tests, zero new failures. Remaining failures are
  pre-existing (incl. the #152 trio)*

**Draft PR feedback received from:** none (opened at submission time)
