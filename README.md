# Secure Dual-Bank OTA Firmware Update System
### STM32F446RE + ESP32-S3 + AWS IoT Core

A complete end-to-end Over-the-Air (OTA) firmware update pipeline for an STM32F446RE microcontroller using STM32 HAL, ESP32-S3 as a network co-processor, and AWS cloud services — enabling remote firmware updates without physical access to the device.

---

## Demo

> OTA update in progress — ESP32-S3 downloading firmware from AWS S3 and streaming to STM32 over UART

![OTA Progress](images/ota_progress.png)

> AWS IoT Job marked as Succeeded after successful firmware verification and bank swap

![AWS Job Succeeded](images/aws_job_succeeded.png)

> STM32 debug output showing CRC32 match and metadata write

![CRC Match](images/crc_match.png)

---

## System Architecture

```
┌─────────────────────────────────────────────┐
│                  AWS Cloud                   │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ S3 Bucket│  │ IoT Core │  │ IoT Jobs  │  │
│  │ fw_vX.bin│  │  (MQTT)  │  │ Job Docs  │  │
│  └──────────┘  └──────────┘  └───────────┘  │
└────────────────────┬────────────────────────┘
                     │ TLS/MQTT + HTTPS
┌────────────────────▼────────────────────────┐
│              ESP32-S3 (Co-processor)         │
│  • MQTT client → receives job notification  │
│  • HTTPS downloader → chunks from S3        │
│  • UART bridge → streams to STM32           │
└────────────────────┬────────────────────────┘
                     │ UART (115200 baud, custom framing)
┌────────────────────▼────────────────────────┐
│           STM32F446RE (Main MCU)             │
│  • Bootloader  0x08000000  32KB              │
│  • Metadata    0x08008000  16KB              │
│  • App Bank A  0x0800C000  80KB  (active)    │
│  • App Bank B  0x08020000  256KB (OTA target)│
└─────────────────────────────────────────────┘
```

---

## Flash Memory Layout

| Region | Sectors | Start Address | Size | Description |
|--------|---------|--------------|------|-------------|
| Bootloader | 0–1 | `0x08000000` | 32 KB | Custom bootloader — flashed once via ST-Link |
| Metadata | 2 | `0x08008000` | 16 KB | Boot flag, CRC32, firmware version, size |
| App Bank A | 3–4 | `0x0800C000` | 80 KB | Active application — initial ST-Link flash |
| App Bank B | 5–6 | `0x08020000` | 256 KB | OTA target — written wirelessly |
| Free | 7 | `0x08060000` | 128 KB | Reserved for future use |

---

## OTA Flow

```
1. Developer builds OTA_Release binary (linked at 0x08020000)
2. Python script computes CRC32 matching STM32 hardware CRC unit
3. Binary uploaded to S3, job document created in AWS IoT Jobs
4. ESP32-S3 receives MQTT notification → downloads binary from S3
5. ESP32-S3 sends firmware in 256-byte chunks over UART with CRC16 framing
6. STM32 writes each chunk to Bank B, ACKs each frame
7. STM32 receives DONE frame → computes CRC32 over Bank B
8. CRC matches → metadata written (boot_flag = BANK_B) → system reset
9. Bootloader verifies CRC32 → jumps to Bank B
10. New firmware running ✅ | AWS Job → SUCCEEDED ✅
```

---

## UART Frame Protocol

```
[ 0xAA ][ 0x55 ][ TYPE:1 ][ SEQ:2 ][ LEN:2 ][ PAYLOAD:N ][ CRC16:2 ]
```

| Type | Value | Description |
|------|-------|-------------|
| JOB | `0x01` | New OTA job info (fw_size, crc32, version) |
| CHUNK | `0x02` | 256-byte firmware chunk |
| DONE | `0x03` | All chunks sent, trigger verification |
| ACK | `0x04` | STM32 → ESP32 acknowledge |
| NACK | `0x05` | STM32 → ESP32 negative acknowledge |

---

## Hardware

| Component | Board | Role |
|-----------|-------|------|
| STM32F446RE | Nucleo-F446RE | Main MCU — runs application + OTA receiver |
| ESP32-S3 | ESP32-S3 DevKit | Network co-processor — WiFi + MQTT + HTTPS |

### Wiring

```
STM32 PA9  (USART1_TX)  →  ESP32-S3 Pin 18 (RX)
STM32 PA10 (USART1_RX)  →  ESP32-S3 Pin 17 (TX)
GND                      →  GND
```

---

## Project Structure

