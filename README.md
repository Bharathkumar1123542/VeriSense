# VeriSense

> Verifiable low-cost climate sensing: signed readings → Merkle log → cross-validated trust score → agent explanations.

VeriSense is a verification layer for low-cost climate sensors. A sensor signs every reading at the source using Ed25519. The reading enters a tamper-evident public Merkle log. A deterministic validator cross-checks it against independent official station data and coarse satellite-assimilated model data. The output is a 0–100 trust score, a label, and a provenance trail that anyone can re-verify.

An agentic layer sits around that deterministic core to investigate suspicious data stretches, explain anomalies in plain language, generate synthetic test scenarios, and mirror log tree heads.

## Documentation

- [Scope](file:///c:/Users/Bharath%20kumar/VeriSense/docs/scope.md) — Problem statement, goals, non-goals, architecture overview
- [PRD](file:///c:/Users/Bharath%20kumar/VeriSense/docs/prd.md) — Product requirements, user personas, UI specifications
- [Spec](file:///c:/Users/Bharath%20kumar/VeriSense/docs/spec.md) — Technical specifications, contracts, cryptographic log, validator algorithms
- [Checklist](file:///c:/Users/Bharath%20kumar/VeriSense/docs/checklist.md) — Implementation schedule and step checklist
