---
name: hokorobi
description: Android kernel vulnerability discovery and Root feasibility research. Search candidate CVEs, verify patches against a target kernel, judge unprivileged reachability, analyze root causes and races, review public LPE evidence, and rank candidates by Root research value (R0-R6 evidence levels, P0-P3 priority). Use when the user asks to find, analyze, or rank Android kernel vulnerabilities for local privilege escalation research, or to produce a Hokorobi report. This skill does not write exploits, privilege escalation payloads, arbitrary kernel read/write primitives, or mitigation bypass code.
---

# Hokorobi

Android Kernel Vulnerability Discovery & Root Feasibility Research.

Final goal: **identify kernel vulnerability directions that can realistically develop into an Android local Root privilege escalation chain** — not merely finding CVEs, analyzing UAFs, or producing reports. Every candidate must answer two questions:

1. Does the vulnerability exist in the target kernel?
2. Does it have the necessary conditions for a Root chain (unprivileged reachability, boundary violation, credible evidence)?

Output: a Markdown research report. Hokorobi does not write escalation payloads, arbitrary kernel read/write primitives, or mitigation bypass code; experimental validation stays separate.

## Language Policy

- Write skill instructions in English.
- Preserve original technical identifiers and terminology.
- Analyze source code and patches in their original language.
- Generate final research reports in the user's requested language.
- Keep code symbols, CVE IDs, commit hashes, and paths unchanged.
- Quote original English evidence when necessary.
- Follow the user's explicitly requested output language.

## Hard Rules (never violate at any stage)

1. **Never infer patch status from kernel version numbers or SPL dates.** Sole exception: the "quick exclusion path" in references/patch-analysis.md (GKI kernel + explicit version boundary + non-vendor module + non-OEM/third-party build; all four preconditions met), and the conclusion must be labeled `version-inferred` to distinguish it from code verification. Otherwise verify the actual code or fix commit in the target kernel.
2. **Never fabricate race window sizes without measurement.** Separate theoretical estimates from measured data and label which one applies.
3. **Never extrapolate debug-kernel (KASAN, debug config) trigger results to production devices.**
4. **Never treat WARN / Oops / KASAN reports as Root success.** Memory corruption evidence ≠ privilege escalation.
5. **Never treat a public device-specific rooting project as a generic Android Root.** Verify its target device, kernel image, and validation method.
6. **Every report must record why expected outcomes were not achieved and which key conditions remain unverified** (Blockers are mandatory).
7. **Downgrade or exclude candidates with insufficient evidence or unmet conditions promptly**; do not keep reinvesting.
8. **No automatic R-level jumps.** R3 never auto-promotes to R4 because it "looks exploitable"; R5 never auto-promotes to R6 because "a similar device worked". Only new credible evidence promotes a level.
9. **Search, don't recall.** Every candidate, piece of evidence, and source must come from a page actually fetched in this run, with a link attached. Never introduce candidates from model memory (well-known historical CVEs are not a source).

## Workflow (eight stages)

```
Discovery → Patch Verification → Reachability → Root Cause Analysis
→ Race & Lifetime Analysis → Privilege Escalation Assessment
→ Feasibility & Ranking → Review & Report
```

Before a candidate enters Patch Verification and Race & Lifetime Analysis, run the **Source Gate (gate 00)** in [checklists/stage-gate.md](checklists/stage-gate.md): determine available source/binary material and the permitted analysis depth (L0–L4 in references/race-analysis.md). L1 (upstream source + fix diff) is the default mode; L2/L3 are enhanced modes. Never require the user to provide a complete kernel up front.

At the end of each stage, run the corresponding gate: pass, downgrade, or exclude. Downgraded/excluded candidates are recorded in state and explained in report §6; never re-analyze them.

## Before you start: state recovery and target resolution

1. Look for `hokorobi-runs/*/state/` in the working directory.
2. If found: load `state/candidates.md` and `state/progress.md`, resume from the interruption point, and **never repeat completed verification**.
3. If not found: create `hokorobi-runs/<device>-<date>/` and initialize state files.
4. **Resolve the target** (two paths):
   - The user provides a kernel image (boot.img / vendor_boot.img / Image / vmlinux): follow references/image-intake.md — **do not ask the user about kernel internals; the image speaks for itself**. Produce kernel version, GKI generation/branch, build commit, real config (IKCONFIG), snapshot anchor; write to state.
   - The user provides a device/SPL: determine `target-mode: live | frozen` (branch EOL / device no longer updated / user declares a fixed version → frozen), and confirm the GKI release tag for the device snapshot (`<branch>.<N>_rNN`, the last `_r00` tag merge before SPL). **SPL ↔ tag mapping is an inference**: use build fingerprint when available, otherwise record it as a Blocker.
   - Neither is available (a bare "find bugs in 6.12" prompt): **ask for the target device or image first**; never choose an anchor unilaterally.
5. Update `state/candidates.md` immediately after each stage of each candidate (candidate, current stage, R level, evidence links, Blockers). Update `state/progress.md` on interruption.

## R-level semantics

- A candidate excluded by a gate has its R level **frozen at the exclusion point** (e.g., a candidate blocked at gate 03 is recorded as R1 even if later root-cause analysis would reach R3; record that in notes only).
- A downgraded candidate is capped at its current value until new evidence lifts it.

## Execution efficiency (mandatory)

