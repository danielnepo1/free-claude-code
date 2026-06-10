# AGENTIC DIRECTIVE

> This file is identical to AGENTS.md. Keep them in sync: any edit here must be mirrored there.

## PROJECT OVERVIEW

**Free Claude Code** is an Anthropic-compatible proxy that lets Claude Code (CLI, VS Code,
JetBrains ACP, or a Discord/Telegram bot) talk to any LLM provider. It accepts Anthropic
Messages API traffic on `/v1/messages`, routes the request to a configured provider/model,
translates the provider's stream back into Anthropic SSE, and returns it unchanged to the client.

- **Language / runtime**: Python 3.14 (see `.python-version`), managed with [uv](https://github.com/astral-sh/uv).
- **Web framework**: FastAPI + Uvicorn (ASGI), always-streaming responses.
- **Packaging**: `hatchling`; version lives in `pyproject.toml` (`2.0.0`).
- **Entry point**: `server.py` builds the app via `api.app.create_asgi_app`.
- **Console scripts**: `fcc-server` (proxy), `fcc-claude` (launch Claude Code against the proxy),
  `fcc-init` (advanced env scaffold), `free-claude-code` (alias for `fcc-server`).
- **Config surface**: a local **Admin UI** at `/admin` (loopback only) edits the managed env file
  (`~/.fcc/.env`); `.env.example` is the read-only reference for env key names.

## CODING ENVIRONMENT

- Install astral uv using "curl -LsSf https://astral.sh/uv/install.sh | sh" if not already installed and if already installed then update it to the latest version
- Install Python 3.14.0 stable using `uv python install 3.14.0` if not already installed (requires uv >=0.9; see `[tool.uv] required-version` in `pyproject.toml`)
- Always use `uv run` to run files instead of the global `python` command.
- Current uv ruff formatter is set to py314 which has supports multiple exception types without paranthesis (except TypeError, ValueError:)
- Read `.env.example` for environment variables.
- All CI checks must pass; failing checks block merge.
- Add tests for new changes (including edge cases), then run `uv run pytest`.
- Run checks in this order: `uv run ruff format`, `uv run ruff check`, `uv run ty check`, `uv run pytest`.
- Do not add `# type: ignore` or `# ty: ignore`; fix the underlying type issue.
- All 5 checks are enforced in `tests.yml` on push/merge (parallel jobs: suppression grep, ruff-format, ruff-check, ty, pytest).
- Branch protection: set **required status checks** to **all** of those statuses (e.g. **Ban type ignore suppressions**, **ruff-format**, **ruff-check**, **ty**, **pytest**—use the exact labels GitHub shows, which may be prefixed with **CI /**). Remove **ci** from required checks if it was previously added for the old gate job.

## IDENTITY & CONTEXT

- You are an expert Software Architect and Systems Engineer.
- Goal: Zero-defect, root-cause-oriented engineering for bugs; test-driven engineering for new features. Think carefully; no need to rush.
- Code: Write the simplest code possible. Keep the codebase minimal and modular.

## ARCHITECTURE PRINCIPLES

- **Shared utilities**: Put shared Anthropic protocol logic in neutral `core/anthropic/` modules. Do not have one provider import from another provider's utils.
- **DRY**: Extract shared base classes to eliminate duplication. Prefer composition over copy-paste.
- **Encapsulation**: Use accessor methods for internal state (e.g. `set_current_task()`), not direct `_attribute` assignment from outside.
- **Provider-specific config**: Keep provider-specific fields (e.g. `nim_settings`) in provider constructors, not in the base `ProviderConfig`.
- **Dead code**: Remove unused code, legacy systems, and hardcoded values. Use settings/config instead of literals (e.g. `settings.provider_type` not `"nvidia_nim"`).
- **Performance**: Use list accumulation for strings (not `+=` in loops), cache env vars at init, prefer iterative over recursive when stack depth matters.
- **Platform-agnostic naming**: Use generic names (e.g. `PLATFORM_EDIT`) not platform-specific ones (e.g. `TELEGRAM_EDIT`) in shared code.
- **No type ignores**: Do not add `# type: ignore` or `# ty: ignore`. Fix the underlying type issue.
- **Complete migrations**: When moving modules, update imports to the new owner and remove old compatibility shims in the same change unless preserving a published interface is explicitly required.
- **Maximum Test Coverage**: There should be maximum test coverage for everything, preferably live smoke test coverage to catch bugs early

## REPOSITORY MAP

```text
free-claude-code/
├── server.py              # ASGI entry point: app = create_asgi_app()
├── api/                   # FastAPI layer: routes, services, routing, optimizations, admin UI
│   ├── app.py             # App factory, lifespan, exception handlers (ProviderError -> Anthropic JSON)
│   ├── routes.py          # /v1/messages, /v1/messages/count_tokens, /v1/models, /health, /stop, probes
│   ├── services.py        # ClaudeProxyService: orchestrates routing -> provider -> SSE
│   ├── model_router.py    # Maps Claude model name -> configured provider/model (ResolvedModel)
│   ├── runtime.py         # AppRuntime: composes registry, CLI manager, messaging; owns lifecycle
│   ├── dependencies.py    # FastAPI deps: get_settings, require_api_key, resolve_provider
│   ├── gateway_model_ids.py  # Encode/decode "gateway" model ids for the /model picker
│   ├── detection.py / optimization_handlers.py  # Local fast-path answers to trivial Claude probes
│   ├── admin_routes.py / admin_config.py / admin_static/  # Loopback-only Admin UI
│   └── web_tools/         # Server-side web_search / web_fetch tool egress + streaming
├── core/                  # Provider-NEUTRAL protocol helpers (must not import api/messaging/cli/providers/config)
│   └── anthropic/         # SSE builder, content/conversion, thinking, tokens, tools, errors, stream contracts
├── providers/             # Provider adapters + registry + rate limiting
│   ├── base.py            # BaseProvider (ABC) + ProviderConfig (pydantic base)
│   ├── openai_compat.py   # OpenAIChatTransport: OpenAI chat-completions -> Anthropic SSE
│   ├── anthropic_messages.py  # AnthropicMessagesTransport: native /v1/messages passthrough
│   ├── registry.py        # Factories + ProviderRegistry (cache, model discovery, validation, cleanup)
│   ├── <provider>/        # One package per provider (client.py, optional request.py)
│   └── error_mapping.py / exceptions.py / rate_limit.py / model_listing.py
├── config/                # Settings + neutral metadata (must not import api/messaging/cli/core/providers)
│   ├── settings.py        # Pydantic Settings; reads .env, ~/.fcc/.env, FCC_ENV_FILE (later wins)
│   ├── provider_catalog.py   # SINGLE SOURCE of provider ids, transports, credentials, default base URLs
│   ├── provider_ids.py / paths.py / constants.py / nim.py / logging_config.py
├── messaging/             # Discord/Telegram bot wrapper, sessions, reply-trees, rendering, voice
│   ├── platforms/         # base.py (MessagingPlatform interface), discord.py, telegram.py, factory.py
│   ├── rendering/         # Markdown -> platform-specific formatting
│   └── trees/             # Reply-branch conversation trees
├── cli/                   # Console-script entrypoints + Claude subprocess management
├── smoke/                 # Local-only live E2E smoke tests (opt-in via FCC_LIVE_SMOKE=1)
├── tests/                 # Hermetic unit + contract tests (must pass with plain `uv run pytest`)
└── scripts/               # install.sh / install.ps1
```

## REQUEST FLOW

1. Claude Code sends an Anthropic Messages request to `POST /v1/messages` (`api/routes.py`).
   Auth is checked by `require_api_key` (x-api-key / bearer / anthropic-auth-token).
2. `ClaudeProxyService` (`api/services.py`) tries local optimizations (`try_optimizations`) and
   web server-tool handling first; trivial probes are answered locally to save latency/quota.
3. `ModelRouter` (`api/model_router.py`) resolves the incoming Claude model name to a
   `provider_id` + `provider_model` using `MODEL_OPUS`/`MODEL_SONNET`/`MODEL_HAIKU`, falling
   back to `MODEL`. Gateway-picker model ids are decoded via `gateway_model_ids`.
4. `ProviderRegistry.get` returns a cached `BaseProvider`; the provider's `stream_response`
   yields Anthropic SSE. OpenAI-chat providers translate chat-completions deltas; native
   providers stream `/v1/messages` through with block-policy normalization.
5. The provider normalizes thinking blocks, tool calls, token usage, and errors into the shape
   Claude Code expects. `ProviderError` is rendered to Anthropic JSON by the app exception handler.

## KEY CONVENTIONS

- **Two transport bases** — extend exactly one:
  - `OpenAIChatTransport` (`providers/openai_compat.py`) for OpenAI chat-completions upstreams
    (e.g. NIM, Gemini, Mistral, Groq, Cerebras, OpenCode). `transport_type="openai_chat"`.
  - `AnthropicMessagesTransport` (`providers/anthropic_messages.py`) for native Anthropic
    `/v1/messages` upstreams (e.g. OpenRouter, DeepSeek, Kimi, Wafer, Z.ai, Fireworks, local
    LM Studio / llama.cpp / Ollama). `transport_type="anthropic_messages"`.
- **`config/provider_catalog.py` is the single source of truth** for supported provider ids,
  transport type, credential env/attr, default base URLs, and capabilities. A startup assertion
  and `tests/contracts/test_import_boundaries.py` enforce that `PROVIDER_CATALOG`,
  `PROVIDER_FACTORIES` (in `providers/registry.py`), and `SUPPORTED_PROVIDER_IDS` stay in sync.
- **Import boundaries are contract-tested** (`tests/contracts/test_import_boundaries.py`):
  - `core/` may not import `api`, `messaging`, `cli`, `smoke`, `providers`, or `config`.
  - `config/` may not import `api`, `messaging`, `cli`, `smoke`, `providers`, or `core`.
  - `providers/` may not import `api`, `messaging`, or `cli`.
  - `api/` may only import the narrow provider facade: `providers`, `providers.base`,
    `providers.exceptions`, `providers.registry`.
  - `messaging/` may not import `api`/`cli`/`smoke`; only `providers.nvidia_nim.voice` is allowed.
- **Settings come from `config.settings.Settings`** (pydantic-settings), layered from `.env`,
  `~/.fcc/.env` (managed by the Admin UI), and `FCC_ENV_FILE` (later files override). Never
  hardcode provider literals — read `settings.provider_type`, `settings.model`, etc.
- **Always streaming**: `/v1/messages` returns a `StreamingResponse` of Anthropic SSE.
- **Logging**: use `loguru`; keep error logs metadata-only unless `log_api_error_tracebacks`
  is set. Use `core.trace.trace_event` for structured tracing.
- **Python 3.14 syntax**: parenthesis-free multi-exception handlers (`except TypeError, ValueError:`)
  are valid; ruff targets `py314`. Double-quoted strings, 88-col lines, isort first-party groups.

## ADDING A PROVIDER

1. Add a `ProviderDescriptor` entry to `PROVIDER_CATALOG` in `config/provider_catalog.py`
   (id, `transport_type`, credential env/attr, default base URL, capabilities).
2. Create `providers/<id>/` with a `client.py` provider subclassing `OpenAIChatTransport` or
   `AnthropicMessagesTransport`, plus an optional `request.py` for request shaping.
3. Register a factory function in `PROVIDER_FACTORIES` (`providers/registry.py`).
4. Add the credential field to `config/settings.py` and document the key in `.env.example`.
5. Add unit tests under `tests/providers/` and, if relevant, smoke coverage under `smoke/`.
6. Keep provider-specific quirks inside the provider package — never import across providers;
   put shared protocol logic in `core/anthropic/`.

## TESTING

- **Hermetic tests** live in `tests/` and MUST pass with plain `uv run pytest` (no network).
  Includes unit tests (`tests/api`, `tests/providers`, `tests/messaging`, `tests/config`,
  `tests/core`, `tests/cli`) and architecture/contract tests (`tests/contracts/`).
- **pytest** uses `pytest-asyncio` and runs parallel by default (`addopts = "-n auto"`,
  `testpaths = ["tests"]`). Markers: `live`, `provider`, `messaging`, `cli`, `clients`,
  `voice`, `contract`, `smoke_target`.
- **Smoke tests** (`smoke/`) are local-only and opt-in via `FCC_LIVE_SMOKE=1`; they may launch
  subprocesses and call real providers/bots. See `smoke/README.md` for targets, the
  `FCC_SMOKE_*` env knobs, and failure classes. Do not put hermetic tests in `smoke/`.

## CI

CI (`.github/workflows/tests.yml`, on push/PR to `main`/`master`) runs five required checks in
parallel: a `# type: ignore` / `# ty: ignore` suppression ban (grep), `ruff format --check`,
`ruff check`, `ty check`, and `pytest -v`. uv is pinned via `[tool.uv] required-version`.

## COGNITIVE WORKFLOW

1. **ANALYZE**: Read relevant files. Do not guess.
2. **PLAN**: Map out the logic. Identify root cause or required changes. Order changes by dependency.
3. **EXECUTE**: Fix the cause, not the symptom. Execute incrementally with clear commits.
4. **VERIFY**: Run ci checks and relevant smoke tests. Confirm the fix via logs or output.
5. **SPECIFICITY**: Do exactly as much as asked; nothing more, nothing less.
6. **PROPAGATION**: Changes impact multiple files; propagate updates correctly.

## SUMMARY STANDARDS

- Summaries must be technical and granular.
- Include: [Files Changed], [Logic Altered], [Verification Method], [Residual Risks] (if no residual risks then say none).

## TOOLS

- Prefer built-in tools (grep, read_file, etc.) over manual workflows. Check tool availability before use.
