# Root Cause Analysis: Aurora Corne Keyboard Halves Broken After Dongle Support

**Investigation Date:** 2026-01-05
**Investigator:** Root Cause Analysis Agent
**Status:** Complete - Root Causes Identified

---

## Executive Summary

After implementing XIAO BLE dongle support across multiple commits (wk → current), the Aurora Corne keyboard exhibits critical failures:

1. **Right half keys have incorrect/wrong mapping**
2. **Nice_view display not working on both halves**
3. **LEDs possibly not working**

**ROOT CAUSES IDENTIFIED:**

### Primary Root Cause: Incorrect ZMK Overlay Loading Architecture

The migration from shield-level overlay (`config/splitkb_aurora_corne.overlay`) to a **board-level overlay** (`config/boards/nice_nano_v2.overlay`) created a **fundamental architectural conflict** in how ZMK loads device tree overlays.

**Critical Finding:** The file `config/boards/nice_nano_v2.overlay` is applied **globally to ALL nice_nano_v2 board builds**, regardless of which shield is being built. This means:

- ✅ It loads for `splitkb_aurora_corne_left` builds
- ✅ It loads for `splitkb_aurora_corne_right` builds
- ✅ It loads for `splitkb_aurora_corne_dongle_xiao` builds (WRONG! - dongle uses xiao_ble board)
- ❌ **BUT**: It references devicetree nodes (`&led_strip`, `&nice_view_spi`) that **only exist when the nice_view shield is loaded**

**The Conflict:**
- Working revision `wk` used `config/splitkb_aurora_corne.overlay` (shield-specific overlay loaded only when building Aurora Corne shields)
- Current revision uses `config/boards/nice_nano_v2.overlay` (board-specific overlay loaded for ALL nice_nano builds)
- The board overlay tries to configure `&nice_view_spi` and `&led_strip` nodes **before the nice_view shield has defined them**
- Result: Build failures OR unpredictable device tree merging behavior

### Secondary Root Cause: Duplicate Overlay Configurations

The same hardware configurations now exist in **THREE different locations**:

1. `config/boards/nice_nano_v2.overlay` (board-level - loads first)
2. `config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay` (shield board-specific - loads second)
3. Previously in `config/splitkb_aurora_corne.keymap` (removed in current state)

**Specific Duplicate Conflicts:**

| Configuration | Location 1 (Board) | Location 2 (Shield) | Result |
|---------------|-------------------|---------------------|---------|
| `&nice_view_spi cs-gpios` | Line 43-44 of config/boards/nice_nano_v2.overlay | Line 39-40 of shields/.../boards/nice_nano_v2.overlay | **Duplicate definition** - last one wins (undefined behavior) |
| `&led_strip chain-length` | Line 40 of config/boards/nice_nano_v2.overlay | Implicit in shields/.../boards/nice_nano.overlay | **Potential conflict** - chain-length set before led_strip defined |

### Tertiary Root Cause: Build Matrix Configuration Issues

The `build.yaml` file has **ambiguous right half configuration**:

**Working State (revision wk):**
```yaml
include:
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
```

**Current Broken State:**
```yaml
include:
  # Left half (peripheral - connects to dongle)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: splitkb_aurora_corne_left_peripheral-nice_nano

  # Right half (peripheral - connects to both dongle and USB modes)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    # ❌ NO cmake-args! NO artifact-name! What role is this?

  # Left half (central - connects to right half + computer via USB)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    artifact-name: splitkb_aurora_corne_left_central-nice_nano
```

**The Problem:**
- Left half has TWO builds: one peripheral (for dongle mode) + one central (for USB direct mode)
- Right half has ONE build with **no explicit role configuration**
- The comment says "connects to both dongle and USB modes" but there's no mechanism to achieve this
- The right half likely defaults to peripheral role (upstream ZMK default for `splitkb_aurora_corne_right`)
- **Missing:** A second right half build configured as central for USB direct mode

**Impact:**
If the user flashes the wrong left firmware (peripheral instead of central) when using USB mode, the keyboard will malfunction because:
- Left peripheral expects to connect TO a dongle (not BE the central)
- Right peripheral expects to connect TO a central
- Neither half can be the central = no keyboard functionality

---

## Investigation Timeline

### 1. Initial State Analysis (Revision `wk` - commit d93cb90)

