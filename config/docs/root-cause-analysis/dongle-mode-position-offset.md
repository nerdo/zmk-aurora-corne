# Root Cause Analysis: -3 Position Offset in ZMK Dongle Mode

## Executive Summary

**Problem**: Aurora Corne keyboard exhibits a -3 key position offset in dongle mode (both halves peripheral) compared to standalone mode (left=central, right=peripheral).

**Root Cause**: In ZMK split keyboard architecture, peripherals use their local matrix transform to calculate positions before transmission. When the 6-column physical layout is selected but the actual physical keyboard has only 5 columns per half, the peripherals send positions calculated from their 6-column matrix transform (which includes gaps for non-existent outer columns), while the dongle uses its own 6-column transform to interpret keymaps. The -3 offset equals exactly 3 missing outer column positions (one per physical row: rows 0, 1, 2).

**Impact**: Keys are misinterpreted - pressing position 3 on the keyboard triggers the keymap binding for position 0, causing complete keymap misalignment.

---

## Investigation Timeline

### 1. Initial Observation
User reported that Aurora Corne Mini (5-column variant) works perfectly in standalone mode but has -3 position offset in dongle mode.

### 2. Hypothesis Formation
The -3 offset matches exactly the difference between 5-column and 6-column transform layouts:
- 6-column layout: 42 keys total (3 rows × 6 cols per half + 6 thumb keys)
- 5-column layout: 36 keys total (3 rows × 5 cols per half + 6 thumb keys)
- Difference in first 3 rows: 3 keys (one outer column per row)

### 3. Code Investigation

#### ZMK Split Architecture Discovery

**Key Finding 1**: Position calculation happens on the peripheral, not the central.

From ZMK source code analysis:

**File**: `/app/src/split/bluetooth/service.c`
```c
static int zmk_split_bt_position_pressed(uint8_t position) {
    WRITE_BIT(position_state[position / 8], position % 8, true);
    return send_position_state();
}

static int zmk_split_bt_position_released(uint8_t position) {
    WRITE_BIT(position_state[position / 8], position % 8, false);
    return send_position_state();
}
```

The peripheral receives a `position` value (already calculated) and sends it as-is to the central using bit-packed array encoding.

**Key Finding 2**: The position is calculated by matrix transform on the peripheral.

**File**: `/app/src/matrix_transform.c`
```c
int32_t zmk_matrix_transform_row_column_to_position(zmk_matrix_transform_t mt,
                                                     uint32_t row,
                                                     uint32_t column) {
    column += mt->col_offset;
    row += mt->row_offset;

    if (!mt->lookup_table) {
        return (row * mt->columns) + column;
    }

    uint16_t lookup_index = ((row * mt->columns) + column);
    if (lookup_index >= mt->len) {
        return -EINVAL;
    }

    int32_t val = mt->lookup_table[lookup_index];
    if (val == 0) {
        return -EINVAL;
    }

    return val - INDEX_OFFSET;
}
```

This function:
1. Takes the row/column from kscan (GPIO matrix indices)
2. Applies col_offset and row_offset
3. Looks up the position in the transform's map array
4. Returns the keymap position index

**Key Finding 3**: Each device (peripheral or central) uses its own devicetree matrix transform.

