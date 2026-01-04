# ZMK Dongle Configuration Research: Corne with XIAO BLE

**Research Date:** 2026-01-04
**Target Hardware:** Seeeduino XIAO BLE (seeeduino_xiao_ble)
**Keyboard:** Corne (splitkb_aurora_corne)
**Current Setup:** nice!nano v2 with nice!view displays (no dongle)

---

## Executive Summary

### Critical Findings

1. **Dongle setup requires creating a custom shield** in `boards/shields/corne/` (or your keyboard variant) with three core files: `Kconfig.shield`, `Kconfig.defconfig`, and `<shield>_dongle_xiao.overlay`

2. **XIAO BLE is fully supported** as a dongle controller in ZMK. Multiple working configurations exist in the community using `seeeduino_xiao_ble` board with Corne keyboards.

3. **Settings reset is mandatory** before first pairing. All three devices (dongle + left + right) must be flashed with settings_reset firmware before flashing the dongle configuration.

4. **Peripheral conversion required** for existing keyboard halves. Your current nice!nano v2 boards must be converted to peripheral-only mode using cmake args: `-DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`

5. **ZMK version compatibility**: Your repository appears to be on Zephyr 4.1 (based on recent commits). XIAO BLE requires special pin handling for nRF52840 NFC pins on Zephyr 4.1.

### Key Benefits

- **Improved battery life**: Left keyboard (former central) battery life increases from 2-4 weeks to 5-8 months as peripheral
- **Reduced latency**: Average latency decreases by ~6.5ms for peripherals (though former central adds ~1ms from USB hop)
- **USB always connected**: Dongle acts as permanent USB connection point

---

## 1. Shield Definition Structure

### Required File Structure

```
zmk-aurora-corne/
├── boards/
│   └── shields/
│       └── splitkb_aurora_corne/
│           ├── Kconfig.shield                    # Shield declaration
│           ├── Kconfig.defconfig                 # Dongle-specific config
│           ├── splitkb_aurora_corne_dongle_xiao.overlay  # Hardware definition
│           ├── splitkb_aurora_corne_dongle_xiao.conf     # Dongle settings
│           ├── splitkb_aurora_corne_left_peripheral.overlay (optional)
│           └── splitkb_aurora_corne_left_peripheral.conf (optional)
└── build.yaml                                     # Build configuration
```

### File: `Kconfig.shield`

**Purpose:** Declares available shield variants to the build system.

**Location:** `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/Kconfig.shield`

**Content:**
```kconfig
# Copyright (c) 2020 Pete Johanson
# SPDX-License-Identifier: MIT

config SHIELD_SPLITKB_AURORA_CORNE_DONGLE_XIAO
    def_bool $(shields_list_contains,splitkb_aurora_corne_dongle_xiao)

config SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_left_peripheral)

config SHIELD_SPLITKB_AURORA_CORNE_RIGHT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_right_peripheral)
```

**Notes:**
- Add these entries to existing `Kconfig.shield` if it already exists
- The `def_bool` syntax automatically enables the config when the shield name is in the build list
- Naming convention: `SHIELD_<UPPERCASE_SHIELD_NAME>`

### File: `Kconfig.defconfig`

**Purpose:** Configures BLE central role, peripheral count, and connection limits for the dongle.

**Location:** `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/Kconfig.defconfig`

**Content:**
```kconfig
# Copyright (c) 2020 Pete Johanson
# SPDX-License-Identifier: MIT

if SHIELD_SPLITKB_AURORA_CORNE_DONGLE_XIAO

config ZMK_KEYBOARD_NAME
    default "Aurora Corne"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

config ZMK_SPLIT
    default y

# Set this to the number of peripherals (left + right = 2)
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 2

# Formula: ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS + desired BT profiles (default 5)
# 2 peripherals + 5 profiles = 7
config BT_MAX_CONN
    default 7

# Must match BT_MAX_CONN
config BT_MAX_PAIRED
    default 7

if ZMK_DISPLAY

config ZMK_DISPLAY_WORK_QUEUE_DEDICATED
    default y

endif # ZMK_DISPLAY

endif # SHIELD_SPLITKB_AURORA_CORNE_DONGLE_XIAO
```

**Configuration Breakdown:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `ZMK_KEYBOARD_NAME` | "Aurora Corne" | Max 16 characters, appears in Bluetooth name |
| `ZMK_SPLIT_ROLE_CENTRAL` | `y` | Dongle acts as BLE central |
| `ZMK_SPLIT` | `y` | Enable split keyboard functionality |
| `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` | `2` | Number of peripherals (left + right) |
| `BT_MAX_CONN` | `7` | Total BLE connections (2 peripherals + 5 host profiles) |
| `BT_MAX_PAIRED` | `7` | Must equal BT_MAX_CONN |
| `ZMK_DISPLAY_WORK_QUEUE_DEDICATED` | `y` | Dedicated work queue for display (recommended for dongles) |

**Critical Notes:**
- `BT_MAX_CONN` = peripherals + desired host profiles (default 5)
- Both `BT_MAX_CONN` and `BT_MAX_PAIRED` must have identical values
- Display work queue prevents display updates from blocking key scanning

### File: `splitkb_aurora_corne_dongle_xiao.overlay`

**Purpose:** Defines hardware configuration using devicetree syntax. Contains mock kscan (since dongle has no keys) and matrix transform.

**Location:** `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.overlay`

**Content:**
```dts
/*
 * Copyright (c) 2020 Pete Johanson
 * SPDX-License-Identifier: MIT
 */

#include <dt-bindings/zmk/matrix_transform.h>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
        // Optional: zmk,physical-layout = &foostan_corne_6col_layout;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-mock";
        label = "KSCAN_MOCK";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;

        // 42-key Corne layout (6 columns per side)
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10) RC(0,11)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10) RC(1,11)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10) RC(2,11)
                                    RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
        >;
    };
};

// XIAO BLE I2C bus for OLED display (optional)
&xiao_i2c {
    status = "okay";

    oled: ssd1306@3c {
        compatible = "solomon,ssd1306fb";
        reg = <0x3c>;
        width = <128>;
        height = <64>;
        segment-offset = <0>;
        page-offset = <0>;
        display-offset = <0>;
        multiplex-ratio = <63>;
        segment-remap;
        com-invdir;
        inversion-on;
        prechargep = <0x22>;
    };
};
```

**Key Components Explained:**

1. **Mock KSCAN:**
   ```dts
   kscan0: kscan {
       compatible = "zmk,kscan-mock";
       columns = <0>;
       rows = <0>;
       events = <0>;
   };
   ```
   - Required because ZMK expects a keyboard scanner
   - Dongle has no physical keys, so this is a placeholder
   - All dimensions set to 0

2. **Matrix Transform:**
   - Must match your **original keyboard's matrix layout**
   - Copy from existing splitkb_aurora_corne shield definition
   - This tells the dongle how to interpret events from peripherals
   - `RC(row, col)` notation maps physical positions

3. **XIAO Pin References:**
   - XIAO uses `&xiao_d N` format (e.g., `&xiao_d 0` for D0)
   - XIAO I2C bus: `&xiao_i2c`
   - Unlike Pro Micro which uses `&pro_micro N`