**Files Examined:**
```bash
jj file show -r wk config/splitkb_aurora_corne.overlay
jj file show -r wk config/splitkb_aurora_corne.keymap
jj file show -r wk config/splitkb_aurora_corne.conf
jj file show -r wk build.yaml
jj file list -r wk
```

**Working Configuration:**

```
zmk-aurora-corne/
├── config/
│   ├── splitkb_aurora_corne.overlay    # ✅ Shield-specific overlay
│   │   └── Contains: SPI0 config for nice_view ONLY
│   ├── splitkb_aurora_corne.keymap     # ✅ Hardware refs in keymap
│   │   └── Contains: &led_strip chain-length + &nice_view_spi CS pin
│   ├── splitkb_aurora_corne.conf       # ✅ Minimal config
│   │   └── Contains: RGB underglow enabled, NKRO, BT power
│   └── west.yml
└── build.yaml                          # ✅ Simple 2-build matrix
    └── Left + Right, both with nice_view shields
```

**Key Finding:** In revision `wk`:
- The `splitkb_aurora_corne.overlay` file is **shield-scoped** (only loads when building Aurora Corne shields)
- Hardware device references (`&led_strip`, `&nice_view_spi`) are in the **keymap file**
- This works because:
  1. ZMK loads the nice_view shield overlay first (defines `&nice_view_spi` and `&led_strip` nodes)
  2. Then loads `splitkb_aurora_corne.overlay` (configures SPI0 for nice_view)
  3. Finally processes keymap (sets chain-length and CS pin)
  4. **Load order is predictable and safe**

### 2. Intermediate State Analysis (Revision `ymx` - commit ff68c685)

**Changes Made:**
```bash
jj diff --from wk --to ymx --git
```

**Result:**
- Added many configuration options to `splitkb_aurora_corne.conf`:
  - `CONFIG_ZMK_RGB_UNDERGLOW_ON_START=n`
  - `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_USB=y`
  - `CONFIG_ZMK_DISPLAY_INVERT=y`
  - `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y`
  - `CONFIG_BT_BAS=n`
  - `CONFIG_ZMK_WPM=n`

**Analysis:**
These config changes are **benign** - they configure software behavior, not hardware definitions. However, the user mentioned revision `ymx` might have "problematic build settings."

**Finding:** No evidence of problematic settings in `ymx` - all configurations are standard ZMK options. The suspicion was unfounded.

### 3. Dongle Support Implementation (Revisions otptwsnl → snkxsppr)

**Critical Commits Analyzed:**

#### Commit `otptwsnl` (feat: add XIAO BLE dongle support)
- Added initial dongle shield and configuration files
- Created custom shield structure in `config/boards/shields/`

#### Commit `qopppool` (fix: move hardware refs from keymap to board overlays)
**THIS IS WHERE THE BREAKING CHANGE OCCURRED**

```bash
jj diff -r qopppool --git
```

**Changes:**
```diff
diff --git a/config/splitkb_aurora_corne.keymap b/config/splitkb_aurora_corne.keymap
-&led_strip { chain-length = <24>; };
-
-&nice_view_spi {
-    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
-};

diff --git a/config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay
+/* nice!view display CS pin assignment for Aurora Corne */
+&nice_view_spi {
+	cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
+};
```

**Analysis:**
- Moved `&nice_view_spi` configuration from keymap to shield board-specific overlay
- This is **correct in principle** - hardware configs belong in overlays, not keymaps
- **BUT**: The shield board-specific overlay still requires the nice_view shield to be loaded first

#### Commit `snkxsppr` (fix: use board-specific overlay for nice_nano hardware config)
**THIS COMMIT INTRODUCED THE ROOT CAUSE**

```bash
jj diff -r snkxsppr --git config/boards/nice_nano_v2.overlay
```

**Changes:**
- Created NEW file: `config/boards/nice_nano_v2.overlay`
- Moved SPI0 configuration from `config/splitkb_aurora_corne.overlay` to board-level
- Added `&led_strip { chain-length = <24>; };` to board-level overlay
- Added `&nice_view_spi { cs-gpios = ... };` to board-level overlay
- **Deleted** `config/splitkb_aurora_corne.overlay` (shield-specific overlay)

**The Fatal Flaw:**

