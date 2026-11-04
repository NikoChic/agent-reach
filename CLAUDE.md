# CLAUDE.md

Agent Reach: a Python CLI (`agent-reach`) that gives AI agents internet access through
"channels" (web, YouTube, GitHub, V2EX, Bilibili, Xiaohongshu, Reddit, ...). Every command
prints one JSON shape: `{schema_version, ok, channel, action, backend, items, meta, error}`.
This repo is a fork of `Panniantong/Agent-Reach`; upstream URLs in docs point there on purpose.

## Setup, build, test

```bash
python -m venv .venv
.venv/bin/pip install -c constraints.txt -e ".[dev]"   # always use constraints.txt

.venv/bin/pytest -q                       # full suite, ~20s, no network needed
.venv/bin/pytest tests/test_v2ex_channel.py -q   # single file
.venv/bin/ruff check agent_reach tests    # lint (rules E, F, I; line length 100)
.venv/bin/mypy agent_reach                # types (CI runs this; tests/ excluded)
./test.sh                                 # clean-venv integration run (slow; optional)
python -m build                           # wheel (CI checks skill/*.md + scripts ship in it)
```

Before committing, run: `pytest -q`, `ruff check agent_reach tests`, `mypy agent_reach`.
CI (`.github/workflows/pytest.yml`) runs Python 3.10–3.13 on Linux, plus macOS and Windows.

Do **not** run `ruff format`: the codebase is not formatted with it and it would rewrite
~65 files. Only `ruff check` (`--fix` is fine for import sorting).

## Code style

- Python 3.10+ (no 3.11-only syntax), type hints on functions, `from typing import Any` etc.
- Files start with `# -*- coding: utf-8 -*-` and a module docstring.
- Short docstrings on functions. English for code, comments and commit messages.
- Keep code cross-platform (Windows CI): use `pathlib`, `utils/process.py` helpers for
  subprocesses (`utf8_subprocess_env`), always pass `encoding="utf-8"` when reading/writing text.
- Write private files via the helpers in `utils/paths.py` (e.g. `atomic_write_private_text`).

## Layout

- `agent_reach/cli.py` → `runtime/cli.py`: argument parsing; `runtime/dispatch.py`: backend
  fallback; `runtime/result.py`: `make_item`, `ReachError` and error codes.
- `agent_reach/channels/`: one `Channel` subclass per platform, registered in
  `channels/__init__.py`. `base.py` defines `Channel`, `Action`, `Param`.
- `agent_reach/runtime/tools.py` (`TOOLS`) and `runtime/manage.py` (`PREREQS`, setup, skill
  install): private tool installs under `~/.agent-reach/`.
- `agent_reach/backends/`: shared backends (OpenCLI, CloakBrowser, ...).
- `agent_reach/skill/SKILL.md` (Chinese) and `SKILL_en.md` (English): the agent skill shipped
  in the wheel. Change both together.
- `tests/`: pytest; `conftest.py` isolates HOME and **blocks real network and non-local
  subprocesses** — mock HTTP responses in tests.

## Channel rules (see CONTRIBUTING.md for the full contract)

- Backends return items built with `make_item(...)` and raise
  `ReachError(code, message, hint=...)` with `NEED_SETUP`, `NEED_LOGIN`, `UPSTREAM_BROKEN`,
  `TIMEOUT`, `RATE_LIMITED`, `NOT_FOUND` or `BAD_INPUT`. Hints are for the agent and must
  never tell it to log in for the user.
- Never read a personal browser's cookie database or log in on the user's behalf.
- Everything stored lives under `~/.agent-reach/`; never install anything globally.
- New/changed channel: add tests under `tests/` (see `test_channel_contracts.py`), and
  update the README channel table (`README.md` and `docs/README_en.md`) and `llms.txt` if the
  channel list or its requirements change.
- Never commit keys, cookies, account names or machine-specific paths.

## Docs, changelog, versions

- User-facing docs are bilingual: `README.md` (Chinese) and `docs/README_en.md` (English).
- `CHANGELOG.md`: each version starts with a plain summary for users in Chinese and English
  (1–4 points, no jargon, no file names); technical entries follow. It becomes the release body.
- The version lives in both `pyproject.toml` and `agent_reach/__init__.py`; keep them equal.
- Any change under `agent_reach/` invalidates `.github/release-acceptance.json`; releases are
  maintainer-only (see CONTRIBUTING.md "Releasing"). Don't tag or edit the acceptance record.
