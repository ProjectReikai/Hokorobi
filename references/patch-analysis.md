# Patch Verification — Patch Checking

Goal: confirm whether the vulnerable code / fix patch actually exists in the target kernel. This is the basis for R1.

## Core discipline

**Never determine patch status from any of the following**:

- Kernel version number (e.g. "6.1.75 already contains the fix")
- SPL / security bulletin dates (vendors may lag, partially merge, or re-implement fixes)
- Vendor statements (statement ≠ code)
- Android common branch status (vendor branches may merge independently or miss fixes)

The only trustworthy basis: **the code shape of the affected function in the target kernel's actual source**, or the real existence of the fix commit in the target branch (git archaeology).

There is, however, one **quick-exclusion path**: when all of the following preconditions hold, the candidate may be judged "patched" without code verification (the fix entered a version → code before the fix is necessarily broken → the target is built from that version or later → the target is unaffected):

## Quick-exclusion path (version inference, GKI kernels only)

**All must hold**:

1. The fix has entered an **explicit version boundary**: an upstream stable tag (v6.6.x) or an ASB/SPL bulletin explicitly states "fixed in", and the fix commit is locatable (hash verified, not retold).
2. The target runs a **GKI kernel** (a branch Google builds from a known tag, e.g. android15-6.6), and the target version/branch head is **later than** the tag containing the fix — the branch head is verifiable at `https://android.googlesource.com/kernel/common/+log/refs/heads/<branch>`.
3. The affected code is in the **GKI kernel body**, not in a vendor DLKM / OEM private module (GKI updates do not cover the latter).
4. The target is not an OEM deep-modified kernel or a third-party image (OEM/third parties merge fixes independently; their timing does not follow the GKI tag).

**Output requirement**: a quick-exclusion conclusion must be labeled `patched (version-inferred, GKI scope)`, distinct from code-level `patched (code-verified)`; the evidence record includes the fix tag, target branch head, and judgment date.

**If any precondition fails** → return to the full code-verification path. The following cases **never** allow version inference:

- Vendor modules / OEM private kernels
- User-supplied third-party images or self-built kernels
- Ambiguous fix boundaries (multiple backported commits, fix-the-fix, vendor rewrites)
- Uncertainty whether the target is a GKI device

Quick exclusion does not apply to the "confirm affected" direction — declaring vulnerable may only go through code verification (a version earlier than the fix tag ≠ the target was not fixed early by the OEM; but for GKI devices, if the target branch head is earlier than the tag containing the fix, you may label `affected (version-inferred, GKI scope)` and proceed to full verification to confirm).

**OEM GKI-based image special rules** (user supplied a boot.img and intake identified an OEM build; see image-intake.md): **the two paths are strictly separated and never mixed**.

| | GKI / official image | OEM image (kleaf / vendor tree) |
|---|---|---|
| Path determination | `g<hash>` lookup returns **200** → official/GKI | `g<hash>` lookup returns **404** → OEM (see image-intake.md 3.5) |
| Version string `g<hash>` | Locatable in kernel/common → exact anchor | **Marked `non-authoritative`, usable for no criterion at all** (built in a vendor-private tree) |
| Version number / tag | May support patch-status inference (quick-exclusion four preconditions / frozen rules) | **Only scopes candidates and ranks priority**; never infers patch status |
| Basis for patch conclusions | Code verification or version inference (labeled inferred) | **Only L3 binary fingerprints or real-device validation** |
| R1 grant condition | code-verified or inferred (GKI scope + preconditions met) | **Only L3 / OEM source / real device** |

- An OEM image's `build_date` is only a **ranking signal**: a fix merged later than build_date → high priority (physically impossible for it to be in the build), but still not a conclusion and still requires L3; a fix before build_date → low priority, and it still requires L3 to judge patched (the OEM may have merged only part of the fix).
- Extreme exception: the vendor fixed the same defect internally ahead of the public commit — therefore even "fix later than build" is only high priority, never a conclusion.

**L3 cherry-pick identification recipe** (judging on an OEM binary whether a fix was merged):

1. Extract "fix fingerprints" from the upstream fix commit, in four classes:
   - **New reference/release sequences**: refcount inc_not_zero, `fput()`, `atomic_long_inc_not_zero`, calls during the RCU grace period;
   - **Removed parameters or comparisons**: old checks deleted by the fix (e.g. `tail->len != last_len`) still present → unfixed;
   - **Function split / new symbols**: whether helpers introduced by the fix (e.g. `ep_remove_epi`, inlined `epi_fget`) appear in the symbol table / call graph;
   - **Call-site migration**: the fix moves an operation from function A to function B (e.g. `flush_sigqueue` from `release_task` to `exit_signals`) — the fingerprint is "call count in A should be 0 / in B should be 1"; this works best for migration-type fixes (measured case: a `flush_sigqueue` migration between task-exit functions).
2. On the target binary: `vmlinux-to-elf` recovers kallsyms → `objdump -d` locates the function → compare fingerprints; with DWARF/BTF, parse struct offsets first (`binder_transaction.to_proc`, `file.f_count`, `epitem.ffd`) before reading disassembly.
3. **At least two independent fingerprint classes** (symbol + call shape / struct offset) are required for a verdict;
   - **A missing symbol ≠ unfixed**: static helpers are often inlined, so an absent symbol is only weak evidence (measured: the absence of `timer_lock_sighand` counts only as supporting evidence; the main evidence is the call shape of `pid_task`/`__lock_task_sighand`). A single missing symbol can only be recorded as `L3 partial`, never used to declare affected.
