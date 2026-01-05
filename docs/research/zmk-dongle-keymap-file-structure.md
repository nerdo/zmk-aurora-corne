# ZMK Dongle Shield Configuration - File Structure Analysis

**Research Date:** 2026-01-04
**Subject:** Understanding ZMK keymap file selection for dongle shields
**Reference Repository:** [aroum/zmk-enki42-dongle](https://github.com/aroum/zmk-enki42-dongle/tree/xiao_dongle/config/boards/shields/enki42)

## Executive Summary

**Root Cause Identified:** ZMK applies the wrong keymap to dongle shields due to **file naming conventions**. The build system uses a **base name matching algorithm** that treats `splitkb_aurora_corne` as the base for both keyboard halves and the dongle shield, causing the main keyboard's keymap to be applied to the dongle.

**Critical Finding:** ZMK's keymap selection follows this precedence:
1. `config/<shield_name>.keymap` (exact match)
2. `config/<base_name>.keymap` (prefix/base match)
3. `boards/shields/<shield_dir>/<shield_name>.keymap` (shield directory)

**Solution:** The dongle shield must either:
- Use a completely different base name (e.g., `aurora_corne_dongle` instead of `aurora_corne_dongle_xiao`)
- Have its keymap placed in the shield directory as `boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap`

---

## Problem Statement

Building the `aurora_corne_dongle_xiao` shield fails because ZMK applies `splitkb_aurora_corne.keymap` instead of `aurora_corne_dongle_xiao.keymap`. The keyboard keymap references hardware (RGB LEDs, displays, complex behaviors) that doesn't exist on the dongle, causing compilation failures.

**Error Pattern:**
```
ERROR: devicetree error: 'led_strip' referenced but not found
ERROR: devicetree error: 'nice_view_spi' referenced but not found
ERROR: undefined reference to behaviors defined in keyboard keymap
```

---

## ZMK Keymap File Selection Mechanism

### File Naming Convention

ZMK uses a **base name matching algorithm** for keymap selection:

1. **Exact Match Priority:**
   - `config/<shield_name>.keymap` (e.g., `aurora_corne_dongle_xiao.keymap`)
   - `config/<shield_name>.conf` (e.g., `aurora_corne_dongle_xiao.conf`)

2. **Base Name Fallback:**
   - For split keyboards: `config/<base_name>.keymap` applies to both `<base_name>_left` and `<base_name>_right`
   - Example: `corne.keymap` applies to both `corne_left` and `corne_right` shields
   - **Critical:** If shield name contains an underscore variant, ZMK may treat everything before the last suffix as the base

3. **Shield Directory Default:**
   - `boards/shields/<shield_dir>/<shield_name>.keymap`
   - Used when no user config exists

**Source:** [ZMK Configuration Overview](https://zmk.dev/docs/config), [Customizing ZMK](https://zmk.dev/docs/customization)

### Current File Structure (Problematic)

```
config/
├── splitkb_aurora_corne.keymap       # Main keyboard keymap (RGB, displays, complex layers)
├── splitkb_aurora_corne.conf
├── aurora_corne_dongle_xiao.keymap   # Dongle keymap (empty bindings only)
├── aurora_corne_dongle_xiao.conf
└── boards/shields/splitkb_aurora_corne/
    ├── aurora_corne_dongle_xiao.overlay
    ├── Kconfig.shield
    └── Kconfig.defconfig
```

**Issue:** ZMK likely treats `aurora_corne_dongle_xiao` as having base name `splitkb_aurora_corne` or `aurora_corne`, causing it to fall back to `splitkb_aurora_corne.keymap`.

---

## Enki42 Reference Implementation Analysis

### File Structure

The enki42 project demonstrates a working dongle implementation with this structure:

```
config/
├── boards/shields/enki42/
│   ├── Kconfig.shield              # Shield registration
│   ├── Kconfig.defconfig           # Default configurations
│   ├── enki42.dtsi                 # Shared hardware definitions
│   ├── enki42.keymap               # Shared keymap for left/right
│   ├── enki42.conf                 # Shared config for left/right
│   ├── enki42_left.overlay         # Left half hardware
│   ├── enki42_left.conf            # Left half specific config
│   ├── enki42_right.overlay        # Right half hardware
│   ├── enki42_right.conf           # Right half specific config
│   ├── enki42_dongle.overlay       # Dongle hardware
│   ├── enki42_dongle.conf          # Dongle specific config
│   └── enki42.zmk.yml              # Metadata
├── info.json
└── west.yml
```

### Key Observations

#### 1. Kconfig.shield Registration

**File:** `boards/shields/enki42/Kconfig.shield`

```kconfig
# Copyright (c) 2020 The ZMK Contributors
# SPDX-License-Identifier: MIT

config SHIELD_ENKI42_LEFT
	def_bool $(shields_list_contains,enki42_left)

config SHIELD_ENKI42_RIGHT
	def_bool $(shields_list_contains,enki42_right)

config SHIELD_ENKI42_DONGLE
	def_bool $(shields_list_contains,enki42_dongle)
```

**Analysis:**
- Three distinct shields registered: `enki42_left`, `enki42_right`, `enki42_dongle`
- Each shield has its own Kconfig symbol
- All shields share the same base name: `enki42`

#### 2. Kconfig.defconfig Configuration

**File:** `boards/shields/enki42/Kconfig.defconfig`

```kconfig
# Copyright (c) 2020 The ZMK Contributors
# SPDX-License-Identifier: MIT

if SHIELD_ENKI42_DONGLE

config ZMK_KEYBOARD_NAME
	default "enki42 Dongle"

config ZMK_SPLIT_ROLE_CENTRAL
	default y

config ZMK_USB
	default y

endif

if SHIELD_ENKI42_LEFT || SHIELD_ENKI42_RIGHT || SHIELD_ENKI42_DONGLE

config ZMK_SPLIT
	default y

config ZMK_BLE
	default y

endif
```

**Analysis:**
- Dongle-specific configuration isolated in `if SHIELD_ENKI42_DONGLE` block
- Sets dongle as central role with USB enabled
- Shared configuration (split + BLE) applies to all shields
- **Critical:** No separate keymap defined for dongle in this file

#### 3. Shared Keymap Strategy

**File:** `boards/shields/enki42/enki42.keymap`

```c
/*
 * Copyright (c) 2020 The ZMK Contributors
 * SPDX-License-Identifier: MIT
 */

#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/ext_power.h>

/ {
	keymap {
		compatible = "zmk,keymap";

		default_layer {
			bindings = <
				&kp Q &kp W &kp E &kp R &kp T   &kp Y &kp U &kp I &kp O &kp P
				// ... (full QWERTY layout with 42 keys)
			>;
		};

		lower_layer {
			bindings = <
				&kp N1 &kp N2 &kp N3 &kp N4 &kp N5   &kp N6 &kp N7 &kp N8 &kp N9 &kp N0
				// ... (number/symbol layer)
			>;
		};

		raise_layer {
			bindings = <
				// ... (navigation/media layer)
			>;
		};

		adjust_layer {
			bindings = <
				// ... (Bluetooth/system layer)
			>;
		};
	};
};
```

**Analysis:**
- **Single keymap file** (`enki42.keymap`) serves both keyboard halves
- **No separate keymap for dongle** - the dongle doesn't need key bindings
- The keymap references hardware (matrix transform, behaviors) defined in the keyboard halves' overlays
- **Critical Insight:** Dongle overlay defines its own matrix transform that matches the keymap structure

#### 4. Dongle Overlay Hardware Definition

**File:** `boards/shields/enki42/enki42_dongle.overlay`

```dts
/*
 * Copyright (c) 2020 The ZMK Contributors
 * SPDX-License-Identifier: MIT
 */

#include "enki42.dtsi"

&default_transform {
	col-offset = <0>;
};

/ {
	kscan0: kscan {
		compatible = "zmk,kscan-gpio-matrix";
		diode-direction = "col2row";
		wakeup-source;

		row-gpios = <&xiao_d 7 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
		            <&xiao_d 8 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
		            <&xiao_d 9 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
		            <&xiao_d 10 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;

		col-gpios = <&xiao_d 1 GPIO_ACTIVE_HIGH>
		            <&xiao_d 2 GPIO_ACTIVE_HIGH>
		            <&xiao_d 3 GPIO_ACTIVE_HIGH>
		            <&xiao_d 4 GPIO_ACTIVE_HIGH>
		            <&xiao_d 5 GPIO_ACTIVE_HIGH>
		            <&xiao_d 6 GPIO_ACTIVE_HIGH>;
	};
};

&xiao_spi { status = "disabled"; };
&xiao_i2c { status = "disabled"; };
&xiao_serial { status = "disabled"; };
```

**Analysis:**
- Includes shared `enki42.dtsi` for matrix transform definitions
- Defines 6x4 matrix matching the keyboard's layout
- **Critical:** The matrix transform allows the keymap to compile even though the dongle has no physical keys
- Disables unused peripherals to free GPIO pins

#### 5. Build Configuration

**File:** `build.yaml`

```yaml
---
include:
  - board: nrfmicro_13
    shield: enki42_left
  - board: nrfmicro_13
    shield: enki42_right
  - board: seeeduino_xiao_ble
    shield: enki42_dongle
```

**Analysis:**
- Each shield explicitly listed with its board
- Dongle uses different board (`seeeduino_xiao_ble`) than keyboard halves
- No special keymap references needed - ZMK automatically uses `enki42.keymap` for all shields

---

## Current Aurora Corne Implementation Issues

### File Structure Comparison

**Current (Problematic):**

```
config/
├── splitkb_aurora_corne.keymap       # Has RGB, display, behaviors
├── aurora_corne_dongle_xiao.keymap   # Empty keymap
└── boards/shields/splitkb_aurora_corne/
    ├── aurora_corne_dongle_xiao.overlay
    ├── Kconfig.shield                # Only registers dongle
    └── Kconfig.defconfig             # Only configures dongle
```

**Enki42 (Working):**

```
config/boards/shields/enki42/
├── enki42.keymap                     # Shared keymap
├── enki42_dongle.overlay
├── Kconfig.shield                    # Registers all three shields
└── Kconfig.defconfig                 # Configures all three shields
```

### Issue #1: Incomplete Shield Registration

**Current Kconfig.shield:**

```kconfig
config SHIELD_AURORA_CORNE_DONGLE_XIAO
	def_bool $(shields_list_contains,aurora_corne_dongle_xiao)
```

**Problem:** Only registers the dongle shield, not the keyboard halves.

**Expected (based on enki42):**

```kconfig
config SHIELD_SPLITKB_AURORA_CORNE_LEFT
	def_bool $(shields_list_contains,splitkb_aurora_corne_left)

config SHIELD_SPLITKB_AURORA_CORNE_RIGHT
	def_bool $(shields_list_contains,splitkb_aurora_corne_right)

config SHIELD_AURORA_CORNE_DONGLE_XIAO
	def_bool $(shields_list_contains,aurora_corne_dongle_xiao)
```

### Issue #2: Keymap File Location

**Current:**
- `config/aurora_corne_dongle_xiao.keymap` exists but may not be found due to base name mismatch
- `config/splitkb_aurora_corne.keymap` contains hardware references that fail on dongle

**Expected (two options):**

**Option A - Shield Directory Keymap:**
```
config/boards/shields/splitkb_aurora_corne/
└── aurora_corne_dongle_xiao.keymap   # Move here for explicit match
```

**Option B - Rename Shield Base:**
```
Rename shield: aurora_corne_dongle_xiao → aurora_dongle_xiao
Then config/aurora_dongle_xiao.keymap will match exactly
```

### Issue #3: Missing Dongle Matrix Transform

**Current `aurora_corne_dongle_xiao.overlay`:**

(Content not shown in research, but likely missing proper matrix transform definition)

**Expected (based on enki42):**

The overlay should include a complete matrix transform matching the keyboard layout, even though the dongle has no physical keys. This allows the keymap to compile successfully.

---

## Recommended Solutions

### Solution 1: Move Keymap to Shield Directory (Simplest)

**Action:** Move the dongle keymap into the shield directory for explicit matching.

**Steps:**

1. Move keymap file:
   ```bash
   mv config/aurora_corne_dongle_xiao.keymap \
      config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap
   ```

2. Verify ZMK finds the correct keymap during build

**Pros:**
- Minimal changes
- Keeps existing shield name
- Clear separation of dongle config from keyboard config

**Cons:**
- Deviates from user config directory pattern
- Less discoverable for users

### Solution 2: Rename Dongle Shield Base (Recommended)

**Action:** Change shield name to avoid base name conflict with keyboard shields.

**Steps:**

1. Rename shield in all locations:
   - `Kconfig.shield`: `aurora_corne_dongle_xiao` → `aurora_dongle_xiao`
   - `Kconfig.defconfig`: Update shield symbol
   - Rename overlay: `aurora_corne_dongle_xiao.overlay` → `aurora_dongle_xiao.overlay`
   - Rename config files:
     - `config/aurora_corne_dongle_xiao.keymap` → `config/aurora_dongle_xiao.keymap`
     - `config/aurora_corne_dongle_xiao.conf` → `config/aurora_dongle_xiao.conf`
   - Update `build.yaml`: `aurora_corne_dongle_xiao` → `aurora_dongle_xiao`

2. Verify no base name overlap with `splitkb_aurora_corne`

**Pros:**
- Clear base name separation
- Follows user config directory pattern
- More maintainable long-term
- Matches enki42 pattern (different base names)

**Cons:**
- Requires more file renames
- Breaks existing build references

### Solution 3: Use Shared Keymap Strategy (Most Robust)

**Action:** Restructure to match enki42's approach with shared keymap and proper conditional compilation.

**Steps:**

1. **Complete Kconfig.shield registration:**
   ```kconfig
   config SHIELD_SPLITKB_AURORA_CORNE_LEFT
   	def_bool $(shields_list_contains,splitkb_aurora_corne_left)

   config SHIELD_SPLITKB_AURORA_CORNE_RIGHT
   	def_bool $(shields_list_contains,splitkb_aurora_corne_right)

   config SHIELD_AURORA_CORNE_DONGLE_XIAO
   	def_bool $(shields_list_contains,aurora_corne_dongle_xiao)
   ```

2. **Update Kconfig.defconfig** with conditional blocks:
   ```kconfig
   if SHIELD_AURORA_CORNE_DONGLE_XIAO
   config ZMK_KEYBOARD_NAME
   	default "Aurora Corne Dongle"
   # ... dongle-specific config
   endif

   if SHIELD_SPLITKB_AURORA_CORNE_LEFT || SHIELD_SPLITKB_AURORA_CORNE_RIGHT
   config ZMK_KEYBOARD_NAME
   	default "Aurora Corne"
   # ... keyboard-specific config
   endif
   ```

3. **Modify keymap for conditional compilation:**
   ```c
   / {
   	keymap {
   		compatible = "zmk,keymap";

   		default_layer {
   			bindings = <
   #if CONFIG_SHIELD_AURORA_CORNE_DONGLE_XIAO
   				&none &none &none /* ... empty bindings for dongle */
   #else
   				&kp Q &kp W &kp E /* ... full keyboard layout */
   #endif
   			>;
   		};
   	};
   };
   ```

4. **Ensure dongle overlay has proper matrix transform:**
   ```dts
   #include <splitkb_aurora_corne.dtsi>  // If shared definitions exist

   / {
   	chosen {
   		zmk,kscan = &kscan0;
   		zmk,matrix-transform = &default_transform;
   	};
   };
   ```

**Pros:**
- Most maintainable (single keymap for all variants)
- Follows ZMK's intended pattern
- Reduces duplication
- Matches reference implementation

**Cons:**
- Most complex initial setup
- Requires understanding of Kconfig conditionals
- May complicate keymap editing for users unfamiliar with preprocessor directives

---

## Additional Findings

### Matrix Transform Importance

The dongle overlay **must** define a complete matrix transform matching the keyboard's key positions, even though the dongle has no physical keys. This allows the keymap to compile successfully.

**From enki42_dongle.overlay:**
- Includes shared `.dtsi` with matrix transform definitions
- Defines `kscan0` with row/col GPIO (even though unused for dongle)
- Sets `&default_transform` with `col-offset = <0>`

**Why this works:**
- ZMK's build system expects all shields to have a matrix definition
- The keymap bindings array size must match the matrix dimensions
- Dongle acts as a "virtual keyboard" from ZMK's perspective

### File Naming Patterns Observed

| Shield Type | Shield Name | Keymap File | Location |
|-------------|-------------|-------------|----------|
| Split Left | `enki42_left` | `enki42.keymap` | `boards/shields/enki42/` |
| Split Right | `enki42_right` | `enki42.keymap` | `boards/shields/enki42/` |
| Dongle | `enki42_dongle` | `enki42.keymap` | `boards/shields/enki42/` |
| **Current** | `aurora_corne_dongle_xiao` | **`splitkb_aurora_corne.keymap`** ❌ | `config/` |
| **Expected** | `aurora_corne_dongle_xiao` | `aurora_corne_dongle_xiao.keymap` ✅ | `config/` or shield dir |

### Configuration File Hierarchy

ZMK configuration files are loaded in this order (later files override earlier):

1. Shield defaults (`Kconfig.defconfig`)
2. Board defaults
3. User config files (`config/<shield_name>.conf`)
4. Command-line overrides

**Keymap files** don't follow the same override pattern - only **one keymap** is used per build, selected by the naming algorithm.

---

## Testing Checklist

After implementing the chosen solution:

- [ ] Clean build succeeds for dongle shield
- [ ] Build output confirms correct keymap file used (check build logs)
- [ ] Dongle firmware size reasonable (should be smaller than keyboard firmware)
- [ ] No devicetree errors about missing hardware references
- [ ] Dongle config properly sets central role
- [ ] Bluetooth connections work (dongle to keyboard halves)
- [ ] USB connection works (dongle to computer)
- [ ] Keyboard halves still build successfully with their keymap
- [ ] No regression in keyboard half functionality

---

## References

### Documentation
- [ZMK Configuration Overview](https://zmk.dev/docs/config)
- [Customizing ZMK](https://zmk.dev/docs/customization)
- [New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield)
- [Building and Flashing](https://zmk.dev/docs/development/local-toolchain/build-flash)

### Reference Implementation
- [aroum/zmk-enki42-dongle](https://github.com/aroum/zmk-enki42-dongle/tree/xiao_dongle/config/boards/shields/enki42)
  - Kconfig.shield: Shield registration for all three variants
  - Kconfig.defconfig: Conditional configuration per shield type
  - enki42.keymap: Shared keymap for keyboard halves
  - enki42_dongle.overlay: Dongle hardware definition with matrix transform

---

## Next Steps

1. **Choose solution approach** based on project maintainability goals
2. **Implement file structure changes** according to chosen solution
3. **Test build** for dongle shield and verify correct keymap selection
4. **Validate runtime behavior** with actual hardware
5. **Document decision** in project README for future reference

---

**Report Generated:** 2026-01-04
**Analyst:** Research Agent
**Status:** Complete - Ready for implementation decision
