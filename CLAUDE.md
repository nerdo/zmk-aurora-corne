# ZMK Aurora Corne Configuration Guide

## Critical ZMK Quirks & Lessons Learned

### 1. Shield Prefix Matching Applies User Config Files

**Problem**: ZMK applies `config/<prefix>.overlay`, `config/<prefix>.keymap`, and `config/<prefix>.conf` to ANY shield that starts with that prefix.

**Example**: Shield `splitkb_aurora_corne_dongle_xiao` will load:
- `config/splitkb_aurora_corne.overlay` (prefix match!)
- `config/splitkb_aurora_corne.keymap` (prefix match!)
- `config/splitkb_aurora_corne.conf` (prefix match!)

**Impact**: If the overlay/keymap references nodes that don't exist on all boards (e.g., `&pro_micro_i2c`, `&led_strip`, `&nice_view_spi`), builds for boards without those nodes will fail.

**Solution**: Name the dongle shield so it does NOT start with the same prefix as the keyboard shields. For example, use `corne_dongle` instead of `splitkb_aurora_corne_dongle_xiao`.

### 2. User Board Overlays Don't Exist

**Problem**: `config/boards/<board>.overlay` is NOT a valid location for user configuration!

**What ZMK Actually Looks For**:
- `config/<shield>.overlay` - Applied based on shield name (with prefix matching)
- `config/<shield>.keymap` - Applied based on shield name (with prefix matching)
- `config/<shield>.conf` - Applied based on shield name (with prefix matching)

**What ZMK Does NOT Look For**:
- `config/boards/<board>.overlay` - This is IGNORED!

**Solution**: All user overlay configuration must go in shield-named files.

### 3. Dongle Keymap Must Have Full Key Bindings

**Problem**: In ZMK split architecture, the central (dongle) applies the keymap to translate key positions to keycodes. If the dongle keymap has `&none` for all keys, no keys will register!

**How It Works**:
1. Peripherals (keyboard halves) send raw key **positions** to the central
2. Central (dongle) applies its **keymap** to translate positions to keycodes
3. Central sends keycodes to the host computer

**Impact**: A dongle with all `&none` bindings will connect but keys won't work.

**Solution**: Use a shared base keymap file:
- `splitkb_aurora_corne_base.dtsi` - All behaviors, macros, combos, and keymap bindings
- `splitkb_aurora_corne.keymap` - Includes base + device refs (`&led_strip`, `&nice_view_spi`)
- `corne_dongle.keymap` - Includes just the base (no device refs)

### 4. BLE Device Name Max Length

**Problem**: `CONFIG_ZMK_KEYBOARD_NAME` has a maximum length of 16 characters.

**Error**: `static assertion failed: "ERROR: BLE device name is too long. Max length: 16"`

**Example**:
- "Aurora Corne Dongle" (19 chars) - TOO LONG
- "Corne Dongle" (12 chars) - OK

### 5. Dongle Central Peripheral Count

**Problem**: Default dongle only accepts 1 peripheral connection.

**Solution**: Set in dongle's Kconfig.defconfig:
```kconfig
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 2

config BT_MAX_CONN
    default 7  # 2 peripherals + 5 BT profiles

config BT_MAX_PAIRED
    default 7
```

### 6. Nodes That Don't Exist on All Boards

These nodes are board/shield specific and will cause build failures if referenced for the wrong board:

| Node | Exists On | Does NOT Exist On |
|------|-----------|-------------------|
| `&pro_micro_i2c` | nice_nano (Pro Micro pinout) | xiao_ble |
| `&led_strip` | Aurora Corne shield | Dongle (no LEDs) |
| `&nice_view_spi` | nice_view shield | Dongle (no display) |
| `&pro_micro` | nice_nano | xiao_ble |

### 7. Dongle Requirements

A ZMK dongle needs:
1. **Matrix transform** - Must match the keyboard layout even though dongle has no keys
2. **Mock kscan** - Use `zmk,kscan-mock` with 0 columns/rows/events
3. **Full keymap** - Same key bindings as keyboard halves (central processes the keymap!)
4. **BLE central config** - `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`
5. **Peripheral count** - `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2` for split keyboard
6. **Physical layouts in same order as peripherals** - See critical quirk #8 below!

### 8. Physical Layout Index Synchronization (CRITICAL)

**Problem**: Dongle mode causes -3 (or other) key position offset compared to standalone mode.

**Root Cause**: ZMK's split protocol sends physical layout **indices** (array positions) from central to peripherals to synchronize layouts. Layout indices are determined by **include order** in devicetree, NOT by layout name.

