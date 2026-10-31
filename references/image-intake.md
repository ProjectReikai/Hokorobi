# Image Intake — Parsing Image Input

The user may not provide the "device + SPL + branch" trio and instead drop a kernel image (`boot.img` / `vendor_boot.img` / `Image` / `vmlinux`). **Never ask the user about kernel internals** — the image itself is the most authoritative source of target metadata; parse it yourself. This file is the standard parsing procedure.

## Input and output

| Input | Output |
|---|---|
| boot.img / vendor_boot.img | Kernel Image (possibly compressed) → version string → all metadata |
| Raw Image / vmlinux | Go straight to the version-string/config extraction steps |

Write the output to state (run metadata):

```
- kernel base: 6.6.118
- GKI generation: android15 (from the android15-8 KMI string) → branch android15-6.6
- build commit: g<hash> → **non-authoritative** (OEM private build-tree top commit; never a patch criterion)
- build id: ab<buildid> (Android Build ID)
- build date: 2026-07-22 (ranking signal only, see below)
- build type: user / userdebug / eng / debug-variant (determined from config, see below)
- snapshot anchor: refs/tags/android15-6.6.118_r00 (used only to scope candidates and set the delta start)
- config: extracted (IKCONFIG) / not extracted (<fallback note>)
```

**Criteria separation for OEM images (hard rule)**:

- The version string's `g<hash>` is **always marked `non-authoritative`** on OEM images — it is the build top commit of a vendor-private tree, a 404 in the public tree is normal, and it must never be used for any patch-status judgment.
- The version number/tag does exactly two things: scope candidates (which fixes exist after that baseline) and set the delta start. **It never infers patch status.**
- `build_date` is only a ranking signal: a fix later than build_date → high priority (physically impossible for it to be in the build); the conclusion still requires L3.
- The threshold for patch conclusions and R1: L3 binary fingerprints, public OEM source, or real-device validation (see patch-analysis.md).

**Build-type determination**: when the version string/Build ID is insufficient to determine the build type, read the config — debug options enabled such as `CONFIG_KASAN=y` / `CONFIG_KFENCE=y` / `CONFIG_DEBUG_PAGEALLOC` → mark `debug-variant`.

**debug-variant decision table** (evidence applicability; prevents improvisation each run):

| Level | debug/KASAN image | Rule |
|---|---|---|
| R1 | ✅ allowed | For candidate discovery; record the build type |
| L3 | ✅ allowed | Must have independent binary fingerprints; record config differences (KASAN/KFENCE/LTO/compiler) |
| L4 | ❌ no production-level conclusion | Experiments on a debug image count only as **L4-debug** |
| R5 | ⚠️ conditional | Debug reproductions may be recorded, but never treated as production reproductions |
| R6 | ❌ requires production-environment validation | Debug results cannot be upgraded directly |

**Conclusion scope (important)**: debug configs (KASAN/KFENCE), compiler optimization, LTO, kernel config, and OEM conditional compilation can all change the actual machine code or even the code paths — **an L3 conclusion holds only for "the binary that was analyzed"**. Before migrating to a production image, verify code paths and build differences (config diff + disassembly comparison of key functions); until then label the conclusion `scoped-to-analyzed-binary`.

Also warn that "the image may not be your device's actual kernel" (wrong image supplied, or a diagnostic package) — write the warning at the top of state.

## Parsing steps (shell + python3, no external dependencies)

### 1. Parse the boot image header

Read the first 64 bytes to determine the header version:

