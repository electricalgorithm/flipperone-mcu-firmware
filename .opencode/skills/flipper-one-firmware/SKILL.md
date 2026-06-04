---
name: flipper-one-firmware
description: >
  Help users work with the Flipper One MCU firmware — the RP2350-based low-power co-processor for the Flipper One dual-processor device.
  Use this skill whenever someone asks about building, modifying, debugging, or understanding this firmware.
  It covers: adding services and applications, writing I2C/SPI/UART hardware drivers, modifying the FURI framework, working with the RP2350 Pico SDK,
  FreeRTOS configuration, the Clay-based GUI, power management (BQ25792 charger), the JD9853 display, IQS7211E touchpad, DRV2605L haptic,
  WS2812 PIO LEDs, I2C intercom with the Linux CPU, and the CLI system.
  This project uses CMake + Ninja, C11/C++20, FreeRTOS, the Pico SDK 2.2.0, and the FURI application framework.
  Apply this skill when the user references targets/, lib/drivers/, applications/, or lib/corelibs/,
  or asks any firmware-level question about this repo.
compatibility:
  - CMake >= 3.13
  - ARM GCC (arm-none-eabi) toolchain
  - Raspberry Pi Pico SDK 2.2.0
  - Ninja build system
  - picotool (flashing)
  - OpenOCD (debugging)
---

# Flipper One MCU Firmware

This firmware runs on the **Raspberry Pi RP2350** (dual Cortex-M33 / RISC-V) — the low-power co-processor in the Flipper One dual-processor device. It controls the LCD display, buttons, touchpad, LEDs, haptic motor, battery management, USB Power Delivery, and power states. It communicates with the main Linux CPU (Rockchip RK3576) over SPI, I2C, and UART.

## Quick Start

### Prerequisites

Dependencies required to build (installed separately, not in the repo):

- **ARM GCC toolchain** (arm-none-eabi-gcc) — the CI uses xpack version 14.2.1-1.1
- **Raspberry Pi Pico SDK** 2.2.0
- **CMake** >= 3.13
- **Ninja** build system
- **picotool** or **OpenOCD** for flashing

Git submodules (update after cloning):

```bash
git submodule update --init --recursive
```

### Building

```bash
# Configure (from project root):
mkdir -p build
PICO_SDK_PATH=/path/to/pico-sdk cmake -B build -DCMAKE_BUILD_TYPE=Release

# Build:
cmake --build build --config Release --parallel $(nproc)
```

Build outputs in `build/`:
- `flipperone-mcu-firmware.uf2` — Flashable UF2 binary
- `flipperone-mcu-firmware.elf` — ELF for debugging
- `flipperone-mcu-firmware.hex` / `.bin` / `.dis` / `.elf.map`

CMake variables you can override:
- `PICO_BOARD` (default `pimoroni_pga2350`)

### Flashing

```bash
# Via picotool (UF2):
picotool load build/flipperone-mcu-firmware.uf2 -fx

# Via OpenOCD with CMSIS-DAP:
openocd -s /path/to/openocd/scripts \
  -f interface/cmsis-dap.cfg \
  -f target/rp2350.cfg \
  -c "adapter speed 5000; program build/flipperone-mcu-firmware.elf verify reset exit"
```

### Debugging

- SEGGER RTT is integrated for real-time logging output
- OpenOCD with CMSIS-DAP for GDB debugging (Cortex-Debug extension in VS Code)
- The `.vscode/launch.json` has preconfigured launch and attach configurations

## Firmware Architecture

The firmware is organized in layers:

```
┌─────────────────────────────────────────┐
│ Applications (main/, debug/, services/) │  ← Event-loop driven
├─────────────────────────────────────────┤
│ GUI Service (Clay + JD9853 display)     │  ← Immediate-mode UI
├─────────────────────────────────────────┤
│ Application Framework (FURI)            │  ← Threads, events, records, pubsub
├─────────────────────────────────────────┤
│ Hardware Abstraction Layer (furi_hal/)  │  ← I2C, SPI, UART, GPIO, PWM, ADC, USB
├─────────────────────────────────────────┤
│ Board Support Package (furi_bsp/)       │  ← Board init, expanders, stdio
├─────────────────────────────────────────┤
│ Hardware Drivers (lib/drivers/)         │  ← Chip-level drivers
├─────────────────────────────────────────┤
│ Pico SDK + FreeRTOS                     │  ← MCU HAL + RTOS
└─────────────────────────────────────────┘
```

