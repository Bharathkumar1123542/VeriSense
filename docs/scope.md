# VeriSense — Scope

**Doc set:** `scope.md` (this file) · `prd.md` · `spec.md` · `checklist.md`
**Version:** 1.0 · **Date:** 2026-10-19 · **Status:** frozen for build. Changes go through §12.

---

## 1. What it is

VeriSense is a verification layer for low-cost climate sensors. A sensor signs every reading at the source. The reading enters a tamper-evident public log. A validator then cross-checks it against independent official station data and coarse satellite-assimilated model data. The output is a 0–100 trust score, a label, and a provenance trail anyone can re-verify.

One agentic layer sits around that deterministic core. Agents investigate suspicious stretches of data, explain them in plain language, generate synthetic test data, and mirror the log's tree heads to independent storage. **No agent ever touches a signature, a hash, or a score.**

**Thesis.** Two questions get blurred in most IoT projects:

| Question | Answered by |
|---|---|
| *Integrity* — did this reading come from this device, unaltered? | Ed25519 signatures + Merkle log |
| *Plausibility* — is the number believable? | Cross-validation against independent references |

Neither is enough alone. Together they produce data a researcher or a city agency can act on.

## 2. Who it's for

| Audience | What they get |
|---|---|
| **Community sensor owners** (NGOs, schools, hobbyists) | A device that earns credibility: signed, scored, with a plain-language reason for every flag. |
| **Researchers** | A filterable export (CSV, GeoJSON, API) where every row carries a trust score and a provenance ID. |
| **City agency analysts** | A map of which community sensors can be used for decisions right now, and why. |
| **Hackathon judges** (secondary) | A working vertical slice they can attack live: tamper with a reading and watch it fail. |

## 3. Problem

1. Community sensors produce data nobody can check, so it is rarely used for decisions.
2. Verification schemes that exist rely on paperwork or on energy-heavy proof-of-work chains.
3. A signature proves a reading came from a device, not that the number is true. "Signed garbage" is still garbage.

## 4. Goals and non-goals

**Goals**

| ID | Goal |
|---|---|
| G1 | Every accepted reading is signed at source and provable as included in an append-only log. |
| G2 | Every reading gets a reproducible trust score with human-readable reasons. |
| G3 | Faults (drift, spikes, stuck values, tampering, replay) are detected, and the detection rates are measured and published. |
| G4 | Suspicious periods are investigated and explained by a grounded, multi-agent workflow on Microsoft Foundry. |
| G5 | The whole system runs from one `docker compose up` and one hosted demo link. |

**Non-goals**

| Non-goal | Why |
|---|---|
| Proving a reading is *true* | Signing proves origin. Validation yields confidence, not certification. |
| Official certification of sensors | Out of mandate and out of time. |
| A new data portal or dataset host | We read existing public sources and add a trust layer. |
| Tokens, payments, incentives | Out of scope; see §7. |

## 5. Scope at a glance

Priorities: **P0** = must ship, **P1** = ship if P0 is green at the checkpoint, **P2** = stretch.

### P0 — Must have

| ID | Item |
|---|---|
| M1 | Ed25519 signing at source (ESP32 firmware **or** simulator emitting the identical signed format) |
| M2 | Append-only Merkle transparency log (RFC 6962 hashing), signed tree heads, inclusion and consistency proofs |
| M3 | Ingest API: signature, registration, sequence and timestamp checks; replay rejection |
| M4 | Validator: integrity gate, plausibility, spatial agreement against ≥ 1 real source (OpenAQ) |
| M5 | Trust score (0–100), label, plain-language reasons, per-reading provenance, deterministic re-run |
| M6 | Dashboard: map with time slider, device detail, reading detail with "verify it yourself", integrity feed |
| M7 | Export: CSV and GeoJSON with trust and provenance columns |
| M8 | Synthetic data: seeded generator with fault models and ground-truth labels |
| M9 | Evaluation harness producing detection rate, false-positive rate, time-to-detect and trust-band calibration |
| M10 | Agent layer on Microsoft Foundry + Microsoft Agent Framework: **Investigator**, **Explainer**, **Scenario Designer**, orchestrated by one workflow |
| M11 | Custom MCP server exposing read-only verification tools to the agents |
| M12 | Demo panel (fault injection, tamper, replay) in replay mode |
| M13 | README, architecture diagram, `docker compose up`, demo video, limits and threat-model section |

