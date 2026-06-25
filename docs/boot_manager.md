OpenC6 Boot Manager & Application Binary Interface (ABI)

The Boot Manager (boot_menu.c and boot_manager.h) handles the selection,
transmission, and memory-mapping of modular user payloads. The ABI
(openc6_abi.h) serves as the core contract between the host BIOS and the
payload, decoupling the program's logic from physical hardware addresses and
ESP-IDF compilation artifacts.

1. Interactive Boot Menu (GPIO 9)
If the hardware boot button (PIN_BTN_BOOT / GPIO 9) is held during startup, the
BIOS intercepts the regular boot cycle and opens an interactive physical menu.

Navigation Logic:

  - Short Press (< 500 ms): Cycle to the next boot option. The on-board POST LED
    blinks a corresponding pattern using cyan light (color 0, 255, 255) to
    indicate the current option index (from 1 to 5 blinks).
  - Long Press (>= 1000 ms): Confirm and execute the selected option. The POST
    LED flashes green (0, 255, 0) to acknowledge selection.

Available Boot Options (boot_option_t):

| Enum Identifier         | Option Index | Display Name        | Target & Action                                                                       |
| :---------------------- | :----------: | :------------------ | :------------------------------------------------------------------------------------ |
| `BOOT_OPT_NETWORK`      | `0`          | Network Boot (PXE)  | Attempts to fetch and execute a payload from the configured remote web server.        |
| `BOOT_OPT_SERIAL_RAM`   | `1`          | Serial Boot (RAM)   | Opens a serial listener to stream a binary directly into volatile execution SRAM.     |
| `BOOT_OPT_SERIAL_FLASH` | `2`          | UNIX Shell (FS)     | Redirects console I/O to UART1 and launches an interactive micro UNIX shell for file system management and execution. |
| `BOOT_OPT_DEFAULT`      | `3`          | Default OS (Flash)  | Maps the local payload storage partition and executes the default operating system.   |
| `BOOT_OPT_SETUP`        | `4`          | BIOS Setup          | Starts the local Wi-Fi Access Point and launches the Web Configuration Console.       |

2. Serial Payload Receiver Protocol

The serial receiver is designed to securely stream unlinked binary programs
directly from a host development machine into volatile RAM. It bypasses conventional 
flashing tools, optimizing deployment speed during development.

2.1 Hardware Interface & Port Routing

The serial bootloader supports two communication paths, configured via the
compiler flag USE_EXTERNAL_CP2102:

1.  External CP2102 UART1 Interface (Enabled by default): Uses physical pins
    TX=18 and RX=19 at 115,200 baud. This isolates debug console logs (standard
    UART0/USB) from binary stream traffic.
2.  Native USB-JTAG Interface: Uses the ESP32-C6 integrated USB-to-Serial JTAG
    controller.

2.2 Synchronization & Handshake State Machine

The receiver employs a lightweight packet-less streaming protocol with
byte-level flow control:

    Host PC (Python script)                       OpenC6 BIOS
             │ ◄────────── "##OPENC6_SYNC##" ──────────┤ (Sent every 500ms)
             ├───────── Preamble [0x5A, 0xA5] ────────►│
             ├──────── Size Bytes [4-Byte LE] ────────►│
             │ ◄──────────── ACK [0x06] ───────────────┤ (Size validation OK)
             │                                         │
             ├───────── Chunk Data (64 Bytes) ────────►│
             │ ◄──────────── ACK [0x06] ───────────────┤ (Repeated until finished)
             │                                         │
             ├──────────── EOT [0x04] ────────────────►│
             │                                         ▼
                                                [ Launch Payload ]

1.  Synchronization Beacon: The BIOS streams `##OPENC6_SYNC##` every 500 ms over the active port.
2.  Size Preamble: The host responds with a synchronization frame: two magic bytes (0x5A, 0xA5) followed immediately by a 4-byte little-endian unsigned integer representing the file size.
3.  Preamble Validation: The BIOS checks internal RAM size bounds (up to ~500 KB). If valid, it responds with `CMD_ACK` (0x06); otherwise, it sends `CMD_NAK` (0x15) and aborts.
4.  Memory Allocation & Flash Protection: 
    - RAM Target: Allocates a contiguous execution memory block using `MALLOC_CAP_EXEC | MALLOC_CAP_INTERNAL`.
    - Flash Target Security: Direct serial streaming to flash is explicitly blocked inside `boot_manager_serial_listen` to protect the log-structured integrity of the `openc6_fs` partition from raw corruptions. Persistent deployments must be managed via the Micro UNIX Shell.
5.  Streaming Phase: The host sends binary data in 64-byte chunks. The BIOS returns an `ACK` (0x06) byte to the host for every 64 bytes processed.
6.  End of Transmission (EOT): The host signals completion by sending `CMD_EOT` (0x04).

2.3 Micro UNIX Shell (FS Explorer & Launcher)

When selecting `BOOT_OPT_SERIAL_FLASH`, the BIOS hooks standard standard input/output descriptors to the dedicated hardware UART1 port (`/dev/uart/1`) using the ESP-IDF Virtual File System (VFS) driver layers. This launches an interactive, command-line interface for manual manipulation of the log-structured `openc6_fs` architecture.

