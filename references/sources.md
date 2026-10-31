# Sources — Fixed Data Sources and URL Templates

This file is Hokorobi's only URL list. During search and verification, **prefer the fixed entry points and URL templates below**; never assemble or guess URLs yourself. Only when a template does not apply, use websearch to locate a page, and verify that the hit belongs to the corresponding domain in this table.

## Android kernel source

| Source | Entry point |
|---|---|
| Android Common Kernel (GKI branches, top-level) | https://android.googlesource.com/kernel/common/ |
| GKI branch file browsing | `https://android.googlesource.com/kernel/common/+/refs/heads/<branch>/<path>` (e.g. `refs/heads/android15-6.6/mm/util.c`) |
| GKI branch commit / log | `https://android.googlesource.com/kernel/common/+log/refs/heads/<branch>/<path>` |
| Fetch raw file content | Append `?format=TEXT` to the file URL above (returns base64; decode it) |
| Branch/tag list | `https://android.googlesource.com/kernel/common/+refs` |
| GKI defconfig | `https://android.googlesource.com/kernel/common/+/refs/heads/<branch>/arch/arm64/configs/gki_defconfig?format=TEXT` (component-presence verification) |
| AOSP sepolicy | https://android.googlesource.com/platform/system/sepolicy/ (prefer a shallow clone and grep allow rules; far faster than fetching files one by one) |
| App-process seccomp policy | ⚠️ The old entry `platform/system/seccomp` now 404s. The app's syscall filter actually lives on the bionic side: `https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/seccomp/` (policy files are in this directory); if the repo moves, relocate with `websearch site:android.googlesource.com seccomp policy bionic`, and record a Blocker if relocation fails |

Branch naming: `android14-6.1`, `android15-6.6`, etc.; a GKI branch is **not equivalent** to the upstream Linux kernel with the same version number, and the two cannot substitute for each other.

### Gitiles log recipes (required reading for commit-stream mining)

**Preferred: local shallow clone (an order of magnitude faster than paging HTTP)**:

```bash
# --shallow-since must be at least 30 days earlier than the anchor, or the anchor merge is cut off (measured pitfall)
git clone --filter=blob:none --shallow-since=<anchor date minus 30 days> \
    --no-checkout --single-branch --branch <branch> \
    https://android.googlesource.com/kernel/common ktest
# Verify the anchor is inside the clone range (must return 0 before continuing):
git merge-base --is-ancestor <anchor_sha> HEAD   # non-zero → git fetch --deepen=500 and retry
# Everything afterwards is instantaneous locally:
git log --oneline --grep='use-after-free\|UAF\|double free\|refcount\|race' -i
git log --name-status -1 <commit>      # screen files (tree objects, instantaneous)
git show -p <commit>                    # fetch diff (first use lazily fetches blobs ~30s, then cached)
```

Measured: clone ~9 s; `git log --grep` 0.03 s; replaces 59 pages of paged requests. Fetch diffs only for Triage top-K.

**Fallback: Gitiles JSON paging/ranges** (when a local clone is impossible):

webfetch returns 400 for `+log`, and `?format=TEXT` only works for blobs. Fetching logs **must use shell + curl**:

- Basic form: `curl -s 'https://android.googlesource.com/kernel/common/+log/refs/heads/<branch>?format=JSON&n=1000'` (**measured n cap is 1000**, about 8 s per page; do not page with n=100)
- **The JSON is preceded by one XSS-protection line (`)]}'`); the most robust way to strip it is `tail -c +5`**.
- Pagination: the `next` field in the response gives the next cursor; add `&s=<cursor>` to the request.
- **Range syntax (preferred, avoids deep paging)**: `+log/<from>..<to>`, e.g. `+log/<snapshot_tag_merge_sha>..refs/heads/<branch>?format=JSON&n=1000`. Note that Gitiles returns ranges in **commit topology order**, not strictly truncated by date — just page until the range ends.
- Anchor location: the target GKI branch (`android<OS version>-<kernel version>`) is dual-line: mainline `<branch>` and `<branch>-lts` merge into each other. The mainline has a monthly release merge `Merge tag '<branch>.<N>_r00'`; the device snapshot anchor is the last `_r00` tag merge before SPL (details in the discovery.md anchor section; tag naming follows the actual target branch log, never assume from memory). Anchor scanning requires complete pagination with no page cap (dozens of pages when the OEM baseline lags by months).
- **Topology membership verification (standard candidate action)**: decide whether a commit is already in the device snapshot by release tag, not by date —
  - log method: `+log/refs/tags/<branch>.<N>_r00/<path>?format=JSON` (fix commit in the tag's log → already in device);
  - blob method (does membership and code-shape verification in one shot; best value): `https://android.googlesource.com/kernel/common/+/refs/tags/<branch>.<N>_r00/<path>?format=TEXT` (base64-decode and compare with the upstream post-fix code).
- Each log entry yields `commit`, `committer.time`, and the first line of `message` (subject) for keyword filtering; keywords match the subject line only, never the full text.
- Upstream stable (git.kernel.org) also supports range logs: `https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/log/?h=linux-6.6.y` (HTML page, or `?qt=range&q=<from>..<to>`).

### Data sources under frozen mode (EOL / stopped updates)

- Cross-branch mining: fetch logs of newer GKI branches (e.g. `android16-*`) with the recipes above; after locating affected files, return to the frozen branch to check code shape (blob URL + `?format=TEXT`).
- Material for the device-freeze determination: kernel.org per-version EOL pages (https://endoflife.date/linux or https://kernel.org/category/releases.html) and the device support-policy page.

## Upstream kernel

| Source | Entry point |
|---|---|
| Linux Stable | https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/ |
| mainline | https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/ |
| Commit lookup | `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<hash>` (same for the stable tree) |
| CVE announce mailing list | https://lore.kernel.org/linux-cve-announce/ |
| RCU semantics reference | https://docs.kernel.org/RCU/index.html |

**Note**: kernel.org (including git/docs) deploys Anubis anti-bot, so webfetch-style retrieval is stopped by a proof-of-work challenge; lore and NVD may also be unreachable in this environment. When blocked, switch mirrors per "Anti-bot interception and fallbacks" below, **without lowering the evidence tier**.

## Anti-bot interception and fallbacks (Anubis / 403 / timeout)

### Principles

1. **A content-equivalent mirror = first-party evidence**. Git is content-addressed: the same commit hash on a mirror is the same object as on kernel.org, and its evidentiary value is identical. This differs fundamentally from a "second-hand retelling" (pitfalls §8) — retellings on security media/aggregators are still not first-party evidence.
2. When the primary source is unreachable, switch to the mirror in the table below; record both "primary source unreachable (symptom + date) + the mirror URL actually used".
3. When an Anubis interception page is recognized ("Making sure you're not a bot" / proof-of-work challenge), retry the same URL at most once, then switch mirrors immediately.
4. If all mirrors fail for a class of material → record it honestly in Blockers; never downgrade and fabricate.

### Fallback table

| Blocked primary source | Fallback | Notes |
|---|---|---|
| git.kernel.org (mainline) | `https://github.com/torvalds/linux` (official mirror) | Commit pages are directly accessible; raw diff via the `.patch` suffix: `https://github.com/torvalds/linux/commit/<sha>.patch` |
| git.kernel.org (stable content) | ① GKI branches (the D1 delta already contains merged stable fixes; android.googlesource is reachable) ② Chrome OS kernel mirror `https://chromium.googlesource.com/chromiumos/third_party/kernel` (stable base branches; verify the exact branch name before use) ③ native shell git protocol `git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git` (port 9418, bypasses HTTP anti-bot; verify availability on site) | For the D2 channel (stable fixes after the GKI head), prefer ③, fall back to ② |
| lore.kernel.org | The content equivalent of linux-cve-announce is already in cvelistV5 (see below); for general technical discussion use `https://www.spinics.net/lists/` and the oss-security archive `https://www.openwall.com/lists/oss-security/` | Corroborate mail evidence with the message-id; if it cannot be corroborated, lower the evidence confidence and note it in the record |
| NVD (nvd.nist.gov) | ① CVE.org official API `https://cveawg.mitre.org/api/cve/<CVE-ID>` ② authoritative CVE repository `https://github.com/CVEProject/cvelistV5` (raw JSON: `https://raw.githubusercontent.com/CVEProject/cvelistV5/main/cves/<YYYY>/<NNxx>/<CVE-ID>.json`, where `NNxx` is the thousand-block containing the first two digits of the number, e.g. 43xx) | Kernel CNA CVE records are themselves published in cvelistV5, so this is a first-party source, not merely an NVD substitute |
| kernel.org CVE tree (vulns.git) | Same as the NVD fallback (kernel CNA data ultimately lands in cvelistV5) | |
| docs.kernel.org | `Documentation/` inside the mirror repositories (e.g. `https://github.com/torvalds/linux/tree/master/Documentation/RCU`) | |

## Vulnerabilities and advisories

| Source | Entry point | Use |
|---|---|---|
| Android Security Bulletin | https://source.android.com/docs/security/bulletin | Generic SPL, ASB-CVEs; monthly body template: `https://source.android.com/docs/security/bulletin/<YYYY>/<YYYY-MM-01>` |
| Pixel Security Bulletin | https://source.android.com/docs/security/bulletin/pixel | Pixel device-level SPL; monthly body template: `https://source.android.com/docs/security/bulletin/pixel/<YYYY>/<YYYY-MM-01>` (note the two path layers: month pages end with `-01`, SPL dates end with `-05`) |
| NVD | `https://nvd.nist.gov/vuln/detail/<CVE-ID>` | CVE details, reference links |
| MITRE CVE record | `https://www.cve.org/CVERecord?id=<CVE-ID>` | Official record |
| syzkaller upstream dashboard | https://syzkaller.appspot.com/upstream | Fix commits, crash types, reproducers |
| oss-security archive | https://www.openwall.com/lists/oss-security/ | Technical discussion |
| Google Project Zero blog | https://googleprojectzero.blogspot.com/ | Deep analysis |
| Google bug tracker archive | https://issuetracker.google.com/ | Some require login; if inaccessible handle per pitfalls §8 |

## Public PoC / privilege-escalation research (leads and verification only; must pass the root-feasibility.md checklist)

| Source | Entry point |
|---|---|
| GitHub search | `https://github.com/search?q=<CVE-ID>+exploit&type=repositories` |
| Exploit-DB | https://www.exploit-db.com/ |
| Packet Storm | https://packetstormsecurity.com/ |

## Vendor source (enhanced mode L2)

- Vendor Gitiles / GitHub mirrors have no unified entry point; locate by vendor with websearch (e.g. `site:github.com <vendor> kernel`), then record the result in the state's target verification section.
- If vendor source cannot be located, L2 is unavailable; follow the source-acquisition priority in race-analysis.md and stay at L1.

### Two supplementary anti-bot recipes

**Recipe A: bulk-enumerate kernel CVEs by month** (when lore is unreachable and NVD search is restricted)

```bash
# Shallow-clone the authoritative CVE repository (huge: measured cves/2026 ~842MB / 83k JSON; sparse-checkout only the target year)
# Use an agreed global cache directory, reused across runs, so it is never downloaded twice:
CACHE=~/.hokorobi/cvelistV5
[ -d "$CACHE" ] || git clone --depth 1 --filter=blob:none --sparse \
    https://github.com/CVEProject/cvelistV5 "$CACHE"
cd "$CACHE" && git sparse-checkout set cves/2026
# Layout: cves/2026/<NNxxx>/CVE-2026-<N>.json; iterate by thousand-block of the number
# The **correct field** for filtering the Linux kernel CNA (measured):
#   cveMetadata.assignerShortName == "Linux" (orgId 416baaa9-...)
#   ⚠️ Do NOT use providerMetadata.shortName=="kernel" or containers.cna.assignerShortName — measured 0 hits
# Self-check: if 0 hits, first validate the field with jq '.cveMetadata.assignerShortName'; never announce "no CVEs in window" on that basis
```

**Recipe A2: converge by subsystem + date buckets** (when the CNA total is in the thousands):

- First filter to the window by fix/disclosure date (`jq` on `cveMetadata.datePublished` or the fix referenced);
- Then bucket by affected file-path prefix: `net/`, `fs/`, `mm/`, `ipc/`, `drivers/android/`, `kernel/`, etc., intersected with target component presence (IKCONFIG);
- **Declare coverage** in the artifact: which months / how many records were enumerated, which buckets were sampled, which were not read line by line — explicitly mark uncovered parts "not read line by line"; never imply a full review.

**Recipe B: commit hash / symbol → CVE reverse lookup**

- NVD keyword search (measured working): `https://nvd.nist.gov/vuln/search/results?query=<commit-hash or function name>&search_type=all`
- Local cvelistV5 grep: `grep -r "<commit-hash>" cves/2026/` (references usually contain the fix commit)
- A reverse-lookup hit only proves association, not the disclosure date — take the disclosure date from the CVE record itself.

## Usage discipline

- Every piece of evidence entering state/report records "source URL + access date (YYYY-MM-DD)".
- If the entries above fail (404, login required, deleted, anti-bot blocked) → switch mirrors per "Anti-bot interception and fallbacks"; only when all fail record it in Blockers per pitfalls §8; never pass off a second-hand retelling as evidence.
- websearch is only for discovering candidates and locating vendor repos not covered by the table; any conclusion supporting an R level must return to a verifiable first-party source (including content-equivalent mirrors).