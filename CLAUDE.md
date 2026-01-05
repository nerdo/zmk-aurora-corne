# ZMK Aurora Corne Configuration Guide

## Critical ZMK Quirks & Lessons Learned

### 1. Shield Prefix Matching Applies User Config Files

**Problem**: ZMK applies `config/<prefix>.overlay`, `config/<prefix>.keymap`, and `config/<prefix>.conf` to ANY shield that starts with that prefix.

**Example**: Shield `splitkb_aurora_corne_dongle_xiao` will load:
- `config/splitkb_aurora_corne.overlay` (prefix match!)
- `config/splitkb_aurora_corne.keymap` (prefix match!)
- `config/splitkb_aurora_corne.conf` (prefix match!)

**Impact**: If the overlay/keymap references nodes that don't exist on all boards (e.g., `&pro_micro_i2c`, `&led_strip`, `&nice_view_spi`), builds for boards without those nodes will fail.

**Solution**: Use board-specific overlays (`config/boards/<board>.overlay`) for hardware-specific configuration. These only apply to builds using that specific board.

### 2. Board Overlay Filename Must Match Board Identifier

**Problem**: The board overlay filename must match the board identifier used in `build.yaml`.

**Example**:
- Board in build.yaml: `nice_nano@2.0.0`
- Correct overlay: `config/boards/nice_nano.overlay`
- WRONG: `config/boards/nice_nano_v2.overlay` (won't be loaded!)

**Impact**: If filename doesn't match, the overlay is silently ignored and hardware won't be configured.

### 3. User Config Board Overlays Load AFTER Shields

**Correct Loading Order**:
1. Board DTS
2. ZMK board overlays
3. Shield overlays (defines nodes like `&led_strip`, `&nice_view_spi`)
4. User config overlays (`config/boards/*.overlay`, `config/<shield>.overlay`)
5. Keymap (last)

**Impact**: You CAN reference shield-defined nodes in `config/boards/<board>.overlay` because shields load first.

### 4. Dongle-Specific Files May Not Override Prefix Matches

**Problem**: Even if `config/splitkb_aurora_corne_dongle_xiao.keymap` exists, ZMK may still use `config/splitkb_aurora_corne.keymap` via prefix matching.

**Observed Behavior**: The build log shows which keymap is used:
```
-- Using keymap file: .../config/splitkb_aurora_corne.keymap
```

**Solution**: Don't rely on shield-specific files overriding prefix matches. Instead, ensure prefix-matched files don't contain board-specific references.

### 5. Nodes That Don't Exist on All Boards

These nodes are board/shield specific and will cause build failures if referenced for the wrong board:

| Node | Exists On | Does NOT Exist On |
|------|-----------|-------------------|
| `&pro_micro_i2c` | nice_nano (Pro Micro pinout) | xiao_ble |
| `&led_strip` | Aurora Corne shield | Dongle (no LEDs) |
| `&nice_view_spi` | nice_view shield | Dongle (no display) |
| `&pro_micro` | nice_nano | xiao_ble |

### 6. Dongle Requirements

A ZMK dongle needs:
1. **Matrix transform** - Must match the keyboard layout even though dongle has no keys
2. **Mock kscan** - Use `zmk,kscan-mock` with 0 columns/rows/events
3. **Separate keymap** - Minimal keymap with `&none` bindings, NO device refs
4. **BLE central config** - `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`

## Current Architecture

### File Structure
```
config/
├── boards/
│   └── nice_nano.overlay          # nice_nano-specific hardware (SPI0, nice_view, LEDs)
├── boards/shields/splitkb_aurora_corne/
│   ├── Kconfig.shield             # Dongle shield declaration
│   ├── Kconfig.defconfig          # Dongle BLE central config
│   └── splitkb_aurora_corne_dongle_xiao.overlay  # Dongle matrix/kscan
├── splitkb_aurora_corne.keymap    # Main keymap (NO device refs!)
├── splitkb_aurora_corne.conf      # Main config
├── splitkb_aurora_corne_dongle_xiao.keymap  # Dongle keymap (minimal)
└── splitkb_aurora_corne_dongle_xiao.conf    # Dongle config
```

### Build Targets
- `nice_nano@2.0.0` + `splitkb_aurora_corne_left` - Left keyboard half
- `nice_nano@2.0.0` + `splitkb_aurora_corne_right` - Right keyboard half
- `xiao_ble` + `splitkb_aurora_corne_dongle_xiao` - Wireless dongle
- `nice_nano@2.0.0` + `settings_reset` - Reset nice_nano settings
- `xiao_ble` + `settings_reset` - Reset xiao_ble settings

## Debugging Tips

### Check Which Files ZMK Loads
Look for these lines in build output:
```
-- ZMK Config devicetree overlay: ...
-- Using keymap file: ...
-- Found devicetree overlay: ...
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `undefined node label 'pro_micro_i2c'` | Overlay applied to xiao_ble | Move to `config/boards/nice_nano.overlay` |
| `undefined node label 'led_strip'` | Keymap applied to dongle | Remove from keymap, put in board overlay |
| `undefined node label 'nice_view_spi'` | Keymap applied to dongle | Remove from keymap, put in board overlay |
| `Need a matrix transform` | Dongle missing transform | Add matrix transform to dongle overlay |
