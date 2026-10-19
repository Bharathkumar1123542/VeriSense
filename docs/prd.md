# VeriSense — Product Requirements (PRD)

**Version:** 1.0 · **Date:** 2026-10-19 · **Companion docs:** `scope.md` (what and why), `spec.md` (how it is built)
**Keywords:** MUST and MUST NOT are binding. MAY marks an explicitly optional behaviour. Priorities: P0 ship, P1 ship if P0 is green, P2 stretch (see `scope.md` §5).

---

## 1. Product summary

VeriSense turns a low-cost environmental sensor into a source of evidence. Each reading is signed by the device, recorded in a public tamper-evident log, cross-checked against independent references, and displayed with a trust score and a reason a non-expert can read.

**Category fit.** Climate data verification and environmental monitoring. Pilot pollutant: PM2.5. Pilot region: Delhi NCR (`scope.md` §8). Reference sources: OpenAQ station data and CAMS model data via Open-Meteo.

**What an agent does here.** Agents handle the work that needs judgement and language: investigating a suspicious stretch of data, explaining it to three audiences, designing synthetic test scenarios, and publishing log checkpoints to independent storage. They never alter a score.

## 2. Personas

| ID | Persona | Context | Goal | Key tasks |
|---|---|---|---|---|
| P-1 | **Asha**, community sensor owner | Runs two sensors for a neighbourhood NGO. No data-science background. | Show her data is credible. Know when her sensor misbehaves. | Register a device, read its health, understand a flag, act on a suggestion |
| P-2 | **Dr. Rao**, air-quality researcher | Combines community and official data in studies. | Use only data he can defend in a methods section. | Filter by trust, export with provenance, verify a reading independently |
| P-3 | **Meera**, city agency analyst | Decides where to deploy official monitors and when to issue advisories. | Know which community sensors she can rely on right now. | Scan the map, open a suspect sensor, read the operational summary |
| P-4 | **Judge** | Has a few minutes and a rubric. | See it work, try to break it. | Watch the demo, tamper with a reading, check the numbers |

## 3. User stories and acceptance criteria