4. **Optional Display Configuration:**
   - Example shows SSD1306 128x64 OLED on I2C
   - Address `0x3c` is standard for most OLEDs
   - Can also use SH1107, SH1106, or nice!view (SPI)

**Display Alternatives:**

```dts
// For 128x32 SSD1306
oled: ssd1306@3c {
    compatible = "solomon,ssd1306fb";
    reg = <0x3c>;
    width = <128>;
    height = <32>;
    segment-offset = <0>;
    page-offset = <0>;
    multiplex-ratio = <31>;
    segment-remap;
    com-invdir;
};

// For nice!view (SPI) - requires disabling I2C
&xiao_i2c {
    status = "disabled";
};

&xiao_spi {
    status = "okay";
    cs-gpios = <&xiao_d 6 GPIO_ACTIVE_HIGH>;

    nice_view: ls011b7dh01@0 {
        compatible = "sharp,ls011b7dh01";
        reg = <0>;
        spi-max-frequency = <1000000>;
        width = <160>;
        height = <68>;
    };
};
```

### File: `splitkb_aurora_corne_dongle_xiao.conf`

**Purpose:** Dongle-specific runtime configuration settings.

**Location:** `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.conf`

**Content:**
```conf
# Dongle BLE settings
CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_BT_MAX_CONN=7
CONFIG_BT_MAX_PAIRED=7

# Display support (optional)
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y

# Battery reporting from peripherals
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_BATTERY_REPORT_INTERVAL=60

# Disable battery service (dongle is USB powered)
CONFIG_BT_BAS=n

# Optional: Enable ZMK Studio support
# CONFIG_ZMK_STUDIO=y
```

**Setting Explanations:**

| Setting | Purpose |
|---------|---------|
| `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y` | Dongle fetches battery levels from peripherals to display |
| `CONFIG_ZMK_BATTERY_REPORT_INTERVAL=60` | Report battery every 60 seconds |
| `CONFIG_BT_BAS=n` | Disable Battery Service (dongle is USB-powered) |
| `CONFIG_ZMK_STUDIO=y` | Enable ZMK Studio RPC for live configuration |

---

## 2. XIAO BLE Board Definition in ZMK

### Board Identifier

**Official ZMK Board Name:** `seeeduino_xiao_ble`

**Chipset:** nRF52840 (Nordic Semiconductor)

**ZMK Support Status:** Fully integrated since Zephyr 3.0 update

### Pin Mapping Reference

XIAO BLE uses **green-labeled D-prefixed pin names** in devicetree:

| Physical Label | Devicetree Reference | GPIO Port | Notes |
|----------------|---------------------|-----------|-------|
| D0 | `&xiao_d 0` | P0.02 | |
| D1 | `&xiao_d 1` | P0.03 | |
| D2 | `&xiao_d 2` | P0.28 | |
| D3 | `&xiao_d 3` | P0.29 | |
| D4 | `&xiao_d 4` | P0.04 | |
| D5 | `&xiao_d 5` | P0.05 | |
| D6 | `&xiao_d 6` | P1.11 | Often used as SPI CS |
| D7 | `&xiao_d 7` | P1.12 | |
| D8 | `&xiao_d 8` | P1.13 | SPI SCK |
| D9 | `&xiao_d 9` | P1.14 | SPI MISO |
| D10 | `&xiao_d 10` | P1.15 | SPI MOSI |

### Pre-configured Buses

**I2C (default enabled):**
```dts
&xiao_i2c {
    status = "okay";
    // SDA: D4 (P0.04)
    // SCL: D5 (P0.05)
};
```

**SPI:**
```dts
&xiao_spi {
    status = "okay";
    // SCK: D8 (P1.13)
    // MOSI: D10 (P1.15)
    // MISO: D9 (P1.14)
    cs-gpios = <&xiao_d 6 GPIO_ACTIVE_HIGH>;  // D6 as CS
};
```

**UART:**
```dts
&xiao_serial {
    status = "okay";
    // TX: D6
    // RX: D7
};
```

### Zephyr 4.1 Compatibility Note

**CRITICAL for XIAO BLE on Zephyr 4.1:**

If using nRF52840 NFC pins (common on XIAO), you must disable NFC functionality in your overlay:

```dts
&nfct {
    status = "disabled";
};
```

Without this, builds may fail or produce unexpected behavior on Zephyr 4.1+.

**Your Current Setup:**
Based on your recent commits ("remove deprecated CONFIG_WS2812_STRIP for Zephyr 4.1 compatibility"), you're on Zephyr 4.1. Ensure NFC is disabled in the dongle overlay.

---

## 3. BT Connection Settings: Dongle vs Peripherals

### Role Architecture

```
┌─────────────┐  BLE Central-Peripheral  ┌─────────────┐
│   Dongle    │◄───────────────────────────│ Left Half   │
│ (Central)   │                            │ (Peripheral)│
└──────┬──────┘                            └─────────────┘
       │
       │ BLE Central-Peripheral             ┌─────────────┐
       └────────────────────────────────────►│ Right Half  │
                                             │ (Peripheral)│
       USB Connection                        └─────────────┘
       │
       ▼
┌─────────────┐
│    Host     │
│  Computer   │
└─────────────┘
```

### Central (Dongle) Configuration

**Board:** `seeeduino_xiao_ble`
**Shield:** `splitkb_aurora_corne_dongle_xiao`

**Kconfig Settings (in `Kconfig.defconfig` or `.conf`):**
```kconfig
CONFIG_ZMK_SPLIT=y
CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_BT_MAX_CONN=7
CONFIG_BT_MAX_PAIRED=7
```

**Behavior:**
- Listens for BLE connections from peripherals
- Aggregates key events from both halves
- Runs keymap logic and generates HID events
- Communicates with host over USB or BLE
- Manages up to 5 host profiles (laptops, phones, tablets)

### Peripheral (Left/Right Halves) Configuration

**Board:** `nice_nano@2.0.0` (your current setup)
**Shield:** `splitkb_aurora_corne_left_peripheral` or `splitkb_aurora_corne_right_peripheral`

**Method 1: Dedicated Peripheral Shield (Recommended)**

Create separate shield definitions with peripheral-specific settings:

**File:** `splitkb_aurora_corne_left_peripheral.overlay`
```dts
// Copy entire content from splitkb_aurora_corne_left.overlay
// No changes needed - overlay is identical to standard left half
```

**File:** `splitkb_aurora_corne_left_peripheral.conf`
```conf
# Peripheral-specific settings
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000  # 15 minutes

# Optional: Reduce power consumption
CONFIG_BT_CTLR_TX_PWR_0=y  # Lower TX power (vs +8dBm)
```

**Build Configuration:**
```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

**Method 2: Reuse Existing Shields with Cmake Args**

If you don't want to create separate peripheral shields, override the role at build time:

```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

**Key Settings:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `CONFIG_ZMK_SPLIT` | `y` | Enable split keyboard mode |
| `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` | `n` | **CRITICAL:** Device is peripheral, not central |

**Behavior:**
- Scans keyboard matrix for key presses
- Sends key position events to central (dongle) via BLE
- Does **not** run keymap logic
- Cannot communicate with host directly
- No USB HID functionality even when plugged in
- Better battery life (5-8 months vs 2-4 weeks)

