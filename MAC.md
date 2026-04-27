# Python Dev Machine Setup with uv

## Context

Fresh macOS (Apple Silicon) environment with only Homebrew's Python 3.14.4 — no uv, no pyenv, no Python tooling. Goal: stand up a modern uv-centric Python dev machine where every project is isolated in a virtual environment, with guardrails strong enough that accidentally polluting the system/user site-packages becomes an error.

`/Users/etoews/dev/etoews/python/` is a notes/reference directory — Python projects themselves will live in other directories on disk. Conventions therefore live at **user-scope** (so they apply wherever `uv init` runs), and this file is the human-readable playbook to refer back to.

Decisions:
- Install uv via Homebrew
- Default Python for new projects: 3.14 (uv-managed)
- Strong venv enforcement: `PIP_REQUIRE_VIRTUALENV=1` + uv defaults + global `CLAUDE.md`
- No globally-installed uv tools — add `ruff`/`pytest`/etc. per project via `uv add --dev`

## Setup steps

### 1. Install uv via Homebrew

```
brew install uv
```

Verify: `uv --version`. uv installs to `/opt/homebrew/bin/uv`, already on PATH via the existing `brew shellenv` line in `~/.zprofile`.

### 2. Install uv-managed Python 3.14 and pin globally

```
uv python install 3.14
uv python pin --global 3.14
```

`uv python pin --global` writes `~/.config/uv/.python-version`, making 3.14 the default for any new project without its own `.python-version`. Homebrew's Python 3.14.4 stays for ad-hoc `python3`; uv-managed builds live under `~/.local/share/uv/python/` and are only used by uv workflows.

### 3. Create global uv config at `~/.config/uv/uv.toml`

```toml
# Prefer uv-managed Pythons; don't silently use a random system Python
python-preference = "only-managed"

# Auto-download a pinned Python version if missing
python-downloads = "automatic"

# Compile .pyc files on install — faster first run, small disk cost
compile-bytecode = true
```

`uv pip` already refuses to install outside a venv by default (requires an explicit `--system` flag to override), so no extra config is needed for that path.

### 4. Add venv guardrail to `~/.zprofile`

Append:

```
export PIP_REQUIRE_VIRTUALENV=1
```

Makes any plain `pip install …` (Homebrew-Python pip, or any non-uv pip) fail with "Could not find an activated virtualenv" unless a venv is active. Paired with step 3, both code paths refuse to install globally.

### 5. Create `~/.claude/CLAUDE.md` (user-scope conventions)

Short Claude-facing file loaded in *every* session:

```markdown
# Python / uv conventions

- All Python work uses uv. Never run bare `pip install` — it will fail anyway (PIP_REQUIRE_VIRTUALENV=1).
- Start projects with `uv init <name>` (library) or `uv init --app <name>` (app).
- Add deps with `uv add <pkg>` / `uv add --dev <pkg>`. Remove with `uv remove <pkg>`.
- Run code with `uv run <cmd>` (auto-activates .venv). Scripts: `uv run python script.py`.
- After pulling: `uv sync`.
- Per-project Python version: `uv python pin 3.X` (writes `.python-version`). Global default is 3.14.
- Commit: `pyproject.toml`, `uv.lock`, `.python-version`. Gitignore: `.venv/`.
- For global CLI tools (ruff, pre-commit, etc.), use `uv tool install <pkg>` — not `pip install --user`.
- Full playbook: `/Users/etoews/dev/etoews/python/MAC.md`.
```

### 6. Extend user-scope Claude permissions

Edit `~/.claude/settings.json` to merge a `permissions.allow` block (preserving existing `model`, `effortLevel`, `voiceEnabled`, `voice` keys):

```json
{
  "permissions": {
    "allow": [
      "Bash(uv --version)",
      "Bash(uv init:*)",
      "Bash(uv add:*)",
      "Bash(uv remove:*)",
      "Bash(uv sync:*)",
      "Bash(uv lock:*)",
      "Bash(uv run:*)",
      "Bash(uv python:*)",
      "Bash(uv tree:*)",
      "Bash(uv venv:*)",
      "Bash(uv tool list)"
    ]
  }
}
```

