---
title: LUKS, Btrfs UUID mismatch, and dracut boot recovery
slug: luks-btrfs-uuid-mismatch-and-dracut-boot-recovery
type: incident
status: draft
date: 2026-02-22
updated: 2026-02-22
tags:
  - linux
  - fedora
  - dracut
  - luks
  - btrfs
  - boot
summary: Fedora dropped into dracut emergency mode because the encrypted root path was not being unlocked consistently before the Btrfs root mount.
verification_status: verified
verified_on: 2026-02-22
---

# LUKS, Btrfs UUID mismatch, and dracut boot recovery

## Context

I rebooted Fedora 42 and got punted straight into dracut emergency mode.

The error chain was basically:

- systemd failed to start cryptsetup
- timeout waiting for dracut init queue hooks
- it kept whining about a UUID (`8c7e...`) it could not find

At first glance, it looked like a missing disk. It was not.

## Goal

- boot the system cleanly without dropping into dracut emergency mode
- prove the relationship between the LUKS UUID and the Btrfs root UUID
- make the initramfs unlock path consistent with the expected mapper name

## Environment

- Fedora 42
- `/dev/sda3` as the encrypted LUKS partition
- `/dev/mapper/cryptroot` as the expected unlocked mapping
- Btrfs subvolumes with `root` mounted at `/` and `home` mounted at `/home`
- dracut + systemd initramfs environment

## The Actual Problem

Dracut was trying to mount the root filesystem (`8c7e...`, Btrfs UUID) but the encryption layer (LUKS UUID `5af4...`) was not being unlocked properly in initramfs. So it waited, timed out, and dropped me into a tiny shell with half the tools missing.

The key realisation was that there were two different UUID layers:

- `rd.luks.uuid=luks-5af4...` is the LUKS container UUID
- `root=UUID=8c7e...` is the Btrfs filesystem UUID inside the unlocked mapper

Dracut complaining about `8c7e...` did not mean the encrypted disk was gone. It meant the thing containing it had not been unlocked correctly.

## Mistakes I Made

### 1. Assuming initramfs has real tools

I immediately went for `lsblk`, `cryptsetup`, and `vgscan` like it was a normal system.

- `lsblk`: not installed
- `cryptsetup`: not a command
- `vgscan`: not a command

Turns out the dracut shell’s `$PATH` is basically `/usr/local/bin:/usr/bin`. Anything in `/sbin` might as well not exist. Fedora was not broken; I was just expecting a full distro inside an initramfs.

### 2. Chasing the wrong UUID

I tried grepping for `5cf...` like an idiot. The actual LUKS UUID was `5af...`.

That produced fake “nothing found” results and wasted time by making the disk look missing when it was not.

## Live Recovery

Since `cryptsetup` was not in `PATH`, I used:

```bash
systemd-cryptsetup attach cryptroot /dev/sda3 -
```

After entering the passphrase, `/dev/mapper/cryptroot` appeared as `dm-0`. Then I checked what the mapper contained:

```bash
blkid /dev/mapper/cryptroot
```

It showed Btrfs UUID `8c7e...` and label `fedora`.

Dracut also expects the real operating system root to be mounted at `/sysroot`, so I mounted the `root` subvolume there:

```bash
mkdir -p /sysroot
mount -t btrfs -o subvol=root /dev/mapper/cryptroot /sysroot
exit
```

Exiting let dracut continue booting normally.

## The Final State

- `/dev/sda3` unlocks to the expected `cryptroot` mapper
- the initramfs has consistent crypttab information
- Btrfs root mounts cleanly from the unlocked mapper
- the system boots normally without emergency shell recovery

## What Changed

After booting, I verified the real layout:

- `/dev/sda3` is the encrypted partition
- `/dev/mapper/cryptroot` is the unlocked mapping
- Btrfs subvolumes `root` and `home` are mounted as expected

The consistency issue was that `/etc/crypttab` existed, but the mapping name was `luks-5af4...` while the actual mapper I expected was `cryptroot`.

I fixed it by explicitly matching the mapping name and rebuilding initramfs:

```bash
sudo cp -a /etc/crypttab /etc/crypttab.bak.$(date +%s)

# Changed entry in /etc/crypttab from:
# luks-5af4... UUID=5af4... none discard
# To:
# cryptroot UUID=5af4... none discard

sudo dracut -f
```

## Verification

Proof captured after the fix:

- checked initramfs contents and confirmed `etc/crypttab` was included
- confirmed `systemd-cryptsetup` and the relevant generator were present
- checked kernel cmdline and confirmed `rd.luks.uuid=luks-5af4...`, `root=UUID=8c7e...`, and `rootflags=subvol=root`
- rebooted cleanly without returning to dracut emergency mode

## Failure Modes Worth Caring About

- mixing up the LUKS UUID and the inner Btrfs UUID
- assuming a missing Btrfs root UUID means the disk itself is gone
- expecting full userspace tooling inside dracut emergency shell
- leaving `crypttab` mapping names inconsistent with the mapper name the boot path actually expects