### Connection Topology Comparison

**Before (Dongleless):**
```
Left Half (Central) ◄──BLE──► Right Half (Peripheral)
       │
       └──► Host (USB or BLE)
```

**After (With Dongle):**
```
Dongle (Central) ◄──BLE──► Left Half (Peripheral)
       │
       ├──BLE──► Right Half (Peripheral)
       │
       └──► Host (USB or BLE)
```

**Key Differences:**
- Both halves become equals (both peripherals)
- Dongle manages all host connections
- Improved battery life on both halves
- Lower average latency for peripherals
- Dongle must always be plugged in

---

## 4. Working Examples: Corne + XIAO BLE Dongle

### Community Repository Analysis

#### Example 1: redmasters/zmk-config-dongle

**Repository:** https://github.com/redmasters/zmk-config-dongle

**Structure:**
```
zmk-config-dongle/
├── boards/
│   ├── nice_nano_v2.overlay
│   ├── puchi_ble_v1.overlay
│   └── shields/
│       └── corne/
│           ├── Kconfig.shield
│           ├── Kconfig.defconfig
│           ├── corne_dongle_xiao.overlay
│           ├── corne_dongle_xiao.conf
│           ├── corne_dongle_pro_micro.overlay
│           └── corne_dongle_pro_micro.conf
├── config/
│   ├── corne.keymap
│   ├── corne.conf
│   └── west.yml
└── build.yaml
```

**Build Configuration Excerpt:**
```yaml
include:
  - board: seeeduino_xiao_ble
    shield: corne_dongle_xiao
    cmake-args: -DCONFIG_ZMK_KEYBOARD_NAME="Corne Dongle" -DCONFIG_ZMK_STUDIO=y
  - board: nice_nano_v2
    shield: corne_left_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
  - board: nice_nano_v2
    shield: corne_right_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
  # Settings reset utilities
  - board: seeeduino_xiao_ble
    shield: settings_reset
  - board: nice_nano_v2
    shield: settings_reset
```

**Key Features:**
- Supports both XIAO BLE and Pro Micro dongles
- Includes optional OLED display configurations (128x32, 128x64, 128x128, SH1107)
- Provides settings_reset targets for all boards
- Uses zmk-rgbled-widgets for LED status indicators

**Display Options in Overlay:**
```dts
// Default: SSD1306 128x64
oled: ssd1306@3c {
    compatible = "solomon,ssd1306fb";
    reg = <0x3c>;
    width = <128>;
    height = <64>;
    segment-remap;
    com-invdir;
    inversion-on;
};

// Commented alternatives available:
// - SSD1306 128x32
// - SH1106 128x64
// - SH1107 128x128
// - nice!view (SPI)
```

#### Example 2: DarrenVictoriano/zmk-config

**Repository:** https://github.com/DarrenVictoriano/zmk-config

**Description:** Custom Corne layout with nice!nano v2 + nice!view + Seed XIAO as dongle

**Highlights:**
- Demonstrates nice!view display on dongle
- Shows integration with zmk-rgbled-widgets
- Provides detailed flashing instructions

**README Flashing Instructions:**
1. Flash settings_reset to dongle
2. Flash settings_reset to left keyboard
3. Flash settings_reset to right keyboard
4. Flash dongle firmware
5. Flash left peripheral firmware
6. Flash right peripheral firmware

#### Example 3: mctechnology17/zmk-config

**Repository:** https://github.com/mctechnology17/zmk-config

**Supported Keyboards:** Corne, Sofle, Lily58

**Unique Features:**
- zmk-oled-adapter for flexible OLED sizes
- Extensive documentation on dongle setup
- Makefile for local Docker builds

**Build Command Example:**
```bash
west build -b seeeduino_xiao_ble -- \
  -DSHIELD="corne_dongle_xiao dongle_display oled_adapter_seeeduino_xiao_ble_128x64"
```

### Common Patterns Across Examples

1. **Shield naming:** `<keyboard>_dongle_xiao` or `<keyboard>_dongle_pro_micro`
2. **Peripheral shields:** `<keyboard>_left_peripheral` and `<keyboard>_right_peripheral`
3. **Settings reset inclusion:** All examples provide settings_reset build targets
4. **Display flexibility:** Most support multiple OLED sizes via comments or adapters
5. **Cmake args for peripherals:** Consistently use `-DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`

---

## 5. Custom Shield Overlays: Required or Optional?

### Short Answer: **REQUIRED**

Custom shield overlays are **mandatory** for dongle functionality. ZMK does not provide pre-built dongle shields.

### Why Custom Shields Are Required

1. **No Official Dongle Shields:**
   - ZMK repository includes keyboard shields (e.g., `corne_left`, `corne_right`)
   - **No built-in dongle variants** (e.g., no `corne_dongle` in official ZMK)
   - Community repositories demonstrate custom implementations

2. **Hardware-Specific Configuration:**
   - Dongle overlay must match your keyboard's matrix transform
   - Pin mappings differ between boards (Pro Micro vs XIAO vs nice!nano)
   - Display configuration varies (OLED type, size, interface)

3. **Role Configuration:**
   - `Kconfig.defconfig` sets central role and connection limits
   - Cannot be achieved with standard keyboard shields

### What You Must Create

**Minimum Files (3):**
1. **Kconfig.shield** - Declares dongle shield to build system
2. **Kconfig.defconfig** - Configures BLE central role and connection settings
3. **<shield>_dongle_xiao.overlay** - Devicetree hardware definition

**Recommended Additional Files (2):**
4. **<shield>_dongle_xiao.conf** - Runtime configuration overrides
5. **<shield>_dongle_xiao.keymap** - Optional keymap for macros on dongle (advanced)

### Can You Reuse Existing Shield Content?

**Yes, with modifications:**

1. **Matrix Transform:** Copy directly from existing keyboard shield
   ```dts
   // From splitkb_aurora_corne_left.overlay
   default_transform: keymap_transform_0 {
       compatible = "zmk,matrix-transform";
       columns = <12>;
       rows = <4>;
       map = <...>;  // Copy entire map
   };
   ```

2. **Physical Layout:** Reference if available
   ```dts
   chosen {
       zmk,physical-layout = &splitkb_aurora_corne_layout;
   };
   ```

3. **Mock KSCAN:** Standard template (always same for dongles)
   ```dts
   kscan0: kscan {
       compatible = "zmk,kscan-mock";
       columns = <0>;
       rows = <0>;
       events = <0>;
   };
   ```

### Alternative Approach: Overlay Includes

**Advanced technique to reduce duplication:**

**File:** `splitkb_aurora_corne_common.dtsi`
```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;
        map = <
            // Full matrix definition
        >;
    };
};
```

**File:** `splitkb_aurora_corne_dongle_xiao.overlay`
```dts
#include "splitkb_aurora_corne_common.dtsi"

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };
};

&xiao_i2c {
    status = "okay";
    // OLED config...
};
```

**Benefits:**
- Maintains single source of truth for matrix transform
- Easier to update if layout changes
- Used by several community repositories

---

## 6. Converting Peripherals to Connect to Dongle

