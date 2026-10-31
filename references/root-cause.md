# Root Cause Analysis — Root Cause

Goal: confirm the vulnerability's root cause with trustworthy evidence. This is the basis for R3. **R3 requires evidence, not reproduction** — the fix commit's diff is itself first-party evidence.

## Evidence priority

1. **Upstream fix commit + fix description** (first-party, most trustworthy).
2. **Maintainer/researcher mailing-list discussion** (lore.kernel.org, oss-security).
3. **syzkaller dashboard crash reports and reproducers** (prove triggerability; note their runtime is a VM + KASAN, see pitfalls).
4. **NVD/vendor descriptions** (retold; leads only, cannot alone support R3).

## Minimum evidence set by vulnerability type

### UAF / refcounting
- Free-site and use-site functions and code paths (file + line-level location)
- Lifetime management mechanism: refcount / RCU / lock, and exactly where the defect lies (missing put, premature free, insufficient lock coverage, error-path leak)
- How the fix commit changes lifetime ownership

### OOB / out-of-bounds
- The memory region accessed, the expected boundary vs the actual boundary
- The exact location of the missing/incorrect bounds check (off-by-one, type truncation, integer overflow)
- Access type: read/write, and whether the out-of-bounds distance is controllable

### Double Free
- Paths of the two frees and their triggering preconditions (error path, concurrency, misjudged reference)

### Race
- The two interleaved execution paths and which locks each holds
- The window nature revealed by the fix approach: adding a lock, atomics, converting to RCU, serialization — the fix commit's choice is the strongest clue to the window's nature

### Logic / permission defect
- The authorization check or invariant that was broken (which check is bypassed under what conditions)
- The difference set between the caller's assumed identity and the actually reachable identity
- **Do not force-fit the memory-safety analysis framework**: this type has no "object lifetime"; the focus is the permission boundary and caller identity, and race windows usually do not apply

## Analysis record format

```
### CVE-YYYY-NNNNN root cause
- defective behavior: <one sentence describing the broken invariant>
- affected objects/functions: <file + function>
- root cause class: <refcount-miss / premature-free / lock-scope / bounds-check / auth-bypass / ...>
- fix commit analysis: <what the fix does, hence what the root cause is>
- security invariant broken: <e.g. "object replaceable after credential check" / "length truncation causes heap overflow">
- evidence: <first-party evidence link + access date; retold sources marked separately>
- R3: granted / not-granted (reason)
```

## R3 determination

- granted: first-party evidence (fix commit/maintainer discussion) explicitly supports the root-cause description above, and that evidence matches the target kernel's code shape.
- not-granted: only retellings, only crash symptoms without a root-cause explanation, or the root-cause analysis does not match the target kernel code. Then the R level is capped at R2 and the missing evidence goes into Blockers.

## Gates (end of this stage)

- [ ] Every candidate has an explicit root-cause class and a statement of the broken invariant
- [ ] At least one first-party evidence item (commit/mailing list) supports it; retold sources do not count
- [ ] Logic/permission candidates are not force-fitted into the memory-safety framework
- [ ] not-granted candidates are capped at R2 with the missing evidence recorded