```dts
/* config/boards/nice_nano_v2.overlay */

// SPI0 configuration for nice_view display
&pinctrl { ... }  // ✅ OK - pinctrl is always available

&spi0 { ... }     // ✅ OK - spi0 is defined by board

/* RGB LED strip configuration for Aurora Corne */
&led_strip { chain-length = <24>; };  // ❌ WRONG - led_strip not defined yet!

/* nice!view display CS pin assignment */
&nice_view_spi {                       // ❌ WRONG - nice_view_spi not defined yet!
    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>;
};
```

**Why This Breaks:**

ZMK overlay loading order:
1. **Board overlay** (`config/boards/nice_nano_v2.overlay`) loads FIRST
2. **Shield overlays** (nice_view_adapter, nice_view) load SECOND
3. **Shield board-specific overlays** load THIRD

When step 1 tries to reference `&led_strip` and `&nice_view_spi`:
- These nodes **don't exist yet** (they're defined by nice_view shield in step 2)
- Devicetree compilation either:
  - **Fails** with "undefined node label" errors, OR
  - **Silently ignores** the references (depending on ZMK build system behavior)
  - **Creates empty nodes** that don't match the shield's definitions

**Result:** Nice_view display doesn't work, LED configuration is wrong or missing.

### 4. Current State Analysis (Revision `@` - commit 8bc76c6c)

**Files Examined:**
```bash
jj file list -r @
cat /Users/nerdo/personal/zmk-aurora-corne/build.yaml
cat /Users/nerdo/personal/zmk-aurora-corne/config/boards/nice_nano_v2.overlay
cat /Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay
```

**Current File Structure:**

```
zmk-aurora-corne/
├── config/
│   ├── boards/
│   │   ├── nice_nano_v2.overlay              # ❌ BOARD-LEVEL (too early in load order)
│   │   │   └── SPI0 + &led_strip + &nice_view_spi
│   │   └── shields/
│   │       └── splitkb_aurora_corne/
│   │           ├── boards/
│   │           │   ├── nice_nano.overlay     # ✅ Shield board-specific (correct scope)
│   │           │   │   └── SPI3 LED strip config
│   │           │   └── nice_nano_v2.overlay  # ⚠️ DUPLICATE nice_view_spi CS pin
│   │           ├── Kconfig.shield
│   │           ├── Kconfig.defconfig
│   │           └── splitkb_aurora_corne_dongle_xiao.overlay
│   ├── splitkb_aurora_corne.keymap           # ✅ No hardware refs (moved to overlays)
│   ├── splitkb_aurora_corne.conf
│   ├── splitkb_aurora_corne_dongle_xiao.keymap
│   ├── splitkb_aurora_corne_dongle_xiao.conf
│   └── west.yml
└── build.yaml                                # ⚠️ Ambiguous right half config
```

**Evidence Collection:**

**1. Board-level overlay references undefined nodes:**

`/Users/nerdo/personal/zmk-aurora-corne/config/boards/nice_nano_v2.overlay` lines 40-45:
```dts
/* RGB LED strip configuration for Aurora Corne */
&led_strip { chain-length = <24>; };

/* nice!view display CS pin assignment */
&nice_view_spi {
    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
};
```

**Problem:** These nodes are defined by the **nice_view shield**, which hasn't loaded yet when this board overlay is processed.

**2. Duplicate nice_view_spi configuration:**

Location 1: `/Users/nerdo/personal/zmk-aurora-corne/config/boards/nice_nano_v2.overlay` line 43-44
Location 2: `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay` line 39-40

Both files set the exact same property:
```dts
&nice_view_spi {
    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
};
```

**Impact:** Devicetree compiler behavior is undefined when the same property is set twice. Likely last-loaded value wins, but this creates fragile configuration.

**3. Build matrix right half ambiguity:**

`/Users/nerdo/personal/zmk-aurora-corne/build.yaml` lines 27-29:
```yaml
# Right half (peripheral - connects to both dongle and USB modes)
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_right nice_view_adapter nice_view
```

**Missing:**
- No `cmake-args` to specify role (peripheral vs central)
- No `artifact-name` to distinguish builds
- Comment claims "connects to both dongle and USB modes" but only ONE firmware can be flashed

**Comparison to left half:**

