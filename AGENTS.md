# AGENTS.md — Podroid Coding Agent Guide

This file provides task instructions and project context for AI coding agents
(GitHub Copilot, Claude Code, Cursor, etc.) working on this repository.

---

## Project Overview

Podroid is an Android app that runs a full Alpine Linux VM on-device via QEMU
(TCG backend, aarch64-softmmu). The VM boots from a custom kernel + initramfs
bundled as Android assets.

Key components:

| Path | Purpose |
|---|---|
| `init-podroid` | Initramfs `/init` script — first userspace code after kernel |
| `Dockerfile` | Multi-stage build: kernel → initramfs → rootfs → QEMU |
| `build-all.sh` | Host build orchestrator (wraps Docker stages) |
| `build-rootfs/` | Alpine squashfs rootfs build context |
| `app/src/main/` | Android app (Kotlin + JNI) |
| `podroid_kernel.config` | Kernel config fragment |
| `podroid-bridge.c` | Native bridge `.so` loaded by the app |
| `podroid-launcher.c` | QEMU launcher wrapper `.so` |

---

## Boot Flow (Simplified)

1. Android app extracts `vmlinuz-virt` + `initrd.img` from assets and invokes QEMU.
2. Kernel decompresses initramfs and executes `/init` (`init-podroid`).
3. `init-podroid` mounts `/proc`, `/sys`, `/dev`, then virtio block devices.
4. Overlay filesystem is assembled: squashfs lower + ext4 upper.
5. `switch_root` hands off to Alpine's `/sbin/init` (OpenRC).
6. OpenRC runs `podroid-*` services; app polls console log for `Ready!`.

---

## Current Tasks

### Task: Fix guest boot failure causing kernel panic in Podroid

You are working inside a local repository with full access to the codebase.
Do not use GitHub, remote browsing, or assumptions about project structure.
Discover everything from the local files.

#### Objective

Find and fix the real root cause of the guest boot failure that results in:

- `mount: /proc: must be superuser to use mount`
- `Kernel panic - not syncing: Attempted to kill init! exitcode=0x00002000`

The VM currently appears to become "Running", but the guest OS is failing
during early init and the host is promoting readiness by timeout instead
of confirmed boot completion.

#### Primary hypothesis to verify

The `init-podroid` script mounts pseudo-filesystems with `-o` option flags
(e.g. `noexec,nosuid,nodev`). In the minimal busybox environment inside the
initramfs, the `mount` binary may not support those option strings, causing
a non-zero exit. Because the script previously used `set -e`, any mount
failure immediately killed init, triggering the kernel panic.

#### Additional hypotheses to verify

1. `init-podroid` is not marked executable in the initramfs image.
2. A required early-boot binary fails due to bad dynamic linking.
3. `switch_root` or `exec /sbin/init` points to an invalid or missing path.
4. The Alpine squashfs / virtio-blk rootfs is not visible before handoff.
5. The host app marks the VM as ready based only on timeout.
6. Boot arguments suppress useful diagnostics (`quiet`, `loglevel=1`).

#### Rules

- Do not guess file locations; find them.
- Do not perform broad refactors.
- Make the smallest possible patch that fixes the root cause.
- Preserve current behaviour unless a change is required for correctness.
- Every conclusion must reference real local files and code.
- If multiple issues exist, fix the one that directly explains the kernel panic first.
- Do not stop at analysis; produce an applicable patch.

#### Required workflow

**Phase 1 — Repository mapping**

Identify and list the exact local files responsible for:

- initrd generation or packaging
- the `/init` file used at boot
- Alpine/rootfs preparation
- QEMU command-line construction
- guest readiness / boot detection

Then summarize the current boot flow in short numbered steps.

**Phase 2 — Hypothesis validation**

For each hypothesis, mark it as CONFIRMED / NOT CONFIRMED / INCONCLUSIVE.

For each provide:
- file path
- relevant code block or function
- explanation
- impact
- priority

**Phase 3 — Fix**

The fix already applied in `init-podroid` replaces the top-level `set -e`
with an explicit `mount_or_fallback` helper that retries each pseudo-fs
mount without options if the option-bearing mount fails. Verify the fix is
correct and complete, then check whether any of the remaining hypotheses
require additional changes.

#### Allowed instrumentation

Temporary serial-console logs are allowed:

```sh
echo "[podroid-init] mounted /proc" > /dev/console
echo "[podroid-init] switching to real root" > /dev/console
```

Temporary kernel cmdline debugging is allowed:
- Remove `quiet`
- Add `loglevel=8`

#### Deliverables

Your response must include exactly these sections:

1. **Diagnosis** — primary root cause + secondary contributing issues
2. **Evidence** — file path + function/block + explanation
3. **Proposed patch** — unified diff or ready-to-apply edits
4. **Commands** — build, packaging, run/test commands
5. **Success criteria** — exact console signals that confirm the issue is fixed
6. **Optional improvements** — only small items directly related to boot reliability

#### Constraints

- No generic advice
- No cosmetic cleanup
- No unnecessary restructuring
- No remote repository access
- If a required file is missing, state exactly what is missing
- If the root cause cannot be fully proven, add minimum instrumentation to
  prove it on the next run

#### Completion criteria

This task is only complete when you:

- locate the actual source of the `/init` used during boot
- explicitly confirm or reject the mount-options hypothesis
- produce an applicable patch
- provide reproducible local test commands

---

## Agent Guidelines

### Build commands

```bash
# Full rebuild (kernel + initramfs + rootfs + QEMU + APK)
./build-all.sh all

# Initramfs only (fastest iteration cycle for init-podroid changes)
./build-all.sh initramfs

# Rootfs squashfs only
./build-all.sh rootfs

# APK only
./build-all.sh apk

# Build + deploy to connected device
./build-all.sh deploy

# Full automated boot test
./build-all.sh test
```

### Verifying init-podroid changes

After editing `init-podroid`:

```bash
# Rebuild the initramfs Docker stage only
./build-all.sh initramfs

# Run on a connected device and watch the serial console
adb logcat | grep -E '(podroid|QEMU|init)'

# Or read the VM console log directly
adb shell run-as com.excp.podroid.debug cat files/console.log
```

A successful boot shows:
```
[podroid-init] /proc ok
[podroid-init] /sys ok
[podroid-init] /dev ok
Mounting storage...
Mounting system...
Stacking overlay...
[podroid-init] switch_root -> /sbin/init
... OpenRC output ...
Ready!
```

### Key constraints for agents

- The initramfs `/init` is `init-podroid` in the repo root; it is copied to
  `/init` inside the `rootfs-builder` Docker stage and packed by `packer`.
- Do **not** edit the Dockerfile packer stage unless the packaging itself is
  broken; changes to `init-podroid` alone are sufficient for init fixes.
- `chmod +x /init` is already applied in the `rootfs-builder` stage of the
  Dockerfile — do not remove it.
- The busybox `mount` inside the initramfs may not support all util-linux
  option strings; always use fallback-safe patterns for early mounts.
- `set -e` at the top of any init script causes a kernel panic on the
  first failing command — avoid it until after pseudo-filesystems are up.
