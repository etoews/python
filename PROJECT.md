# Python Project Best Practices

Opinionated, single-path conventions for Python projects on this machine. Prescriptive — fork any section if your project genuinely needs otherwise. Machine-level setup (uv install, global config, venv enforcement) lives in **MAC.md**; this file is the next layer up: how to structure and run an individual project.

## Stack

| Purpose | Tool | Install |
|---|---|---|
| Deps & envs | **uv** | `brew install uv` (see MAC.md) |
| Lint + format | **ruff** | `uv add --dev ruff` |
| Tests | **pytest** | `uv add --dev pytest pytest-cov` |
| Type check | **ty** | `uv add --dev ty` |
| Logging | stdlib `logging` | built-in |

ty is pre-1.0. If it blocks you on a legitimate typing pattern, fall back to mypy for that project (`uv add --dev mypy`) and move on.

---

## 1. Project structure

**Use the `src/` layout.** It forces tests to import the *installed* package, which catches packaging mistakes early (missing files, wrong `package_data`, import-path bugs) — you will not discover these at `pip install` time on a user's machine if you skip this.

```
myproj/
├── pyproject.toml
├── uv.lock
├── .python-version
├── README.md
├── .gitignore
├── src/
│   └── myproj/
│       ├── __init__.py
│       ├── __main__.py        # for `python -m myproj`
│       ├── exceptions.py
│       └── _logging.py
└── tests/
    ├── conftest.py
    └── test_something.py
```

Rules:
- No top-level `__init__.py`. Nothing importable lives outside `src/`.
- `tests/` is **not** a package — no `__init__.py`. pytest discovers by path.
- Mirror package structure under `tests/` once you have >1 module.
- One package per repo. Monorepos with multiple packages are a separate pattern not covered here.

**`.gitignore`** (minimum):
```
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
.coverage
dist/
build/
*.egg-info/
```

**Commit:** `pyproject.toml`, `uv.lock`, `.python-version`. **Gitignore:** `.venv/`.

---

## 2. `pyproject.toml`

Single source of truth for metadata, build, dependencies, and tool config. See the full template in §10a.

