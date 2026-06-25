# OpenC6 BIOS Core: Initialization, Boot Dispatcher & ABI

The Core module (`bios_init.c` and `bios_core.h`) serves as the primary host system initialization engine. It acts as a 3rd-stage bootloader on top of the ESP-IDF. Its main responsibilities include managing power transitions, validating the system state via Power-On Self-Test (POST), routing the boot path, and exporting the System Application Binary Interface (ABI) to user payloads.

---

## 1. Module Overview

Upon CPU start, the BIOS Core takes control of the High-Performance (HP) core. It decides whether to launch the local BIOS Setup utility (Web UI), execute an OTA firmware update, enter serial bootloaders, or perform local booting of a stored application payload from the file system. 

Local booting utilizes a hybrid execution architecture: the system first attempts to load the binary directly into execution-capable RAM. If the binary size exceeds available SRAM, the Core automatically falls back to Execute-In-Place (XIP) execution by deploying the file to a dedicated, contiguous flash partition.

Additionally, this module remains active in the background when running payloads, servicing the low-power (LP-core) Management Engine watchdog and routing hardware event interrupts.

---

## 2. Hardware Configuration & GPIO Pinout

These hardware definitions are declared in `bios_core.h` and used across the initialization stage:

| Define Constant   | GPIO Pin | Description                                              |
| :---------------- | :------: | :------------------------------------------------------- |
| `PIN_BTN_GND`     | `3`      | Software-controlled ground for the physical Power Button |
| `PIN_BTN_SENSE`   | `4`      | Power Button sensory pin (Active Low)                    |
| `PIN_BTN_BOOT`    | `9`      | Hardware boot strapping pin (Standard ESP32 BOOT button) |
| `PIN_CMOS_GND`    | `1`      | Software-controlled ground for the Clear CMOS jumper     |
| `PIN_CLEAR_NVRAM` | `2`      | Sense pin for Clear CMOS Jumper (Active Low)             |
| `PIN_POST_LED`    | `8`      | Onboard POST status LED indicator                        |

---

## 3. Core Initialization & State Machine Flow

The entry point of the HP-core is `bios_core_start()`. It runs a deterministic state machine to determine the target execution context.

```text
       [ Power On / Reset ]
                │
                ▼
   [ Legacy Wakeup Measurement ] (Before task delays)
                │
                ▼
    [ Component Initialization ] (NVRAM, LED, WiFi, ME)
                │
                ▼
  [ Network BIOS Update Check ] ──(Pending?)──► [ PXE BIOS OTA ]
                │
                ▼
    [ CMOS Clear Jumper Check ] ──(Shorted?)──► [ Reset NVRAM ]
                │
                ▼
       [ POST Verification ] ────(Failed?)────► [ System Halts / Rollback ]
                │
                ▼
     [ Boot Selection Matrix ] ◄──(GPIO 9 Hold)
                │
    ┌───────────┼───────────────┬────────────────┐
    ▼           ▼               ▼                ▼
[Serial RAM] [Serial Flash] [PXE Net Boot] [Flash OS Boot]
                │                                │
                ▼                                │
             (temporarily removed)               │
                                       ┌─────────┴─────────┐
                                       ▼                   ▼
                                 [RAM Execution]   [XIP Flash Fallback]
```

### Key Boot Stages:

1. **Legacy Wakeup Measurement:** Executed immediately before FreeRTOS scheduler context-delays. If waking up from deep sleep without the Management Engine (Legacy Mode), it measures how long the power button is held.
2. **Component Init:** Brings up the virtual NVRAM (NVS partition), starts LED status controls, and configures the Wi-Fi management system.
3. **Network BIOS Update (PXE-based OTA):** If NVRAM indicates a pending update flag (`BIOS_UPDATE_PENDING`), the BIOS transitions into a network recovery loop, fetches the new binary, updates itself, and restarts.
4. **CMOS Jumper Protection:** Checks if `PIN_CLEAR_NVRAM` is pulled to `PIN_CMOS_GND`. If detected, NVRAM is immediately restored to factory default settings to recover from invalid configurations.
5. **POST Verification:** Performs hardware validation. If running an unverified firmware partition (during OTA testing), it cancels the rollback flag once the POST check succeeds.
6. **Boot Selection Matrix (Local OS Execution):** Evaluates boot conditions using physical buttons, sleep wakeup sources, and the LP-core Management Engine (ME) state registers. When executing local boot from the file system, the Core processes `payload.bin` located in the `/downloaded/` folder as follows:
    * **Stage 1 (RAM Boot):** The system first attempts to allocate a contiguous block of memory in internal SRAM with execution capabilities (`MALLOC_CAP_EXEC`). If allocation succeeds, the segmented chunks of the file are read from `openc6_fs` directly into RAM, and executed.
    * **Stage 2 (XIP Fallback):** If internal RAM is full or the file size exceeds available heap, the system invokes the XIP fallback path. It erases the dedicated, contiguous `payload_xip` partition (1 MB), copies the file chunks sequentially into it, maps the partition via the hardware MMU (`esp_partition_mmap`), and jumps to the mapped virtual instruction address.

