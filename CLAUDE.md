# CrossPoint Reader - Custom Build Notes

## Project Overview
CrossPoint Reader is an open-source firmware replacement for the Xteink X4 e-paper display reader. It targets the ESP32-C3 microcontroller with ~380KB usable RAM.

- **Repo**: https://github.com/crosspoint-reader/crosspoint-reader
- **Platform**: PlatformIO + Arduino framework
- **Display**: 800x480 e-ink (480x800 in portrait mode)
- **Rendering**: Binary 1-bit monochrome (black/white), with 4-level greyscale support for sleep screens

## Current Branch: `custom-v16`

**Base**: v0.16.0 + PRs #442, #506, #433

Custom v0.16.0 build with popup refactoring, power button hold duration, and reading menu features.

### Recent Commits (newest first)
| Commit | Description |
|--------|-------------|
| cae74ec | build: Update v16.0 custom firmware with serialization fix |
| acc33d4 | fix: Move powerButtonHoldDuration to end of serialization |
| 7c49a2a | docs: Update CLAUDE.md for v16.0 custom build |
| bbbde71 | fix: Add null checks for section in delete cache |
| 8cc4618 | Merge branch 'pr-433' into custom-v16 |
| e83a70c | Merge branch 'pr-506' into custom-v16 |
| fd0e025 | Merge branch 'pr-442' into custom-v16 |

## Repository Structure

**Remotes:**
- `origin` → Private fork: https://github.com/nkocher/crosspoint-reader-custom.git
- `upstream` → Main repo: https://github.com/crosspoint-reader/crosspoint-reader

**Branches:**
- `master` - Tracks v0.16.0
- `custom-v16` - Your custom build (current)
- `backup/theme-clarity-2026-01-27` - Theme work backup

**Tags:**
- `v16.0-custom` - Current custom release
- `backup-pre-v16-cleanup` - Backup before cleanup

## Project Structure

```
src/
  activities/           # UI screens
    home/              # Home screen
    boot_sleep/        # Boot and sleep screens (greyscale support)
    settings/          # Settings menu
    reader/            # EPUB/TXT/XTC reader activities
    network/           # WiFi, file transfer
  CrossPointSettings.h # Settings system
  CrossPointState.h    # Runtime state (openEpubPath, etc.)

lib/
  GfxRenderer/         # Display rendering
    Bitmap.cpp         # BMP loading with dithering
    GfxRenderer.cpp    # Drawing primitives
  hal/                 # Hardware abstraction
    HalDisplay.h       # Display wrapper
    HalGPIO.h          # GPIO wrapper
  Epub/                # EPUB parsing and rendering
  Txt/                 # TXT file rendering

open-x4-sdk/          # Hardware SDK (submodule)
  libs/display/EInkDisplay/  # E-ink driver with greyscale LUT support
```

## Build & Verification Commands

```bash
# Build
pio run

# Flash (when device is connected)
pio run -t upload

# If wrong port detected (e.g., Bluetooth speaker), specify explicitly:
pio run -t upload --upload-port /dev/cu.usbmodem144101

# Or flash specific binary
esptool.py --chip esp32c3 write_flash 0x10000 builds/firmware-v16.0-custom.bin

# Monitor serial output
# pio device monitor -b 115200  # Often fails with termios error
stty -f /dev/cu.usbmodem144101 115200 && cat /dev/cu.usbmodem144101  # Reliable alternative

# Static analysis
pio check --fail-on-defect low --fail-on-defect medium --fail-on-defect high

# Format check
./bin/clang-format-fix && git diff --exit-code
```

**Pre-commit Hook:**
- Runs automatically on `git commit` (format → build → cppcheck)
- Takes ~2 minutes to complete
- Background commits write to `/private/tmp/claude/.../tasks/*.output`
- Use `--no-verify` to skip hook when re-committing after validation

## Syncing with Upstream

Pull new features from main repo while maintaining custom changes:
```bash
# Fetch and update master from upstream
git fetch upstream
git checkout master
git merge upstream/master
git push origin master

# Integrate upstream updates into custom branch
git checkout custom-v16
git merge master
# Resolve conflicts, prioritizing upstream fixes over custom features
git push origin custom-v16
```

