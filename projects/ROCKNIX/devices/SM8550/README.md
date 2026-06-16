# SM8550 kernel: building and debugging suspend/resume

This covers building just the `linux` package for SM8550 (Retroid Pocket 6 / AYN Thor)
on an x86_64 build machine, deploying it to a device for testing, and using the SDAM
breadcrumb mechanism to debug suspend/resume failures.

## Build host requirements

Build on genuine x86_64 hardware running Linux, with Docker (or Podman) installed and
the invoking user in the `docker` group. The project's container image
(`ghcr.io/rocknix/rocknix-build:latest`) is amd64-only — there is no emulation step
needed on x86_64, and no native host-toolchain bootstrap to fight. Group membership
changes (`usermod -aG docker $USER`) only take effect in a fresh login session, so log
out/in (or `newgrp docker`) after adding yourself to the group. Do not use `sudo` —
the build explicitly refuses to run as root.

## Building the kernel package

`make docker-package` only forwards variables that are *exported* in your shell, since
they're dumped to a `.env` file and passed into the container. Export, don't just set:

```bash
export PROJECT=ROCKNIX DEVICE=SM8550 ARCH=aarch64 PACKAGE=linux
make docker-package
```

This builds only the `linux` package, not a full system image — much faster than
`make image` for kernel-only iteration.

## Locating the build artifact

```bash
build.ROCKNIX-SM8550.aarch64/install_pkg/linux-<version>/.image/Image
```

SM8550 uses the `qcom-abl` bootloader, so `.image/Image` is not a raw kernel binary —
it's the gzip-compressed kernel concatenated with all device DTBs, wrapped in an
Android-style boot image via `mkbootimg`. This is the file to deploy.

Kernel modules land under:

```bash
build.ROCKNIX-SM8550.aarch64/install_pkg/linux-<version>/usr/lib/kernel-overlays/base/lib/modules/<kernel-version>/
```

## Deploying to a device

The device's bootloader reads a plain file (`KERNEL`) off the FAT `/flash` partition —
not a raw Android boot partition — so deployment is a simple file copy, not a
partition write.

```bash
# back up the current working kernel first
ssh root@<device-ip> "mount -o remount,rw /flash && cp /flash/KERNEL /flash/KERNEL.bak && sync"

# copy the new boot image over
scp build.ROCKNIX-SM8550.aarch64/install_pkg/linux-<version>/.image/Image root@<device-ip>:/flash/KERNEL

# reboot into it
ssh root@<device-ip> "sync && mount -o remount,ro /flash && reboot"
```

Before relying on the existing kernel modules baked into the running SYSTEM image,
confirm `uname -r` on the device matches the version directory under
`usr/lib/kernel-overlays/base/lib/modules/` from the build above. If they differ, drop
the new modules tree into `/storage/.cache/kernel-overlays/` instead of overwriting the
read-only squashfs — see `packages/sysutils/busybox/scripts/kernel-overlays-setup` for
the overlay mechanism.

To recover from a kernel that doesn't boot: `cp /flash/KERNEL.bak /flash/KERNEL` over
SSH, or pull the SD card if SSH is unreachable.

### Verifying the new kernel is actually running

`uname -r` alone won't prove anything if the kernel version string hasn't changed
between builds (e.g. you're only changing patches against the same upstream version).
Confirm the swap took with a checksum match plus a build-timestamp sanity check:

```bash
md5sum build.ROCKNIX-SM8550.aarch64/install_pkg/linux-<version>/.image/Image
ssh root@<device-ip> "md5sum /flash/KERNEL /flash/KERNEL.bak"
```

`/flash/KERNEL` on the device should match your local `Image` hash exactly, and differ
from `/flash/KERNEL.bak`. Since this device boots straight from that file (no copy into
a separate boot partition), a hash match is conclusive proof of what was loaded this
boot.

```bash
ssh root@<device-ip> "cat /proc/version"
```

This should show the build host/user and a timestamp matching your most recent build,
not whatever machine/date built the previous kernel.

## Debugging suspend/resume with the SDAM breadcrumb

`projects/ROCKNIX/packages/hardware/quirks/platforms/SM8550/sleep.d/{pre,post}/003-sdam-debug`
write a marker byte to `/sys/bus/nvmem/devices/spmi_sdam0/nvmem` at offset `0x50` around
each suspend cycle:

- `pre` hook writes `0x01` before suspend begins.
- `post` hook writes `0xff` after resume completes and post-resume hooks run.

Offset `0x50` matters: `spmi_sdam0` only accepts reads/writes within `0x50-0x7f` (48
bytes) — offset `0x0` is out of bounds for this device and always fails with
`Invalid argument` (confirmed against jaewun's working `thor-suspend-fixes` setup,
which uses the same `0x50-0x7f` window). `dmesg` will show
`qcom,spmi-sdam ...: Invalid SDAM offset <N>` for anything outside that range,
regardless of length — even a request matching the device's full reported size fails
if the offset itself is wrong.

After a wake failure, read the byte back (before triggering any further suspend, which
overwrites it):

```bash
dd if=/sys/bus/nvmem/devices/spmi_sdam0/nvmem bs=1 count=1 skip=80 | od -An -tx1
```

- `01` — the kernel never got back from suspend far enough to run the post-resume
  hook. The failure is inside the kernel suspend/resume path itself.
- `ff` — the kernel-level suspend/resume cycle completed. If you still observed a wake
  failure despite this, look at userspace (compositor/display, input, audio), not the
  kernel PM path.
