# ZMK Split BLE Dongle Pairing Analysis

**Date:** 2026-01-05
**Research Focus:** Why Aurora Corne keyboard halves fail to pair with custom XIAO BLE dongle
**Context:** User has Aurora Corne halves (splitkb_aurora_corne_left/right) and custom corne_dongle shield, pairing fails even after settings_reset

---

## Executive Summary

**Root Cause Identified:** The dongle's `Kconfig.defconfig` is **missing the critical `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` setting**, which tells the central how many peripherals to expect and allow connections from.

**Current Configuration:**
- Dongle `BT_MAX_CONN` = 5 (supports host connections)
- Dongle `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` = **NOT SET** (defaults to 1)
- Build config only provides peripheral configuration for **left half only**
- **Right half missing** from dongle-mode build entirely

**Required Fixes:**
1. Set `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` to `2` in dongle's Kconfig.defconfig
2. Adjust `BT_MAX_CONN` and `BT_MAX_PAIRED` to accommodate 2 peripherals + desired BT profiles
3. Add right half peripheral build configuration to build.yaml
4. Flash settings_reset to all three devices (dongle, left, right)
5. Flash new firmware to all three devices
6. Reset all three simultaneously to initiate pairing

**Secondary Issue:** `CONFIG_ZMK_KEYBOARD_NAME` mismatch ("Corne Dongle" vs "dCorne mini") is **not relevant** to split pairing - this only affects the BLE device name shown to host computers, not peripheral discovery.

---

## How ZMK Split BLE Pairing Works

### Discovery and Pairing Mechanism

ZMK uses an **internal pairing protocol** between central and peripherals:

1. **Central Advertisement:** When the central has an open slot for a peripheral, it advertises for connections using a ZMK-specific protocol (not visible to non-ZMK devices)

2. **Peripheral Response:** Any peripheral that has not yet bonded to a central will automatically pair to the advertising central

3. **Bond Storage:** Bonding information is stored with hardware addresses on both sides, similar to how Bluetooth profiles work between keyboard and host

4. **Security:** BLE connections use Elliptic Curve Diffie Hellman (ECDH) for key generation and establish long-term keys (LTK) for secure communication

**Key Point:** The pairing is based on **hardware addresses and ZMK's internal protocol**, NOT on matching `CONFIG_ZMK_KEYBOARD_NAME` values.

### Peripheral Slot Allocation

The central must be configured to expect the correct number of peripherals via `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS`:
- **Unibody keyboard (1 peripheral):** Set to `1`
- **Split keyboard with left/right halves (2 peripherals):** Set to `2`

If this value is not set or set incorrectly:
- Default is typically `1` (only one peripheral slot)
- Central will only accept connections from the first peripheral that pairs
- Second peripheral cannot pair because no slot is available