### P1 — Should have

| ID | Item |
|---|---|
| S1 | CUSUM drift detection with humidity correction and suggested calibration offset |
| S2 | Satellite-assimilated model consistency check (CAMS via Open-Meteo) |
| S3 | **Mirror workflow**: signed tree heads published to immutable Azure Blob storage through the **Azure MCP Server**, with log-fork detection |
| S4 | Offline buffering and batch upload |
| S5 | One real ESP32 device in the demo |
| S6 | Hosted demo on Azure (Container Apps, PostgreSQL, Static Web Apps) |
| S7 | Research Copilot agent (natural-language dataset queries over verified data) |
| S8 | Foundry groundedness evaluation of explanations |

### P2 — Stretch

| ID | Item |
|---|---|
| X1 | Adversary loop: Scenario Designer reads the evaluation misses and proposes harder scenarios near the detection boundary |
| X2 | In-browser verification (WebCrypto / `@noble/ed25519`) |
| X3 | Peer agreement across community sensors |
| X4 | Owner approval of a suggested calibration offset (human-in-the-loop pause in the workflow) |
| X5 | Public-chain anchoring of tree heads |
| X6 | Second sensor type (water quality) |

### Cut — not in this build

| Item | Reason | Reinstate only if |
|---|---|---|
| Payments, tokens, marketplace | Not required by any judging criterion; adds legal surface | Never in this build |
| Mobile app | Dashboard is responsive | Never in this build |
| User accounts and login | Static tokens cover the three visibility levels | Hosted demo is abused |
| Learned outlier model | Adds an unexplainable score component | Evaluation shows rule-based recall < 0.7 |
| Per-reading LLM calls | Cost and non-determinism; agents run per *episode* | Never |
| Foundry-hosted agent deployment | Agents run in our own container | All P0 and P1 items are done |

## 6. Hard design rules

These rules bound the agentic layer. Any change to them needs a recorded decision (§12).

1. **Determinism.** Re-running the validator on stored inputs MUST yield the same score, label and provenance hash. The LLM is not an input to that function.
2. **LLM proposes, code disposes.** Agent output is stored beside the scores, never inside them. Agent claims that cite numbers MUST pass a grounding check against recorded tool output, otherwise a rule-based fallback is shown.
3. **Read-only agents.** The Investigator and Explainer have no write tools. Only the Scenario Designer can create a scenario record, and only through a schema-validated call.
4. **Episode-level LLM use.** Agents run on *episodes* (a contiguous stretch of flagged windows), never on each reading.
5. **Honest demo.** Replay mode shows a permanent "REPLAY — historical data, virtual clock" badge. Pseudo-sensors built from real stations use leave-one-out: their own station is withheld from the reference set.
6. **Privacy by construction.** The device signs coordinates already rounded to the owner's chosen precision (default 2 decimals, about 1.1 km). The server never holds a precise location.

## 7. Hackathon fit

The report targets the **IEEE ClimateChain Global Hackathon** (Devpost, track *Climate Data & Environmental Monitoring*). The judging rubric supplied with this task names **Microsoft Foundry, Microsoft Agent Framework and Azure MCP** as hero technologies. This plan satisfies both: the report's deterministic core is unchanged, and the agentic layer is added around it. The agentic layer is decoupled through the `episodes` table, so dropping it later would not break the core.

| Criterion (20% each) | Evidence we ship | Where |
|---|---|---|
| Technological Implementation | Seeded, labelled, vectorised synthetic-data generator with Latin-hypercube coverage; RFC 6962 log with property tests; Foundry model access; Agent Framework workflow; Azure MCP | `spec.md` S03–S08, S20–S23 |
| Agentic Design & Innovation | Investigator runs a hypothesis → deterministic counterfactual → verdict loop; grounding gate; fan-out/fan-in explanations; Scenario Designer closes a loop with the evaluator; Mirror workflow detects a forked log | `spec.md` §9 |
| Real-World Impact & Applicability | Offline-capable, about US$50 node, self-hostable, Apache-2.0, explicit threat model, published error rates | `prd.md` §1, §9 |
| User Experience & Presentation | Map-first dashboard, plain-language reasons, three audience explanations, 3:30 demo script, balanced frontend/backend | `prd.md` §6, §7 |
| Adherence to Hackathon Category | Climate data verification and environmental monitoring, end to end, with real OpenAQ and CAMS data | `prd.md` §1 |

