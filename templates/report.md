# Hokorobi — Android Root Vulnerability Research

<!-- Generation rules:
  - File name: Hokorobi-Report-YYYY-MM-DD.md (date = report generation date)
  - Ordering: §3 and §4 are both sorted by Priority (P0→P3), then Confidence high→low, then weighted score
  - §4: every subsection of every candidate is mandatory; write "none" or "unverified (reason)" when empty, never leave blank
  - All dates use YYYY-MM-DD; every link carries an access date
  - The report must not contain escalation payloads, arbitrary read/write implementations, or mitigation bypass code (boundary statement in §8)
  - Write the report in the language requested by the user; this English scaffold is the default. Translate headings and labels when another language is requested. Keep code symbols, CVE IDs, commit hashes, and paths unchanged.
-->

## 1. Research Scope

- Research period: <!-- candidate CVE disclosure window -->
- Target device: <!-- model / vendor / SoC -->
- Kernel version: <!-- branch + exact version -->
- Architecture:
- Build fingerprint:
- Research objective: <!-- e.g. screen for directions that can develop into a generic LPE -->
- Verification sources: <!-- target kernel source repository URL / image information -->

## 2. Executive Summary

- Total candidates: <!-- N -->
- Confirmed affected candidates: <!-- R1+, N -->
- Root-relevant candidates: <!-- R2+, N -->
- Public LPE evidence: <!-- R4+, N (one sentence + verification conclusion each) -->
- Major uncertainties: <!-- global Blockers, 3–5 items -->

## 3. Root Research Ranking

| Rank | CVE | Type | Reachability | Evidence | Difficulty | Priority |
|------|-----|------|--------------|----------|------------|----------|
| 1 | | | yes/conditional/no | R<n> | high/medium/low | P<n> |

## 4. Detailed Findings

### CVE-YYYY-NNNNN

#### Overview
- Disclosure date:
- Subsystem:
- Vulnerability type:
- Affected versions:
- Fix commit:

#### Target Verification
- Kernel image and hash: <!-- or source repo + branch/tag -->
- Patch status: <!-- affected / patched / uncertain -->
- Source or binary evidence: <!-- file + function, or commit-archaeology conclusion -->
- Vendor backport uncertainty: <!-- item by item: none / partial merge / rewrite, etc. -->

#### Reachability
- Entry point: <!-- complete call chain -->
- Required privileges: <!-- untrusted_app / conditional... -->
- Relevant kernel configuration:
- Target-specific restrictions: <!-- SELinux/capability/HAL intermediate layers -->

#### Root Cause Analysis
- Defective behavior:
- Affected objects:
- Security invariant:
- Supporting evidence: <!-- first-party evidence + access date -->

#### Race Assessment
- Evidence level: L<0-4> <!-- + actual acquisition path A/B/C/D -->
- Source revision / commit:
- Race required: yes/no
- Synchronization mechanisms:
- Object lifetime summary:
- Patch difference:
- Target applicability:
- Window evidence: <!-- for L4, cite source + numbers; otherwise mark theoretical -->
- Reproduction observations: <!-- state the environment -->
- Uncertainty:

#### Privilege Escalation Assessment
- Potential security impact: <!-- info-leak/DoS/cred-tamper/selinux-escape -->
- Public LPE evidence: <!-- per-material verification conclusion -->
- Android applicability:
- Existing limitations:
- Unverified assumptions:

#### Evidence Axes
- Patch status (target): <!-- affected / patched / uncertain / unverified -->
- Reachability (target): <!-- reachable / conditional / blocked / unverified -->
- Public LPE evidence: <!-- none / lead / R4-argument / R5-cross-device / R5-target / R5-conditional -->
- Target validation: <!-- none / L4-debug / cross-device / target-device -->
<!-- The R level is a display summary; these axes are independent and must not be derived from it. -->

#### Research Decision
- Evidence level: R<n> <!-- one line of grant reasoning per tier -->
- Research priority: P<n>
- Confidence: high/medium/low
- Key blockers: <!-- point to specific obtainable evidence -->
- Recommended validation objectives: <!-- what to validate next, and what counts as success -->

#### References
- Official advisories:
- Upstream fix:
- Public technical research:

## 5. Comparative Assessment

<!-- Compare candidates on Root relevance, evidence quality, target applicability, and validation cost. Tables allowed. -->

## 6. Excluded and Deferred Candidates

| CVE | Status | Reason | Stage | Re-entry trigger |
|-----|--------|--------|-------|------------------|
| | excluded/deferred | <!-- patched / unreachable / evidence unavailable --> | gate 0x | <!-- what new evidence would bring it back --> |

## 7. Research Recommendations

<!-- Priority-ordered research directions, missing evidence and acquisition plans, and directions explicitly not recommended for further work. -->

## 8. Boundaries & Method Notes

- This report assesses Root escalation research value. It does not contain escalation payloads, arbitrary kernel read/write implementations, or mitigation bypass code; experimental validation is performed separately.
- Scoring and evidence-level definitions are in the Hokorobi skill's references/scoring.md and references/root-feasibility.md.

## 9. Sources

<!-- Original links and access dates (YYYY-MM-DD). -->