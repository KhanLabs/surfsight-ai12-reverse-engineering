
# Technical Deep Dive & Engineering Notes
**The raw data, hex dumps, dead ends, and platform quirks of the AI-12 Reverse Engineering Project.**

If you read the main `README.md`, you know the high-level story and the major gotchas. This document is the unfiltered engineering log. It contains the deep memory dumps, the graveyard of hypotheses I tested and ruled out, the exact hex structures of the modem files, and the annoying platform quirks that cost me hours of debugging. 

If you're trying to replicate my work or dig deeper into the Qualcomm MSM8953 modem on this board, you'll need this.

---

## 1. Unit 1 Brick Forensics (Sahara & EDL Deep Dive)
When I bricked Unit 1 (`[UNIT1-SERIAL-REDACTED]`) via a bad `aboot` flash, it became my dedicated EDL test subject. It constantly cycles between two USB states. If you brick yours, here is exactly what to expect from the Qualcomm Sahara protocol.

### PID 9008 (Firehose Mode) = Silent Death
When it hits 9008, USB enumerates, but the Sahara bulk endpoints (EP 0x81 in, EP 0x01 out) are completely dead. 
* I tested every driver: WinUSB, libusbK (which crashes with `0xC0000005` on libusb 1.0.24), UsbDk, and native serial. 
* I even passed it through WSL2 via USB/IP at 480M High-Speed. 
* **Result:** Total timeout. The SBL initialized the USB hardware, but hung before the Sahara handler actually started. Transport isn't the issue; the device-side eDload path is just dead.

### PID 900E (Memory Debug / SDI Ramdump) = A Read-Only Purgatory
Sometimes it boots into 900E. Sahara v2 *does* respond here, but it's heavily restricted:
* **Serial Num Read:** Works (returns the chip serial, redacted here).
* **Memory Debug Dump:** Works. I successfully dumped the 2GB DDR (DDRCS0). 
* **Secure Boot Fuses (QFPROM):** Blocked. Returns `NAK 0x19 INVALID_MEMORY_READ`. I can't tell if the secure boot fuses are blown, which means I don't know if a generic Firehose loader would ever be accepted.
* **Firehose Loader Injection:** Blocked. When I try to force `SWITCH_MODE` to offer the Xiaomi generic Firehose loader, it immediately rejects it with `END_IMAGE status=0x09 INVALID_IMAGE_TYPE` before accepting a single byte. This is a hard policy block, not a cryptographic signature rejection (which would be `0x21` or `0x25`).

### The DDR Ramdump Findings
Dumping the 2GB RAM via 900E let me reconstruct the exact boot chain failure:
* **PBL executed:** Found `boot_pbl_v1.c` and "PBL, Start/Delta/End" timing markers.
* **SBL1 executed:** Found `sbl1_efs_handle_cookies` and `boot_sd_ramdump.c`.
* **TrustZone/QSEE launched:** Found a valid ARM32 ELF in DDR (entry `0x8f600000`) near genuine TZBSP strings (`tzbsp_psci_cpu_boot_notifier`, `/qsee/img_auth`).
* **The Failure:** There was **zero** `ANDROID!` magic anywhere in the 2GB dump. `boot.img` was never loaded into RAM. Execution failed precisely at the ABOOT (Little Kernel) stage due to a secure-boot integrity check failure.

---

## 2. The Full 54-Partition Map
The AI-12 uses a single-slot layout (no A/B seamless updates). Here is the exact block map for `mmcblk0` if you are writing backup scripts or doing raw `dd` restores.

