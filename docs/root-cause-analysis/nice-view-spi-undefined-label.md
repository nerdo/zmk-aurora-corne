# Root Cause Analysis: undefined node label 'nice_view_spi'

**Date**: 2026-02-28
**Error**: `devicetree error: zmk/app/boards/shields/nice_view/nice_view.overlay:7 (column 1): parse error: undefined node label 'nice_view_spi'`
**Build target**: `nice_nano@2.0.0` + shields `splitkb_aurora_corne_left nice_view_adapter nice_view`

---

## Executive Summary

Two related issues cause this build failure. The primary issue is that ZMK's migration to Zephyr v4.1 and HWMv2 board system (PR #3145, merged 2026-02-12) requires the `//zmk` qualifier when specifying the nice_nano board. The build spec `nice_nano@2.0.0` selects the BASE board (without ZMK customizations), which causes the shield board overlay lookup to use a filename pattern that does not match the `nice_view_adapter` board overlay file, leaving `nice_view_spi` undefined.

A secondary issue exists in `config/splitkb_aurora_corne.overlay`: it contains a copy of the old `nice_view_adapter` board overlay content (without the `nice_view_spi:` label alias) that would conflict with the adapter's overlay once the board ID is corrected.

**Required changes**:
1. Update board IDs in `build.yaml` from `nice_nano@2.0.0` to `nice_nano//zmk`
2. Remove the duplicated SPI0 setup from `config/splitkb_aurora_corne.overlay` (lines 1-30)
3. Retain only the necessary overrides: led_strip chain-length, nice_view_spi CS pin, and physical layout

---

## Investigation Timeline

### Step 1: ZMK Commit History

Searched ZMK `main` branch for recent relevant commits. Key commits identified:

- `c06fa48c` (2025-12-10): `feat!: Move to zephyr v4.1` - major build system upgrade
- `6690d535` (2026-02-12): `refactor(core): Adjust our approach for upstream Zephyr boards (#3145)` - moved to ZMK-specific board variants requiring `//zmk` qualifier

### Step 2: nice_view_adapter/boards File Listing (Current State)

**Before PR #3145** (at commit `6e7e0de2`):
```
bluemicro840_v1.overlay
mikoto.overlay
nice_nano.overlay          <- matched build string "nice_nano"
nice_nano_v2.overlay       <- matched build string "nice_nano_v2"
nrfmicro_11.overlay
...
```

**After PR #3145** (current `main`):
```
bluemicro840_nrf52840_zmk.overlay
mikoto_nrf52840_zmk.overlay
nice_nano_nrf52840_zmk.overlay    <- requires qualifier /nrf52840/zmk to match
nrfmicro_nrf52840_zmk_1_1_0.overlay
...
```

The file content is **identical** - only the filename changed to match the new HWMv2 board naming convention.

### Step 3: ZMK Board Architecture with HWMv2

ZMK uses a two-tier board system:

**Base board** (in `app/module/boards/nicekeyboards/nice_nano/board.yml`):
```yaml
board:
  name: nice_nano
  vendor: nicekeyboards
  socs:
    - name: nrf52840
  revision:
    format: major.minor.patch
    default: 2.0.0
    revisions:
      - name: 1.0.0
      - name: 2.0.0
```
This defines `nice_nano` as a board with SoC `nrf52840`, no ZMK-specific features.

**ZMK extension** (in `app/boards/nicekeyboards/nice_nano/board.yml`):
```yaml
board:
  extend: nice_nano
  variants:
    - name: zmk
      qualifier: nrf52840
```
This adds the `zmk` variant (full qualifier: `nrf52840/zmk`) to the base board.

Valid board targets after HWMv2 resolution:
- `nice_nano/nrf52840` → base board, no ZMK-specific features (SPI1 only, no SPI0 pinctrl)
- `nice_nano/nrf52840/zmk` → ZMK variant, full ZMK support

### Step 4: Board ID Resolution Analysis

When the user specifies `nice_nano@2.0.0` (no `//zmk`):

1. `BOARD = nice_nano`, `BOARD_REVISION = 2.0.0`
2. Zephyr auto-fills `BOARD_QUALIFIERS = nrf52840` (single-SoC rule from `boards.cmake` line 269-270)
3. `nice_nano/nrf52840` is a valid board target (the base board)
4. Build proceeds using the BASE board (no ZMK variant features)