## 8. Decisions that close the report's open questions (§13 of the report)

| Open question | Decision | Impact if wrong |
|---|---|---|
| Event duration, team size | Plan sized for **10 working days**, **3–4 people**, workstreams A–E from the report | Apply the cut order in §10 |
| Judging criteria | Use the five rubric items in §7 as written | Only the demo script and §7 table change |
| Pilot region | **Delhi NCR** (centre 28.6139 N, 77.2090 E, radius 25 km). Day-1 gate: ≥ 8 OpenAQ stations with ≥ 90% hourly PM2.5 completeness over the last 14 days. Fallback order: Mumbai Metropolitan Region, then Los Angeles basin | Region is one config file; nothing else changes |
| Ledger component expected? | Provide the transparency log plus an **immutable Azure Blob mirror**. Describe both as ledger-agnostic. Optional chain anchoring is P2 | Add X5 |
| Live or recorded demo | Record the primary video from replay mode. Keep a live walkthrough as backup | None |
| Data access and licences | OpenAQ v3 (free API key), Open-Meteo (non-commercial, CC BY 4.0). Snapshot files are committed only if every included provider's licence permits redistribution; otherwise `make snapshot` builds them on first run | Demo needs network on first run |
| Hardware | Simulator is the baseline; ESP32 is P1 | Demo stays honest via leave-one-out |

## 9. Constraints and assumptions

| Area | Constraint |
|---|---|
| Time | 10 working days. Day 10 is for the video and submission only. |
| Team | 3–4 people. Workstreams: A device and signing · B log and cryptography · C validator and analytics · D dashboard and UX · E data, evaluation, agents and storytelling. With fewer than 3 people, pre-apply cuts 1–6 in §10. |
| Budget | Assumes US$100 of Azure credit. Budget alerts at 50% and 80%. LLM calls happen only per episode, with a 60-minute cool-down per device. |
| Stack | Python 3.12, FastAPI, PostgreSQL 16, React + MapLibre, Docker Compose, Microsoft Foundry, Microsoft Agent Framework (Python), Azure MCP Server. Full table in `spec.md` §2. |
| Licence | Apache-2.0 |
| Data | No personal data beyond coarsened location and an owner label. |

## 10. Cut order and checkpoints

If a checkpoint is missed, cut from the top of this list until the next checkpoint is reachable. Never cut M1–M7 or M13.

| # | Cut first → last |
|---|---|
| 1 | S7 Research Copilot |
| 2 | S8 Foundry groundedness evaluation |
| 3 | X1–X6 (all stretch) |
| 4 | S2 satellite check |
| 5 | S5 real ESP32 (simulator only) |
| 6 | S4 offline buffering demo (code stays, demo beat goes) |
| 7 | S3 mirror workflow (describe in the architecture slide as designed, not built) |
| 8 | S6 hosted Azure demo (judges run `docker compose up`) |

| Checkpoint | Must be true |
|---|---|
| End of Day 3 | Sign → ingest → log → inclusion proof works end to end with the simulator |
| End of Day 5 | Validator produces scores with reasons against real OpenAQ data; dashboard shows the map |
| End of Day 7 | Investigator + Explainer workflow produces a grounded episode explanation |
| End of Day 9 | Full demo script runs without manual intervention; evaluation numbers exist |

## 11. Definition of done (submission-ready)

- [ ] Every P0 item passes its "Done when" in `spec.md`.
- [ ] 100% of unsigned, tampered and replayed readings are rejected in the automated suite.
- [ ] `docker compose up` followed by `make demo` reaches the demo state on a clean machine.
- [ ] `eval/results/report.md` exists and the Evidence page shows its numbers.
- [ ] Demo video is within the length limit in the official rules.
- [ ] README includes the limits and threat-model section.
- [ ] Devpost text covers: inspiration, what it does, how it was built, challenges, accomplishments, lessons, next steps.

## 12. Change control

1. Any addition MUST name what it replaces, and the replaced item moves down the cut order.
2. Record every scope decision as one dated line in `docs/decisions.md`.
3. Contract changes (record schema, API, canonicalisation) after Day 2 need agreement from the owners of workstreams A, B and C, because they invalidate signed test fixtures.
