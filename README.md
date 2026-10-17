
# MTK FRP Bypass via mtkclient — Field Notes & Troubleshooting Guide

A practical, battle-tested walkthrough for removing Factory Reset Protection (FRP) from MediaTek-based Android devices using [mtkclient](https://github.com/bkerler/mtkclient) on Ubuntu Linux. Written from two real recovery sessions (an Infinix Hot 11 Play / MT6765 and a generic MT6570 whitelabel board), including every dead end and how it was resolved.

> ⚠️ **Use responsibly.** This process is intended for devices you own or have explicit authorization to service (your own phone after a factory reset, a device you purchased secondhand with proof of ownership, or client devices in a repair-shop context with documentation). FRP exists as an anti-theft measure — bypassing it on a device that isn't yours may be illegal in your jurisdiction. Always verify ownership before proceeding.

---

## Why Linux

Everything in this guide was done on Ubuntu, and that wasn't incidental — low-level MTK flashing tools behave meaningfully differently depending on the OS, and Linux is where tools like mtkclient are actually developed and tested first.

**Pros of doing this on Linux:**

- **Direct USB access.** mtkclient talks to the MediaTek BROM/preloader interface over raw USB (via `libusb`), and Linux exposes USB devices to userspace far more directly than Windows does. No driver signing fights, no proprietary MTK driver installers that half-work depending on Windows version.
- **No DA driver hell.** On Windows, getting SP Flash Tool or similar tools to even *see* a MediaTek device in BROM mode often means manually installing unsigned MediaTek USB VCOM drivers, disabling driver signature enforcement, and hoping the right one binds to the right mode (preloader vs. BROM vs. DA are sometimes three different device IDs). On Linux, `lsusb` either shows the `0e8d:` vendor ID or it doesn't — the feedback loop is immediate and honest.
- **Scriptable and transparent.** Every step above is a plain CLI command with visible, readable logs. You can see exactly which stage failed (handshake, DRAM setup, DA upload, extension init, partition lookup) rather than parsing an opaque GUI progress bar.
- **udev rules solve permissions permanently.** Once set, you never fight USB permissions again — compare this to Windows, where driver conflicts can resurface after any Windows Update.
- **Free and open tooling.** mtkclient is open-source and actively maintained; the Windows-centric alternative (SP Flash Tool) is closed-source, leaked/mirrored rather than officially distributed in places, and far less transparent about what it's doing to your device.

**Cons / genuine challenges:**

- **Documentation assumes comfort with a terminal.** There's no polished installer — you're installing Python dependencies, occasionally fighting a broken `sudo`, and reading raw exception tracebacks (as this session did, repeatedly) instead of clicking through a wizard.
- **udev/permission setup is a real first-run barrier.** Unlike Windows (where you fight drivers) or macOS (where this mostly doesn't work well at all due to USB restrictions), Linux's barrier is understanding *why* an unprivileged user can't claim a USB device by default, and that a one-time udev rule fixes it permanently. It's a small concept to learn, but it's a blocker until you do.
- **Distro/kernel quirks.** Things like `dmesg -w` requiring elevated permissions on newer kernels, or a dangling `/etc/alternatives/sudo` symlink after a broken package upgrade, are genuinely distro-specific annoyances that have nothing to do with MTK flashing itself but still cost real troubleshooting time.
- **Preloader/DA coverage is community-sourced.** mtkclient's bundled preloader library (hundreds of `.bin` files, as seen above) is comprehensive but crowd-assembled — obscure whitelabel boards sometimes require trial-and-error matching, as happened with the MT6570 board in this guide.
- **No official GUI equivalent on Linux** to SP Flash Tool's point-and-click scatter-file flashing — mtkclient's GUI exists but the CLI is the primary, best-supported interface, so some Windows-native users have a steeper on-ramp.

**Net take:** Linux is the better platform for this *specifically because* it gets out of the way — the challenges are front-loaded into setup, not scattered throughout every flashing session the way Windows driver conflicts tend to be.

---

## What This Is Actually For

Tools like mtkclient exist in a gray zone defined entirely by *who's using them and why* — the technical capability (direct, low-level chip access that bypasses OS-level locks) is identical whether the use is legitimate or not. It's worth being explicit about where this sits.

**Legitimate, well-established uses:**

- **Recovering your own device** after you forgot credentials tied to an account you no longer control, following a factory reset you performed yourself.
- **Right-to-repair and secondhand device refurbishment** — repair shops and resellers routinely receive devices with FRP still active from a previous owner, verified via purchase receipt, IMEI records, or a bill of sale, and need to clear it before resale.
- **Bricked-device recovery** — BROM-level tools are often the *only* way back from a corrupted bootloader, failed OTA update, or bad root/custom-ROM attempt, regardless of FRP being involved at all.
- **Security research and firmware analysis** — understanding how secure boot (SBC/SLA/DAA) is implemented on a given chip, which is exactly what you saw play out in this session: some boards ship genuinely unprotected (`SBC enabled: False`), which is itself a meaningful security research finding about that device's manufacturing/QA process.
- **E-waste reduction** — a huge fraction of "dead" secondhand phones sold cheaply or discarded are perfectly functional hardware locked out by a forgotten account; tools like this keep usable hardware in circulation instead of in a landfill.

**Where it gets legally and ethically murky:**

- Using these exact same steps on a device that was lost or stolen from someone else, to erase evidence of the rightful owner's account, is unauthorized access to a device you don't own — often a specific crime (in the US, this overlaps with state computer-crime statutes; many countries have similar device-tampering or trafficking-in-stolen-goods laws) regardless of the method used.
- A legitimate repair shop's practice is to document chain of ownership *before* doing an FRP removal, precisely because the tooling can't tell the difference between "I forgot my password" and "I found/stole this phone."

**The honest framing:** mtkclient and tools like it are infrastructure — the same BROM-level access that lets you rescue a bricked device you own is indistinguishable, at the protocol level, from what it'd take to strip someone else's phone. The ethical weight sits entirely on provenance and consent, not on the tool. Any serious writeup or repo covering this should say that plainly rather than pretend the tool is single-purpose.

---

## Table of Contents

- [Why Linux](#why-linux)
- [What This Is Actually For](#what-this-is-actually-for)
- [What is BROM mode?](#what-is-brom-mode)
- [Prerequisites](#prerequisites)
- [Installation (Ubuntu)](#installation-ubuntu)
- [Fixing a broken `sudo`](#fixing-a-broken-sudo)
- [Entering BROM Mode](#entering-brom-mode)
- [Basic FRP Erase (chips with RAM-dumpable preloader)](#basic-frp-erase)
- [Advanced: Boards Requiring a Manual Preloader](#advanced-boards-requiring-a-manual-preloader)
- [Common Errors & Fixes](#common-errors--fixes)
- [Finding the Right Partition Name](#finding-the-right-partition-name)
- [Permissions: Removing the `sudo`/`pkexec` Requirement](#permissions-removing-the-sudopkexec-requirement)
- [Device Notes](#device-notes)
- [Disclaimer](#disclaimer)

---

## What is BROM mode?

BROM (Boot ROM) is a small, immutable bootloader burned directly into the MediaTek SoC itself — it exists outside of flash storage, so it's reachable even on a device with corrupted, wiped, or FRP-locked storage. Flashing tools talk to BROM over USB to upload a Download Agent (DA), which then has low-level read/write access to the eMMC/UFS storage.

**Key facts:**
- The phone's screen stays **completely black** in BROM mode — this is normal, not a hang.
- BROM mode typically times out after **30–60 seconds** if nothing establishes a handshake.
- ADB is irrelevant here — FRP-locked devices block ADB, but BROM operates below the OS entirely.

---

## Prerequisites

- Ubuntu (or any Linux distro) with a working `python3` and `pip3`
- A **data-capable** USB cable (many "charge-only" cables will silently fail every step below)
- A USB 2.0 port preferred — USB 3.0 ports occasionally drop MTK devices mid-handshake
- Battery above ~30% — BROM/DA sessions drain surprisingly fast, and a flat battery looks identical to "no detection" (see [Common Errors](#common-errors--fixes))

---

## Installation (Ubuntu)

```bash
sudo apt update
sudo apt install python3-pip python3-usb libusb-1.0-0 git

git clone https://github.com/bkerler/mtkclient
cd mtkclient
pip3 install -r requirements.txt --user --break-system-packages
```

> `--break-system-packages` is required on newer Debian/Ubuntu releases due to [PEP 668](https://peps.python.org/pep-0668/).

If the install stalls on a large optional dependency (commonly `pyside6_addons`, which is only needed for mtkclient's GUI), install just the CLI essentials instead:

```bash
pip3 install colorama pyusb pycryptodome --user --break-system-packages
```

---

## Fixing a broken `sudo`

If `sudo` reports `command not found` despite `which sudo` resolving correctly, your distro's `update-alternatives` symlink may be dangling (pointing at a non-existent binary):

```bash
ls -l /etc/alternatives/sudo   # check where the symlink actually points
```

**Fix, from a root shell:**
```bash
su -
apt install --reinstall sudo
```

**If the root account has no password set** (common on stock Ubuntu Desktop), use `pkexec` instead — it provides a GUI privilege prompt without needing a root password:

```bash
pkexec python3 mtk.py printgpt
```

As a last resort, boot into recovery mode (hold `Shift`/`Esc` at boot → *Advanced options* → *recovery* → root shell → `mount -o remount,rw /`) to set a root password with `passwd`.

---

## Entering BROM Mode

1. **Power the phone fully off.** If it's stuck on a "no command" or similar screen, hold power ~15 seconds until it's dead. Tap power once afterward to confirm nothing happens.
2. **Hold Vol+ and Vol− together.**
3. **Plug the USB cable into the laptop while still holding both buttons.**
4. Hold for ~3–5 seconds after connecting, then release.

**Verify the connection landed in BROM mode** with a polling loop (catches the brief window reliably):

```bash
while true; do lsusb | grep -i 0e8d && { echo "BROM DETECTED"; break; }; sleep 1; done
```

A new line like:
```
Bus 001 Device 012: ID 0e8d:20ff MediaTek Inc.
```
confirms BROM mode (vendor ID `0e8d` = MediaTek). If nothing appears after ~60 seconds, try:
- Holding **Vol− only** (some boards don't respond to both buttons)
- A different USB port (prefer black/USB 2.0 over blue/USB 3.0)
- A different cable (rule out charge-only cables)

---

## Basic FRP Erase

For chips where mtkclient can dump the preloader directly from device RAM (no external file needed) — this covers most modern MediaTek chips with security features enabled, e.g. MT6765/Helio G35:

```bash
cd ~/mtkclient
python3 mtk.py e frp
```

Expected successful output ends with:
```
DAXFlash - Formatting addr 0x1888000 with length 0x100000, please standby....
Done |██████████| 100.0% Erasing: (0x100000/0x100000)
DAXFlash - Successsfully formatted addr 0x1888000 with length 1048576.
```

Then reboot:
```bash
python3 mtk.py reset
```
(or unplug and hold power ~5–10 seconds if the session already dropped connection)

If the device still prompts for the previous Google account after reboot, do a full data wipe instead:
```bash
python3 mtk.py e userdata
```
This only touches data partitions — `seccfg`, `nvram`, and IMEI storage are untouched, so baseband/IMEI survive intact.

---

## Advanced: Boards Requiring a Manual Preloader

Older or whitelabel/rebrand boards (e.g. generic MT6570 devices with no RAM-dumpable preloader) will fail with:

```
DAXFlash - [LIB]: No emmc info, can't parse existing preloaders.
DAXFlash - [LIB]: No preloader given. Operation may fail due to missing dram setup.
```

This means mtkclient needs a **matching preloader binary** to correctly configure DRAM timing before it can proceed. mtkclient ships a large library of known preloaders:

```bash
ls ~/mtkclient/mtkclient/Loader/Preloader/ | grep -i <your_chip>
```

If your exact board isn't listed, whitelabel/ODM boards often share a reference design with named "generic" preloaders (e.g. `preloader_fise6570_ctlc_m.bin`, `preloader_magc6570_cweg_m.bin`, `preloader_sr6570_cweg_m.bin` for MT6570). **Try each candidate in turn** — a mismatched preloader fails cleanly rather than causing damage:

```bash
python3 mtk.py e frp --preloader "/path/to/preloader_candidate.bin"
```

**Signs you picked the right file:**
- `DRAM setup passed` appears (instead of looping on "No emmc info")
- The EMMC ID/CID printed are real values (not `0x0`)
- DA stage 2 boots successfully

If stage 2 upload fails with a `USBError(5, 'Input/Output Error')` even with the correct preloader, this is usually a **flaky USB connection**, not a DRAM problem — retry 2–3 times and/or swap the USB port or cable before concluding the preloader is wrong.

### Skipping custom DA extensions

Some generic preloaders get far enough to boot the DA fully (SLA disabled, EMMC correctly identified) but then fail specifically at:
```
DAXFlash - [LIB]: DA Extensions failed to enable
```
mtkclient's extensions add convenience features on top of the base Download Agent; they aren't always required for a basic erase. Fall back to MediaTek's unmodified **stock DA** to skip them:

```bash
python3 mtk.py e frp --preloader "/path/to/preloader.bin" --stock
```

---

## Common Errors & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| `sudo: command not found` | Broken `update-alternatives` symlink | See [Fixing a broken sudo](#fixing-a-broken-sudo) |
| `ModuleNotFoundError: No module named 'colorama'` | Incomplete `pip install` | `pip3 install colorama pyusb pycryptodome --user --break-system-packages` |
| `Handshake failed, retrying...` | Device attached in **preloader mode**, not BROM | Fully power off (15s hold), then hold Vol+/Vol− *before and during* plug-in |
| `lsusb` shows nothing after button combo | Not fully powered off, bad cable, or wrong port | Confirm fully off; try Vol− only; swap cable/port |
| No charging icon, no vibration, no detection at all | **Deeply discharged battery** — common after a long flashing session | Wall-charge (not via laptop) for 15–30 min before retrying |
| `No emmc info, can't parse existing preloaders` (looping) | Chip needs a manually supplied preloader | See [Advanced section](#advanced-boards-requiring-a-manual-preloader) |
| `USBError(5, 'Input/Output Error')` at stage 2 upload | Flaky USB connection | Retry; swap port/cable |
| `DA Extensions failed to enable` | Generic/mismatched preloader doesn't support mtkclient's extensions | Add `--stock` flag |
| `Error: Couldn't detect partition: frp` | This board stores FRP state under a different partition name | See [Finding the Right Partition Name](#finding-the-right-partition-name) |
| `USBError(16, 'Resource busy')` | A kernel driver (e.g. `cdc_acm`) grabbed the port first | `sudo rmmod cdc_acm` before retrying |
| Two terminals both running `mtk.py` | Competing processes fighting over the USB device | `pkill -f mtk.py` before every fresh attempt |

---

## Finding the Right Partition Name

On older Android builds (6.0 and earlier) and many non-reference-design boards, FRP state isn't always stored in a partition literally named `frp`. Dump the full GPT instead of guessing:

```bash
python3 mtk.py printgpt --preloader "/path/to/preloader.bin" --stock
```

Common alternate names to check for in the output:
- `nvram`
- `protect1` / `protect2` (or `protect_f` / `protect_s`)
- `nvdata`
- `para`

Once identified, erase it directly:
```bash
python3 mtk.py e <partition_name> --preloader "/path/to/preloader.bin" --stock
```

---

## Permissions: Removing the `sudo`/`pkexec` Requirement

Running mtkclient as root every time is annoying and can occasionally interfere with USB device claiming. A udev rule grants your normal user direct access to MediaTek USB devices permanently:

```bash
pkexec bash -c 'echo "SUBSYSTEM==\"usb\", ATTR{idVendor}==\"0e8d\", MODE=\"0666\"" > /etc/udev/rules.d/99-mtk.rules && udevadm control --reload && udevadm trigger'
```

After this, run mtkclient without `sudo` or `pkexec` at all:
```bash
python3 mtk.py printgpt
```

---

## Device Notes

### Infinix Hot 11 Play (X688B) — MT6765 / Helio G35
- Preloader dumps cleanly from device RAM; no manual preloader file needed.
- SLA/DAA auth bypassed automatically by mtkclient's built-in exploit chain.
- `frp` is a standard 1 MB partition at a fixed GPT offset.
- Clean `e frp` erase, confirmed working.

### Generic MT6570 / MT8321 whitelabel board ("VIWA-LIGHT MINI", Android 6.0)
- No RAM-dumpable preloader — **required** a manually supplied preloader file.
- `SBC enabled: False` — this board ships genuinely unprotected, which is why mtkclient logs *"Unprotected device, we assume we can patch directly!"*
- Correct preloader identified by trial: `preloader_magc6570_cweg_m.bin` (from mtkclient's bundled generic ODM preloader set) — confirmed by real EMMC ID/CID values appearing in output, versus placeholder `0x0` values from mismatched candidates.
- Required `--stock` to get past a `DA Extensions failed to enable` error.
- FRP partition name not yet confirmed as `frp` on this board — investigation ongoing via `printgpt`.

---

## Disclaimer

This guide documents a technical process using the open-source [mtkclient](https://github.com/bkerler/mtkclient) project. It is provided for legitimate device-recovery purposes only — unlocking devices you own, or devices you have explicit written authorization to service. The author(s) of this guide and of mtkclient are not responsible for misuse. Always confirm proof of ownership before performing FRP removal on any device that isn't unambiguously yours.