4. Verdict: both classes point to fixed → `patched (L3)`; both point to the vulnerable shape → `affected (L3)`; mixed or only one class → `uncertain`, recording which class is missing.
5. Mind the base address and KASLR: static analysis uses file-relative addresses, never runtime addresses; verify the `vmlinux-to-elf` base manually.

**OEM image date criterion**: `build_date` is only a ranking signal (see the OEM criteria separation in image-intake.md) — a fix later than build_date ranks high, but never constitutes a conclusion; every OEM patch conclusion still requires L3.

## Promotion rules under frozen mode (target-mode: frozen)

After branch EOL / device stopped-update, the branch head is a **constant**; a single comparison of "device snapshot vs frozen branch head" is the **final conclusion**, with no time-varying uncertainty about "will next month's SPL fix it":

- **The affected direction may be inferred** (within GKI scope): the fix commit is in the frozen branch head and later than the device snapshot anchor → the device carries the hole, labeled `affected (frozen-inferred, GKI scope)`, final. Vendor modules and OEM-modified kernels still require code verification.
- The quick-exclusion path is likewise promoted to a final verdict (precondition 2 takes the frozen head as "branch head").
- In frozen mode, state's patch conclusions do not expire because of monthly merges, but **candidates newly discovered via cross-branch mining** still require individual code verification (fixes on new branches never appear in the frozen branch's log, so version inference is impossible).

## Full verification path (default)

**Prerequisite**: the Source Gate (gate 00) in stage-gate.md must already have passed before this stage starts. With only a CVE description and no fix diff, this stage cannot start; with only an approximate kernel version, you must not claim "the target kernel is confirmed affected".

## Online verification path

**Use the fixed URL templates in references/sources.md**. Try in order; stop at the first success:

1. **Target vendor kernel source tree** (online browsing)
   - **Device snapshot tag blob verification (the standard action for GKI devices; best value)**: directly compare the affected file's code shape at `https://android.googlesource.com/kernel/common/+/refs/tags/<branch>.<N>_r00/<path>?format=TEXT` — one fetch completes both topology membership (is the fix in the device) and code shape (is it the vulnerable form), free of commit-date interference.
   - GKI branch: `https://android.googlesource.com/kernel/common/+/refs/heads/<branch>/<path>` (append `?format=TEXT` for raw content)
   - Vendor Gitiles / GitHub mirror (locating method in sources.md)
   - Open the affected file's blob page on the target branch/tag and compare with the upstream post-fix code.
2. **Commit search**
   - GKI branch log: `https://android.googlesource.com/kernel/common/+log/refs/heads/<branch>/<path>`
   - Search the target repository for the fix commit subject (vendors often change commit messages but keep the code; the hash may change through cherry-pick).
   - Search for the core code line introduced by the fix (take the most distinctive +/- line from the fix diff and search it literally).
3. **Google-published factory images / GKI kernels**
   - For a GKI device, verify the correspondence between the GKI kernel version and the android-common branch.
   - GKI does not cover vendor modules (DLKM); vulnerabilities inside vendor modules must be confirmed separately.

## Vendor backport trap checklist

- **Incomplete backport**: only some hunks merged; the symptom is fixed but not the root cause.
- **Backport rewrite**: the vendor substitutes its own implementation; behavioral equivalence must be compared.
- **Fake fix**: the commit is merged but introduces a new equivalent defect (rare but real).
- **Fix later reverted**: reverted or overwritten by a subsequent vendor merge.
- **Mainline fix spans multiple commits**: one fix is followed by a fix-the-fix; checking only the first yields a wrong verdict.
- **Affected file absent from the vendor branch**: the subsystem may have been trimmed out, so R1 does not hold at all.

Every trap must be explicitly checked and its conclusion recorded during verification; never assume the trap is absent.

## Verification record format

```
### CVE-YYYY-NNNNN patch verification
- path: quick-exclusion / full-code-verification
- target repo: <URL>
- branch/tag: <e.g. android15-6.6-lts or vendor branch>
- inspected file(s): <path list; for the quick-exclusion path, the tag containing the fix + target branch head>
- fix commit found: yes(original hash) / no / modified-backport(hash) / partial / version-inferred(tag)
- compared against: <upstream fix commit hash>
- vendor backport uncertainty: <per-item conclusions from the trap checklist; for the quick-exclusion path, the verification result of preconditions 1-4>
- conclusion: affected / patched (code-verified) / patched (version-inferred, GKI scope) / not-applicable / uncertain
- evidence links + access date
```

## R-level determination

- `affected` (target code is still the vulnerable shape) → the candidate may rise to **R1**.
- `patched (version-inferred, GKI scope)` → exclude, record in report §6 (state the inference basis).
- `patched (code-verified)` with complete backport verification → exclude, record in report §6.
- `uncertain` → retain but cap the R level at R0; write the uncertainty into Blockers, labeled "requires further local image/source verification".

## Gates (end of this stage)

- [ ] Every candidate has an explicit `affected / patched (code-verified) / patched (version-inferred) / uncertain` conclusion
- [ ] Candidates on the quick-exclusion path have all four preconditions verified item by item, on file
- [ ] Every conclusion carries at least one verifiable code location (file path + function name) or commit hash / version boundary
- [ ] The backport trap checklist is checked item by item, on file (the quick-exclusion path checks the applicable preconditions)
- [ ] `uncertain` candidates have an explicit list of missing evidence and how to obtain it