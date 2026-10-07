# Kernel Exploit Research — Xiaomi 5.4-qgki (veux & moonstone)

**Date:** 2026-10-07/08
**Devices:** Redmi Note 11 Pro 5G (veux, 5.4.274-qgki) & POCO X5 5G (moonstone, 5.4.191-qgki)
**SoC:** Snapdragon 695 (SM6375)

---

## Summary

| CVE | Status | Notes |
|-----|--------|-------|
| CVE-2026-43284 (DirtyFrag) | ❌ Dead | Needs MSG_SPLICE_PAGES (kernel 6.5+) |
| CVE-2026-24088 (Qualcomm fastboot) | ⚠️ Bootloader-specific | TheFliss: "unknown command" on his device |
| CVE-2022-2602 (io_uring UAF) | ❌ Dead | Kernel blocks register path (EFAULT/EINVAL) |
| CVE-2022-20421 (BadSpin) | ❌ Tried, no success | Builds OK, loops without winning race |
| CVE-2023-20938 (binder UAF) | ✅ Vulnerable (both) | Not yet attempted |
| CVE-2023-4622 (af_unix UAF) | ✅ Vulnerable (both) | Not yet attempted |
| CVE-2026-43499 (GhostLock) | ❓ Possibly vulnerable | GhostLock app doesn't work on 5.4 by design |
| KGSL CVEs (5x) | ❓ Possibly vulnerable | 2024-38399 has writeup+PoC |

---

## 1. DirtyFrag (CVE-2026-43284)

**Verdict:** Not feasible on 5.4.

The bug exists in 5.4 (code from 4.11), but the exploit requires `MSG_SPLICE_PAGES`
UDP support for the page-cache write primitive. This was introduced in kernel 6.5.
On 5.4, `splice()` to UDP does a regular copy — no shared page attachment.

CVE-2026-43500 (RxRPC variant) also not applicable — RxRPC code introduced in 6.8.

Repo: https://github.com/HirokiAkihiko/dirtyfrag-research

---

## 2. CVE-2026-24088 (Qualcomm ABL Fastboot)

**Verdict:** Bootloader-specific, not universal.

Bug in Qualcomm ABL: `fastboot oem set-gpu-preemption` doesn't sanitize arguments,
allowing injection of `androidboot.selinux=permissive` into kernel cmdline.

- Affects SM6375 (confirmed by third-party research)
- No bootloader unlock needed
- Tethered (must repeat each boot)

**Test result (TheFliss):** `FAILED (remote: 'unknown command')` — his bootloader
doesn't implement the command. This CVE depends on the specific ABL build.

---

## 3. CVE-2022-2602 (io_uring UAF)

**Verdict:** Dead on these devices. Kernel actively blocks the register path.

**Work done:**
- PoC ported from x86_64 to ARM64 Android (no liburing, vendored minimal helpers)
- Compiled with NDK r27d → 2.2MB static binary
- Device testing via LADB with 4 diagnostic binaries

**Findings:**
- `CONFIG_IO_URING=y` confirmed
- `io_uring_setup` works with plain flags (SQPOLL blocked → needs CAP_SYS_ADMIN)
- `IORING_REGISTER_FILES` → EFAULT (consistent, even via raw SVC)
- `IORING_REGISTER_BUFFERS` → EINVAL
- Conclusion: kernel-level block on io_uring register path, not SELinux/seccomp

The UAF primitive requires successful `IORING_REGISTER_FILES`. Without it,
the exploit chain cannot proceed.

Bundle: `~/workspace/cve-2022-2602-bundle/` (source, binaries, diagnostics)

---

## 4. CVE-2022-20421 (BadSpin)

**Verdict:** Builds and runs, but doesn't succeed on veux.

**Work done:**
- Source: `mateusvdcastro/badspin`
- Adapted for veux:
  - Removed SPL check in `exploit.c` (`find_dev_config()`)
  - Added veux device entry in `dev_config.h` (model 2201116SG, Android 13, kernel 5.4.274, SPL 2024-12)
- Built via GitHub Actions (NDK r27d) → `libbadspin.so` (345K, ARM64)
- Tested on device via LADB with `TEST_VULN=1`

**Test result:** `dev_config_init` initially failed (SPL month mismatch: entry had
month=1, device has 2024-12). Fixed via GitHub. After fix, exploit runs and
triggers the vulnerability ("Trigger use-after-free") but loops indefinitely
without winning the race condition.

Repo: https://github.com/HirokiAkihiko/badspin-veux (private)

---

## 5. Remaining Candidates (verified vulnerable, not yet attempted)

### CVE-2023-20938 (binder UAF)
- **Status:** Vulnerable on both veux and moonstone
- Fix commits `bdc1c5fac982` and `4df153652cc4` not present in Xiaomi source
- Next recommended target

### CVE-2023-4622 (af_unix UAF race)
- **Status:** Vulnerable on both devices
- Fix `4821df2ffe38` not present; `skb_peek_tail` still lockless
- Race condition — harder to make reliable

### CVE-2026-43499 (GhostLock/futex)
- **Status:** Possibly vulnerable (pattern `current->pi_blocked_on = NULL` present)
- Note: GhostLock app itself doesn't work on 5.4 by design (timing issue)

### KGSL CVEs (Adreno GPU)
- CVE-2023-33106, CVE-2023-33107, CVE-2023-33021, CVE-2024-38399, CVE-2025-27038
- **Status:** Possibly vulnerable (no KGSL updates since 2023-06 on veux)
- CVE-2024-38399 has public writeup + PoC
- GPU exploits are complex

---

## Key Lesson

**Version labels are misleading.** veux-r-oss is labeled 5.4.274 but is missing
stable 5.4.y backports that should be there (binder fixes from 2023-06, af_unix
fix from 2023-08). Always verify at the code level, not by version number.

---

## Files & Repos

- DirtyFrag research: https://github.com/HirokiAkihiko/dirtyfrag-research (public)
- BadSpin veux: https://github.com/HirokiAkihiko/badspin-veux (private)
- CVE-2022-2602 bundle: `~/workspace/cve-2022-2602-bundle/`
- CVE hunt evidence: `~/workspace/cve-hunt/`