When the user specifies `nice_nano//zmk` (correct):

1. `BOARD = nice_nano`, no revision (defaults to 2.0.0 per `zmk.yml`)
2. Zephyr resolves `//zmk` → `BOARD_QUALIFIERS = /nrf52840/zmk` (SoC auto-filled)
3. Full board target: `nice_nano/nrf52840/zmk`
4. Build uses the ZMK variant with full SPI0 configuration

### Step 5: Shield Board Overlay Filename Resolution

Zephyr's `shields.cmake` calls `zephyr_file(CONF_FILES <shield>/boards ...)` which uses `zephyr_build_string` with `MERGE REVERSE` to generate candidate filenames.

**For `nice_nano@2.0.0` (no `//zmk`)**:
```
BOARD = nice_nano
BOARD_REVISION = 2.0.0 → 2_0_0
BOARD_QUALIFIERS = nrf52840

Candidates (REVERSE order, checked first-to-last):
  1. nice_nano_nrf52840.overlay     ← fallback
  2. nice_nano_nrf52840_2_0_0.overlay  ← with revision
```
Neither file exists in `nice_view_adapter/boards/`. The overlay is NOT loaded.

**For `nice_nano//zmk` (correct)**:
```
BOARD = nice_nano
BOARD_REVISION = 2.0.0 (default) → 2_0_0
BOARD_QUALIFIERS = nrf52840/zmk

Candidates (REVERSE order):
  1. nice_nano_nrf52840_zmk.overlay        ← fallback (EXISTS!)
  2. nice_nano_nrf52840_zmk_2_0_0.overlay  ← with revision
```
`nice_nano_nrf52840_zmk.overlay` IS found and loaded. It defines `nice_view_spi: &spi0 { ... }`.

### Step 6: Cascade to Build Failure

`nice_view.overlay` line 7:
```c
&nice_view_spi {
    status = "okay";
    nice_view: ls0xx@0 {
```

The label `nice_view_spi` must be defined before `nice_view.overlay` is processed. This label is created by `nice_view_adapter/boards/nice_nano_nrf52840_zmk.overlay`:
```c
nice_view_spi: &spi0 {
    compatible = "nordic,nrf-spim";
    ...
};
```
When this file is not loaded (due to wrong board ID), `nice_view_spi` is never defined, and the DTC compiler fails with the reported error.

### Step 7: Secondary Issue in User's Config Overlay

The user's `config/splitkb_aurora_corne.overlay` contains (lines 1-30):
```c
// start https://github.com/zmkfirmware/zmk/blob/8ed556df.../nice_view_adapter/boards/nice_nano_v2.overlay
&pinctrl {
    spi0_default: spi0_default { ... };
    spi0_sleep: spi0_sleep { ... };
};

&spi0 {  // <-- MISSING "nice_view_spi: " label alias prefix!
    compatible = "nordic,nrf-spim";
    cs-gpios = <&pro_micro 19 GPIO_ACTIVE_HIGH>;  // <-- Wrong pin (should be pro_micro 1)
};
```

This was copied from the old `nice_nano_v2.overlay` (commit `8ed556df`, Oct 2024), but:
1. The `nice_view_spi:` label alias was not copied (so the label is never defined here either)
2. The CS pin was changed from `pro_micro 1` to `pro_micro 19` (the correct override is `pro_micro 20` / `gpio0 29`)
3. This content belongs in a board-specific overlay under a `boards/` subdirectory, NOT in the shield config overlay

If the board ID is fixed to `nice_nano//zmk`, `nice_view_adapter` will load its board overlay AND the user's overlay will ALSO try to define `spi0_default`/`spi0_sleep` pinctrl nodes - causing a duplicate definition conflict.

### Step 8: User's Correct Intent Identified

The user's overlay also contains:
```c
&nice_view_spi {
    cs-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>; // Assigning CS to D20
};
```

This is correct for the Aurora Corne board. The `nice_view_adapter` defaults to `cs-gpios = <&pro_micro 1 GPIO_ACTIVE_HIGH>` (D1 = gpio0 6), but the Aurora Corne hardware connects the nice_view CHIP_SELECT line to D20 (gpio0 29 on the nRF52840).

