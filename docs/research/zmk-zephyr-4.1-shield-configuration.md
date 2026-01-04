# ZMK Zephyr 4.1 Shield Configuration Research

**Research Date:** 2026-01-04
**Focus:** Understanding Zephyr 4.1 changes for ZMK custom shields and user config repositories
**Status:** Complete

## Executive Summary

ZMK's upgrade to Zephyr 4.1 (December 2025) introduces **Hardware Model V2 (HWMv2)** and significant structural changes affecting custom shields and user config repositories. The key finding: **`config/boards/` is the correct location for user config repositories** - contrary to initial assumptions about deprecation. Your recent commit (164afdc) moving shields from `boards/` to `config/boards/` was the correct fix.

### Critical Findings

1. **User config repos MUST use `config/boards/` structure** - This is NOT deprecated for user repositories
2. **Board extensions** (`boards/extensions/`) are for extending UPSTREAM boards, not for custom shields
3. **Modules** are for shareable hardware definitions, not single-keyboard user configs
4. **LED strip errors** require board-specific overlays in `config/boards/shields/<shield>/boards/<board>.overlay`
5. **CONFIG_WS2812_STRIP removed** - Already fixed in commit e1d64b6

---

## 1. Zephyr 4.1 Breaking Changes for ZMK

### 1.1 Hardware Model V2 (HWMv2) Migration

**Impact:** ALL out-of-tree keyboards must be updated to use HWMv2.

**Key Changes:**
- Board naming now uses revision suffixes: `nice_nano@2.0.0` instead of `nice_nano_v2`
- HWMv1 board name aliases removed (deprecated in Zephyr 3.7)
- Better support for multi-core SoCs (e.g., nRF5340)

**Evidence from build.yaml:**
```yaml
- board: nice_nano@2.0.0  # ✅ Already using HWMv2 syntax
  shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
```

**Source:** [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1)

### 1.2 LED Strip Configuration Changes

**Breaking Change:** `CONFIG_WS2812_STRIP` removed from Kconfig.

**Migration:** Remove from all `.conf` files.

**Status in your repo:** ✅ Already fixed in commit e1d64b6
```conf
# /Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne.conf
# Line 3 is commented out - correct for Zephyr 4.1
# CONFIG_WS2812_STRIP=y
```

**Source:** [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1)

### 1.3 nRF52840 Configuration Migrations

**NFC Pins:** Moved from Kconfig to devicetree
```dts
// Old (deprecated):
// CONFIG_NFCT_PINS_AS_GPIOS=y

// New (required):
&uicr {
    nfct-pins-as-gpios;
};
```

**DC/DC Configuration:** Moved from Kconfig to devicetree - configure `&reg0` or `&reg1` nodes.

**Source:** [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1)

### 1.4 Devicetree Syntax Changes

**No major syntax changes identified** in Zephyr 4.1 for basic shield overlays. The devicetree language remains stable. Changes are primarily in:
- Hardware-specific configurations (NFC, DC/DC, bootloader)
- LED driver properties (removed Kconfig flags)
- Board naming conventions (HWMv2)

---

## 2. Directory Structure: config/boards vs boards/extensions vs Modules

### 2.1 User Config Repository Structure (Your Use Case)

**CORRECT STRUCTURE for user config repos:**
```
zmk-config/
├── config/
│   ├── boards/                      # ✅ REQUIRED for custom shields
│   │   └── shields/
│   │       └── splitkb_aurora_corne/
│   │           ├── Kconfig.shield
│   │           ├── Kconfig.defconfig
│   │           ├── *.overlay
│   ├── *.conf                       # Shield configurations
│   ├── *.keymap                     # Keymaps
│   ├── splitkb_aurora_corne.overlay # Main board overlay (optional)
│   └── west.yml
├── build.yaml
```

**Evidence from your commit 164afdc:**
```
fix(zmk): move custom shields to correct config directory

- Move shields from boards/ to config/boards/ (required for user config repos)
- Fixes shield discovery errors in GitHub Actions build.
```

**Why this works:**
- ZMK searches `zmk-config/config` directory for user customizations
- The `west.yml` `self.path: config` setting makes `config/` the board root
- Custom shields in `config/boards/shields/` are discovered during build

