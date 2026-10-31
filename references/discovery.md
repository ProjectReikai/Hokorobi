# Discovery — Candidate Vulnerability Discovery

Goal: build a candidate CVE list matching the target kernel and time range. This stage only collects; it does not judge value. Judgment happens in later stages.

## Input requirements

Before starting, obtain the following from the user, the image, or state (**image input takes priority** — when the user provides boot.img/Image, parse per references/image-intake.md; metadata comes from the image, never require the user to describe kernel internals):

- Target device model / vendor / SoC (or the device the image came from)
- Kernel version and branch: for image input, derive from the version string (stable base + GKI generation → `<branch>`); for dictated input, e.g. android15-6.6
- Device current SPL / build date: time anchor for the commit channel with dictated input; **not needed for image input** (the anchor comes from the version string/tag)
- target-mode: live / frozen (frozen criteria in "Frozen mode")
- Build fingerprint (if known)
- Research time range (disclosure-date window for the CVE/semantic channels; the commit channel does not use this window, it uses the delta window)
- Architecture (arm64 / x86_64; for image input, confirm from the Image magic)
- Research objective (e.g. "find directions that can develop into a generic LPE" or "reproduce a known privilege escalation")

If all three are missing (no image, no device, no SPL — a "find bugs in 6.x" style prompt), ask the user for the target device or image first; never choose an anchor unilaterally.

## Data sources

**Always use the fixed entry points and URL templates in references/sources.md**; never assemble URLs yourself. Summary:

| Source | Use |
|---|---|
| Android Security Bulletin / Pixel Bulletin | Vendor SPL fix lists, ASB-CVEs |
| NVD / MITRE CVE records | CVE descriptions, CVSS, reference links |
| kernel.org CVE announcements (lore.kernel.org/linux-cve-announce) and the Stable tree | Upstream CVE attribution and fix commits |
| syzkaller dashboard (syzkaller.appspot.com/upstream) | Fix commits, crash types, reproducers |
| oss-security / lwn / Project Zero blog | Technical discussion, root-cause analysis |
| GitHub / Exploit-DB | Public PoCs, privilege-escalation research (lead-level only; must pass the verification checklist) |

## Search strategy (three segments: commit-stream mining → CVE channel → semantic targeting)

Discovery is **retrieval, not recall**. Three hard constraints:

1. **Never introduce candidates from model memory.** Every candidate must come from a **first-party page actually fetched in this run**, with that page's link recorded.
2. **The disclosure date must be verified to fall inside the window.** Before a candidate enters the list, verify its disclosure/announcement date from a first-party source (the commit channel uses the merge date); discard candidates outside the window (unless the user explicitly asks for historical CVEs).
3. Segment order: commit stream first (freshest), CVE channel fills in consequence labels (lagging), semantic targeting adds root relevance; deduplicate across channels by fix commit hash.

### Segment 1: GKI branch commit-stream mining (primary channel)

**CVE records lag behind git history**: upstream fixes land in mainline/stable first, CVE registration often follows months later, and ASB is a filtered subset. The freshest n-days hide in the target branch's log.

**Scan-direction discipline (mandatory)**: always scan from the branch head **backwards** (Gitiles log is naturally newest-first):

- Commits after the snapshot anchor are very likely not in the device — this is the bulk of the delta, harvest it first;
- But under merge topology, date ≠ membership (see the "topology ≠ date" warning below): every candidate in the range must pass tag blob/log topology verification before it counts as "confirmed absent from the device";
- Commits before the snapshot anchor are uncertain (they may have reached the device via some release) — stop when you reach the anchor, do not dig deeper; candidates before the anchor are left to the CVE channel and Patch Verification.

- For fetching methods see the "Gitiles log recipes" in sources.md: `+log` via shell + curl + `?format=JSON` (webfetch returns 400); paginate with the `s=` cursor; use range syntax `+log/<from>..<to>` for an exact delta.

**Anchor location (common GKI branch structure)**: the target GKI branch (`android<OS version>-<kernel version>`, e.g. android14-6.1 / android15-6.6 / android16-8.x) has a **dual-line structure** — the mainline branch `<branch>` and `<branch>-lts`, merging into each other. The device snapshot's real anchor is not a stable merge but a **release tag**:

