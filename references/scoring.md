# Scoring — Priority Scoring and Ranking

Executed in the Feasibility & Ranking stage. For each candidate, output Priority (P0–P3), Evidence Level (R0–R6), Confidence (high/medium/low), and Blockers.

## Scoring table (six weighted dimensions)

| Dimension | Weight | 0 points | 5 points |
|---|---|---|---|
| Evidence the vulnerability exists in the target kernel | 20% | R0 | R1+ with complete backport verification |
| Ordinary-app reachability | 20% | R2 does not hold | R2 holds with high parameter controllability |
| Public Root escalation evidence | 25% | No public argument | R5/R6 with evidence passing the checklist |
| Vulnerability reproduction reliability | 15% | No trigger evidence at all | Trustworthy public reproduction with clear trigger conditions |
| Platform mitigations and their applicability | 10% | Key mitigations fully applicable | Mitigations inapplicable or publicly bypassed |
| Experiment and validation cost | 10% | Requires many devices/firmware/long cycles | An ordinary device + adb suffices |

Execution rules:
- For every dimension, **write the reasoning and evidence references first, then the number**. A dimension without supporting evidence scores 0 and goes into Blockers; never give a middle score from impression.
- "Public Root escalation evidence" counts only **material that passed the root-feasibility.md checklist**; unverified claims are treated as leads (≤1 point). **This dimension is about the existence of trustworthy, independently verifiable success records, not about asking the model to guess a success rate.**
- Total = Σ(dimension score/5 × weight).

## Priority mapping

| Priority | Condition (highest matching applies) |
|---|---|
| P0 | R≥4 and passed verification; or R3 with no mitigation block and sufficient reachability (worth immediate investment) |
| P1 | R3 with trustworthy reachability; or R2 with strong public evidence not yet verified (worth preparing validation) |
| P2 | R2/R3 but heavy Blockers (competing devices / unclear mitigations / high cost) |
| P3 | R0–R1, or already blocked by mitigations (record only, no investment) |

The P level must be consistent with the R level: below R2 may not be P0; an R6 candidate is handled as "target already achieved" — assess its research applicability rather than investing in development.

## Confidence

- High: every R-level evidence item is first-party and directly matches the target.
- Medium: second-hand sources, indirect inference (e.g. vendor-branch inference), or approximate device matching exist.
- Low: key verification depends on unobtainable material, or the conclusion rests on a single path.

Confidence describes **how robust the conclusion is with respect to the evidence**; low confidence does not automatically lower the P level but must be declared prominently in the report.

## Output format

```
### CVE-YYYY-NNNNN scoring
- dim evidence(20%): <x>/5 — <reasoning + references>
- dim reachability(20%): <x>/5 — <reasoning + references>
- dim public-lpe(25%): <x>/5 — <reasoning + references>
- dim repro-reliability(15%): <x>/5 — <reasoning + references>
- dim mitigations(10%): <x>/5 — <reasoning + references>
- dim cost(10%): <x>/5 — <reasoning + references>
- weighted: <0-100>
- Priority: P<n> (triggered condition)
- Evidence Level: R<n>
- Confidence: high/medium/low
- Blockers: <specific list of missing evidence>
```

## Ranking

The ranking table in report §3 is sorted by Priority ascending (P0 first); within the same level, by Confidence descending, then by weighted score descending. CVSS does not participate in ranking.

## Gates (end of this stage)

- [ ] Every candidate's every dimension has reasoning and references, or an explicit 0 + Blocker
- [ ] No dimension is scored on speculation
- [ ] Priority and Evidence Level consistency has been checked
- [ ] Candidates placed at P3 have a downgrade/exclusion reason, recorded in report §6