`uv tool install` / `uv tool uninstall` are deliberately **not** auto-allowed — they mutate global state and deserve a prompt.

### 7. Install VS Code extensions

Opinionated Python-stack extensions. Install from the terminal:

```
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension ms-python.debugpy
code --install-extension ms-python.vscode-python-envs
code --install-extension charliermarsh.ruff
code --install-extension astral-sh.ty
code --install-extension tamasfe.even-better-toml
code --install-extension ms-toolsai.jupyter
```

`astral-sh.ty` is Astral's preview type-checker extension — runs as a language server alongside Pylance.

### 8. Configure VS Code user settings

Merge into `~/Library/Application Support/Code/User/settings.json`:

```json
"[python]": {
  "editor.defaultFormatter": "charliermarsh.ruff",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.ruff": "explicit",
    "source.organizeImports.ruff": "explicit"
  }
},
"[toml]": {
  "editor.defaultFormatter": "tamasfe.even-better-toml"
}
```

- `source.fixAll.ruff` applies ruff autofixes on save.
- `source.organizeImports.ruff` sorts and prunes imports on save (replaces isort).

## Daily recipes (reference)

### Start a new project

```
cd ~/dev/somewhere
uv init myproj          # library-style layout; use `uv init --app myproj` for a CLI/app
cd myproj
uv add requests         # runtime dep
uv add --dev ruff pytest  # dev deps
uv run pytest           # run tests
uv run python -m myproj # run the app
```

### Clone an existing uv project

```
git clone <repo> && cd <repo>
uv sync                 # materializes .venv from uv.lock
uv run <cmd>            # no manual `source .venv/bin/activate` needed
```

### Switch Python version for one project

```
uv python pin 3.13
uv sync   # rebuilds .venv against the new interpreter
```

### One-off script with inline deps (no project)

```
uv run --with httpx python - <<'PY'
import httpx
print(httpx.get("https://example.com").status_code)
PY
```

### Upgrade a dependency

```
uv lock --upgrade-package requests
uv sync
```

### Upgrade everything

```
uv lock --upgrade
uv sync
```

### Global CLI tool

```
uv tool install ruff        # puts `ruff` on PATH at ~/.local/bin/ruff
uv tool list
uv tool upgrade ruff
uv tool uninstall ruff
```

## Files created / modified

- **New**: `~/.config/uv/uv.toml`
- **New**: `~/.claude/CLAUDE.md`
- **New**: `/Users/etoews/dev/etoews/python/MAC.md` (this file)
- **Edit**: `~/.zprofile` (append one export)
- **Edit**: `~/.claude/settings.json` (add `permissions.allow`)
- **Edit**: `~/Library/Application Support/Code/User/settings.json` (add `[python]` and `[toml]` format-on-save blocks)
- **Install**: VS Code extensions listed in step 7
- **Unchanged**: Homebrew's Python 3.14, `/Users/etoews/dev/etoews/python/.claude/settings.local.json`

## Verification

Run from a fresh shell so `~/.zprofile` is re-sourced:

1. `uv --version` — prints a version.
2. `uv python list --only-installed` — shows a uv-managed 3.14.x.
3. Venv guardrail: `/opt/homebrew/bin/pip3 install --dry-run requests` should fail with "Could not find an activated virtualenv".
4. End-to-end smoke test in a throwaway dir outside this one:
   ```
   cd /tmp && uv init demo && cd demo
   uv add requests
   uv run python -c "import requests; print(requests.__version__)"
   rm -rf /tmp/demo
   ```
   should create `demo/.venv/`, write `pyproject.toml` + `uv.lock`, print a version.
5. `uv pip install --dry-run requests` run outside any project dir — should fail because `pip.require-virtualenv = true`.
6. Start a fresh Claude session in a new directory and confirm it follows the uv conventions without prompting (proves `~/.claude/CLAUDE.md` is loading).