### Startup Flow

In `targets/*/src/main.c`:
1. `furi_hal_memory_init()` — Memory subsystem
2. `furi_init()` — FURI kernel
3. `furi_hal_log_init()` — Logging (UART)
4. `furi_hal_init_early()` — Early HAL (Cortex, NVM, OS, I2C buses)
5. Spawn `init_task` thread:
   - `furi_hal_log_hardware_init()` — UART for logging
   - `furi_hal_init()` — GPIO, ADC, OTP, interrupts
   - `furi_bsp_init()` — GPIO expander, stdio via USB CDC
   - `flipper_init()` → starts all services and applications from `applications/applications.c`
6. Reset core 1 (workaround for OpenOCD multicore issue)
7. `furi_run()` — Start FreeRTOS scheduler

## Directory Map

| Path | Contents |
|------|----------|
| `targets/*/` | Target-specific code (main.c, HAL, BSP, config) |
| `targets/*/furi_hal/` | Hardware abstraction (GPIO, I2C, SPI, UART, ADC, PWM, power, USB, flash, interrupts, clocks, NVM, OTP, version, memory) |
| `targets/*/furi_bsp/` | Board support (expander init, stdio init, assert handler) |
| `targets/*/config/` | Board configuration (FURI config, TinyUSB, linker) |
| `applications/` | All services, apps, debug tools |
| `applications/services/` | 18 long-lived system services (gui, input, power, desktop, haptic, led, usb, pd, i2c_intercom, i2c_negotiator, cli, etc.) |
| `applications/main/` | User applications (cpu, self_check) |
| `applications/debug/` | Debug/test apps (keypad_test, touchpad_test, haptic_test) |
| `applications/template/` | App template to copy for new apps |
| `lib/corelibs/` | FURI framework (thread, event_loop, record, pubsub, log, check, string, mutex, semaphore, kernel) |
| `lib/drivers/` | 18 hardware drivers (display JD9853, haptic DRV2605L, touch IQS7211E, charger BQ25792, fuel gauge BQ28Z620, current INA219/INA4230, USB PD FUSB302, GPIO expanders TCA6416A/PCAL6416, etc.) |
| `lib/freertos/` | FreeRTOS kernel (Raspberry Pi fork, RP2350 port) |
| `lib/cli/` | CLI shell framework (args, ANSI, auto-complete) |
| `lib/toolbox/` | Utilities (hex, color, strint, callbacks) |
| `lib/pico-kvstore/` | Key-value store on littlefs VFS |
| `lib/rtt/` | SEGGER RTT real-time logging |
| `lib/tusb/` | TinyUSB descriptors |
| `lib/uart_pio/` | UART via PIO |
| `lib/containers/` | Pipe/FIFO data structures |
| `assets/` | Images/icons → converted to C at build time |
| `docs/` | Display datasheets and configuration |

## Code Conventions

### General Rules

- **Language**: C11 (firmware), C++20 (some drivers)
- **Naming**: `snake_case` for functions/variables, `PascalCase` for types, `SCREAMING_SNAKE` for macros/enums
- **Header guards**: `#pragma once` everywhere
- **Struct opacity**: Forward-declare in `.h`, define in `.c` — users only see pointers
- **Allocation**: `_alloc()` returns a malloc'd instance, `_free()` frees it. Services allocate once and never free.
- **Entry points**: `int32_t service_srv(void* p)` for services, `int32_t app_bodyname(void* p)` for apps
- **C++ interop**: Wrap public declarations in `#ifdef __cplusplus extern "C" { #endif`
- **Unused parameters**: Use the `UNUSED(x)` macro

### Tag and Logging

Every module starts with:
```c
#define TAG "ModuleName"
```

Log levels:
```c
FURI_LOG_E(TAG, "Error: %d", code);   // error
FURI_LOG_W(TAG, "Warn: %s", msg);     // warning
FURI_LOG_I(TAG, "Info");              // info
FURI_LOG_D(TAG, "Debug: %d", val);    // debug
FURI_LOG_T(TAG, "Trace");             // trace
```