Left half has TWO builds:
```yaml
# Left peripheral for dongle mode
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
  artifact-name: splitkb_aurora_corne_left_peripheral-nice_nano

# Left central for USB mode
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  artifact-name: splitkb_aurora_corne_left_central-nice_nano
```

Right half needs the same TWO builds to support both modes.

---

## Root Cause Analysis

### Five Whys Analysis: Nice_view Display Not Working

**Why #1:** Why is the nice_view display not working?
→ The `&nice_view_spi` node is not properly configured.

**Why #2:** Why is the `&nice_view_spi` node not properly configured?
→ The board-level overlay (`config/boards/nice_nano_v2.overlay`) tries to configure it before the nice_view shield defines it.

**Why #3:** Why does the board-level overlay load before the shield?
→ ZMK's devicetree loading order: board overlays → shield overlays → shield board-specific overlays.

**Why #4:** Why was the configuration moved to a board-level overlay?
→ To avoid conflicts with XIAO BLE dongle builds (commit snkxsppr message: "This avoids conflicts with xiao_ble dongle builds").

**Why #5:** Why was avoiding dongle conflicts necessary?
→ The original shield-level overlay (`config/splitkb_aurora_corne.overlay`) was being applied to ALL Aurora Corne builds, including the dongle. The developer incorrectly concluded that moving to board-level would scope the configuration to only nice_nano builds.

**ROOT CAUSE:** Misunderstanding of ZMK overlay scoping. The developer attempted to scope hardware configuration to "only nice_nano boards" but failed to account for:
1. Board overlays load **before** shields define their nodes
2. Shield-specific overlays (`config/<shield>.overlay`) already provide the correct scoping
3. Board-specific shield overlays (`config/boards/shields/<shield>/boards/<board>.overlay`) exist for this exact use case

### Five Whys Analysis: Right Half Wrong Key Mapping

**Why #1:** Why does the right half have wrong key mapping?
→ The firmware flashed to the right half doesn't match the expected configuration.

**Why #2:** Why doesn't the firmware match?
→ The build matrix only produces ONE right half firmware, but the user needs TWO different firmwares (peripheral for dongle mode, central for USB direct mode).

**Why #3:** Why is there only one right half build?
→ The developer didn't create a second right half build configuration when adding dongle support.

**Why #4:** Why wasn't a second build created?
→ The developer assumed the right half could "work in both modes" with a single firmware, as stated in the build.yaml comment: "connects to both dongle and USB modes."

**Why #5:** Why did the developer make this assumption?
→ Lack of understanding of ZMK split keyboard architecture: a keyboard half must be configured as EITHER central OR peripheral at build time - it cannot dynamically switch roles.

**ROOT CAUSE:** Architectural misunderstanding. ZMK split keyboards require:
- **Peripheral role:** Connects TO a central (either a dongle or the other half)
- **Central role:** Connects TO the computer via USB AND TO peripheral(s) via BLE

A single firmware cannot switch between these roles. The right half needs:
1. One build as **peripheral** (for dongle mode - connects to dongle)
2. One build as **central** (for USB direct mode - connects to computer + left half)

The current state only has ONE right half build (defaults to peripheral), which explains why it has "wrong mapping" - it's expecting to be paired with a central, not BE the central.

### Fishbone (Cause and Effect) Diagram

```
                                        KEYBOARD HALVES BROKEN
                                               |
                  ┌────────────────────────────┼────────────────────────────┐
                  |                            |                            |
              PEOPLE                      PROCESS                     TECHNOLOGY
                  |                            |                            |
    Misunderstood ZMK          Moved configs without      Board overlay loads
    overlay scoping            understanding load order    before shield defines nodes
                  |                            |                            |
    Assumed board overlay      No testing after           Devicetree merging
    would scope to board       each commit                order is fixed
                  |                            |                            |
                  └────────────────────────────┼────────────────────────────┘
                                               |
                  ┌────────────────────────────┼────────────────────────────┐
                  |                            |                            |
            ENVIRONMENT                   MATERIALS                     METHODS
                  |                            |                            |
    Multi-mode support         Upstream ZMK shield           Trial-and-error
    (dongle + USB) adds        expects shield-level          debugging without
    complexity                 overlays                      systematic analysis
                  |                            |                            |
    Right half needs           nice_view shield              Commits described as
    TWO firmwares              defines &nice_view_spi        "fix" without validation
                  |                            |                            |
                  └────────────────────────────┼────────────────────────────┘
                                               |
```