**Source:** [ZMK Split Keyboards Documentation](https://zmk.dev/docs/features/split-keyboards), [ZMK Keyboard Dongle Documentation](https://zmk.dev/docs/development/hardware-integration/dongle)

---

## Evidence from Current Configuration

### Dongle Kconfig.defconfig Analysis

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/corne_dongle/Kconfig.defconfig`

```kconfig
if SHIELD_CORNE_DONGLE

config ZMK_KEYBOARD_NAME
	default "Corne Dongle"

config ZMK_SPLIT
	default y

config ZMK_SPLIT_ROLE_CENTRAL
	default y

# Central connects to both halves
config BT_MAX_CONN
	default 5

config BT_MAX_PAIRED
	default 5

# Larger ACL buffer for split communication
config BT_BUF_ACL_RX_SIZE
	default 255

config BT_BUF_ACL_TX_SIZE
	default 255

endif
```

**Critical Missing Configuration:**
```kconfig
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
	default 2
```

**Comment Analysis:** Line 15 states "# Central connects to both halves" but the configuration doesn't actually enable this capability. This comment indicates the *intent* but the implementation is incomplete.

### Build Configuration Analysis

**File:** `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`

```yaml
# === KEYBOARD HALVES (DONGLE MODE) ===
# Both halves are peripherals, connecting to the dongle
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  cmake-args: -DEXTRA_CONF_FILE=../../config/splitkb_aurora_corne_left_peripheral.conf
  artifact-name: splitkb_aurora_corne_left_peripheral-nice_nano_v2-zmk

# === XIAO BLE DONGLE ===
# Named 'corne_dongle' to avoid prefix matching with keyboard config files
- board: xiao_ble
  shield: corne_dongle
```

**Missing:** Right half peripheral build configuration. Only the left half has a dongle-mode build entry.

### Required Right Half Configuration

**Missing from build.yaml:**
```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_right nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
  artifact-name: splitkb_aurora_corne_right_peripheral-nice_nano_v2-zmk
```

**Source:** [ZMK Keyboard Dongle Documentation - Build Configuration](https://zmk.dev/docs/development/hardware-integration/dongle)

---

## Bluetooth Connection Mathematics

### Formula for BT_MAX_CONN and BT_MAX_PAIRED

For split keyboards with dongles:

```
BT_MAX_CONN = BT_MAX_PAIRED = ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS + desired_bluetooth_profiles
```

**Default Bluetooth profiles:** 5 (allows connection to 5 different host devices)

### Current vs Required Configuration

**Current (Incorrect):**
```kconfig
config BT_MAX_CONN
	default 5

config BT_MAX_PAIRED
	default 5
```

This configuration:
- `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` defaults to 1 (implicit)
- Allocates 5 connections total
- Can theoretically support 1 peripheral + 4 host profiles
- **Problem:** With 2 keyboard halves, only the first to connect gets a slot

**Required (Correct):**
```kconfig
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
	default 2

config BT_MAX_CONN
	default 7

config BT_MAX_PAIRED
	default 7
```

This configuration:
- Explicitly sets 2 peripheral slots
- Allocates 7 connections total
- Supports 2 peripherals + 5 host profiles
- Both keyboard halves can connect simultaneously

**Alternative (Fewer Profiles):**
```kconfig
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
	default 2

config BT_MAX_CONN
	default 5

config BT_MAX_PAIRED
	default 5
```

This configuration:
- Explicitly sets 2 peripheral slots
- Allocates 5 connections total
- Supports 2 peripherals + 3 host profiles
- More memory-efficient if 5 host profiles aren't needed

**Source:** [ZMK Split Configuration Documentation](https://zmk.dev/docs/config/split)

---

## Configuration Name Analysis (Not the Issue)

### CONFIG_ZMK_KEYBOARD_NAME Role

**Current Values:**
- Dongle: `CONFIG_ZMK_KEYBOARD_NAME="Corne Dongle"`
- Keyboard halves: `CONFIG_ZMK_KEYBOARD_NAME="dCorne mini"`

**What This Controls:**
- Display name of device over USB
- Display name when advertising to host computers via BLE
- What appears in Bluetooth device lists on your computer/phone

**What This Does NOT Control:**
- Peripheral discovery between split keyboard parts
- Internal ZMK pairing protocol
- Which peripherals can connect to which centrals

### Why Names Don't Need to Match

ZMK's split keyboard pairing uses:
- **Hardware MAC addresses** for device identification
- **ZMK-specific BLE advertisement protocol** that's invisible to non-ZMK devices
- **Bond storage with hardware addresses** rather than string matching

The keyboard name is purely for human-readable identification when pairing to host devices (computers, tablets, phones), not for split keyboard internal communication.

**Evidence:** ZMK documentation states "For that side, the keyboard name is assigned and the central config is set. The peripheral side is not assigned a name." This confirms peripherals don't even need a name for split pairing to work.

**Source:** [ZMK System Configuration Documentation](https://zmk.dev/docs/config/system), [ZMK Split Keyboards Documentation](https://zmk.dev/docs/features/split-keyboards)

---

## Settings Reset and Pairing Sequence

### Why Settings Reset is Necessary

**Stored Bond Information:**
- Each device remembers previous pairings via stored bonds
- Hardware addresses are stored for bonded devices
- When configuration changes (e.g., changing which part is central), old bonds become invalid
- Old bonds prevent new pairing attempts

### Proper Settings Reset Procedure

1. **Flash settings_reset firmware** to all three devices:
   - Dongle (xiao_ble)
   - Left half (nice_nano)
   - Right half (nice_nano)

2. **Important:** Settings reset firmware has Bluetooth disabled to prevent auto-re-pairing during the reset process. Devices will not appear in Bluetooth lists until regular firmware is flashed.

3. **Flash regular firmware** to all three devices with corrected configuration

4. **Simultaneous reset:** Power on or reset all three devices at approximately the same time
   - Ground reset pins, or
   - Press reset buttons, or
   - Power switch on simultaneously

5. **Verification:** Check that both halves connect to dongle (usually indicated by LED status if configured)

**Why Simultaneous Reset Helps:** When all devices start advertising/scanning at the same time, the peripheral discovery window aligns, increasing likelihood of successful initial pairing.

**Source:** [ZMK Connection Issues Troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues)

---

## Comparison with Working Examples

### Example: mctechnology17/zmk-config

**Documented Configuration Pattern:**
- Tested with seeeduino_xiao_ble used as dongle
- Supports both left and right halves as peripherals
- Configuration files exist for:
  - `corne_dongle_pro_micro.conf`
  - `corne_dongle_xiao.conf`
- Works with multiple board types (nice!nano v2, puchi_ble_v1, xiao_ble)

**Key Lesson:** XIAO BLE is confirmed working as a dongle for Corne keyboards when properly configured.

**Source:** [mctechnology17/zmk-config GitHub Repository](https://github.com/mctechnology17/zmk-config)

### Example: DarrenVictoriano/zmk-config

**Configuration:**
- Corne keyboard layout
- nice!nano v2 + nice!view (keyboard halves)
- Seeed XIAO as dongle
- Step-by-step flashing instructions included

**Key Lesson:** XIAO BLE + nice!nano v2 halves is a tested working combination for Corne dongle setups.

**Source:** [DarrenVictoriano/zmk-config GitHub Repository](https://github.com/DarrenVictoriano/zmk-config)

---

## Root Cause Summary

### The Complete Picture

The pairing failure has **one primary root cause** with **one secondary issue**:

**Primary Root Cause:**
1. Missing `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2` in dongle configuration
2. Dongle defaults to accepting only 1 peripheral
3. First peripheral to connect (likely left half) succeeds
4. Second peripheral (right half) has no available slot, fails to pair

**Secondary Issue:**
1. Right half peripheral build configuration missing from build.yaml
2. Without this, right half firmware is built in standalone mode (as central)
3. A central will not pair with another central
4. Even if peripheral slots were available, right half wouldn't attempt to pair as peripheral

**Why Settings Reset Alone Didn't Fix It:**
- Settings reset correctly clears old bonds
- However, the underlying configuration issue remains
- No amount of bond clearing will create additional peripheral slots
- Configuration must be fixed in Kconfig.defconfig and build.yaml

**Why CONFIG_ZMK_KEYBOARD_NAME is Irrelevant:**
- This setting only affects host device pairing (computer ↔ keyboard)
- Internal split pairing uses hardware addresses, not string matching
- Different names between dongle and halves is perfectly valid

---

## Recommended Solution

### Step 1: Update Dongle Kconfig.defconfig

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/boards/shields/corne_dongle/Kconfig.defconfig`

**Add the missing configuration:**

```kconfig
if SHIELD_CORNE_DONGLE

config ZMK_KEYBOARD_NAME
	default "Corne Dongle"

config ZMK_SPLIT
	default y

config ZMK_SPLIT_ROLE_CENTRAL
	default y

# ADD THIS SECTION:
# Number of peripherals (2 for left + right halves)
config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
	default 2

# Central connects to 2 peripherals + 5 BT host profiles
config BT_MAX_CONN
	default 7

config BT_MAX_PAIRED
	default 7

# Larger ACL buffer for split communication
config BT_BUF_ACL_RX_SIZE
	default 255

config BT_BUF_ACL_TX_SIZE
	default 255

endif
```

### Step 2: Update build.yaml

**File:** `/Users/nerdo/personal/zmk-aurora-corne/build.yaml`

**Add right half peripheral build configuration:**

```yaml
# === KEYBOARD HALVES (DONGLE MODE) ===
# Both halves are peripherals, connecting to the dongle
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_left nice_view_adapter nice_view
  cmake-args: -DEXTRA_CONF_FILE=../../config/splitkb_aurora_corne_left_peripheral.conf
  artifact-name: splitkb_aurora_corne_left_peripheral-nice_nano_v2-zmk

# ADD THIS SECTION:
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_right nice_view_adapter nice_view
  cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
  artifact-name: splitkb_aurora_corne_right_peripheral-nice_nano_v2-zmk
```

### Step 3: Optional - Create Right Half Peripheral Config

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne_right_peripheral.conf`

```conf
# Right half peripheral mode for dongle operation
# When using a dongle, both halves must be peripherals

# Disable central role - dongle is the central
CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

# Import the main config
# Note: ZMK will also load splitkb_aurora_corne.conf via prefix matching
```

**Then update build.yaml to use it:**

```yaml
- board: nice_nano@2.0.0
  shield: splitkb_aurora_corne_right nice_view_adapter nice_view
  cmake-args: -DEXTRA_CONF_FILE=../../config/splitkb_aurora_corne_right_peripheral.conf
  artifact-name: splitkb_aurora_corne_right_peripheral-nice_nano_v2-zmk
```

### Step 4: Flash Sequence

1. **Build all firmware** via GitHub Actions with updated configuration

2. **Download firmware files:**
   - `settings_reset-xiao_ble-zmk.uf2`
   - `settings_reset-nice_nano_v2-zmk.uf2` (x2, for both halves)
   - `corne_dongle-xiao_ble-zmk.uf2`
   - `splitkb_aurora_corne_left_peripheral-nice_nano_v2-zmk.uf2`
   - `splitkb_aurora_corne_right_peripheral-nice_nano_v2-zmk.uf2`

3. **Flash settings_reset to all three devices:**
   - Dongle: `settings_reset-xiao_ble-zmk.uf2`
   - Left half: `settings_reset-nice_nano_v2-zmk.uf2`
   - Right half: `settings_reset-nice_nano_v2-zmk.uf2`

4. **Flash regular firmware to all three devices:**
   - Dongle: `corne_dongle-xiao_ble-zmk.uf2`
   - Left half: `splitkb_aurora_corne_left_peripheral-nice_nano_v2-zmk.uf2`
   - Right half: `splitkb_aurora_corne_right_peripheral-nice_nano_v2-zmk.uf2`

5. **Reset all three simultaneously** (press reset button or power on at same time)

6. **Verify pairing** - both halves should connect to dongle

---

## Additional Configuration Considerations

### Current Keyboard Settings

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/splitkb_aurora_corne.conf`

```conf
CONFIG_ZMK_KEYBOARD_NAME="dCorne mini"
CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y
CONFIG_ZMK_HID_REPORT_TYPE_NKRO=y
CONFIG_ZMK_HID_KEYBOARD_NKRO_EXTENDED_REPORT=y
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
CONFIG_ZMK_RGB_UNDERGLOW=y
```

**Impact Analysis:**
- `CONFIG_ZMK_KEYBOARD_NAME="dCorne mini"` - Loaded by both halves via prefix matching, but **dongle has its own name**, so this is fine
- `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` - Increases transmission power, can improve reliability but increases power consumption
- Other settings are peripheral-compatible

**No changes needed** to this file for dongle pairing.

### Dongle-Specific Settings

**File:** `/Users/nerdo/personal/zmk-aurora-corne/config/corne_dongle.conf`

```conf
# Dongle configuration - wireless central receiver
# Connects to both keyboard halves and forwards keystrokes via USB

# Enable split keyboard support
CONFIG_ZMK_SPLIT=y

# Dongle is the central (connects to peripherals)
CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y

# Enable USB for output to computer
CONFIG_ZMK_USB=y

# Disable features not needed on dongle
CONFIG_ZMK_RGB_UNDERGLOW=n
CONFIG_ZMK_BACKLIGHT=n

# BLE settings for reliable split communication
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
```

**Analysis:**
- Settings are appropriate for dongle use
- `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` may improve connection reliability
- No changes needed

---

## Testing and Verification

### Expected Behavior After Fix

1. **Power on sequence:**
   - Dongle powers on, starts advertising for peripherals
   - Left half powers on, discovers dongle, pairs
   - Right half powers on, discovers dongle, pairs

2. **Visual indicators** (if LEDs configured):
   - Successful peripheral connections typically indicated by LED status
   - nice!view displays (if configured) may show connection status

3. **Keystroke verification:**
   - Type on left half → characters appear on computer
   - Type on right half → characters appear on computer
   - All keys functional from both halves

### Troubleshooting if Still Fails

**If only one half connects:**
- Verify `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` is set to 2 in built firmware
- Check build logs for configuration warnings
- Confirm both peripheral firmware files were built and flashed

**If neither half connects:**
- Verify dongle firmware has `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`
- Check that peripheral firmware was built with `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`
- Try sequential reset: dongle first, wait 10 seconds, then halves

**If connection is intermittent:**
- May indicate BLE interference or range issues
- Try increasing `CONFIG_BT_CTLR_TX_PWR_PLUS_8` on peripherals (already set)
- Ensure dongle USB cable provides stable power

---

## References

### Official ZMK Documentation

- [Split Keyboards](https://zmk.dev/docs/features/split-keyboards) - Core split keyboard functionality
- [Keyboard Dongle](https://zmk.dev/docs/development/hardware-integration/dongle) - Dongle setup guide
- [Split Configuration](https://zmk.dev/docs/config/split) - Split-specific config options
- [System Configuration](https://zmk.dev/docs/config/system) - General system config including keyboard name
- [Connection Issues Troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues) - Pairing troubleshooting

### Community Resources

- [beekeeb.com - How to add dongle support to ZMK keyboards](https://beekeeb.com/how-to-add-dongle-and-prospector-support-to-hshs52-hshs46/) - Detailed walkthrough
- [SliceMK Documentation - ZMK Dongle vs Dongleless](https://docs.slicemk.com/firmware/zmk/wireless/dongle/) - Comparison and setup
- [GitHub Issue #2373](https://github.com/zmkfirmware/zmk/issues/2373) - Documents need for ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2

### Working Examples

- [DarrenVictoriano/zmk-config](https://github.com/DarrenVictoriano/zmk-config) - Corne + XIAO dongle + nice!nano v2
- [mctechnology17/zmk-config](https://github.com/mctechnology17/zmk-config) - Multi-keyboard dongle configurations

---

## Conclusion

The pairing failure is caused by a **missing configuration parameter** (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS`) and a **missing build entry** (right half peripheral mode) - not by keyboard name mismatches or fundamental incompatibility.

The fix is straightforward:
1. Add `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2` to dongle Kconfig.defconfig
2. Adjust BT_MAX_CONN and BT_MAX_PAIRED accordingly
3. Add right half peripheral build configuration to build.yaml
4. Follow proper settings reset and flashing procedure

This configuration is well-documented in ZMK's official documentation and proven working in multiple community implementations using the same hardware combination (XIAO BLE dongle + nice!nano v2 halves).