Pro Micro pin mapping (from `arduino_pro_micro_pins.dtsi`):
- `pro_micro 1` = `gpio0 6` (default in nice_view_adapter)
- `pro_micro 19` = `gpio0 2` (what the user accidentally put in their SPI copy)
- `pro_micro 20` = `gpio0 29` (correct CS for Aurora Corne, what the user intended for nice_view_spi)

---

## Root Cause Analysis (Five Whys)

**Why did the build fail?**
The DTC compiler could not find the `nice_view_spi` node label when processing `nice_view.overlay`.

**Why was `nice_view_spi` undefined?**
The `nice_view_adapter` shield's board-specific overlay (`nice_nano_nrf52840_zmk.overlay`) was not loaded during the build.

**Why was the board overlay not loaded?**
Zephyr's shield board overlay filename resolution generates candidate names based on `BOARD_QUALIFIERS`. With `nice_nano@2.0.0` (no `//zmk`), the qualifiers produce `nice_nano_nrf52840.overlay` as the candidate filename. This file does not exist — only `nice_nano_nrf52840_zmk.overlay` exists.

**Why does `nice_nano@2.0.0` produce the wrong qualifiers?**
ZMK PR #3145 (2026-02-12) refactored board definitions to use Zephyr HWMv2 board extensions. In this new system, the ZMK-specific board variant requires the `//zmk` qualifier (shorthand for `/nrf52840/zmk`). Without `//zmk`, you get the base board (no ZMK customizations), which produces different board qualifier strings that don't match the renamed overlay files.

**Why was the board.yaml not updated?**
The user's configuration predates PR #3145. The build previously worked because the old overlay was named `nice_nano_v2.overlay` (matching the old board ID convention `nice_nano_v2`). PR #3145 renamed overlays to use the new HWMv2 naming scheme, breaking configs that still use pre-4.1 board IDs.

---

## Evidence

### nice_view_adapter/boards/nice_nano_nrf52840_zmk.overlay (current)
File path: `zmk/app/boards/shields/nice_view_adapter/boards/nice_nano_nrf52840_zmk.overlay`
```c
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

nice_view_spi: &spi0 {         // <-- defines the "nice_view_spi" label
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&pro_micro 1 GPIO_ACTIVE_HIGH>;
};

&pro_micro_i2c {
    status = "disabled";
};
```

### nice_view.overlay (current, line 7 = line that fails)
```c
&nice_view_spi {   // line 7 - "undefined node label" error
    status = "okay";
    nice_view: ls0xx@0 {
        ...
    };
};
```

### ZMK Blog Post: Board ID Migration Table

From `docs/blog/2025-12-09-zephyr-4-1.md`:
```
nice!nano (nice_nano):
  nice_nano    → nice_nano@1.0.0//zmk  (short: nice_nano@1//zmk)
  nice_nano_v2 → nice_nano@2.0.0//zmk  (short: nice_nano//zmk)
```

The short form `nice_nano//zmk` is equivalent to `nice_nano@2.0.0//zmk` because `default_revision: "2.0.0"` is set in `nice_nano.zmk.yml`.

### Zephyr boards.cmake: Single-SoC Auto-Fill (Lines 265-275)
```cmake
if(LIST_BOARD_QUALIFIERS)
  list(LENGTH LIST_BOARD_SOCS socs_length)
  if(socs_length EQUAL 1)
    set(BOARD_SINGLE_SOC TRUE)
    if(NOT DEFINED BOARD_QUALIFIERS)
      set(BOARD_QUALIFIERS "${LIST_BOARD_SOCS}")   # auto-sets to "nrf52840"
    elseif("/${BOARD_QUALIFIERS}" MATCHES "^//.*")
      string(REGEX REPLACE "^/" "${LIST_BOARD_SOCS}/" BOARD_QUALIFIERS "${BOARD_QUALIFIERS}")  # fills in SoC for //zmk
    endif()
  endif()
endif()
```

For `nice_nano@2.0.0`: `BOARD_QUALIFIERS` not defined → auto-set to `nrf52840`
For `nice_nano//zmk`: `BOARD_QUALIFIERS` starts as `/zmk` (matches `^//.*`) → replaced with `nrf52840/zmk`