Drivers may use conditional debug logging:
```c
#ifdef DRIVER_DEBUG_ENABLE
#define DRIVER_DEBUG(...) FURI_LOG_D(__VA_ARGS__)
#else
#define DRIVER_DEBUG(...)
#endif
```

### Assertions / Error Handling

```c
furi_assert(cond);   // Debug-only
furi_check(cond);    // Always-on (crashes on failure)
furi_crash("msg");   // Hard crash
furi_halt("msg");    // Halt CPU
```

Use `furi_check` for precondition checks at the start of every public function.

### ISR Functions

Interrupt handlers must be marked with special attributes:
```c
static void __isr __not_in_flash_func(handler_name)(void* ctx) {
    // ISR-safe code only
}
```

Note: Flash is currently assumed safe for core 1 (`PICO_FLASH_ASSUME_CORE1_SAFE=1`). If multi-core access to flash becomes needed, this define must be removed and proper flash access synchronization implemented.

### I2C Communication (Standard Pattern)

Always acquire the bus handle before any transaction and release after:

```c
static int device_write_reg(Device* inst, uint8_t reg, uint8_t data) {
    furi_check(inst);
    uint8_t buf[2] = {reg, data};
    furi_hal_i2c_acquire(inst->i2c_handle);
    int ret = furi_hal_i2c_master_tx_blocking(
        inst->i2c_handle, inst->address, buf, sizeof(buf), FURI_HAL_I2C_TIMEOUT_US);
    furi_hal_i2c_release(inst->i2c_handle);
    return ret;
}
```

Available I2C bus handles (defined in `furi_hal_i2c_config.h`):
- `furi_hal_i2c_handle_main` — main bus (display, touch, haptic, etc.)
- `furi_hal_i2c_handle_control` — control bus (charger, fuel gauge, USB PD, etc.)

I2C functions available:
- `furi_hal_i2c_acquire()` / `furi_hal_i2c_release()`
- `furi_hal_i2c_master_tx_blocking()` / `_nostop` variant
- `furi_hal_i2c_master_rx_blocking()` / `_nostop` variant
- `furi_hal_i2c_master_trx_blocking()`  (combined tx + rx with restart)
- `furi_hal_i2c_device_ready()` — probe for device existence
- Slave mode: `furi_hal_i2c_slave_set_callback()`, `_write_blocking`, `_read_blocking`, `_bus_reset`

## Key Code Patterns

### Service Pattern

Services are the building blocks of the firmware — long-lived threads running an event loop:

```c
struct MyService {
    FuriEventLoop* event_loop;
    FuriMessageQueue* message_queue;
    // driver handles, state, etc.
};

static MyService* service_alloc(void) {
    MyService* inst = malloc(sizeof(MyService));
    inst->event_loop = furi_event_loop_alloc();
    inst->message_queue = furi_message_queue_alloc(8, sizeof(ServiceMessage));

    // Init drivers
    inst->driver = driver_init(inst->i2c_handle);

    // Subscribe queue to event loop
    furi_event_loop_subscribe_message_queue(
        inst->event_loop, inst->message_queue,
        FuriEventLoopEventIn, message_callback, inst);

    // Publish for other modules
    furi_record_create(RECORD_MYSERVICE, inst);
    return inst;
}

int32_t my_service_srv(void* p) {
    UNUSED(p);
    MyService* inst = service_alloc();
    furi_event_loop_run(inst->event_loop);
    return 0;
}
```

