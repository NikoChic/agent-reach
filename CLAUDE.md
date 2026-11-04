# CLAUDE.md

Agent Reach: one CLI (`agent-reach`) that gives AI agents internet access and always prints one JSON envelope. Python 3.10+, package `agent_reach/`, built with hatchling. Upstream is `Panniantong/Agent-Reach`. The PyPI package named `agent-reach` is a different project; users install from GitHub releases.

## Setup, build, test

```bash
python -m venv .venv
.venv/bin/pip install -c constraints.txt -e ".[dev]"   # always install with the constraints file

.venv/bin/pytest -q                      # full suite, about 20s, no network or real accounts needed
.venv/bin/pytest -q tests/test_web_channel.py -k <name>
.venv/bin/ruff check agent_reach tests   # lint (E, F, I; line length 100, E501 ignored)
.venv/bin/mypy agent_reach               # type check (CI runs it; tests/ is excluded)
./test.sh                                # clean venv + temp HOME, CLI smoke test, then pytest
python -m build                          # wheel; CI checks that skill/*.md and scripts/*.sh are included
```

Run pytest, ruff and mypy before every commit. All three must pass, as CI requires (Python 3.10 to 3.13 on Linux, plus macOS and Windows).

Live channel checks (`agent-reach add|login|check <channel> --text`) need real logins and keys, so don't run them unless I ask. Tests must never touch real accounts, the user's browser or the real `~/.agent-reach`. `tests/conftest.py` sandboxes HOME and installers, so rely on it and use `monkeypatch` for upstream calls.

## Layout

- `agent_reach/cli.py`: entry point (`agent_reach.cli:main`); `runtime/cli.py` handles parsing.
- `agent_reach/channels/`: one `Channel` subclass per platform, registered in `ALL_CHANNELS` in `channels/__init__.py`. `base.py` defines `Channel`, `Action` and `Param`.
- `agent_reach/runtime/`: `result.py` (envelope, `make_item`, `ReachError`), `dispatch.py` (tries backends in order and falls back on retryable errors), `manage.py` (add/remove/doctor, `PREREQS`), `tools.py` (`TOOLS`: privately installed upstream tools with `tested`/`floor`/`exclude`), plus `browser*.py`, `cdp.py`, `bridge.py` and `sessions.py` (the shared CloakBrowser profile and logins), and `state.py`.
- `agent_reach/backends/`: shared backend adapters (opencli, cloak, boss_native).
- `agent_reach/skill/`: the agent skill shipped in the wheel (`SKILL.md` in Chinese, `SKILL_en.md` in English, `references/`).
- `docs/`, `README.md` (Chinese), `docs/README_en.md`, `llms.txt`: user and agent docs.
- `scripts/`: release acceptance tooling.

## Channel contract (see CONTRIBUTING.md for full detail)

- Declare `name`, `description`, `can_handle(url)`, `level` (`default`/`optional`/`experimental`), and `actions` with **ordered** `backends` and a `live_test`. Optionally declare `tools`, `prereqs`, `login_url` or `session`+`login_argv`, and `setup_problems()`.
- Backends return `make_item(...)`. On failure they raise `ReachError(code, message, hint=...)`, where code is one of `NEED_SETUP`, `NEED_LOGIN`, `UPSTREAM_BROKEN`, `TIMEOUT`, `RATE_LIMITED`, `NOT_FOUND` or `BAD_INPUT`. Hints are written for the agent.
- New or changed channels need unit tests under `tests/` (follow `tests/test_channel_contracts.py`). Update the README channel table if a channel is added or its requirements change.

## Hard rules (some are enforced by tests such as `test_auth_guidance_policy.py`)

- Never read a personal browser's Cookie database, and never log in for the user. Hints and docs must not tell the agent to log in on the user's behalf, and must not add QR or automatic cookie-login guidance.
- Everything a channel stores lives under `~/.agent-reach/`. Never install anything globally; add the tool to `runtime.tools.TOOLS` instead.
- Never commit keys, cookies, account names or machine-specific paths.
- Don't edit `.github/release-acceptance.json` by hand. Any change under `agent_reach/` invalidates it, and only maintainers regenerate it with `scripts/record_release_acceptance.py`.
- When bumping the version, keep `pyproject.toml` and `agent_reach/__init__.py` (`__version__`) in sync; the release workflow checks both against the tag.
- When changing dependency ranges, update the pins in `constraints.txt` too.

## Style and conventions

- Follow the existing style: type hints, `from __future__`/`typing` as used nearby, small modules, and `# -*- coding: utf-8 -*-` headers where present. Keep imports sorted (ruff `I`).
- User-facing strings and docs are often bilingual (Chinese first, then English). When changing one language in README, SKILL or CHANGELOG, update the other as well.
- Every `CHANGELOG.md` version section starts with a short plain summary for users, in Chinese and in English: 1 to 4 points, with no file names, commands or jargon. Technical notes go below it.
- Commit messages are in English. Keep changes small and focused.
- Don't commit `.venv/`, `uv.lock` or `.claude/settings.local.json` (all are gitignored).