| ID | Story | Acceptance (Given / When / Then) | Pri |
|---|---|---|---|
| US-01 | As Asha, I register my device without creating an account. | Given a device with a key pair, when its signed registration is posted, then the device appears in the registry with the visibility and coordinate precision I chose, and the server holds no coordinate finer than that precision. | P0 |
| US-02 | As Asha, I see my sensor's trust at a glance. | Given a device reporting for 2+ hours, when I open the map, then my device shows a label, a score and an icon, and a click opens its detail view. | P0 |
| US-03 | As Asha, I understand why a reading was flagged. | Given a flagged reading, when I open it, then I see at least one plain-language reason with concrete numbers, and no raw JSON is needed. | P0 |
| US-04 | As Dr. Rao, I verify a reading without trusting your server. | Given a public reading, when I run the displayed command, then the script checks the device signature, leaf hash, inclusion proof and tree-head signature and prints `VERIFIED`. | P0 |
| US-05 | As Dr. Rao, I export only data above a trust threshold. | Given a filter of `min_trust=80` and a date range, when I download CSV or GeoJSON, then every row has `trust ≥ 80`, a `provenance_id`, and the file's manifest lists the tree head it was cut against. | P0 |
| US-06 | As Dr. Rao, I re-run a validation and get the same answer. | Given a stored window, when I call replay, then the score, label and provenance hash match the stored values exactly. | P0 |
| US-07 | As Meera, I see which sensors are usable now. | Given the time slider at "now", when I filter the map to *Verified + Plausible*, then only those devices are visible. | P0 |
| US-08 | As Meera, I read an operational summary of a problem sensor. | Given an open episode, when I open the *Agency* tab, then I see a headline, a cause, a recommended action and a caution about data use, each under the length limits in FR-52. | P1 |
| US-09 | As Asha, I learn that my sensor is drifting and by how much. | Given a sensor with a sustained bias, when drift is flagged, then the device card shows direction, size of bias and a suggested offset in µg/m³. | P1 |
| US-10 | As a judge, I tamper with a reading and watch it fail. | Given replay mode and the demo panel, when I edit a stored reading, then the reading's label becomes *Rejected*, reason `LOG_INCLUSION_FAIL` appears, and its inclusion proof fails to verify. | P0 |
| US-11 | As a judge, I replay an old signed reading. | Given a previously accepted reading, when I resend it, then ingest answers `409 SEQ_REPLAY` and an integrity event appears in the feed. | P0 |
| US-12 | As a judge, I see a real pollution event *not* penalised. | Given a regional event in the replay data, when the sensor and stations all rise together, then windows stay *Verified* and the reading detail shows `REGIONAL_EVENT_CONFIRMED`. | P0 |
| US-13 | As a judge, I see the operator get caught rewriting history. | Given replay mode, when the log is rewritten via the demo panel, then the next mirror check fails and a red banner reads "Log inconsistent with mirrored tree head". | P1 |
| US-14 | As a judge, I ask for a custom test scenario in plain English. | Given the demo panel, when I type "slow drift on node 2 for two days during a smoke event", then a validated scenario spec is shown, I confirm it, and a labelled synthetic dataset runs through the validator. | P0 |
| US-15 | As Dr. Rao, I ask a data question in plain English. | Given the Research Copilot, when I ask "verified PM2.5 above 100 last week", then I get filters, a row count and a download link, and every number cited matches the query result. | P1 |
| US-16 | As anyone, I see published error rates. | Given the Evidence page, when it loads, then I see detection rate, false-positive rate, time-to-detect and error-by-trust-band from the latest evaluation run, with the run date and seed. | P0 |

## 4. Functional requirements

### 4.1 Device and signing

| ID | Requirement | Pri |
|---|---|---|
| FR-01 | A device MUST create its own Ed25519 key pair. The private key MUST NOT leave the device. | P0 |
| FR-02 | Each reading MUST be signed over a canonical byte string (`spec.md` §5). The signed record MUST contain device ID, sequence number, UTC timestamp, rounded coordinates, sensor model, firmware hash, PM2.5, temperature and relative humidity. | P0 |
| FR-03 | The simulator MUST emit the identical signed format and MUST support these faults: drift, spike, stuck, clock skew, tamper, replay. | P0 |
| FR-04 | A device MUST buffer readings while offline and upload them in sequence order later. | P1 |
| FR-05 | Registration MUST be self-signed by the device key and MUST include the owner's visibility choice and coordinate precision. | P0 |

### 4.2 Transparency log

| ID | Requirement | Pri |
|---|---|---|
| FR-10 | Every accepted reading MUST become a leaf in a Merkle tree using RFC 6962 hashing. | P0 |
| FR-11 | The log MUST publish a signed tree head at least every 30 seconds while new leaves exist. | P0 |
| FR-12 | The log MUST serve inclusion proofs and consistency proofs. | P0 |
| FR-13 | The log MUST record its own receive time inside each leaf, and ingest MUST return a signed receipt immediately. | P0 |
| FR-14 | Signed tree heads MUST be mirrored to immutable Azure Blob storage. The mirror write MUST go through the Azure MCP Server. If the MCP upload fails twice, the system MUST fall back to the Azure SDK and record `method = sdk_fallback`. | P1 |
| FR-15 | If a new tree head is not consistent with the last mirrored head, the system MUST raise integrity event `LOG_FORK_DETECTED` and show a banner within 60 seconds. | P1 |

### 4.3 Ingest

