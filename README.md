# The Recording That Wasn't There

### I wanted to reverse-engineer a cheap MP3 player and see if I could run Rockbox. Instead, I recovered nearly four hours of audio its interface couldn't show me—and I'm still trying to understand why it was there, and what the voices are saying.

**A raw-storage digital forensics investigation of an Actions Semiconductor MP3 player**  
**Investigated:** October 11–12, 2026 · **Status:** Ongoing · **Platform:** Kubuntu Live + Kali ARM64 (VMware) + macOS

> **The finding:** A player whose music interface appeared empty exposed a **3-hour, 43-minute, 39.584-second WAV** in its raw USB mass-storage space. I also found an invalid FAT32 boot sector, a recoverable Windows software archive, and **3,102 consecutive identical 1 MiB blocks** toward the end of its apparent 4 GB storage.
>
> **The question I can't shake:** What is being said in that recording? Who or what put it there, and why couldn't I find it through the player's interface? The recovered bytes establish that the audio is present in USB-accessible storage; they do **not** explain its origin or whether it was deliberately hidden.
>
> **What this does *not* prove:** I have not demonstrated malicious firmware, covert recording, counterfeit storage capacity, or that anyone previously owned the device.

**Single-file research report.** The narrative and full technical record are integrated below. Expand the *Technical evidence & commands* sections for preserved commands, hashes, terminal outputs, and reproducibility scripts. Full original Binwalk logs are included at the end.

---


## Why I started looking

I've had this generic MP3 player for **about a year**. I bought it on eBay, where it was advertised as new, because I liked the idea of carrying around a small dedicated music device. In practice, I didn't enjoy using it. The interface was clunky, the buttons felt awkward, and navigating through music was more frustrating than it should have been.

Rather than replace it, I wondered whether I could make it better myself.

