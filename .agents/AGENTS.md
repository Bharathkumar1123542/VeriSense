# AGENTS.md — VeriSense
Verifiable low-cost climate sensing: signed readings → Merkle log → cross-validated trust score → agent explanations.

## Source of truth
- Before coding, read `spec.md` §0 plus the step you are on (§16); `prd.md` defines behaviour and user-facing copy; `checklist.md` defines order.
- Work one step (S01–S30) at a time. Do not start a step whose dependencies are not done.
- Spec beats code. Change a contract (§5 schemas, API, `config/validator.yaml`) only after a dated line in `docs/decisions.md`.

## Working loop
- Start each session by naming the step, the files you will touch and the test that proves it; read neighbouring code before editing.
- After each change run the narrowest test first, then `make test`. If a spec rule blocks the change, stop and report instead of working around it.

## Stack and layout
- Backend: Python 3.12, FastAPI, Pydantic v2, PyNaCl, psycopg 3, PostgreSQL 16, `uv`. Code in `backend/src/verisense/{core,ingest,log,refs,validator,episodes,agents,mcp_server,sim,api}`.
- Dashboard: React 18, TypeScript strict, Vite, MapLibre, TanStack Query in `dashboard/`. Firmware: PlatformIO in `firmware/esp32/`.
- Agents: Microsoft Agent Framework on Microsoft Foundry. Tools come from `verisense.mcp_server` and the Azure MCP Server (storage). Framework imports stay inside `verisense/agents/`.

## Commands
- `make up` (compose stack) · `make test` · `make eval` · `make demo` · `make snapshot` · `uv run ruff check && uv run mypy backend/src`
- One test: `uv run pytest backend/tests/unit/test_x.py -k name`. Dashboard: `npm test`, and `npm run gen:api` after any route change.

## Hard rules
- Agents annotate episodes only. They never write scores, readings, windows, the log or devices (Scenario Designer may only create scenarios). No LLM call per reading.
- Validator functions are pure: no clock, network, DB or LLM. Same inputs must give the same `provenance_id`. Use `math.fsum`; integer scores are `floor(x + 0.5)`.
- Signed or hashed material uses CJ1: integers and restricted ASCII only, never floats.
- Every number in agent text must pass the grounding check. Rule-based fallbacks must work with `VS_AGENTS_ENABLED=0`.
- Never send owner labels or tokens to agents or logs. Never commit secrets or `.env`. Never store coordinates finer than the signed, pre-rounded ones.
- Never edit `tests/vectors/` to make a test pass. Update the OpenAPI snapshot only when a route intentionally changes.

## Conventions
- `snake_case` in Python, SQL and JSON; env vars `VS_*`; labels lowercase; reason and error codes `UPPER_SNAKE`; times are integer Unix ms with an `_ms` suffix.
- The signed record key is `dev`; everywhere else it is `device_id`. User-facing reason text comes only from the `prd.md` §6.10 templates.
- Parameterised SQL only. Pydantic `extra="forbid"` on every request model. Errors follow `spec.md` §5.7.
- One step per change, tests alongside code, no unrelated refactors.

## Testing
- Agent tests use `FakeChatClient`; `@live` tests hit Foundry and run only when asked. Postgres comes from compose.
- Merkle changes need the Hypothesis property tests. Validator changes need golden fixtures and the 1,000-replay determinism test.

## Gotchas
- Time comes from `core.clock`, never `time.time()`; replay mode runs on a virtual clock and the REPLAY badge must stay visible.
- Pseudo-sensors exclude their host station (leave-one-out). Never remove that to improve scores.
- Azure MCP tool names are in `docs/azure-mcp-tools.md`; if blob upload is absent, use the SDK fallback (`spec.md` S22).

## Done means
- The step's "Done when" in `spec.md` §16 holds, new behaviour has tests, `make test`, ruff and mypy pass, and `checklist.md` is ticked.
- Ask before: adding dependencies, editing applied migrations, changing score weights, enabling admin routes outside demo mode, or running paid cloud calls beyond smoke tests.
