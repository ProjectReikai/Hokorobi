# Stage Gate — Stage Gate Checklist

For every candidate, **after each completed stage**, run the corresponding gate. Output one of three: `pass` (advance to the next stage) / `downgrade` (cap the R level, retain) / `exclude` (exclude, record in report §6).

General rules:

- Gate decisions are based on **recorded evidence**, not expectations. Work you plan to do is not evidence.
- `downgrade` / `exclude` must include a one-line reason, and you must check pitfalls.md §7: this is an evidence problem, not an effort problem.
- An `exclude`d candidate's R level is **frozen at the exclusion point** (see "R-level semantics" in SKILL.md); later analysis does not change its R level.
- If a candidate reaches the same gate a second time at the same R level with no new evidence → automatic `exclude` (prevents sunk-cost loops); note in state "conditions that could re-include it".

## 00 Source Gate (run first; determines analysis depth)

Before Patch Verification and Race & Lifetime Analysis, determine the available source/binary material and thus the permitted analysis depth (corresponding to the L levels in race-analysis.md):

| Situation | Next action |
|---|---|
| Original vulnerable code and fix diff available | Full source analysis allowed (≥L1) |
| Only the fix diff | Partial analysis allowed, marked incomplete context (L1, restricted) |
| Only the CVE description | **Stop deep analysis**, mark source pending (L0) |
| Target GKI branch source found | Verify the target baseline, but keep vendor-modification uncertainty (L2) |
| Target binary obtained | Binary-level patch checking allowed (L3); requires disassembly tools, otherwise stop at L2 |
| Only an approximate kernel version | **Must not claim the target kernel is confirmed affected** |

Also confirm that `target-mode` (live / frozen) has been determined and written to state; in frozen mode, run R6 dedup before gate 06 (EOL devices have dense public exploits; see the frozen-mode section in discovery.md).

Record format: `- gate 00: source = upstream+diff, GKI branch android15-6.6, target-mode = live → L1 (target applicability unconfirmed)`

## 01 Discovery gate

- [ ] At least two independent sources (CVE ID or fix commit + a verifiable technical source); commit-only candidates (the fix commit page as the sole first-party source) may advance, labeled `channel: commit-only`
- [ ] Type/subsystem marked
- [ ] Retrieval followed the three-segment order (commit stream → CVE channel → semantic targeting), not broad websearch or model memory
- [ ] Every candidate's disclosure date verified inside the window; commit-channel candidates marked consequence: TBD and smell
- [ ] **Enumeration completeness**: how many ASB issues were fetched, whether lore/cvelistV5 enumeration covered back to the window start, whether unreachable gaps are recorded in Blockers — uncovered parts must not be written as "complete" in state, only "partially complete (gap: …)"
- Failing → stay in `needs-verification`, do not enter Patch Verification (except commit-only candidates).

## 02 Patch Verification gate → determines R1

- [ ] Source Gate (gate 00) passed (at least an upstream fix diff or an explicit version boundary); otherwise this stage cannot start
- [ ] Conclusion is affected / patched (code-verified) / patched (version-inferred) / uncertain
- [ ] Quick-exclusion candidates: all four preconditions (verifiable stable tag, GKI scope, non-vendor module, non-OEM/third-party build) verified item by item, on file
- [ ] Backport trap checklist checked item by item, on file
- [ ] With only an approximate kernel version, no claim that "the target kernel is confirmed affected"
- `patched` (either path, preconditions complete) → **exclude**: record in §6 with the exclusion path.
- `affected` → pass, grant R1.
- `uncertain` → downgrade, cap the R level at R0, list the missing material in Blockers.

## 03 Reachability gate → determines R2

- [ ] Component presence verified (CONFIG / vendor modules / Android semantic differences); component absent → exclude
- [ ] Complete call chain (missing links explicitly marked)
- [ ] SELinux / capability / seccomp constraints checked item by item
- [ ] Candidates from generalized Linux searches: their desktop-context assumptions re-checked item by item against Android
- `blocked` (component absent or entry hard-denied) → **exclude**: the R level is capped at the gate 02 conclusion; **when gate 02 was not executed (terminated early by pre-screening), record R0**, note `unreachable; patch state not verified`, and record under §6 "unreachable".
- `reachable` → pass, grant R2.
- `conditional / indirect` → pass and keep the conditions as Blockers; R2 marked conditional.

## 04 Root Cause gate → determines R3

- [ ] Root cause class + statement of the broken invariant
- [ ] First-party evidence support (commit/mailing list)
- Failing → downgrade, cap at R2. Logic/permission types must not use the wrong framework (pitfalls §9).

## 05 Race & Lifetime gate (does not change the R level)

- [ ] Source Gate (gate 00) executed, L level determined and not skipped
- [ ] Lifetime table and lock-relationship table based on actual code (≥L1); with no source, stop at L0 without inventing structure
- [ ] Window evidence labeled with its source tier; no probability numbers when not L4
- [ ] No environment-extrapolation statements (pitfalls §3)
- [ ] App CPU/background constraints accounted for
- Pass or fail only affects the Difficulty score, it does not block the flow (except L0: at L0 mark the analysis incomplete).

## 06 Privilege Escalation gate → determines R4–R6

- [ ] Every public LPE evidence item passed the 6-item checklist
- [ ] Impact classification has no cross-tier inference
- [ ] R levels granted tier by tier with reasons; no skips
- No public argument → the R level stays at R3 (this is a normal state, not a failure). Claimed but unverified → stay at R3 and downgrade the claim to a lead.

## 07 Feasibility & Ranking gate

- [ ] All six scoring.md dimensions have reasoning + references, or an explicit 0 + Blocker
- [ ] Priority and R-level consistency check passed
- [ ] P3 candidates have exclusion reasons

## 08 Review & Report gate (global)

- [ ] Every detailed entry contains Blockers and "unverified key conditions" (hard rule 6)
- [ ] Failed/downgraded candidates all have attribution (pitfalls §6)
- [ ] Ranking follows P0–P3, not CVSS
- [ ] Sources include original links and access dates
- [ ] state/ has been synchronized to the report's final conclusions; a copy of the report is archived outside the run

## Gate record format (write to state/candidates.md)

```
- gate 02: affected → R1 (evidence: <commit/file location>, 2026-10-30)
- gate 03: reachable → R2 (SELinux policy reference)
- gate 06: R4 denied — public repo reproduction failed checklist item 3 (crash screenshot only, no root validation)
```