```
├── Secure_Firmware_Boot/          # Bootloader project (STM32CubeIDE)
│   └── Core/
│       ├── Inc/metadata.h         # Flash addresses, boot flags, metadata struct
│       └── Src/main.c             # Bootloader: CRC32 verify + bank swap jump
│
├── Secure_Firmware_Apln/         # Application project (STM32CubeIDE)
│   ├── Core/
│   │   ├── Inc/
│   │   │   ├── metadata.h         # Shared with bootloader
│   │   │   └── ota.h              # OTA frame protocol, state machine
│   │   └── Src/
│   │       ├── main.c             # Application entry + OTA_Process loop
│   │       ├── ota.c              # UART ISR, ring buffer, frame parser, flash write
│   │       └── system_stm32f4xx.c # VTOR relocation for bank A / bank B
│   ├── STM32F446RETX_FLASH.ld     # Bank A linker script (0x0800C000, 80KB)
│   └── STM32F446RETX_FLASH_OTA.ld # Bank B linker script (0x08020000, 256KB)
│
├── ESP32_OTA_Bridge/              # ESP32-S3 Arduino sketch
│   ├── ESP32_OTA_Bridge.ino       # Setup + loop
│   ├── aws_iot.h / aws_iot.cpp    # WiFi, MQTT, AWS IoT Jobs, HTTP downloader
│   ├── aws_certs.h                # Device certificate + private key + root CA
│   ├── ota_bridge.h / ota_bridge.cpp # UART framing, chunk send, ACK handler
│
└── tools/
    └── compute_crc32.py           # CRC32 tool matching STM32 hardware CRC unit
```

---

## Build Configurations

### STM32 App — Two build configs in STM32CubeIDE

| Config | Linker Script | FLASH Origin | Use |
|--------|--------------|-------------|-----|
| Debug | `STM32F446RETX_FLASH.ld` | `0x0800C000` | Flash via ST-Link (bank A) |
| OTA_Release | `STM32F446RETX_FLASH_OTA.ld` | `0x08020000` | Generate `.bin` for OTA upload |

`OTA_Release` has preprocessor define `BANK_B_BUILD` which sets `SCB->VTOR = 0x08020000` in `SystemInit()`.

---

## AWS Setup

1. **S3 bucket** — stores firmware binaries
2. **IoT Thing** — `STM32_OTA_Device` with X.509 certificate
3. **IoT Policy** — allows connect, subscribe, publish, receive on jobs topics
4. **IoT Job** — custom job with job document:

```json
{
  "firmwareUrl": "https://bucket.s3.region.amazonaws.com/fw_vX.bin?<presigned>",
  "version": 20,
  "size": 18728,
  "crc32": 1813621012
}
```

---

## CRC32 Tool

The Python tool generates CRC32 values that exactly match the STM32F4 hardware CRC unit:

```bash
python tools/compute_crc32.py path/to/OTA_Release/Secure_Firmware_Appln.bin
```

Output:
```
File:    Secure_Firmware_Appln.bin
Size:    18728 bytes
CRC32:   0x6C19A914  (1813621012)
First 16 bytes: 00 00 02 20 0D 15 02 08 ...

── Job document ──────────────────────────────
{
  "firmwareUrl": "PASTE_PRESIGNED_URL_HERE",
  "version": 20,
  "size": 18728,
  "crc32": 1813621012
}
```

---

## Key Technical Details

### Bootloader jump mechanism
Bootloader reads stack pointer and reset handler from the app's vector table, validates both against expected ranges, disables SysTick and NVIC, relocates VTOR, sets MSP, and calls the reset handler with interrupts re-enabled.

### CRC32 endianness
STM32F4 hardware CRC processes 32-bit words in little-endian byte order (native ARM memory access). Python script uses `struct.unpack('<I', ...)` to match this exactly. This was a critical debugging insight — big-endian unpacking causes consistent CRC mismatches.

### Non-blocking OTA reception
UART bytes arrive via ISR into a 1024-byte ring buffer. `OTA_Process()` is called in the main loop to drain the buffer and run the frame parser state machine — no blocking delays during chunk reception.

### Rollback protection
If CRC32 verification fails after OTA, the bootloader clears the BANK_B boot flag and jumps to bank A — the device always recovers to a known-good state.

---

## Development Tools

- STM32CubeIDE
- STM32CubeProgrammer
- Arduino IDE 2.x (ESP32-S3)
- Python 3.x
- AWS Console (IoT Core, S3)

---

## Results

- ✅ Full OTA pipeline working end-to-end
- ✅ 18728 bytes transferred in ~73 chunks of 256 bytes
- ✅ CRC32 hardware verification passing consistently
- ✅ Automatic bank swap and bootloader jump working
- ✅ AWS IoT Job status → SUCCEEDED
- ✅ Rollback to bank A on CRC failure

---

## Images 

```
images/
├── ota_progress.png         → ESP32 Serial Monitor showing chunk progress
├── aws_job_succeeded.png    → AWS console showing Job Succeeded
├── crc_match.png            → STM32 debug output showing CRC OK
├── system_architecture.png  → Block diagram
```