Message queues + FuriApiLock pattern enables thread-safe public API:
```c
// In header:
void my_service_do_thing(void);

// In .c:
typedef struct {
    MyServiceMessageType type;
    FuriApiLock lock;      // blocking result sync
    uint32_t result;
} MyServiceMessage;

// Public API:
void my_service_do_thing(void) {
    MyService* inst = furi_record_open(RECORD_MYSERVICE);
    MyServiceMessage msg = { .type = MyServiceMessageTypeDoThing, .lock = furi_api_lock_alloc() };
    furi_check(furi_message_queue_put(inst->message_queue, &msg, FuriWaitForever) == FuriStatusOk);
    furi_api_lock_wait(msg.lock);    // block until service processes it
    furi_api_lock_free(msg.lock);
    furi_record_close(RECORD_MYSERVICE);
}

// In message queue callback:
static void message_callback(FuriMessageQueue* queue, void* context) {
    MyService* inst = context;
    MyServiceMessage msg;
    while(furi_message_queue_get(inst->message_queue, &msg, 0) == FuriStatusOk) {
        switch(msg.type) {
            case MyServiceMessageTypeDoThing:
                msg.result = do_thing(inst);
                furi_api_lock_release(msg.lock);
                break;
        }
    }
}
```

Register the service in `applications/applications.c` → `FLIPPER_SERVICES[]`.

### Application Pattern (with GUI)

Applications use Views — UI elements managed by the GUI service:

```c
typedef struct { /* model data */ } AppModel;

typedef struct {
    Gui* gui;
    View* view;
    FuriEventLoop* event_loop;
} App;

// Layout callback (Clay rendering)
static bool app_layout(void* _model) {
    AppModel* model = _model;
    CLAY(CLAY_ID("Container"), LAYOUT_DEFAULTS) {
        CLAY_TEXT(CLAY_STRING("Hello"), TEXT_CONFIG);
    }
    return false;  // false = no additional redraw
}

// Input callback
static bool app_input(InputEvent* event, void* context) {
    // handle button press
    return true;  // true = consumed
}

static App* app_alloc(void) {
    App* inst = malloc(sizeof(App));
    inst->gui = furi_record_open(RECORD_GUI);
    inst->event_loop = furi_event_loop_alloc();

    inst->view = view_alloc();
    view_allocate_model(inst->view, ViewModelTypeLockFree, sizeof(AppModel));
    view_set_layout_callback(inst->view, app_layout);
    view_set_input_callback(inst->view, app_input, inst);
    gui_add_view(inst->gui, inst->view, GuiViewPriorityApplication);
    return inst;
}

// Entry point (register in FLIPPER_APPS[] or FLIPPER_AUTORUN_APPS[])
int32_t app_body(void* p) {
    UNUSED(p);
    App* inst = app_alloc();
    furi_event_loop_run(inst->event_loop);
    // cleanup
    return 0;
}
```

View priorities (from `gui.h`):
- `GuiViewPriorityDesktop = 0` — background
- `GuiViewPriorityApplication = 50000` — normal app
- `GuiViewPriorityMenu = 100000` — overlay

### Event Loop API

`FuriEventLoop` is the central dispatch mechanism. Subscribe any synchronisation primitive:

```c
furi_event_loop_subscribe_message_queue(loop, mq, FuriEventLoopEventIn, cb, ctx);
furi_event_loop_subscribe_event_flag(loop, flag, FuriEventLoopEventIn, cb, ctx);
furi_event_loop_subscribe_stream_buffer(loop, sb, FuriEventLoopEventIn, cb, ctx);
furi_event_loop_subscribe_semaphore(loop, sem, FuriEventLoopEventIn, cb, ctx);
furi_event_loop_subscribe_mutex(loop, mtx, FuriEventLoopEventIn, cb, ctx);
furi_event_loop_set_custom_event_callback(loop, cb, ctx);  // ISR→thread
furi_event_loop_tick_set(loop, interval_ms, cb, ctx);      // periodic tick

furi_event_loop_run(loop);   // blocks
furi_event_loop_stop(loop);  // from another callback
```

### Hardware Driver Pattern

```c
struct MyDriver {
    const FuriHalI2cBusHandle* i2c_handle;
    uint8_t address;
    // GPIO pin config, state, etc.
};

MyDriver* my_driver_init(const FuriHalI2cBusHandle* i2c_handle) {
    MyDriver* inst = malloc(sizeof(MyDriver));
    inst->i2c_handle = i2c_handle;
    // ... I2C config, GPIO init ...
    return inst;
}

void my_driver_deinit(MyDriver* inst) {
    furi_check(inst);
    // ... gpio deinit, free ...
    free(inst);
}
```

### Record API (IPC Between Modules)