```text
modem=p1       fsc=p2        ssd=p3        sbl1=p4       sbl1bak=p5    rpm=p6
rpmbak=p7      tz=p8         tzbak=p9      devcfg=p10    devcfgbak=p11 dsp=p12
modemst1=p13   modemst2=p14  DDR=p15       fsg=p16       sec=p17       splash=p18
aboot=p19      abootbak=p20  dtbo=p21      dtbobak=p22   vbmeta=p23    vbmetabak=p24
boot=p25       recovery=p26  devinfo=p27   system=p28    vendor=p29    cache=p30
persist=p31    misc=p32      keystore=p33  config=p34    oem=p35       limits=p36
mota=p37       dip=p38       mdtp=p39      syscfg=p40    mcfg=p41      lksecapp=p42
lksecappbak=p43 cmnlib=p44   cmnlibbak=p45 cmnlib64=p46  cmnlib64bak=p47 keymaster=p48
keymasterbak=p49 apdp=p50    msadp=p51     dpo=p52       logdump=p53   userdata=p54
```
*Note: External SD is `mmcblk1`. Do not write directly to `/vendor` (p29) from Android, it is dm-verity protected (`dm-0`).*

---

## 3. Deep Hex & Memory Structures

### Reverse Engineering `rffe_scan.dat`
The modem writes a log to the EFS at `/rfc/rffe_scan/rffe_scan.dat` on every boot, detailing the RF-front-end (RFFE) bus enumeration. I dumped this and reverse-engineered the structure. It is exactly **204 bytes (51 words)**, little-endian uint32.

**Header (7 words):**
* `word0 = 2` (format/version)
* `word1 = 7` (header length in words)
* `word2 = 11` (record size in words)
* `word3 = 87` (The active RFC Card ID!)
* `word4 = 4` (record count)
* `word5 = 4` (record count repeat)
* `word6 = 0` (reserved)

**Records (4 × 11 words):**
* *Retraction Note:* Early on, I saw `0xFFFFFFFF` in the "reserved" field of the records and theorized it meant "failed enumeration". **I was wrong.** That field is constant across *every* record. A real per-device status would vary. It's just an unused/reserved slot.
* *The Anomaly:* Record index 1 has a triple `15, 15, 10`. The other three records have matching triples (e.g., `1,1,1` or `14,14,14`). I still don't know what `15, 15, 10` means (min/target/max? Tx/Rx/diversity?) without Qualcomm's proprietary field spec.

### Modem Firmware (`modem.b13`)
I ran `strings` on the Hexagon QDSP6 code segment `modem.b13` and found the exact RFC symbol table. The firmware *only* compiles 8 cards:
`rfc_75`, `rfc_87`, `rfc_157`, `rfc_191`, `rfc_218`, `rfc_229`, `rfc_232`, `rfc_249`.
It also references `rfc_vreg_mgr_wtr1605_sv.cpp`. While `wtr1605` is a known Qualcomm transceiver, that string is common across generic 8953 builds, so I can't 100% confirm the physical silicon part number without tearing the RF shields off the board.

---

## 4. Dead Ends
I went down a lot of rabbit holes trying to find a software fix or a live RF diagnostic path. Here is what I tested so you don't have to:

1. **FactoryKit & `mmi_telephone.so`**: I decompiled `FactoryKit.apk` hoping for hidden engineering menus. The "RF_Cal" screens just read NV pass/fail flags. The `mmi.xml` config references a `TELEPHONE` module for dialing 112, but the backing `mmi_telephone.so` library doesn't even exist on the filesystem. It's a dead reference from Qualcomm's generic template.
2. **`PktRspTest`**: Found this binary in `/vendor/bin` and got excited. Turns out it's just Qualcomm's canonical DIAG "Hello World" sample. It has no arguments; it just initializes DIAG, sends "Hello world from FTM Test App", and loops. Total decoy.
3. **`DIAG_SUBSYS_FTM_ANT` (94)**: A leaked header showed this is a dedicated Antenna FTM subsystem. I grepped the entire `/vendor` and `/system` partitions. **Zero references.** It was stripped from this build.
4. **FTM RxAGC Opcodes**: FTM (Factory Test Mode) uses command/response, which *isn't* blocked like the streaming logs. However, the actual payload layouts for querying RxAGC/RSSI are locked inside Qualcomm's proprietary modem source. Guessing opcodes is a great way to accidentally key the transmitter or wipe calibration data. I refused to risk it.
5. **USB D+ "Deep Flash" Cable**: Standard Qualcomm recovery involves shorting USB D+ to GND to force EDL. I built a cable and tried it. Because USB-C *doesn't power the AP* on this board, the PBL makes its boot decisions independently of the USB data lines. Shorting D+ literally just hides the device from the host OS while held. It does nothing to trigger EDL.
6. **Generic Xiaomi Firehose Loaders**: I tried using the generic `daisy_prog_emmc_firehose_8953_ddr.mbn` (from the Xiaomi Mi A2 Lite, which shares the SDM450). The Sahara protocol flat-out refuses it with `0x09 INVALID_IMAGE_TYPE` before transferring a single byte. Fleet vendors lock these down tight.

