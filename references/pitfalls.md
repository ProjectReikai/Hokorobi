# Pitfalls — Hardened Lessons Learned

Core lesson: **vulnerability existence, vulnerability triggering, memory corruption, and Root escalation are different evidence tiers, and each requires independent evidence**. Whenever you catch yourself making a cross-tier inference at any stage, come back and reread this file.

Every pitfall abstracts a real historical research lesson; all apply to all candidates and none target a specific CVE.

## 1. Misjudging patch status

- **Symptom**: declaring "patched" or "affected" from a version number/SPL/bulletin date.
- **Correction**: only the target kernel's code shape or git archaeology counts. SPL lag, partial merges, and rewrite-style backports all really exist.
- **Check**: have you gone item by item through the backport trap checklist in patch-analysis.md?
- **Variant**: substituting upstream Linux source of some version for the android GKI branch source when drawing conclusions. A GKI branch (e.g. android15-6.6) is not equivalent to the upstream kernel with the same version number and cannot directly substitute for it.

## 2. Fabricating race windows

- **Symptom**: "running it 10^6 times should hit it", "the window is roughly a few hundred nanoseconds".
- **Correction**: no numbers without measurement. Label analysis evidence depth per L0–L4 in race-analysis.md; runtime numbers can only come from L4. Execution count has no necessary relation to hit probability.

## 3. Environment extrapolation

- **Symptom**: it triggered on syzkaller/KASAN/a debug kernel, so infer "triggerable on a real device".
- **Correction**: KASAN on a debug build widens the window and changes allocation timing. Label observations with their environment; they are weak evidence for "triggerability" only, never evidence of target-device behavior.

## 4. Confusing corruption with escalation

- **Symptom**: treating an Oops/WARN/KASAN report as "exploitation succeeded".
- **Correction**: a crash is a failure form. Escalation requires proof of cred tampering or a SELinux escape (boundary analysis in root-feasibility.md).

## 5. Generalizing device-specific projects

- **Symptom**: some repo got root on a particular user's phone → "this CVE gives generic root".
- **Correction**: six-item checklist: target, reproduction steps, result validation method, build type (user vs userdebug), SELinux state, independence. Results from userdebug+permissive are discounted.

## 6. Missing failure attribution

- **Symptom**: the report only says "did not succeed" without saying why.
- **Correction**: whenever an expected outcome is not achieved, record: what was assumed, what was observed, and at which step the gap lies (which evidence tier is missing). This is the map for the next research round.

## 7. Sunk-cost loop

- **Symptom**: repeated investment in one candidate while evidence stays at a low R level.
- **Correction**: at every stage gate, check: is there new evidence raising the R level versus the previous round? No → downgrade or defer, and move to the next candidate. Reserve `exclude` for clear inapplicability. Deferred/excluded candidates stay in state and report §6; a trigger (e.g. new public evidence) can bring them back.

## 8. False evidence from online fetch failures

- **Symptom**: webfetch cannot get the original tracker/repo, so filling in from media reports or memory.
- **Correction**: distinguish three cases:
  - **Anti-bot interception** (Anubis etc.): switch to a **content-equivalent mirror** per sources.md "Anti-bot interception and fallbacks". Git is content-addressed — the same commit hash on a mirror is the same object as on the primary source; **a mirror is first-party evidence and does not lower the evidence tier**.
  - **Material missing/deleted**: record in Blockers.
  - **Second-hand retelling** (security media/aggregators): can only locate leads, never support an R level.
- After recognizing an interception page, retry the same URL at most once, then switch mirrors; do not keep hitting it.

## 9. Framework mismatch for logic/permission vulnerabilities

- **Symptom**: inventing "race windows" and "object lifetimes" for auth-bypass-type vulnerabilities.
- **Correction**: choose the assessment framework by vulnerability type (classification in root-cause.md). Logic types go through permission-boundary analysis.

## 10. Distorted report ranking

- **Symptom**: ranking by CVSS or by "analysis depth".
- **Correction**: rank by Root Research Priority (P0–P3); low-confidence candidates that were not analyzed thoroughly go later, never earlier just because their write-up is long.

## 11. Structural inference without a code basis

- **Symptom**: writing a "Thread A lookup / Thread B free" lifetime-conflict story from the CVE description alone.
- **Correction**: Race & Lifetime Analysis must obey the Source Gate (stage-gate.md gate 00) and the L-level rules: without the affected function's source, do not produce a lifetime table; with only a fix diff, produce a partial analysis marked as incomplete context; with only an advisory, stop at L0 and do not go deeper.