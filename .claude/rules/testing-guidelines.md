---
paths:
  - "backend/tests/*.py"
  - "backend/tests/**/*.py"
description: Testing framework and assertion guidelines
---

# Testing Guidelines

- **Read `backend/tests/conftest.py` before writing any test.** Know what fixtures already exist (e.g. `db_manager_fixture`) and use them instead of reinventing setup or mocking their targets. Reaching for a `MagicMock` of something that already has a fixture is the usual sign this step was skipped.
- **Don't mock `DatabaseManager` — use `db_manager_fixture`** (conftest; temp-file SQLite). Assert on the persisted outcome (the getter that reads the row back), not on save-call wiring. A `MagicMock` DB with `side_effect=[...]` lists encodes the exact call order and silently rots when the sequence changes — seed the real DB and read it back instead. Mark heavier DB tests `@pytest.mark.local`. Mocking the network/LLM layer (`_fetch_*`, external clients) is still correct — only the DB should be real.
- Use `pytest_mock` library instead of pytest's `monkeypatch` fixture or `unittest.mock`
- Mark local database tests with `@pytest.mark.local` to exclude them from CI runs
- Shared fixtures and test constants live in `backend/tests/conftest.py`

## Where tests live (mirror the package tree)

`backend/tests/` mirrors `backend/app/`. This is the rule for *where a new test goes* — don't read filenames to guess the mapping.

- **Mirror the source package.** A test for `backend/app/<pkg>/<module>.py` lives at `backend/tests/<pkg>/test_<module>.py` — e.g. `app/services/pricing/vendor_pricing.py` → `tests/services/pricing/test_vendor_pricing.py`. The directory tells you the package; the filename matches the module under test. Never drop a new `test_*.py` at the tests root.
- **No feature-grouping folders that don't exist in `app/`.** Group by the production package, not by concept (no top-level `<feature>/` folders). A test that spans modules goes with the module it primarily exercises.
- **The tests root is reserved for cross-cutting guards** that map to no single module — e.g. `test_no_sync_db_in_async.py`, which scans many source dirs. Everything else nests.
- **Repo-root `scripts/` tools mirror into `backend/tests/scripts/`.** One-off CLI/seed scripts live outside `backend/app/` (e.g. `scripts/seed_vendor_data.py`), so their tests nest in a dedicated `backend/tests/scripts/` subtree (`scripts/<tool>.py` → `backend/tests/scripts/test_<tool>.py`), never at the tests root. The test's `from scripts.<tool> import …` still resolves to the repo-root package, not this mirror.
- **`conftest.py` stays at `backend/tests/conftest.py`.** pytest scopes a `conftest.py`'s fixtures to its own directory and below, so the project-wide fixtures must stay at the root. A subtree may add its own `conftest.py` — but only for real `@pytest.fixture`s local to that subtree, never as a home for plain helper functions imported by `from ...conftest import`.
- **Shared helpers live at the level of the tests they serve.** Co-locate a helper used by one area inside that area's folder (e.g. `core/algos/algo_helpers.py`) — it doesn't need its own folder. Reserve the top-level `support/` package only for helpers that cut across multiple test areas (e.g. `support/extractor_configs.py`, `support/utils.py`). Reusable fixtures and ground-truth data corpora live in `fixtures/`.

## What to Test (and What Not To)

- **Don't test Pydantic itself** — never write tests that just verify Pydantic default values, field constraints (e.g. `ge=0, le=1`), or type coercion. These test the framework, not your code.
- **No trivial set-then-check tests** — don't create an object with values and then assert those same values back (e.g. `score = Score(mode=RELATIVE)` → `assert score.mode == RELATIVE`). These provide zero value.
- **Test business logic and behavior** — focus on custom validators, computed fields, class methods with logic (e.g. `from_alternates`), algorithmic correctness, and integration workflows.
- **Shared test constants** live in `backend/tests/conftest.py` — don't duplicate them across test files.

## How to Assert

- **Avoid string-content assertions on logs, status messages, error text, or violation prose.** Examples to avoid: `assert "value decreases" in violation`, `assert "failed" in caplog.text`, `assert err.args[0].startswith("invalid ")`. They couple tests to human-readable wording rather than behavior, so any reword silently breaks the test on something that isn't a real regression. Prefer asserting on structured fields: enum/`Literal` discriminators, list lengths, dedicated `.failed_for(...)` test helpers, or raised exception **types**. If the structure doesn't exist yet, either add it (when the test value is real) or weaken the assertion to a presence check (e.g. `assert err.violations`) — the test name should document *what* failure mode you're checking.