**Source:** [ZMK Configuration Overview](https://zmk.dev/docs/config)

### 2.2 Board Extensions (NOT for User Shields)

**Purpose:** Extend UPSTREAM Zephyr/ZMK boards without modifying core definitions.

**Structure:**
```
<board_root>/
└── boards/
    └── extensions/
        └── <board_dir_name>/    # e.g., seeeduino_xiao_ble
            └── board.cmake       # Extension files
```

**When to use:**
- Extending boards from upstream Zephyr that ZMK hasn't configured
- Adding default settings (like `CONFIG_ZMK_USB`) to existing boards
- **NOT for custom shields or keyboard definitions**

**Your Use Case:** ❌ NOT APPLICABLE - You're creating custom shields, not extending upstream boards.

**Source:** [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1)

### 2.3 Modules (NOT for Single-Keyboard Configs)

**Purpose:** Shareable, reusable keyboard/component definitions.

**Naming:** `zmk-<type>-<description>` (e.g., `zmk-keyboard-corne-dongle`)

**When to use:**
- Open-source hardware designs for public sharing
- Reusable components (behaviors, drivers, visual effects)
- Version-controlled keyboard definitions across multiple projects

**Structure:**
```
zmk-keyboard-myboard/
├── zephyr/
│   └── module.yml          # Module definition
├── boards/
│   └── shields/
│       └── myboard/
├── dts/                    # Devicetree bindings
├── src/                    # Source code
├── CMakeLists.txt
└── Kconfig
```

**Your Use Case:** ❌ NOT APPLICABLE - You have a single-keyboard user config repo.

**Recommendation:** Stick with user config structure unless you plan to:
- Share your dongle/peripheral shield configuration publicly
- Use the same shield across multiple projects
- Maintain versioned releases

**Source:** [ZMK Module Creation](https://zmk.dev/docs/development/module-creation)

---

## 3. Fixing "undefined node label 'led_strip'" Error

### 3.1 Root Cause

**Problem:** RGB underglow support must be added at the BOARD level, not shield level.

**Why:** LED strip drivers rely on hardware-specific interfaces (SPI, I2S) that shields don't control.

**Source:** [ZMK RGB Underglow Documentation](https://zmk.dev/docs/development/hardware-integration/lighting/underglow)

### 3.2 Solution: Board-Specific Shield Overlays

**Structure:**
```
config/boards/shields/splitkb_aurora_corne/
├── boards/                    # Board-specific overlays
│   ├── nice_nano.overlay      # LED config for nice_nano
│   └── xiao_ble.overlay       # LED config for XIAO BLE
├── Kconfig.shield
├── Kconfig.defconfig
└── *.overlay                   # Main shield overlays
```

**Example for nice_nano (nRF52840 SPI-based):**

File: `config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay`

```dts
#include <dt-bindings/led/led.h>

&pinctrl {
    spi3_default: spi3_default {
        group1 {
            psels = <NRF_PSEL(SPIM_MOSI, 0, 6)>;  // LED data pin
        };
    };

    spi3_sleep: spi3_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_MOSI, 0, 6)>;
            low-power-enable;
        };
    };
};

&spi3 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    pinctrl-0 = <&spi3_default>;
    pinctrl-1 = <&spi3_sleep>;
    pinctrl-names = "default", "sleep";

    led_strip: ws2812@0 {
        compatible = "worldsemi,ws2812-spi";
        label = "WS2812";

        /* SPI */
        reg = <0>;
        spi-max-frequency = <4000000>;

        /* WS2812 */
        chain-length = <27>;              // Number of LEDs (adjust for Corne)
        spi-one-frame = <0x70>;
        spi-zero-frame = <0x40>;

        color-mapping = <LED_COLOR_ID_GREEN
                         LED_COLOR_ID_RED
                         LED_COLOR_ID_BLUE>;
    };
};

/ {
    chosen {
        zmk,underglow = &led_strip;
    };
};
```

**Critical Properties:**

| Property | Description | Typical Value |
|----------|-------------|---------------|
| `compatible` | LED driver type | `"worldsemi,ws2812-spi"` |
| `chain-length` | Number of LEDs | Varies (27 for full Corne underglow) |
| `spi-max-frequency` | SPI clock speed | `4000000` (4 MHz) |
| `color-mapping` | LED color order | Green, Red, Blue (WS2812 standard) |
| `spi-one-frame` | WS2812 "1" encoding | `0x70` |
| `spi-zero-frame` | WS2812 "0" encoding | `0x40` |

**Integration:** Add to root devicetree's `chosen` node:
```dts
chosen {
    zmk,underglow = &led_strip;
};
```

**Source:** [ZMK RGB Underglow Documentation](https://zmk.dev/docs/development/hardware-integration/lighting/underglow)

### 3.3 Platform-Specific Notes

**nRF52-based boards (nice_nano):**
- ✅ Use `&spi3` for LED strips (simplest)
- ⚠️ Avoid low-frequency I/O pins (causes wireless interference)

**RP2040-based boards:**
- Requires PIO-based SPI implementation
- Needs pinctrl definitions for clock, MOSI, MISO

**Your dongle (XIAO BLE - nRF52840):**
- Can use same SPI3 approach as nice_nano
- Adjust GPIO pin numbers for XIAO pinout

---

## 4. Extending Existing Shields (Peripheral Variant)

### 4.1 Your Implementation (splitkb_aurora_corne_left_peripheral)

**Analysis of your current approach:**

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/Kconfig.shield`
```kconfig
config SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_left_peripheral)
    select SHIELD_SPLITKB_AURORA_CORNE_LEFT  # ✅ Inherits base left shield
```

**Evaluation:** ✅ **CORRECT APPROACH**

This `select` statement:
1. Enables the base `SHIELD_SPLITKB_AURORA_CORNE_LEFT` shield
2. Inherits all devicetree definitions from ZMK's upstream shield
3. Allows Kconfig overrides for peripheral role

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_left_peripheral.overlay`
```dts
/* Hardware definition inherited from splitkb_aurora_corne_left shield */
/* This file intentionally left minimal - peripheral role set via Kconfig */
```

**Evaluation:** ✅ **CORRECT - Minimal overlay for role-only variant**

The devicetree inheritance happens automatically via `select SHIELD_SPLITKB_AURORA_CORNE_LEFT`.

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/Kconfig.defconfig`
```kconfig
if SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL

config ZMK_SPLIT_ROLE_CENTRAL
    default n  # ✅ Peripheral mode

config BT_MAX_CONN
    default 1  # ✅ Only connects to dongle

config BT_MAX_PAIRED
    default 1  # ✅ Optimized for peripheral

endif
```

**Evaluation:** ✅ **CORRECT - Kconfig-only differentiation**

This is the proper way to create a peripheral variant when the hardware is identical but the role differs.

**Source:** [ZMK New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield)

### 4.2 Pattern for Shields with Hardware Differences

**When to add devicetree content to variant overlay:**

Use case: If your peripheral variant had different hardware (e.g., different encoders, LEDs, or matrix)

**Example structure:**
```dts
/* Inherit base hardware */
/* Hardware modifications for peripheral variant */

&kscan0 {
    /* Override matrix scanning settings */
    debounce-press-ms = <2>;
    debounce-release-ms = <2>;
};

/ {
    /* Add peripheral-specific hardware */
    peripheral_led: gpio_led {
        compatible = "gpio-leds";
        gpios = <&pro_micro 19 GPIO_ACTIVE_HIGH>;
    };
};
```

**Your case:** ❌ NOT NEEDED - Hardware is identical, only role differs (Kconfig-only).

---

## 5. Devicetree Overlay Syntax Requirements (Zephyr 4.1)

### 5.1 Core Syntax (Unchanged in 4.1)

**Node references:** Use ampersand (`&`) to modify existing nodes:
```dts
&existing_node {
    property = <value>;
};
```

**Creating new nodes:**
```dts
/ {
    new_node: label {
        compatible = "vendor,device";
        property = <value>;
    };
};
```

**Including files:**
```dts
#include <dt-bindings/zmk/matrix_transform.h>
#include <dt-bindings/led/led.h>
```

### 5.2 Common Patterns for Shields

**Matrix transform:**
```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) ...
        >;
    };
};
```

**Kscan (key scanning):**
```dts
/ {
    kscan0: kscan {
        compatible = "zmk,kscan-gpio-matrix";
        label = "KSCAN";
        diode-direction = "col2row";

        row-gpios = <&pro_micro 21 GPIO_ACTIVE_HIGH>;
        col-gpios = <&pro_micro 15 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;
    };
};
```

**Physical layout:**
```dts
/ {
    default_layout: keymap_layout_0 {
        compatible = "zmk,physical-layout";
        transform = <&default_transform>;
        kscan = <&kscan0>;
    };

    chosen {
        zmk,physical-layout = &default_layout;
    };
};
```

**Source:** [ZMK New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield)

### 5.3 Split Keyboard Composite Kscan

**Your dongle implementation:**

File: `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.overlay`

```dts
/ {
    chosen {
        zmk,kscan = &kscan_comp;         // ✅ Use composite kscan
        zmk,matrix_transform = &default_transform;
    };

    /* This is required for split communication */
    kscan_comp: kscan_composite {
        compatible = "zmk,kscan-composite";
        label = "KSCAN_COMP";
        rows = <4>;
        columns = <12>;                   // Combined columns (6L + 6R)
    };

    default_transform: matrix_transform {
        compatible = "zmk,matrix-transform";
        rows = <4>;
        columns = <12>;
        map = <
            RC(0,0) RC(0,1) ... RC(0,11)  // Full keyboard mapping
            RC(1,0) RC(1,1) ... RC(1,11)
            RC(2,0) RC(2,1) ... RC(2,11)
            RC(3,3) RC(3,4) ... RC(3,8)   // Thumb keys
        >;
    };
};
```

**Evaluation:** ✅ **CORRECT - Dongle handles combined matrix from peripherals**

**Source:** [ZMK New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield)

### 5.4 NFCT Disable for XIAO BLE

**Your implementation:**
```dts
/* Disable NFC for Zephyr 4.1+ compatibility with XIAO BLE */
&nfct {
    status = "disabled";
};
```

**Alternative Zephyr 4.1 approach:**
```dts
&uicr {
    nfct-pins-as-gpios;  // Reclaim NFC pins as GPIO
};
```

**Recommendation:** Both work. Your approach disables NFC entirely. The `&uicr` approach reclaims pins as GPIO (more explicit about intent).

---

## 6. Actionable Recommendations

### 6.1 Immediate Actions (No Changes Needed)

✅ **Your current structure is CORRECT:**
- `config/boards/shields/` for custom shields ✅
- Shield inheritance via `select SHIELD_*` ✅
- Kconfig-only peripheral variant ✅
- HWMv2 board names (`nice_nano@2.0.0`) ✅
- `CONFIG_WS2812_STRIP` removed ✅

### 6.2 If You Want RGB Underglow

**Required:** Add board-specific overlays for LED configuration.

**Files to create:**

1. `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay`
   - Configure `&spi3` with LED strip on appropriate GPIO
   - Set `chain-length` to number of LEDs per half
   - Add `chosen { zmk,underglow = &led_strip; };`

2. `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/xiao_ble.overlay`
   - Similar SPI3 configuration for XIAO pinout
   - Adjust GPIO pin numbers for XIAO

**Enable in .conf files:**
```conf
CONFIG_ZMK_RGB_UNDERGLOW=y
# CONFIG_WS2812_STRIP=y  # ❌ REMOVE - deprecated in Zephyr 4.1
```

**Note:** You already have underglow enabled in `splitkb_aurora_corne.conf`:
```conf
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_ZMK_RGB_UNDERGLOW_ON_START=n
CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_USB=y
```

This will work once you add the board-specific overlays.

### 6.3 Optional: Improve NFCT Disable

**Current (works fine):**
```dts
&nfct {
    status = "disabled";
};
```

**Zephyr 4.1 recommended:**
```dts
&uicr {
    nfct-pins-as-gpios;
};
```

**Benefit:** More explicit about reclaiming NFC pins as GPIO (for XIAO's P0.09/P0.10).

### 6.4 Module Conversion (Optional - Only if Sharing)

**Convert to module IF:**
- You want to share this dongle configuration publicly
- You plan to use it across multiple projects
- You want version-controlled releases

**Steps:**
1. Create new repo: `zmk-keyboard-aurora-corne-dongle`
2. Add `zephyr/module.yml`
3. Move `config/boards/shields/` to `boards/shields/`
4. Reference as module in `config/west.yml`

**Recommendation:** ❌ NOT NEEDED for single-keyboard user config.

---

## 7. Research Gaps & Further Investigation

### 7.1 Unanswered Questions

1. **Board extension deprecation timeline:**
   - Web search found NO specific "config/boards deprecated" warning
   - Documentation clearly states `config/boards/` is for user customizations
   - Your commit 164afdc confirms this is the correct structure
   - **Conclusion:** No deprecation for user config repos - confusion likely from board extensions feature

2. **LED strip chain-length for Corne:**
   - Need to count actual LEDs on Aurora Corne PCB
   - Typical Corne: 27 LEDs per half (or 54 total)
   - Verify your specific PCB variant

3. **XIAO BLE GPIO pin mapping:**
   - Need to identify correct GPIO pin for LED data on XIAO
   - Consult XIAO BLE pinout diagram
   - Likely different from nice_nano's P0.06

### 7.2 Documentation Ambiguities

**ZMK docs lack clarity on:**
- User config vs module vs board extensions decision matrix
- When to use `config/boards/` vs `boards/` (now clear from research)
- Board-specific overlay discovery mechanism (underdocumented)

**Recommendation:** Trust empirical evidence (your working build) over ambiguous docs.

---

## 8. Evidence Collection

### 8.1 Your Repository State (Verified Working)

**Commit history evidence:**
```
164afdc - fix(zmk): move custom shields to correct config directory
e1d64b6 - fix: remove deprecated CONFIG_WS2812_STRIP for Zephyr 4.1 compatibility
```

**Current structure (verified):**
```
/Users/nerdo/personal/zmk-aurora-corne/
├── config/
│   ├── boards/
│   │   └── shields/
│   │       └── splitkb_aurora_corne/
│   │           ├── Kconfig.shield                                # ✅ Present
│   │           ├── Kconfig.defconfig                             # ✅ Present
│   │           ├── splitkb_aurora_corne_dongle_xiao.overlay      # ✅ Present
│   │           └── splitkb_aurora_corne_left_peripheral.overlay  # ✅ Present
│   ├── splitkb_aurora_corne.conf                                 # ✅ Present
│   ├── splitkb_aurora_corne.overlay                              # ✅ Present
│   ├── splitkb_aurora_corne_dongle_xiao.conf                     # ✅ Present
│   ├── splitkb_aurora_corne_dongle_xiao.keymap                   # ✅ Present
│   └── west.yml                                                  # ✅ Present
└── build.yaml                                                    # ✅ Present
```

**Build configuration (verified working):**
```yaml
# build.yaml - Uses HWMv2 board names
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
```

### 8.2 Code Snippets for Reference

**Shield inheritance pattern (Kconfig.shield):**
```kconfig
# Lines 7-10 from Kconfig.shield
config SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_left_peripheral)
    select SHIELD_SPLITKB_AURORA_CORNE_LEFT  # Inherits base shield
