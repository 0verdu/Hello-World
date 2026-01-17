# FIRMWARE EVIDENCE - SoC_RAM.bin

## FILE METADATA
File: SoC_RAM.bin
Size: 2097152 bytes (2.0M)

## CRITICAL OFFSETS

### 1. LPM SCAN CONFIGURATION TABLE
Offset: 0x190340
Pattern: Repeated 20+ times, 32 bytes per entry

Entry Structure:
  +0x00: Magic (00 00 04 c0)
  +0x04: RSSI threshold (b5 = -75 dBm)
  +0x08: Function pointer
  +0x0C: Flags (00 00 00 70)
  +0x10-0x1F: Additional config

HEX DUMP (0x190340):
00 00 04 c0 b5 00 00 00 78 97 87 00 00 00 00 70
00 00 00 00 34 00 00 00 10 38 a0 06 00 01 00 f0

SIGNIFICANCE: 
This table is NOT cleared during radio-disablement phase.
RSSI value 0xB5 persists, causing immediate scan resume.

### 2. CONNECTION FLAGS (FL/CF)
Offset: 0x0cdd30
Context: Embedded in ARM Thumb code

HEX DUMP (0x0cdd30):
01 00 24 00 01 00 44 00 f1 d8 ff ff 10 b5 1c 46
40 f2 1a 33 98 42 25 d1 93 88 c9 68 db 43 03 f0

DECODED:
  01 00      = FL register (0x01 = UNAUTHENTICATED)
  24 00      = CF register (0x24 = BTPipe + iWiFi)
  10 b5      = ARM: push {r4, lr}
  1c 46      = ARM: mov r4, r3
  
SIGNIFICANCE:
FL=0x01 allows unauthenticated CompanionLink connections.
CF=0x24 enables BTPipe (L2CAP) + iWiFi routing on unauth link.

### 3. ADDITIONAL 0xB5 INSTANCES
All at 32-byte intervals in table:

0x190340: b5 (entry 0)
0x190360: b5 (entry 1)
0x190380: b5 (entry 2)
0x1903a0: b5 (entry 3)
0x1903c0: b5 (entry 4)
0x1903e0: b5 (entry 5)
0x190400: b5 (entry 6)
[continues for 20+ entries]

Pattern confirms this is a configuration array.

### 4. RADIO CONTROL STRING
Offset: 0x10e5bf
String: "radio_pwrsave "

Context: Function name in string table.
Likely function: Power management/radio control.
Related functions should clear LPM state (but don't).

## CORRELATION WITH TRACEV3 LOGS

### From tracev3 (0000000000000005.tracev3):
Offset 0x3D1B: "CID 0x6D810001, FL 0x1 < Unauth >, CF 0x24"

Matches firmware pattern:
  Firmware 0x0cdd30: 01 00 24 00 (FL=0x01, CF=0x24)
  Tracev3 0x3D1B:    FL 0x1, CF 0x24
  
### From tracev3:
String: "RSSI -75"

Matches firmware value:
  Firmware 0x190344: b5 (signed byte = -75)
  
## VERIFICATION COMMANDS

# Extract LPM table
dd if=SoC_RAM.bin bs=1 skip=$((0x190340)) count=512 | xxd

# Extract FL/CF code region  
dd if=SoC_RAM.bin bs=1 skip=$((0x0cdd30)) count=256 | xxd

# Search for all 0xB5 occurrences
od -A x -t x1 SoC_RAM.bin | grep " b5 00 00 00"

# Disassemble ARM code region
arm-none-eabi-objdump -D -b binary -m arm -M force-thumb \
  --start-address=0x0cdd30 --stop-address=0x0cde00 SoC_RAM.bin
