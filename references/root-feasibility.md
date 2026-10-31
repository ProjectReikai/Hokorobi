# Root Feasibility — Escalation Evidence Assessment

**This file is Hokorobi's core constraint**: how to assess local-escalation evidence, and how to distinguish "vulnerability reproduction" from "actual privilege escalation". The Privilege Escalation Assessment stage executes this file; Feasibility & Ranking relies on its conclusions.

## Distinctions that may never be crossed

Vulnerability existence, vulnerability triggering, memory corruption, and Root escalation are **four independent factual tiers**; each tier requires its own evidence, and lower tiers may not be used to infer higher ones:

```
exists(R1) → reachable(R2) → root cause confirmed(R3) → [technical argument(R4)] → [comparable-device reproduction(R5)] → [target-device success(R6)]
```

Typical errors (actually made historically):
- UAF exists → infer "arbitrary kernel read/write achievable". Wrong. Between a UAF and arbitrary read/write lie unproven steps: heap layout, object reuse, type confusion.
- Kernel crash/WARN/KASAN report → declare "exploitation succeeded". Wrong. A crash is a failed form of corruption, not privilege escalation.
- A public repo claims root → declare "this CVE is escalatable". Wrong. Verify its device, image, and validation method (see below).

## Complete R-level criteria

| Level | Criterion (all must hold to grant the level) |
|---|---|
| R1 | Vulnerable-shape code verified in the target kernel (including vendor branches/DLKM); backport traps excluded |
| R2 | R1 + a complete call chain reachable under ordinary app (untrusted_app) privileges, with no SELinux/capability hard block |
| R3 | R2 + root cause confirmed by first-party evidence (fix commit/maintainer analysis) and consistent with the target code shape |
| R4 | R3 + a public local-escalation **technical argument** exists: written analysis or **readable implementation** explaining how the defect advances to a permission-boundary violation (e.g. controllable object reuse → cred overwrite path), with every step backed by code or experiment |
| R5 | R4 + **verifiable public escalation reproduction** (marked with the three tiers below; never merged) |
| R6 | R5 + an independently verifiable Root result on the **target device** (same model / same kernel version / same build), with a trustworthy validation method |

### R5 three-tier markers (must be distinguished, never equivalent)

| Marker | Definition | Usage restriction |
|---|---|---|
| **R5-target** | Actual reproduction on the target device and target kernel build | Highest tier; may serve as a prerequisite for R6 |
| **R5-cross-device** | Real reproduction on another device, with verifiable evidence | May raise research priority; **does not prove the target device is exploitable** |
| **R5-conditional** | Reproduction evidence exists, but build differences, reproduction conditions, or evidence completeness are unconfirmed | Must not be merged into verified evidence |

Hard requirements:

- **Same kernel branch ≠ same binary**; no direct extrapolation. Cross-device conclusions must record build differences.
- **A third-party port without actual validation cannot alone constitute R5** (it counts only as supporting material for R5-conditional).
- Must record: reproduction device, kernel build (version string/build fingerprint), key config, reproduction result, and evidence source — each evidence item recorded independently; **evidence from different devices must never be merged** into one "verified" claim.
- Patch status, vulnerability reachability, and race exploitability are **three different dimensions** and cannot substitute for each other (patched ≠ unreachable ≠ unexploitable).

### Scope of L4

- **L4-debug**: trigger observations on a debug image (KASAN/KFENCE/DEBUG_PAGEALLOC etc.) prove only "triggerable on that build"; they are never extrapolated to production and do not participate in R5/R6.
- **L4** (production-grade): requires observations on a production build, or on an equivalent build after verifying build differences.

### On public root implementations (GitHub PoC / exploit repos)

This is typical R4 evidence material — no "bump one level" rule is needed; it should support R4 by itself, **provided it passes the verification checklist below** (readable code, trustworthy result validation, explicit target). Two disciplines:

- **Reproduction quality affects Confidence and Blockers, not the R level**: an imperfect/unstable implementation still grants R4 as usual, but Confidence drops to medium/low and a Blocker records "reproduction reliability not independently verified". The R level measures the evidence tier; Confidence measures conclusion robustness — expressing them separately carries the most information.
- **No automatic +1 rule of any kind**: a repo existing ≠ verification passed. A repo with only crash screenshots, no proof of privilege escalation, or an unclear environment (userdebug/permissive) does not support R4; downgrade it to a lead. Reaching R5 still requires a trustworthy reproduction on a comparable device (hard rule 8).

R levels reflect the **highest verified tier**. Any skipped intermediate tier must be filled in (e.g. R4 is impossible while R2 is unconfirmed).

### Promotion/demotion rules

- Promotion: accept only newly verified first-party evidence; never inference, analogy, or "industry consensus".
- Demotion: immediately demote and record the reason when evidence is found invalid (reproduction unverifiable, device mismatch, paper retracted).
- **No automatic jumps**: R3→R4 requires a public argument document; R5→R6 requires target-device validation. These two edges are the easiest to cross on wishful thinking.

## Verification checklist for R4/R5 public evidence

For every public material claiming escalation, verify item by item and record:

1. **Target explicitness**: are the device model, kernel version, and build fingerprint stated and relevant to this project's target?
2. **Reproducibility**: are complete steps/code provided, or only result screenshots in a video or README?
3. **Result validation method**: how is root proven? `id`/`su` output, SELinux domain, or merely "it didn't crash"? A crash ≠ root.
4. **Environment authenticity**: user build or userdebug/engineering build? SELinux enforcing or permissive? Bootloader unlocked? — a "root" on userdebug+permissive is worth far less for studying the real attack surface.
5. **Independence**: is there third-party reproduction, or only the author's own claim?
6. **Timeliness**: are the claimed kernel version and the fix timeline self-consistent? (Claims of successful escalation on a post-fix version deserve special scrutiny.)

If any item fails → that material cannot support the corresponding R level; downgrade it to a "lead" and record it in notes.

## Boundary analysis in the Android context

When assessing "where escalation leads", distinguish three outcomes and never mix them:
- **kernel cred tampering** (uid→0 but SELinux domain still restricted): limited value on some Android versions, constrained by SELinux + domain transitions.
- **SELinux escape** (entering a permissive/su domain or disabling enforce): a necessary condition for full root, Android-specific.
- **Information leak / DoS only**: when there is no evidence of an escalation path, classify honestly; it does not participate in P0–P1 ranking.

Mitigation applicability (linking to the corresponding scoring dimension): KASLR, PAN/PXN, CONFIG_KASAN (debug builds only), CFI, RODATA, SELinux state, and whether object allocation is in a protected region (e.g. cred has dedicated protection). Record the **applicability** of mitigations, not a vague "mitigations exist".

## Record format

```
### CVE-YYYY-NNNNN privilege escalation assessment
- potential impact: <info-leak / DoS / cred-tamper / selinux-escape / unknown>
- public LPE evidence: <checklist result for each material>
- android applicability: <the Android boundary analysis above>
- existing limitations: <limitations acknowledged by the authors of public material>
- unverified assumptions: <assumptions the material relies on but has not verified>
- evidence level: R<0-6> (one line of grant reasoning per tier)
- blockers: <specific missing evidence preventing promotion>
```

## Gates (end of this stage)

- [ ] Every public evidence item has passed the 6-item verification checklist; failures are downgraded
- [ ] The impact classification has no cross-tier inference (no statement deriving root from a crash)
- [ ] The R level is granted tier by tier with reasons; no skips
- [ ] Blockers point to specific obtainable evidence ("needs an actual measurement of XX on XX", not "insufficient evidence")