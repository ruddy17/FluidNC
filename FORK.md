# Local ESP32-S3 controller fork

This fork follows `bdring/FluidNC` and retains two changes needed by the
ESP32-S3 Super Mini CoreXY controller:

- `wifi_s3_4m`: a 3 MiB factory application and 960 KiB LittleFS on 4 MB flash.
  Firmware OTA is disabled for this layout; updates use USB/serial.
- CoreXY/TMC2209 homing cleanup: refresh released coupled limit inputs and
  disable the StallGuard window when returning to ordinary StealthChop.
  This fix was merged upstream in PR #1910, including a guard that prevents
  homing from continuing after an abort during the initial limit-refresh dwell.

`main` contains both changes on top of upstream. The separate
`codex/corexy-homing-fix` branch is the source of
[merged upstream PR #1910](https://github.com/bdring/FluidNC/pull/1910) and contains
only the homing fix, without the partition changes. `main` includes the
maintainer's dwell guard from commit `57100b8d` as a separate backport.

## Build

With PlatformIO installed in the active Python environment:

```sh
python -m platformio run -e wifi_s3_4m
python -m platformio run -e wifi_s3_4m -t build_merged
python tools/check_s3_4m_image.py .pio/build/wifi_s3_4m
```

See [the 4 MB layout instructions](FluidNC/ld/esp32s3/4MB.md) for offsets
and installation details. An application-only update writes `firmware.bin`
at `0x10000` and preserves the existing filesystem and machine configuration.

## FigUI and machine configuration

The repository's tracked `FluidNC/data` remains the upstream default.
Machine settings and downloaded UI files are local build inputs, not part
of the public fork. The canonical stage configuration is maintained in the
BoardScope project, separately from this firmware repository.

For the existing setup, `build/figui-data` contains the preserved
[FigUI v1.3.2](https://github.com/figamore/FigUI/releases/tag/v1.3.2) assets
and a copy of the stage configuration. To recreate this directory on another
computer, download that release's `index.html.gz`, copy `favicon.ico` from
`FluidNC/data`, and add your machine's `config.yaml`. Refresh that config
copy before building an installation image.

On PowerShell, build the combined image using those files:

```powershell
$env:PLATFORMIO_DATA_DIR = (Resolve-Path build/figui-data).Path
try {
    python -m platformio run -e wifi_s3_4m -t build_merged
    if ($LASTEXITCODE -ne 0) { throw 'Combined image build failed' }
    python tools/check_s3_4m_image.py .pio/build/wifi_s3_4m
    if ($LASTEXITCODE -ne 0) { throw 'Image validation failed' }
} finally {
    Remove-Item Env:PLATFORMIO_DATA_DIR -ErrorAction SilentlyContinue
}
```

The UI does not need to be reflashed when updating only the application.
