# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

A **notes/reference directory**, not a source code repository. Actual Python projects live in other directories on disk. There is nothing to build, test, or lint here — just two markdown files maintained by hand.

## Files

- **MAC.md** — one-time macOS setup playbook for a Python dev machine (uv, VS Code extensions, global config, shell env var, Claude permissions). Steps are numbered and meant to be executable top-to-bottom on a fresh Mac.
- **PROJECT.md** — per-project Python best practices once the machine is set up (src layout, `pyproject.toml`, ruff, pytest, ty, docstrings, logging, error handling) plus copy-paste templates. Referred to when bootstrapping any new Python project anywhere on disk.

MAC.md is the machine layer; PROJECT.md is the next layer up. PROJECT.md links back to MAC.md for install steps and does not duplicate them.

## Editing conventions

- **MAC.md documents universal fresh-machine setup only.** Don't add one-off repairs for this specific machine's prior state (e.g., "uninstall extension X we installed years ago"). A reader on a clean Mac wouldn't need them. Perform one-offs in execution but leave them out of the doc.
- **Don't duplicate `~/.claude/CLAUDE.md`.** That file already holds the short uv conventions loaded into every Claude session globally (`uv add` not `pip install`, `.venv` per project, etc.). MAC.md and PROJECT.md exist to go deeper, not to mirror it.
- **For reference/playbook docs, the doc *is* the plan.** When asked to write or update MAC.md / PROJECT.md, write the final content directly to the target file — don't produce a separate meta-plan about what the doc will contain.

## Stack the docs prescribe

uv (deps/envs) · ruff (lint+format) · pytest (tests) · ty (type check, with mypy as documented fallback) · stdlib logging. Python 3.14 is the global default (uv-managed). `src/` layout, `pyproject.toml` as single source of truth, `uv.lock` committed.

# Python / uv conventions

- All Python work uses uv. Never run bare `pip install` — it will fail anyway (`PIP_REQUIRE_VIRTUALENV=1`).
- Start projects with `uv init <name>` (library) or `uv init --app <name>` (app).
- Add deps with `uv add <pkg>` / `uv add --dev <pkg>`. Remove with `uv remove <pkg>`.
- Run code with `uv run <cmd>` (auto-activates `.venv`). Scripts: `uv run python script.py`.
- After pulling: `uv sync`.
- Per-project Python version: `uv python pin 3.X` (writes `.python-version`). Global default is 3.14.
- Commit: `pyproject.toml`, `uv.lock`, `.python-version`. Gitignore: `.venv/`.
- For global CLI tools (ruff, pre-commit, etc.), use `uv tool install <pkg>` — not `pip install --user`.