| ID | Requirement | Pri |
|---|---|---|
| FR-20 | Ingest MUST reject unsigned, badly signed, unregistered, revoked-device, replayed, future-dated and over-age readings with the error codes in `spec.md` §5.7. | P0 |
| FR-21 | A sequence gap MUST be accepted and recorded as `SEQ_GAP`. | P0 |
| FR-22 | Every rejection MUST create an integrity event visible in the feed. | P0 |
| FR-23 | Batch upload MUST accept up to 500 records and return a per-record result. | P1 |

### 4.4 Validation and scoring

| ID | Requirement | Pri |
|---|---|---|
| FR-30 | The validator MUST run the integrity gate, which fails on invalid signature, unregistered or revoked device, non-increasing sequence, timestamp outside bounds, or failed log inclusion. A gate failure MUST give trust 0 and label *Rejected*. | P0 |
| FR-31 | The validator MUST run plausibility checks: range, rate of change, spike outlier, stuck value. | P0 |
| FR-32 | The validator MUST compare hourly sensor means against distance-weighted nearby official stations, with tolerance scaled to sensor error and local spread. | P0 |
| FR-33 | The validator MUST apply a humidity correction for supported optical particulate sensors before comparison, and record whether it was applied. | P1 |
| FR-34 | The validator MUST track the running sensor-minus-reference difference with CUSUM and raise a drift flag with a suggested offset. | P1 |
| FR-35 | The validator MUST compare against CAMS model data and use it for regional-event confirmation. It MUST NOT enter the weighted score. | P1 |
| FR-36 | When no good reference exists, the label MUST be *Unverifiable* with a reason, not a low score. | P0 |
| FR-37 | Trust = Gate × 100 × (0.25 Plausibility + 0.35 Spatial + 0.25 Temporal + 0.15 Context), with the caps in `spec.md` §7.6. | P0 |
| FR-38 | Re-running the validator on stored inputs MUST yield identical score, label, reasons and provenance hash. | P0 |
| FR-39 | Each reading MUST have a provenance record (`spec.md` §7.7) retrievable by ID. | P0 |

### 4.5 Agentic insights

| ID | Requirement | Pri |
|---|---|---|
| FR-50 | The system MUST open an *episode* for a contiguous stretch of flagged windows and run the episode workflow on it. | P0 |
| FR-51 | The Investigator MUST test its hypotheses by calling the deterministic validator with counterfactual inputs (an offset, an excluded station), and MUST cite tool output for every number it states. | P0 |
| FR-52 | The Explainer MUST produce three variants (citizen, researcher, agency), each with a headline ≤ 90 characters, a body ≤ 600 characters and ≤ 3 actions of ≤ 120 characters each. | P0 |
| FR-53 | A failed grounding check MUST replace the agent text with a rule-based template and mark it `source: rules`. | P0 |
| FR-54 | Agents MUST run per episode, with a 60-minute cool-down per device unless severity increases. | P0 |
| FR-55 | The Research Copilot MUST answer dataset questions using only tool results and MUST return the filters it applied. | P1 |

### 4.6 Synthetic data and evaluation

| ID | Requirement | Pri |
|---|---|---|
| FR-60 | The generator MUST be deterministic for a given scenario spec and seed. | P0 |
| FR-61 | Generated data MUST carry ground-truth labels per sample (`fault_active`, `fault_type`, `magnitude`). | P0 |
| FR-62 | The Scenario Designer MUST turn plain English into a `ScenarioSpec` that passes schema validation, with one automatic repair retry. | P0 |
| FR-63 | The evaluation harness MUST report per-fault precision, recall and time-to-detect; false-positive rate on healthy sensors and on regional events; and mean absolute error by trust band. | P0 |
| FR-64 | The pseudo-sensor's own station MUST be excluded from its reference set (leave-one-out). | P0 |
| FR-65 | The Scenario Designer MAY read evaluation misses and propose harder scenarios (adversary loop). | P2 |

### 4.7 Dashboard, export, demo

