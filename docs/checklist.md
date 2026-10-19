# VeriSense — Build Checklist

**Version:** 1.0 · **Date:** 2026-10-19 · Step IDs (`S01` …) match `spec.md` §16. "Done when" conditions live there.
**Format:** `[ ] ID · priority · workstream · task`. Workstreams: **A** device/signing · **B** log/crypto · **C** validator · **D** dashboard/UX · **E** data, eval, agents, story.
**Rule:** if a checkpoint is missed, cut from the top of the cut order in `scope.md` §10. Never cut S01–S18 or S30.

---

## Day 1 — Foundations

- [ ] S01 · P0 · all · Repo, `uv` workspace, ruff/mypy/pytest, CI, `.env.example`, Makefile
- [ ] S01 · P0 · E · Foundry project + model deployment `verisense-chat`; run `scripts/foundry_smoke.py`
- [ ] S01 · P0 · E · OpenAQ API key; Azure budget alerts at 50% and 80%
- [ ] S09 · P0 · E · Region gate: ≥ 8 OpenAQ stations, ≥ 90% completeness, 14 days (Delhi NCR, else Mumbai MMR, else Los Angeles); write result to `docs/region.md`
- [ ] S02 · P0 · A B C · Start contract freeze: record schema, CJ1, error codes

## Day 2 — Contracts and crypto

- [ ] S02 · P0 · A B C · Contract sign-off recorded in `docs/decisions.md`; OpenAPI skeleton committed
- [ ] S03 · P0 · B · CJ1/CJ1F, sign/verify, device ID, receipts; ≥ 10 vectors in `tests/vectors/reading_v1.json`
- [ ] S05 · P0 · B · Migrations from spec §6; `migrate` command
- [ ] S04 · P0 · B · Pure Merkle functions with property tests

## Day 3 — Log end to end

- [ ] S04 · P0 · B · Database-backed append (10,000 appends < 30 s)
- [ ] S06 · P0 · B · Registry + ingest + `register_device.py`; T-ING all green
- [ ] S07 · P0 · B · STH loop, receipts, log proof routes
- [ ] S22 · P1 · E · Run `scripts/azure_mcp_tools.py`; record tool names in `docs/azure-mcp-tools.md`; decide upload path (MCP or SDK)
- [ ] S08 · P0 · C · Clock + replay clock
- [ ] **Checkpoint:** sign → ingest → log → inclusion proof works with a script-driven device

## Day 4 — Data and simulation

- [ ] S09 · P0 · E · OpenAQ, Open-Meteo, CAMS adapters; replay provider; snapshot built; `find_events.py` picks the regional-event window
- [ ] S10 · P0 · A E · Generator core, fault models, device runtime, fault-command polling
- [ ] S11 · P0 · C · Gate + plausibility; `provisional_loop`
- [ ] S26 · P1 · A · Firmware: key, SNTP, sensors, signing (host vector test first)

## Day 5 — Scoring and map

- [ ] S12 · P0 · C · Windows, humidity correction, reference selection, spatial, context, leave-one-out
- [ ] S13 · P0 · C · Scoring, labels, caps, reasons, provenance, determinism test (`temporal = 1.0` for now)
- [ ] S16 · P0 · D · Dashboard shell, replay badge, log chip, map, slider, list, badges
- [ ] S15 · P0 · D B · Query API routes (start)
- [ ] **Checkpoint:** real OpenAQ data produces scores with reasons; map shows devices

## Day 6 — Drift, episodes, detail screens

- [ ] S13 · P1 · C · CUSUM drift, suggested offset, CAMS check, regional-event flag
- [ ] S14 · P0 · C · Window finaliser, episode builder, rule-based diagnosis
- [ ] S15 · P0 · D B · Remaining routes, visibility, export + manifest
- [ ] S17 · P0 · D · Device detail, reading detail, verify panel, integrity feed, export form
- [ ] S18 · P0 · B · `verify_reading.py` (good reading passes, edited reading fails with the failing step named)

## Day 7 — Agents

- [ ] S19 · P0 · E · Custom MCP server with all tools and output truncation
- [ ] S20 · P0 · E · Runtime, Investigator, grounding, reconcile, templates (fake-client tests + live smoke)
- [ ] S21 · P0 · E D · Explainer ×3, episode workflow, Episode screen with *Test it*
- [ ] **Checkpoint:** injected drift produces an episode with a grounded diagnosis and three explanations

## Day 8 — Mirror, synthetic data, evaluation

- [ ] S22 · P1 · E B · Mirror workflow through Azure MCP, SDK fallback, fork detection, banner
- [ ] S23 · P0 · E · `ScenarioSpec`, campaign sampler (Latin hypercube), Scenario Designer, dataset card
- [ ] S24 · P0 · E D · Evaluation harness (direct mode), report, Evidence page
- [ ] S26 · P1 · A · Firmware: upload, offline buffer, flush; one real device posting scored data

## Day 9 — Demo and deployment

- [ ] S25 · P0 · D E · Admin routes, Demo panel, Scenario box, `demo_seed.py`, `make demo`
- [ ] S27 · P1 · E · Research Copilot
- [ ] S28 · P1 · all · Compose finalised; Bicep; hosted demo
- [ ] Run the full demo script (`prd.md` §7) twice without manual intervention
- [ ] Clean-machine test: `docker compose up --build` then `make demo`
- [ ] **Checkpoint:** demo script runs end to end; `eval/results/report.md` exists

## Day 10 — Polish and submit

- [ ] S29 · P1 · E · Traces in Application Insights; Foundry groundedness run
- [ ] S30 · P0 · all · README, architecture diagram, `docs/limits.md`
- [ ] S30 · P0 · E · Record the video (≤ official limit); screenshots; Devpost text
- [ ] S30 · P0 · all · Usability check with 3–5 non-technical testers (can they explain a flag in under a minute?)
- [ ] Submit with time to spare

## Release gates (all must be ticked)

- [ ] 100% of unsigned, tampered and replayed readings rejected in the automated suite
- [ ] 0 mismatches in 1,000 validator replays
- [ ] Demo script runs without manual intervention
- [ ] Clean-machine `docker compose up --build` + `make demo` reaches the demo state
- [ ] README states the limits and threat model
- [ ] Evidence page shows the latest evaluation numbers, including any missed targets

## Stretch (only after every P0 and P1 line above is ticked)

- [ ] X1 Adversary loop · X2 In-browser verify · X3 Peer agreement · X4 Offset approval · X5 Chain anchoring · X6 Water quality