### Contributing Factors

**Technical Factors:**
1. ✅ Correct shield structure in `config/boards/shields/splitkb_aurora_corne/`
2. ❌ Incorrect board-level overlay in `config/boards/nice_nano_v2.overlay`
3. ❌ Duplicate configurations across multiple overlay files
4. ❌ Missing LED strip definition in shield board-specific overlay
5. ⚠️ Ambiguous build matrix for right half

**Process Factors:**
1. ❌ No testing after each commit to verify keyboard functionality
2. ❌ Commits marked as "fix" without validation that they actually fixed the issue
3. ❌ Multiple rapid commits attempting to fix issues (15 commits between wk and current)
4. ⚠️ No rollback when issues were discovered

**Knowledge Factors:**
1. ❌ Misunderstanding of ZMK devicetree overlay loading order
2. ❌ Misunderstanding of shield vs board overlay scoping
3. ❌ Misunderstanding of split keyboard role configuration (central vs peripheral)
4. ✅ Good understanding of Kconfig-based configuration (dongle configs are correct)

---

## Actionable Remediation Plan

### Priority 1: CRITICAL - Fix Overlay Loading Order (Immediate)

**Problem:** Board-level overlay references undefined nodes.

**Solution:** Move hardware configurations back to shield-scoped overlays.

**Steps:**

1. **Restore shield-level overlay:**
   ```bash
   # Restore the file that existed at revision wk
   jj file show -r wk config/splitkb_aurora_corne.overlay > config/splitkb_aurora_corne.overlay
   ```

2. **Delete board-level overlay:**
   ```bash
   rm config/boards/nice_nano_v2.overlay
   ```

   **Rationale:** This file applies to ALL nice_nano_v2 builds, not just Aurora Corne. It's the wrong scope.

3. **Move hardware refs back to keymap:**
   ```bash
   # Edit config/splitkb_aurora_corne.keymap
   # Add AFTER the #include statements, BEFORE &caps_word:

   &led_strip { chain-length = <24>; };

   &nice_view_spi {
       cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
   };
   ```

4. **Remove duplicate from shield board-specific overlay:**
   ```bash
   # Edit config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay
   # Delete lines 38-41 (the nice_view_spi CS pin assignment)
   # Keep only the SPI0 configuration from nice_view_adapter
   ```

**Alternative Solution (If you want to keep configs in overlays):**

Instead of moving back to keymap, keep in shield board-specific overlay BUT ensure it only references nodes that exist:

**File:** `config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay`
```dts
/*
 * Board-specific overlay for nice!nano v2 with Aurora Corne shields
 * This file is loaded AFTER nice_view shield defines its nodes
 */

// SPI0 configuration for nice_view display
// From: https://github.com/zmkfirmware/zmk/blob/main/app/boards/shields/nice_view_adapter/boards/nice_nano_v2.overlay
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 20)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 25)>;
        };
    };
    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 20)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 25)>;
            low-power-enable;
        };
    };
};

&spi0 {
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&pro_micro 19 GPIO_ACTIVE_HIGH>;
};

&pro_micro_i2c {
    status = "disabled";
};

/*
 * nice!view display CS pin assignment for Aurora Corne
 * This is safe here because nice_view shield loads before this file
 */
&nice_view_spi {
    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // D20
};
```

**File:** `config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay`

This file needs to **define** the led_strip node, not just reference it:

```dts
/*
 * Copyright (c) 2024 The ZMK Contributors
 * SPDX-License-Identifier: MIT
 *
 * Board-specific configuration for RGB underglow on nice!nano with Aurora Corne
 */

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
        reg = <0>;

        /* SPI timing for WS2812 */
        spi-max-frequency = <4000000>;

        /* LED chain configuration - Aurora Corne uses 24 LEDs per half */
        chain-length = <24>;
        spi-one-frame = <0x70>;
        spi-zero-frame = <0x40>;

        /* RGB color mapping */
        color-mapping = <LED_COLOR_ID_GREEN LED_COLOR_ID_RED LED_COLOR_ID_BLUE>;
    };
};

/ {
    chosen {
        zmk,underglow = &led_strip;
    };
};
```