**Sections:**
- `[project]` — name, version, `requires-python`, `dependencies`, authors, description, license
- `[build-system]` — `hatchling` (uv init's default; fine for 99% of projects)
- `[dependency-groups]` — dev deps go here (PEP 735). `uv add --dev foo` writes to `dependency-groups.dev`.
- `[project.optional-dependencies]` — user-facing extras (`pip install myproj[redis]`). Different from dev groups.
- `[tool.ruff]`, `[tool.pytest.ini_options]`, `[tool.ty]` — tool config

**Dependency pinning:**
- **Lower bound only** in `pyproject.toml`: `requests>=2.32`.
- Upper bound is `uv.lock`'s job. Don't write `==` pins in `pyproject.toml` — it breaks downstream consumers.
- Exception: pin exactly when a known incompatibility exists; add a one-line comment with the reason.

**Versioning:** Start at `0.1.0`. Bump per semver when you publish. For unpublished internal tools, `0.0.1` forever is fine.

---

## 3. Ruff (lint + format)

One tool. Replaces black, isort, flake8, pyupgrade, and most of pylint.

**Daily commands:**
```
uv run ruff check --fix      # lint with autofix
uv run ruff format           # format (black-compatible)
```

**Recommended rule selection** (goes in `[tool.ruff.lint]`):
- `E`, `F` — pycodestyle errors, pyflakes (catch real bugs)
- `I` — import sorting (replaces isort)
- `UP` — pyupgrade (modernize syntax as you bump Python version)
- `B` — flake8-bugbear (common pitfalls)
- `SIM` — flake8-simplify (redundant patterns)
- `RUF` — ruff-specific rules

**Per-file ignores** for tests:
- `S101` (assert-used): pytest uses asserts; this is correct.
- `D100`-series (missing docstrings): tests don't need them.

Config can live in `[tool.ruff]` inside `pyproject.toml` (default; §10a) or a standalone `ruff.toml` (§10b). Pick one. Do not duplicate.

**In CI: `ruff format --check`** (fails if unformatted) and `ruff check` (no `--fix`).

---

## 4. Pytest

**Directory:** `tests/` at project root, no `__init__.py`. Mirror package structure.

**Config** goes in `[tool.pytest.ini_options]`:
```toml
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
```

- `-ra` — summary of all non-passing outcomes at the end
- `--strict-markers` — typos in `@pytest.mark.xxx` are errors, not silent no-ops
- `--strict-config` — same for config keys

**Naming:** `test_*.py`, `Test*` classes, `test_*` functions. Don't deviate.

**Fixtures** go in `conftest.py` at the nearest common ancestor of the tests that use them. Don't put app-wide fixtures in a single root `conftest.py` when only one subdirectory uses them.

**Parametrize, don't loop:**
```python
# Yes
@pytest.mark.parametrize("n,expected", [(1, 1), (2, 4), (3, 9)])
def test_square(n, expected):
    assert square(n) == expected

# No
def test_square():
    for n, expected in [(1, 1), (2, 4), (3, 9)]:
        assert square(n) == expected   # first failure hides the rest
```

**Structure each test as Arrange-Act-Assert**, separated by blank lines. If a test has more than one "Act," split it.

**Coverage:** `uv run pytest --cov=myproj --cov-report=term-missing`. No coverage gating in CI until the suite is mature — gating too early creates test-for-the-metric-not-for-the-bug habits.

---

## 5. ty (type check)

```
uv add --dev ty
uv run ty check
```

**All projects are typed.** Every function and method gets a full signature — parameters and return type — in both `src/` and `tests/`. No exceptions for "private" helpers, no "add them later". `ty check` runs clean on every commit; CI fails on type errors.

- Return `-> None` is explicit, not optional.
- Class attributes get annotations (`name: str`), not just `__init__` parameters.
- Tests count: `def test_foo() -> None:`.

**Modern syntax only** (Python 3.14+):
```python
# Yes
def fetch(ids: list[int]) -> dict[str, Item] | None: ...

# No — legacy
from typing import List, Dict, Optional
def fetch(ids: List[int]) -> Optional[Dict[str, Item]]: ...
```

`list[int]`, `dict[str, X]`, `X | None` work without imports on 3.9+ for annotations and 3.10+ for runtime. On 3.14 everything works everywhere.

**ty fallback:** if `ty check` can't handle something your project legitimately needs (rare but happens with complex Protocols, PEP 695 generics edge cases, or plugin-heavy frameworks), swap ty for mypy on *that* project: `uv remove ty && uv add --dev mypy`, replace `[tool.ty]` with `[tool.mypy]`, and note the swap in the README. Not a defeat; a documented escape hatch.

**Don't use `# type: ignore` without a specific rule:** `# type: ignore[arg-type]`, not bare `# type: ignore`. Bare ignores hide new errors forever.

---

## 6. Docstrings

**Google style.** Concise, well-supported by ruff, ty, IDEs, and Sphinx if you ever add docs.

**What to document:**
- Every public module, class, function, method.
- Private helpers: only when the *why* isn't obvious from the code.

**Document intent, not mechanics.** A docstring that reads "Returns the user's name" for `def get_user_name() -> str` is noise. Document:
- Preconditions the caller must meet
- What "empty" or "missing" means for this function
- Exceptions raised and when
- Non-obvious side effects

**Template:**
```python
def charge_card(amount_cents: int, card: Card) -> ChargeResult:
    """Charge a card, retrying once on transient network errors.

    Idempotency is the caller's responsibility — pass a unique
    idempotency_key on the card if you want this to be safe to retry.

    Args:
        amount_cents: Amount in the card's currency. Must be positive.
        card: Tokenized card object from the payments SDK.

    Returns:
        ChargeResult with status and provider reference.

    Raises:
        ConfigError: If the payments SDK is not configured.
        BackendTimeout: If the provider times out after the single retry.
    """
```

One-line docstrings are fine for obvious functions: `"""Return the config directory path."""`.

---

## 7. Logging

**stdlib `logging`. Never `print()`** in library or application code (CLIs output to stdout directly; that's not the same thing).

**Module-level logger:**
```python
import logging
logger = logging.getLogger(__name__)
```

Never `logging.getLogger()` (that's the root logger — configuring it affects every library).

**Libraries do not configure logging.** Add `logging.getLogger("myproj").addHandler(logging.NullHandler())` in `src/myproj/__init__.py` and stop there. Let the application configure.

**Applications configure once** at the entry point — see `_logging.py` template in §10d.

**Use `%` formatting for lazy eval:**
```python
# Yes — %s interpolation only happens if DEBUG is enabled
logger.debug("fetched %s items in %.2fs", len(items), elapsed)

# No — f-string formats even when the message is filtered out
logger.debug(f"fetched {len(items)} items in {elapsed:.2f}s")
```

**`logger.exception` inside `except`** — captures the traceback. `logger.error("...", exc_info=True)` works too.

**Never log:** passwords, tokens, full request/response bodies, PII. Log identifiers (user_id, request_id), not contents.

---

## 8. Error handling

**Raise specific exceptions.** `raise Exception(...)` is unfilterable — every caller has to either catch everything or nothing.

**Define a project exception hierarchy** in `src/myproj/exceptions.py`:
```python
class MyProjError(Exception):
    """Base class for all myproj exceptions."""

class ConfigError(MyProjError):
    """Raised when required configuration is missing or invalid."""

class BackendTimeout(MyProjError):
    """Raised when an external backend doesn't respond in time."""
```

Callers catch `MyProjError` for "anything from us" or specific subclasses for targeted handling.

**Catch at boundaries, not in the middle.** Translate third-party exceptions at the seam where your code meets the external lib:
```python
def fetch_user(user_id: str) -> User:
    try:
        resp = httpx.get(f"{BASE}/users/{user_id}", timeout=5)
        resp.raise_for_status()
    except httpx.TimeoutException as e:
        raise BackendTimeout(f"users endpoint timed out for {user_id}") from e
    except httpx.HTTPStatusError as e:
        raise BackendError(f"users endpoint: {e.response.status_code}") from e
    return User.model_validate(resp.json())
```

Downstream code catches `MyProjError` subclasses, not `httpx.*`. The `from e` preserves the original traceback chain for debugging.

**`except ... pass` is a bug.** If you truly mean to swallow, leave a one-line comment with the reason:
```python
try:
    shutil.rmtree(tmpdir)
except FileNotFoundError:
    pass  # already cleaned up by caller; safe to ignore
```

Never bare `except:` — always `except Exception:` at minimum (so KeyboardInterrupt can still kill the process).

---

## 9. Dependency management

**Add deps:**
```
uv add requests              # runtime
uv add --dev pytest ruff ty  # dev
```

**Optional extras** (feature flags for users):
```toml
[project.optional-dependencies]
redis = ["redis>=5"]
postgres = ["psycopg[binary]>=3.2"]
```

Users install with `uv add 'myproj[redis]'` or `pip install 'myproj[redis]'`.

**Upgrade:**
```
uv lock --upgrade-package requests   # one dep
uv lock --upgrade                    # everything
uv sync                              # apply to .venv
```

Commit the `uv.lock` change in a single-purpose commit with a subject like `deps: upgrade requests to 2.33`.

**Inspect:** `uv tree`.

**Script with ad-hoc deps** (no project):
```
uv run --with httpx --with rich python script.py
```

---

## 10. Templates

Copy-paste-ready. Rename `myproj` to your project name throughout.

### 10a. `pyproject.toml` (full starter)

```toml
[project]
name = "myproj"
version = "0.1.0"
description = "One line."
readme = "README.md"
requires-python = ">=3.14"
authors = [{ name = "Your Name", email = "you@example.com" }]
license = { text = "MIT" }
dependencies = [
    # runtime deps here, e.g.:
    # "httpx>=0.28",
]

[project.scripts]
myproj = "myproj.__main__:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/myproj"]

[dependency-groups]
dev = [
    "pytest>=8",
    "pytest-cov>=5",
    "ruff>=0.8",
    "ty>=0.0.1",
]

[tool.ruff]
line-length = 100
target-version = "py314"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]
ignore = []

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "D"]

[tool.ruff.format]
quote-style = "double"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
filterwarnings = ["error"]  # treat warnings as errors in tests

[tool.ty.src]
include = ["src"]
```

### 10b. Standalone `ruff.toml` (alternative to `[tool.ruff]` in pyproject.toml — pick one, not both)

```toml
line-length = 100
target-version = "py314"

[lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]
ignore = []

[lint.per-file-ignores]
"tests/**" = ["S101", "D"]

[format]
quote-style = "double"
```

### 10c. `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --locked

      - name: Ruff — lint
        run: uv run ruff check

      - name: Ruff — format check
        run: uv run ruff format --check

      - name: ty — type check
        run: uv run ty check

      - name: pytest
        run: uv run pytest --cov=myproj --cov-report=term-missing
```

Pin `astral-sh/setup-uv` to the current major tag; bump intentionally.

### 10d. `src/myproj/_logging.py`

```python
"""Logging setup. Call configure() once from the app entry point."""
from __future__ import annotations

import json
import logging
import os
import sys
from datetime import datetime, timezone


class _JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.fromtimestamp(record.created, tz=timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
        }
        if record.exc_info:
            payload["exc"] = self.formatException(record.exc_info)
        return json.dumps(payload)


def configure() -> None:
    """Configure root logging based on env vars.

    LOG_FORMAT=json      -> machine-readable JSON on stdout
    LOG_FORMAT=<other>   -> human-readable dev format (default)
    LOG_LEVEL=<level>    -> logging level name (default INFO)
    """
    level = os.environ.get("LOG_LEVEL", "INFO").upper()
    fmt = os.environ.get("LOG_FORMAT", "dev").lower()

    handler = logging.StreamHandler(sys.stdout)
    if fmt == "json":
        handler.setFormatter(_JsonFormatter())
    else:
        handler.setFormatter(
            logging.Formatter(
                "%(asctime)s %(levelname)-8s %(name)s: %(message)s",
                datefmt="%Y-%m-%d %H:%M:%S",
            )
        )

    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(level)
```

Call `configure()` from `src/myproj/__main__.py` (or your CLI entry point) — never from library code.

---

## 11. Quick reference

| Command | Purpose |
|---|---|
| `uv init --app myproj` | New application project |
| `uv init myproj` | New library project |
| `uv add <pkg>` | Add runtime dep |
| `uv add --dev <pkg>` | Add dev dep |
| `uv remove <pkg>` | Remove dep |
| `uv sync` | Install locked deps into `.venv` |
| `uv sync --locked` | Same, but fail if lockfile is stale (CI) |
| `uv lock --upgrade` | Bump all deps |
| `uv lock --upgrade-package <pkg>` | Bump one |
| `uv run <cmd>` | Run command inside project venv |
| `uv run python -m myproj` | Run the app module |
| `uv run pytest` | Tests |
| `uv run ruff check --fix` | Lint + autofix |
| `uv run ruff format` | Format |
| `uv run ty check` | Type check |
| `uv tree` | Show dep graph |
| `uv python pin 3.X` | Pin project Python version |

See MAC.md for one-time machine setup.