Merge specific PR from upstream:
```bash
# Fetch PR branch
git fetch upstream pull/<PR_NUMBER>/head:pr-<PR_NUMBER>

# Merge into custom branch
git checkout custom-v16
git merge pr-<PR_NUMBER> --no-edit
# Or cherry-pick: git cherry-pick <commit-hash>
```

## Settings System Critical Patterns

**CRITICAL: Settings Serialization Order**
- `src/CrossPointSettings.cpp` - saveToFile() and loadFromFile() must use IDENTICAL order
- New settings MUST be added at END of sequence (after last field, before "New fields" comment)
- NEVER insert new settings in middle - breaks backward compatibility with existing settings.bin files
- Increment `SETTINGS_COUNT` constant when adding new settings
- Pattern for new field:
  ```cpp
  serialization::writePod(outputFile, sleepScreenCoverFilter);
  serialization::writePod(outputFile, newSettingHere);  // ADD AT END
  // New fields added at end for backward compatibility
  ```

**Settings Corruption Recovery:**
- Symptom: `abort() at PC 0x420f8b3f` crash after SD card detection
- Cause: Mismatch between firmware's expected settings order and SD card's settings.bin
- Fix: Remove SD card to boot with defaults, or reflash correct firmware

## Important Files for Common Issues

| Issue | Check These Files |
|-------|-------------------|
| Settings not saving | `src/CrossPointSettings.cpp` - descriptor serialization order |
| Settings merge conflicts | Increment `SETTINGS_COUNT`, preserve save/load order |
| Display artifacts | `open-x4-sdk/libs/display/EInkDisplay/src/EInkDisplay.cpp` - refresh modes |
| Reader crashes | Check null-safety: `section ? section->field : defaultValue` pattern |

## Code Style Patterns

**House Style** (follows existing codebase conventions):
- No documentation comments unless code is not self-explanatory
- Flatten callbacks: expose actions as private methods instead of nested lambdas
- Null-check pattern: `section ? section->currentPage : 0`

**Settings System:**
- Adding new setting requires incrementing `SETTINGS_COUNT` constant
- Settings serialize in order: `serialization::writePod(outputFile, fieldName)`
- Use `readAndValidate(file, field, MAX_VALUE)` for enums with bounds
- Load order must match save order exactly

## v16.0 PR Integration Fixes

**PR #506: Settings Serialization Order**
- **Issue**: Added `powerButtonHoldDuration` in middle of serialization sequence
- **Impact**: Broke backward compatibility, caused abort() crashes when loading old settings.bin
- **Fix**: Moved to end of sequence in CrossPointSettings.cpp:152
- **Pattern**: Always add new settings at END (see Settings System section above)

**PR #433: Delete Cache Null Checks**
- **Issue**: `section` pointer could be null when deleting cache
- **Fix**: Added null checks in EpubReaderActivity.cpp before accessing section->currentPage
- **Pattern**: `uint16_t backupPage = section ? section->currentPage : 0;`

## Backup Firmware Files

Located in `builds/`:
- `firmware-v16.0-custom.bin` - Current custom build (v16.0 + PRs)
- Rollback: `esptool.py --chip esp32c3 write_flash 0x10000 builds/firmware-v16.0-custom.bin`

## Integrated PRs

| PR | Feature | Status |
|----|---------|--------|
| #442 | Popup logic refactoring | Merged to custom-v16 |
| #506 | Power button hold duration | Merged to custom-v16 |
| #433 | Reading menu + delete cache | Merged to custom-v16 (with null-check fix) |
| #511 | Display epub metadata on Recents | Included in v0.16.0 base |
| #522 | HAL abstraction (HalDisplay, HalGPIO) | Included in v0.16.0 base |

## IDE Notes

Language server diagnostics (missing headers, unknown types) are false positives -
the IDE isn't configured for ESP32/FreeRTOS embedded development. If `pio run` succeeds, the code is correct.