From ZMK documentation ([Split Keyboards](https://zmk.dev/docs/features/split-keyboards)):
> "In split keyboards running ZMK, one part is assigned the 'central' role which receives key position and sensor events from the other parts that are called 'peripherals.'"

From ZMK documentation ([New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield)):
> "When a key is pressed, a kscan event is generated from it with a row and a column value corresponding to the zero-based indices of the row-gpios and col-gpios pins that triggered the event, respectively. Then, the 'key position' triggered is the index of the RC(row, column) in the matrix transform where row and column are the indices as mentioned above."

### 4. Matrix Transform Analysis

#### Upstream Aurora Corne Transforms

**File**: `zmk/app/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne.dtsi`

**6-Column Transform (default_transform)**:
```dts
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
```

This defines a 42-key layout with positions 0-41.

**5-Column Transform (five_column_transform)**:
```dts
five_column_transform: keymap_transform_1 {
    compatible = "zmk,matrix-transform";
    columns = <10>;
    rows = <4>;
    map = <
RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) RC(0,7) RC(0,8) RC(0,9) RC(0,10)
RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10)
RC(2,1) RC(2,2) RC(2,3) RC(2,4) RC(2,5)  RC(2,6) RC(2,7) RC(2,8) RC(2,9) RC(2,10)
                RC(3,3) RC(3,4) RC(3,5)  RC(3,6) RC(3,7) RC(3,8)
    >;
};
```

This defines a 36-key layout with positions 0-35, **skipping RC(0,0), RC(1,0), RC(2,0), RC(0,11), RC(1,11), RC(2,11)** - the outer columns.

#### Right Half Column Offset

**File**: `zmk/app/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_right.overlay`

```dts
&default_transform {
    col-offset = <6>;
};

&five_column_transform {
    col-offset = <6>;
};
```

Both transforms use `col-offset = 6` on the right half. This was fixed in [PR #1584](https://github.com/zmkfirmware/zmk/pull/1584) to address [Issue #1570](https://github.com/zmkfirmware/zmk/issues/1570).

From the ZMK split keyboard documentation:
> "To allow the kscan matrices to be joined in the matrix transform, an offset is applied to the matrix transform of peripherals. This offset means that when the right half of the keyboard has a key event triggered by the GPIO pins at the indices 0,0 of its row-gpios and col-gpios arrays respectively, it will interpret it as an RC(0,3) event rather than an RC(0,0) event."

### 5. Position Calculation Examples

#### Scenario 1: Standalone Mode (Left=Central, Right=Peripheral)

**Configuration**:
- Left half: Uses physical layout `foostan_corne_6col_layout` → `default_transform`
- Right half: Uses physical layout `foostan_corne_6col_layout` → `default_transform` with `col-offset = 6`

**Left Half** (acting as central):
- Physical key at GPIO row 0, col 0
- Matrix transform calculation: `lookup_index = (0 * 12) + 0 = 0`
- Map lookup: `default_transform.map[0] = RC(0,0)` → position **0**
- This position is processed locally by the central's keymap

**Right Half** (acting as peripheral):
- Physical key at GPIO row 0, col 0
- Matrix transform calculation with col-offset: `lookup_index = (0 * 12) + (0 + 6) = 6`
- Map lookup: `default_transform.map[6] = RC(0,6)` → position **6**
- This position is transmitted to central
- Central's keymap processes position 6 correctly

Result: All positions align correctly between physical layout and keymap.

#### Scenario 2: Dongle Mode (Both Halves=Peripheral, Dongle=Central)

**Configuration**:
- Dongle: Uses `default_transform` (6-column, 42-key layout)
- Left half: User set `zmk,physical-layout = &foostan_corne_6col_layout` in overlay
- Right half: Inherits default physical layout

**User's Configuration** (`/config/splitkb_aurora_corne.overlay`):
```dts
/ {
    chosen {
        zmk,physical-layout = &foostan_corne_6col_layout;
    };
};
```

This forces the left half peripheral to use `default_transform` (6-column).

**Left Half Peripheral** (Aurora Corne Mini - physically 5 columns):
- Physical keyboard only has columns 1-5 (no column 0 or column 6)
- But using `default_transform` which expects columns 0-5
- Physical key at GPIO row 0, col 0 (which is actually the SECOND column physically)
- Matrix transform calculation: `lookup_index = (0 * 12) + 0 = 0`
- Map lookup: `default_transform.map[0] = RC(0,0)` → position **0**
- Transmitted to dongle: position 0
- But this is physically the second column, which should be position **1** in a 5-column layout!

**The Critical Mismatch**:
- Physical Aurora Corne Mini has GPIO columns 0-4 (5 columns)
- These correspond to physical layout columns 1-5 (the middle 5 of a 6-column layout)
- But the GPIO column 0 maps to RC(0,0) in the transform
- When it should map to RC(0,1) for a 5-column physical layout

**Position Offset Calculation**:
```
Row 0: Missing outer column (col 0) = -1 position
Row 1: Missing outer column (col 0) = -1 position
Row 2: Missing outer column (col 0) = -1 position
Row 3: No outer columns (thumb keys only)
Total offset: -3 positions
```

### 6. Comparison with Upstream Foostan Corne

Let me check the upstream Foostan Corne shield definition to understand the physical layout system:

From upstream `zmk/app/boards/shields/corne/corne.dtsi`, the physical layout definitions reference:
```dts
&foostan_corne_6col_layout {
    transform = <&default_transform>;
};

&foostan_corne_5col_layout {
    transform = <&five_column_transform>;
};
```

These physical layout nodes are defined in ZMK's layout system and map to the appropriate transforms.

---

## Root Cause Identification

### The Fundamental Issue

**In ZMK split keyboard architecture, each peripheral independently calculates key positions using its own matrix transform before transmitting to the central.**

The position offset occurs because:

1. **Peripheral Position Calculation**: Each peripheral (left and right halves) uses its devicetree matrix transform to convert GPIO row/col → position index

2. **Transform Mismatch**: When the Aurora Corne Mini (5 physical columns) uses the 6-column transform:
   - GPIO column indices 0-4 exist on the hardware
   - Transform expects GPIO columns representing layout columns 0-5
   - But physically, GPIO col 0 is layout column 1, GPIO col 1 is layout column 2, etc.

3. **No Central Reinterpretation**: The dongle central receives position indices from peripherals and looks them up in its own transform, but **does not recalculate them** - it uses the positions as keymap indices directly

4. **Position Transmission**: The peripheral sends the calculated position (0-41 for 6-col layout), and the dongle's keymap uses this position directly to look up bindings

### Why Standalone Mode Works

In standalone mode:
- Left half is central: processes its own keys using its transform (correct)
- Right half is peripheral: calculates positions with `col-offset = 6`, transmits to left central
- The `col-offset = 6` on the right half correctly shifts its positions to the right side of the keymap (positions 6-11 per row)

Both halves use the same transform (6-column), so position calculations align.

### Why Dongle Mode Fails

In dongle mode:
- Left half is peripheral: calculates positions using 6-column transform
- Right half is peripheral: calculates positions using 6-column transform with `col-offset = 6`
- Dongle is central: has 6-column transform for keymap interpretation

The issue: The physical Aurora Corne Mini has 5 columns, but GPIO wiring:
- Aurora Corne (original): GPIO cols 0-5 map to layout cols 0-5 (6 physical columns)
- Aurora Corne Mini: GPIO cols 0-4 map to layout cols 1-5 (5 physical columns, outer removed)

When using 6-column transform on Aurora Corne Mini:
- GPIO col 0 → transform looks up position for layout col 0 → position 0
- But GPIO col 0 is physically layout col 1 → should be position 1
- Result: -1 offset per row with missing column = -3 total

---

## Evidence Collection

### Code Evidence

#### 1. Peripheral Position Transmission
**File**: `/app/src/split/bluetooth/service.c`
```c
static int zmk_split_bt_position_pressed(uint8_t position) {
    WRITE_BIT(position_state[position / 8], position % 8, true);
    return send_position_state();
}
```
- Peripheral sends the position value as-is, pre-calculated by its matrix transform

#### 2. Matrix Transform Lookup
**File**: `/app/src/matrix_transform.c`
```c
int32_t zmk_matrix_transform_row_column_to_position(zmk_matrix_transform_t mt,
                                                     uint32_t row,
                                                     uint32_t column) {
    column += mt->col_offset;
    row += mt->row_offset;

    if (!mt->lookup_table) {
        return (row * mt->columns) + column;
    }

    uint16_t lookup_index = ((row * mt->columns) + column);
    if (lookup_index >= mt->len) {
        return -EINVAL;
    }

    int32_t val = mt->lookup_table[lookup_index];
    if (val == 0) {
        return -EINVAL;
    }

    return val - INDEX_OFFSET;
}
```
- Each device uses its own transform to calculate positions
- The lookup_index calculation uses the GPIO row/col directly

#### 3. Transform Definitions

**6-Column Transform** includes outer columns:
```dts
map = <
RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) ...
```
Position 0 = RC(0,0) = GPIO row 0, col 0

**5-Column Transform** skips outer columns:
```dts
map = <
RC(0,1) RC(0,2) RC(0,3) RC(0,4) RC(0,5)  RC(0,6) ...
```
Position 0 = RC(0,1) = GPIO row 0, col 1

#### 4. User Configuration
**File**: `/config/splitkb_aurora_corne.overlay`
```dts
/ {
    chosen {
        zmk,physical-layout = &foostan_corne_6col_layout;
    };
};
```
- Forces 6-column physical layout (and thus 6-column transform)
- But physical hardware is 5-column Aurora Corne Mini

### Log Evidence

User reported symptoms:
- Standalone mode: Works perfectly
- Dongle mode: -3 position offset
- Offset is consistent across all keys
- Offset equals exactly 3 outer column positions

### Configuration Evidence

From ZMK documentation and upstream code:
- `col-offset` is used to shift peripheral positions for split keyboards
- Both `default_transform` and `five_column_transform` use `col-offset = 6` on right half
- Physical layouts map to transforms: `foostan_corne_6col_layout` → `default_transform`, `foostan_corne_5col_layout` → `five_column_transform`

---

## Five Whys Analysis

**Problem**: Dongle mode has -3 position offset

**Why 1**: Why is there a -3 position offset?
- Because the left peripheral calculates positions using the wrong transform for its physical layout

**Why 2**: Why does the left peripheral use the wrong transform?
- Because the user configured `zmk,physical-layout = &foostan_corne_6col_layout` in the overlay, forcing the 6-column transform

**Why 3**: Why does using the 6-column transform cause a -3 offset on a 5-column keyboard?
- Because the 6-column transform includes positions for outer columns (0 and 11) that don't exist on the 5-column hardware, creating a mismatch between GPIO column indices and transform column positions

**Why 4**: Why doesn't the dongle central correct this mismatch?
- Because ZMK split keyboard architecture has peripherals calculate positions locally using their own transform, then transmit the position index to the central, which uses it directly as a keymap index without recalculation

**Why 5**: Why do peripherals calculate positions locally instead of sending raw GPIO row/col?
- Because this is a fundamental ZMK architectural decision to allow peripherals to have independent matrix transforms and reduce protocol complexity - peripherals send positions (up to 128 keys in 16 bytes bit-packed), not raw GPIO data

---

## Fishbone (Cause and Effect) Diagram

```
                                   -3 Position Offset in Dongle Mode
                                              |
                    ┌─────────────────────────┴─────────────────────────┐
                    |                                                     |
             People & Process                                      Technology
                    |                                                     |
    ┌───────────────┴───────────────┐                   ┌────────────────┴────────────────┐
    |                               |                   |                                 |
User configured                Documentation      ZMK Split Architecture        Hardware Mismatch
wrong physical layout          unclear about       peripherals calculate         5-col physical
in overlay                     Aurora Corne        positions locally             with 6-col transform
    |                          Mini variant             using own transform           |
    |                               |                   |                             |
    └───────────────┬───────────────┘                   └──────────────┬──────────────┘
                    |                                                  |
             Configuration Error                              Design Limitation
                    |                                                  |
                    └─────────────────────┬────────────────────────────┘
                                          |
                    ┌─────────────────────┴─────────────────────┐
                    |                                           |
              Environment                                   Materials
                    |                                           |
    ┌───────────────┴───────────────┐           ┌──────────────┴──────────────┐
    |                               |           |                             |
Dongle mode:                   Matrix transform      Physical layout         Transform map
both halves peripheral,        mismatch between      definitions don't       arrays define
different config than          peripheral and        specify GPIO            position indices
standalone mode                physical hardware     wiring details          |
    |                               |                   |                     |
    └───────────────┬───────────────┘                   └──────────┬──────────┘
                    |                                              |
          Split Mode Difference                           Abstraction Gap
                    |                                              |
                    └─────────────────┬────────────────────────────┘
                                      |
                           ROOT CAUSE: Peripheral uses
                           6-column transform on 5-column
                           hardware, calculating positions
                           with -3 offset before transmission
```

---

## Actionable Remediation Plan

### Immediate Fix (Symptom Resolution)

**Action**: Change the physical layout selection in peripheral configuration

**For Left Half** - Modify `/config/splitkb_aurora_corne.overlay`:
```dts
/ {
    chosen {
        zmk,physical-layout = &foostan_corne_5col_layout;  // Change from 6col to 5col
    };
};
```

**Result**: Left peripheral will use `five_column_transform`, calculating positions 0-35 correctly for the 5-column physical layout.

### Root Cause Correction (Prevent Recurrence)

**Option 1: Physical Layout Selection (Recommended)**

Ensure the physical layout matches the actual hardware:
- Aurora Corne (full size): Use `foostan_corne_6col_layout`
- Aurora Corne Mini: Use `foostan_corne_5col_layout`

**Option 2: Custom Transform for Aurora Corne Mini**

Create a transform that maps the Mini's GPIO columns correctly:
```dts
mini_transform: mini_transform {
    compatible = "zmk,matrix-transform";
    columns = <10>;  // Total columns including right half
    rows = <4>;
    map = <
        // Left half: GPIO cols 0-4 map to layout positions 0-4
        RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4)
        // Right half: GPIO cols 0-4 with col-offset 5 map to layout positions 5-9
        RC(0,5) RC(0,6) RC(0,7) RC(0,8) RC(0,9)
        ...
    >;
};
```

**Option 3: Document Hardware Variant Clearly**

Add comments to configuration files indicating:
- Which physical hardware variant is being used
- Which transform should be selected for that variant
- The difference between Aurora Corne and Aurora Corne Mini GPIO wiring

### Preventive Measures

1. **Configuration Validation**: Add build-time checks for physical layout vs. kscan column count mismatch

2. **Documentation Enhancement**: Document the relationship between:
   - Physical hardware variants (full Corne vs. Mini)
   - GPIO column wiring
   - Matrix transform selection
   - Physical layout selection

3. **Template Configurations**: Provide example configurations for each hardware variant with clear comments

### Testing Strategy

To validate the fix:

1. **Position Mapping Test**: Create a test keymap with unique outputs for each position (e.g., numbers 0-35)

2. **Both Modes Test**: Verify positions are correct in both:
   - Standalone mode (left=central)
   - Dongle mode (dongle=central)

3. **All Keys Test**: Systematically test each physical key to ensure it triggers the correct keymap position

---

## Risk Assessment

### Impact Analysis

**Current Impact**: HIGH
- Complete keymap misalignment in dongle mode
- All keys trigger incorrect bindings
- Keyboard effectively unusable in dongle mode

**Risk of Fix**: LOW
- Configuration change only (devicetree overlay)
- No code modification required
- Easily reversible
- Well-tested ZMK functionality (physical layouts are standard feature)

### Urgency Classification

**Urgency**: HIGH
- Blocking primary use case (dongle mode)
- Simple configuration fix available
- No workaround exists without reconfiguring keymap positions

---

## Sources and References

### ZMK Documentation
- [Split Keyboards](https://zmk.dev/docs/features/split-keyboards) - Split keyboard architecture
- [New Keyboard Shield](https://zmk.dev/docs/development/hardware-integration/new-shield) - Matrix transform documentation
- [Keyboard Scan Configuration](https://zmk.dev/docs/config/kscan) - KScan configuration
- [Layout Configuration](https://zmk.dev/docs/config/layout) - Physical layout configuration

### ZMK Source Code
- [matrix_transform.c](https://github.com/zmkfirmware/zmk/blob/main/app/src/matrix_transform.c) - Position calculation implementation
- [split/bluetooth/service.c](https://github.com/zmkfirmware/zmk/blob/main/app/src/split/bluetooth/service.c) - Peripheral position transmission
- [split/bluetooth/central.c](https://github.com/zmkfirmware/zmk/blob/main/app/src/split/bluetooth/central.c) - Central position reception
- [splitkb_aurora_corne.dtsi](https://github.com/zmkfirmware/zmk/blob/main/app/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne.dtsi) - Transform definitions
- [splitkb_aurora_corne_right.overlay](https://github.com/zmkfirmware/zmk/blob/main/app/boards/shields/splitkb_aurora_corne/splitkb_aurora_corne_right.overlay) - Right half col-offset

### ZMK Issues and Pull Requests
- [Issue #1570](https://github.com/zmkfirmware/zmk/issues/1570) - Column offset bug report
- [PR #1584](https://github.com/zmkfirmware/zmk/pull/1584) - Column offset fix
- [PR #1662](https://github.com/zmkfirmware/zmk/pull/1662) - Transform validation fix
- [Issue #721](https://github.com/zmkfirmware/zmk/issues/721) - Row-offset property discussion

### Devicetree Bindings
- [zmk,matrix-transform.yaml](https://github.com/zmkfirmware/zmk/blob/main/app/dts/bindings/zmk,matrix-transform.yaml) - Matrix transform binding specification

---

## Conclusion

The -3 position offset in dongle mode is caused by a configuration mismatch: the Aurora Corne Mini (5 physical columns per half) is configured to use the 6-column physical layout, causing the left peripheral to calculate positions using a transform that includes non-existent outer columns.

The fix is straightforward: change the physical layout selection from `&foostan_corne_6col_layout` to `&foostan_corne_5col_layout` in the overlay file to match the actual hardware.

This investigation reveals the importance of understanding ZMK's split keyboard architecture: peripherals calculate positions locally using their own matrix transforms before transmitting to the central. Any transform mismatch on the peripheral will cause position offset issues that cannot be corrected at the central.