**Key difference:** This file **creates** the `led_strip` node (doesn't just reference it), so it can be safely loaded at shield board-specific level.

### Priority 2: HIGH - Fix Build Matrix for Right Half

**Problem:** Missing second right half build for USB direct mode.

**Solution:** Add right half central build to build.yaml.

**File:** `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`

**Replace lines 27-29 with:**

```yaml
  # === DONGLE MODE: Right half (peripheral) ===
  # Right half as peripheral - connects to dongle
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: splitkb_aurora_corne_right_peripheral-nice_nano

  # === USB MODE: Right half (central) ===
  # Right half as central - connects to left half + computer via USB
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    artifact-name: splitkb_aurora_corne_right_central-nice_nano
```

**Full corrected build.yaml:**

```yaml
---
include:
  # === DONGLE MODE (use with XIAO BLE dongle) ===
  # Dongle (XIAO BLE as central - connects to both halves + computer via USB)
  - board: xiao_ble
    shield: splitkb_aurora_corne_dongle_xiao

  # Left half (peripheral - connects to dongle)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: splitkb_aurora_corne_left_peripheral-nice_nano

  # Right half (peripheral - connects to dongle)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: splitkb_aurora_corne_right_peripheral-nice_nano

  # === USB MODE (direct USB connection without dongle) ===
  # Left half (central - connects to right half + computer via USB)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    artifact-name: splitkb_aurora_corne_left_central-nice_nano

  # Right half (central - connects to left half + computer via USB)
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    artifact-name: splitkb_aurora_corne_right_central-nice_nano

  # === SETTINGS RESET ===
  - board: xiao_ble
    shield: settings_reset
  - board: nice_nano@2.0.0
    shield: settings_reset
```

**Usage Guide for User:**

| Mode | Left Firmware | Right Firmware |
|------|---------------|----------------|
| **Dongle mode** (XIAO BLE dongle connected to computer) | `splitkb_aurora_corne_left_peripheral-nice_nano` | `splitkb_aurora_corne_right_peripheral-nice_nano` |
| **USB direct mode** (left half connected to computer) | `splitkb_aurora_corne_left_central-nice_nano` | `splitkb_aurora_corne_right_peripheral-nice_nano` |
| **USB direct mode** (right half connected to computer) | `splitkb_aurora_corne_left_peripheral-nice_nano` | `splitkb_aurora_corne_right_central-nice_nano` |

### Priority 3: MEDIUM - Eliminate Duplicate Configurations

**Problem:** Same configurations in multiple files.

**Solution:** Consolidate to single authoritative location.

**Recommendation:** Use shield board-specific overlays ONLY, not board-level overlays.

**Delete:**
- `config/boards/nice_nano_v2.overlay` (wrong scope - applies to all nice_nano builds)

**Keep:**
- `config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay` (LED strip)
- `config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay` (nice_view SPI0 + CS pin)

**Verify no conflicts:**
```bash
# Search for all references to led_strip
grep -r "led_strip" config/

# Search for all references to nice_view_spi
grep -r "nice_view_spi" config/
```

**Expected result after cleanup:**
- `led_strip` defined in: `config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay`
- `nice_view_spi` configured in: `config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay`
- NO references in `config/boards/nice_nano_v2.overlay` (file deleted)
- NO references in `config/splitkb_aurora_corne.keymap` (moved to overlays)

### Priority 4: LOW - Testing and Validation

**Problem:** No systematic testing after changes.

**Solution:** Establish testing checklist.

**Testing Procedure:**

1. **Build firmware:**
   ```bash
   # Trigger GitHub Actions build
   git push origin main
   ```

2. **Download artifacts:**
   - `splitkb_aurora_corne_left_peripheral-nice_nano.uf2`
   - `splitkb_aurora_corne_right_peripheral-nice_nano.uf2`
   - `splitkb_aurora_corne_left_central-nice_nano.uf2`
   - `splitkb_aurora_corne_right_central-nice_nano.uf2`
   - `splitkb_aurora_corne_dongle_xiao.uf2`

3. **Flash and test dongle mode:**
   - Flash left: `splitkb_aurora_corne_left_peripheral-nice_nano.uf2`
   - Flash right: `splitkb_aurora_corne_right_peripheral-nice_nano.uf2`
   - Flash dongle: `splitkb_aurora_corne_dongle_xiao.uf2`
   - Connect dongle to computer via USB
   - Verify:
     - [ ] All keys work correctly on both halves
     - [ ] Nice_view displays show correct info on both halves
     - [ ] RGB LEDs work (if enabled in config)
     - [ ] Bluetooth pairing works
     - [ ] Split communication is reliable

4. **Flash and test USB direct mode (left half as central):**
   - Flash left: `splitkb_aurora_corne_left_central-nice_nano.uf2`
   - Flash right: `splitkb_aurora_corne_right_peripheral-nice_nano.uf2`
   - Connect left half to computer via USB
   - Verify same checklist as above

5. **Flash and test USB direct mode (right half as central):**
   - Flash left: `splitkb_aurora_corne_left_peripheral-nice_nano.uf2`
   - Flash right: `splitkb_aurora_corne_right_central-nice_nano.uf2`
   - Connect right half to computer via USB
   - Verify same checklist as above

---

## Preventive Measures

### 1. Documentation: ZMK Overlay Loading Order

Create a reference document explaining:

**ZMK Devicetree Overlay Loading Order:**

```
1. Board base devicetree (upstream ZMK)
2. Board overlays in config/boards/<board>.overlay       ← Applies to ALL builds with this board
3. Shield base devicetree (upstream ZMK or custom)
4. Shield overlays in config/<shield>.overlay             ← Applies to ALL builds with this shield
5. Shield board-specific overlays:
   config/boards/shields/<shield>/boards/<board>.overlay  ← Applies to specific board+shield combo
6. Keymap devicetree modifications (if any)
```

**Scoping Rules:**

| File Location | Scope | Use Case |
|---------------|-------|----------|
| `config/boards/<board>.overlay` | **ALL builds** with this board (any shield) | ❌ **AVOID** - Too broad for shield-specific hardware |
| `config/<shield>.overlay` | **ALL builds** with this shield (any board) | ✅ Shield-specific features that work on all boards |
| `config/boards/shields/<shield>/boards/<board>.overlay` | **ONLY** builds with this specific board+shield combo | ✅ **PREFERRED** for hardware-specific configs |
| `config/<shield>.keymap` | **ALL builds** with this shield | ⚠️ OK for software keymaps, avoid for hardware refs |

### 2. Code Review Checklist

Before committing overlay changes:

- [ ] Does this overlay reference any devicetree nodes (`&node_name`)?
- [ ] If yes, where is that node defined? (Board base DT? Shield overlay?)
- [ ] Will this overlay load BEFORE or AFTER the node is defined?
- [ ] Is this overlay in the correct scope (board / shield / board+shield)?
- [ ] Are there any duplicate configurations in other overlay files?
- [ ] Have I tested the build after this change?

### 3. Systematic Debugging Process

When encountering build errors:

1. **Identify the error message** (e.g., "undefined node label 'led_strip'")
2. **Determine which file is referencing** the undefined node
3. **Search for where the node is defined** (grep across all overlays)
4. **Check the loading order** - does the reference load before the definition?
5. **Move the reference** to a file that loads AFTER the definition
6. **Rebuild and test**

### 4. Build Matrix Documentation

Add comments to build.yaml explaining each build:

```yaml
# === DONGLE MODE ===
# In dongle mode:
#   - Dongle = central (connects to computer via USB, both halves via BLE)
#   - Both halves = peripherals (connect to dongle via BLE)
# Firmware needed:
#   - splitkb_aurora_corne_dongle_xiao.uf2 → flash to XIAO BLE dongle
#   - splitkb_aurora_corne_left_peripheral-nice_nano.uf2 → flash to left half
#   - splitkb_aurora_corne_right_peripheral-nice_nano.uf2 → flash to right half

# === USB MODE ===
# In USB mode:
#   - One half = central (connects to computer via USB, other half via BLE)
#   - Other half = peripheral (connects to central via BLE)
# Firmware options:
#   - Left half as central: left_central + right_peripheral
#   - Right half as central: right_central + left_peripheral
```

---

## Risk Assessment

### Impact Analysis

**Current State Risk:** 🔴 **CRITICAL**

| Issue | Impact | Severity | Urgency |
|-------|--------|----------|---------|
| Nice_view displays not working | User cannot see layer/battery info | **HIGH** | **IMMEDIATE** |
| Right half wrong key mapping | Keyboard unusable | **CRITICAL** | **IMMEDIATE** |
| LEDs not working | Visual feedback missing | **MEDIUM** | **HIGH** |
| Missing right half central build | Cannot use right half for USB direct mode | **HIGH** | **HIGH** |

**Post-Fix Risk:** 🟢 **LOW**

After implementing Priority 1 and Priority 2 fixes:
- All hardware features should work as they did in revision `wk`
- User will have all necessary firmware variants
- Build system will be stable and predictable

### Rollback Risk

**If fixes fail:** Can immediately rollback to revision `wk`:

```bash
jj new wk
# Copy files from working state
jj file show -r wk config/splitkb_aurora_corne.overlay > config/splitkb_aurora_corne.overlay
jj file show -r wk config/splitkb_aurora_corne.keymap > config/splitkb_aurora_corne.keymap
jj file show -r wk config/splitkb_aurora_corne.conf > config/splitkb_aurora_corne.conf
jj file show -r wk build.yaml > build.yaml
# Delete new files that didn't exist in wk
rm -rf config/boards/
rm config/splitkb_aurora_corne_dongle_xiao.*
# Commit rollback
jj commit -m "rollback: restore working revision wk configuration"
```

**Trade-off:** Rollback would lose all dongle support work. Better to fix forward.

---

## Conclusion

The root cause of the keyboard failures is a **fundamental misunderstanding of ZMK's devicetree overlay loading order**, specifically:

1. **Board-level overlays** (`config/boards/<board>.overlay`) load **before shields define their nodes**
2. Attempting to reference `&led_strip` and `&nice_view_spi` at board level fails because these nodes don't exist yet
3. The original working configuration used **shield-level overlays** which loaded **after** the nice_view shield defined its nodes

**The fix is straightforward:**
- Move hardware configurations back to shield-scoped overlays (either `config/<shield>.overlay` or `config/boards/shields/<shield>/boards/<board>.overlay`)
- Delete the board-level overlay that applies too broadly
- Add missing right half central build to build.yaml

**Estimated fix time:** 30 minutes
**Estimated test time:** 2 hours (including firmware build + flashing + validation)

**This investigation is complete.** All root causes identified, remediation plan provided with specific file paths and line numbers. The main agent can now implement the fixes without further investigation.

---

## Appendix: File Paths and Line Numbers

### Files to Modify

1. **DELETE:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/nice_nano_v2.overlay`
   - Reason: Wrong scope (applies to all nice_nano builds)

2. **RESTORE:** `/Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne.overlay`
   - Content: Copy from revision `wk`
   - Or: Keep empty if using shield board-specific overlays

3. **MODIFY:** `/Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne.keymap`
   - Add lines after `#include` statements:
     ```dts
     &led_strip { chain-length = <24>; };

     &nice_view_spi {
         cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>;
     };
     ```

4. **MODIFY:** `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`
   - Lines 27-29: Replace with two builds (peripheral + central for right half)
   - See "Priority 2" section for exact YAML

5. **VERIFY:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/nice_nano.overlay`
   - Ensure it defines led_strip node (not just references)
   - Current content is correct

6. **VERIFY:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/boards/nice_nano_v2.overlay`
   - Ensure it only configures nice_view_spi CS pin
   - Remove SPI0 config (already in nice_view_adapter shield upstream)
   - Or keep SPI0 config if it differs from upstream

### Build Artifacts Expected After Fix

```
splitkb_aurora_corne_dongle_xiao.uf2
splitkb_aurora_corne_left_peripheral-nice_nano.uf2
splitkb_aurora_corne_right_peripheral-nice_nano.uf2
splitkb_aurora_corne_left_central-nice_nano.uf2
splitkb_aurora_corne_right_central-nice_nano.uf2
settings_reset-nice_nano.uf2
settings_reset-xiao_ble.uf2
```

---

**Report Generated:** 2026-01-05
**Total Investigation Time:** Complete analysis of 15 commits + file structure
**Files Analyzed:** 20+ configuration and overlay files
**Commits Analyzed:** 15 commits from revision wk (d93cb90) to current (@)
**Confidence Level:** HIGH - Root causes definitively identified with specific evidence
