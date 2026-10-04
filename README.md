# Reviving the Surfsight / Lytx / GEO AI-12 Fleet Dashcam

**Turning an e-waste fleet dashcam into a cloud-free local recorder: a reverse engineering log**

## Background

I got a Surfsight AI-12 (also sold as the Lytx or GEO AI-12) out of an e-waste bin. It is a commercial dual-camera fleet dashcam with a lot of hardware inside: a Qualcomm SDM450 chip, LTE, GPS, two cameras, and on-device AI. It is built to report everything to a company's cloud service.

I had two goals: turn it into a recorder that I own and that works without the cloud, and find out why its LTE modem did not work at all. This document is my reverse engineering log. If you are trying to root, unlock or revive one of these units, it should save you time.

> **My test units**
> - **Unit 1 (`[UNIT1-SERIAL-REDACTED]`)**: used for the first teardown and EDL tests. I bricked it while writing `aboot`. It is now stuck in Qualcomm crash-dump mode, but it provided all of the hardware and EDL findings.
> - **Unit 2 (`[UNIT2-SERIAL-REDACTED]`)**: working and rooted. All of the live modem, DIAG and RFC tests were done on this one.

**More detail:** close-up photos of the chip, power chip, RAM, antennas and connectors are in [HARDWARE_PHOTOS.md](HARDWARE_PHOTOS.md). Lower-level notes (partition maps, dead ends, Sahara/EDL analysis) are in [TECHNICAL_DEEP_DIVE.md](TECHNICAL_DEEP_DIVE.md).

---

## How sure each finding is

Every technical claim in this document has one of these tags:

- **[CONFIRMED]**: seen directly and repeated on hardware I own.
- **[STRONG]**: backed by several separate observations, but not one deciding test.
- **[THEORY]**: an idea that fits the evidence but is not proven.
- **[REJECTED]**: an idea that was tested and shown to be wrong.
- **[UNKNOWN]**: not worked out. The text says what is known, what is not, and why.

A tag applies to the sentence or row it is attached to, not to a whole section.

---

## Power layout and the hidden battery

This is the most important section. **Do not rely on normal Android reboot commands to enter EDL mode.**

I could not get this device into EDL mode reliably until I opened it and found how it is powered. It has **four** separate power sources:

1. **5V/2A pin**: powers the main chip and the full Android boot.
2. **USB-C**: *only* powers the EDL/PBL USB ROM logic (9008 mode). It does **not** power the main Android processor.
3. **Internal LiPo battery (~300 mAh)**: a small white 3-pin JST battery on the back of the main board. It keeps the IMEM/SMEM memory powered.
4. **Supercapacitor**: keeps the device running for about 4 seconds after external power is cut.

**Why this matters:** when you run `adb reboot edl`, the SBL writes an EDL flag to IMEM and reboots. Because the internal battery keeps IMEM powered, that flag survives unplugging the cables and normal reboots. **To do a true cold boot and clear the EDL flag, you have to unplug the internal LiPo battery.**

There is also no clock battery (RTC coin cell). Every time the device loses all power, the clock resets to 2009-01-01. This causes a serious problem, described below.

---

## Hardware and software overview

### Specs `[CONFIRMED]`

- **Chip**: Qualcomm SDM450 (MSM8953), with a PMI632 power chip.
- **RAM and storage**: 2GB LPDDR3 and **16GB eMMC** `[CONFIRMED]` (Samsung KMQE60013B eMCP, which reports itself as `QE63BB`, manfid `0x15` = Samsung; confirmed with `blockdev --getsize64 /dev/block/mmcblk0` = 15,634,268,160 bytes). Recordings go on a **removable microSD card of up to 128GB** (confirmed with `mmcblk1` = 127,865,454,592 bytes). That card is where the "128GB" on the retail label comes from, not the internal storage.
- **OS**: Android 9 (Pie), AOSP with no Google services. Build `3.9.209`.
- **Cameras**: 2 feeds (road and cabin) through the standard Android Camera2 HAL. The app (`surfdash2`) contains RTSP/ONVIF client code, but the cabin camera is a normal CSI sensor, not an IP camera.
- **Bootloader**: unlocked (`ro.boot.verifiedbootstate=orange`), but there is no real `fastboot`. `adb reboot bootloader` only opens a screen inside the Android kernel. **EDL is the only low-level way in.**

### Rooting and system-as-root