- The mainline branch has a monthly release merge of the form `Merge tag '<branch>.<N>_r00'` (e.g. `android15-6.6.143_r00` on 2026-09-25, 144_r00 on 2026-10-24);
- **Device snapshot anchor = the last `_r00` tag merge before the SPL date** (tag date ≈ ten days before SPL); e.g. SPL 2026-10-05 → the device is most likely `143_r00`;
- Conservative scan anchor = one merge earlier than the device anchor (a stable merge or an earlier `_r00`) — better to look too far than to miss;
- Anchor scanning requires **complete pagination** (print every subject per page) **only when a local clone is unavailable**; once the local shallow clone succeeds (sources.md recipe), search locally with `git log` and skip pagination. When pagination is needed there is no fixed page cap — when the device baseline lags by months the delta can span dozens of pages / thousands of commits (measured: device built 2026-07, branch head 2026-10 → 59 pages / 5876 commits). Do not rely on truncated output (merge commits may not be in the first screen).

**Topology ≠ date (critical warning)**: "commits after the anchor = definitely not in the device" **does not hold strictly**:

- Content inside the range with a committer date earlier than the anchor may enter the branch after the anchor via later merge topology (fresh n-day);
- A direct commit with a newer date inside the range may already have reached the device with a `_r00` tag (not part of the delta).
- **Fallback rule**: for every candidate that enters deep analysis, perform **topology membership verification against the device snapshot tag's blob/log** — check whether the fix commit is already inside the tag (`+log/refs/tags/<branch>.<N>_r00/<path>`), or directly compare the affected file's code shape in the tag (blob + `?format=TEXT`). Membership is decided by topology, never by date. This is also the standard action in Patch Verification (see patch-analysis.md).
- The SPL ↔ release-tag mapping is itself an inference: unless the user supplied a build fingerprint, record "device snapshot = some `_rNN` tag" as an assumption in Blockers. Tag naming may change across branch generations (e.g. `android15-6.6.N_r00` vs other forms); always follow the actual `Merge tag '...'` lines in the target branch log, never assume naming from memory.
- Tighten keyword filtering to the subject line: `use-after-free` / `race` / `refcount` / `double free` / `out-of-bounds`.
- **Unknown consequence is the inherent cost of this channel**; handle it per the rules below, never guess severity:
  - Record `consequence: TBD`;
  - Use **fix-diff shape** for cheap coarse ranking (weak evidence, affects ordering only, never the R level):
    - `smell: strong` — the change touches refcount/lock/RCU ownership of a long-lived shared object, or spans cross-subsystem objects;
    - `smell: weak` — a one-off bounds check, or a fix only on an error path;
    - `smell: unknown` — the diff shape does not permit a judgment;
  - Formal consequence determination is deferred to diff analysis in the Root Cause stage (that is ground truth);
  - **CVE backfill**: the kernel CNA registers upstream fixes broadly and with a lag; candidates usually match up later in lore/linux-cve-announce and NVD. A successful backfill upgrades the candidate to the CVE channel with consequence labels; if it does not match, keep it as a commit-only candidate (never drop it merely for lacking a CVE ID).

### Frozen mode (EOL / stopped-update devices)

Decide `target-mode` before starting: branch EOL, device no longer receiving updates (support ended / OEM stopped), or the user explicitly declares a fixed version — any one → `frozen`, otherwise `live`. Write it to state (`target-mode: live | frozen`). A GKI device's kernel version follows the OS major version; freeze determination uses the OS/kernel the device **actually stays on**, not the branch calendar.

Frozen mode changes the nature of the whole study — **the delta never converges again, and confirmed vulnerabilities never expire**:

1. **D1 one-shot bulk harvest**: fetch the whole delta from the device snapshot to the frozen branch head in one go (range syntax); this is the definite, final n-day set.
2. **Primary channel = cross-branch delta**: after freezing, the stable→GKI flow stops; the continuously growing delta comes from mainline/new branches (e.g. newer `android<NN>-<X.Y>` branches, newer LTS) fixing code that the target branch (its LTS baseline) still uses but that is **never backported**. Mining direction is the reverse of D1: instead of scanning the target branch, take **fix commits from newer branches and check back** whether the target branch still contains the same affected code shape (new branch log → locate affected file and function → check in the frozen branch whether that code exists and is unfixed). A hit is a permanent n-day.
3. **Version inference is promoted to a final verdict** (see the frozen-mode section in patch-analysis.md): within GKI scope, the affected direction may also be inferred, labeled `frozen-inferred`.
4. **R6 dedup first**: EOL devices are a dense area of public exploits — first answer "has anyone already pwned this device". The target niche is "EOL + no public success yet" or "public exploitation exists on a different SoC/config". The root-feasibility verification checklist runs first in frozen mode.
5. If a live-mode report finds a candidate that also applies to a frozen device, note cross-device timeliness in Research Recommendations (the same direction has higher research value on frozen devices).

