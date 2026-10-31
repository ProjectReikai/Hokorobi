# Race & Lifetime Analysis — Race and Object-Lifetime Analysis

Goal: analyze race windows and object lifetimes based on **code evidence**. This stage does not change the R level (root-feasibility decides R levels), but it directly affects the Difficulty score and the Race Assessment section of the report.

**Core discipline**: there must be a code basis; never infer from the CVE description alone. But a complete target-kernel source tree is not required, and the kernel certainly does not need to be built first.

## Evidence levels (L0–L4)

The L level measures **analysis evidence depth** and is a different scale from the R level (Root escalation evidence maturity); **the two must never be mixed**:

| Level | Input material | What it enables | Limitation |
|---|---|---|---|
| L0 | Advisory / CVE description only | Record the candidate, determine what material is needed next | No structural analysis at all |
| L1 | Upstream source + fix diff | Analyze race root cause, locks, RCU, refcounting, object lifetime | Cannot determine whether the device uses the same implementation |
| L2 | Kernel source corresponding to the target device (GKI branch or vendor) | Analyze target code, vendor modifications, patch status | The actual optimized instructions may differ from the source |
| L3 | Target device `Image` / `vmlinux` | Verify machine code, call paths, memory barriers, and patch fingerprints | Requires symbol recovery, disassembly, or decompilation tools |
| L4 | Diagnostic runtime evidence (KASAN, syzkaller reports, public measurements) | Cite window observations and trigger data | The observation environment is not equivalent to the original device (see pitfalls §3); observations on a debug build are **L4-debug** and must not be extrapolated |

L1 is the default working mode; L2/L3 are enhanced modes, upgraded according to material availability. L levels may not skip: without L2 material you must not write "this path exists in the target kernel".

## Kernel source acquisition priority

Try in order and record the level actually reached:

| Source | Use | Entry point |
|---|---|---|
| A (default) | Upstream fix commit and source | kernel.org, GitHub mirror, lore.kernel.org (fixed URL templates in references/sources.md) |
| B (preferred) | The corresponding Android GKI branch | android.googlesource.com/kernel/common (note: a GKI branch such as android15-6.6 cannot be replaced by upstream Linux 6.6; file/commit URL templates in sources.md) |
| C (optional) | Vendor public source | Vendor Gitiles / GitHub mirror (locating method in sources.md) |
| D (enhanced) | User-supplied kernel binary | `Image` / `vmlinux`, with disassembly tools |

Reference: the [Linux RCU Handbook](https://docs.kernel.org/RCU/index.html) for interpreting RCU semantics.

## Analysis steps (execute in order; record the output of each step)

1. **Obtain the vulnerable and fixed versions of the affected function** (only after the Source Gate passes).
2. **Read the diff** and identify the changed synchronization protocol: the patch adds `refcount_inc_not_zero()`, moves `fput()`, adds lock protection, switches to RCU — the fix choice is the strongest clue to the window's nature. When you see such changes, trace the object's ownership rules, **do not immediately judge exploitability**.
3. **Read the necessary context**: callers, callees, object struct definitions.
4. **Build two tables**:
   - Object-lifetime table: allocation points, reference holders, free points, access paths remaining after free;
   - Lock-relationship table: what each lock / RCU critical section / refcount / memory barrier protects.
5. **Check for omissions**: any access without reference protection, any free-ordering problem.
   Distinguish a "candidate lifetime conflict" from a "confirmed UAF":

   ```
   Thread A                      Thread B
   --------
   Lookup(object)                Release(object)
                                 Free(object)
   Access(object)
   ```

   The diagram above is only a candidate conflict. Confirming a UAF additionally requires: A's access is unprotected by a lock or valid reference, **and** B's free can occur in that interleaving.
6. **Map to the target**: project the conclusions onto the GKI branch or vendor source (when L2 material exists); otherwise explicitly label "based on upstream code; target applicability unconfirmed".
7. **Record unresolved assumptions**.

## Optional external tools

Hokorobi does not implement its own parsers/disassemblers; guide the agent to use existing tools, chosen by availability, and **never require installation**:

| Tool | Use |
|---|---|
| Gitiles / GitHub | Online source, commit history, diffs (default path) |
| `git diff` / `git grep` | Code analysis after a local clone |
| `clang` / LLVM | AST, static source inspection |
| `sparse` | Linux kernel annotations and static checks |
| Coccinelle | Semantic patch matching |
| Ghidra / REA | Binary disassembly and decompilation (L3) |
| **BTF / `pahole` / `bpftool`** | Read struct offsets from the target image (a high-value L3 entry when `CONFIG_DEBUG_INFO_BTF=y`; offsets feed straight into disassembly) |
| **`vmlinux-to-elf`** | Recover the kallsyms symbol table from a raw Image/vmlinux → objdump disassembly (verify the base manually) |

With only a binary and no reverse-engineering tools, explicitly state "binary-level verification cannot be completed"; the L level stops at L2.

## Hard rules

- Never infer target patch state from upstream alone.
- Never claim a numeric race window without evidence (no numbers without evidence; numbers can only come from L4).
- Never equate a source-level race with a confirmed UAF.
- Never claim binary verification without binary evidence.
- If no source is accessible, mark the analysis incomplete (L0).

## Android-specific factors (counted into Difficulty)

- preempt/voluntary-preemption config, and the EAS scheduler's effect on the window;
- differences in vendor kernel lock implementations;
- the app's CPU quota and background-execution limits (cgroup freezer, foreground constraints) are **real constraints** on looping to hit the window.

### Window/difficulty mapping (reference for scoring's Difficulty)

| Evidence and window state | Difficulty meaning |
|---|---|
| Source confirms single-thread triggerability (no concurrent interleaving) | Low |
| L1/L2 confirms a window of tens of instructions, requires 2-thread coordination, object preallocatable | Medium |
| Extremely narrow window + requires precise heap layout + app constrained by CPU quota | High |
| Only theoretical interleaving possible, no hit evidence at all (≤L1 and no L4 observation) | Extremely high, and credibility questionable |

## Required output (write to state/candidates.md and the report's Race Assessment section)

```
### CVE-YYYY-NNNNN race assessment
- evidence level: L<0-4> (+ actual acquisition path A/B/C/D)
- source revision / commit: <exact commit the analysis is based on>
- affected functions: <file + function list>
- object lifetime summary: <alloc/hold/free/residual access>
- synchronization summary: <what each lock/RCU/barrier protects>
- patch difference: <what synchronization protocol the fix changed>
- target applicability: <conclusion and uncertainty when mapped to the target kernel>
- race required: yes / no
- window evidence: <for L4, cite source + numbers; otherwise mark theoretical>
- reproduction observations: <state the runtime environment>
- app constraints: <CPU quota, background limits>
- confidence: high/medium/low
- missing evidence: <what is needed to raise the L level>
```