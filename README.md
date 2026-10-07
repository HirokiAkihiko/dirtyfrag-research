# DirtyFrag Research — 5.4-qgki

Research on CVE-2026-43284 (DirtyFrag) feasibility for Xiaomi devices with kernel 5.4.

## Devices
- veux: 5.4.274-qgki (Snapdragon 695 / SM6375)
- moonstone: 5.4.191-qgki (Snapdragon 695 / SM6375)

## Findings

### DirtyFrag (CVE-2026-43284): NOT FEASIBLE on 5.4
- Bug exists in 5.4 (code from 4.11), but exploit requires MSG_SPLICE_PAGES UDP support (kernel 6.5+)
- 5.4 lacks the primitive → no page-cache write → dead end
- Same for CVE-2026-43500 (RxRPC, needs 6.8+)

### CVE-2026-24088 (Qualcomm Fastboot): PROMISING
- Bug in Qualcomm ABL: `fastboot oem set-gpu-preemption` unsanitized → inject `androidboot.selinux=permissive`
- Confirmed on Snapdragon 695 (SM6375) — same SoC as veux/moonstone
- No bootloader unlock needed
- Tethered (repeat each reboot)

**Test:**
```bash
adb reboot bootloader
fastboot oem set-gpu-preemption 0 androidboot.selinux=permissive
# OKAY = vulnerable, FAILED = patched
```

### CVE-2025-39964 (AF_ALG race): UNCERTAIN
- Affects 2.6.38+, CISA KEV
- Needs ARM64 porting, per-build offsets

## Repos
- Upstream: https://github.com/diabl0w/DFRoot (active)
- Archived fork: https://github.com/ankitrawatgit/DirtyFrag-Android-Root-Jailbreak

## Date
2026-10-07