1. **Batch fetching, unified analysis.** Before each stage, list all independent data requests, fire them in parallel (multiple curls in one shell command / background jobs; several websearch/webfetch calls at once), then do **one** consolidated reasoning pass before the next fetch. **Never interleave fetch → think → fetch one at a time** — this is the main cause of runaway run time (measured: a run can drop from ~1 hour to ~15 minutes).
2. **Use a local shallow clone for commit streams, not paginated HTTP.** One-time:
   `git clone --filter=blob:none --shallow-since=<anchor date minus 30 days> --no-checkout --single-branch --branch <branch> <repo>`
   (measured ~9 s). All `git log --grep` / `--name-status` queries then run locally in 0.01–0.03 s, replacing dozens of paginated pages (measured: 59 pages → 1 clone). Gitiles `n=1000` single-page fetching is the fallback.
3. **name-status before diff.** In a blobless clone `git log --name-status` is instant (tree objects); `git show -p` lazily fetches blobs on first use (measured ~30 s per commit, then cached). Triage files with name-status; only fetch full diffs for Triage top-K, in batch/parallel.
4. **Generate parsing artifacts once, reuse everywhere.** The image's `vmlinux.elf` / symbol table / BTF offsets are produced once during intake and stored in `state/`; every candidate reuses them. Never re-disassemble per candidate.
5. **Tiered budget.** Discovery does subject/symbol-level filtering only; L3 binary verification goes to Triage top-K only (default ≤3); full deep-dive candidates default ≤3. Record wall-clock time per stage in `state/progress.md` to locate bottlenecks.
6. **Cache and deduplicate.** Fetch each URL / commit / file once per run; store under `state/raw/` and reuse. Evidence shared across candidates (same CVE record, same file blob) is never re-fetched.
7. **Fingerprint cache and artifact reuse** (L3 is typically ~60% of total time):
   - `state/cache/fingerprints.md`: key = **`Image SHA-256` + fix commit + function/field name**; record verdict, fingerprint class, and evidence. Later candidates on the same OEM build and cross-run resumes hit the cache without re-disassembling. Entries carry build fingerprint (version string / Build ID) and config digest; a build change invalidates the cache automatically.
   - **Intake artifacts are reused across runs by Image SHA**: `Image` / `config.txt` / `vmlinux.elf` / `symbols.txt` are reused when the SHA matches (verify SHA first); symbol recovery is the single largest one-time cost.
8. **Disassemble by symbol, not the whole image.** macOS LLVM objdump does **not** accept `--disassemble=<symbol>` (measured: unknown argument). Portable recipe:

   ```bash
   # Get symbol start/end via nm/llvm-nm ordering, then disassemble the range
   llvm-nm -n state/vmlinux.elf | awk '/ <target>$/{getline; print $1}'
   objdump -d --start-address=0x<start> --stop-address=0x<end> state/vmlinux.elf
   # With GNU objdump: objdump -d --disassemble=<symbol>
   ```

   Dump once to `state/disasm/<func>.txt` for reuse across candidates and runs. Symbol recovery (vmlinux-to-elf) runs once per build.
9. **Pre-screen reachability to terminate early** (OEM / L3-heavy targets): before L3, run a cheap gate 03 pre-check — IKCONFIG component presence + entry surface (sepolicy allow / seccomp / capability). Clearly unreachable candidates are **terminated immediately** without L3, recorded as `unreachable; patch state not verified`. This does not reduce key verification: patch status of an unreachable candidate has zero research value.
10. **Never cut key verification to save time.** What may be cut: L3 for unreachable candidates, repeated parsing, serial fetching. What may not: ≥2 independent fingerprints for reachable candidates, gates, evidence trails.

## Reference routing

Load only the file for the current stage, never all at once:

| Stage | Reference |
|---|---|
| Discovery | references/discovery.md |
| Patch Verification | references/patch-analysis.md |
| Reachability | references/reachability.md |
| Root Cause Analysis | references/root-cause.md |
| Race & Lifetime Analysis | references/race-analysis.md |
| Privilege Escalation Assessment | references/root-feasibility.md |
| Feasibility & Ranking | references/scoring.md |
| Review & Report | templates/report.md |
| Any uncertainty | references/pitfalls.md |
| Any source lookup | references/sources.md (fixed URL list, preferred over free search) |
| User supplies a kernel image | references/image-intake.md |

## Evidence levels (two independent scales, never mix)

**L0–L4: analysis evidence depth** (Race & Lifetime Analysis) — L0 advisory only, L1 upstream source + fix diff, L2 target source, L3 target binary, L4 diagnostic runtime evidence. See references/race-analysis.md.

**R0–R6: Root escalation evidence maturity** (overall assessment):

| Level | Evidence state | Meaning for Root research |
|---|---|---|
| R0 | CVE description only | Cannot judge yet |
| R1 | Vulnerable code exists in the target kernel | Research prerequisite met |
| R2 | Reachable under normal app privileges | Real attack surface exists |
| R3 | Root cause confirmed by credible evidence | Memory-safety or permission-defect evidence |
| R4 | Public local-privilege-escalation technical argument | Stronger Root research value |
| R5 | Public escalation reproduction on a comparable device | Applicability can be studied |
| R6 | Independently verified Root result on the target device | Target achieved |

Full criteria and promotion/demotion rules: references/root-feasibility.md. An R level reflects the **highest verified evidence tier**, never speculation.

## Output

Final artifact: `hokorobi-runs/<run>/Hokorobi-Report-YYYY-MM-DD.md`, using templates/report.md, sorted primarily by Root Research Priority (P0–P3), not CVSS. Report body follows the user's requested language.

Success criterion: the report states which vulnerabilities have credible Android local Root research value, how far the evidence supports them (R level), which preconditions remain unmet (Blockers), and which directions are not worth pursuing.