# 🎯 PTAC - Protocolo de Transmisión de Archivos Críticos

This repository contains preliminary academic work related to the PTAC protocol.

All content is published for research and educational purposes only.

You are free to study, modify, and build upon this work, provided that:
- Proper attribution to the original author is given.
- Any derived work is non-commercial.

> **⚠️ Status — Academic work in development.**
> This repository documents the PTAC protocol (degree thesis, UNAB 2026). The **source code is not yet public** — it is under active development and available on request. All content is published for **research and educational purposes only**: you are free to study and build upon it with attribution, but derived work must be **non-commercial**.

> **A resilient, decentralized, and transparent radio-frequency file transfer system for emergency responders.**
> 
> *Autenticidad sin Ocultamiento* — Integrity > Robustness > Speed.

![PTAC](https://img.shields.io/badge/PTAC-Protocol-blue.svg) ![Hardware](https://img.shields.io/badge/Hardware-Pi_Zero_2W_+_Pico-green.svg) ![RF](https://img.shields.io/badge/RF-433.95_MHz-orange.svg)

---

## 📖 Table of Contents
1. [Overview](#-overview)
2. [Hardware Architecture](#-hardware-architecture)
3. [Installation & Flashing](#-installation--flashing)
   - [Host Setup (Raspberry Pi Zero 2W)](#host-setup-raspberry-pi-zero-2w)
   - [Firmware Setup (Raspberry Pi Pico)](#firmware-setup-raspberry-pi-pico)
4. [CLI Usage Guide](#-cli-usage-guide)
5. [How It Works (The 4-Layer Stack)](#-how-it-works)
6. [Development & Testing](#-development--testing)

---

## 🌋 Overview

**PTAC** (Protocolo de Transmisión de Archivos Críticos) is an embedded RF system designed to guarantee the **integral, decentralized, and transparent** transmission of critical binary files (like tactical maps, structural plans, medical reports) during natural disasters when conventional telecommunications have collapsed.

Operating in the **433.95 MHz** ISM band using **2-GFSK at 250 kbps**, PTAC uses an advanced 4-layer protocol stack to prioritize data integrity over speed, ensuring that a file received is **identical bit-by-bit** to the file sent, using cryptographic hashes (SHA-256) and Forward Error Correction (Reed-Solomon).

---

## 🔩 Hardware Architecture

Each PTAC node costs under **~$50 USD** and consists of three main components:

| Component | Role | Specifications |
|---|---|---|
| **Raspberry Pi Zero 2W** | Main Host / Orchestrator | Runs the Python PTAC stack, SQLite, and CLI. |
| **Raspberry Pi Pico (RP2040)** | Radio Controller | Runs MicroPython firmware to manage real-time SPI communication with the radio. |
| **CC1101 (Texas Instruments)** | RF Transceiver | 433.95 MHz, 2-GFSK, 250 kbps, 0 dBm. |

### Wiring Diagram (Pico ↔ CC1101)

| Raspberry Pi Pico | CC1101 | Function |
|-------------------|--------|---------|
| `GP2` (Pin 4)     | `SCLK` | SPI Clock |
| `GP3` (Pin 5)     | `MOSI` | SPI Master Out Slave In |
| `GP4` (Pin 6)     | `MISO` | SPI Master In Slave Out |
| `GP5` (Pin 7)     | `CSN`  | SPI Chip Select |
| `GP6` (Pin 9)     | `GDO0` | Interrupt / Sync word detection |
| `3V3(OUT)` (Pin 36)| `VCC` | Power (3.3V, **DO NOT USE 5V!**) |
| `GND` (Pin 38)    | `GND`  | Common Ground |

*The Pi Zero 2W connects to the Pi Pico via USB (Serial/UART at 115200 bps).*

---

## 🛠️ Installation & Flashing

### Host Setup (Raspberry Pi Zero 2W)

1. Clone the repository and install the Python dependencies:
   ```bash
   git clone https://github.com/brnomt/PTAC.git
   cd PTAC
   pip install -r requirements.txt
   ```
   *(Dependencies include `reedsolo`, `lz4`, `pyyaml`, and `pyserial-asyncio`)*

2. Generate the default configuration file:
   ```bash
   python3 main.py config --generate
   ```

### Firmware Setup (Raspberry Pi Pico)

The Pico runs a custom MicroPython firmware that acts as a UART-to-SPI bridge for the CC1101.

1. **Install MicroPython on the Pico**:
   - Hold the `BOOTSEL` button on the Pico while plugging it into your computer via USB.
   - Download the official MicroPython `.uf2` file from [micropython.org](https://micropython.org/download/rp2-pico/).
   - Drag and drop the `.uf2` file onto the `RPI-RP2` drive that appears.

2. **Flash the PTAC Firmware**:
   We provide a helper script using `mpremote` to copy the firmware files to the Pico.
   ```bash
   cd firmware/
   ./flash.sh
   ```
   *If you are on Windows, you can manually copy `cc1101_spi.py` and `main.py` using Thonny IDE or `mpremote cp cc1101_spi.py : && mpremote cp main.py :`.*

3. **Verify**: The Pico's onboard LED will flash 3 times on boot, indicating the firmware is running and waiting for UART commands from the Host.

---

## 💻 CLI Usage Guide

The `main.py` script is the primary entry point for using the PTAC protocol.

### Send a File
```bash
python3 main.py send /path/to/tactical_map.pdf
```
*Options:*
- `--peer`: Address of the receiving node (default: "RF")
- `--compression`: Override compression algorithm (default: `lz4`)
- `--loopback`: Run in loopback simulation mode (no hardware required).

### Receive a File
```bash
python3 main.py receive --output-dir ./received_files
```
*Listens on the radio for incoming SYN packets and automatically negotiates the transfer.*

### View Transfer Status & Metrics
All transfers are logged to a local SQLite database (`ptac_sessions.db`).
```bash
# List all historical sessions
python3 main.py status

# View detailed metrics for a specific session
python3 main.py status --session-id <UUID>
```

### Run an End-to-End Simulation
You can test the entire protocol stack entirely in software (without a Pi Pico or CC1101) by using the Loopback Radio simulator:
```bash
python3 main.py test --size 50000 --error-rate 0.01 --latency 10
```

---

## 🧠 How It Works

PTAC implements a custom 4-layer OSI-inspired stack tailored for critical RF environments.

### 1. Application Layer
- **Compression**: Files are compressed using LZ4 to reduce airtime.
- **Integrity**: A complete SHA-256 hash of the *original* file is calculated before transmission.
- **Metadata**: JSON metadata (name, size, hash) is sent in a dedicated `FILE_META` frame.

### 2. Transport Layer (ARQ)
- **Fragmentation**: The compressed file is fragmented into small chunks.
- **Selective-Repeat ARQ**: Uses a sliding window (size 8). If a packet is lost, only that specific packet is retransmitted (NACK), up to 5 times.
- **Adaptive Timeout**: Uses an Exponentially Weighted Moving Average (EWMA) based on the round-trip time (RTT) to dynamically adjust ACK timeouts.

### 3. Link Layer (FEC & Interleaving)
This layer ensures that corrupted bits over the air can be repaired without needing a retransmission.
- **Reed-Solomon RS(255,193)**: Can mathematically repair up to 31 corrupted bytes per packet.
- **Virtual Padding (Shortened Codes)**: The CC1101 hardware has a strict 255-byte FIFO limit. PTAC uses *Virtual Padding* (66 bytes) to send 127-byte data chunks. Combined with ARQ headers (22B), RS parity (62B), and Link headers (10B), the packet size over the air is exactly 221 bytes—fitting safely inside the hardware limit!
- **Block Interleaving**: Matrix-based interleaving (depth 16) scatters burst errors across multiple packets.
- **Software CRC-16**: Added on top of the CC1101's hardware CRC as defense-in-depth.

### 4. Physical Layer
- **CC1101 Driver**: The Pi Pico firmware configures the CC1101 registers via SPI for 2-GFSK at 250 kbps, handles variable-length packet transmission, and reads RSSI/LQI metrics.

---

## 🧪 Development & Testing

We use `pytest` for rigorous unit and integration testing.

```bash
# Run all unit tests
python3 -m pytest tests/test_ptac.py -v

# Run the End-to-End (E2E) protocol pipeline test
python3 tests/test_e2e.py --suite
```

### Self-Tests
Each layer module contains an executable `__main__` block that acts as a standalone sandbox test:
```bash
python3 -m ptac.link.codec       # Tests CRC, RS, and Interleaving
python3 -m ptac.transport.arq    # Tests ARQ logic and Window management
```

---
*Developed by Bruno Moraga Torres — Ingeniería en Computación e Informática, UNAB (2026).*

Author: [**Bruno Moraga Torres**](https://github.com/brnomt)

Year: 2026.