---

## 4. Power State Transitions (S5 Soft-Off)

The BIOS Core implements power management transitions via the `bios_enter_s5_state()` routine.

```c
void bios_enter_s5_state(void);
```

* **Management Engine (ME) Mode:** If `is_me_enabled` is true, the main processor powers down, signaling `ulp_me_hp_is_awake = 0` to the LP-core. It enables ULP-wakeup, allowing the low-power coprocessor to manage button-clicks, watchdogs, and thermal profiles.
* **Legacy Mode (ME Disabled):** If the ME is disabled, the core manually configures GPIO pins 3 (as output low) and 4 (as pull-up input) and invokes `esp_sleep_enable_ext1_wakeup()` to allow standard EXT1 hardware wakeups.

---

## 5. System ABI Specifications (openc6_abi_t)

To decouple payloads from the ESP-IDF framework, the BIOS exposes its internal services via a structure pointer passed to payloads through CPU register `a0`. The structure is populated in `bios_init.c` as `bios_abi`.

### 5.1 System Controls

* `sys_reset`: Invokes `esp_restart()` to perform a complete hardware reboot.
* `delay_ms`: Suspends execution for a specific duration yielding the CPU core via the RTOS scheduler (prevents Watchdog starvation).
* `print`: Outputs standard diagnostic text to the active console interface.
* `get_random`: Generates hardware-entropy-based random integers.
* `malloc`: Allocates a block of memory dynamically from the BIOS heap caps.
* `free`: Deallocates memory blocks previously allocated by the system heap.
* `set_led_color`: Directly controls the onboard status RGB LED color parameters.

### 5.2 Cryptography

* `sha256`: Performs hardware-accelerated SHA256 computations using the PSA Crypto subsystem.

### 5.3 System Metrics & Telemetry

* `get_free_ram`: Returns free internal 8-bit capable memory blocks (SRAM).
* `get_total_ram`: Returns the total size of physical internal SRAM.
* `get_total_flash`: Dynamically queries the onboard SPI flash capacity.

### 5.4 Math Coprocessor Emulation (FPU-less Payload Helper)

Because payloads are compiled bare-metal, floating-point operations can be expensive. The BIOS provides fast integer-only emulators:

* `math_isqrt`: Calculates fast integer square root using binary-shifting bit-operations.
* `math_sin_deg`: Evaluates Trigonometric Sine given an integer degree. Returns a fixed-point result scaled by 10000 (e.g., sin(30) returns 5000).
* `math_cos_deg`: Evaluates Trigonometric Cosine using the same fixed-point scaling factor.

### 5.5 Wi-Fi Networking Services

* `wifi_connect`: Configures credentials in NVRAM and starts the station engine.
* `wifi_start_ap`: Initiates a local access point.
* `wifi_is_connected`: Checks active connectivity status.

### 5.6 File System Services

The BIOS exposes core file system operations to payloads, allowing them to read, write, and delete files on the log-structured `openc6_fs` partition without needing to bundle filesystem drivers:

* `fs_write_file`: Writes data to a file inside a specified parent directory. Accepts an extra `force` flag for write persistence management. It handles chunk-segmented allocation automatically, manages versioning, and triggers garbage collection if space is low.
* `fs_read_file`: Reads a specific byte range from a file by its string name and parent directory ID. It automatically maps and reassembles fragmented chunks using the dynamic on-demand RAM cache, ensuring fast reading.
* `fs_delete`: Deletes a file or directory by its string name inside a specific parent folder by appending a tombstone record and updating the virtual RAM index.
