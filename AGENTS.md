# Repository Guidelines

## Project Structure & Module Organization
This repository is a collection of small Python automation projects, each maintained as its own `uv` package:

- `exa-register/`: Exa registration flow, browser solver, mail provider helpers, and local config.
- `grok-register/`: Grok/x.ai flow plus a vendored `luckmail/` client.
- `openai-register/`: OpenAI registration flow, upload helpers, and another vendored `luckmail/` client.
- `tavily-register/`: Tavily signup CLI, captcha handling, and YAML-driven configuration.

Keep changes scoped to the relevant subdirectory. Treat generated files such as `keys/`, `tokens/`, `exa_apikeys.txt`, `api_keys.txt`, `failed.txt`, `run.log`, `.env`, and `config.yaml` as local runtime artifacts, not source files.

## Build, Test, and Development Commands
Install dependencies from the project you are editing:

```bash
cd exa-register && uv sync
cd grok-register && uv sync
cd openai-register && uv sync --extra cpa   # only when CPA support is needed
cd tavily-register && uv sync
```

Run the main entrypoint directly with `uv run`:

```bash
cd exa-register && uv run python exa_core.py
cd grok-register && uv run python grok.py --email-provider luckmail
cd openai-register && uv run python openai_register.py --once
cd tavily-register && uv run python batch_signup.py --help
```

## Coding Style & Naming Conventions
Follow existing Python style in each module: 4-space indentation, straightforward scripts, and minimal abstraction unless duplicated logic justifies extraction. Use `snake_case` for files, functions, variables, and CLI flags. Preserve current entrypoint names such as `exa_core.py`, `grok.py`, and `openai_register.py`. Prefer targeted helpers over repo-wide shared utilities unless at least two projects need the same behavior.

## Testing Guidelines
There is no unified automated test suite yet. For now, validate changes with the smallest runnable command in the affected subproject and document the exact command used in your PR. When adding tests, place them under a local `tests/` package inside the subproject and use `test_*.py` naming.

## Commit & Pull Request Guidelines
Recent history uses short conventional-style subjects such as `feat(openai-register): ...`, `fix: ...`, and `docs(readme): ...`. Keep commits focused to one subproject when possible. PRs should include:

- a short summary of behavior changes
- the commands run for validation
- any required env vars, APIs, or captcha/mail providers
- screenshots or logs only when they clarify a UI/browser regression

## Security & Configuration Tips
Never commit live API keys, tokens, `.env` files, purchased mailbox data, or generated account outputs. Prefer updating `.gitignore` when a new runtime artifact is introduced.