| ID | Requirement | Pri |
|---|---|---|
| FR-70 | Map with devices coloured and iconned by label, time slider over the available range, and label filter. | P0 |
| FR-71 | Device detail: current trust, drift indicator, 24-hour uptime, gap count, and a chart of sensor against reference stations. | P0 |
| FR-72 | Reading detail: sub-score bars, reasons, reference stations used (ID, distance, value, retrieval time), provenance card, verify panel. | P0 |
| FR-73 | Integrity feed listing rejections and fork alarms in reverse time order. | P0 |
| FR-74 | Export to CSV and GeoJSON with filters: date range, bounding box, device, minimum trust. | P0 |
| FR-75 | Demo panel (replay mode only): inject faults, tamper, replay, rewrite log, run a natural-language scenario, control replay speed. | P0 |
| FR-76 | Replay mode MUST show a permanent badge: "REPLAY — historical data, virtual clock". | P0 |
| FR-77 | Visibility: `public` data is open, `research` requires a researcher token, `private` requires the admin token. | P0 |

## 5. Non-functional requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-01 | Ingest latency | p95 ≤ 150 ms per reading at 20 requests/second on 1 vCPU |
| NFR-02 | Validation latency | A window is finalised ≤ 5 s after its references become available |
| NFR-03 | Agent latency | Investigation plus explanation completes ≤ 60 s at p95 |
| NFR-04 | Dashboard load | First meaningful paint ≤ 3 s on a throttled 4G profile; JS ≤ 600 KB gzipped |
| NFR-05 | Signing cost | ≤ 5 ms per reading on ESP32 (measured and reported) |
| NFR-06 | Payload size | ≤ 600 bytes per signed reading on the wire |
| NFR-07 | Determinism | Zero mismatches in 1,000 randomly chosen replays |
| NFR-08 | Reliability | Worker restarts MUST NOT lose or duplicate validations or episode runs (idempotent by key) |
| NFR-09 | Security | Strict request schemas, body ≤ 256 KB, per-device ingest limit 2 requests/second, CORS limited to the dashboard origin, admin routes disabled outside demo mode, no secrets in the repo |
| NFR-10 | Accessibility | WCAG 2.1 AA: contrast, keyboard operation, visible focus, label never conveyed by colour alone, chart data available as a table, `prefers-reduced-motion` respected |
| NFR-11 | Portability | `docker compose up --build` works on Linux, macOS and Windows (WSL2) with only Docker installed |
| NFR-12 | Cost | LLM calls occur only per episode. Total cloud spend within the budget in `scope.md` §9 |
| NFR-13 | Licensing | Apache-2.0 for code; attribution for OpenAQ and Open-Meteo shown in the footer and README |

## 6. UX specification

### 6.1 Principles

1. **Show the reason, not just the number.** Every score sits next to a sentence.
2. **Never colour alone.** Each label has an icon and a word.
3. **Evidence is one click away.** Any claim opens the data behind it.
4. **Calm by default.** Most of the screen is quiet; only problems draw the eye.
5. **Say what we don't know.** *Unverifiable* is a first-class state.

### 6.2 Information architecture

| Route | Screen | Primary audience |
|---|---|---|
| `/` | Map | Meera, Judge |
| `/devices/:device_id` | Device detail | Asha |
| `/readings/:reading_id` | Reading detail + verify | Dr. Rao |
| `/episodes/:episode_id` | Episode: diagnosis and explanations | Asha, Meera |
| `/export` | Export builder (+ Research Copilot, P1) | Dr. Rao |
| `/evidence` | Evaluation results | Judge |
| `/integrity` | Integrity feed | Dr. Rao, Judge |
| `/demo` | Demo panel (replay mode, admin token) | Judge |

### 6.3 Visual language

| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | `#FFFFFF` | `#0F1419` | Page |
| `--surface` | `#F5F7FA` | `#1A2129` | Cards |
| `--text` | `#12202E` | `#E8EDF2` | Body |
| `--text-muted` | `#4B5B6B` | `#A3B0BD` | Secondary |
| `--verified` | `#1B7F4B` | `#4CC38A` | Label colour |
| `--plausible` | `#1D6FA5` | `#6BB6E8` | Label colour |
| `--suspect` | `#9A6700` | `#E3B341` | Label colour |
| `--rejected` | `#C0392B` | `#FF7B72` | Label colour |
| `--unverifiable` | `#5B6670` | `#9AA5B1` | Label colour |
| `--pending` | `#5B6670` (dashed outline) | `#9AA5B1` (dashed outline) | Label colour |

Every text and icon colour above MUST reach 4.5:1 against its surface. Typeface: Inter, falling back to the system UI font. Numbers use tabular figures. Radius 8 px. Spacing scale 4/8/12/16/24/32 px.

| Label | Icon | Word shown |
|---|---|---|
| verified | circle with check | Verified |
| plausible | circle with dot | Plausible |
| suspect | triangle with exclamation | Suspect |
| rejected | octagon with cross | Rejected |
| unverifiable | circle with question mark | Unverifiable |
| pending | clock | Pending |