- magic `ANDROID!` (8 bytes);
- **v3/v4**: `kernel_size`@8, `ramdisk_size`@12, `header_size`@20, `header_version`@40; in v4 the ramdisk lives in vendor_boot (boot.img's ramdisk_size may be 0);
- **v0/v2**: fields are kernel_size@8, kernel_addr@12, ramdisk_size@16, …, `page_size`@36, `dt_size`@28, `header_version`@40.

Kernel payload offset = **page rounded up** (v4 header_size=1584, but measured payload is still aligned to a 4096 page — trust the actual magic scan: look for the kernel magic within 4KB/8KB).

### 2. Identify and decompress the kernel format

By the payload's leading magic:

| magic | Format | Handling |
|---|---|---|
| `1f 8b` | gzip | `gunzip` / python gzip decompress |
| `04 22 4d 18` | LZ4 | `lz4 -d` (record a Blocker if the tool is missing) |
| `ARM\x64` at offset +0x38 | Raw arm64 Image (uncompressed; common on newer 30MB+ devices) | Use directly |

### 3. Extract the version string (the most critical step)

```bash
strings -a <Image> | grep -m1 "Linux version"
```

Example output and field interpretation:

```
Linux version 6.6.118-android15-8-g<hash>-ab<buildid>-4k (kleaf@...)
              └─┬─┘  └───┬──┘ └──┬──┘ └──┬──┘ └┬┘
        stable base  GKI gen  commit   Build ID  4K-page variant
```

- **Stable base** (6.6.118): the image's own exact version anchor;
- **GKI generation** (`android15-8`): `android<OS gen>-<KMI gen>` → target branch `android15-6.6`;
- **`g<hash>`**: the build tree's top commit;
- **Build date** and compiler (clang/lto/pgo) can also be evidenced from the string.

### 3.5 Commit authority determination (404 check, settles the path in one step)

Send a HEAD-level request for `g<hash>` and use the HTTP status code to distinguish the two paths:

```bash
curl -s -o /dev/null -w '%{http_code}' \
  'https://android.googlesource.com/kernel/common/+/<hash>'
```

| Result | Meaning | Path |
|---|---|---|
| **200** | Commit exists in public kernel/common | **Official/GKI path**: exact anchor usable, version inference allowed |
| **404** | Private-tree commit (OEM/self-built) | **OEM path**: `non-authoritative`; patch conclusions must come from L3/source/real device |

- On 404, additionally search the OEM's public source mirror (GPL release): finding the same hash flips the situation to source-available, the highest priority.
- 404 ≠ "the vendor did not merge the fix"; it only means "this build commit cannot be located publicly" — patch status is determined separately per patch-analysis.md.

### 4. IKCONFIG extraction (the device's real config)

```
Search the Image byte stream for b'IKCFG_ST'; the gzip stream starts 8 bytes after it.
Decompress streaming with zlib.decompressobj(16+MAX_WBITS) (tolerates trailing garbage).
```

This produces the full `.config` (~7000+ lines); persist it (e.g. `state/config.txt`).

- **IKCONFIG is the first authority for component-presence verification**: it is the device's actual compile config and outranks the online gki_defconfig (OEMs modify configs; defconfig is only the GKI baseline). The reachability step 0 uses this file first.
- If the image has no IKCFG (`CONFIG_IKCONFIG` off): fall back to gki_defconfig, label it "baseline approximation; OEM additions/removals unverified", and record a Blocker.

### 5. Anchor strategy (the delta window for image input)

In priority order:

1. **Original GKI image**: `g<hash>` exists in kernel/common → exact anchor, delta = `+log/<hash>..refs/heads/<branch>`.
2. **OEM GKI-based image** (the most common OEM scenario for this procedure): `g<hash>` is not in the public tree → use the stable base from the version string to locate **`refs/tags/<branch>.<stable version>_r00`** (e.g. `android15-6.6.118_r00`; the tag shares the stable version number and was measured to exist).
   - **Take the anchor as the `Merge tag '<branch>.<N>_r00' into <branch>` merge commit on the mainline branch** (measured: it can differ from the tag's dereferenced commit by about two weeks); the tag dereference (the "Merge 6.x.y into <branch>-lts" line) is only used to confirm the tag exists and matches the version, never as the anchor.
   - ⚠️ Record a Blocker: the OEM tree = GKI baseline + vendor patches, so "based on 6.6.118" does not mean identical to the tag's content — the OEM may cherry-pick extra fixes or introduce extra holes. **Even GKI-body components can only be inferred to R0**; R1+ requires L3 binary verification or OEM source (see the L3 cherry-pick identification recipe in patch-analysis.md).
3. **Pure upstream version string (no GKI generation marker)**: a non-GKI/self-built kernel; neither quick exclusion nor version inference is available — always go through full code/binary verification, and assess the L level from the binary side.

Image input has no SPL date — **the anchor comes from the version string, not from SPL inference**, which is more precise than "the last tag before SPL". target-mode determination is unaffected (still based on branch EOL / device stopped-update).

## Interfaces with other stages

- **discovery**: the commit-channel delta window uses the step-5 anchor; target metadata comes from intake output, not from the user's dictation.
- **reachability step 0**: check IKCONFIG for component presence first, then fall back to gki_defconfig.
- **patch-analysis**: vendor changes in an OEM image violate precondition 4 of the quick-exclusion path (OEM deep kernel modification) — **even GKI-body components can only be inferred to R0**; R1+ requires L3 binary verification or OEM source (see the L3 cherry-pick identification recipe).
- **race-analysis L levels**: having an image = L3 material in hand (`Image`/`vmlinux`), which supports binary-level patch checking with disassembly tools; without tools, the L level stops at L2 and this is recorded.

## Generate artifacts once, reuse throughout

During intake, generate once and store in `state/`; every later candidate reuses them directly (never re-parse/re-disassemble per candidate):

- `state/Image` (the extracted kernel payload)
- `state/config.txt` (the full .config decompressed from IKCONFIG)
- `state/vmlinux.elf` + `state/symbols.txt` (`vmlinux-to-elf` recovery once, for all candidates' objdump)
- `state/btf_offsets.txt` (candidate struct offsets extracted from BTF, appended as needed)
- `state/raw/` (cache of fetched raw logs / CVEs / blobs)

## Common pitfalls

- v4 header `header_size` does not match the actual payload offset (page-alignment rules get confused) — go by where the kernel magic actually is; never blindly trust the header field.
- An OEM LOCALVERSION may omit `-g<hash>` (stripped release info) — then the anchor degrades to the stable version number; record "build commit unavailable" as a Blocker.
- The Image may be concatenated (kernel + dtb): strings/IKCONFIG searches are unaffected, but truncate by size before disassembling.
- The version string appears multiple times (multiple embedded copies) — take the first hit; the content is identical.