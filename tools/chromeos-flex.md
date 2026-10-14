# ChromeOS Flex — installer, desktop boot, first-run gotchas

Verified 2026-10-14 on an older desktop with integrated graphics plus a discrete card, booted from a USB 2.0 stick; sources are Google's Flex help pages unless marked "observed".

## Installer image
- Google's recovery list (what the Chromebook Recovery Utility reads): <https://dl.google.com/dl/edgedl/chromeos/recovery/cloudready_recovery2.json> — entries STABLE / BETA / DEV / LTC / LTR with `url` (a `.bin.zip`, ~1.37 GB), `filesize` (the unzipped `.bin`: 9,523,261,952 bytes), `sha1`, `md5`, `version`, `chrome_version`. The documented shortcut <https://dl.google.com/chromeos-flex/images/latest.bin.zip> is the same stable image without a checksum.
- Write the `.bin` to the whole stick and verify: `utilities/usb-image-write/write_image_usb.sh` (macOS); timings and the full procedure in `tools/macos-usb-drives.md`. The Chromebook Recovery Utility on Windows did not list a no-brand stick (2026-10-13); the direct-image route needs nothing from the utility.
- The stick needs at least 9.5 GB of real capacity; Google says 8 GB or more and warns that some SanDisk and other sticks fail as installers (reformat, or use another brand).

## Booting the installer on a desktop — the ladder that worked (observed 2026-10-14)

| Symptom | Cause | Fix |
|---|---|---|
| Black screen with the Chrome logo and "chromeOS Flex", minutes past the normal 1–3 min from a USB 2.0 stick | the TV was on the **discrete graphics card**; Google known issue: desktops with integrated Intel graphics plus a card in a slot may not output video from the card's ports; Nvidia is "generally unstable" | move the HDMI to the **motherboard's** port (or pull the card) |
| Cloudy white screen, mouse cursor visible, the cursor can leave the picture although the signal is 1080p | ChromeOS extended the desktop onto a **phantom second display** (a VGA or DVI-A port with nothing attached — analog ports have no plug detection — or the still-installed card's output) and put the first-run dialog there; the real screen shows the setup wallpaper | **Ctrl + F4** (= Ctrl + Fullscreen on a PC keyboard) toggles mirror mode and the welcome page appears; Alt + F4 swaps the primary display; Ctrl + Alt + Z (ChromeVox) reads the page aloud if it exists off-screen |
| Blank screen or blinking cursor right after the boot menu | unsupported device (Intel Gen 3-and-older GMA 900/950/3100/3150 and PowerVR GMA 500/600/3600/3650 will not boot Flex), a bad USB port or dongle, or a bad stick | Google's troubleshoot page: certified list, another USB port, recreate the stick |

Boot keys: F12 Dell and Lenovo, F9 HP, F2 Acer and Intel, Del Asus, F2 or F12 Toshiba, otherwise Esc, F1–F12 or Enter; choose the `UEFI:` entry when the stick appears twice ("1962: No operating system found" → recreate the stick and pick the UEFI entry). Storage must be AHCI, not RAID. Google recommends Secure Boot on.

## Installing
- Disconnect the Windows drive and other drives first: the installer **picks the target itself** (it skips the stick, tries to skip removable media and takes an internal drive larger than about 16 GB); there is no disk picker. To check which drive is attached, boot without the stick: if Windows comes up, the wrong drive is connected.
- "Try it first" runs from the stick (slow, no updates, nothing carries over); "Install ChromeOS Flex" erases the target; the PC powers itself off when done — remove the stick, power on, reconnect the other drives, set the boot order.
- Expect the mirror toggle from the live session not to carry over: press Ctrl + F4 again on the first boot after installing, then make it stick in Settings → Device → Displays (or remove the card, or disable the phantom port in the BIOS).
- The first Google account to sign in becomes the owner (only a reinstall changes it): the parent first, then Add person → For a child, then turn off guest browsing and restrict sign-in (Settings → Privacy and security → Manage other people).

Sources: Google Flex help — [Create the USB installer](https://support.google.com/chromeosflex/answer/11541904), [Boot device using the USB installer](https://support.google.com/chromeosflex/answer/11552352), [Known issues](https://support.google.com/chromeosflex/answer/11541806), [Troubleshoot installation failures](https://support.google.com/chromeosflex/answer/11542398); the mirror shortcut: [FCIT helpdesk](https://fcit.freshservice.com/support/solutions/articles/4000049180).