### Architecture Change

**Before (Dongleless):**
- Left half: Central role (runs keymap logic)
- Right half: Peripheral role (sends events to left)
- Left half connects to host

**After (With Dongle):**
- Dongle: Central role (runs keymap logic)
- Left half: Peripheral role (sends events to dongle)
- Right half: Peripheral role (sends events to dongle)
- Dongle connects to host

### Step-by-Step Conversion Process

#### Step 1: Flash Settings Reset (CRITICAL)

**Why:** Clears existing pairing bonds between left/right halves. Without this, devices may not pair correctly.

**Process:**

1. **Build settings reset firmware:**
   Add to `build.yaml`:
   ```yaml
   - board: nice_nano@2.0.0
     shield: settings_reset
   - board: seeeduino_xiao_ble
     shield: settings_reset
   ```

2. **Flash sequence:**
   ```
   a. Flash settings_reset to left nice!nano
   b. Flash settings_reset to right nice!nano
   c. Flash settings_reset to XIAO BLE dongle
   ```

3. **Important:** Do not power multiple devices simultaneously during reset

**Settings reset disables Bluetooth** temporarily. You won't see devices in BT lists until firmware is flashed.

#### Step 2: Create Peripheral Shield Definitions

**Option A: Dedicated Peripheral Shields (Recommended)**

**Benefits:**
- Cleaner separation of concerns
- Peripheral-specific optimizations (sleep settings, lower TX power)
- Easier to maintain

**File:** `splitkb_aurora_corne_left_peripheral.overlay`
```dts
// Copy entire content from splitkb_aurora_corne_left.overlay
// No modifications needed - hardware is identical
```

**File:** `splitkb_aurora_corne_left_peripheral.conf`
```conf
# Enhanced sleep settings for peripherals
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000  # 15 minutes

# Power optimization
CONFIG_BT_CTLR_TX_PWR_0=y  # 0dBm (vs +8dBm) - adequate for dongle proximity

# Optional: Faster connection intervals for lower latency
# CONFIG_BT_PERIPHERAL_PREF_MIN_INT=6
# CONFIG_BT_PERIPHERAL_PREF_MAX_INT=12
```

**File:** `splitkb_aurora_corne_right_peripheral.overlay`
```dts
// Copy entire content from splitkb_aurora_corne_right.overlay
```

**File:** `splitkb_aurora_corne_right_peripheral.conf`
```conf
# Same as left_peripheral.conf
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000
CONFIG_BT_CTLR_TX_PWR_0=y
```

**Add to Kconfig.shield:**
```kconfig
config SHIELD_SPLITKB_AURORA_CORNE_LEFT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_left_peripheral)

config SHIELD_SPLITKB_AURORA_CORNE_RIGHT_PERIPHERAL
    def_bool $(shields_list_contains,splitkb_aurora_corne_right_peripheral)
```

**Option B: Reuse Existing Shields with Cmake Args**

**Benefits:**
- No new files required
- Simpler initial setup

**Drawback:**
- Cannot customize peripheral-specific settings easily
- Cmake args scattered in build.yaml

**Build configuration:**
```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

**Critical cmake-args:**
- `-DCONFIG_ZMK_SPLIT=y` - Enable split mode
- `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` - **Device is peripheral (not central)**

#### Step 3: Update Build Configuration

**File:** `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`

**Current Configuration:**
```yaml
include:
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
```

**Updated Configuration (With Dongle):**
```yaml
include:
  # Dongle (Central)
  - board: seeeduino_xiao_ble
    shield: splitkb_aurora_corne_dongle_xiao
    cmake-args: -DCONFIG_ZMK_KEYBOARD_NAME="Aurora Corne" -DCONFIG_ZMK_STUDIO=y
    artifact-name: aurora_corne_dongle_xiao

  # Left Peripheral
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: aurora_corne_left_peripheral

  # Right Peripheral
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: aurora_corne_right_peripheral

  # Settings reset utilities
  - board: nice_nano@2.0.0
    shield: settings_reset
    artifact-name: settings_reset_nice_nano

  - board: seeeduino_xiao_ble
    shield: settings_reset
    artifact-name: settings_reset_xiao_ble
```

**Artifact Naming:**
- Helps identify firmware files in GitHub Actions artifacts
- Pattern: `<keyboard>_<variant>_<board>.uf2`

#### Step 4: Flash Firmware in Correct Order

**CRITICAL ORDER:**

1. ✅ **Settings reset** already flashed (Step 1)

2. **Flash dongle firmware:**
   - File: `aurora_corne_dongle_xiao-seeeduino_xiao_ble-zmk.uf2`
   - Put XIAO BLE in bootloader mode (double-tap reset)
   - Copy .uf2 to XIAO-SENSE drive

3. **Flash left peripheral:**
   - File: `aurora_corne_left_peripheral-nice_nano_v2-zmk.uf2`
   - Put left nice!nano in bootloader mode
   - Copy .uf2 to NICENANO drive

4. **Flash right peripheral:**
   - File: `aurora_corne_right_peripheral-nice_nano_v2-zmk.uf2`
   - Put right nice!nano in bootloader mode
   - Copy .uf2 to NICENANO drive

5. **Wait for pairing:**
   - Plug in dongle to USB
   - Power on both halves (or press reset)
   - Wait 10-15 seconds for auto-pairing
   - Press keys on both halves to verify connectivity

**Troubleshooting pairing failures:**
- Simultaneously reset dongle and both halves (ground reset pins or press reset buttons)
- Ensure devices are within 1 meter during initial pairing
- Check dongle LED indicators (if configured)

#### Step 5: Verify Peripheral Behavior

**Expected Peripheral Behavior:**

✅ **When plugged into USB:**
- Device powers on
- **No USB keyboard functionality** (not recognized as HID device)
- Cannot type or send keypresses to host
- May show as USB device but not as keyboard

✅ **When powered by battery:**
- Scans keyboard matrix normally
- Sends key events to dongle via BLE
- Display shows status (if configured)

❌ **What peripherals CANNOT do:**
- Run keymap logic (layer switching, macros, hold-tap behaviors)
- Connect to host devices directly
- Act as standalone keyboards

### Automatic Pairing Process

**How Peripherals Find the Dongle:**

1. **Dongle broadcasts** as BLE central with specific identifier
2. **Peripherals scan** for central matching their configuration
3. **Pairing happens automatically** on first power-up (no button presses needed)
4. **Bond is stored** in persistent storage (survives power cycles)

**Pairing Identifiers:**

Peripherals and dongle must have matching keyboard identity. This is configured via:

- `CONFIG_ZMK_KEYBOARD_NAME` in Kconfig.defconfig
- Must be **identical** across dongle and peripherals
- Max 16 characters

**Example:**
```kconfig
# In Kconfig.defconfig for dongle AND peripheral shields
config ZMK_KEYBOARD_NAME
    default "Aurora Corne"
