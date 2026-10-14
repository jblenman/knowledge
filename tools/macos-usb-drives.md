# macOS — USB sticks and external disks from the shell

Identify, size, test and reformat a USB stick from a terminal session: read-only commands first, the destructive ones marked. Verified on macOS 26.6 (Apple silicon), 2026-10-14.

## Identify the drive

| Need | Command | Notes |
|---|---|---|
| Which disk is the stick | `diskutil list` | look for `(external, physical)`; `FDisk_partition_scheme` = MBR, `GUID_partition_scheme` = GPT; a `(free space)` row = unallocated space |
| Size, exact sector count | `diskutil info diskN` | `Disk Size: … (exactly N 512-Byte-Units)`. A genuine stick reports an odd count (NAND minus spare area); a perfectly round count such as 2,048,000,000 is a programmed value — a counterfeit hint, not proof |
| Volume, filesystem, used and free | `diskutil info diskNsM` | FAT32 shows as `Windows_FAT_32`; `Allocation Block Size` |
| USB vendor and product IDs, serial, link speed | `ioreg -p IOUSB -l -w0 \| grep -E '"(USB Product Name\|USB Vendor Name\|USB Serial Number\|idVendor\|idProduct\|bcdUSB\|Device Speed\|bMaxPacketSize0)" ='` | ids print in **decimal** (1165 = 0x048d). `Device Speed` 1 = full (12 Mb/s), 2 = high (USB 2.0, 480 Mb/s), 3 = SuperSpeed (5 Gb/s), 4 = SuperSpeed+. `bcdUSB` 512 = 0x0200 = a USB 2.0 device; `bMaxPacketSize0` 64 = USB 2.0 (SuperSpeed devices report 9, meaning 512) |
| Same, human-readable | `system_profiler SPUSBHostDataType` | **macOS 26: the data type is `SPUSBHostDataType`; `SPUSBDataType` still runs but returns an empty list** (`system_profiler -listDataTypes` shows the live names). Prints hex IDs, `Link Speed: 480 Mb/s`, the serial |
| SCSI inquiry strings from the bridge | `ioreg -l -w0 \| grep '"Device Characteristics" ='` | `Vendor Name` / `Product Name` / `Product Revision Level`; `diskutil info` shows the product string as `Device / Media Name` |
| Decode vendor:product | `curl -o usb.ids http://www.linux-usb.org/usb.ids` then `awk '/^048d  /{p=1;print;next} /^[0-9a-f]{4}  /{p=0} p' usb.ids` | the community database; one request |
| Real throughput | write 1 GiB: `dd if=/dev/urandom of=/Volumes/X/t.bin bs=1m count=1024; sync` · drop the read cache: `diskutil unmount diskNsM; diskutil mount diskNsM` · read: `dd if=/Volumes/X/t.bin of=/dev/null bs=1m` | no sudo; delete the file afterwards. USB 2.0 sticks land around 15–30 MB/s |

Raw device access: `/dev/diskN` and `/dev/rdiskN` are `root:operator 640` and the console user is not in `operator`, so `fdisk /dev/diskN`, `dd` to or from the device and raw-probe tests need `sudo` (the user types the password; in Claude Code the `!` prefix runs the command on a terminal). `diskutil eraseDisk`, `partitionDisk`, `unmountDisk`, `mount` and `eject` work **without** sudo on an external disk. SMART and the "Solid State" field are empty behind a USB bridge.

## Counterfeit-capacity sticks — the signature

Cheap "512 GB / 1 TB / 2 TB" sticks from marketplaces are very often 8–64 GB of NAND whose controller is programmed to report a huge size; writes past the real end are dropped or wrap around, and files vanish once the stick is unplugged. Signature seen on a no-brand "1 TB" stick (2026-10-14):

