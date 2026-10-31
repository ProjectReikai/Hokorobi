# Hokorobi (綻び)

Android kernel vulnerability discovery and **Root feasibility** research, packaged as a standalone Markdown skill.

Hokorobi helps a research agent move from "there may be a kernel bug" to a ranked, evidence-backed answer to two questions:

1. Does the vulnerability exist in the target kernel?
2. Does it have the conditions required for an Android local Root chain — unprivileged reachability, a confirmed root cause, a credible escalation argument?

The output is a research report, not an exploit. Hokorobi deliberately does **not** write escalation payloads, privilege-escalation code, arbitrary kernel read/write primitives, or mitigation bypasses. Experimental validation stays a separate activity.

## What it covers

- **Discovery** through three channels: GKI commit-stream mining (freshest), CVE/bulletin enumeration, and semantic targeting.
- **Patch verification** against the actual target: source code shape, OEM binary fingerprints, or git archaeology — never version-number guesswork.
- **Reachability** under `untrusted_app` (SELinux, seccomp, capabilities, component presence).
- **Root cause** and **race/lifetime** analysis with explicit evidence levels.
- **Root feasibility** assessment using two independent scales:
  - `L0`–`L4`: analysis evidence depth.
  - `R0`–`R6`: Root escalation evidence maturity.
- **Priority ranking** (`P0`–`P3`) based on Root research value, not CVSS.

## Evidence, feasibility, and priority

Hokorobi keeps three dimensions separate:

| Dimension | Scale | Meaning |
|---|---|---|
| Analysis evidence depth | `L0`–`L4` | How deep the code/binary evidence goes |
| Root escalation maturity | `R0`–`R6` | How far the escalation evidence goes |
| Research priority | `P0`–`P3` | Where to invest research effort next |

Evidence depth, escalation maturity, and research priority are independent dimensions. A high score in one does not imply a high score in another.

- `L4` does not mean Root is feasible.
- `R5` does not mean the target device can directly use a public exploit.
- `P0` is a research priority, not an exploitation success rate.

## Structure

```
.
├── SKILL.md                     # Entry point: rules, workflow, efficiency discipline, routing
├── references/
│   ├── discovery.md             # Candidate discovery (three channels, anchors, frozen mode, triage)
│   ├── sources.md               # Fixed URLs, Gitiles recipes, anti-bot fallbacks
│   ├── image-intake.md          # boot.img/Image parsing, 404 path split, IKCONFIG, debug variants
│   ├── patch-analysis.md        # Quick exclusion, OEM separation, L3 fingerprint recipe
│   ├── reachability.md          # Component check, call chains, socket-family recipe
│   ├── root-cause.md            # Root-cause evidence by vulnerability class
│   ├── race-analysis.md         # L0–L4, lifetime/lock tables, window honesty
│   ├── root-feasibility.md      # R0–R6, R5 tiers, public-evidence checklist
│   ├── scoring.md               # Six weighted dimensions, P0–P3 mapping
│   └── pitfalls.md              # Eleven hardened lessons
├── checklists/
│   └── stage-gate.md            # Source Gate and gates 01–08
├── templates/
│   └── report.md                # Report template (body language follows the user)
├── README.md
├── LICENSE
└── .gitignore
```

## Usage

Hokorobi is a Markdown-only skill with no bundled executable dependencies. Its analysis capabilities depend on the tools available to the host agent. Load `SKILL.md` as the entry point and follow the reference routing table for the current stage.

Typical prompts:

- "Assess this boot.img for Android local Root research candidates."
- "Which kernel vulnerabilities disclosed since January 2026 are worth investigating on this device?"
- "Verify whether CVE-XXXX-YYYYY is actually patched in this kernel."

When the user supplies a kernel image (`boot.img`, `vendor_boot.img`, `Image`, or `vmlinux`), Hokorobi treats the image as the authoritative metadata source and extracts what the host tooling allows, instead of asking the user about kernel internals. Analysis then degrades according to what the host agent can actually do:

| Available capability | Analysis path |
|---|---|
| Binary tooling + target image | Binary analysis: symbol recovery, disassembly, binary patch fingerprints |
| Target/GKI source only | Source analysis: code-shape comparison, commit archaeology |
| Advisory/CVE description only | Advisory-only analysis: recorded as incomplete, no structural claims |

Capability degradation must be stated explicitly. If binary tooling is unavailable, source-level findings are never presented as binary verification.

When the user supplies only a device/SPL, Hokorobi determines `target-mode: live | frozen` before choosing an anchor. With neither, it asks for the target first.

## Research Persistence

Hokorobi keeps research progress in the working directory rather than in hidden internal state.

Each run maintains:

- `hokorobi-runs/<run>/state/candidates.md` — candidate status, evidence, blockers, and current assessment.
- `hokorobi-runs/<run>/state/progress.md` — completed stages, pending work, and resume instructions.

An existing run is resumed instead of repeating completed verification work. Final reports are preserved alongside the research state.

Resuming does not freeze conclusions. Evidence that may have expired — a newer upstream fix, a changed device build, or newly published reproduction — can and should be re-verified. "Do not repeat completed work" never means "never update a conclusion".

## Principles

- Search, don't recall: every candidate comes from a page fetched in this run.
- Patch status, reachability, and race exploitability are three separate dimensions; none substitutes for another.
- Debug builds yield `L4-debug` observations only; they never become production `R5`/`R6`.
- OEM images: the version-string `g<hash>` is `non-authoritative`; patch conclusions require binary fingerprints, OEM source, or real-device validation.
- Public device-specific rooting projects are not generic Android Root.
- Blockers are mandatory: every report states what was not achieved and which evidence is missing.

## Language policy

- Skill instructions are written in English.
- Technical identifiers, symbols, CVE IDs, commit hashes, and paths are preserved unchanged.
- Source code and patches are analyzed in their original language.
- Final research reports are written in the language requested by the user.

## License

Apache License 2.0. See [LICENSE](LICENSE).

Copyright 2026 ProjectReikai Organization.