```

### Connection Topology After Conversion

```
┌──────────────────┐
│  XIAO BLE Dongle │ (Central Role)
│  "Aurora Corne"  │
└────────┬─────────┘
         │
         │ BLE Connection 1
         ├──────────────────────► ┌─────────────────────┐
         │                        │ nice!nano v2 - Left │ (Peripheral)
         │                        │ + nice!view         │
         │                        └─────────────────────┘
         │
         │ BLE Connection 2
         └──────────────────────► ┌─────────────────────┐
                                  │ nice!nano v2 - Right│ (Peripheral)
                                  │ + nice!view         │
                                  └─────────────────────┘
         │
         │ USB Connection
         ▼
┌──────────────────┐
│   Host Device    │
│ (Computer/Phone) │
└──────────────────┘
```

---

## 7. Common Pitfalls and Troubleshooting

### Pitfall 1: Skipping Settings Reset

**Symptom:** Peripherals won't pair with dongle, or pairing is unstable

**Cause:** Old pairing bonds from dongleless setup still exist

**Solution:**
1. Flash `settings_reset` to **all three devices** (dongle + left + right)
2. Power off all devices
3. Flash actual firmware in order: dongle → left → right
4. Power on and wait for auto-pairing

**Prevention:** Always include settings_reset in build.yaml for easy access

### Pitfall 2: Incorrect BT_MAX_CONN Value

**Symptom:** Build fails or peripherals can't connect

**Cause:** `BT_MAX_CONN` doesn't account for peripherals + host profiles

**Solution:**
```kconfig
# Formula: peripherals + desired_host_profiles
CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2  # left + right
CONFIG_BT_MAX_CONN=7  # 2 + 5 (default profiles)
CONFIG_BT_MAX_PAIRED=7  # Must match BT_MAX_CONN
```

**Prevention:** Always use formula: `BT_MAX_CONN = peripherals + 5`

### Pitfall 3: Mismatched ZMK_KEYBOARD_NAME

**Symptom:** Peripherals never discover dongle

**Cause:** Keyboard name differs between dongle and peripherals

**Solution:** Ensure identical names in all Kconfig.defconfig files:
```kconfig
# Must be EXACTLY the same across all devices
config ZMK_KEYBOARD_NAME
    default "Aurora Corne"
```

**Prevention:** Define name once in shared `.dtsi` or carefully copy-paste

### Pitfall 4: Dongle Placement on Desktop

**Symptom:** Unstable connection, frequent disconnects, high latency

**Cause:** Dongle plugged into rear USB port, metal case blocks signal

**Solution:**
- Use **front USB port** or USB extension cable
- Keep dongle within 1 meter of keyboard halves
- Avoid placing dongle behind metal enclosures

**Prevention:** Test with phone/tablet first (BT is more stable on mobile)

### Pitfall 5: Wrong Bluetooth Profile Selected

**Symptom:** Keyboard doesn't respond, but appears paired

**Cause:** Dongle is on profile 2, but host is bonded to profile 1

**Solution:**
1. On dongle, cycle through BT profiles using keymap (BT_SEL commands)
2. Check which profile shows "connected" on host
3. Use `BT_CLR` on empty profiles to avoid confusion

**Prevention:** Use consistent profile slots (e.g., profile 0 = main computer)

### Pitfall 6: Peripheral Cmake Args Missing

**Symptom:** Left half acts as central, tries to connect to host instead of dongle

**Cause:** Forgot `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` in build.yaml

**Solution:**
```yaml
# REQUIRED for peripherals
cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

**Prevention:** Use dedicated peripheral shields with role baked into Kconfig.defconfig

### Pitfall 7: Flashing Order Confusion

**Symptom:** Pairing doesn't happen automatically

**Cause:** Flashed peripherals before dongle, peripherals already scanned and gave up

**Solution:** Always flash in order:
1. Settings reset all
2. Dongle firmware
3. Left peripheral firmware
4. Right peripheral firmware

**Prevention:** Document flashing order in repository README

### Pitfall 8: Zephyr 4.1 NFC Pin Conflict (XIAO BLE)

**Symptom:** Build fails with cryptic errors about pin conflicts

**Cause:** XIAO BLE uses nRF52840 NFC pins, Zephyr 4.1 requires explicit disable

**Solution:** Add to dongle overlay:
```dts
&nfct {
    status = "disabled";
};
```

**Prevention:** Check ZMK blog post "Zephyr 4.1 Update" for breaking changes

### Pitfall 9: Display Work Queue Not Dedicated

**Symptom:** Keyboard feels laggy, occasional missed keypresses

**Cause:** Display updates block key scanning on dongle

**Solution:**
```kconfig
CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y
```

**Prevention:** Always enable for dongles with displays (included in example configs)

### Pitfall 10: Forgetting Settings Reset Targets in Build

**Symptom:** No easy way to generate settings_reset firmware

**Cause:** Didn't add settings_reset to build.yaml

**Solution:**
```yaml
- board: nice_nano@2.0.0
  shield: settings_reset
- board: seeeduino_xiao_ble
  shield: settings_reset
```

**Prevention:** Include in initial build.yaml setup

### Diagnostic Checklist

**When pairing fails:**

- [ ] Settings reset flashed to all devices?
- [ ] `ZMK_KEYBOARD_NAME` identical on all devices?
- [ ] `BT_MAX_CONN = peripherals + 5`?
- [ ] `BT_MAX_PAIRED = BT_MAX_CONN`?
- [ ] Peripheral cmake-args include `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`?
- [ ] Dongle Kconfig has `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`?
- [ ] Devices within 1 meter during pairing?
- [ ] Reset buttons pressed simultaneously?

**When connection is unstable:**

- [ ] Dongle in front USB port (not rear)?
- [ ] Metal case blocking signal?
- [ ] Correct BT profile selected on dongle?
- [ ] Host device removed old pairing bonds?

---

## 8. ZMK Version and Compatibility Notes

### Current ZMK Version Status (as of 2026-01-04)

**Latest Stable Release:** ZMK v0.3
**Development Branch:** Uses Zephyr 4.1 (as of December 2025)

**Your Repository Status:**
Based on commits "remove deprecated CONFIG_WS2812_STRIP for Zephyr 4.1 compatibility" and "update configuration for zmk 4.1 upgrade", you are on **Zephyr 4.1**.

### Zephyr Version History and Breaking Changes

| Zephyr Version | ZMK Adoption | Key Changes Affecting Dongles |
|----------------|--------------|-------------------------------|
| 3.0 | April 2022 | XIAO BLE officially integrated, improved BLE stack |
| 3.5 | February 2024 | ZMK stuck here for 1.5 years due to Zephyr 3.6 breaking changes |
| 4.0 | Skipped by ZMK | Breaking API changes incompatible with ZMK |
| 4.1 | December 2025 | **Current ZMK main branch**, large leap forward |

### Zephyr 4.1 Compatibility Requirements

#### 1. NFC Pin Handling (XIAO BLE)

**Issue:** nRF52840 boards (including XIAO BLE) use pins that conflict with NFC peripheral

**Solution:** Explicitly disable NFC in overlay:
```dts
&nfct {
    status = "disabled";
};
```

**Applies to:** Any board using P0.09 or P0.10 (common on XIAO)

#### 2. WS2812 LED Strip Configuration

**Issue:** `CONFIG_WS2812_STRIP` deprecated in Zephyr 4.1

