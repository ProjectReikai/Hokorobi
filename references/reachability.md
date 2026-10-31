# Reachability — Ordinary-App Reachability

Goal: confirm whether the affected code can be reached under **ordinary app privileges (untrusted_app SELinux domain, uid 100+)**. This is the basis for R2.

## Step 0: Component-presence check (Android-specific prerequisite)

Candidates from generalized Linux searches (and all candidates) must first confirm that the affected subsystem **exists and is built** on the Android target; otherwise everything downstream is meaningless:

1. **CONFIG verification**: priority order — ① **target image IKCONFIG** (when the user supplied an image, the device's actual compile config, most authoritative; see image-intake.md) ② GKI defconfig (online: `https://android.googlesource.com/kernel/common/+/refs/heads/<branch>/arch/arm64/configs/gki_defconfig?format=TEXT`; OEM additions/removals are unverified, so it is an approximation). `CONFIG_XXX=n` or absent → the component is not in the kernel → **exclude**, record under §6 "component absent".
2. **Vendor modules**: components not covered by defconfig may live in vendor DLKMs (gki_defconfig does not include vendor-module config); check vendor source/device config.
3. **Android semantic differences**: config enabled ≠ Linux desktop semantics. Note:
   - unprivileged user namespaces: disabled by default on Android (trigger chains depending on them do not hold for apps)
   - unprivileged eBPF: disabled on Android (`kernel.unprivileged_bpf_disabled`)
   - io_uring: has a history of being enabled in newer Android versions; verify per version
   - per-syscall SELinux denials and seccomp filtering (the app process's seccomp policy lives on the bionic side in `libc/seccomp/`, encoded BPF; the old entry `system/seccomp` now 404s)
4. If the component exists but the trigger semantics depend on the disabled items above → mark `blocked` (hard block), and record explicitly how this differs from the Linux desktop conclusion.

## Trigger-chain analysis path

Trace backwards from the vulnerability site to the user-space entry, answering layer by layer:

1. **Kernel entry**: which syscall / ioctl / netlink / binder command / filesystem operation / setsockopt triggers the affected function?
2. **Namespaces and mounts**: does that entry exist in the mount namespace visible to an ordinary app? Key Android restrictions:
   - whether SELinux policy allows untrusted_app to access that device node/socket (e.g. `/dev/`, netlink types, binder);
   - whether binder ioctls are open to apps (usually open, but individual commands have permission checks);
   - whether the network subsystem requires NET_ADMIN (apps lack this capability unless they use user namespaces — unprivileged userns is disabled on Android by default);
   - entries disabled on Android or denied by SELinux, such as io_uring, eBPF, perf;
   - vendor/DLKM drivers (camera, GPU, NFC) are reached through HAL service processes rather than directly by apps — here the app is only an indirect attack surface; assess whether the HAL exposes controllable input.
3. **Parameter controllability**: once the entry is reachable, how many trigger parameters can the app control (object size, allocation count, content, lifetime)? Reachable but with very weak parameter control significantly raises exploitation difficulty; record it but do not deny R2 for that reason.

## Verification methods (online)

- Read the permission checks of the entry function in the target kernel source (capability, LSM hooks, SELinux policy references).
- Android SEPolicy is verifiable online in the AOSP source `system/sepolicy/` (android.googlesource.com/platform/system/sepolicy) for untrusted_app allow rules.
- Compare against behavior described on public devices (security research, CDD requirements) to judge whether the vendor tightened or loosened anything.

## R2 determination

| Conclusion | Condition |
|---|---|
| R2 holds | Complete trigger chain: untrusted_app → explicit entry → affected code, with no SELinux/capability hard block |
| R2 partial | The entry needs specific conditions (e.g. reconnect after a particular HAL crash, a specific file present); record the conditions as Blockers |
| R2 does not hold | The entry is hard-denied by SELinux or requires root/capability |
| Indirect | Reached only through a system service process (HAL etc.) | Record the intermediate layer and injectable input surface; state confidence |

## Common misjudgments

- **Generalizing from Linux desktop conclusions**: user namespaces, eBPF, io_uring, and seccomp policy have completely different availability on desktop vs Android. Any Linux-context "escalatable" conclusion must be re-verified item by item in this stage.
- **Treating "apps use Bluetooth/NFC/location every day" as kernel-entry reachability**: framework-layer use ≠ untrusted_app reaching a kernel syscall directly — privileged daemons and the SELinux boundary sit in between; check allow rules one by one.
- **Treating a root/shell-privileged PoC's trigger as app-reachable**: many kernel PoCs require `unshare` first or debugfs; ordinary apps cannot.
- **Ignoring the Treble/GKI boundary**: with a GKI kernel + vendor DLKM, the module boundary determines which component gets hit.
- **Assuming apps can open all /dev nodes**: Android device nodes are controlled by SELinux per domain.

## Socket-family reachability recipe

untrusted_app reachability of socket-class candidates (BT, CAN, NFC, ALG, RXRPC, etc.) is a high-frequency verification item; standard recipe:

1. **SELinux**: shallow-clone sepolicy (`git clone --depth 1 https://android.googlesource.com/platform/system/sepolicy`) and grep the target family's socket-class allow rules, e.g. `grep -rn "bluetooth_socket\|can_socket\|nfc_socket" private/ public/` — the rules referenced by untrusted_app.te and its attributes (bluetoothdomain etc.) are all in the tree. Note: for attribute-granted rules, check which class is actually allowed (e.g. bluetoothdomain grants only unix_stream_socket, not bluetooth_socket).
2. **seccomp**: check whether the app process's syscall filter permits socket(2) for that family. Note the actual location is the **bionic side `libc/seccomp/`** (`system/seccomp` now 404s), and the policy is **encoded BPF (`seccomp_bpfs.h`), so you cannot grep syscall names directly** — verification method: decode the allowlist from the generated header, or compare against AOSP's published app seccomp policy documentation.
3. **Capability prerequisite**: for families like AF_CAN that require `CAP_NET_ADMIN` to create an interface, confirm there is no acquisition path once Android disables unprivileged userns.
4. Vendor sepolicy may tighten but not loosen (if AOSP mainline has no allow, vendor trees usually do not either); mark Blockers when uncertain.

## Verification record format

```
### CVE-YYYY-NNNNN reachability
- entry: <full call chain: syscall/ioctl/...>
- required privilege: untrusted_app / shell / root / specific capability
- SELinux constraints: <referenced policy rules and their source>
- parameter controllability: <high/medium/low + reasoning>
- indirect path: <none / description of the path through HAL xx>
- conclusion: reachable / conditional / blocked / indirect
- R2: granted / conditional / denied
- Blockers: <unconfirmed items>
```

## Gates (end of this stage)

- [ ] Every candidate has a complete call chain (entry → intermediate layer → affected function); missing links are explicitly marked
- [ ] SELinux / capability constraints are checked item by item with sources cited
- [ ] `blocked` candidates are downgraded/excluded (R level capped at the gate 02 conclusion, at most R1), recorded in report §6
- [ ] `conditional / indirect` candidates list all preconditions