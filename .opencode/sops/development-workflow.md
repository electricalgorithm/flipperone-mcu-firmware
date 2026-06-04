# Development Workflow — SOP

## End-to-End flow

1. **Understand** the codebase structure:
   - `applications/` — firmware apps (services, main apps, debug apps)
   - `targets/` — board-specific code (HAL, BSP, main entry)
   - `lib/` — corelibs, drivers, FreeRTOS, TinyUSB
   - `assets/` — compiled assets

2. **Make changes** following existing patterns:
   - C11 / C++20 code
   - Follow FURI framework conventions in `lib/corelibs/`
   - Use the FURI HAL (`targets/*/furi_hal/`) for hardware access

3. **Build** using the build SOP (`.opencode/sops/build-firmware.md`)

4. **Flash & test** using the flash SOP (`.opencode/sops/flash-deploy.md`)

5. **Debug**:
   - RTT/SEGGER logging via `lib/segger_rtt/`
   - UART serial console output
   - TinyUSB for USB device communication
   - FreeRTOS trace/debug features

## Architecture Notes

The Flipper One has a dual-processor architecture:
- **Low-Power MCU**: RP2350 (this firmware) — LCD, buttons, touchpad, battery, power states
- **High-Performance CPU**: Rockchip RK3576 running Linux — USB, HDMI, Wi-Fi, Ethernet, audio
- **Inter-Core Comms**: SPI, I2C, UART between the two

## MCP Tools Available
- **picotool**: Query device info, reboot, flash (RP2350 operations)
- **GitHub**: PRs, issues, repo management
- **Filesystem**: File read/write operations
- **Serial**: Serial communication with devices
