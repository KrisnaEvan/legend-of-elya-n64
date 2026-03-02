# RustChain Mining Bridge for Legend of Elya N64

Optional RustChain mining attestation module for the Legend of Elya N64 ROM.
Enables a real Nintendo 64 console to participate in RustChain's Proof-of-Antiquity
(RIP-200) mining network, earning RTC tokens by proving genuine vintage hardware.

The N64 (1996, NEC VR4300 MIPS R4300i @ 93.75 MHz) qualifies for a **3.0x antiquity
multiplier** -- older than any PowerPC Mac in the fleet.

## Architecture

Bidirectional bridge between the N64 and a RustChain attestation node:

```
N64 Console                 Raspberry Pi Pico            Host PC
+-----------+    Joybus     +---------------+    USB     +------------------+
| R4300i    | ------------> | GP1 data pin  | --------> | n64_bridge.py    |
| n64_attest|   Port 2      | main.cpp      |  Serial   |                  |
|           |   pak WRITE    | (firmware)    |  CDC      | polls /epoch     |
|           | <------------ |               | <-------- | polls /balance   |
|           |   pak READ     | CHAIN:hex\n   |           | polls /eligible  |
|           |   @ 0x8000     |               |           |                  |
+-----------+               +---------------+           +---> RustChain Node
                                                              50.28.86.131
```

**Data flows:**

- **N64 --> RustChain** (attestation): The N64 runs 5 hardware fingerprint checks on
  real silicon (CP0 PRId, COUNT timing, VI scanline, memory ratio, anti-emulation).
  Results are written as 32-byte pages to the controller pak via joybus WRITE commands.
  The Pico relays these over USB serial as `PAK_W:ADDR:HEXDATA` lines. The host bridge
  parses, assembles, and POSTs the attestation to the RustChain `/attest/submit` endpoint.

- **RustChain --> N64** (chain state): The host bridge polls the RustChain API every
  10 seconds for epoch, balance, and eligibility data. It packs this into a 32-byte
  page and sends it to the Pico via `CHAIN:64hexchars\n` serial command. The Pico
  serves this page to the N64 on pak READ at address `0x8000`. The N64 ROM validates
  the magic bytes (`RC`) and XOR checksum, then displays live chain data on screen.

## Directory Structure

```
mining/
  pico/
    main.cpp          -- Pico firmware: joybus POLL/READ/WRITE handler + USB serial relay
  host/
    n64_bridge.py     -- Python host bridge: serial <-> RustChain API
  n64/
    n64_attest.c      -- N64-side attestation checks + rendering + pak I/O
    n64_attest.h      -- AttestState struct, check IDs, phase defines, API
```

## Hardware Requirements

- **Nintendo 64 console** (any region: NUS-001, NUS-101, iQue)
- **Raspberry Pi Pico** (RP2040) -- not Pico W, just the basic Pico
- **USB cable** (micro-USB for Pico to host PC)
- **Joybus wiring**: Pico GP1 to N64 controller port 2 data line (green wire)
  - Port 1 is reserved for the player's real controller + EverDrive
  - The Pico appears as a standard controller with a "pak present" flag
- **Host PC** with Python 3.7+ and `pyserial` installed
- **EverDrive 64** or other flashcart to load the ROM

## Chain State Protocol

The 32-byte chain state page at pak address `0x8000`:

| Bytes  | Field                | Type       | Description                          |
|--------|----------------------|------------|--------------------------------------|
| 0-1    | Magic                | `"RC"`     | Validation magic bytes               |
| 2      | Version              | uint8      | Protocol version (1)                 |
| 3      | Flags                | uint8      | Bit0=last_accepted, Bit1=eligible, Bit2=data_valid |
| 4-5    | Epoch                | BE uint16  | Current RustChain epoch              |
| 6-9    | Slot                 | BE uint32  | Current slot number                  |
| 10-13  | Balance              | BE uint32  | Milli-RTC balance (5250 = 5.250 RTC) |
| 14-17  | Accepted             | BE uint32  | Server-confirmed accepted count      |
| 18-21  | Rejected             | BE uint32  | Server-confirmed rejected count      |
| 22-23  | Multiplier           | BE uint16  | Antiquity multiplier * 100 (300 = 3.0x) |
| 24     | Miners               | uint8      | Enrolled miners this epoch           |
| 25     | Rotation size        | uint8      | Lottery rotation size                |
| 26-27  | Turn offset          | BE uint16  | Slots until your turn                |
| 28-31  | Checksum             | 4x uint8   | XOR of bytes 0-27, repeated 4 times  |

All multi-byte fields are big-endian (native on N64 MIPS).