### Segment 2: CVE channel (with consequence labels, prevents omissions)

Walk back from the newest until the window start is covered:

1. **Pixel Security Bulletin / ASB, newest issue first**: bulletin pages enumerate CVEs by month. Device-level SPL (Pixel bulletin) takes priority over generic ASB. **The ASB body does not necessarily have a Kernel section every month** (measured in 2026: only the 2026-03/06/09 issues had one) — for months without a Kernel section, record "no Kernel entries in this issue" honestly; do not fill in from other months, and do not stop enumerating because of it.
2. **lore.kernel.org/linux-cve-announce/ archive**: upstream CVE announcements ordered by date; page back to the window start (if anti-bot blocks, browse the openwall/spinics mirrors by month).
3. **NVD / MITRE**: fetch details per CVE-ID.
4. **syzkaller dashboard**: recently fixed bugs can add upstream fixes not yet enumerated.

**Enumeration is not optional**: budget pressure may only cut the number of deep dives, never degrade the CVE channel into "check this month's bulletin + backfill commit candidates" — that way CVEs disclosed inside the window (several months) would never enter the candidate pool. Minimum requirement: page every bulletin issue + the kernel-cve-announce archive back to the window start, forming a complete list of kernel CVEs in the window (the list itself may contain only ID + subject; Triage decides what to deep-dive).

When a primary source is blocked by anti-bot (kernel.org / lore / NVD all possible), switch to a content-equivalent mirror per sources.md "Anti-bot interception and fallbacks" (GitHub mirrors / cveawg API / cvelistV5); **do not lower the evidence tier and do not interrupt the flow**.

### Segment 3: Semantic targeting (root relevance)

**Search breadth is generalized Linux root escalation**: the candidate pool covers kernel LPE across the whole Linux ecosystem (upstream, distributions, public exploit research), not only material containing "android". But **conclusions are never generalized**: every candidate found must re-run component-presence and reachability verification against the Android target.

Use queries like these (with the fixed entry points in sources.md, plus year/month qualifiers):

- `linux kernel privilege escalation CVE 2026` / `linux kernel LPE 2026` (general primary query)
- **`linux kernel exploit <year>`, `"we were able to exploit" kernel`, `kernel exploit writeup <year>`, `kernelCTF exploit`** — public exploit claims/named campaigns are the highest-value signal (research orgs/teams announcing successful exploitation, publishing writeups); search these proactively, do not wait for CVE descriptions to surface them
- `android kernel privilege escalation CVE 2026`, `android kernel EoP bulletin <latest month>` (Android-specific)
- `kernel UAF exploit <year>`, `linux kernel root exploit <recent>`
- Follow deep analysis of recent kernel vulnerabilities in Project Zero blog, security-vendor research blogs, distribution advisories (Red Hat/SUSE/Ubuntu kernel EoP advisories are naturally pre-filtered by escalation relevance)
- Prioritize Android bulletin entries marked EoP / high severity
- Public Linux escalation projects (e.g. distribution n-day exploit collections) as lead sources

Hits must also pass first-party source verification and the time-window check; second-hand articles are leads only. **Linux-context reachability/exploitability conclusions (e.g. "needs userns", "needs eBPF") are hypotheses to be verified, not facts** — verify item by item against Android in the Reachability stage.

`public-lpe-lead` candidates **must not be silently dropped even if disclosed before the target SPL**: first run quick exclusion/code verification to reach a patched/affected conclusion, and record it with reasons in report §6 (public exploit claims remain R4/R5 material for other devices with earlier snapshots or frozen devices, and are part of the research map).

Never open with a broad websearch for "android kernel CVE" before the channels above — it hits piles of second-hand aggregators and well-known historical CVEs, slow and biased.

## Cross-channel merge and deep-dive selection (Triage, mandatory after the three segments)

Merge hits from the three segments into a **unified candidate pool** (deduplicate by fix commit hash). Semantic-channel hits are not merely "leads" — **they are candidates** and must enter candidates.md for ranking; never leave them as side notes in progress.

**Run the Android precondition pre-filter first (affects ordering only, does not replace gate 03)**: run a cheap precondition check over the whole pool — `CONFIG_USER_NS` dependency, `CAP_NET_ADMIN` dependency, SUID helper dependency, AF_* socket reachability (rough sepolicy check). Clearly unmet candidates are down-ranked but retained; the deep-dive budget goes first to those that pass. The purpose is to stop desktop-oriented candidates and public-lpe-leads from eating the entire budget.