```

**Peripheral role configuration (Kconfig.defconfig):**
```kconfig
# Lines 34-52 from Kconfig.defconfig
if SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL

config ZMK_KEYBOARD_NAME
    default "Aurora Corne Left Peripheral"

config ZMK_SPLIT
    default y

config ZMK_SPLIT_ROLE_CENTRAL
    default n  # Peripheral mode

config BT_MAX_CONN
    default 1  # Only connects to dongle

config BT_MAX_PAIRED
    default 1  # Optimized for peripheral

endif
```

**Dongle composite kscan (splitkb_aurora_corne_dongle_xiao.overlay):**
```dts
# Lines 6-30
/ {
    chosen {
        zmk,kscan = &kscan_comp;
        zmk,matrix_transform = &default_transform;
    };

    kscan_comp: kscan_composite {
        compatible = "zmk,kscan-composite";
        label = "KSCAN_COMP";
        rows = <4>;
        columns = <12>;
    };

    default_transform: matrix_transform {
        compatible = "zmk,matrix-transform";
        rows = <4>;
        columns = <12>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10) RC(0,11)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10) RC(1,11)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10) RC(2,11)
                                    RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
        >;
    };
};
```

---

## 9. Sources

### Primary Sources (ZMK Official Documentation)

1. [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1) - Breaking changes, HWMv2, board extensions
2. [ZMK Configuration Overview](https://zmk.dev/docs/config) - User config repository structure
3. [ZMK New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield) - Shield creation, devicetree syntax
4. [ZMK Module Creation](https://zmk.dev/docs/development/module-creation) - Module vs user config guidance
5. [ZMK RGB Underglow](https://zmk.dev/docs/development/hardware-integration/lighting/underglow) - LED strip configuration

### Secondary Sources (Zephyr Project)

6. [Zephyr 4.1.0 Release Notes](https://docs.zephyrproject.org/latest/releases/release-notes-4.1.html) - HWMv2, board name aliases removal

### Repository Evidence

7. Your commit 164afdc - "fix(zmk): move custom shields to correct config directory"
8. Your commit e1d64b6 - "fix: remove deprecated CONFIG_WS2812_STRIP for Zephyr 4.1 compatibility"
9. ZMK board extensions examples - https://github.com/zmkfirmware/zmk/tree/main/app/boards/extensions

---

## 10. Conclusion

Your repository structure is **CORRECT for a user config repository**. The `config/boards/shields/` location is the proper approach for custom shields in user configs - this is NOT deprecated. Board extensions are a separate feature for extending upstream boards, not for custom shields.

**Key Takeaways:**

1. ✅ **Keep using `config/boards/` for custom shields** - This is the correct structure
2. ✅ **Your shield inheritance approach is optimal** - `select SHIELD_*` in Kconfig.shield
3. ✅ **Your Zephyr 4.1 compatibility fixes are complete** - HWMv2 names, no CONFIG_WS2812_STRIP
4. ⚠️ **To add RGB underglow:** Create board-specific overlays in `config/boards/shields/<shield>/boards/<board>.overlay`
5. ❌ **Don't convert to module** unless you plan to share the configuration publicly

**No structural changes needed** - Your current setup follows ZMK best practices for user config repositories with custom shields.