- USB `048d:1234` → usb.ids "Chipsbank CBM2199 Flash Drive"; SCSI inquiry `VendorCo` / `Disk 3.0` / rev `2.00` — placeholder strings left by the controller's mass-production tool (`VendorCo ProductCode` is the textbook fake-drive device name).
- Enumerates as USB 2.0 high-speed (480 Mb/s) with "3.0" in the name; ~18 MB/s write, ~26 MB/s read.
- Reported size exactly 2,048,000,000 sectors (1,048,576,000,000 bytes).
- None of this is proof: **test before trusting it** (below). Sources: [datarecovery.com — The 2TB flash drive scam](https://datarecovery.com/2022/03/the-2tb-flash-drive-scam-why-high-capacity-flash-drives-are-fakes/) (fake 2 TB sticks built on Chipsbank CBM2199E with 8–32 GB of NAND), [HD Sentinel — detect a fake pendrive](https://www.hdsentinel.com/how_to_detect_fake_pendrive_memory_card.php), [linux-usb usb.ids](http://www.linux-usb.org/usb.ids).

## Test the real capacity

| Method | Needs | Time on a USB 2.0 stick | What it tells |
|---|---|---|---|
| **Raw spot test**: write an 8 MiB random block at 1, 16, 40, 64, 100, 200, 300, 500, 700, 900 and 975 GiB (`dd if=pat of=/dev/rdiskN bs=1m seek=<MiB> count=8`), read each back from `/dev/rdiskN` (raw = uncached, no re-plug needed), `cmp` | sudo; destroys the contents | ~3 min | a FAIL past some offset bounds the real size; an early offset failing after later writes = wrap-around |
| **F3** (`brew install f3`, v10): `f3write /Volumes/X` fills the free space with 1 GiB files, `f3read /Volumes/X` verifies them; `--end-at=N` stages it | no sudo; the volume must span the whole disk, so reformat first | the first 64 GiB ≈ 1.8 h; a full 1 TB ≈ 27 h at 18 / 26 MB/s | the real usable size |
| `f3probe` | Linux only (probes the raw device) | minutes | — |

If it is fake, re-partition to the real size so the stick stays usable: `diskutil partitionDisk diskN 2 MBR ExFAT NAME 30G "Free Space" none 0` (format, name, size per partition; `Free Space` leaves a gap, size `0` = fill).

## Reformat to the full size (destroys everything on the stick)

```
diskutil unmountDisk diskN
diskutil eraseDisk ExFAT NAME MBR diskN
```

No sudo. MBR + exFAT is readable on Windows, macOS, TVs and consoles; GPT is fine when only Windows 10/11 and macOS will see it. `diskutil listFilesystems` names the personalities (`ExFAT`, `MS-DOS FAT32`, …). Windows equivalent: `diskpart` → `list disk` → `select disk N` → `clean` → `create partition primary` → `format fs=exfat quick` → `assign` (Disk Management cannot do it: its FAT32 formatter stops at 32 GB and "Extend Volume" is NTFS-only).

**Why a stick shows one small partition:** the **Windows Media Creation Tool** formats any stick larger than 32 GB as one 32 GiB FAT32 partition labelled `ESD-USB` and leaves the rest unallocated; the Windows Format dialog then only re-formats that one volume, so the stick "won't format bigger". Sources: [oofhours.com — media tool with USB support](https://oofhours.com/2024/08/29/another-media-tool-release-this-one-with-usb-support/), [techygeekshome — bootable USB on large drives](https://techygeekshome.info/create-a-bootable-usb-flash-drive-using-large-capacity-drives/).

## ChromeOS Flex installer image (facts verified 2026-10-14)

Google's recovery list for Flex, the one the Chromebook Recovery Utility reads: <https://dl.google.com/dl/edgedl/chromeos/recovery/cloudready_recovery2.json> — five entries (STABLE, BETA, DEV, LTC, LTR) with `url` (a `.bin.zip`, ~1.37 GB), `filesize` (the unzipped `.bin`: 9,523,261,952 bytes = 9.5 GB, so the stick needs at least that much real capacity), `sha1`, `md5`, `version`, `chrome_version`. The documented shortcut <https://dl.google.com/chromeos-flex/images/latest.bin.zip> is the same stable image without a checksum. Write the `.bin` to the whole raw device (`sudo dd if=….bin of=/dev/rdiskN bs=4m status=progress`, macOS dd accepts `status=progress`) and verify by reading the written bytes back and comparing SHA-1.