### 6.4 Global frame

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ VeriSense   Map  Export  Evidence  Integrity  [Demo]     ⛓ Log consistent ✓  │
│ [REPLAY — historical data, virtual clock · 2026-10-14 06:00 UTC · 180×]      │
└──────────────────────────────────────────────────────────────────────────────┘
```

- The log chip shows: *Log consistent · mirrored 2 min ago* (green), *Mirror pending* (grey), or *Log inconsistent with mirrored tree head* (red, full-width banner below the header).
- The replay badge is hidden in live mode.

### 6.5 Map screen

```
┌─────────────────────────────────────────────┬──────────────────────────┐
│                                             │ Filter: ☑Verified ☑Plaus. │
│            MapLibre map                     │ ☑Suspect ☑Rejected ☑Unver.│
│     ● device (icon + colour by label)       │ ─────────────────────────│
│     ring thickness = context confidence     │ ✓ vs-01  Verified   92   │
│                                             │ ▲ vs-02  Suspect    51   │
│                                             │ ? vs-03  Unverifiable —  │
│                                             │ ⏱ vs-04  Pending         │
├─────────────────────────────────────────────┴──────────────────────────┤
│ ◀ ▶  ─────────●──────────────  Oct 14 03:00 UTC            [Now]       │
└────────────────────────────────────────────────────────────────────────┘
```

- Marker colour and icon follow §6.3. Marker position uses the device's coarsened public coordinates.
- Selecting a marker or list row opens a side card (label, score, top reason, *Open device*).
- The slider steps in hours and drives both the map and the list. Keyboard: arrow keys step one hour, Shift + arrow steps six.

### 6.6 Device detail

Sections, top to bottom:

1. **Header**: device ID (short form `vs-xxxxxx`), owner label, sensor model, visibility, *Leave-one-out* badge when applicable.
2. **Health strip**: trust now (label + score), drift indicator (↑, ↓ or none, with size in µg/m³), 24-hour uptime %, gap count, last seen.
3. **Chart**: corrected hourly sensor mean against each reference station (thin lines, one per station, distance in the legend), with the tolerance band `reference ± sigma` shaded. Below it, a strip of coloured window labels. A *Show as table* toggle gives the same data in a table.
4. **Episodes**: list with status, severity, headline.
5. **Recent readings**: table of the last 50 with trust, label, top reason.

### 6.7 Reading detail and verify panel

```
┌ Reading vs-3fa2…-1042 ────────────────── ✓ Verified · 88 ───────────────┐
│ PM2.5 61 µg/m³ (corrected 57)   RH 54%   T 29.4 °C   2026-10-14 06:12   │
│                                                                         │
│ Why 88?                                                                 │
│  Plausibility ████████████ 1.00   Spatial ██████████░░ 0.84             │
│  Temporal     ███████████░ 0.95   Context ████████░░░░ 0.70             │
│  • Matches 4 nearby stations within 12% (median distance 6.2 km).       │
│  • Humidity correction applied.                                         │
│                                                                         │
│ References used (leave-one-out: none)                                   │
│  ID        Distance   Value   Retrieved                                 │
│  …                                                                      │
│                                                                         │
│ Verify it yourself                                                      │
│  Device key   3fa2…  Leaf hash  9c1d…  Log index 18,402  Tree size 18,977│
│  $ python scripts/verify_reading.py <url>      [Copy]  [Download script]│
└─────────────────────────────────────────────────────────────────────────┘
```

- The sub-score bars always show the number next to the bar.
- Reasons are listed most severe first, then positive evidence.
- For *Rejected* readings caused by the gate, the bars are replaced by the failing check and its explanation.

### 6.8 Episode screen

- Header: device, time span, severity icon, status (*open* or *closed*).
- **Diagnosis card**: cause (plain name), evidence strength (*strong*, *moderate*, *weak*), recommended action. Source chip: *Agent, grounded* or *Rules*.
- **Explanation tabs**: Citizen · Researcher · Agency. Default tab follows the `?as=` URL parameter, else Citizen.
- **Test it** panel: shows each counterfactual run as a line, for example *"If this sensor read 9.8 µg/m³ lower, 6 of 7 flagged hours would be Verified."* Each line links to the validator inputs.
- **Evidence list**: links to the windows and readings cited.

### 6.9 Export, Evidence, Integrity, Demo

| Screen | Content |
|---|---|
| Export | Filter form (dates, bounding box, device, min trust, label), a live row count, preview of 20 rows, buttons *CSV* and *GeoJSON*. In P1, a Research Copilot box fills the form from plain English and shows the filters it chose. |
| Evidence | Four charts from the latest evaluation: recall by fault type and magnitude, false-positive rate, time-to-detect distribution, mean error by trust band. Header shows run date, seed, number of scenarios. A *Download results* link serves `metrics.json`. |
| Integrity | Reverse-chronological list: time, device, code, one-line explanation. Filter by code. Fork alarms are pinned at the top. |
| Demo | Left column: fault injection (device, fault type, parameters, duration), tamper, replay, rewrite log. Right column: replay speed and start/pause, plus the *Describe a scenario* box (Scenario Designer) showing the generated spec for confirmation before it runs. |

### 6.10 Copy: reasons shown to users

This table is the source of truth. `spec.md` `reasons.py` MUST implement these templates verbatim. Placeholders in braces are filled from validator output and rounded as shown.

| Code | Sev | Template |
|---|---|---|
| `SIG_INVALID` | fail | The signature does not match this device's key. The reading was changed or came from another device. |
| `DEVICE_UNREGISTERED` | fail | This device has not been registered. |
| `DEVICE_REVOKED` | fail | This device's key was revoked before this reading was received. |
| `SEQ_REPLAY` | fail | This reading's sequence number was already used. It looks like an old reading sent again. |
| `TS_FUTURE` | fail | The device clock is {skew_s} s ahead of the server. |
| `TS_TOO_OLD` | fail | The reading is more than {max_days} days old on arrival. |
| `TS_NON_MONOTONIC` | fail | The timestamp goes backwards compared with the previous reading. |
| `LOG_INCLUSION_FAIL` | fail | The stored reading no longer matches the public log. It was altered after it was recorded. |
| `RANGE_VIOLATION` | fail | {value} {unit} is outside what this sensor model can measure ({lo}–{hi}). |
| `RATE_EXCEEDED` | warn | PM2.5 changed by {delta} µg/m³ per minute, faster than the sensor can physically follow. |
| `SPIKE_OUTLIER` | warn | This value ({value} µg/m³) is far from the readings around it (typical {median} µg/m³). |
| `STUCK_VALUE` | warn | The sensor has reported the same value for {minutes} minutes. |
| `LOW_COVERAGE` | warn | Only {pct}% of expected readings arrived in this hour. |
| `NO_REFERENCE` | info | No official station within {radius_km} km reported for this hour. |
| `LOW_CONTEXT` | info | The nearby references are too few or too far to judge this sensor ({n} within {d} km). |
| `SPATIAL_AGREES` | ok | Matches {n} nearby stations within {pct}% (median distance {d} km). |
| `SPATIAL_HIGH_BIAS` | warn | Reads {pct}% higher than {n} nearby stations (median distance {d} km) this hour. |
| `SPATIAL_LOW_BIAS` | warn | Reads {pct}% lower than {n} nearby stations (median distance {d} km) this hour. |
| `DRIFT_UP` | warn | Has read higher than nearby stations for {hours} hours, growing to about {bias} µg/m³. Suggested offset: −{bias} µg/m³. |
| `DRIFT_DOWN` | warn | Has read lower than nearby stations for {hours} hours, growing to about {bias} µg/m³. Suggested offset: +{bias} µg/m³. |
| `SAT_AGREES` | ok | Regional model data shows a similar level ({model} µg/m³). |
| `SAT_DISAGREES` | info | Regional model data shows {model} µg/m³; model data is coarse, so this has little weight. |
| `REGIONAL_EVENT_CONFIRMED` | ok | A regional pollution event is under way: nearby stations are also {factor}× their usual level. |
| `REGIONAL_EVENT_REFS_ONLY` | ok | A regional pollution event is under way (stations only; model data unavailable). |
| `SAT_ONLY_CAP` | info | Only regional model data was available, so the label cannot be higher than Plausible. |
| `HUMIDITY_CORRECTED` | info | Humidity correction applied. |
| `HUMIDITY_CORRECTION_SKIPPED` | info | No humidity correction exists for this sensor model, so a wider tolerance was used. |
| `HUMIDITY_CORRECTION_EXTRAPOLATED` | info | Concentrations were above the range the humidity correction was fitted on, so a wider tolerance was used. |
| `SEQ_GAP` | info | {n} readings are missing before this one. |
| `BUFFERED_UPLOAD` | info | Stored offline and uploaded {lag} later. |

### 6.11 States and errors

| Situation | Behaviour |
|---|---|
| Loading | Skeletons shaped like the final content; no spinners longer than 400 ms without skeleton |
| No data | Message plus the next action ("No readings in this range. Widen the date range.") |
| API error | Inline banner with the error code and a *Retry* button; page content stays |
| Agent text pending | Template text shown immediately; replaced when the agent output passes grounding. Chip shows *Rules* until then |
| Agent output rejected | Template text stays; chip shows *Rules*; no error shown to the user |
| Pending window | Label *Pending*, score shown as "—", tooltip "Waiting for official stations to publish this hour" |

### 6.12 Accessibility and responsiveness

- Breakpoints: ≥ 1024 px two-pane layouts; 640–1023 px single pane with the side card as a bottom sheet; < 640 px list-first, map behind a toggle.
- All interactive elements are keyboard reachable with a visible 2 px focus ring.
- Trust changes announce through an `aria-live="polite"` region.
- Charts expose a data table alternative and do not rely on hover.
- Animations are limited to 150 ms fades and are disabled under `prefers-reduced-motion`.

## 7. Demo narrative (3 min 30 s, recorded in replay mode)

| Time | Beat | What the viewer sees | Rubric item served |
|---|---|---|---|
| 0:00–0:20 | Hook | "A signature proves origin, not truth." Map with live pseudo-sensors, replay badge visible | Impact, Category |
| 0:20–0:50 | Verify | Open a reading, show the reasons, run the verify script, `VERIFIED` | Technological |
| 0:50–1:30 | Drift | Inject drift. Score erodes, drift flag appears. Episode opens; Investigator's *Test it* line shows the offset restoring *Verified*; Explainer tabs | Agentic, UX |
| 1:30–1:50 | Spike and stuck | Both flagged with reasons | Technological |
| 1:50–2:20 | Tamper and replay | Judge-style edit of a reading → *Rejected*, proof fails. Replay → `409 SEQ_REPLAY` | Technological, UX |
| 2:20–2:45 | Real regional event | Values rise together; windows stay *Verified*; `REGIONAL_EVENT_CONFIRMED` | Impact |
| 2:45–3:00 | Operator rewrites history | Mirror check fails; red banner | Agentic, Technological |
| 3:00–3:15 | Synthetic scenario | Type a scenario in English; spec appears; Evidence page numbers | Agentic, Technological |
| 3:15–3:30 | Close | Architecture slide naming Foundry, Agent Framework, Azure MCP; export download | All |

If any beat is cut under `scope.md` §10, the video skips it and the time moves to the Evidence page.

## 8. Success metrics and release gates

**Release gates (block submission):**

| Gate | Threshold |
|---|---|
| Unsigned, tampered and replayed readings rejected in the automated suite | 100% |
| Determinism replay check (NFR-07) | 0 mismatches in 1,000 |
| `docker compose up --build` followed by `make demo` on a clean machine | Reaches the demo state |
| Demo script run end to end | No manual intervention |

**Report-only metrics (published on the Evidence page, with actual values even if targets are missed):**

| Metric | Target |
|---|---|
| Spike recall (factor ≥ 6) | ≥ 0.90 |
| Stuck-value recall (≥ 60 min) | ≥ 0.95 |
| Drift recall at 1 µg/m³/h | ≥ 0.90 |
| Median time-to-detect, drift at 1 µg/m³/h | ≤ 12 virtual hours |
| False-positive window rate, healthy sensors | ≤ 0.05 |
| False-positive window rate, healthy sensors in regional events | ≤ 0.10 |
| Mean absolute error by trust band | Strictly decreasing from Suspect to Plausible to Verified, with ≥ 30 windows per band |
| Explanation grounding pass rate (deterministic check) | ≥ 0.95 |
| Foundry groundedness score on 50 explanations (P1) | Reported; no target |
| Usability: 3–5 non-technical testers explain a flag in under a minute | ≥ 3 of 5 succeed |
| Signing time on ESP32 (P1) | Reported in µs |

## 9. Real-world applicability

| Question | Answer |
|---|---|
| Can it be deployed in production? | Stateless validator, one log service, one worker, PostgreSQL. Self-hostable with Docker Compose; hosted reference deployment on Azure. |
| What does a node cost? | ESP32 board, PMS5003 and BME280, about US$50 or less. |
| Does it work offline? | Yes, readings are buffered and signed with sequence numbers; they stay verifiable after late upload. |
| What are the limits? | Stated in the README: signing proves origin not truth; "tamper-evident", not "tamper-proof"; reference stations are not street-level ground truth; enrolment trust is weak in the MVP; satellite-assimilated data is coarse; device clocks can be wrong. |

## 10. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Judges expect a blockchain | Medium | Present the log as ledger-agnostic; show the immutable mirror; explain the energy trade-off in one slide |
| "Signed garbage" objection | High | Open the demo with it; cross-validation is the answer |
| Arbitrary score weights | Medium | Weights are published; the Evidence page shows error by trust band |
| Scope: three shallow components | Medium | Cut order in `scope.md` §10; P0 first |
| Reference data gaps or outage during recording | Medium | Replay mode uses a committed snapshot, not live calls |
| LLM output errors | Medium | Grounding gate, rule-based fallback, deterministic scores |
| Azure MCP tool names differ from expectation | Medium | Day-3 tool listing; SDK fallback documented in `spec.md` S22 |
| Agent Framework API changes between releases | Medium | Version pinned; all framework code isolated in `verisense/agents/` |