**Solution:** Remove deprecated config (you've already done this per commit history)

**New approach:** Uses devicetree-based LED strip configuration

#### 3. UART Async Mode Limitation

**Issue:** DMA-based UART broken on Zephyr 3.5-4.1 for nRF52

**Impact:** Affects UART-based communication (not relevant for BLE dongles)

**Workaround:** None needed for standard BLE dongle setup

### Version Pinning for Stability

**Official Recommendation:** Pin to specific ZMK commit for production use

**How to pin in config/west.yml:**
```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main  # Change to specific commit hash for pinning
      import: app/west.yml
  self:
    path: config
```

**Example pinning to specific commit:**
```yaml
revision: a8b88f68bc1e46d5de70ab1a4c224e26f909bd97  # Replace with desired commit
```

**Benefits:**
- Reproducible builds
- Protection from breaking changes
- Stable feature set

**When to update:**
- New features needed
- Critical bug fixes
- After testing on development branch

### Board Support Requirements

**XIAO BLE (seeeduino_xiao_ble):**

| Feature | ZMK Version Required | Notes |
|---------|---------------------|-------|
| Basic support | Zephyr 3.0+ | Fully integrated |
| Dongle mode | Any ZMK version | Community-proven since 2022 |
| nice!view SPI | Zephyr 3.0+ | Requires SPI configuration |
| I2C OLED | Zephyr 3.0+ | Works with SSD1306, SH1106, SH1107 |
| Zephyr 4.1 | Latest ZMK main | Requires NFC disable |

**nice!nano v2:**

| Feature | ZMK Version Required | Notes |
|---------|---------------------|-------|
| Peripheral mode | Any ZMK version | Standard functionality |
| nice!view | Zephyr 2.5+ | Native support |
| Peripheral role | Any ZMK version | Use cmake args or Kconfig |

### Known Issues and Limitations

#### Issue 1: Battery Level Reporting with 3+ Parts

**Symptom:** Compile error when using dongle + split keyboard (3 parts total)

**Status:** Resolved in recent ZMK commits

**Workaround (if needed):**
```conf
# Disable battery level proxy if hitting compilation errors
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=n
```

**Modern solution:** Enable battery fetching instead:
```conf
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
```

#### Issue 2: Async UART on nRF52

**Symptom:** UART-based communication unstable on Zephyr 3.5-4.1

**Impact:** None for BLE dongles (only affects wired split keyboards)

**Workaround:** Use synchronous UART mode (default)

#### Issue 3: Studio Support Requirement

**Symptom:** ZMK Studio features don't work on older firmware

**Solution:** Add to build cmake-args:
```yaml
cmake-args: -DCONFIG_ZMK_STUDIO=y
```

**Compatibility:** Requires physical layout definitions in overlay

### Recommended Configuration for Maximum Compatibility

**For production use on Zephyr 4.1:**

**Dongle overlay additions:**
```dts
// Add near top of file for nRF52840 boards
&nfct {
    status = "disabled";
};
```

**Dongle .conf safety settings:**
```conf
# Proven stable settings for Zephyr 4.1
CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_BT_MAX_CONN=7
CONFIG_BT_MAX_PAIRED=7
CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_BT_BAS=n

# Optional: Enable Studio support
CONFIG_ZMK_STUDIO=y
```

**Peripheral .conf optimizations:**
```conf
# Extended battery life settings
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000  # 15 min
CONFIG_BT_CTLR_TX_PWR_0=y  # Lower power adequate for dongle proximity

# Optional: Faster reconnection
CONFIG_BT_PERIPHERAL_PREF_MIN_INT=6
CONFIG_BT_PERIPHERAL_PREF_MAX_INT=12
```

---

## 9. Implementation Roadmap for Your Setup

### Current State Analysis

**Repository:** `/Users/nerdo/personal/zmk-aurora-corne`

**Current Hardware:**
- Left: nice!nano v2 with nice!view
- Right: nice!nano v2 with nice!view
- Mode: Dongleless (left is central)

**Current Firmware:**
- Zephyr: 4.1 (confirmed by recent commits)
- RGB underglow: Enabled
- Display: Enabled (inverted)
- NKRO: Enabled
- BT TX power: +8dBm

**Current Build Targets:**
```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_right nice_view_adapter nice_view
```

### Recommended Implementation Plan

#### Phase 1: Repository Structure Setup

**Task 1.1: Create shield directory structure**
```bash
mkdir -p boards/shields/splitkb_aurora_corne
```

**Task 1.2: Create Kconfig.shield**

File: `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/Kconfig.shield`

Content: [See Section 1 - Kconfig.shield]

**Task 1.3: Create Kconfig.defconfig**

File: `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/Kconfig.defconfig`

Content: [See Section 1 - Kconfig.defconfig]

Note: Add sections for both dongle and peripheral shields

**Task 1.4: Locate existing matrix transform**

Check if Aurora Corne already has a matrix transform defined:
```bash
# Search for existing overlay
cat /Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne.overlay
```

Copy the matrix transform from this file (if it exists) or from the official ZMK Aurora Corne shield.

#### Phase 2: Dongle Shield Creation

**Task 2.1: Create dongle overlay**

File: `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.overlay`

Key requirements:
- Copy matrix transform from existing shield
- Add mock kscan (0 columns, 0 rows)
- Disable NFC for Zephyr 4.1 compatibility: `&nfct { status = "disabled"; };`
- Optional: Add OLED configuration on I2C

**Task 2.2: Create dongle .conf**

File: `/Users/nerdo/personal/zmk-aurora-corne/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.conf`

Settings: [See Section 1 - dongle .conf]

Consider including:
- Display settings (match your current setup)
- Battery reporting from peripherals
- Studio support

#### Phase 3: Peripheral Shield Creation

**Decision point:** Dedicated peripheral shields vs cmake args

**Recommended:** Create dedicated peripheral shields for cleaner configuration

**Task 3.1: Create peripheral overlays**

Files:
- `splitkb_aurora_corne_left_peripheral.overlay`
- `splitkb_aurora_corne_right_peripheral.overlay`

Content: Copy from existing Aurora Corne left/right overlays (hardware identical)

**Task 3.2: Create peripheral .conf files**

Files:
- `splitkb_aurora_corne_left_peripheral.conf`
- `splitkb_aurora_corne_right_peripheral.conf`

Settings:
```conf
# Extended sleep for battery life
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000

# Lower TX power (dongle is nearby)
CONFIG_BT_CTLR_TX_PWR_0=y

# Keep your existing settings
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_ZMK_RGB_UNDERGLOW_ON_START=n
CONFIG_ZMK_DISPLAY_INVERT=y
```

**Task 3.3: Update Kconfig files**

Add peripheral shield entries to `Kconfig.shield` and `Kconfig.defconfig`

#### Phase 4: Build Configuration

**Task 4.1: Update build.yaml**

File: `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`

Add new targets:
```yaml
include:
  # Dongle
  - board: seeeduino_xiao_ble
    shield: splitkb_aurora_corne_dongle_xiao
    cmake-args: -DCONFIG_ZMK_KEYBOARD_NAME="dCorne mini" -DCONFIG_ZMK_STUDIO=y
    artifact-name: dcorne_dongle_xiao

  # Peripherals
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: dcorne_left_peripheral

  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: dcorne_right_peripheral

  # Settings reset
  - board: nice_nano@2.0.0
    shield: settings_reset
    artifact-name: settings_reset_nice_nano

  - board: seeeduino_xiao_ble
    shield: settings_reset
    artifact-name: settings_reset_xiao_ble

  # Keep existing dongleless builds for fallback
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    artifact-name: dcorne_left_dongleless

  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    artifact-name: dcorne_right_dongleless
```

Note: Keeping dongleless builds provides easy rollback option

**Task 4.2: Commit and push**

Trigger GitHub Actions build to verify compilation

#### Phase 5: Hardware Preparation

**Task 5.1: Acquire XIAO BLE**

**Part:** Seeed Studio XIAO nRF52840 (also called XIAO BLE)
**Purchase:** [Seeed Studio](https://www.seeedstudio.com/Seeed-XIAO-BLE-nRF52840-p-5201.html) or Amazon

**Optional:** OLED display (SSD1306 128x64, I2C)

**Task 5.2: Test XIAO BLE**

Before building dongle enclosure:
1. Flash settings_reset
2. Flash dongle firmware
3. Verify USB connection
4. Test pairing with keyboards

#### Phase 6: Flashing and Testing

**Task 6.1: Download firmware from GitHub Actions**

Artifacts:
- `settings_reset_nice_nano.uf2`
- `settings_reset_xiao_ble.uf2`
- `dcorne_dongle_xiao-seeeduino_xiao_ble-zmk.uf2`
- `dcorne_left_peripheral-nice_nano_v2-zmk.uf2`
- `dcorne_right_peripheral-nice_nano_v2-zmk.uf2`

**Task 6.2: Flash settings reset (CRITICAL)**

Order:
1. Left nice!nano: `settings_reset_nice_nano.uf2`
2. Right nice!nano: `settings_reset_nice_nano.uf2`
3. XIAO BLE: `settings_reset_xiao_ble.uf2`

**Task 6.3: Flash firmware**

Order:
1. XIAO BLE dongle: `dcorne_dongle_xiao-seeeduino_xiao_ble-zmk.uf2`
2. Left peripheral: `dcorne_left_peripheral-nice_nano_v2-zmk.uf2`
3. Right peripheral: `dcorne_right_peripheral-nice_nano_v2-zmk.uf2`

**Task 6.4: Initial pairing**

1. Plug dongle into USB
2. Power on both halves (or press reset)
3. Wait 15 seconds
4. Press keys on both halves
5. Verify keystrokes on host

**Task 6.5: Verify functionality**

Checklist:
- [ ] All keys responsive on both halves
- [ ] Layer switching works
- [ ] nice!view displays update
- [ ] RGB underglow functions
- [ ] BT profile switching works (if configured)
- [ ] Dongle display shows status (if configured)

#### Phase 7: Optimization and Troubleshooting

**Task 7.1: Monitor battery life**

Track peripheral battery consumption over 1-2 weeks:
- Expected: 5-8 months with moderate use
- Compare to previous dongleless setup (2-4 weeks)

**Task 7.2: Fine-tune settings**

Based on testing results:
- Adjust `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` if needed
- Tweak BT TX power if connection unstable
- Optimize display refresh if laggy

**Task 7.3: Document setup**

Create README in repository with:
- Flashing instructions
- Troubleshooting guide
- Hardware requirements
- Known issues

### Rollback Plan

**If dongle setup fails:**

1. **Flash dongleless firmware** (kept in build.yaml):
   - `dcorne_left_dongleless-nice_nano_v2-zmk.uf2`
   - `dcorne_right_dongleless-nice_nano_v2-zmk.uf2`

2. **Reset settings** on both nice!nanos

3. **Resume normal operation** in dongleless mode

**No hardware damage possible** - all changes are firmware-based

### Estimated Timeline

| Phase | Duration | Blockers |
|-------|----------|----------|
| 1. Repository setup | 1-2 hours | Understanding matrix transform structure |
| 2. Dongle shield | 1 hour | Overlay syntax |
| 3. Peripheral shields | 30 min | Copy-paste from existing |
| 4. Build config | 30 min | YAML syntax |
| 5. Hardware prep | 1-2 weeks | XIAO BLE shipping time |
| 6. Flashing/testing | 1 hour | Pairing troubleshooting |
| 7. Optimization | Ongoing | Real-world usage data |

**Total active work:** 4-5 hours
**Total elapsed time:** 1-2 weeks (hardware acquisition)

---

## 10. Additional Resources

### Official ZMK Documentation

- **Keyboard Dongle Guide:** https://zmk.dev/docs/development/hardware-integration/dongle
- **New Shield Creation:** https://zmk.dev/docs/development/hardware-integration/new-shield
- **Split Configuration:** https://zmk.dev/docs/config/split
- **Bluetooth Configuration:** https://zmk.dev/docs/config/bluetooth
- **Connection Troubleshooting:** https://zmk.dev/docs/troubleshooting/connection-issues
- **Zephyr 4.1 Update:** https://zmk.dev/blog/2025/12/09/zephyr-4-1

### Community Examples (Tested Configurations)

1. **redmasters/zmk-config-dongle**
   https://github.com/redmasters/zmk-config-dongle
   Comprehensive Corne dongle setup with XIAO BLE and Pro Micro variants

2. **DarrenVictoriano/zmk-config**
   https://github.com/DarrenVictoriano/zmk-config
   nice!nano + nice!view + XIAO BLE dongle with detailed flashing instructions

3. **mctechnology17/zmk-config**
   https://github.com/mctechnology17/zmk-config
   Multi-keyboard support (Corne, Sofle, Lily58) with OLED adapters

4. **aroum/zmk-enki42-dongle**
   https://github.com/aroum/zmk-enki42-dongle
   Corne-like keyboard with dongle, shows alternative approaches

5. **beekeeb.com Guide**
   https://beekeeb.com/how-to-add-dongle-and-prospector-support-to-hshs52-hshs46/
   Step-by-step tutorial for adding dongle support

### Third-Party Documentation

- **SliceMK ZMK Dongle Guide:** https://docs.slicemk.com/firmware/zmk/wireless/dongle/
- **SliceMK Bluetooth Pairing:** https://docs.slicemk.com/firmware/zmk/wireless/bluetooth/
- **SliceMK Bond Reset:** https://docs.slicemk.com/firmware/zmk/wireless/nvsclear/

### Hardware Datasheets

- **XIAO nRF52840 (BLE):** https://wiki.seeedstudio.com/XIAO_BLE/
- **nRF52840 SoC:** https://www.nordicsemi.com/Products/nRF52840
- **nice!nano v2:** https://nicekeyboards.com/docs/nice-nano/

### ZMK Source Code References

- **Split BLE Kconfig:** https://github.com/zmkfirmware/zmk/blob/main/app/src/split/bluetooth/Kconfig
- **Dongle Documentation Source:** https://github.com/zmkfirmware/zmk/blob/main/docs/docs/development/hardware-integration/dongle.md
- **System Configuration:** https://github.com/zmkfirmware/zmk/blob/main/app/Kconfig

### Tools and Utilities

- **ZMK Studio:** https://zmk.dev/docs/features/studio (Live configuration tool)
- **Keymap Editor:** https://nickcoutsos.github.io/keymap-editor/ (Visual keymap editor)
- **GitHub Actions for ZMK:** Automatic firmware builds on push

---

## Appendix A: Complete File Examples

### Example 1: Minimal Dongle Setup (No Display)

**File:** `splitkb_aurora_corne_dongle_xiao.overlay`
```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10) RC(0,11)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10) RC(1,11)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10) RC(2,11)
                                    RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
        >;
    };
};

&nfct {
    status = "disabled";
};
```

### Example 2: Dongle with SSD1306 128x64 OLED

**File:** `splitkb_aurora_corne_dongle_xiao.overlay`
```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10) RC(0,11)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10) RC(1,11)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10) RC(2,11)
                                    RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
        >;
    };
};

&xiao_i2c {
    status = "okay";

    oled: ssd1306@3c {
        compatible = "solomon,ssd1306fb";
        reg = <0x3c>;
        width = <128>;
        height = <64>;
        segment-offset = <0>;
        page-offset = <0>;
        display-offset = <0>;
        multiplex-ratio = <63>;
        segment-remap;
        com-invdir;
        inversion-on;
        prechargep = <0x22>;
    };
};

&nfct {
    status = "disabled";
};
```

### Example 3: Dongle with nice!view (SPI)

**File:** `splitkb_aurora_corne_dongle_xiao.overlay`
```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <12>;
        rows = <4>;
        map = <
            RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10) RC(0,11)
            RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10) RC(1,11)
            RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10) RC(2,11)
                                    RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
        >;
    };
};

&xiao_i2c {
    status = "disabled";
};

&xiao_spi {
    status = "okay";
    cs-gpios = <&xiao_d 6 GPIO_ACTIVE_HIGH>;

    nice_view: ls011b7dh01@0 {
        compatible = "sharp,ls011b7dh01";
        reg = <0>;
        spi-max-frequency = <1000000>;
        width = <160>;
        height = <68>;
    };
};

&nfct {
    status = "disabled";
};
```

### Example 4: Complete build.yaml for Dongle Setup

**File:** `build.yaml`
```yaml
include:
  # ===== DONGLE SETUP =====

  # Dongle (Central)
  - board: seeeduino_xiao_ble
    shield: splitkb_aurora_corne_dongle_xiao
    cmake-args: -DCONFIG_ZMK_KEYBOARD_NAME="dCorne mini" -DCONFIG_ZMK_STUDIO=y
    artifact-name: dcorne_dongle_xiao

  # Left Peripheral
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: dcorne_left_peripheral

  # Right Peripheral
  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right_peripheral nice_view_adapter nice_view
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: dcorne_right_peripheral

  # ===== SETTINGS RESET =====

  - board: seeeduino_xiao_ble
    shield: settings_reset
    artifact-name: settings_reset_xiao_ble

  - board: nice_nano@2.0.0
    shield: settings_reset
    artifact-name: settings_reset_nice_nano

  # ===== DONGLELESS FALLBACK =====

  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_left nice_view_adapter nice_view
    artifact-name: dcorne_left_dongleless

  - board: nice_nano@2.0.0
    shield: splitkb_aurora_corne_right nice_view_adapter nice_view
    artifact-name: dcorne_right_dongleless
```

---

## Appendix B: Quick Reference Commands

### Flashing Workflow (Full Reset)

```bash
# 1. Settings reset - Left nice!nano
# Put board in bootloader mode (double-tap reset)
cp settings_reset_nice_nano.uf2 /Volumes/NICENANO/

# 2. Settings reset - Right nice!nano
# Put board in bootloader mode
cp settings_reset_nice_nano.uf2 /Volumes/NICENANO/

# 3. Settings reset - XIAO BLE
# Put board in bootloader mode (double-tap reset)
cp settings_reset_xiao_ble.uf2 /Volumes/XIAO-SENSE/

# 4. Flash dongle firmware
cp dcorne_dongle_xiao-seeeduino_xiao_ble-zmk.uf2 /Volumes/XIAO-SENSE/

# 5. Flash left peripheral
cp dcorne_left_peripheral-nice_nano_v2-zmk.uf2 /Volumes/NICENANO/

# 6. Flash right peripheral
cp dcorne_right_peripheral-nice_nano_v2-zmk.uf2 /Volumes/NICENANO/

# 7. Plug dongle into USB, power on halves, wait 15 seconds
```

### Troubleshooting Commands

```bash
# Check ZMK version in your repository
cat config/west.yml | grep revision

# Search for existing matrix transform
grep -r "matrix-transform" config/

# Verify overlay syntax (requires devicetree compiler)
dtc -I dts -O dtb boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_dongle_xiao.overlay

# List connected BLE devices (macOS)
system_profiler SPBluetoothDataType

# Monitor USB devices (macOS)
system_profiler SPUSBDataType | grep -A 5 "XIAO"
```

---

## Conclusion

This research provides a complete guide to implementing ZMK dongle functionality for your Corne keyboard with XIAO BLE. Key takeaways:

1. **Custom shield creation is required** - no pre-built dongle shields exist
2. **XIAO BLE is fully supported** and widely used in community configurations
3. **Settings reset is mandatory** before initial pairing
4. **Peripheral conversion is straightforward** using cmake args or dedicated shields
5. **Zephyr 4.1 compatibility** requires NFC disable on XIAO BLE
6. **Battery life improvements are significant** (5-8 months vs 2-4 weeks for peripherals)

The implementation roadmap in Section 9 provides step-by-step guidance tailored to your specific setup. Community examples demonstrate proven configurations you can adapt.

**Next Steps:**
1. Review Section 9 implementation plan
2. Create shield files in repository
3. Update build.yaml configuration
4. Order XIAO BLE hardware
5. Flash and test dongle setup

**Estimated effort:** 4-5 hours active work, 1-2 weeks elapsed time (hardware shipping)

---

**Research Sources:**

- [ZMK Keyboard Dongle](https://zmk.dev/docs/development/hardware-integration/dongle)
- [ZMK Split Keyboards](https://zmk.dev/docs/features/split-keyboards)
- [ZMK New Shield Guide](https://zmk.dev/docs/development/hardware-integration/new-shield)
- [ZMK Split Configuration](https://zmk.dev/docs/config/split)
- [ZMK Bluetooth Configuration](https://zmk.dev/docs/config/bluetooth)
- [ZMK Connection Troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues)
- [ZMK Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1)
- [SliceMK Dongle Guide](https://docs.slicemk.com/firmware/zmk/wireless/dongle/)
- [beekeeb Dongle Tutorial](https://beekeeb.com/how-to-add-dongle-and-prospector-support-to-hshs52-hshs46/)
- [redmasters/zmk-config-dongle](https://github.com/redmasters/zmk-config-dongle)
- [DarrenVictoriano/zmk-config](https://github.com/DarrenVictoriano/zmk-config)
- [mctechnology17/zmk-config](https://github.com/mctechnology17/zmk-config)
- [aroum/zmk-enki42-dongle](https://github.com/aroum/zmk-enki42-dongle)