## Bridge Detection

The Pico identifies itself via the standard N64 POLL response by setting the analog
stick to a magic signature:

- `stick_x = 0x50` ('P' = 80 decimal)
- `stick_y = 0x42` ('B' = 66 decimal)

The N64 ROM checks this on controller port 2 via `controller_read()`. If the magic
is detected, the mining menu becomes available. No bridge = no mining option shown
(the game still works fine as a standalone LLM demo).

## Hardware-Derived Wallet

Each N64 console gets a unique wallet ID derived from its physical silicon:

- **CP0 PRId** register (R4300i revision/stepping)
- **RDRAM_CONFIG** register (memory controller configuration)
- **RDRAM_DEVICE_ID** registers (per-module unique IDs -- each RDRAM chip differs)
- Second RDRAM bank ID (expansion pak adds a different chip)

These are hashed together to produce a wallet ID in the format `n64-XXXXXXXXXXXXXXXX`
(16 hex characters from two 32-bit hashes). The same ROM on different N64 consoles
produces different wallet IDs. No seed phrases or private keys needed -- the hardware
IS the key.

## Building the Pico Firmware

Requires the [Pico SDK](https://github.com/raspberrypi/pico-sdk) and a joybus
library (e.g., [GP2040-CE joybus](https://github.com/OpenStickCommunity/GP2040-CE)):

```bash
cd mining/pico

# Set up Pico SDK
export PICO_SDK_PATH=/path/to/pico-sdk

# Build
mkdir build && cd build
cmake .. -DPICO_BOARD=pico
make -j4

# Flash: hold BOOTSEL on Pico, plug USB, copy .uf2
cp n64_pico_bridge.uf2 /media/RPI-RP2/
```

The firmware runs at 130 MHz for tight joybus timing. LED blinks on each POLL
(~60 Hz when N64 is running). USB serial runs at 115200 baud.

## Running the Host Bridge

Install dependencies:

```bash
pip install pyserial requests
```

### Monitor mode (watch pak events):

```bash
python3 mining/host/n64_bridge.py --monitor
```

### One-shot attestation:

```bash
python3 mining/host/n64_bridge.py --submit
```

### Continuous mining (recommended):

```bash
python3 mining/host/n64_bridge.py --continuous WALLET_ID
```

Where `WALLET_ID` is the `n64-XXXXXXXXXXXXXXXX` displayed on screen by the ROM.
In continuous mode, the bridge:

1. Monitors serial for attestation data from the N64
2. Auto-submits each complete attestation to the RustChain node
3. Polls the RustChain API every 10 seconds for epoch/balance/eligibility
4. Pushes 32-byte chain state pages back to the N64 via the Pico

### Custom serial port:

```bash
python3 mining/host/n64_bridge.py --port /dev/ttyACM1 --continuous n64-abc123
```

## Hardware Fingerprint Checks

The N64 ROM runs 5 checks to prove real hardware:

| # | Check           | What it measures                              | Pass criteria        |
|---|-----------------|-----------------------------------------------|----------------------|
| 1 | CPU PRId        | CP0 Register 15 (processor ID)                | R4300i family (0x0B) |
| 2 | COUNT Timing    | 200 NOPs measured via CP0 COUNT register       | 50-500 ticks         |
| 3 | VI Scan         | Video interface scanline progression rate       | 1-100 lines delta    |
| 4 | Memory Ratio    | Cached (KSEG0) vs uncached (KSEG1) access time | Ratio > 200 (2x)    |
| 5 | Anti-Emulation  | RDRAM config registers + COUNT jitter variance | Non-zero RDRAM       |

4 out of 5 checks must pass for mining eligibility. Real N64 hardware passes all 5.
Emulators fail on COUNT timing (synthetic), memory ratio (uniform), and/or RDRAM
registers (zero or absent).

## Integration with Legend of Elya

The mining module is integrated into the main ROM (`legend_of_elya.c`) as an
optional menu item. When the game detects a Pico bridge on port 2, a "Mine RTC"
option appears in the menu. The attestation screen shows:

- Real-time check results (PASS/FAIL with hex values)
- Live mining stats (epoch, slot, balance, multiplier)
- Hardware-derived wallet ID
- Accepted/rejected attestation counts

Mining runs in the background during the LLM inference demo. Press B to return
to the main game at any time.

## See Also

- [Main README](../README.md) -- Legend of Elya N64 LLM overview
- [RustChain RIP-200](https://github.com/Scottcjn/rustchain-bounties) -- Proof-of-Antiquity specification
- [Block Explorer](https://50.28.86.131/explorer) -- Live RustChain network
