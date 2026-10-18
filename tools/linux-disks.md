# Linux disks from scripts — partitioning, filesystems, BIOS GRUB, Windows chainload

Lessons from an installer that clones a running Debian 13 system onto another disk (built and rehearsed under QEMU, 2026-10). Generic.

## sfdisk script syntax
- A script line is comma-separated **named fields**: `start=`, `size=`, `type=`, `name=`, `uuid=`, `attrs=`, `bootable`. Omit `start` for "next free sector", omit `size` for "to the end". Header lines: `label: gpt`, `unit: sectors`, `first-lba: 2048`.
- The positional form (`- 2048 TYPE name=…`) is rejected: `line N: unsupported command`.
- Example — GPT for a BIOS-GRUB system with a separate boot filesystem and a data partition:
  ```
  sfdisk -q --wipe always /dev/sdX <<SF
  label: gpt
  unit: sectors
  first-lba: 2048
  size=2048,    type=21686148-6449-6E6F-744E-656564454649, name="bios-boot"
  size=2097152, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, name="boot", uuid=$BOOT_UUID
  size=$ROOT_SECTORS, type=4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709, name="root-x86-64", uuid=$ROOT_UUID
  type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, name="data", uuid=$DATA_UUID
  SF
  udevadm settle
  ```
- `sfdisk` comes from the **`fdisk`** package; `util-linux` alone does not ship it on a minimal Debian image.
- Partition device names after the table is written: `lsblk -n -o PATH /dev/sdX` (handles `sda2` vs `nvme0n1p2`).

## mkfs quirks
- `mkfs.fat` / `mkfs.vfat` (dosfstools) has **no `-q`**: an unknown option prints the help and exits 1 — under `set -e` the script dies there, after whatever ran before it. `mke2fs -q` is fine.

## BIOS GRUB on a GPT disk
- Needs a **BIOS boot partition** (type `21686148-6449-6E6F-744E-656564454649`, 1 MiB) for core.img. From any running system with `grub-pc-bin`: `grub-install --target=i386-pc --boot-directory=/mnt/target/boot /dev/sdX`. Afterwards the MBR starts `eb 63 90` (`dd if=/dev/sdX bs=1 count=3 | od -An -tx1`) — a quick "did the installer get that far" check.
- A hand-written `grub.cfg` needs only: `insmod part_gpt`, `insmod fat`, `search --no-floppy --fs-uuid --set=root <boot filesystem UUID>`, then `linux /path/vmlinuz root=PARTUUID=… <cmdline>` and `initrd a b c` (several initrds concatenate; microcode first).
- **Windows on an MBR disk** — Debian's own `/etc/grub.d/30_os-prober` recipe (grub-common 2.12):
  ```
  menuentry "Windows" {
      insmod part_msdos
      insmod ntfs
      insmod drivemap
      search --no-floppy --fs-uuid --set=root <NTFS serial of "System Reserved">
      drivemap -s (hd0) ${root}
      chainloader +1
  }
  ```
  bootmgr lives on the small "System Reserved" partition, not on C:; the "UUID" `lsblk` shows for NTFS is the volume serial. `drivemap -s` swaps BIOS drive numbers so bootmgr finds its disk as drive 0x80 when the Linux disk boots first. A UEFI/GPT Windows cannot be chainloaded from BIOS GRUB at all — check `PTTYPE` of the Windows disk first (`lsblk -o NAME,PTTYPE,PARTTYPENAME,FSTYPE,LABEL,UUID`).

## systemd and cloned disks
- `systemd-gpt-auto-generator` mounts a partition of **type ESP** with no fstab line, as an autofs automount (seen at `/boot` on Debian 13, systemd 257, although `/efi` existed); mounting it a second time fails with `Can't mount, would change RO state` — read the files under the existing mount.
- A cloned disk's fstab must not use `LABEL=` names the source medium also carries (stick and installed disk present together): rewrite to `PARTUUID=` at install time.

## Rehearsing an installer in QEMU
- Phase 1: boot the source image as a USB stick (`-device qemu-xhci -device usb-storage,bus=xhci.0,drive=stick,bootindex=0`) with an empty raw "SSD" and a fake Windows disk (NTFS partitions labelled "System Reserved" and "Windows") attached as virtio drives; run the installer over ssh (`hostfwd=tcp::2222-:22`). Phase 2: boot the SSD alone and check `findmnt /`, the data mount, `systemctl --failed`.
- `scp` the current installer into the booted guest before running it — an installer fix then needs no image rebuild. Fail fast: when the MBR's first bytes are still zero after phase 1, skip phase 2 (a boot that cannot happen otherwise eats the whole ssh timeout).
- The fake Windows disk has no boot code, so the chainload itself is not exercised — only that the menu entry is generated and parses.