**How Layout Indices Are Assigned**:
```c
// In ZMK source: app/src/physical_layouts.c
static const struct zmk_physical_layout *const layouts[] = {
    DT_FOREACH_STATUS_OKAY(zmk_physical_layout, PHYS_LAYOUT_REF)
};
```
The `DT_FOREACH_STATUS_OKAY` macro iterates layouts in devicetree include order.

**Aurora Corne Layout Indices** (from `splitkb_aurora_corne.dtsi`):
```c
#include <layouts/foostan/corne/5column.dtsi>  // → index 0
#include <layouts/foostan/corne/6column.dtsi>  // → index 1
```

**The Bug**: If the dongle only includes `6column.dtsi`:
- Dongle: `foostan_corne_6col_layout` = index **0** (only layout)
- Peripheral: `foostan_corne_6col_layout` = index **1**
- Dongle sends "select layout 0" → Peripheral selects `layouts[0]` = **5-column**!
- 5-column transform causes -3 position offset

**The Fix**: Dongle MUST include layouts in the SAME ORDER as the keyboard shields:
```c
/* In corne_dongle.overlay */
#include <layouts/foostan/corne/5column.dtsi>  // → index 0
#include <layouts/foostan/corne/6column.dtsi>  // → index 1

/ {
    chosen {
        zmk,physical-layout = &foostan_corne_6col_layout;  // selects index 1
    };
};

&foostan_corne_5col_layout {
    transform = <&five_column_transform>;
};

&foostan_corne_6col_layout {
    transform = <&default_transform>;
};
```

**Verification**: After flashing new dongle firmware, do a **settings reset on all devices** to clear any saved layout state.

**ZMK Source Reference** (central.c):
```c
static void update_peripherals_selected_physical_layout(struct k_work *_work) {
    uint8_t layout_idx = zmk_physical_layouts_get_selected();  // Returns array index
    for (int i = 0; i < ZMK_SPLIT_BLE_PERIPHERAL_COUNT; i++) {
        update_peripheral_selected_layout(&peripherals[i], layout_idx);
    }
}
```

## Current Architecture

### File Structure
```
config/
├── boards/shields/corne_dongle/
│   ├── Kconfig.shield             # Dongle shield declaration
│   ├── Kconfig.defconfig          # Dongle BLE central config
│   └── corne_dongle.overlay       # Dongle matrix/kscan
├── splitkb_aurora_corne.overlay   # nice_nano-specific hardware (SPI0, I2C)
├── splitkb_aurora_corne.keymap    # Keyboard halves keymap (includes base + device refs)
├── splitkb_aurora_corne.conf      # Keyboard halves config
├── splitkb_aurora_corne_base.dtsi # SHARED keymap (behaviors, macros, keymap bindings)
├── corne_dongle.keymap            # Dongle keymap (includes base only)
└── corne_dongle.conf              # Dongle config
```

### Keymap Structure (Single Source of Truth)
```
splitkb_aurora_corne_base.dtsi     <- Edit this file for keymap changes
        ↑                ↑
        |                |
splitkb_aurora_corne.keymap    corne_dongle.keymap
(adds &led_strip,              (no device refs)
 &nice_view_spi)
```

### Build Targets
- `nice_nano@2.0.0` + `splitkb_aurora_corne_left` - Left keyboard half (standalone central)
- `nice_nano@2.0.0` + `splitkb_aurora_corne_right` - Right keyboard half
- `nice_nano@2.0.0` + `splitkb_aurora_corne_left` + cmake-args - Left peripheral (for dongle mode)
- `xiao_ble` + `corne_dongle` - Wireless dongle (central)
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
| `undefined node label 'pro_micro_i2c'` | Overlay applied to xiao_ble | Use shield-named overlay, rename dongle to avoid prefix match |
| `undefined node label 'led_strip'` | Keymap applied to dongle | Use shared base keymap without device refs |
| `undefined node label 'nice_view_spi'` | Keymap applied to dongle | Use shared base keymap without device refs |
| `Need a matrix transform` | Dongle missing transform | Add matrix transform to dongle overlay |
| Keys not registering | Dongle keymap has `&none` | Dongle needs full keymap (central processes it!) |
| Only one half connects | `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` not set | Set to 2 in dongle Kconfig.defconfig |
| -3 position offset in dongle mode | Physical layout index mismatch | Include layouts in SAME ORDER as keyboard shield (see quirk #8) |
| Wrong keys register in dongle mode | Peripheral using wrong transform | Check layout include order matches between dongle and peripherals |
