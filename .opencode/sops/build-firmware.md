# Build Firmware — SOP

## Prerequisites
- ARM GCC toolchain: `arm-none-eabi-gcc` (v14.x from xpack-dev-tools)
- CMake >= 3.13
- Ninja build system
- Raspberry Pi Pico SDK 2.2.0 (set `PICO_SDK_PATH` env var)
- FreeRTOS-Kernel (submodule, Raspberry Pi fork)

## Build Steps

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

## Outputs
Located in `build/`:
- `flipperone-mcu-firmware.uf2` — deployable firmware
- `flipperone-mcu-firmware.elf` — ELF with debug symbols
- `flipperone-mcu-firmware.bin` — raw binary
- `flipperone-mcu-firmware.hex` — Intel HEX
- `flipperone-mcu-firmware.dis` — disassembly
- `flipperone-mcu-firmware.elf.map` — linker map

## Clean Build
```bash
rm -rf build && cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build --parallel
```

## CI Reference
The CI pipeline in `.github/workflows/build.yml` also runs this exact build sequence on pushes to main/PRs.