```c
// Publishing side (typically in service alloc):
furi_record_create("record_name", pointer_to_instance);

// Consuming side:
MyType* inst = furi_record_open("record_name");
// ... use inst ...
furi_record_close("record_name");
```

Known record names (from service headers):
- `RECORD_GUI "Gui"`
- `RECORD_POWER "power"`
- `RECORD_HAPTIC "haptic"`
- `RECORD_INPUT_EVENTS "input_events"`
- `RECORD_INPUT_TOUCH_EVENTS "input_touch_events"`
- `RECORD_LED "led"`
- `RECORD_USB "usb"`
- `RECORD_DESKTOP "desktop"`
- `RECORD_NOTIFICATION "notification"`

### Threading

```c
// Regular thread (app pattern):
FuriThread* thread = furi_thread_alloc_ex("Name", stack_size, callback, context);
furi_thread_set_priority(thread, FuriThreadPriorityNormal);
furi_thread_start(thread);
furi_thread_join(thread);
furi_thread_free(thread);

// Service threads (more efficient, cannot be joined/freed):
// Use furi_thread_alloc_service() — but the service pattern above handles this
// automatically when registered via FLIPPER_SERVICES[].

// Thread flags (lightweight ISR→thread signaling):
// In ISR: furi_thread_flags_set(thread_id, FLAG_BIT);
// In thread: furi_thread_flags_wait(FLAG_BIT, FuriFlagWaitAny, FuriWaitForever);
```

### GUI / Clay UI

The GUI uses **Clay** (v0.14), an immediate-mode layout library. Applications define layouts in callbacks using the CLAY() macro hierarchy. The renderer converts Clay command arrays to the JD9853 QSPI display buffer.

Helper macros in `clay_helper.h`:
- `CLAY_APP_ID(x)` — prefix element IDs with module TAG
- `clay_helper_string_from(furi_string)` — FuriString → Clay_String
- `clay_helper_string_from_chars(c_string)` — char* → Clay_String
- `clay_fixed_image(image)` — Place a bitmap image

See `applications/services/gui/clay_render.h` for available render primitives.

## Build Configuration Details

Key build defines (from `CMakeLists.txt`):
- `PICO_DEFAULT_UART_TX_PIN=0`, `PICO_DEFAULT_UART_RX_PIN=1`, baud 230400
- `PARAM_ASSERTIONS_ENABLE_ALL=1`
- `PICO_FLASH_ASSUME_CORE1_SAFE=1` — single-core flash assumption
- `USE_PRINTF`, `USE_DBG_PRINTF`
- `PICO_USE_MALLOC_MUTEX=0` — no malloc mutex (single-core)

Standard libraries linked (see CMakeLists.txt `target_link_libraries`):
`pico_stdlib`, `hardware_pwm`, `hardware_spi`, `hardware_i2c`, `hardware_adc`, `hardware_dma`, `hardware_uart`, `hardware_watchdog`, `hardware_clocks`, `pico_unique_id`, `tinyusb_device`, `uart_pio`, `ws2812`, `pio_i2c`, `i2c_multi`, `freertos`, `cli`, `containers`, `mlib`, `kvstore`, `rtt`, `assets`, `cmsis_core`

## Important Hardware Details

### I2C Buses
| Bus | Handle | Devices |
|-----|--------|---------|
| i2c0 | `furi_hal_i2c_handle_control` | BQ25792, BQ28Z620, INA219, INA4230, FUSB302, HD3SS3220, TPS62868x |
| i2c1 | `furi_hal_i2c_handle_main` | IQS7211E, DRV2605L, TCA6416A, PCAL6416 |
| i2c2 | CPU intercom | I2C slave to main CPU |

### Other Interfaces
- **Display**: JD9853 via QSPI (dual/quad SPI, not I2C)
- **LEDs**: WS2812B addressable LEDs via PIO
- **Buttons**: 13 physical buttons read via TCA6416A GPIO expander (I2C)
- **Touch**: IQS7211E capacitive touch controller (I2C, 4-wire touchpad)
- **Haptic**: DRV2605L with LRA actuator (I2C)
- **USB**: Device mode via TinyUSB (CDC serial for VCP)
- **Debug UART**: UART0 on GP0/GP1, 230400 baud
- **CPU UART**: UART1 for main CPU communication
