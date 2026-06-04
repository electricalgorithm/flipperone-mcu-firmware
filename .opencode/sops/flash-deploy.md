# Flash & Deploy — SOP

## Via picotool (BOOTSEL mode)
1. Put the Pico in BOOTSEL mode (hold BOOTSEL button while connecting USB, or use `picotool_reboot` with `force: true, usb_mass_storage: true`)
2. Flash the UF2:
   ```
   picotool load build/flipperone-mcu-firmware.uf2
   ```
3. Reboot to application mode:
   ```
   picotool reboot
   ```

## Using OpenCode MCP tools (picotool MCP)
Use the `picotool_info` tool with `force: true` to auto-reboot a running device to BOOTSEL mode. Use the system shell (bash) for `picotool load` if direct command execution is available.

## Verify
After flashing, verify the device:
```
picotool info --device
```
Or use the `picotool_info` MCP tool with `device: true`.

## Release Builds
CI produces release UF2 artifacts pushed to `update.flipperzero.one/builds/flipper-one-mcu/`.