Supported Shell Commands:
* `help`                  - Displays the complete shell utility documentation index.
* `format`                - Destructively formats the current file system storage area.
* `ls [<path>]`           - Lists available directory nodes and descriptors inside the given or active path.
* `cd <path>`             - Alters the active working directory context node.
* `mkdir <path>`          - Allocates and builds a new virtual directory index record.
* `write [-f] <path> <t>` - Appends text data directly into a file. Use the `-f` flag to overwrite existing allocations.
* `cp <src> <dst_path>`   - Extracts data from a source file and writes a duplicate record to the destination.
* `mv <src> <dst_path>`   - Performs file renaming or transfers file objects across parent nodes.
* `cat <path>`            - Dumps file contents directly into the stdout terminal stream.
* `rm <path>`             - Appends a tombstone record to effectively remove a file or directory node.
* `boot <ram|xip> <path>` - Executes binary payloads dynamically using one of two execution schemes:
    - `ram`: Allocates execution-safe heap memory inside internal SRAM, copies the binary, flushes instruction caches, and jumps.
    - `xip`: Erases and writes the binary to the `payload_xip` flash partition, maps it to the instruction cache controller via `esp_partition_mmap()`, and executes via hardware Execute-In-Place.
* `exit`                  - Dismantles active console VFS drivers and triggers a full device restart.

3. Payload Execution & Memory Requirements

3.1 IRAM Execution Security Bypass

Executing arbitrary machine instructions from SRAM requires special
configuration on RISC-V architectures. ESP-IDF features native hardware memory
protection that flags SRAM pages as non-executable by default.

To execute RAM payloads, the following configuration must be explicitly disabled
in your BIOS compilation target sdkconfig:

CONFIG_ESP_SYSTEM_MEMPROT_FEATURE=n

The BIOS checks for this capability during compilation by asserting
MALLOC_CAP_EXEC.

3.2 Handover and I-Cache Invalidation

Once the binary is fully written to RAM or mapped to virtual instruction cache
space (XIP), the BIOS shuts down its serial drivers, changes the status LED, and
prepares for execution:

1.  Pipeline Flush: Invokes `asm volatile ("fence.i")` to invalidate the CPU
    instruction cache and pipeline, ensuring the CPU does not execute stale or
    unwritten cache blocks.
2.  Execution Jump: Passes control to the entry point using the ABI structure
    pointer as the first argument (a0 register in RISC-V):

typedef void (*payload_entry_t)(const openc6_abi_t *) __attribute__((noreturn));
launch_payload(&bios_abi);

4. ABI Specification Details (openc6_abi_t)

The system ABI acts as a robust jump table containing stable function pointers.
This prevents payloads from needing to link against ESP-IDF component APIs
directly, reducing the payload size from hundreds of kilobytes down to compact
bare-metal binaries.

typedef struct {
    uint32_t magic;    // System validation tag (0x43364249 - "C6BI")
    uint32_t version;  // Interface revision tracking (Current: 1)

    // Core OS & HW Wrappers
    void (*sys_reset)(void);
    void (*set_led_color)(uint8_t r, uint8_t g, uint8_t b);
    void (*delay_ms)(uint32_t ms);
    void* (*malloc)(uint32_t size);
    void (*free)(void* ptr);
    void (*print)(const char *str);
    uint32_t (*get_random)(void);
    void (*sha256)(const uint8_t *input, uint32_t len, uint8_t *output);

    // Optimized Math Wrappers
    uint32_t (*math_isqrt)(uint32_t x);
    int32_t (*math_sin_deg)(int32_t angle_deg);
    int32_t (*math_cos_deg)(int32_t angle_deg);

    // Network & Wi-Fi Interfaces
    int32_t (*wifi_connect)(const char* ssid, const char* pass);
    int32_t (*wifi_start_ap)(const char* ssid, const char* pass);
    int32_t (*wifi_is_connected)(void);

    // Telemetry & Hardware Info
    uint32_t (*get_free_ram)(void);
    uint32_t (*get_total_ram)(void);
    uint32_t (*get_total_flash)(void);

    // File System ABI exports
    void (*fs_write_file)(const char *name, const uint8_t *data, uint32_t len, uint32_t parent_id, uint8_t force);
    int32_t (*fs_read_file)(const char *name, uint8_t *dest, uint32_t offset, uint32_t len, uint32_t parent_id);
    void (*fs_delete)(const char *name, uint32_t parent_id);
} openc6_abi_t;

### ABI Design Highlights:
* **FPU-less Compatibility:** Floating-point units are absent or slow in raw payload environments. The math emulation functions `math_sin_deg` and `math_cos_deg` use highly optimized double-precision lookup tables inside the BIOS but return simple, fast `int32_t` fixed-point values scaled by 10000 (e.g., sin(45°) ≈ 0.7071, returning 7071 to the payload).

* **Hardware-Accelerated Cryptography:** The `sha256` function abstracts complex cryptographic state configuration, allowing raw payloads to invoke the ESP32-C6 hardware crypto engine seamlessly via a single call.

* **Network Encapsulation:** The payload can initiate wireless tasks (connecting to a router or hosting a setup AP) using simple string parameters, leaving socket management, IP address assignment, and Wi-Fi drivers entirely to the host BIOS.