When deep-dive slots are limited, choose in this order (**root relevance first, not smell**):

1. **`public-lpe-lead`**: candidates with public LPE research/PoC/named campaign (kernelCTF, exploit repo, security-vendor deep analysis) — the strongest signal: Hokorobi's final goal is the root direction, and public evidence maps directly to R4/R5 potential, the highest research value.
2. **`bulletin-eop`**: ASB/Pixel bulletin entries marked EoP/Critical (compare with the device SPL first; already-fixed ones go through quick exclusion but are recorded).
3. **`smell: strong`** commit-only candidates (high primitive potential but no public evidence at all).
4. The rest (weak smell / conditional).

`smell` measures primitive potential, `public-lpe-lead` measures research value — the two are not interchangeable. Abandoned high-priority candidates go into a deferred list with reasons. public-lpe-lead candidates also go through Patch Verification (they may already be fixed — exclusion is a valid outcome, keep the reason on file).

## Time filtering strategy (ranking heuristic, not a conclusion)

**Prefer candidates disclosed after the target build time**:

- Using the target's current SPL / build date as the reference point, CVEs disclosed later get the highest priority (the fix likely has not reached the device).
- Candidates disclosed **before** the target SPL are **very likely already fixed** — as a ranking heuristic put them at the end of the queue, but **never declare them fixed on that basis**: vendor branches missing fixes, partial backports, and DLKMs outside GKI update coverage do happen; the final state can only come from code verification in Patch Verification (hard rule 1).
- Example: target android15-6.6, SPL 2026-10: 2026-disclosed candidates first; 2025 candidates at the end but still verifiable.

## Screening strategy

1. **Rough screen by subsystem**: prioritize subsystems relevant to the target kernel config (binder, io_uring, netfilter/nftables, AF_PACKET, USB/camera drivers, Ion/DMA-BUF, LSM, etc.).
2. **Preference by type**: mark memory-corruption classes (UAF/OOB/double free) and permission/logic classes separately; later assessment paths differ.
3. **Exclude the clearly irrelevant**: architecture mismatch (x86-only), requires physical access, requires root to trigger — exclude and record.
4. Do not rank by CVSS; CVSS correlates weakly with Root research value.

## Search keyword templates

- `"CVE-<subsystem>" site:android.googlesource.com`
- `<subsystem> use-after-free privilege escalation android`
- `<CVE-id> exploit OR PoC OR LPE OR root`
- `android security bulletin <YYYY-MM> kernel`

## Handling online fetch failures

Content webfetch cannot access (deleted repos, login-required trackers, Google-internal bug links):

- **Record honestly as "evidence unavailable"** and write it into the candidate's Blockers.
- Never pass off caches, mirrors, or second-hand retellings as first-party evidence; mark retold sources separately and lower their credibility weight.
- Second-hand sources (security media) may only be used to discover leads, never as a basis for R-level promotion.

## Stage output

Update `state/candidates.md`, one record per candidate:

```
### CVE-YYYY-NNNNN or <commit-hash-short> (commit-channel candidates may lack a CVE ID yet)
- channel: commit / cve / semantic (list all channels that hit)
- type: UAF / OOB / race / logic-permission / ...
- subsystem: <e.g. binder / nftables / ...>
- disclosure: <date; commit channel uses merge date>
- upstream fix commit: <hash>
- consequence: <advisory description / TBD (commit channel)>
- smell: strong / weak / unknown (commit-channel diff coarse ranking only; weak evidence never enters R levels)
- sources: <verified link list, with access dates>
- status: discovered
- gates: none (append each gate conclusion below this record in stage-gate.md format)
- notes: <initial impression, no conclusions>
```

## Gates

- Every candidate has at least two independent sources (CVE ID or fix commit + one verifiable technical source); otherwise mark `needs-verification` and do not advance.
  - **Commit-only exception**: a fix commit page on a GKI branch is itself a first-party source; a single source may enter gate 02, labeled `channel: commit-only`; consequence verification is deferred to Root Cause (consequence: TBD).
- Every candidate's disclosure date has been verified from a first-party page and falls inside the research window; candidates coming from model memory rather than a page fetched in this run are removed.
- Commit-channel candidates are labeled `consequence: TBD` and `smell`; severity was not invented.
- When the candidate count is too large (>20), coarse-rank by Triage priority (public-lpe-lead > bulletin-eop > smell: strong > the rest), keep top N with the trade-off explained, and put the rest in a "deferred" list.