---

## 5. Factory Infrastructure & FFBM
The device ships with Qualcomm's factory test stack, but it's heavily neutered.

* **FFBM Gating**: The native USB diag interface (`diag,serial_cdev,rmnet,adb`) is only bound if the device boots into Fast Factory Boot Mode (`ro.bootmode=ffbm-0x`). In normal Android boot, raw USB DIAG yields "Access Denied". You *must* use QCSuper over `adb shell` + `su`.
* **The `IFactory` HIDL Interface**: I dumped the HIDL stub. It has exactly 9 methods (e.g., `runApp`, `enterShipMode`, `wifiEnable`). `runApp` is not a generic shell exec; the `vendor_mmid` daemon hardcodes it to only launch `ftmdaemon` (NFC) or `mm-audio-ftm` (Audio). There is no hidden RF method.
* **`ftm_test_config`**: I read the FTM sequence files. They contain *only* audio test cases (speaker, mic, codec loopback). Grep for `lte|rf_|cell|rssi` returns absolutely nothing.

---

## 6. Engineering Quirks
If you are writing scripts to interact with this dashcam, beware of these platform-specific landmines that cost me hours of debugging.

### The `su -c` SELinux Context Trap
If you chain commands like this:
`su -c "cmd1 && cmd2"`
Magisk's `u:r:magisk:s0` context **only applies to `cmd1`**. By the time `cmd2` runs, the context drops back to `u:r:shell:s0`. You will get bizarre "Permission Denied" errors that look like standard Linux DAC issues, but they are actually SELinux blocking you. 
**Fix:** Issue one `su -c` per privileged command. Never chain with `&&` or `;` inside the string.

### PowerShell vs. ADB
PowerShell will actively sabotage your ADB commands. 
1. It expands `$(...)` and `$var` inside double-quoted strings, completely breaking commands like `adb shell "...getprop..."`.
2. It mangles binary outputs when using `>`. 
**Fix:** Push LF-normalized `.sh` scripts to the device and execute them, or use `System.Diagnostics.Process` in C#/Python. If you are using Git Bash on Windows, you *must* run:
`export MSYS_NO_PATHCONV=1; export MSYS2_ARG_CONV_EXCL="*"`

### On-Device `grep` is Broken
The native busybox/toolbox `grep` on this Android 9 build silently fails and returns empty when using alternation (e.g., `grep -rl 'a\|b\|c'`). It will hide real matches from you. 
**Fix:** Only use single patterns per `grep` call when scanning the device filesystem.

### The `-120 dBm` Placeholder
In the telephony registry, the signal strength is pinned to `-120 dBm`. **Do not cite this as proof the receiver front-end works.** Because streaming DIAG logs are blocked at the kernel (`errno: 22`), we can't get a live RxAGC reading. The `-120` figure is likely just a NAS-layer placeholder for "No Service", not a live hardware poll.

### System-as-Root (SAR) Block Level RO
Because this is Legacy SAR, `mount -o remount,rw /` will fail. The kernel enforces Read-Only at the block device level. You must clear it first:
```bash
su -c 'blockdev --setrw /dev/block/mmcblk0p28'
su -c 'mount -o remount,rw /'
# ... do your edits ...
su -c 'mount -o remount,ro /'
```

---

*End of Deep Dive. If you manage to find the eMMC DAT0 test point or figure out what the `15, 15, 10` triple in `rffe_scan.dat` means, please open a PR!*