### Zephyr extensions.cmake: Build String with MERGE REVERSE
```cmake
# calling zephyr_build_string(... BOARD nice_nano BOARD_REVISION 2.0.0 BOARD_QUALIFIERS nrf52840 MERGE REVERSE)
# returns: ["nice_nano_nrf52840", "nice_nano_nrf52840_2_0_0"]
# checked in order: nice_nano_nrf52840.overlay (missing), nice_nano_nrf52840_2_0_0.overlay (missing)
# → board overlay NOT loaded

# calling zephyr_build_string(... BOARD nice_nano BOARD_REVISION 2.0.0 BOARD_QUALIFIERS nrf52840/zmk MERGE REVERSE)
# returns: ["nice_nano_nrf52840_zmk", "nice_nano_nrf52840_zmk_2_0_0"]
# checked in order: nice_nano_nrf52840_zmk.overlay (EXISTS!), ...
# → board overlay loaded, nice_view_spi defined
```

---

## Actionable Remediation Plan

### Priority 1: Fix Board IDs in build.yaml (IMMEDIATE)

Update **all** `nice_nano@2.0.0` entries to `nice_nano//zmk`:

```yaml
# build.yaml - change this pattern:
- board: nice_nano@2.0.0

# to this (for all occurrences):
- board: nice_nano//zmk
```

The `nice_nano//zmk` shorthand is equivalent to `nice_nano@2.0.0//zmk` because `2.0.0` is the default revision.

**Important**: Also consider updating `xiao_ble` to `xiao_ble//zmk` for the dongle build. Currently `xiao_ble` without `//zmk` uses Zephyr's stock board definition (missing ZMK-specific battery ADC, QSPI flash, and UF2 boot configuration). The dongle may function without these, but `xiao_ble//zmk` is the intended board ID for ZMK builds.

### Priority 2: Clean Up config/splitkb_aurora_corne.overlay (IMMEDIATE)

The current overlay (lines 1-30) copies content that belongs in `nice_view_adapter`'s board overlay. Once the board ID is fixed, this copy will **conflict** with the adapter's overlay (duplicate `spi0_default`/`spi0_sleep` pinctrl definitions).

**Remove** lines 1-30 (the SPI setup copy) and keep only the Aurora Corne-specific overrides:

```c
// Device-specific configuration (not applicable to dongle)
&led_strip { chain-length = <24>; };

// Override nice_view CS to D20 (Aurora Corne-specific hardware wiring)
&nice_view_spi {
    cs-gpios = <&pro_micro 20 GPIO_ACTIVE_HIGH>;  // D20 = gpio0 29 on nice_nano
};

/*
 * FORCE 6-column layout explicitly.
 */
/ {
    chosen {
        zmk,physical-layout = &foostan_corne_6col_layout;
    };
};
```

Note: The CS pin can use either `<&pro_micro 20 GPIO_ACTIVE_HIGH>` or `<&gpio0 29 GPIO_ACTIVE_HIGH>` — both refer to the same physical pin. Using `&pro_micro 20` is preferred for consistency with the Pro Micro abstraction.

### Priority 3: Verify Settings Reset After Firmware Update

After flashing updated firmware, perform a settings reset on all devices (as noted in `CLAUDE.md` quirk #8) to clear any saved state that might interfere with the new firmware.

---

## Preventive Measures

1. **Pin ZMK revision**: Consider using a pinned revision in `config/west.yml` instead of `revision: main` to prevent future upstream breaking changes from affecting builds without warning.

2. **Follow ZMK migration guides**: When ZMK releases major version changes (like the Zephyr 4.1 upgrade), consult the blog post at `docs/blog/2025-12-09-zephyr-4-1.md` in the ZMK repository for required configuration changes.

3. **Board ID format**: The new convention for all ZMK boards is `<board>//zmk` (or `<board>@<revision>//zmk` for non-default revisions). The `//zmk` suffix is mandatory for all in-tree ZMK boards going forward.

---

## Risk Assessment

**Impact**: High — build completely fails, no firmware can be generated.

**Urgency**: High — affects all nice_nano-based builds tracking ZMK `main` after 2026-02-12.

**Fix Complexity**: Low — the primary fix is a one-line change per build target in `build.yaml`, plus removing ~30 lines of conflicting content from the overlay file.

**Risk of Fix**: Low — the changes are well-understood and revert to using the intended ZMK infrastructure rather than working around it.