My original plan was to identify the chipset, acquire the existing firmware, bring it into **Ghidra**, and see whether I could modify the interface or possibly explore a **[Rockbox](https://www.rockbox.org/) port**. Rockbox is open-source replacement firmware for supported digital music players; I did **not** establish that my particular device is supported or that a port is feasible. This was a research goal, not an installation attempt.

I didn't buy the device to hunt for malware or to recover somebody else's recordings. That investigation developed because the storage behaved in ways I couldn't explain. Once I found a recording that I hadn't gone looking for, though, my priorities changed. I was—and still am—**extremely curious about what's being said**, whether those sounds really are conversations, **why such a long WAV ended up on this device**, and why none of it showed up when I used the normal interface.

Curiosity is the reason I kept investigating; it isn't evidence of malicious behavior. Before I can make a claim about the recording, I need to establish what the bytes actually show and what they leave unanswered.

I should also be clear about the timeline: **the device was advertised as new when purchased, but I had owned it for approximately a year when I made the forensic image**. The “new” listing is not evidence that nobody had ever written to its flash memory, and I cannot date any recovered content simply from that listing.

### The questions driving this investigation

The questions are now more personal—and more forensic—than whether I can replace the firmware:

- **What am I hearing?** Some portions sound like distant speech in a language that, to my ear, might be from East or Southeast Asia. I cannot reliably identify the language or understand the words. Is it conversation, speech captured in a test environment, or something else?
- **Why is the recording there?** Did the player's own recorder create it, was it copied during manufacturing, or is it unindexed data from earlier use or provisioning? What evidence could distinguish those explanations?
- **Why can't I see it?** The player showed an empty library, yet the raw image contains a valid WAV. Is that simply the consequence of unusable filesystem metadata, or is there some other storage or indexing behavior I haven't understood? **Invisible in the UI does not, by itself, mean intentionally concealed.**
- **What can I actually do about it?** Preserve the original state, investigate the audio without exposing anyone's private conversation, compare the WAV to a controlled recording from the device, and only then decide whether the storage or device needs repair—or whether there is a defensible privacy issue to report.

I don't have those answers yet. This README is the evidence trail for getting closer to them.


## The short version

| Observation | What I established |
|---|---|
| USB identity | `10d6:1101`, `ACTIONS / HS USB FlashDisk`; SCSI `ACTIONS USB DISK FOB 2.0` |
| Raw image | `4,148,166,656` bytes (`3,956 MiB`, about `3.86 GiB`) |
| Partition | MBR lists a FAT32-type partition starting at LBA `128`, but the expected boot sector is **all zeros** |
| Windows software | Valid Microsoft Cabinet archive at `0x801000`; **120 entries**, including Actions/ADFU driver information |
| Repeating content | **3,102 MiB in one continuous run** equals repeated copies of the same 1 MiB block |
| Recovered audio | WAV at **`0x2000000` / 32 MiB**, **`03:43:39.584`**, 16 kHz, 16-bit PCM |
| Audible content | I heard rustling, impacts, a persistent hum, and what sounded like distant voices; **language is unverified** |
| Firmware dump | **Not acquired** — this is a USB-accessible **storage** image, not a firmware dump |
| Malware | **None established**; limited antivirus scanning of extracted Windows components found zero detections |

<details>
<summary><strong>Contents</strong></summary>

- [Why I started looking](#why-i-started-looking)
- [The questions driving this investigation](#the-questions-driving-this-investigation)
- [The short version](#the-short-version)
- [The setup and evidence preservation](#the-setup-and-evidence-preservation)
- [The first anomaly: a broken filesystem](#the-first-anomaly-a-broken-filesystem)
- [Something unexpected in the raw bytes](#something-unexpected-in-the-raw-bytes)
- [The high-entropy data that repeated](#the-high-entropy-data-that-repeated)
- [The discovery: a nearly four-hour WAV](#the-discovery-a-nearly-four-hour-wav)
- [Listening to the recovered audio and investigating unidentified speech](#listening-to-the-recovered-audio-and-investigating-unidentified-speech)
- [Was this recorded by the player?](#was-this-recorded-by-the-player)
- [Security implications and limitations](#security-implications-and-limitations)
- [What I learned](#what-i-learned)
- [What's next](#whats-next)
- [References and responsible evidence handling](#references-and-responsible-evidence-handling)
- [Raw Binwalk logs (verbatim)](#raw-binwalk-logs-verbatim)

</details>

---


## The setup and evidence preservation

I wanted to investigate the player without exposing my primary machine unnecessarily or accidentally modifying the original device. I connected the **MP3 player to a separate physical laptop booted into Kubuntu Live**, and I used that laptop to acquire the USB-readable storage. I then analyzed the image from an **ARM64 Kali Linux virtual machine running in VMware on my Apple Silicon Mac**.

The device itself was not directly connected to the Mac during acquisition. Later, a copy of the recovered audio was moved to macOS for local transcription work.

```text
Generic MP3 player  (10d6:1101)
         │
         │ USB mass storage
         ▼
 Separate Kubuntu Live laptop
         │
         │ dd, read full logical device
         ▼
 SanDisk USB drive: actions_storage.img
         │
         │ mounted read-only
         ▼
 Kali Linux ARM64 VM on Apple Silicon Mac
         │
         ├── partition analysis and raw-sector inspection
         ├── Binwalk / CAB carving / PE inspection
         ├── block comparison and hashing
         └── WAV carving, audio metadata and playback
                     │
                     └── copied FLAC → macOS (local MLX Whisper work)
```

### Acquisition

The device exposed an approximately 3.86 GiB disk at `/dev/sdd` **on that Kubuntu boot**. Linux block-device paths can change; `/dev/sdd` is the recorded source, not a universal device name. I imaged it with:

```bash
sudo dd if=/dev/sdd \
  of=/run/media/kubuntu/myusb/actions_storage.img \
  bs=4M iflag=fullblock status=progress conv=fsync
```

The resulting image was exactly **4,148,166,656 bytes**, matching the reported logical-device size. On Kali, the SanDisk was mounted **read-only** at `/mnt/actions-usb`; the image was read and carved from there.

```bash
sudo fdisk -l /mnt/actions-usb/actions_storage.img
```

```text
Disk size: 3.86 GiB, 4148166656 bytes, 8101888 sectors
Units: sectors of 1 * 512 = 512 bytes
Disklabel type: dos
Partition 1: start 128, end 4050815, sectors 4050688, type 0x0b W95 FAT32
```

**Forensic limitation:** There was no saved SHA-256 baseline at the moment of acquisition on Kubuntu. Hashing the preserved image afterward is still useful for integrity control, but I am not claiming a fully verified acquisition-time chain of custody. The acquisition notes and forensic limitations are reproduced immediately below.



<details>
<summary><strong>Technical evidence &amp; commands: USB identification, read-only imaging and disk geometry</strong></summary>

Commands in this section describe the recorded Kubuntu/Kali acquisition; device paths such as `/dev/sdd` are host-specific.

### Hardware, USB identity, systems and file paths

| Component | Recorded details |
|---|---|
| Device | Generic inexpensive MP3/MP4 player; eBay listing advertised **sold as new**; enclosure apparently sealed |
| Normal interface | Empty music library in UI; device has **voice recorder** menu/feature |
| USB ID | `10d6:1101` — Actions Semiconductor |
| USB manufacturer/product | `ACTIONS` / `HS USB FlashDisk` |
| `usb.ids` text | Descriptions include “D-Wave 2GB MP4 Player / AK1025 MP3/MP4 Player”; this is **not** verified silicon identity |
| USB version/device revision | USB 2.0; `bcdDevice 1.00` (reported in earlier enumeration) |
| Endpoint descriptors | Bulk IN `0x81`, bulk OUT `0x02` (reported) |
| SCSI inquiry | Vendor `ACTIONS`, product `USB DISK FOB 2.0`, product revision `0` |
| Initial device nodes | One 0-byte Actions logical device, plus ~3.86 GiB storage device (assignments around `/dev/sdc`, `/dev/sdd`; these vary) |
| Acquisition computer | Separate physical Kubuntu **live** laptop; original player physically attached here |
| Backup USB | SanDisk Ultra Flair, VID:PID `0781:5591`, ~57.3 GB |
| Kali computer | Apple Silicon MacBook running Kali **ARM64/AArch64** in VMware |
| Player connection to Mac | **No direct physical connection** reported |
| Kali SanDisk node | `/dev/sda1` |
| Read-only backup mount | `/mnt/actions-usb` |
| Evidence image | `/mnt/actions-usb/actions_storage.img` |
| Analysis root | `/home/jungl3master/actions-lab/` |
| CAB path | `~/actions-lab/artifacts/suspected.cab` |
| Extracted CAB | `~/actions-lab/artifacts/cab_contents/` |
| PE candidates | `~/actions-lab/artifacts/pe_candidates/` |
| Audio samples | `~/actions-lab/artifacts/audio_samples/` |
| Full WAV | `~/actions-lab/artifacts/recovered_audio/full_recording.wav` |
| Lossless FLAC | `~/actions-lab/artifacts/recovered_audio/full_recording.flac` |
| Suggested transcription output | `~/actions-lab/artifacts/transcription/` — not confirmed created |

### Original SCSI inquiry command (reported performed)

```bash
sudo sg_inq /dev/sdd
```

Observed SCSI text (relevant fields as recorded):

```text
Vendor identification: ACTIONS
Product identification: USB DISK FOB 2.0
Product revision level: 0
Peripheral device type: disk
```

---

### Acquisition command — **performed on Kubuntu**


```bash
sudo dd if=/dev/sdd \
  of=/run/media/kubuntu/myusb/actions_storage.img \
  bs=4M iflag=fullblock status=progress conv=fsync
```

The source byte count and resulting image byte count were reported to match. No I/O errors were reported; the original player was safely disconnected. A baseline SHA-256 checksum on Kubuntu **was not saved/verified in the session**; a hash generated later on Kali is not a substitute for two independent acquisition hashes. Report did not include a complete `dd` progress transcript, so no progress bytes or throughput are invented here.

### Geometry and partition listing — observed


```bash
sudo fdisk -l /mnt/actions-usb/actions_storage.img
```

```text
Disk size: 3.86 GiB, 4148166656 bytes, 8101888 sectors
Units: sectors of 1 * 512 = 512 bytes
Disklabel type: dos
Partition 1: start 128, end 4050815, sectors 4050688, type 0x0b W95 FAT32
```

Exact calculations:

| Quantity | Value |
|---|---:|
| Raw image | **4,148,166,656 bytes** = **3,956 MiB** |
| Sectors | **8,101,888 × 512 bytes** |
| Half-disk point | LBA **4,050,944** = byte offset **2,074,083,328** |
| Partition start | LBA **128** = byte offset **65,536** |
| Partition last LBA | **4,050,815** |
| Partition length | **4,050,688 sectors** = **2,073,952,256 bytes** = **1,977.875 MiB** |
| Reserved before partition | 128 sectors = 64 KiB |
| Unpartitioned gap between partition end and half-point | 128 sectors = 64 KiB |

**Observation:** partition boundaries are symmetrical around the half-capacity boundary; this looks deliberate, but a synthesized MBR, stale partition table, formatter template or other configuration have **not** been distinguished.

</details>



<details>
<summary><strong>Technical evidence &amp; commands: Firmware tooling explored during the original Rockbox investigation</strong></summary>

This section documents compiled tooling and research leads, **not** a successful firmware dump or a supported Rockbox port.

### Actions Semiconductor firmware investigation (separate track)

Two public projects considered:

- `https://github.com/Hannibal-pl/usbfw` — tools for certain Actions MP3 firmware/USB devices.
- `https://github.com/ilyakurdyukov/actions_flash` — research/tooling including ATJ2127/ATJ2157 families and ADFU operations.

### usbfw compilation — performed on Kali ARM64

```bash
git clone https://github.com/Hannibal-pl/usbfw.git
cd ~/actions-lab/usbfw
make
```

Initial linking failed because library link options preceded object files. Makefile ordering was changed so objects come before `-lusb-1.0`; successful command resembled:

```bash
gcc -Wall -g -std=gnu99 -flto -o usbfw \
  afi.o cmdline.o context.o commands.o fw.o main.o tools.o \
  -L/usr/lib64 -lm -lusb-1.0
```

Produced `usbfw`, `tools/apinfo`, `tools/drvinfo`. `./usbfw --help` included:

| Option | Intended function/limitation |
|---|---|
| `-e` / `--enumerate-devices` | Device enumeration |
| `-i` / `--inquiry` | Inquiry |
| `-C` / `--capacity` | Read storage capacity |
| `-F` / `--format-capacity` | Format-capacity inquiry |
| `-I` / `--header-info` | Firmware header info |
| `-S` / `--sys-info` | System information |
| `-r` / `--read` | Generic read |
| `-R` / `--read-fw` | Firmware-related read |
| `-P` / `--dump-raw-fw` | Raw firmware dump command |
| `-A` / `--dump-afi-fw` | AFI dump |
| `-T` / `--test-ram-access`, `-M` / `--read-ram` | RAM-related operations |
| `-w` / `--write` | **Writes data: never run casually** |
| `-E` / `--entry` | **Dangerous per program help** |
| `-D` / `--detach` | Resets/detaches device |

**No commands were executed against the original player using this firmware tool.** The present image is a user-storage dump; firmware access would require appropriate physical connection, vendor protocol, correct chip family and a recovery strategy. Exact SoC remains unknown. Do not infer an AK1025 from a `usb.ids` label or ATJ2127/2157 merely because the normal USB ID matches.

The [actions_flash project documentation](https://github.com/ilyakurdyukov/actions_flash) describes normal mode `10d6:1101` and ADFU mode `10d6:10d6` for certain ATJ2127/ATJ2157 players. These are **tool-family findings, not independently measured traits of this exact player**. Prior to bootloader experimentation, establish a recovery route, battery/boot-key requirements, device-specific chip ID and risks of vendor commands.

---

</details>


## The first anomaly: a broken filesystem

The player seemed empty through its music UI, so I expected either a normal FAT32 volume or an ordinary directory structure. The partition table looked plausible at first: one FAT32-typed entry starting at **LBA 128** and ending at **LBA 4,050,815**.

The numbers were unusual. The raw device had **8,101,888 sectors**; half was **4,050,944 sectors**. The partition starts 128 sectors in and ends 128 sectors *before* the halfway point. That's a remarkably tidy layout, although it doesn't by itself identify its origin or prove the controller synthesized it.

Then I tried read-only filesystem checks:

```bash
sudo losetup --find --show --read-only --partscan \
  /mnt/actions-usb/actions_storage.img

sudo fsck.vfat -n /dev/loop0p1
```

```text
fsck.fat 4.2 (2021-01-31)
Logical sector size is zero.
```

At the expected FAT32 volume boot sector (**LBA 128**), the sector was **entirely zeroed**. The conventional backup boot-sector location (**LBA 134**) was also all zero. Read-only mounting failed.

That was my first sign that the ordinary directory listing was not going to tell the whole story.

Importantly, **an invalid filesystem is not the same as an empty disk**. A filesystem keeps track of names, directories, allocation, and metadata. If those structures are missing or unusable, the underlying logical sectors can still contain recoverable content.



<details>
<summary><strong>Technical evidence &amp; commands: MBR bytes, invalid FAT32, filesystem errors, and sector samples</strong></summary>

### Read-only loop association and failure


**Performed or observed commands:**

```bash
sudo losetup --find --show --read-only --partscan /mnt/actions-usb/actions_storage.img
sudo losetup -a
sudo fsck.vfat -n /dev/loop0p1
sudo mount -t vfat -o ro,nosuid,nodev,noexec /dev/loop0p1 /mnt/mp3-analysis
```

Relevant output:

```text
/dev/loop0: [2049]:22 (/mnt/actions-usb/actions_storage.img)

fsck.fat 4.2 (2021-01-31)
Logical sector size is zero.

mount: wrong fs type, bad option, bad superblock,
       missing codepage or helper program, or other error
```

### MBR bytes — observed


```bash
xxd -g1 -s 0x1be -l 66 /mnt/actions-usb/actions_storage.img
```

```text
000001be: 00 01 01 00 0b fe ff ff 80 00 00 00 00 cf 3d 00
```

The MBR signature at the end of sector 0 is `55 aa`; disk ID reported `0x00000000`. Partition entry is consistent with type `0x0b`, start `0x80` = 128 and length `0x3DCF00` = 4,050,688. There is no evidence of a valid FAT32 BIOS Parameter Block at LBA 128.

### Boot sector / conventional backup reads — performed


```bash
sudo dd if=/mnt/actions-usb/actions_storage.img bs=512 skip=128 count=1 status=none | xxd -g1
sudo dd if=/mnt/actions-usb/actions_storage.img bs=512 skip=134 count=1 status=none | xxd -g1
```

**Result:** Both 512-byte sectors were reported **all zero**. LBA 134 is the usual location of the FAT32 backup boot sector *if* the partition uses the conventional relative sector 6. Other investigated positions including LBA 129 and LBA 2048 were zeroed in sampled bytes. No repair or formatting was performed.

### Raw-sector sampling — reported first bytes


| LBA | First observed bytes | Interpretation |
|---:|---|---|
| 0 | MBR partition data and `55aa` near end | Partition metadata present |
| 128 | All zero | Expected FAT32 boot sector invalid |
| 129 | All zero (sampled) | Not a normal file/data sector in sample |
| 134 | All zero | Expected conventional backup invalid |
| 2048 | All zero (sampled) | Early zero region |
| 1,000,000 | `8c fe 8c fe fa fe fa fe a5 ff a5 ff 2b 00 2b 00 73 00 73 00 df 00 df 00 cf 00 cf 00 91 ff 91 ff` | Later corroborated as dual-mono PCM-like |
| 2,000,000 | `a9 37 e7 4c b7 39 55 04 1b 74 1c 8a f0 79 2c 34 05 10 39 04 44 f1 8d 90 d2 7f d4 a3 ee 60 fe dd` | High-entropy-looking |
| 3,000,000 | `3e a0 bd 16 2d a3 01 50 65 0a 36 a0 f9 70 e1 f9 d1 c4 81 bc ea 5f 2c 31 04 a9 15 62 b3 3d bd 9e` | High-entropy-looking |
| 4,050,815 | `4c d6 51 40 24 11 94 ce 98 59 6d b2 ef 63 47 ea` | Nonzero at partition end |
| 4,050,816 | `36 a8 b1 1a bd 33 af fe e0 8f 19 8f 08 81 21 39` | Nonzero just outside partition |
| 4,050,817 | `87 1d 33 22 45 70 b9 e3 1f de aa 75 67 eb da 77` | Nonzero outside partition |
| 6,000,000 | `b8 26 ed 46 af 21 8a db f7 98 92 04 ab 22 c1 d9` | Nonzero |
| 8,101,887 | `d8 42 10 01 4d 78 47 1d 96 57 d6 09 d7 5b 5d f0` | Nonzero at last LBA |

These reads are of **logical bytes exposed via USB**, not direct reads of bare NAND pages.

---

</details>


## Something unexpected in the raw bytes

I started examining the image as a byte stream rather than a mounted volume:

```bash
binwalk /mnt/actions-usb/actions_storage.img
```

The scan reported a Microsoft Cabinet archive at:

```text
8392704       0x801000      Microsoft Cabinet archive data, 10071697 bytes, 120 files
```

I carved those exact bytes:

```bash
dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/suspected.cab \
  iflag=skip_bytes,count_bytes \
  skip=8392704 count=10071697 status=progress

cabextract -l ~/actions-lab/artifacts/suspected.cab
cabextract -d ~/actions-lab/artifacts/cab_contents \
  ~/actions-lab/artifacts/suspected.cab
```

The file began with the expected `MSCF` signature, and `cabextract` completed without errors. It contained **120 entries**: Windows executables and libraries, multimedia components, an ActiveX control, setup resources, and USB drivers. Some INF files reference my device's **`USB\VID_10D6&PID_1101`**; others concern Actions ADFU/update modes.

A few items stood out:

| Recovered artifact | Why it mattered |
|---|---|
| `WEXTRACT.EXE` | Windows Cabinet extraction/installer component |
| `grabcom.OCX` | Actions-labelled ActiveX component |
| `transdll.DLL` | PE32 multimedia DLL; exports `CreateAmvInterface` and `GetDeviceIdList` |
| `NetacMOD.sys`, `bulkusb.sys` | Driver files with ADFU-related build paths |
| Windows INF files | References to normal USB mode and ADFU-mode device IDs |
| `DPCA.DLL` | Later recovered as a repeated Visual Studio 2005 installer custom-action DLL |

The archive was not proof of firmware or infection. Its contents **looked consistent with Windows companion/installation software for a portable media device**; their original reason for being in this storage image is still unknown.

A basic ClamAV scan of the extracted archive reported:

```text
Known viruses: 3,628,164
Engine version: 1.4.6
Scanned files: 113
Infected files: 0
```

That is **not a clean bill of health** for the device. It covered those extracted files, not the complete flash image or internal controller firmware.

### The PE signature rabbit hole

A raw search found **52,983 occurrences of the two-byte `MZ` signature**. That number sounded alarming until I validated the PE headers: only **23 plausible PE structures** survived basic checks for `e_lfanew`, `PE\0\0`, section counts, and optional-header format.

Several candidates repeated in an alternating pattern. Hashing their full on-disk PE section ranges showed two groups of **byte-identical DLLs**. One was Microsoft's `DPCA.DLL` (Visual Studio 2005 custom-action component); the other remained unidentified from the metadata available in this session.

Example hashes from the tested candidates:

```text
A | offset 0x11AEE00 | 214,016 bytes
SHA256: 21100a197e3674e9f68a5dd92ed14a15c1b86611bf7003021cb35beaacf23032

B | offset 0x11FBA00 | 151,552 bytes
SHA256: b3df2720a99d9b9d0095ab28912487ead0e3f94fe9209397809b526a7fb14d9e
```

This taught me an early lesson: **a signature match is a hypothesis until the structure around it is validated**. Binwalk also emitted ESP32, eCos, MySQL and JBOOT-looking matches that did not establish the presence of those systems. The PE validation commands and hash results are included in the expanded technical evidence below; the full raw Binwalk scan logs are at the end of this document.



<details>
<summary><strong>Technical evidence &amp; commands: Binwalk, CAB recovery, Windows/ADFU analysis, PE hashes, and full Python scripts</strong></summary>

Reported tool outputs are reproduced as captured. Raw Binwalk reports are preserved verbatim at the end.

### Binwalk scan and false-positive triage

### Performed

```bash
binwalk /mnt/actions-usb/actions_storage.img
```

Both original user-uploaded Binwalk captures are preserved **verbatim in Appendices A and B**. Among their substantive signature results:

| Offset decimal | Hex | Reported signature | Validation |
|---:|---|---|---|
| 8,392,704 | `0x801000` | Microsoft CAB; 10,071,697 bytes; 120 files | **Validated and extracted** |
| 9,437,184 | `0x900000` | RIFF/WAVE 16 kHz stereo PCM | **Real RIFF header; within CAB**; could be an embedded audio asset |
| 10,764,086 | `0xA43F36` | JBOOT STAG with absurd size | Unvalidated, likely false positive in compressed CAB |
| 18,542,080 | `0x11AEE00` | Windows PE | **Validated** with PE parser |
| 18–26 MiB | various | PE, XML, PC bitmaps; `eCos` strings | **Recurring Windows installer structures**; `eCost...` likely false eCos hits |
| 33,554,432 | `0x2000000` | RIFF/WAVE 16 kHz stereo PCM | **Validated; long playable file** |
| multiple other offsets | various | ESP32/ESP32-S2/S3/C6/P4 images, MySQL ISAM/MISAM, OS-9 | **Unvalidated and often implausible**; do not infer ESP32 SoC |

**Important correction:** False-positive signature searches are normal in large image scans. A string `eCostReserve` contains the substring `eCos` but is clearly related to installer costing, not proof of the eCos RTOS. ESP32 hit headers had implausible entry addresses or configurations; they were not independently validated as firmware.

### Recovered CAB — performed

```bash
mkdir -p ~/actions-lab/artifacts

dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/suspected.cab \
  iflag=skip_bytes,count_bytes \
  skip=8392704 count=10071697 status=progress

cabextract -l ~/actions-lab/artifacts/suspected.cab
cabextract -d ~/actions-lab/artifacts/cab_contents ~/actions-lab/artifacts/suspected.cab
```

Observed archive header starts `4d 53 43 46` = `MSCF`; `cabextract` reported **120 entries**, with successful completion:

```text
All done, no errors.
```

The archive's internal filenames resemble `_` followed by uppercase hex GUID-like identifiers, e.g. `_4EB5454793CA45A8B68A6CBFFD4AC73A`. Extracted file types included Windows PE32/PE32+, DLL, OCX, SYS, INF/INI, XML, BMP, text, and some empty files. Metadata dates span the early 2000s to 2022. Presence of an old DLL does not date the disk image or WAV.

---

### Recovered Windows binaries, drivers, and malware scan

### Metadata extraction command — performed

```bash
cd ~/actions-lab/artifacts/cab_contents
find . -maxdepth 1 -type f -print0 | \
  xargs -0 exiftool -csv -FileName -OriginalFileName -FileDescription \
  -CompanyName -ProductName > ~/actions-lab/artifacts/pe_metadata.csv
```

A prior attempt running recursive `exiftool` over the folder did not identify extensionless files as expected; using `find` to pass filenames explicitly corrected that limitation.

### Selected recovered artifacts

| Extracted CAB name | Identity/evidence | Technical relevance |
|---|---|---|
| `_4EB5454793CA45A8B68A6CBFFD4AC73A` | `WEXTRACT.EXE`, Microsoft Win32 Cabinet Self-Extractor; product `Microsoft Windows Operating System`, version `6.00.3790.0` reported | Installer component; metadata alone does not verify signature |
| `_5B996588889747BAA9E14CD51CE52E53` | `NetacMOD.sys`, AMD64 PE32+, version `1.0.0.1`; PDB `e:\work\adfu\sys-update\Debug\amd64\AdfuUd.pdb` | Driver linked to ADFU-oriented development |
| `_0200BEB4EFB34AC8AF68134E35F0622D` | Original filename `bulkusb.sys`; PDB `h:\driver\adfu\objfre_wxp_x86\i386\AdfuUd.pdb` | Older USB driver/tooling |
| `_B68B5462F5494CD681D47D5E97FF7BA0` | Original filename `grabcom.OCX`, “grabcom ActiveX Control Module”, company `actions`, version `1.0.0.1`, build metadata Jun 18, 2007 | Actions-labelled Windows companion component |
| `_F7B153D87A9E41698ADBCB798945DFD2` | `transdll.dll`, version `4.36.3.1124`, localized resource metadata (unrelated to the recovered audio); PE32 DLL | Multimedia converter-related exports and strings |
| `_FE5AB69278B644CA9A74A482666D9C76` | INF/INI, `DriverVer=08/12/2004,2.0.0.0`; includes **`USB\VID_10D6&PID_1101`** | Exact normal-mode USB-ID match; not exact silicon match |
| `_A0338F59FC7B47F6AB375021D253D850` | ADFU INF/INI, `DriverVer=7/24/2008,3.4.0.0` | ADFU mode PIDs & `AdfuUd.sys` Windows service |
| `_735EBCF9CF7C4E0C82840722AD13DA77` | References `\inf\adfuupdate.inf`, `\system32\adfuud.sys` | Driver install metadata |
| `_B50319BF2A0B4628AFFAE51EF694E65E` | Generic USB Flash Disk description | Media-player USB component |

The normal INF also included `10D6` PIDs `1160`, `1100`, `1000`, `1200`, `2200`, and a `1CAE:49E4` entry. The ADFU INF included `10D6` PIDs `8888`, `FF51`, `FF61`, `FF63`, `FF66`, `FF79`, `FF88`, and other `FF76`, `FF96`, `FE01`, `FE02`, `FD01`, `10D6` entries (as reported). Descriptions included `USB 2.0 (HS) ADFU Device`, with service binary path:

```text
%10%\System32\Drivers\AdfuUd.sys
```

### Strings and multimedia functions

Searches included `strings -a` and `grep -Ei 'USB|firmware|flash|ADFU|device|version|upgrade'`. Among reported hits were:

```text
SetFirmwareFile
Eraseflash
AdfuUd.pdb
AdfuUd.SYS
flashsv_decoder
AMV
SWF
AVI
MPEG
WMV
WMA
RMVB
MOV
MP4
MKV
3GP
FLV
```

**Distinction:** `Flash version not supported` often referred to **Adobe Flash media/SWF**, not NAND flash. `SetFirmwareFile` and `Eraseflash` were **strings** in `transdll.dll`; they were **not observed in its PE export table**.

### `transdll.dll` — terminal output

```bash
cd ~/actions-lab/artifacts/cab_contents
file _F7B153D87A9E41698ADBCB798945DFD2
exiftool _F7B153D87A9E41698ADBCB798945DFD2
objdump -p _F7B153D87A9E41698ADBCB798945DFD2 | \
  grep -Ei 'SetFirmwareFile|Eraseflash|Export|DLL Name' | head -50
```

```text
_F7B153D87A9E41698ADBCB798945DFD2: PE32 executable for MS Windows 4.00 (DLL), Intel i386, 5 sections

ExifTool Version Number         : 13.55
File Size                       : 336 kB
File Type                       : Win32 DLL
Machine Type                    : Intel 386 or later, and compatibles
Time Stamp                      : 2022:09:23 03:37:13-04:00
Image File Characteristics      : Executable, No line numbers, No symbols, 32-bit, DLL
File Version Number             : 4.36.3.1124
Language Code                   : [localization value omitted from public excerpt; unrelated to audio]
File Description                : transdll DLL
Internal Name                   : transdll
Original File Name              : transdll.DLL
Product Name                    : transdll Dynamic Link Library

objdump: _F7B153D87A9E41698ADBCB798945DFD2: file format not recognized
```

The native ARM64 `objdump` build may lack PE/COFF target support; this is not proof of a malformed DLL. PE headers were instead parsed with Python's `pefile`.

```bash
sudo apt install python3-pefile
python3 - <<'PY'
import pefile
path = '/home/jungl3master/actions-lab/artifacts/cab_contents/_F7B153D87A9E41698ADBCB798945DFD2'
pe = pefile.PE(path)
print('=== EXPORTED FUNCTIONS ===')
for s in getattr(pe, 'DIRECTORY_ENTRY_EXPORT').symbols:
    print(s.name.decode(errors='replace') if s.name else 'Unnamed', '| RVA:', hex(s.address))
print('\n=== IMPORTED LIBRARIES ===')
for e in pe.DIRECTORY_ENTRY_IMPORT:
    print(e.dll.decode(errors='replace'))
PY
```

**Captured terminal output:**

```text
=== EXPORTED FUNCTIONS ===
CreateAmvInterface | RVA: 0x3040
GetDeviceIdList | RVA: 0xa520
resizeFF | RVA: 0xbc30
run_log | RVA: 0xc1a0

=== IMPORTED LIBRARIES ===
WMVCore.DLL
SHLWAPI.dll
AVI_EncDLL.dll
MFC42.DLL
MSVCRT.dll
KERNEL32.dll
USER32.dll
ADVAPI32.dll
SHELL32.dll
ole32.dll
MSVCP60.dll
```

**Publication note:** The PE resource localization value is deliberately omitted from the excerpt above; it is unrelated to the language of the recovered speech. Other forensic identifiers remain as observed.

**Interpretation:** `CreateAmvInterface`, `WMVCore.DLL`, `AVI_EncDLL.dll` are consistent with an AMV/video conversion component. Windows API imports alone do not prove benign or malicious behavior. Potentially related consumer software mentioned in earlier discussion includes a Sweex MP460 companion package; **this is a possible software-family similarity, not an identification of the physical player**.

### ClamAV scan — reported actual result

```bash
clamscan -r -i ~/actions-lab/artifacts/cab_contents
```

```text
Known viruses: 3,628,164
Engine version: 1.4.6
Scanned directories: 1
Scanned files: 113
Infected files: 0
Data scanned: 20.02 MB
Data read: 19.20 MB
```

120 CAB archive entries versus 113 scanned files may reflect empty entries or scanner counting rules; the exact difference was **not independently reconciled**. The entire 4 GB image and original internal firmware were **not** shown to have been scanned. Further ClamAV WAV/FLAC scans were suggested but their results were not supplied.

---

### `MZ` search, validated PE headers and repeated DLLs

### Naïve raw signature scan — performed

A Python streaming `find()` scan checked `MZ`, `\x7fELF`, and `MSCF` across the full image. User result:

```text
Candidate signature counts:
Windows executable candidate: 52983
ELF executable candidate: 0
Microsoft CAB candidate: 1
```

Two-byte `MZ` hits are *not* themselves executables. Most are incidental binary matches. The absence of a literal ELF magic string does not prove absence of all executable code or firmware.

### Improved PE validation — performed

Python `mmap` scanned `MZ` and checked DOS `e_lfanew`, `PE\x00\x00`, section count and optional-header magic (`0x10B` or `0x20B`). **Verbatim output:**

```text
STORAGE — PE HEADER VALIDATION
-------------------------------------------------------
Potential valid PE: 0x11AEE00
Potential valid PE: 0x11FBA00
Potential valid PE: 0x1252E00
Potential valid PE: 0x12A0400
Potential valid PE: 0x12F6E00
Potential valid PE: 0x1344C00
Potential valid PE: 0x139AE00
Potential valid PE: 0x13E8600
Potential valid PE: 0x143EE00
Potential valid PE: 0x148C800
Potential valid PE: 0x14E2E00
Potential valid PE: 0x1530600
Potential valid PE: 0x1586E00
Potential valid PE: 0x15D4200
Potential valid PE: 0x162AE00
Potential valid PE: 0x1678400
Potential valid PE: 0x16CEE00
Potential valid PE: 0x171C600
Potential valid PE: 0x1772E00
Potential valid PE: 0x17BFA00
Potential valid PE: 0x1804000
Potential valid PE: 0x18A2000
Potential valid PE: 0x18CD000

RESULTS
MZ signatures: 52983
Plausible PE headers: 23
```

### Eight-candidate PE header metadata comparison — performed

```text
Offset: 0x11aee00
Architecture: 0x14c
Sections: 4
Timestamp: 2005-09-23 10:44:38+00:00
Image size: 233472
Header fingerprint: 0a6b68577d5ef331

Offset: 0x11fba00
Architecture: 0x14c
Sections: 5
Timestamp: 2008-05-19 06:24:40+00:00
Image size: 167936
Header fingerprint: 51c0ec9f44c3f15c

Offset: 0x1252e00
Architecture: 0x14c
Sections: 4
Timestamp: 2005-09-23 10:44:38+00:00
Image size: 233472
Header fingerprint: 0a6b68577d5ef331

Offset: 0x12a0400
Architecture: 0x14c
Sections: 5
Timestamp: 2008-05-19 06:24:40+00:00
Image size: 167936
Header fingerprint: 51c0ec9f44c3f15c

Offset: 0x12f6e00
Architecture: 0x14c
Sections: 4
Timestamp: 2005-09-23 10:44:38+00:00
Image size: 233472
Header fingerprint: 0a6b68577d5ef331

Offset: 0x1344c00
Architecture: 0x14c
Sections: 5
Timestamp: 2008-05-19 06:24:40+00:00
Image size: 167936
Header fingerprint: 51c0ec9f44c3f15c

Offset: 0x139ae00
Architecture: 0x14c
Sections: 4
Timestamp: 2005-09-23 10:44:38+00:00
Image size: 233472
Header fingerprint: 0a6b68577d5ef331

Offset: 0x13e8600
Architecture: 0x14c
Sections: 5
Timestamp: 2008-05-19 06:24:40+00:00
Image size: 167936
Header fingerprint: 51c0ec9f44c3f15c
```

### SHA-256 across headers + on-disk section extent — performed

```text
ACTIONS MP3 — PE BINARY COMPARISON
--------------------------------------------------------------------------------

Offset: 0x11aee00
PE size: 214,016 bytes
SHA256: 21100a197e3674e9f68a5dd92ed14a15c1b86611bf7003021cb35beaacf23032

Offset: 0x11fba00
PE size: 151,552 bytes
SHA256: b3df2720a99d9b9d0095ab28912487ead0e3f94fe9209397809b526a7fb14d9e

Offset: 0x1252e00
PE size: 214,016 bytes
SHA256: 21100a197e3674e9f68a5dd92ed14a15c1b86611bf7003021cb35beaacf23032

Offset: 0x12a0400
PE size: 151,552 bytes
SHA256: b3df2720a99d9b9d0095ab28912487ead0e3f94fe9209397809b526a7fb14d9e

Offset: 0x12f6e00
PE size: 214,016 bytes
SHA256: 21100a197e3674e9f68a5dd92ed14a15c1b86611bf7003021cb35beaacf23032

Offset: 0x1344c00
PE size: 151,552 bytes
SHA256: b3df2720a99d9b9d0095ab28912487ead0e3f94fe9209397809b526a7fb14d9e

Offset: 0x139ae00
PE size: 214,016 bytes
SHA256: 21100a197e3674e9f68a5dd92ed14a15c1b86611bf7003021cb35beaacf23032

Offset: 0x13e8600
PE size: 151,552 bytes
SHA256: b3df2720a99d9b9d0095ab28912487ead0e3f94fe9209397809b526a7fb14d9e
```

The four A headers repeat at `0xA4000` = **671,744-byte** intervals. Carved candidates:

```bash
mkdir -p ~/actions-lab/artifacts/pe_candidates

dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/pe_candidates/executable_A.bin \
  iflag=skip_bytes,count_bytes skip=$((0x11AEE00)) count=214016 status=none

dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/pe_candidates/executable_B.bin \
  iflag=skip_bytes,count_bytes skip=$((0x11FBA00)) count=151552 status=none

file ~/actions-lab/artifacts/pe_candidates/executable_A.bin \
     ~/actions-lab/artifacts/pe_candidates/executable_B.bin
exiftool ~/actions-lab/artifacts/pe_candidates/executable_A.bin \
         ~/actions-lab/artifacts/pe_candidates/executable_B.bin
```

**Executable A: observed ExifTool identity**

```text
File Name                       : executable_A.bin
File Size                       : 214 kB
File Type                       : Win32 DLL
Machine Type                    : Intel 386 or later, and compatibles
Time Stamp                      : 2005:09:23 06:44:38-04:00
File Version Number             : 8.0.50727.42
Company Name                    : Microsoft Corporation
File Description                : Microsoft (R) Visual Studio UrlConvert custom action
File Version                    : 8.0.50727.42 (RTM.050727-4200)
Internal Name                   : CustomActions
Original File Name              : DPCA.DLL
Product Name                    : Microsoft® Visual Studio® 2005
Product Version                 : 8.0.50727.42
PDB File Name                   : DPCA.pdb
```

**Executable B:** `PE32` Intel i386 DLL, 5 sections, build timestamp May 19, 2008, linker 6.0, image size 167,936; its `OriginalFileName` was **not provided** in the shared output. Output began:

```text
executable_B.bin: PE32 executable for MS Windows 4.00 (DLL), Intel i386, 5 sections
File Size                       : 152 kB
File Type                       : Win32 DLL
Time Stamp                      : 2008:05:19 02:24:40-04:00
Image File Characteristics      : Executable, No line numbers, No symbols, 32-bit, DLL
Entry Point                     : 0x4c68
```

### Larger surrounding-block comparison — performed; not identical

```text
Offset: 0x11aee00
Size: 671,744 bytes
SHA256: 833528128fb1fc94631f9388ecbaa56095ad546e9f68a92d2eb1c64535f371bd

Offset: 0x1252e00
Size: 671,744 bytes
SHA256: 504fa6c01ab493653e9f470421a576131243e3ff15492370706c75bb698dd4db

Offset: 0x12f6e00
Size: 671,744 bytes
SHA256: 42744ea85fdc8a26706ebe92c8f43bf83c11d0a1c924199e96329a2ec1f9d1bd

Offset: 0x139ae00
Size: 671,744 bytes
SHA256: b49c1c1b3996ad6cbb16d4a4c6267028eca61dfd7d43128edd86632a465a67b6

=== COMPARISON ===
Block 1 vs Block 2: DIFFERENT
  First difference: 0x36e00
Block 1 vs Block 3: DIFFERENT
  First difference: 0x36e00
Block 1 vs Block 4: DIFFERENT
  First difference: 0x36e00
```

Thus the first **224,768 bytes** match, which extends **10,752 bytes** beyond the 214,016-byte PE A extent, while subsequent installer content changes. This fits repeating shared installer DLLs and varying surrounding resources; it does not establish flash corruption.

---

### D1. Raw-image PE validation (performed; produced 23 candidates)


```bash
python3 - <<'PY'
import mmap
import struct
path = '/mnt/actions-usb/actions_storage.img'
print('ACTIONS STORAGE — PE HEADER VALIDATION')
print('-' * 55)
matches = 0
valid = 0
with open(path, 'rb') as f:
    with mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as data:
        pos = 0
        size = len(data)
        while True:
            pos = data.find(b'MZ', pos)
            if pos == -1:
                break
            matches += 1
            if pos + 64 <= size:
                pe_offset = struct.unpack_from('<I', data, pos + 0x3C)[0]
                if 64 <= pe_offset <= 4096:
                    header = pos + pe_offset
                    if header + 26 <= size and data[header:header+4] == b'PE\x00\x00':
                        sections = struct.unpack_from('<H', data, header + 6)[0]
                        optional_magic = struct.unpack_from('<H', data, header + 24)[0]
                        if 1 <= sections <= 96 and optional_magic in (0x10B, 0x20B):
                            valid += 1
                            if valid <= 30:
                                print(f'Potential valid PE: 0x{pos:X}')
            pos += 2
print('\nRESULTS')
print(f'MZ signatures: {matches}')
print(f'Plausible PE headers: {valid}')
PY
```

### D2. Eight extracted-PE header fingerprints (performed)


```bash
python3 - <<'PY'
import pefile
import hashlib
from datetime import datetime, timezone
image = '/mnt/actions-usb/actions_storage.img'
offsets = [0x11AEE00,0x11FBA00,0x1252E00,0x12A0400,
           0x12F6E00,0x1344C00,0x139AE00,0x13E8600]
with open(image, 'rb') as f:
    for offset in offsets:
        f.seek(offset)
        data = f.read(65536)
        try:
            pe = pefile.PE(data=data, fast_load=True)
            timestamp = datetime.fromtimestamp(pe.FILE_HEADER.TimeDateStamp, tz=timezone.utc)
            fingerprint = hashlib.sha256(data[:4096]).hexdigest()[:16]
            print(f'\nOffset: {hex(offset)}')
            print(f'Architecture: {hex(pe.FILE_HEADER.Machine)}')
            print(f'Sections: {pe.FILE_HEADER.NumberOfSections}')
            print(f'Timestamp: {timestamp}')
            print(f'Image size: {pe.OPTIONAL_HEADER.SizeOfImage}')
            print(f'Header fingerprint: {fingerprint}')
        except Exception as e:
            print(f'{hex(offset)}: {e}')
PY
```

### D3. PE section-extent hashing (performed)


```bash
python3 - <<'PY'
import pefile
import hashlib
image = '/mnt/actions-usb/actions_storage.img'
offsets = [0x11AEE00,0x11FBA00,0x1252E00,0x12A0400,
           0x12F6E00,0x1344C00,0x139AE00,0x13E8600]
print('ACTIONS MP3 — PE BINARY COMPARISON')
print('-' * 80)
with open(image, 'rb') as f:
    for offset in offsets:
        try:
            f.seek(offset)
            header = f.read(8192)
            pe = pefile.PE(data=header, fast_load=True)
            section_end = max((section.PointerToRawData + section.SizeOfRawData
                               for section in pe.sections), default=0)
            size = max(pe.OPTIONAL_HEADER.SizeOfHeaders, section_end)
            if not 0 < size <= 8 * 1024 * 1024:
                print(f'{offset:#x}: Suspicious PE size {size}')
                continue
            f.seek(offset)
            data = f.read(size)
            if len(data) != size:
                print(f'{offset:#x}: Truncated data')
                continue
            print(f'\nOffset: {offset:#x}')
            print(f'PE size: {size:,} bytes')
            print(f'SHA256: {hashlib.sha256(data).hexdigest()}')
        except Exception as e:
            print(f'{offset:#x}: {e}')
PY
```

### D4. Four surrounding 671,744-byte block comparison (performed)


```bash
python3 - <<'PY'
import hashlib
image = '/mnt/actions-usb/actions_storage.img'
offsets = [0x11AEE00,0x1252E00,0x12F6E00,0x139AE00]
block_size = 0xA4000
blocks = []
with open(image, 'rb') as f:
    for offset in offsets:
        f.seek(offset)
        data = f.read(block_size)
        blocks.append(data)
        print(f'\nOffset: {offset:#x}')
        print(f'Size: {len(data):,} bytes')
        print(f'SHA256: {hashlib.sha256(data).hexdigest()}')
print('\n=== COMPARISON ===')
for i in range(1, len(blocks)):
    identical = blocks[0] == blocks[i]
    print(f"Block 1 vs Block {i+1}: {'IDENTICAL' if identical else 'DIFFERENT'}")
    if not identical:
        first_difference = next((j for j,(a,b) in enumerate(zip(blocks[0], blocks[i]))
                                 if a != b), None)
        print(f'  First difference: {first_difference:#x}')
PY
```

</details>


## The high-entropy data that repeated

While exploring the rest of the image, I sampled the data distribution and compressibility at increasing offsets. From around **896 MiB onward**, the samples looked almost maximally random:

```text
Offset       Entropy (bits/byte)     Nonzero bytes
896 MiB      ~7.9966                ~99.61%
1024 MiB     ~7.9966                ~99.61%
2048 MiB     ~7.9966                ~99.61%
3072 MiB     ~7.9966                ~99.61%
3840 MiB     ~7.9966                ~99.61%
```

At first I considered compressed or encrypted content. But why did *every* sample have the same distribution? I stopped relying on statistics and compared the actual bytes.

Six separated 64 KiB samples were identical. Then entire **1 MiB blocks** were byte-identical, including samples separated by gigabytes. In the tested window, shifting the data by 512 bytes, 4 KiB, 64 KiB, 128 KiB, 256 KiB or 512 KiB did **not** match; shifting it by **1 MiB** did.

Finally, I compared every aligned 1 MiB block in the image against a reference block at 896 MiB:

```text
=== ACTIONS STORAGE REPETITION MAP ===
Image size: 4,148,166,656 bytes
Total full MiB blocks: 3956

=== MATCHING REGIONS ===
Start: 30 MiB | End: 32 MiB | Length: 2 MiB
Start: 854 MiB | End: 3956 MiB | Length: 3102 MiB

=== SUMMARY ===
Matching blocks: 3106
Percentage of image: 78.51%
Longest continuous run: 854–3956 MiB
```

A closer sector-level scan also found the same reference block at **28–29 MiB**. One further isolated matching block was included in the 3,106 total but was not localized in the provided outputs.

**Result:** **3,102 consecutive 1 MiB blocks** from **854 MiB to the end** are identical. Across the image, **78.51% of aligned 1 MiB blocks** matched that reference block.

This is a statement about the **contents returned by the USB storage interface**, not proof that the NAND is counterfeit, that addresses are physically aliased, or that an attacker filled the disk. It could be a controller-exposed pattern, manufacturing/test data, or some other storage artifact. A real write-and-verify capacity test was **not** performed.



<details>
<summary><strong>Technical evidence &amp; commands: Entropy sampling, repeated-block tests, boundary mapping, and scripts</strong></summary>

### Raw entropy and nonzero-distribution measurements

### 64 KiB samples every 128 MiB — reported output

| Offset (MiB) | Nonzero bytes |
|---:|---:|
| 0 | 0.02% |
| 128 | 92.14% |
| 256 | 92.24% |
| 384 | 89.75% |
| 512 | 89.67% |
| 640 | 89.92% |
| 768 | 91.31% |
| 896 | 99.61% |
| 1024 | 99.61% |
| 1152 | 99.61% |
| 1280 | 99.61% |
| 1408 | 99.61% |
| 1536 | 99.61% |
| 1664 | 99.61% |
| 1792 | 99.61% |
| 1920 | 99.61% |
| 2048 | 99.61% |
| 2176 | 99.61% |
| 2304 | 99.61% |
| 2432 | 99.61% |
| 2560 | 99.61% |
| 2688 | 99.61% |
| 2816 | 99.61% |
| 2944 | 99.61% |
| 3072 | 99.61% |
| 3200 | 99.61% |
| 3328 | 99.61% |
| 3456 | 99.61% |
| 3584 | 99.61% |
| 3712 | 99.61% |
| 3840 | 99.61% |

### Sample entropy and zlib compression ratio — reported

| Offset MiB | Shannon entropy (bits/byte) | zlib size ratio |
|---:|---:|---:|
| 0 | 0.0029 | 0.002 |
| 1 | 0.0000 | 0.001 |
| 4 | 0.0000 | 0.001 |
| 16 | 7.9904 | 0.995 |
| 32 | 6.5895 | 0.543 |
| 64 | 6.5751 | 0.540 |
| 96 | 6.5453 | 0.537 |
| 128 | 6.5530 | 0.538 |
| 256 | 6.5437 | 0.540 |
| 512 | 6.3545 | 0.518 |
| 768 | 6.4763 | 0.528 |
| 832 | 6.4645 | 0.526 |
| 896 | 7.9966 | 1.000 |
| 1024 | 7.9966 | 1.000 |
| 2048 | 7.9966 | 1.000 |
| 3072 | 7.9966 | 1.000 |
| 3840 | 7.9966 | 1.000 |

**Later validation overturned early speculation:** The repeated high-entropy region is not gigabytes of unique, independently random or independently encrypted data. **The same 1 MiB bytes repeat**. The 32–851 MiB region is consistent with the WAV declared extent; some entropy values fit PCM. The 16 MiB sample lies inside the compressed CAB archive.

---

### Repeated high-entropy region: hashes, offsets, boundary

### 64 KiB exact comparisons — performed

```text
=== HIGH-ENTROPY REGION ===
  896 MiB | SHA256: 3c75d3953080d80d25e0
 1024 MiB | SHA256: 3c75d3953080d80d25e0
 1152 MiB | SHA256: 3c75d3953080d80d25e0
 2048 MiB | SHA256: 3c75d3953080d80d25e0
 3072 MiB | SHA256: 3c75d3953080d80d25e0
 3840 MiB | SHA256: 3c75d3953080d80d25e0

=== EXACT SAMPLE COMPARISON ===
896 MiB vs 896 MiB: IDENTICAL
896 MiB vs 1024 MiB: IDENTICAL
896 MiB vs 1152 MiB: IDENTICAL
896 MiB vs 2048 MiB: IDENTICAL
896 MiB vs 3072 MiB: IDENTICAL
896 MiB vs 3840 MiB: IDENTICAL

=== PERIODICITY CHECK ===
Shift      512 bytes: DIFFERENT
Shift     4096 bytes: DIFFERENT
Shift    65536 bytes: DIFFERENT
Shift   1048576 bytes: IDENTICAL

=== LOWER HALF VS UPPER HALF ===
Half-disk offset: 2,074,083,328 bytes
  128 MiB + half-disk offset: DIFFERENT
  512 MiB + half-disk offset: DIFFERENT
  896 MiB + half-disk offset: IDENTICAL
 1500 MiB + half-disk offset: IDENTICAL
```

A half-disk match *within* the region repeating every 1 MiB is **not** independent evidence of capacity aliasing. Half-disk offset is an integral number of MiB.

### Full 1 MiB comparison — performed

```text
=== COMPARISON AGAINST 896 MiB ===
896 MiB vs   512 MiB: DIFFERENT
896 MiB vs   768 MiB: DIFFERENT
896 MiB vs   832 MiB: DIFFERENT
896 MiB vs   895 MiB: IDENTICAL
896 MiB vs   896 MiB: IDENTICAL
896 MiB vs   897 MiB: IDENTICAL
896 MiB vs   898 MiB: IDENTICAL
896 MiB vs   899 MiB: IDENTICAL
896 MiB vs  1024 MiB: IDENTICAL
896 MiB vs  1025 MiB: IDENTICAL
896 MiB vs  2048 MiB: IDENTICAL
896 MiB vs  3072 MiB: IDENTICAL
896 MiB vs  3840 MiB: IDENTICAL

=== PERIODICITY TEST ===
Shift      512 bytes: DIFFERENT
Shift     4096 bytes: DIFFERENT
Shift    65536 bytes: DIFFERENT
Shift   131072 bytes: DIFFERENT
Shift   262144 bytes: DIFFERENT
Shift   524288 bytes: DIFFERENT
Shift  1048576 bytes: IDENTICAL
```

**Interpretation:** Tested 1 MiB blocks are byte-identical, and the tested 2 MiB window repeats after 1 MiB, not the listed smaller shifts. An untested smaller period remains theoretically possible; the test only excludes those sampled shifts.

### Full-image aligned 1 MiB reference-block map — performed

Script concept: read each aligned 1 MiB block, compare against block at 896 MiB, collect matching runs.

```python
from pathlib import Path
IMG = Path('/mnt/actions-usb/actions_storage.img')
MiB = 1024 * 1024
with IMG.open('rb') as f:
    total_blocks = IMG.stat().st_size // MiB
    f.seek(896 * MiB)
    ref = f.read(MiB)
    f.seek(0)
    runs, run_start, matches = [], None, 0
    for i in range(total_blocks):
        same = f.read(MiB) == ref
        if same:
            matches += 1
            if run_start is None: run_start = i
        elif run_start is not None:
            runs.append((run_start, i))
            run_start = None
    if run_start is not None: runs.append((run_start, total_blocks))
    print('Matches', matches, 'of', total_blocks, 'runs', runs)
```

**Verbatim captured output summary:**

```text
=== ACTIONS STORAGE REPETITION MAP ===
Image size: 4,148,166,656 bytes
Total full MiB blocks: 3956

=== MATCHING REGIONS ===
Start: 30 MiB | End: 32 MiB | Length: 2 MiB
Start: 854 MiB | End: 3956 MiB | Length: 3102 MiB

=== SUMMARY ===
Matching blocks: 3106
Percentage of image: 78.51%
Longest continuous run: 854–3956 MiB
```

The original mapping script printed only runs **two blocks or longer**. A subsequent scan found 28–29 MiB (1 MiB); the remaining single matching block(s) have not all been individually enumerated in the shared output. **3106 total** implies **four matching blocks outside** the 854–3956 MiB continuous run. Confirmed earlier runs account for three of those four (`28–29`, `30–32`); location of the final isolated block was **not supplied**.

### Boundary scan at 512-byte sector resolution — performed

```text
=== REPETITION BOUNDARY ANALYSIS ===

Region: 28–34 MiB
Start: 28.000000 MiB (LBA 57344)
End:   29.000000 MiB (LBA 59392)
Length: 1,048,576 bytes

Start: 30.000000 MiB (LBA 61440)
End:   32.000000 MiB (LBA 65536)
Length: 2,097,152 bytes

Region: 850–856 MiB
Start: 854.000000 MiB (LBA 1748992)
End:   856.000000 MiB (LBA 1753088)
Length: 2,097,152 bytes
```

**Confirmed:** the 854–3956 MiB range, **3,102 MiB long**, consists of one aligned 1 MiB reference block repeated 3,102 times. Four additional 1 MiB blocks match elsewhere. Whether this is a controller-generated pattern, leftover factory/test write, storage imaging artifact, or physically stored repeated content **is unknown**. A non-destructive repeated-read comparison and eventual controlled capacity test on hardware are needed to distinguish causes.

---

### D5. Whole-image 1 MiB exact-match map (performed)


```bash
python3 - <<'PY'
from pathlib import Path
IMG = Path('/mnt/actions-usb/actions_storage.img')
MiB = 1024 * 1024
with IMG.open('rb') as f:
    total_size = IMG.stat().st_size
    total_blocks = total_size // MiB
    f.seek(896 * MiB)
    reference = f.read(MiB)
    print('=== ACTIONS STORAGE REPETITION MAP ===')
    print(f'Image size: {total_size:,} bytes')
    print(f'Total full MiB blocks: {total_blocks}')
    f.seek(0)
    runs = []
    run_start = None
    matching_blocks = 0
    for i in range(total_blocks):
        block = f.read(MiB)
        identical = block == reference
        if identical:
            matching_blocks += 1
            if run_start is None:
                run_start = i
        elif run_start is not None:
            runs.append((run_start, i))
            run_start = None
    if run_start is not None:
        runs.append((run_start, total_blocks))
    print('\n=== MATCHING REGIONS ===')
    for start, end in runs:
        length = end - start
        if length >= 2:
            print(f'Start: {start} MiB | End: {end} MiB | Length: {length} MiB')
    print('\n=== SUMMARY ===')
    print(f'Matching blocks: {matching_blocks}')
    print(f'Percentage of image: {matching_blocks / total_blocks * 100:.2f}%')
    if runs:
        longest = max(runs, key=lambda r:r[1]-r[0])
        print(f'Longest continuous run: {longest[0]}–{longest[1]} MiB')
PY
```

### D6. Sector-level boundary comparison (performed)


```bash
python3 - <<'PY'
from pathlib import Path
IMG = Path('/mnt/actions-usb/actions_storage.img')
MiB, SECTOR = 1024*1024, 512
with IMG.open('rb') as f:
    f.seek(896*MiB)
    reference = f.read(MiB)
    print('=== REPETITION BOUNDARY ANALYSIS ===')
    for start_mib,end_mib in [(28,34),(850,856)]:
        start,end = start_mib*MiB,end_mib*MiB
        f.seek(start)
        data = f.read(end-start)
        runs, run_start = [], None
        for i in range(0,len(data),SECTOR):
            absolute = start+i
            position = absolute%MiB
            actual = data[i:i+SECTOR]
            expected = reference[position:position+SECTOR]
            match = actual == expected
            if match and run_start is None:
                run_start = absolute
            elif not match and run_start is not None:
                runs.append((run_start,absolute))
                run_start = None
        if run_start is not None:
            runs.append((run_start,end))
        print(f'\nRegion: {start_mib}–{end_mib} MiB')
        for a,b in runs:
            if b-a >= 8192:
                print(f'Start: {a/MiB:.6f} MiB (LBA {a//SECTOR})')
                print(f'End:   {b/MiB:.6f} MiB (LBA {b//SECTOR})')
                print(f'Length: {b-a:,} bytes\n')
PY
```

</details>


## The discovery: a nearly four-hour WAV

The same Binwalk scan had identified a RIFF/WAV signature at **exactly 32 MiB** (`0x2000000`). I initially treated it as another signature to verify. Parsing the header changed the direction of the investigation.

```text
OFFSET: 0x2000000 (32.00 MiB)
Signature: b'RIFF'
Format: b'WAVE'
Declared RIFF size: 858,853,880 bytes

fmt : 16-byte PCM format
fact: 460 bytes
data: 858,853,376 bytes

Audio format: 1 (PCM)
Channels: 2
Sample rate: 16000 Hz
Bits per sample: 16
Byte rate: 64000
```

**The file declared 3 hours, 43 minutes and 39.584 seconds of audio.** The structure was internally consistent and fit within the raw image. I carved it directly:

```bash
dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/recovered_audio/full_recording.wav \
  bs=4M iflag=skip_bytes,count_bytes \
  skip=33554432 count=858853888 status=progress
```

FFprobe recognized it as a single, valid PCM audio stream:

```text
format_name=wav
codec_name=pcm_s16le
sample_rate=16000
channels=2
bits_per_sample=16
duration=13419.584000
size=858853888
probe_score=99
```

The full RIFF parsing, FFprobe, and ExifTool outputs appear immediately below.

At this point, I wasn't thinking much about Rockbox. I was looking at a recording lasting almost four hours that I had never encountered in the player's menus, and I wanted to know **what it contained**. When the first excerpts sounded like a room with movement and distant voices rather than a simple test tone, that led to a harder question: **was this recorded on the player, copied onto it, or left in storage from some earlier process?**

Just as importantly, **why did I have to carve it out of raw storage to find it?** The broken filesystem is a plausible explanation for the missing directory entry or index, but it does not tell me who wrote the file or whether anyone intended it to be inaccessible.

### Its header was strange in a useful way

The file has an unusually large **460-byte `fact` chunk**. When I inspected those 460 bytes, **every one was zero**. Together with the standard chunk headers, this places the start of the actual PCM samples exactly **512 bytes** after the beginning of the WAV—one conventional logical sector.

```text
WAV-relative offset
0       ┌───────────── RIFF / WAVE (12 B)
12      ├───────────── fmt  (24 B including chunk header)
36      ├───────────── fact header (8 B)
44      ├───────────── 460 zero bytes
504     ├───────────── data header (8 B)
512     └───────────── audio samples begin
```

That pattern **could** be consistent with a sector-aligned recorder or an application that reserves a header sector, but it doesn't identify the software or device that generated it.

There was also a second, much shorter **2.88-second** RIFF/WAV header around **9 MiB**, inside the range of the Windows CAB archive. I validated that header too, but its relationship to the archive is not established. Its existence should not be confused with the nearly four-hour recording.

### The surrounding image made more sense

After verifying the file, a plausible map of the raw image looked like this:

```text
           RAW USB-ACCESSIBLE STORAGE — 3,956 MiB

    0 MiB  ┌──────────────────────────────────────────────┐
           │ Early zeros / MBR / unmountable FAT32 entry │
    ~8 MiB ├──────────────────────────────────────────────┤
           │ Recoverable CAB and Windows setup resources │
   ~26 MiB ├──────────────────────────────────────────────┤
           │ Mixed content and some repeated filler      │
    32 MiB ├──────────────────────────────────────────────┤ ◄ WAV at 0x2000000
           │                                              │
           │    RECOVERED WAV — 03:43:39.584              │
           │    16 kHz • 16-bit PCM • 2 channels          │
           │                                              │
 ~851 MiB  ├──────────────────────────────────────────────┤ ◄ Declared WAV end
           │ Uncharacterized ~2.93 MiB gap                │
   854 MiB ├──────────────────────────────────────────────┤
           │                                              │
           │ Repeated identical 1 MiB blocks              │
           │ 3,102 blocks / 3,102 MiB                     │
           │                                              │
 3,956 MiB └──────────────────────────────────────────────┘
```

This is an **evidence-based logical-image map**, not a verified physical NAND map or a reconstructed filesystem.



<details>
<summary><strong>Technical evidence &amp; commands: RIFF validation, exact carving, PCM metadata, and WAV header scripts</strong></summary>

### The large WAV: discovery, exact structure, carving

### Header validation — performed

The header was parsed at two Binwalk-identified offsets, with these **actual outputs**:

```text
=======================================================
OFFSET: 0x900000 (9.00 MiB)
=======================================================
Signature: b'RIFF'
Format:    b'WAVE'
Declared RIFF size: 184,824 bytes
Declared RIFF end:  9.176270 MiB
Distance from 854 MiB boundary: -885,861,888 bytes

Chunk: b'fmt '
Size:  16 bytes
Start: 9.000011 MiB
Audio format: 1
Channels:     2
Sample rate:  16000
Bits/sample:  16
Byte rate:    64000

Chunk: b'fact'
Size:  460 bytes
Start: 9.000034 MiB

Chunk: b'data'
Size:  184,320 bytes
Start: 9.000481 MiB
Audio data starts: 9.000488 MiB
Audio data ends:   9.176270 MiB
Declared audio data fits within the image.

=======================================================
OFFSET: 0x2000000 (32.00 MiB)
=======================================================
Signature: b'RIFF'
Format:    b'WAVE'
Declared RIFF size: 858,853,880 bytes
Declared RIFF end:  851.066895 MiB
Distance from 854 MiB boundary: -3,075,584 bytes

Chunk: b'fmt '
Size:  16 bytes
Start: 32.000011 MiB
Audio format: 1
Channels:     2
Sample rate:  16000
Bits/sample:  16
Byte rate:    64000

Chunk: b'fact'
Size:  460 bytes
Start: 32.000034 MiB

Chunk: b'data'
Size:  858,853,376 bytes
Start: 32.000481 MiB
Audio data starts: 32.000488 MiB
Audio data ends:   851.066895 MiB
Declared audio data fits within the image.
```

### WAV sector-aligned RIFF layout

| Offset within carved WAV | Component | Bytes |
|---:|---|---:|
| `0–11` | RIFF header, `WAVE` format tag | 12 |
| `12–35` | `fmt ` chunk header + 16-byte PCM format body | 24 |
| `36–43` | `fact` chunk header, declared size 460 | 8 |
| `44–503` | `fact` chunk **payload: 460 zero bytes** | 460 |
| `504–511` | `data` chunk header | 8 |
| `512–858853887` | 858,853,376 bytes of interleaved stereo PCM | 858,853,376 |

The recovered WAV starts at **image byte offset 33,554,432 (`0x2000000`) = LBA 65,536**. Audio starts at **image byte offset 33,554,944 = LBA 65,537**. The file ends at exclusive byte offset **892,408,320 (`0x35311200`)**, **851.06689453125 MiB** from image start. The contiguous 1 MiB repeat region begins at **854 MiB**, leaving **3,075,584 bytes** between the WAV declared end and repeated region start. It has not been characterized completely.

The short WAV at 9 MiB lies **within** the larger CAB archive byte range. It has a RIFF header and 184,320-byte data chunk (2.88 seconds at 64,000 B/s). A self-consistent WAV header inside a CAB does not establish whether it is a separately carved real file or a stored archive asset until matched to CAB contents.

### Full WAV extraction — performed

```bash
mkdir -p ~/actions-lab/artifacts/recovered_audio

dd if=/mnt/actions-usb/actions_storage.img \
  of=~/actions-lab/artifacts/recovered_audio/full_recording.wav \
  bs=4M iflag=skip_bytes,count_bytes \
  skip=33554432 count=858853888 status=progress
```

**This is carving from logical storage, not firmware extraction.** The audio was unindexed/invisible through the player's music library because the expected filesystem metadata was unavailable, but this alone does not establish intentional hiding.

### Verification commands — performed

```bash
file ~/actions-lab/artifacts/recovered_audio/full_recording.wav
ffprobe -v error -show_entries format=duration,size \
  -of default=noprint_wrappers=1 \
  ~/actions-lab/artifacts/recovered_audio/full_recording.wav
```

Actual later complete FFprobe output is reproduced below. Data length is internally consistent: `214,713,344` stereo frames × 4 bytes/frame = `858,853,376` data bytes; at 16,000 frames/s → **13,419.584 seconds = 03:43:39.584**.

---

### Complete audio stream and metadata outputs

### Full audio metrics — actual Python result

```text
=== WAV AUDIO ANALYSIS ===
Channels: 2
Sample rate: 16000
Sample width: 2 bytes
Total frames: 214,713,344
Duration: 3.728 hours

=== Beginning ===
Position: 0.0 seconds
Left/right identical: 100.00%
RMS amplitude: 981.60
Peak amplitude: 7526
First 12 left samples: (7259, 7401, 7526, 7517, 7391, 7166, 6909, 6707, 6551, 6575, 6628, 6621)

=== 10 minutes ===
Position: 600.0 seconds
Left/right identical: 100.00%
RMS amplitude: 1084.84
Peak amplitude: 5189
First 12 left samples: (-441, -406, -491, -601, -553, -530, -539, -477, -320, -65, 84, 196)

=== Middle ===
Position: 6709.8 seconds
Left/right identical: 100.00%
RMS amplitude: 464.91
Peak amplitude: 1979
First 12 left samples: (53, -47, -26, -97, -157, -174, -79, 81, 99, 4, -58, -123)

=== Near end ===
Position: 13414.6 seconds
Left/right identical: 100.00%
RMS amplitude: 777.88
Peak amplitude: 31237
First 12 left samples: (-1354, -1330, -1211, -903, -486, -138, -44, -109, -251, -394, -596, -935)
```

Interpretation: four one-second samples were dual-mono and non-silent; it has **not** been established that *every* frame across 3h44m has identical channels, or that all audio is continuous rather than copied/assembled. Peak 31,237 is close to full scale for positive signed 16-bit samples.

### Complete captured FFprobe stream/format output

```text
[STREAM]
index=0
codec_name=pcm_s16le
codec_long_name=PCM signed 16-bit little-endian
profile=unknown
codec_type=audio
codec_tag_string=[1][0][0][0]
codec_tag=0x0001
sample_fmt=s16
sample_rate=16000
channels=2
channel_layout=unknown
bits_per_sample=16
initial_padding=0
id=N/A
r_frame_rate=0/0
avg_frame_rate=0/0
time_base=1/16000
start_pts=N/A
start_time=N/A
duration_ts=214713344
duration=13419.584000
bit_rate=512000
max_bit_rate=N/A
bits_per_raw_sample=N/A
nb_frames=N/A
nb_read_frames=N/A
nb_read_packets=N/A
DISPOSITION:default=0
DISPOSITION:dub=0
DISPOSITION:original=0
DISPOSITION:comment=0
DISPOSITION:lyrics=0
DISPOSITION:karaoke=0
DISPOSITION:forced=0
DISPOSITION:hearing_impaired=0
DISPOSITION:visual_impaired=0
DISPOSITION:clean_effects=0
DISPOSITION:attached_pic=0
DISPOSITION:timed_thumbnails=0
DISPOSITION:non_diegetic=0
DISPOSITION:captions=0
DISPOSITION:descriptions=0
DISPOSITION:metadata=0
DISPOSITION:dependent=0
DISPOSITION:still_image=0
DISPOSITION:multilayer=0
[/STREAM]
[FORMAT]
filename=/home/jungl3master/actions-lab/artifacts/recovered_audio/full_recording.wav
nb_streams=1
nb_programs=0
nb_stream_groups=0
format_name=wav
format_long_name=WAV / WAVE (Waveform Audio)
start_time=N/A
duration=13419.584000
size=858853888
bit_rate=512000
probe_score=99
[/FORMAT]
```

### Complete captured ExifTool metadata output

```text
[ExifTool]      ExifToolVersion                 : 13.55
[System]        FileName                        : full_recording.wav
[System]        Directory                       : /home/jungl3master/actions-lab/artifacts/recovered_audio
[System]        FileSize                        : 859 MB
[System]        FileModifyDate                  : 2026:10:11 21:48:15-04:00
[System]        FileAccessDate                  : 2026:10:11 21:48:18-04:00
[System]        FileInodeChangeDate             : 2026:10:11 21:48:15-04:00
[System]        FilePermissions                 : -rw-rw-r--
[File]          FileType                        : WAV
[File]          FileTypeExtension               : wav
[File]          MIMEType                        : audio/x-wav
[RIFF]          Encoding                        : Microsoft PCM
[RIFF]          NumChannels                     : 2
[RIFF]          SampleRate                      : 16000
[RIFF]          AvgBytesPerSec                  : 64000
[RIFF]          BitsPerSample                   : 16
[RIFF]          NumberOfSamples                 : 0
[Composite]     Duration                        : 3:43:40
```

**Important date limitation:** October 11, 2026 is the **date of extraction on Kali**; it is not the original recording date. No `ICRD`, `bext`, `iXML`, `DateTimeOriginal`, creator software or trustworthy recording timestamp was recovered from this output. The `NumberOfSamples: 0` reflects zero-filled `fact` information, not an empty audio stream.

### Entire `fact` chunk — actual output

Read 460 bytes at WAV-relative offset 44:

```text
=== FACT CHUNK CONTENT ===
First 128 bytes (hex):
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

ASCII representation:
.................................................................................................................................

Entire chunk statistics:
Size: 460
Zero bytes: 460
Nonzero bytes: 0
```

**Interpretation:** 460 zero bytes align the PCM payload to exactly the next 512-byte logical sector. The `fact` tag is being used here like padding/reserved area; a sector-oriented recorder or storage-image generator *could* do this. It is **not** evidence of embedded malware, and its precise software origin remains unknown.

---

### D7. Validate two WAV headers and RIFF chunks (performed)


```bash
python3 - <<'PY'
import struct
from pathlib import Path
IMG = Path('/mnt/actions-usb/actions_storage.img')
MiB = 1024*1024
with IMG.open('rb') as f:
    total = IMG.stat().st_size
    for offset in [0x900000,0x2000000]:
        print(f'\nOFFSET: 0x{offset:X} ({offset/MiB:.2f} MiB)')
        f.seek(offset)
        header = f.read(12)
        print(f'Signature: {header[:4]!r}')
        print(f'Format: {header[8:12]!r}')
        if header[:4] != b'RIFF' or header[8:12] != b'WAVE':
            print('Not standard WAV')
            continue
        riff_size = struct.unpack('<I',header[4:8])[0]
        riff_end = offset+8+riff_size
        print(f'Declared RIFF size: {riff_size:,} bytes')
        print(f'Declared RIFF end: {riff_end/MiB:.6f} MiB')
        print(f'Distance from 854 MiB: {riff_end-854*MiB:,} bytes')
        pos=offset+12
        for _ in range(12):
            if pos+8 > min(riff_end,total):
                break
            f.seek(pos)
            chunk=f.read(8)
            if len(chunk)!=8:
                break
            chunk_id=chunk[:4]
            chunk_size=struct.unpack('<I',chunk[4:8])[0]
            print(f'Chunk {chunk_id!r}: {chunk_size:,} bytes; at {pos/MiB:.6f} MiB')
            if chunk_id == b'fmt ' and chunk_size >= 16:
                audio_format, channels, rate, byte_rate, align, bits = struct.unpack('<HHIIHH', f.read(16))
                print(f'Format={audio_format}, channels={channels}, rate={rate}, bits={bits}, byterate={byte_rate}')
            if chunk_id == b'data':
                print(f'Audio data start {(pos+8)/MiB:.6f} MiB')
                print(f'Audio data end {(pos+8+chunk_size)/MiB:.6f} MiB')
                break
            next_pos=pos+8+chunk_size+(chunk_size&1)
            if next_pos<=pos:
                break
            pos=next_pos
PY
```

### D8. Inspect zero-filled `fact` chunk (performed)


```bash
python3 - <<'PY'
from pathlib import Path
wav=Path.home()/'actions-lab/artifacts/recovered_audio/full_recording.wav'
with wav.open('rb') as f:
    f.seek(44)
    data=f.read(460)
print('=== FACT CHUNK CONTENT ===')
print('First 128 bytes (hex):')
print(data[:128].hex(' '))
print('ASCII representation:')
print(''.join(chr(b) if 32 <= b <= 126 else '.' for b in data[:128]))
print('Size:',len(data),'Zero bytes:',data.count(0),'Nonzero:',sum(b!=0 for b in data))
PY
```

</details>


## Listening to the recovered audio and investigating unidentified speech

After carving the WAV, I extracted short clips rather than listening through almost four hours immediately. I heard a constant humming or static-like bed, sharp clunks, rustling/handling sounds, and **what sounded like distant people speaking**. One brief section around **10 minutes** sounded to me like a woman speaking; at approximately **one hour**, I heard additional speech-like content in what sounded to me like an East or Southeast Asian language. That impression remains unverified.

**Those are listening observations—not a verified language identification or a transcript.** The recording is noisy and distant. Sound enhancement can distort it, and automatic speech-recognition tools may hallucinate entire sentences from background noise.

**This is the part I'm most curious about.** Hearing what seemed like a voice around the ten-minute mark, then hearing more speech-like audio roughly an hour in, made the file feel less like an abstract forensic artifact. I want to understand the actual content, not just prove that an audio header exists. Are these recognizable words? What language am I hearing, if it really is speech? Do the sounds indicate a continuous recording, or isolated material loaded as a test? I don't want to answer those questions by imagining dialogue that isn't intelligible.

I also recognize the uncomfortable side of that curiosity: if these are real conversations, they could belong to someone who never intended for me—or anyone else—to hear them. That is why I'm keeping the original recording private while I work out what, if anything, can be validated.

I also measured audio from four widely separated one-second samples:

| Position | RMS amplitude | Peak amplitude | Matching left/right samples |
|---|---:|---:|---:|
| Beginning | 981.60 | 7,526 | 100% |
| 10 minutes | 1,084.84 | 5,189 | 100% |
| Middle | 464.91 | 1,979 | 100% |
| Near end | 777.88 | 31,237 | 100% |

The sampled channels were identical—effectively **dual mono**—and the four regions weren't silent. This **doesn't establish that all 3h44m are continuous speech or that every sample is identical across channels**.

I preserved the raw carved WAV and made a losslessly compressed FLAC for easier playback:

```bash
ffmpeg -i ~/actions-lab/artifacts/recovered_audio/full_recording.wav \
  -c:a flac \
  ~/actions-lab/artifacts/recovered_audio/full_recording.flac
```

The FLAC was approximately **223 MiB**, compared with the WAV's approximately **819 MiB**.

### Can AI tell me what they're saying?

As a follow-up, I moved a **copy** of the audio to macOS and began setting up **MLX Whisper** in a Python virtual environment to run local multilingual transcription and English translation in five-minute chunks. I observed the Hugging Face model-download warning for unauthenticated downloads; I have **not** yet established a verified transcript or identified the language. No interpreted speech is presented as fact in this repository.

A model's confident output is not enough. A fluent speaker will need to compare any candidate text against the original, unenhanced audio before I would quote or publish it. **My aim is to learn enough to understand the nature and possible provenance of the audio—not to publish strangers' conversations.** If the content turns out to be personal, I can describe the technical recovery without disclosing what was said. The recording and transcripts **are not included in this repository**.



<details>
<summary><strong>Technical evidence &amp; commands: Audio excerpts, processing, playback errors, Mac MLX Whisper workflow</strong></summary>

Speech recognition is not a confirmed transcript; all statements about voices are preliminary listening observations.

### Audio listening, files and playback troubleshooting

### Listening observations (user report, not verified linguistic attribution)

- At several positions the audio sounded quiet, static-like, with a continuous hum.
- Around **10 minutes**, a brief distant voice sounded possibly feminine and may have been speaking an East or Southeast Asian language, based on an unverified listening impression. Metallic clunks and rustling/handling noises were audible.
- A supplied unprocessed 30-second clip, approximately **09:50–10:20** of original recording, was discussed as containing a hum, impacts/rustling and speech-like sounds. No intelligible words or language were verified.
- Around **one hour**, I heard clear speech-like audio in an unidentified language that sounded possibly East or Southeast Asian to me.
- My playback screenshot of a seek position showed approximately **883.54 seconds = 00:14:43.54**, i.e., about 6.6% of the file.
- The player has a **voice-recording function**. A separate test recording from the physical player had **not** been demonstrated or analyzed; `test_recording.wav` was merely an example filename (and `ffprobe` returned “No such file or directory”).

**Privacy boundary:** Unknown voices could be third-party speech. No speaker identity, exact language, origin location or recording date established. Avoid publishing full audio or identifiable excerpts.

### Four initial sample clips — extracted

```bash
mkdir -p ~/actions-lab/artifacts/audio_samples
python3 - <<'PY'
import wave
from pathlib import Path
IMG = '/mnt/actions-usb/actions_storage.img'
OUT = Path.home() / 'actions-lab/artifacts/audio_samples'
OUT.mkdir(parents=True, exist_ok=True)
DATA_OFFSET = 32 * 1024 * 1024 + 512
RATE, CHANNELS, WIDTH = 16000, 2, 2
DURATION = 15
positions = {'beginning':0, 'ten_minutes':600, 'middle':6700, 'near_end':13400}
with open(IMG, 'rb') as f:
    for name, seconds in positions.items():
        f.seek(DATA_OFFSET + seconds * RATE * CHANNELS * WIDTH)
        audio = f.read(DURATION * RATE * CHANNELS * WIDTH)
        with wave.open(str(OUT / f'{name}.wav'), 'wb') as out:
            out.setnchannels(CHANNELS)
            out.setsampwidth(WIDTH)
            out.setframerate(RATE)
            out.writeframes(audio)
        print('Extracted:', OUT / f'{name}.wav')
PY
```

### Two-minute voice crop — performed

```bash
python3 - <<'PY'
import wave
from pathlib import Path
img = '/mnt/actions-usb/actions_storage.img'
out = Path.home() / 'actions-lab/artifacts/audio_samples'
out.mkdir(parents=True, exist_ok=True)
with open(img, 'rb') as f:
    f.seek(32 * 1024 * 1024)
    with wave.open(f, 'rb') as r:
        rate, ch, width = r.getframerate(), r.getnchannels(), r.getsampwidth()
        r.setpos(570 * rate)
        audio = r.readframes(120 * rate)
with wave.open(str(out / 'voice_original.wav'), 'wb') as w:
    w.setnchannels(ch)
    w.setsampwidth(width)
    w.setframerate(rate)
    w.writeframes(audio)
PY
```

### Filtering commands — performed/used in session

```bash
ffmpeg -hide_banner -y \
  -i ~/actions-lab/artifacts/audio_samples/voice_original.wav \
  -af 'highpass=f=120,lowpass=f=3800,loudnorm' \
  ~/actions-lab/artifacts/audio_samples/voice_enhanced.wav

ffmpeg -hide_banner -y \
  -i ~/actions-lab/artifacts/audio_samples/voice_original.wav \
  -af 'highpass=f=180,lowpass=f=3200,afftdn=nr=12,loudnorm' \
  ~/actions-lab/artifacts/audio_samples/voice_speech.wav
```

Initial `ffplay` failure was:

```text
/home/jungl3master/actions-lab/artifacts/audio_samples/voice_enhanced.wav: No such file or directory
```

The file was later created successfully, and a separate copy was retained for analysis. FFmpeg's `loudnorm` may internally upsample to **192 kHz**, which does not restore information beyond the original 16 kHz audio. Enhanced audio should never replace the unprocessed source for forensic conclusions.

### Unprocessed 30-second clip — performed and uploaded

```bash
ffmpeg -hide_banner -y -ss 20 -t 30 \
  -i ~/actions-lab/artifacts/audio_samples/voice_original.wav \
  -c:a pcm_s16le \
  ~/actions-lab/artifacts/audio_samples/voice_raw_20-50.wav
```

Clip starts at recording **09:50**, ends **10:20**. Short excerpts `voice_enhanced.wav` and `voice_raw_20-50.wav` were retained separately; these are not substitutes for the full 3h44m source recording.

### FLAC lossless conversion — actual completed command/output

```bash
ffmpeg -i ~/actions-lab/artifacts/recovered_audio/full_recording.wav \
  -c:a flac \
  ~/actions-lab/artifacts/recovered_audio/full_recording.flac
```

Captured FFmpeg terminal summary:

```text
Input #0, wav, from '/home/jungl3master/actions-lab/artifacts/recovered_audio/full_recording.wav':
  Duration: 03:43:39.58, bitrate: 512 kb/s
  Stream #0:0: Audio: pcm_s16le, 16000 Hz, stereo, s16, 512 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (pcm_s16le (native) -> flac (native))
Output #0, flac, to '/home/jungl3master/actions-lab/artifacts/recovered_audio/full_recording.flac':
  Stream #0:0: Audio: flac, 16000 Hz, stereo, s16, 128 kb/s
[out#0/flac @ 0xaaaace6fd500] video:0KiB audio:228369KiB subtitle:0KiB other streams:0KiB global headers:0KiB muxing overhead: 0.003544%
size=  228377KiB time=03:43:39.58 bitrate= 139.4kbits/s speed=2.04e+03x elapsed=0:00:06.58
```

FLAC was approximately **228,377 KiB (~223 MiB)**, versus approximately 819 MiB for original WAV. FLAC is lossless for decoded PCM. Keep the carved WAV.

### VLC/FFplay user environment and errors

```bash
vlc ~/actions-lab/artifacts/recovered_audio/full_recording.flac
```

```text
VLC media player 3.0.23 Vetinari (revision 3.0.23-2-0-g79128878dd)
```

The VLC GUI returned immediately. A verbose invocation:

```bash
vlc -vvv ~/actions-lab/artifacts/recovered_audio/full_recording.flac > /tmp/vlc-debug.log 2>&1
echo "Exit code: $?"
tail -n 40 /tmp/vlc-debug.log
```

```text
Exit code: 1
```

No additional log lines were supplied. `ffplay` seeking was offered as a workaround:

```bash
ffplay -ss 00:14:43 ~/actions-lab/artifacts/recovered_audio/full_recording.flac
ffplay -ss 00:09:45 ~/actions-lab/artifacts/recovered_audio/full_recording.flac
ffplay -showmode waves ~/actions-lab/artifacts/recovered_audio/full_recording.flac
```

Audacity was suggested for waveform inspection; installation/launch not confirmed. A full voice-activity scan remained unconfirmed. Later, a **local Mac/Apple Silicon MLX Whisper workflow was initiated**; the model download was observed, but no verified transcripts or translations have been supplied.

---

### Mac transcription follow-up — initiated; results not yet verified

After transferring a **copy** of the carved FLAC to macOS, I started an offline multilingual transcription workflow using **MLX Whisper** in a Python virtual environment. The aim was to obtain timestamped candidate text in the detected original language and an attempted English translation without sharing the original recording with a third-party transcription API.

**Execution/evidence boundary:** I confirmed a Hugging Face *model download* status message (`Downloading bytes: ... 1.52kB` and an `HF_TOKEN` unauthenticated request warning). This is consistent with fetching publicly available model weights/configuration; it is **not** a transcription result, and it does not show that a full 45-chunk run finished. No original-language transcript, English transcript, or independent human validation has been supplied for this case at the report's cutoff.

### Suggested/initiated Mac environment setup

```bash
brew install python@3.12 ffmpeg
mkdir -p ~/mp3-audio-analysis
cd ~/mp3-audio-analysis
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install mlx-whisper
python -c "import mlx_whisper; print('Whisper ready!')"
# Place the recovered FLAC copy at ~/mp3-audio-analysis/full_recording.flac
ls -lh full_recording.flac
```

### Alternative manual 5-minute segmentation commands (suggested)

```bash
mkdir -p chunks results
ffmpeg -hide_banner -y \
  -i ~/mp3-audio-analysis/full_recording.flac \
  -ac 1 -ar 16000 -c:a pcm_s16le \
  -f segment -segment_time 300 -reset_timestamps 1 \
  chunks/part_%03d.wav
```

### Single-file resumable MLX workflow (suggested; user began initial model acquisition)

The following publication-ready version of the proposed script uses automatic language detection rather than presuming a particular language. It was designed to process all ~45 five-minute intervals, store intermediate JSON, translate separately, and rebuild two complete timestamped text outputs. **Its final outputs have not been reported, and the model-generated text should not be treated as evidence until checked against the raw audio.**

```python
import json
import math
import subprocess
from pathlib import Path
import mlx_whisper

BASE = Path.home() / 'mp3-audio-analysis'
AUDIO = BASE / 'full_recording.flac'
RESULTS = BASE / 'results'
TEMP = BASE / 'temp'
MODEL = 'mlx-community/whisper-medium-mlx'
LANGUAGE = None  # Let the model attempt automatic language detection.
CHUNK_SECONDS = 300
RESULTS.mkdir(exist_ok=True)
TEMP.mkdir(exist_ok=True)

def timestamp(seconds):
    hours = int(seconds // 3600)
    minutes = int((seconds % 3600) // 60)
    remaining = seconds % 60
    return f'{hours:02}:{minutes:02}:{remaining:05.2f}'

def get_duration(path):
    output = subprocess.check_output([
        'ffprobe', '-v', 'error', '-show_entries', 'format=duration',
        '-of', 'json', str(path)
    ])
    return float(json.loads(output)['format']['duration'])

def extract_chunk(start, duration, destination):
    subprocess.run([
        'ffmpeg', '-hide_banner', '-loglevel', 'error', '-y',
        '-ss', str(start), '-i', str(AUDIO), '-t', str(duration),
        '-vn', '-ac', '1', '-ar', '16000', '-c:a', 'pcm_s16le',
        str(destination)
    ], check=True)

def save_json(path, data):
    temporary = path.with_suffix('.tmp')
    temporary.write_text(json.dumps(data, ensure_ascii=False, indent=2),
                         encoding='utf-8')
    temporary.replace(path)

if not AUDIO.exists():
    raise SystemExit(f'Audio file not found: {AUDIO}')

duration = get_duration(AUDIO)
total_chunks = math.ceil(duration / CHUNK_SECONDS)
print('Recording:', AUDIO.name)
print('Duration:', timestamp(duration))
print('Model:', MODEL)
print('Chunks:', total_chunks, flush=True)

for i in range(total_chunks):
    start = i * CHUNK_SECONDS
    length = min(CHUNK_SECONDS, duration - start)
    chunk_file = TEMP / f'chunk_{i:03}.wav'
    print(f'[{i + 1}/{total_chunks}] {timestamp(start)} to '
          f'{timestamp(start + length)}', flush=True)

    tasks = [('transcribe', 'original'), ('translate', 'english')]
    pending = [(task, label) for task, label in tasks
               if not (RESULTS / f'chunk_{i:03}_{label}.json').exists()]
    if not pending:
        print('Already completed. Skipping.', flush=True)
        continue

    extract_chunk(start, length, chunk_file)
    try:
        for task, label in pending:
            print('Running', label, flush=True)
            result = mlx_whisper.transcribe(
                str(chunk_file), path_or_hf_repo=MODEL,
                language=LANGUAGE, task=task,
                condition_on_previous_text=False, verbose=False)
            segments = [{
                'start': start + float(seg['start']),
                'end': start + float(seg['end']),
                'text': seg['text'].strip(),
            } for seg in result['segments']]
            save_json(RESULTS / f'chunk_{i:03}_{label}.json', {
                'chunk': i, 'language': result.get('language'),
                'task': task, 'segments': segments})
            print('Saved', len(segments), 'segments.', flush=True)
    finally:
        if chunk_file.exists():
            chunk_file.unlink()

for label in ['original', 'english']:
    combined = []
    for i in range(total_chunks):
        p = RESULTS / f'chunk_{i:03}_{label}.json'
        if not p.exists():
            raise RuntimeError(f'Missing result: {p}')
        combined.extend(json.loads(p.read_text(encoding='utf-8'))['segments'])
    combined.sort(key=lambda seg: seg['start'])
    with (RESULTS / f'FULL_{label}_transcript.txt').open('w', encoding='utf-8') as f:
        for seg in combined:
            f.write(f"[{timestamp(seg['start'])} --> "
                    f"{timestamp(seg['end'])}] {seg['text']}\n")
print('TRANSCRIPTION COMPLETE')
```

Suggested launch command:

```bash
cd ~/mp3-audio-analysis
source .venv/bin/activate
caffeinate -i python -u transcribe.py
```

No transcript was present in the shared case record. Further steps: review candidate speech against original WAV and have a fluent speaker assess language and transcription reliability before quoting, translating, or publishing any words.

---

### D10. WebRTC voice activity scanning and language-neutral Whisper transcription (**proposed; no output supplied**)


```bash
sudo apt install -y python3-venv
python3 -m venv ~/actions-lab/vad-venv
~/actions-lab/vad-venv/bin/pip install webrtcvad-wheels

# One-minute speech excerpt for offline ASR
mkdir -p ~/actions-lab/artifacts/transcription
ffmpeg -hide_banner -y -ss 01:00:00 -t 60 \
  -i ~/actions-lab/artifacts/recovered_audio/full_recording.flac \
  -ac 1 -ar 16000 -c:a pcm_s16le \
  ~/actions-lab/artifacts/transcription/speech_sample.wav

# Suggested local speech recognition project and multilingual model
sudo apt install -y git cmake build-essential
cd ~/actions-lab
git clone https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j 4
sh ./models/download-ggml-model.sh small

# Automatic language detection and transcription, then optional English translation
./build/bin/whisper-cli -m models/ggml-small.bin \
  -f ~/actions-lab/artifacts/transcription/speech_sample.wav \
  -l auto -otxt -of ~/actions-lab/artifacts/transcription/original_transcript

./build/bin/whisper-cli -m models/ggml-small.bin \
  -f ~/actions-lab/artifacts/transcription/speech_sample.wav \
  -l auto -tr -otxt -of ~/actions-lab/artifacts/transcription/english_translation
```

Do **not** treat ASR output on faint environmental audio as a verified transcript. Models can generate plausible text from noise, and incorrect language selection can produce confident nonsense. For any potentially consequential wording, check against the unprocessed clip and preferably a qualified native speaker.

---

</details>


## Was this recorded by the player?

The MP3 player **does have a voice-recording function**. That makes it possible—but not established—that its own microphone generated the recovered WAV. A 16 kHz PCM file and dual-mono samples are consistent with voice capture, but common enough that they don't identify the source.

A useful future test is to make a short, controlled recording on the player *after* preserving the original evidence, then compare its exact RIFF chunk layout with the recovered file. Matching the unusual **460 zero-filled `fact` bytes** and **512-byte sample offset** would be more informative than simply matching sample rate. I have **not** yet performed that reference-recording experiment.

I also checked for a date using ExifTool. The only visible file dates were from **October 11, 2026**, when I carved the file onto Kali—not when it was originally recorded. The WAV did not provide a reliable creation time, microphone identity, location, or recording-software name. Driver and installer build dates also cannot date the recording.

Crucially, **a recording missing from the music UI is not proof it was intentionally hidden**. The nominal filesystem isn't mountable. It may be unindexed content, residue from an earlier partition or provisioning process, or data copied directly into the exposed logical address space. We cannot infer secret recording behavior from its invisibility.

This difference matters to me because the question isn't only *where* the WAV lives; it's *how it got there*. If the player created it, I want to understand its recording format and indexing behavior. If it was preloaded, I want to know whether that is normal for this hardware. If it survived a reset or earlier use, I want to know what failed to erase or hide it—and whether the same thing could happen to anyone else's recordings. None of those scenarios is confirmed yet.



<details>
<summary><strong>Technical evidence &amp; commands: Possible FAT directory timestamp recovery — proposed, not run</strong></summary>

This heuristic was suggested to search for a FAT directory entry containing the WAV size. No matching original filesystem record was demonstrated.

### D9. FAT directory entry heuristic for the WAV file size (**proposed; no result supplied**)


```bash
python3 - <<'PY'
from pathlib import Path
import struct
IMG=Path('/mnt/actions-usb/actions_storage.img')
SIZE=858853888
SIGNATURE=struct.pack('<I',SIZE)
def fat_date(value):
    if value==0: return 'Not recorded'
    year=1980+((value>>9)&127)
    month=(value>>5)&15
    day=value&31
    if not (1<=month<=12 and 1<=day<=31): return 'Invalid'
    return f'{year:04}-{month:02}-{day:02}'
print('=== FAT DIRECTORY ENTRY RECOVERY ===')
print(f'Looking for file size: {SIZE:,} bytes')
matches=0; offset=0
with IMG.open('rb') as f:
    while chunk:=f.read(4*1024*1024):
        pos=0
        while True:
            pos=chunk.find(SIGNATURE,pos)
            if pos==-1: break
            start=pos-28
            if start>=0 and start%32==0:
                entry=chunk[start:start+32]
                name,attr=entry[:11],entry[11]
                if name[0] not in (0x00,0xE5) and not(attr&0x18) and not(attr&0xC0) and attr!=0x0F:
                    created=struct.unpack_from('<H',entry,16)[0]
                    modified=struct.unpack_from('<H',entry,24)[0]
                    matches+=1
                    print('Offset:',hex(offset+start),'Filename:',name,
                          'Created:',fat_date(created),'Modified:',fat_date(modified),'Attr:',hex(attr))
            pos+=1
        offset+=len(chunk)
print('Potential matching entries:',matches)
PY
```

*Forensic caveat:* This heuristic searches for a four-byte size signature and very basic FAT 32-byte directory-entry structure; a raw binary match can be coincidental, and the `start % 32` check is relative to a chunk, not an established FAT directory beginning. Any result would require full filename, attribute, cluster and directory-context validation before interpreting a purported date.

</details>


## Security implications and limitations

The part that bothers me isn't a proven malware infection—there isn't one. It's how misleading a device can be when all you see is its interface. **What unsettles me is not knowing whether these are ordinary test sounds, someone else's recoverable audio, or an artifact of how this particular device handles storage.** I want to investigate that uncertainty rather than turn it into a claim of covert surveillance.

A person can open a music library, see no files, and reasonably assume there is nothing there. But if the filesystem metadata isn't usable, **raw storage may still retain playable material that the UI never reveals**. If those bytes contain somebody else's voice, that is a privacy concern regardless of whether the file is accessible through the music player.

That's a **possible data-remanence issue**, not yet a demonstrated vendor vulnerability or seller misconduct. The device was originally marketed as new, but I'd owned it for roughly a year. I cannot presently prove whether the WAV was created during factory testing, copied as bundled material, recorded during my ownership, or left behind by an earlier user or flash component.

I have also **not** proven:

- That firmware was modified, compromised, or caused unauthorized recording.
- That the flash capacity is fake, even though its contents repeat.
- That the recovered Windows components are malicious; limited antivirus scanning reported no detections.
- That any audible language, speaker, location or original recording date is established.
- That my Mac was infected by handling the passive image or audio.

The investigation remained grounded in **read-only image analysis**. I never executed the recovered Windows binaries. The physical player wasn't attached directly to my Mac as part of the original acquisition.



<details>
<summary><strong>Technical evidence &amp; commands: Malware assessment, host exposure, and competing forensic explanations</strong></summary>

### Malware and host-exposure assessment

**Evidence against a confirmed malware finding:** ClamAV returned **zero known infections in the 113 CAB files scanned**; the long WAV contains recognizable RIFF/PCM structure, one audio stream, a zero-only `fact` chunk, and plays through common decoders; recovered Windows components plausibly fit historical MP3/MP4 companion software. **None of these proves a file safe.** A WAV could in principle exploit a media-parser vulnerability or carry nonexecuting concealed data; that is not demonstrated here.

**Exposure path** as executed:

```text
Original MP3 player --USB--> physical Kubuntu live laptop
    --dd of mass-storage LUN--> actions_storage.img on SanDisk
    --SanDisk USB passthrough--> Kali ARM64 VMware guest on Mac
    --read-only imaging/carving/static analysis--> output files and clips
```

The Mac never enumerated the **original MP3 controller** directly according to the user; the SanDisk carried a passive raw image. That substantially limits plausible USB-controller attack paths to the Mac. Risk is not literally zero because file parsers and VM/host components can have vulnerabilities, but **no host compromise was demonstrated**. Avoid running extracted Windows EXEs/OCXs/drivers; continue working on a copy and keep host tools updated.

**What cannot be claimed** from available evidence: malware infection, data exfiltration, malicious USB HID behavior, covert firmware recording, counterfeit capacity, refurbished hardware, stolen private conversations, vendor conspiracy, or firmware alteration.

---

### What explanations fit the evidence?

| Hypothesis | What supports it | What remains to establish |
|---|---|---|
| **Unindexed/residual data** from previous filesystem or provisioning | Empty UI, FAT boot sector zeros, independently carved CAB + WAV | Whether a usable filesystem existed, whether files were deleted/formatted, who wrote them |
| **Factory-loaded media/test image** | Sector-aligned WAV, Windows companion files, repeated filler; the original listing's “new” condition claim | Evidence of OEM imaging procedure or matching stock product |
| **Prior use/refurbishment** | Long environmental-sounding audio + file-system failure | Provenance; timestamps; verified previous owner or factory usage history |
| **Original player's own voice recorder** | The device offers recording; voice-oriented 16 kHz PCM; 512-byte alignment | Controlled reference recording with matching unusual RIFF header bytes |
| **Controller-generated high-entropy filler** | The same 1 MiB block across >3 GiB | Multiple independent physical reads, controller/NAND research |
| **Counterfeit or aliased capacity** | Half-capacity partition and 78.51% repetition are suspicious-looking | Actual **write/verify** capacity test; repeat data alone cannot prove |
| **Malicious firmware/covert recording** | Nothing specific yet | Code/protocol evidence of unauthorized automatic recording; malicious behavior |

**Confidence classification:**

- **Confirmed:** described USB LUN/image geometry, MBR/FAT sector anomalies, valid CAB, Actions-related Windows software, 23 plausible PE headers, repeated PE A/B bytes, extensive identical high-entropy blocks, valid/playable WAV, zero-only `fact` payload, no WAV creation date found.
- **Supported but limited:** recoverable environmental-like audio in excerpts; installer/multimedia software distribution; possible unindexed or residual content.
- **Plausible only:** sector-oriented voice recording, factory QC/test image, earlier use, deterministic flash filler, device-generated synthetic MBR.
- **Unresolved:** exact SoC, physical NAND size, firmware location, recording origin/language/date, content of the entire audio duration, whether filesystem metadata survives elsewhere.
- **Unsupported as conclusion:** secret surveillance, malware, BadUSB, counterfeit storage, seller fraud.

---

</details>


## What I learned

What started as a small firmware-modification idea turned into a much better lesson in digital forensics.

**Filesystems aren't the data.** A directory listing shows what the operating system can currently index; it is not a trustworthy inventory of every readable logical sector. Losing metadata doesn't automatically erase the content.

**File carving works—but signatures need validation.** A two-byte `MZ`, a suggestive Binwalk label, or a high entropy score is only a starting point. Valid RIFF chunks, coherent lengths, the successful CAB extraction, and identical hashed blocks told a much stronger story than the initial scan hits did.

**High entropy isn't proof of encryption.** The most random-looking region of the image turned out to be the same 1 MiB data repeated for more than 3 GiB. Statistics alone hid that fact.

**A surprising discovery doesn't automatically mean an attack.** I considered malware and malicious firmware early on. The evidence ultimately supported a far narrower claim: unexpected, recoverable audio and unusual storage structure. Not every unexplained byte is malicious.

**Privacy survives outside the UI.** The unsettling part is how easily a long recording can persist as raw data even when there is no obvious file to click. I don't know whose voices are on it or how the recording got there, and I won't treat those unknowns as an excuse to publish their conversations.

**The unanswered questions matter as much as the technical win.** Recovering the WAV was one kind of success; understanding why it existed at all is a separate forensic problem. I'm learning to be comfortable writing *unknown* when the evidence stops, and to design a better experiment instead of filling the gap with a dramatic explanation.


## What's next

**This investigation will be continued.** I'm still deeply curious about **what the voices are saying, why the recording was there in the first place, and why I couldn't see it in the interface**. The next phase is about finding answers I can actually substantiate—not uploading the audio or making sensational claims.

1. **Find out what is being said, if it is intelligible.** Continue the local MLX Whisper transcription experiment, use speech-activity detection to identify promising sections, and compare any candidate words against the unprocessed WAV. I want to work with **someone fluent in the suspected language** to verify short, non-identifying excerpts. If it cannot be reliably understood, I'll document that limitation instead of treating AI-generated text as a transcript.
2. **Investigate why the WAV ended up on the device.** Compare the carved WAV's distinctive 512-byte header and 460-byte zero-filled `fact` chunk with a controlled recording made using the player's built-in recorder **only after** preserving the existing evidence and considering the overwrite risk. Similar file structure could support a common recording implementation; it would not date the audio or identify the speaker.
3. **Understand why the UI showed nothing.** Examine any remaining FAT-style directory metadata or alternative filesystem structures without modifying the source image. Determine whether broken indexing plausibly explains the missing recording before using the word *hidden* to imply intent.
4. **Verify the storage behavior.** Re-read selected logical ranges on the separate Kubuntu machine and compare hashes against the image. Continue studying the repeated 1 MiB region; postpone destructive F3-style capacity testing until the evidence is safely preserved.
5. **Decide what responsible action makes sense.** Keep the raw audio and possible transcripts private. If evidence eventually supports a data-sanitization defect or a misrepresented device, document the specific, reproducible issue and consider asking the seller or manufacturer for clarification. I will not identify, contact, or expose a possible speaker based on speculation. I also won't reset or reformat the original player while it is still useful evidence.
6. **Return to Rockbox and firmware research.** Identify the exact Actions SoC and establish a safe recovery path before any firmware modifications. The original motivation hasn't disappeared—it has become one track of a larger investigation.

I started because I wanted a better music player. Now I have a file I wasn't expecting and more questions than answers. **The unsettling lesson is that “empty” on a device's screen does not necessarily mean empty in its storage.** Recovering the audio demonstrated that; finding out *why it was there* is the work I still want to do.

**Status: ongoing.** Any future updates will separate verified audio content, technical findings, and unresolved hypotheses.

---



<details>
<summary><strong>Technical evidence &amp; commands: Chronological command ledger, proposed tests, and read-only reproducibility checklist</strong></summary>

### Chronological command ledger and execution status

| Phase | Command / operation | Status in shared record | Result |
|---|---|---|---|
| Hardware enumeration | `lsusb`, `lsusb -v`, `sudo sg_inq /dev/sdd` | Observed/reported | `10d6:1101`, ACTIONS, HS USB FlashDisk, SCSI USB DISK FOB 2.0 |
| Acquisition | `sudo dd if=/dev/sdd ... actions_storage.img ...` | **Performed** | Exact 4,148,166,656-byte image |
| Storage mount | Read-only `mount` SanDisk at `/mnt/actions-usb` | **Performed** | Image accessible on Kali |
| Image geometry | `fdisk -l`, `xxd`, `dd` sector reads | **Performed** | Half-sized FAT32 entry, boot sector zeros |
| Loop validation | `losetup --read-only --partscan`, `fsck.vfat -n`, `mount -o ro` | **Performed** | FAT32 failed |
| Entropy | 64 KiB samples, nonzero rate, Shannon entropy, zlib ratio | **Performed** | Different zero/audio/high-entropy regions |
| Signature scan | `binwalk actions_storage.img` | **Performed** | CAB + WAV + numerous installer signatures, false-positive candidates |
| CAB carve | `dd iflag=skip_bytes,count_bytes` | **Performed** | CAB recovered |
| CAB parse | `cabextract -l`, `cabextract -d` | **Performed** | 120 entries; success |
| Windows metadata | `file`, `strings`, `exiftool`, `grep`, `find` | **Performed** | Actions/ADFU drivers, media libraries |
| AV | `clamscan -r -i ...cab_contents` | **Performed** | 0 infected / 113 files |
| PE analysis | `objdump -p ...transdll` | **Failed** | Unsupported file format on toolchain |
| PE analysis | Python `pefile` exports/imports | **Performed** | 4 exports, 11 imported DLLs |
| Signatures | Full raw `MZ`/ELF/MSCF scan | **Performed** | 52,983 / 0 / 1 |
| PE validation | PE header + optional-header checks | **Performed** | 23 plausible PE headers |
| PE hashing | SHA-256 for 8 PE instances | **Performed** | Two alternating byte-identical PE classes |
| PE extent comparison | 671,744-byte region SHA-256 | **Performed** | Regions differ after 0x36e00 bytes |
| PE carve | `dd ... executable_A.bin / B.bin` | **Performed** | A identified `DPCA.DLL` |
| Repetition checks | 64 KiB, full 1 MiB, whole-image map | **Performed** | 78.51% matches reference |
| Boundary check | 512-byte sector equality | **Performed** | Region 854 MiB onward repeats |
| WAV parse | `struct` RIFF chunks, Python `wave` | **Performed** | 3h43m39.584s PCM at 32 MiB |
| WAV carve | `dd ... skip=33554432 count=858853888` | **Performed** | Full WAV recovered |
| WAV metrics | Python sample equality/RMS/peak | **Performed** | Four 1s snippets dual mono, varying amplitudes |
| Audio clips | Python WAV crops, FFmpeg crops | **Performed** | Voice excerpts produced/uploaded |
| Enhancements | FFmpeg `highpass`, `lowpass`, `loudnorm`, `afftdn` | **Performed/referenced** | Filtered audio; not source evidence |
| FLAC | `ffmpeg -i full_recording.wav -c:a flac full_recording.flac` | **Performed** | ~228,377 KiB FLAC, completes |
| UI player | `vlc`, `vlc -vvv` | **Failed** | Exit code 1; no debug log lines shown |
| `ffplay` seek | `ffplay -ss ...` | Suggested workaround | Actual use not established |
| WAV metadata | `ffprobe`, `exiftool -G1 -a -s` | **Performed** | No original date; normal PCM |
| WAV padding | Python byte dump offset 44 length 460 | **Performed** | All 460 bytes zero |
| Date recovery | FAT directory-entry file-size scanner | **Suggested, no output** | Not confirmed run |
| Test recording | `ffprobe ... test_recording.wav` | **Failed** | File did not exist; no test capture |
| VAD | WebRTC VAD + `ffmpeg` pipe script | **Suggested, no output** | Not confirmed installed/executed |
| Transcription | `whisper.cpp` download/build, `whisper-cli -l auto/-tr` | **Suggested, no output** | Alternative to the Mac MLX workflow; no transcript verified |
| Recovery | TestDisk `Analyse → Quick Search` | **Suggested** | No recovery result supplied |
| Capacity check | F3/F3probe on physical device | **Suggested only** | Not run; some tests destructive |
| Date/metadata | `exiftool full_recording.wav` | **Performed** | Extraction-date timestamps only |
| Malware WAV | `clamscan ...full_recording.wav/.flac` | **Suggested; no results** | Do not claim scan was clean |

### Full code inventory for the most important analyses

The exact *operative* Python source or reproducible equivalents is included in the sections above. Additional suggested tooling/workflows not confirmed executed:

```bash
# File system: read-only search (do NOT select Write)
sudo apt install testdisk
testdisk /mnt/actions-usb/actions_storage.img

# Check for known malware in carved audio (no output supplied)
clamscan -i ~/actions-lab/artifacts/recovered_audio/full_recording.wav
clamscan -i ~/actions-lab/artifacts/recovered_audio/full_recording.flac

# Inspect metadata (performed)
exiftool -G1 -a -s ~/actions-lab/artifacts/recovered_audio/full_recording.wav

# Optional GUI app suggested to navigate audio
sudo apt install audacity

# A possible source VAD command environment setup (NOT confirmed run)
python3 -m venv ~/actions-lab/vad-venv
~/actions-lab/vad-venv/bin/pip install webrtcvad-wheels

# An alternative multilingual ASR setup (NOT confirmed run)
git clone https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j 4
sh ./models/download-ggml-model.sh small
# whisper-cli -m models/ggml-small.bin -f some_clip.wav -l auto -otxt
# whisper-cli -m models/ggml-small.bin -f some_clip.wav -l auto -tr -otxt
```

These optional snippets are included as provenance of the *suggested research trajectory*, not as experiments that have produced results.

---

### Evidence preservation and next steps

**Preserve what exists:** Store the original `actions_storage.img` read-only, calculate SHA-256 (`sha256sum`), maintain a copy with checksum and collection notes; hash carved CAB, WAV, FLAC and the selected PE binaries, keeping the original WAV unchanged. The date/times on the carved file reflect extraction and must not be described as recording timestamps.

**Most discriminating next experiments (order matters):**

1. **Reference recording comparison:** Once the physical evidence is safely preserved and further writes accepted, make a controlled ~10-second recording on the MP3 player with its built-in recorder; acquire that file through separate Kubuntu and compare **exact RIFF chunk order, 460-byte zero-filled `fact`, data offset 512, sample rate and channel duplication**. Avoid assuming matching generic PCM format proves same recorder.
2. **Physical stability check:** Reconnect the original player only to the separate Kubuntu laptop; read selected logical ranges twice (read-only), hash the outputs and compare to the saved image. Stable repeated data is not proof of fake capacity; instability may indicate read errors/controller behavior.
3. **Capacity validation, later and destructive:** If appropriate after preservation, test actual writable flash capacity with F3 or equivalent on the physical device. Do not perform destructive probing while evidence preservation is a priority.
4. **Further offline search:** Check TestDisk for alternative filesystem structures; inspect FAT-style directory entries, recognizing that file-size-match searching has a high false-positive rate and cannot independently authenticate a date.
5. **Audio forensics:** Continue the initiated local transcription experiment; use human language review, and run local voice activity detection and extract *time-indexed* candidate sections; use multilingual ASR only as a lead and manually verify against raw audio. Prefer transcribing only the minimum necessary for provenance, with privacy protections.
6. **Firmware separate track:** Identify exact chip/ADFU protocol and a hardware recovery approach before vendor commands. Avoid flashing/erasing with an unidentified device.

**Defensible publication wording:**

> “A raw USB mass-storage image from a generic Actions-based MP3 player sold as new contained an unmountable nominal FAT32 partition, a recoverable Windows media/driver software archive, a playable 3h43m PCM WAV carved from a sector-aligned offset, and more than 3 GiB of repeated data. The recording was not visible through the music interface. Origin and date of the audio, NAND capacity, firmware integrity, and possible prior use are not established.”

**Never publish an identifiable stranger's voice, inferred nationality, personal conversation, or unverified seller allegations.** If releasing research, publish anonymized waveforms/short synthetic replacements, offsets, hashes and reproducible procedures instead.

---


---

### Appendix D — How to reproduce key findings without reconnecting the device

The following are **read-only** on the original image, although each parser still introduces software-level exposure to untrusted data. Paths depend on maintaining the same mount and directories.

```bash
IMG=/mnt/actions-usb/actions_storage.img

# Device-image size and independent checksum on the already-acquired backup
stat -c '%n %s bytes' "$IMG"
sha256sum "$IMG" | tee ~/actions-lab/artifacts/actions_storage.sha256

# Partition structure; FAT32 at LBA 128 is all zeros
fdisk -l "$IMG"
dd if="$IMG" bs=512 skip=128 count=1 status=none | xxd -g1

# Validated archive and audio signatures
xxd -g1 -s 8392704 -l 16 "$IMG"
xxd -g1 -s 33554432 -l 64 "$IMG"

# Carved file geometry and audio properties
stat -c '%n %s bytes' ~/actions-lab/artifacts/recovered_audio/full_recording.wav
ffprobe -v error -show_entries format=duration,size -of default=noprint_wrappers=1 \
 ~/actions-lab/artifacts/recovered_audio/full_recording.wav
exiftool -G1 -a -s ~/actions-lab/artifacts/recovered_audio/full_recording.wav

# Reinspect source CAB and executable headers
cabextract -l ~/actions-lab/artifacts/suspected.cab | tail -n 3
file ~/actions-lab/artifacts/pe_candidates/executable_A.bin
```

**Preservation requirement:** Commands here write only checksum/log output in the analysis workspace, not the physical device or raw image. Still retain an unmodified copy of `actions_storage.img` and record exactly which filesystem, drive and OS version were used. To demonstrate that the image equals what the physical player exposes today, one would need a *fresh, independent* read from the physical device; this remains unperformed.

</details>


## References and responsible evidence handling

**The complete technical command/output record is integrated into this README.** Expand the panels alongside each investigative phase. Appendix-style raw Binwalk outputs are preserved at the end. Experiments that were suggested but not performed remain explicitly marked as such.

**External projects referenced** (their support for similar devices does not identify this player's exact chipset):

- [Rockbox](https://www.rockbox.org/) — the original custom-firmware inspiration and [supported-port list](https://www.rockbox.org/).
- [Hannibal-pl/usbfw](https://github.com/Hannibal-pl/usbfw) — Actions device tooling; compiled locally, **not** used to dump or flash this player.
- [ilyakurdyukov/actions_flash](https://github.com/ilyakurdyukov/actions_flash) — ATJ2127/ATJ2157-related research; shared `10d6:1101` normal-mode ID is **not** proof of the exact SoC.
- [MLX Whisper](https://github.com/ml-explore/mlx-examples/tree/main/whisper) — local multilingual ASR work in progress; no validated transcription reported.

### Responsible disclosure / evidence handling

No original voice audio, complete raw image, identifying transcript, or recovered executable binary is bundled with this public-ready Markdown. Those artifacts may contain sensitive content, have uncertain provenance, or need further review. Reported hashes, excerpts of tool output, and methods are provided so the technical process can be inspected without exposing private speech. The image preserves USB-accessible **logical storage**, not a verified internal firmware dump.

*Case status: open. Findings and interpretations may be updated as additional evidence is collected.*



---

## Raw Binwalk logs (verbatim)

The following two scans were captured during the investigation. **Binwalk signatures are candidates, not automatic proof** that a device contains any named firmware family. They are included for auditability, not as verified claims.



<details>
<summary><strong>Technical evidence &amp; commands: Original complete Binwalk scan files, appendices A and B</strong></summary>

### Original uploaded Binwalk scan outputs

Both scan captures are preserved as originally uploaded. Binwalk results include many candidate signatures that were not independently validated; archive and WAV offsets cited in the main report **were** separately checked. Historical logs and program metadata are not recording timestamps.

### Appendix A — Original Binwalk scan text, attachment `Pasted text(4).txt`

> **Verbatim original uploaded text.** Some lines were already abbreviated/truncated in the source attachment; those remain unchanged. This text is preserved for audit, not treated as an independently validated signature list.

~~~~~text
CRIPTION
--------------------------------------------------------------------------------
8392704       0x801000        Microsoft Cabinet archive data, 10071697 bytes, 120 files
9437184       0x900000        RIFF audio data (WAV), PCM, 2 channels, 16000 sample rate
10764086      0xA43F36        JBOOT STAG header, image id: 15, timestamp 0x9BE00F5D, image size: 2272928492 bytes, image JBOOT checksum: 0xE175, header JBOOT checksum: 0xDA13
18489123      0x11A1F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
18489280      0x11A1FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
18521715      0x11A9E73       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
18526208      0x11AB000       XML document, version: "1.0"
18542080      0x11AEE00       Microsoft executable, portable (PE)
18563468      0x11B418C       XML document, version: "1.0"
18789888      0x11EB600       PC bitmap, Windows 3.x format,, 500 x 70 x 8
18856448      0x11FBA00       Microsoft executable, portable (PE)
18919289      0x120AF79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
19160867      0x1245F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
19161024      0x1245FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
19193459      0x124DE73       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
19197952      0x124F000       XML document, version: "1.0"
19213824      0x1252E00       Microsoft executable, portable (PE)
19235212      0x125818C       XML document, version: "1.0"
19453577      0x128D689       eCos RTOS string reference: "eCostAvailableDifferenceVolumeCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1D"
19453606      0x128D6A6       eCos RTOS string reference: "eCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07"
19453634      0x128D6C2       eCos RTOS string reference: "eCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionF"
19453661      0x128D6DD       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
19453681      0x128D6F1       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Please u"
19455506      0x128DE12       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
19455616      0x128DE80       eCos RTOS string reference: "eCostList{\VSI_MS_Sans_Serif13.0_0_0}The list below includes the drives you can install [ProductName] to, along with each drive'"
19459805      0x128EEDD       Copyright string: "copyright law and international treaties. Unauthorized duplication or distribution of this program, or any portion of it, may re"
19460045      0x128EFCD       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}The installer will guide you through the steps required to install [ProductName]"
19463168      0x128FC00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
19530752      0x12A0400       Microsoft executable, portable (PE)
19593593      0x12AF979       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
19832611      0x12E9F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
19832768      0x12E9FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
19865207      0x12F1E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
19869696      0x12F3000       XML document, version: "1.0"
19885568      0x12F6E00       Microsoft executable, portable (PE)
19906956      0x12FC18C       XML document, version: "1.0"
20111436      0x132E04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
20127583      0x1331F5F       eCos RTOS string reference: "eCostDifferenceRequisVolumeCostRequiredTaille du disqueVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B"
20127609      0x1331F79       eCos RTOS string reference: "eCostRequiredTaille du disqueVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_"
20127643      0x1331F9B       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
20127663      0x1331FAF       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilisez"
20128457      0x13322C9       eCos RTOS string reference: "eCostList1{\MS_Sans_Serif__8_0}OKOKButtonDiskCost{\MS_Sans_Serif__8_0}{116}{80}{80}{80}{80}VolumeCostList{\MS_Sans_Serif__8_0}La"
20136448      0x1334200       PC bitmap, Windows 3.x format,, 500 x 70 x 8
20204544      0x1344C00       Microsoft executable, portable (PE)
20267385      0x1354179       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
20504355      0x138DF23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
20504512      0x138DFC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
20536951      0x1395E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
20541440      0x1397000       XML document, version: "1.0"
20557312      0x139AE00       Microsoft executable, portable (PE)
20578700      0x13A018C       XML document, version: "1.0"
20783180      0x13D204C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
20797513      0x13D5849       eCos RTOS string reference: "eCostSizeLaufwerkVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_"
20799415      0x13D5FB7       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
20804183      0x13D7257       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}Der Installer wird Sie durch die zur Installation von [ProductName] erforderlich"
20807168      0x13D7E00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
20874752      0x13E8600       Microsoft executable, portable (PE)
20937593      0x13F7B79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
21176099      0x1431F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
21176256      0x1431FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
21208695      0x1439E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
21213184      0x143B000       XML document, version: "1.0"
21229056      0x143EE00       Microsoft executable, portable (PE)
21250444      0x144418C       XML document, version: "1.0"
21454924      0x147604C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
21470874      0x1479E9A       eCos RTOS string reference: "eCostAvailableDifferenzaVolumeCostDifferenceRichiestoVolumeCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDia"
21470903      0x1479EB7       eCos RTOS string reference: "eCostDifferenceRichiestoVolumeCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_"
21470932      0x1479ED4       eCos RTOS string reference: "eCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_A"
21470965      0x1479EF5       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
21470985      0x1479F09       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilizza"
21471724      0x147A1EC       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
21475964      0x147B27C       Copyright string: "copyright, dalle leggi sui diritti d'autore e dalle disposizioni dei trattati internazionali. La riproduzione o distribuzione no"
21476243      0x147B393       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}Seguire la procedura di installazione guidata per installare [ProductName] nel s"
21479424      0x147C000       PC bitmap, Windows 3.x format,, 500 x 70 x 8
21547008      0x148C800       Microsoft executable, portable (PE)
21609849      0x149BD79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
21847843      0x14D5F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
21848000      0x14D5FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
21880439      0x14DDE77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
21884928      0x14DF000       XML document, version: "1.0"
21900800      0x14E2E00       Microsoft executable, portable (PE)
21922188      0x14E818C       XML document, version: "1.0"
22126668      0x151A04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
22140899      0x151D7E3       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}[Product"
22143117      0x151E08D       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
22150656      0x151FE00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
22218240      0x1530600       Microsoft executable, portable (PE)
22281081      0x153FB79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
22519587      0x1579F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
22519744      0x1579FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
22552183      0x1581E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
22556672      0x1583000       XML document, version: "1.0"
22572544      0x1586E00       Microsoft executable, portable (PE)
22593932      0x158C18C       XML document, version: "1.0"
22798412      0x15BE04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
22812055      0x15C1597       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}.NET Fra"
22821888      0x15C3C00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
22888960      0x15D4200       Microsoft executable, portable (PE)
22951801      0x15E3779       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
23191331      0x161DF23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
23191488      0x161DFC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
23223924      0x1625E74       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
23228416      0x1627000       XML document, version: "1.0"
23244288      0x162AE00       Microsoft executable, portable (PE)
23265676      0x163018C       XML document, version: "1.0"
23470156      0x166204C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
23484037      0x1665685       eCos RTOS string reference: "eCostAvailableDifferenceVolumeCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1D"
23484066      0x16656A2       eCos RTOS string reference: "eCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07"
23484094      0x16656BE       eCos RTOS string reference: "eCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionF"
23484121      0x16656D9       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
23484141      0x16656ED       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Please u"
23485966      0x1665E0E       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
23486076      0x1665E7C       eCos RTOS string reference: "eCostList{\VSI_MS_Sans_Serif13.0_0_0}The list below includes the drives you can install [ProductName] to, along with each drive'"
23490265      0x1666ED9       Copyright string: "copyright law and international treaties. Unauthorized duplication or distribution of this program, or any portion of it, may re"
23490505      0x1666FC9       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}The installer will guide you through the steps required to install [ProductName]"
23493632      0x1667C00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
23561216      0x1678400       Microsoft executable, portable (PE)
23624057      0x1687979       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
23863075      0x16C1F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
23863232      0x16C1FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
23895671      0x16C9E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
23900160      0x16CB000       XML document, version: "1.0"
23916032      0x16CEE00       Microsoft executable, portable (PE)
23937420      0x16D418C       XML document, version: "1.0"
24141900      0x170604C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
24156387      0x17098E3       eCos RTOS string reference: "eCostAvailableDiferenciaVolumeCostDifferenceRequeridoVolumeCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdD"
24156416      0x1709900       eCos RTOS string reference: "eCostDifferenceRequeridoVolumeCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B"
24156445      0x170991D       eCos RTOS string reference: "eCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm"
24156479      0x170993F       eCos RTOS string reference: "eCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_S"
24156500      0x1709954       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilice "
24158292      0x170A054       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}AceptarOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VS"
24165888      0x170BE00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
24233472      0x171C600       Microsoft executable, portable (PE)
24296313      0x172BB79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
24534819      0x1765F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
24534976      0x1765FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
24567415      0x176DE77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
24571904      0x176F000       XML document, version: "1.0"
24587776      0x1772E00       Microsoft executable, portable (PE)
24609164      0x177818C       XML document, version: "1.0"
24813644      0x17AA04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
24835584      0x17AF600       PC bitmap, Windows 3.x format,, 500 x 70 x 8
24846850      0x17B2202       ESP Image (ESP32): segment count: 5, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x0, hash: none
24899042      0x17BEDE2       ESP Image (ESP32): segment count: 5, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x0, hash: none
24902144      0x17BFA00       Microsoft executable, portable (PE)
24964985      0x17CEF79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
25182208      0x1804000       Microsoft executable, portable (PE)
25443770      0x1843DBA       JBOOT STAG header, image id: 0, timestamp 0x1E0043, image size: 723517440 bytes, image JBOOT checksum: 0x43, header JBOOT checksum: 0x12
25829376      0x18A2000       Microsoft executable, portable (PE)
26005504      0x18CD000       Microsoft executable, portable (PE)
26034768      0x18D4250       ESP Image (ESP32): segment count: 3, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x81ffff, hash: none
33554432      0x2000000       RIFF audio data (WAV), PCM, 2 channels, 16000 sample rate
51118666      0x30C024A       ESP Image (ESP32-P4): segment count: 2, flash mode: QUOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0x10e0201, hash: none
100234506     0x5F9750A       ESP Image (ESP32): segment count: 2, flash mode: QUIO, flash speed: 20MHz, flash size: 1MB, entry address: 0xde0200, hash: none
125814326     0x77FC636       ESP Image (ESP32-S3): segment count: 1, flash mode: SLOW_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x210105, hash: sha256
158421966     0x97153CE       ESP Image segment count: 2, flash mode: DOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0xfb0203, hash: none
189522114     0xB4BE0C2       ESP Image (ESP32-C6): segment count: 2, flash mode: DOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0xcc0203, hash: none
215004322     0xCD0B4A2       ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x9f0104, hash: none
215239782     0xCD44C66       MySQL MISAM index file Version 1
262609742     0xFA71B4E       MySQL ISAM compressed data file Version 1
270282710     0x101C2FD6      ESP Image segment count: 1, flash mode: QUOUT, flash speed: 26MHz, flash size: 1MB, entry address: 0x5c0101, hash: sha256
300302546     0x11E640D2      ESP Image (ESP32-P4): segment count: 1, flash mode: SLOW_READ, flash speed: 20MHz, flash size: 1MB, entry address: 0x1170205, hash: none
325203170     0x136234E2      MySQL ISAM compressed data file Version 1
342155266     0x1464E002      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x440104, hash: none
357113158     0x15491D46      MySQL MISAM index file Version 1
377568486     0x16813CE6      Microware OS-9/68000 module, Revision: -20484, Size: -1342396163, Group ID: -22275, User ID: -3586, Name:
378770834     0x16939592      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x250104, hash: none
385427302     0x16F92766      MySQL MISAM index file Version 1
398587118     0x17C1F4EE      ESP Image segment count: 1, flash mode: SLOW_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0xb0105, hash: none
405309494     0x18288836      MySQL ISAM compressed data file Version 1
409979926     0x186FCC16      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x3f0104, hash: none
417420806     0x18E15606      MySQL MISAM compressed data file Version 1
425940294     0x19635546      ESP Image (ESP32-P4): segment count: 2, flash mode: DIO, flash speed: 20MHz, flash size: 1MB, entry address: 0xc30202, hash: sha256
433978214     0x19DDFB66      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x3f0104, hash: none
449152698     0x1AC586BA      MySQL ISAM compressed data file Version 1
466458906     0x1BCD991A      MySQL MISAM index file Version 1
~~~~~

### Appendix B — Original Binwalk scan text, attachment `Pasted text(5).txt`

> **Verbatim original uploaded text.** Some lines were already abbreviated/truncated in the source attachment; those remain unchanged. This text is preserved for audit, not treated as an independently validated signature list.

~~~~~text
e.img

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
8392704       0x801000        Microsoft Cabinet archive data, 10071697 bytes, 120 files
9437184       0x900000        RIFF audio data (WAV), PCM, 2 channels, 16000 sample rate
10764086      0xA43F36        JBOOT STAG header, image id: 15, timestamp 0x9BE00F5D, image size: 2272928492 bytes, image JBOOT checksum: 0xE175, header JBOOT checksum: 0xDA13
18489123      0x11A1F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
18489280      0x11A1FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
18521715      0x11A9E73       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
18526208      0x11AB000       XML document, version: "1.0"
18542080      0x11AEE00       Microsoft executable, portable (PE)
18563468      0x11B418C       XML document, version: "1.0"
18789888      0x11EB600       PC bitmap, Windows 3.x format,, 500 x 70 x 8
18856448      0x11FBA00       Microsoft executable, portable (PE)
18919289      0x120AF79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
19160867      0x1245F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
19161024      0x1245FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
19193459      0x124DE73       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
19197952      0x124F000       XML document, version: "1.0"
19213824      0x1252E00       Microsoft executable, portable (PE)
19235212      0x125818C       XML document, version: "1.0"
19453577      0x128D689       eCos RTOS string reference: "eCostAvailableDifferenceVolumeCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1D"
19453606      0x128D6A6       eCos RTOS string reference: "eCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07"
19453634      0x128D6C2       eCos RTOS string reference: "eCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionF"
19453661      0x128D6DD       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
19453681      0x128D6F1       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Please u"
19455506      0x128DE12       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
19455616      0x128DE80       eCos RTOS string reference: "eCostList{\VSI_MS_Sans_Serif13.0_0_0}The list below includes the drives you can install [ProductName] to, along with each drive'"
19459805      0x128EEDD       Copyright string: "copyright law and international treaties. Unauthorized duplication or distribution of this program, or any portion of it, may re"
19460045      0x128EFCD       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}The installer will guide you through the steps required to install [ProductName]"
19463168      0x128FC00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
19530752      0x12A0400       Microsoft executable, portable (PE)
19593593      0x12AF979       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
19832611      0x12E9F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
19832768      0x12E9FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
19865207      0x12F1E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
19869696      0x12F3000       XML document, version: "1.0"
19885568      0x12F6E00       Microsoft executable, portable (PE)
19906956      0x12FC18C       XML document, version: "1.0"
20111436      0x132E04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
20127583      0x1331F5F       eCos RTOS string reference: "eCostDifferenceRequisVolumeCostRequiredTaille du disqueVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B"
20127609      0x1331F79       eCos RTOS string reference: "eCostRequiredTaille du disqueVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_"
20127643      0x1331F9B       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
20127663      0x1331FAF       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilisez"
20128457      0x13322C9       eCos RTOS string reference: "eCostList1{\MS_Sans_Serif__8_0}OKOKButtonDiskCost{\MS_Sans_Serif__8_0}{116}{80}{80}{80}{80}VolumeCostList{\MS_Sans_Serif__8_0}La"
20136448      0x1334200       PC bitmap, Windows 3.x format,, 500 x 70 x 8
20204544      0x1344C00       Microsoft executable, portable (PE)
20267385      0x1354179       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
20504355      0x138DF23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
20504512      0x138DFC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
20536951      0x1395E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
20541440      0x1397000       XML document, version: "1.0"
20557312      0x139AE00       Microsoft executable, portable (PE)
20578700      0x13A018C       XML document, version: "1.0"
20783180      0x13D204C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
20797513      0x13D5849       eCos RTOS string reference: "eCostSizeLaufwerkVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_"
20799415      0x13D5FB7       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
20804183      0x13D7257       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}Der Installer wird Sie durch die zur Installation von [ProductName] erforderlich"
20807168      0x13D7E00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
20874752      0x13E8600       Microsoft executable, portable (PE)
20937593      0x13F7B79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
21176099      0x1431F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
21176256      0x1431FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
21208695      0x1439E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
21213184      0x143B000       XML document, version: "1.0"
21229056      0x143EE00       Microsoft executable, portable (PE)
21250444      0x144418C       XML document, version: "1.0"
21454924      0x147604C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
21470874      0x1479E9A       eCos RTOS string reference: "eCostAvailableDifferenzaVolumeCostDifferenceRichiestoVolumeCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDia"
21470903      0x1479EB7       eCos RTOS string reference: "eCostDifferenceRichiestoVolumeCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_"
21470932      0x1479ED4       eCos RTOS string reference: "eCostRequiredSpazio su discoVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_A"
21470965      0x1479EF5       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
21470985      0x1479F09       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilizza"
21471724      0x147A1EC       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
21475964      0x147B27C       Copyright string: "copyright, dalle leggi sui diritti d'autore e dalle disposizioni dei trattati internazionali. La riproduzione o distribuzione no"
21476243      0x147B393       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}Seguire la procedura di installazione guidata per installare [ProductName] nel s"
21479424      0x147C000       PC bitmap, Windows 3.x format,, 500 x 70 x 8
21547008      0x148C800       Microsoft executable, portable (PE)
21609849      0x149BD79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
21847843      0x14D5F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
21848000      0x14D5FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
21880439      0x14DDE77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
21884928      0x14DF000       XML document, version: "1.0"
21900800      0x14E2E00       Microsoft executable, portable (PE)
21922188      0x14E818C       XML document, version: "1.0"
22126668      0x151A04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
22140899      0x151D7E3       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}[Product"
22143117      0x151E08D       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
22150656      0x151FE00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
22218240      0x1530600       Microsoft executable, portable (PE)
22281081      0x153FB79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
22519587      0x1579F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
22519744      0x1579FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
22552183      0x1581E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
22556672      0x1583000       XML document, version: "1.0"
22572544      0x1586E00       Microsoft executable, portable (PE)
22593932      0x158C18C       XML document, version: "1.0"
22798412      0x15BE04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
22812055      0x15C1597       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}.NET Fra"
22821888      0x15C3C00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
22888960      0x15D4200       Microsoft executable, portable (PE)
22951801      0x15E3779       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
23191331      0x161DF23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
23191488      0x161DFC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
23223924      0x1625E74       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
23228416      0x1627000       XML document, version: "1.0"
23244288      0x162AE00       Microsoft executable, portable (PE)
23265676      0x163018C       XML document, version: "1.0"
23470156      0x166204C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
23484037      0x1665685       eCos RTOS string reference: "eCostAvailableDifferenceVolumeCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1D"
23484066      0x16656A2       eCos RTOS string reference: "eCostDifferenceRequiredVolumeCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07"
23484094      0x16656BE       eCos RTOS string reference: "eCostRequiredDisk SizeVolumeCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionF"
23484121      0x16656D9       eCos RTOS string reference: "eCostSizeVolumeVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Se"
23484141      0x16656ED       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Please u"
23485966      0x1665E0E       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}OKOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VSI_MS_"
23486076      0x1665E7C       eCos RTOS string reference: "eCostList{\VSI_MS_Sans_Serif13.0_0_0}The list below includes the drives you can install [ProductName] to, along with each drive'"
23490265      0x1666ED9       Copyright string: "copyright law and international treaties. Unauthorized duplication or distribution of this program, or any portion of it, may re"
23490505      0x1666FC9       Copyright string: "CopyrightWarningText{\VSI_MS_Sans_Serif13.0_0_0}The installer will guide you through the steps required to install [ProductName]"
23493632      0x1667C00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
23561216      0x1678400       Microsoft executable, portable (PE)
23624057      0x1687979       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
23863075      0x16C1F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
23863232      0x16C1FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
23895671      0x16C9E77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
23900160      0x16CB000       XML document, version: "1.0"
23916032      0x16CEE00       Microsoft executable, portable (PE)
23937420      0x16D418C       XML document, version: "1.0"
24141900      0x170604C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
24156387      0x17098E3       eCos RTOS string reference: "eCostAvailableDiferenciaVolumeCostDifferenceRequeridoVolumeCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdD"
24156416      0x1709900       eCos RTOS string reference: "eCostDifferenceRequeridoVolumeCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B"
24156445      0x170991D       eCos RTOS string reference: "eCostRequiredEspacio en discoVolumeCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm"
24156479      0x170993F       eCos RTOS string reference: "eCostSizeVolumenVolumeCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_S"
24156500      0x1709954       eCos RTOS string reference: "eCostVolumeVsdDialogs.1DB77F5A_BA5C_4470_89B6_0B0EC07E3A10MaintenanceForm_ActionFinishedForm{\VSI_MS_Sans_Serif13.0_0_0}Utilice "
24158292      0x170A054       eCos RTOS string reference: "eCostList1{\VSI_MS_Sans_Serif13.0_0_0}AceptarOKButtonDiskCost{\VSI_MS_Sans_Serif13.0_0_0}{116}{80}{80}{80}{80}VolumeCostList{\VS"
24165888      0x170BE00       PC bitmap, Windows 3.x format,, 500 x 70 x 8
24233472      0x171C600       Microsoft executable, portable (PE)
24296313      0x172BB79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
24534819      0x1765F23       eCos RTOS string reference: "eCostReserve a specified amount of space if this component is to be installed.ReserveFolderReserveKeyPrimary key that uniquely i"
24534976      0x1765FC0       eCos RTOS string reference: "eCost recordReserveLocalDisk space to reserve if linked component is installed locally.ReserveSourceDisk space to reserve if lin"
24567415      0x176DE77       eCos RTOS string reference: "eCostRedirectedDllSupportIsolateComponentsCostFinalizeSetODBCFoldersInstallValidateInstallInitializeAllocateRegistrySpaceProcess"
24571904      0x176F000       XML document, version: "1.0"
24587776      0x1772E00       Microsoft executable, portable (PE)
24609164      0x177818C       XML document, version: "1.0"
24813644      0x17AA04C       ESP Image (ESP32): segment count: 4, flash mode: QUIO, flash size: 1MB, entry address: 0x4700502, hash: none
24835584      0x17AF600       PC bitmap, Windows 3.x format,, 500 x 70 x 8
24846850      0x17B2202       ESP Image (ESP32): segment count: 5, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x0, hash: none
24899042      0x17BEDE2       ESP Image (ESP32): segment count: 5, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x0, hash: none
24902144      0x17BFA00       Microsoft executable, portable (PE)
24964985      0x17CEF79       ESP Image (ESP32-S2): segment count: 15, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x6a406a00, hash: none
25182208      0x1804000       Microsoft executable, portable (PE)
25443770      0x1843DBA       JBOOT STAG header, image id: 0, timestamp 0x1E0043, image size: 723517440 bytes, image JBOOT checksum: 0x43, header JBOOT checksum: 0x12
25829376      0x18A2000       Microsoft executable, portable (PE)
26005504      0x18CD000       Microsoft executable, portable (PE)
26034768      0x18D4250       ESP Image (ESP32): segment count: 3, flash mode: QUIO, flash speed: 40MHz, flash size: 1MB, entry address: 0x81ffff, hash: none
33554432      0x2000000       RIFF audio data (WAV), PCM, 2 channels, 16000 sample rate
51118666      0x30C024A       ESP Image (ESP32-P4): segment count: 2, flash mode: QUOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0x10e0201, hash: none
100234506     0x5F9750A       ESP Image (ESP32): segment count: 2, flash mode: QUIO, flash speed: 20MHz, flash size: 1MB, entry address: 0xde0200, hash: none
125814326     0x77FC636       ESP Image (ESP32-S3): segment count: 1, flash mode: SLOW_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x210105, hash: sha256
158421966     0x97153CE       ESP Image segment count: 2, flash mode: DOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0xfb0203, hash: none
189522114     0xB4BE0C2       ESP Image (ESP32-C6): segment count: 2, flash mode: DOUT, flash speed: 20MHz, flash size: 1MB, entry address: 0xcc0203, hash: none
215004322     0xCD0B4A2       ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x9f0104, hash: none
215239782     0xCD44C66       MySQL MISAM index file Version 1
262609742     0xFA71B4E       MySQL ISAM compressed data file Version 1
270282710     0x101C2FD6      ESP Image segment count: 1, flash mode: QUOUT, flash speed: 26MHz, flash size: 1MB, entry address: 0x5c0101, hash: sha256
300302546     0x11E640D2      ESP Image (ESP32-P4): segment count: 1, flash mode: SLOW_READ, flash speed: 20MHz, flash size: 1MB, entry address: 0x1170205, hash: none
325203170     0x136234E2      MySQL ISAM compressed data file Version 1
342155266     0x1464E002      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x440104, hash: none
357113158     0x15491D46      MySQL MISAM index file Version 1
377568486     0x16813CE6      Microware OS-9/68000 module, Revision: -20484, Size: -1342396163, Group ID: -22275, User ID: -3586, Name:
378770834     0x16939592      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x250104, hash: none
385427302     0x16F92766      MySQL MISAM index file Version 1
398587118     0x17C1F4EE      ESP Image segment count: 1, flash mode: SLOW_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0xb0105, hash: none
405309494     0x18288836      MySQL ISAM compressed data file Version 1
409979926     0x186FCC16      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x3f0104, hash: none
417420806     0x18E15606      MySQL MISAM compressed data file Version 1
425940294     0x19635546      ESP Image (ESP32-P4): segment count: 2, flash mode: DIO, flash speed: 20MHz, flash size: 1MB, entry address: 0xc30202, hash: sha256
433978214     0x19DDFB66      ESP Image segment count: 1, flash mode: FAST_READ, flash speed: 26MHz, flash size: 1MB, entry address: 0x3f0104, hash: none
449152698     0x1AC586BA      MySQL ISAM compressed data file Version 1
466458906     0x1BCD991A      MySQL MISAM index file Version 1
501253334     0x1DE084D6      ESP Image segment count: 1, flash mode: QUIO, flash speed: 26MHz, flash size: 1MB, entry address: 0x310100, hash: none
529947510     0x1F965B76      Microware OS-9/68000 module, Revision: 17917, Size: 1174250494, Group ID: -23554, User ID: 5631, Name:
~~~~~

</details>



---

*Investigation ongoing. I'm continuing to investigate the voices, the recording's origin, and why the audio was absent from the interface. The recording and any speech transcription are deliberately not included in this public Markdown. The work documents recoverable USB-addressable storage, **not** extracted internal firmware.*