The device uses legacy system-as-root (SAR). The system partition is mounted directly as `/`, and the kernel is built with `skip_initramfs`. I rooted Unit 2 with **Magisk, using the LEGACYSAR method**.

> **Note on `su -c`:** if you chain commands like `su -c "cmd1 && cmd2"`, only `cmd1` runs with Magisk's `u:r:magisk:s0` context. `cmd2` falls back to `u:r:shell:s0` and you get confusing "permission denied" errors. Use a separate `su -c` for each command.

---

## Investigating the LTE modem

Both units had the same problem. The SIM is detected and airplane mode is off, but the modem reports `OUT_OF_SERVICE`. It keeps searching, finds no cell towers on 2G, 3G or 4G, and the signal stays at the `-120 dBm` noise floor.

I wanted to find out whether this was a software setting or broken hardware. These are the causes I ruled out:

### 1. Carrier settings or band lock `[REJECTED]`

I suspected the modem was locked to US carrier bands. It is not. The active `mcfg_sw` is `ROW_Commercial` (the generic worldwide setting). Also, a band lock would not block 2G and 3G at the same time.

### 2. Corrupt NV settings or calibration `[REJECTED]`

I copied the modem's EFS file system through DIAG. The calibration folders (`/rfc/0087/selfcal/`) are complete. To make sure my DIAG NV reads were correct, I read NV item 550, which decoded to the device's IMEI. Nothing is corrupt.

### 3. Testing every RFC card `[STRONG]`

On the MSM8953, the modem reads a hardware ID (NV item 1878) to pick an "RFC card". The RFC card describes how the radio is wired: antenna switches, signal paths and transceiver ports.

- My NV 1878 was set to **87**.
- In `modem.b13` I found that the firmware only includes 8 RFC cards: `{75, 87, 157, 191, 218, 229, 232, 249}`.

**The test:** I wrote a script that set NV 1878 to each of the 8 cards in turn and rebooted.

- **Result:** with every card except 87, the radio stayed `POWER_OFF`. **RFC 87 was the only card that let the radio turn on and search for a network.**
- **Conclusion:** the modem is set up correctly. RFC 87 is the right card for this board, so the expected radio hardware is present. The fault is in the analog side: either the radio front-end chips are dead, or the antenna is damaged or detuned. Software testing can not go further than this.

---

## DIAG, FTM, and why the radio can not be monitored

I wanted live RxAGC/RSSI readings to prove the radio was receiving nothing. That turned out to be blocked in several ways.

1. **Log streaming is blocked `[CONFIRMED]`**: the kernel driver on this build has the logging IOCTLs removed. Qualcomm's own `diag_mdlog` fails with `errno: 22`. QCSuper connects but returns no GSMTAP frames. So "no log packets" does not prove the radio is silent; the logging path is simply broken.
2. **The factory tools do not measure anything `[CONFIRMED]`**: the device ships with `FactoryKit.apk` and `fastmmi`. I decompiled them looking for engineering radio menus. The "RF_Cal" screens only read pass/fail NV flags, and `ftm_test_config` only contains audio loopback tests. This firmware has no way to take live cellular measurements.
3. **FTM command formats are not known `[UNKNOWN]`**: FTM (Factory Test Mode) commands do work over DIAG, but the formats for radio measurements are only in Qualcomm's private modem source. Guessing them could turn on the transmitter or erase calibration data, so I stopped here.

**How to connect to DIAG:**
The USB DIAG interface is only available in FFBM (Factory Boot Mode). In normal Android, use **QCSuper** through `adb shell` and `su`.

On Windows, QCSuper's USB auto-detect crashes or picks the wrong device. Disable it in Python so it uses the ADB connection instead:

```python
from qcsuper.inputs import usb_modem_pyusb_devfinder as devfinder
class _NF: not_found_reason = devfinder.PyusbDevNotFoundReason.auto_criteria_did_not_match
devfinder.PyusbDevInterface.auto_find = staticmethod(lambda: _NF())
```

---

## Unit 1: the brick and what EDL showed

I bricked Unit 1 while flashing `aboot`. It now switches between two Qualcomm EDL modes (9008 and 900E). If you brick yours, this is what to expect:

- **PID 9008 (Firehose)**: USB connects, but Sahara does not respond and the bulk endpoints time out. The SBL started USB but stopped before the Sahara handler began.
- **PID 900E (memory debug / RAM dump)**: Sahara does respond. You can copy the 2GB RAM dump, which is how I rebuilt the boot chain and showed that it fails at the ABOOT secure-boot stage. **However**, it refuses to accept a Firehose programmer and returns `0x09 INVALID_IMAGE_TYPE` before any data is sent.
- **The "deep flash" cable method does not work `[REJECTED]`**: the usual Qualcomm recovery trick is to short USB D+ to ground. I tried it. Because USB-C does not power the main processor on this board, the PBL decides how to boot without looking at the USB data lines. Shorting D+ only hides the device from the PC. To force a clean 9008 EDL you need the eMMC DAT0 test point, which I have not found yet.

---

## Bugs and warnings

### Recordings are deleted after the device wakes up `[CONFIRMED]`

This is a serious flaw in the `surfdash2` app combined with the hardware.
When the car is parked, the device goes to sleep (triggered by the Bosch BMI160 motion sensor). When it wakes up, it does a **full reboot**, not a light resume.
Because there is no clock battery, the clock resets to 2009. When `surfdash2` starts and sees the wrong date, it resets its `Movies` folder and **deletes recordings from the SD card that were not uploaded yet, without any warning**. I saw about 4GB of recordings drop to about 400MB after one wake-up.

**Workaround:** remove the SD card, or copy your recordings off through the busybox httpd server, before any reboot or sleep testing.

### A factory "enable root" app left in the final firmware `[CONFIRMED]`

I found this while looking into an unfamiliar system app with a Chinese label ("关机界面" / `fun.qucii.com.quciipowerkey`, a harmless power-button and shutdown screen app). Next to it was `com.example.logtest`, a factory testing tool from the same component maker, "Qucii". It is still in the shipped `/system/app` image with `sharedUserId="android.uid.system"`, which gives it full system privileges.

It has three parts:

- **`RootActivity`**: shows an "Enable Root" dialog. Pressing confirm calls `SystemProperties.set("persist.sys.force.root", "1")`, the standard Qualcomm reference-design way of enabling root on a `user` build. It has no `android:exported="false"` and uses `targetSdkVersion=23`, so the screen is exported by default. Any other app can open it, and so can `adb shell am start -a android.intent.action.QUCII_ROOT_ENABLE`, not just the launcher icon.
- **`LogTestService`**: a full engineering logging panel, with switches for main, system, radio, event, kmsg, camera, dumpsys and PMIC logs, AP, modem and USB debug switches, and start/stop control of the programs `/system/bin/quciilog` and `/vendor/bin/diag_mdlog`.
- **`LogBootCompletedReceiver`**: starts automatically at boot. `RECEIVE_BOOT_COMPLETED` is the only permission it asks for. It has no `INTERNET` permission, so it can not send data anywhere.

It does not send data out, so it is not spyware. But it is a real way for a local app to gain extra privileges, and it should have been removed before the firmware shipped. It is separate from the `surfdash2` backdoor that I found and removed during my own firmware changes: that one was in the device maker's app, while this one is factory tooling one layer below it. I disabled it on Unit 2 with `pm disable-user com.example.logtest`, which can be undone, rather than uninstalling it.

---

## Partition map and useful paths

For anyone writing scripts or making backups, this is the single-slot layout (`mmcblk0`):

- `modem`=p1, `sbl1`=p4, `tz`=p8, `aboot`=p19, `boot`=p25, `system`=p28, `vendor`=p29, `userdata`=p54.
- **Warning:** `/vendor` is protected by dm-verity (`dm-0`). Do not write to it directly.
- **Modem firmware:** `/vendor/firmware_mnt/image/` (read-only vfat).
- **Modem EFS:** only reachable through DIAG subsystem 19 (`DIAG_SUBSYS_FS`), not through the Android file system.

---

## What is left to do

If you want to continue this work, these are the open tasks:

1. **Test with an external antenna:** the software is fine, so the radio front-end is the problem. The deciding test is to plug a cheap U.FL LTE antenna straight into the board's cellular connector. If a signal appears, the internal antenna is broken. If not, the board's radio chips are dead.
2. **Find the EDL test point:** I need the eMMC DAT0 pad on the board to force Unit 1 into a clean 9008 mode and try a Firehose restore.

If you map the EDL test points or find the FTM RxAGC command formats, please open an issue or a pull request.

---

## License

The text and photos in this repo are licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE). You can share and adapt them for any purpose, as long as you give credit and link back to this repo.

---

*Last updated: July 2026. Remember to unplug the internal battery before you flash.*
