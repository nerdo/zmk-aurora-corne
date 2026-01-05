# Root Cause Analysis: ZMK Loading Wrong Keymap for Dongle Build

**Date**: 2026-01-04
**Issue**: Build for `aurora_corne_dongle_xiao` shield loads `config/splitkb_aurora_corne.keymap` instead of `config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap`
**Impact**: Build fails with error about `&led_strip` node not existing (dongle has no RGB LEDs)

---

## Executive Summary

**Root Cause**: ZMK's keymap file selection algorithm strips known suffixes from shield names (`_left`, `_right`) but NOT custom variant suffixes (`_dongle_xiao`). When building `aurora_corne_dongle_xiao`, ZMK cannot find `config/aurora_corne_dongle_xiao.keymap` and falls back to searching for a base shield name match. The directory name `splitkb_aurora_corne` matches the existing `config/splitkb_aurora_corne.keymap`, causing the wrong keymap to be selected.

**Immediate Fix**: Move `aurora_corne_dongle_xiao.keymap` from `config/boards/shields/splitkb_aurora_corne/` to `config/` directory where ZMK looks for user keymaps first.

**Root Cause Fix**: Rename the dongle keymap to match ZMK's base shield name pattern `splitkb_aurora_corne_dongle_xiao.keymap` in the `config/` directory.

---

## Investigation Timeline

### 1. Initial Observation

Build log shows:
```
-- Shield(s): aurora_corne_dongle_xiao
-- ZMK Config Kconfig: config/aurora_corne_dongle_xiao.conf ✅
-- Using keymap file: config/splitkb_aurora_corne.keymap ❌ (WRONG)
-- Found devicetree overlay: config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.overlay ✅
-- Found devicetree overlay: config/splitkb_aurora_corne.keymap
```

**Key Finding**: The dongle overlay is loaded correctly from the shield directory, but the keymap is NOT.

### 2. File Structure Analysis

Current structure:
```
config/
├── splitkb_aurora_corne.keymap          # Main keyboard (has &led_strip) - BEING USED
├── splitkb_aurora_corne.conf
├── aurora_corne_dongle_xiao.conf        # Dongle config - LOADED ✅
└── boards/shields/splitkb_aurora_corne/
    ├── aurora_corne_dongle_xiao.keymap  # Dongle keymap - IGNORED ❌
    ├── aurora_corne_dongle_xiao.overlay # Dongle overlay - LOADED ✅
    ├── Kconfig.shield
    └── Kconfig.defconfig
```

### 3. ZMK Keymap Precedence Rules Discovery

From [ZMK Configuration Documentation](https://zmk.dev/docs/config) and [GitHub Issue #1144](https://github.com/zmkfirmware/zmk/issues/1144):

**Keymap File Search Order**:
1. **First**: `${ZMK_CONFIG}/config/${SHIELD_NAME}.keymap`
2. **Second**: `${ZMK_CONFIG}/config/${SHIELD_BASE_NAME}.keymap` (after suffix stripping)
3. **Third**: Shield directory default keymap (in ZMK firmware source)

**Suffix Stripping Logic**:
- ZMK strips `_left` and `_right` suffixes for split keyboards
- Example: `corne_left` → searches for `corne.keymap`
- Example: `kyria_rev2_left` → searches for `kyria_rev2.keymap` (preserves revision)

**Critical Rule**: Config directory keymaps ALWAYS take precedence over shield directory keymaps.

### 4. Shield Name Analysis

Shield name: `aurora_corne_dongle_xiao`

**Problem Identified**:
- ZMK does NOT recognize `_dongle_xiao` as a strippable suffix (only `_left`/`_right` are known)
- Therefore, ZMK searches for: `config/aurora_corne_dongle_xiao.keymap`
- This file does NOT exist
- ZMK then falls back to checking if the shield directory name matches any keymap

**Shield Directory Name**: `splitkb_aurora_corne`

**Fallback Match**:
- ZMK finds `config/splitkb_aurora_corne.keymap` exists
- This matches the shield directory name
- This keymap is selected even though it's the WRONG one for the dongle

### 5. Why Overlay Works But Keymap Doesn't

**Overlay Files**: Use exact shield name matching
- `aurora_corne_dongle_xiao.overlay` matches shield `aurora_corne_dongle_xiao` ✅

**Keymap Files**: Use base name matching with suffix stripping
- `aurora_corne_dongle_xiao.keymap` searched in `config/` → NOT FOUND
- Falls back to `splitkb_aurora_corne.keymap` based on shield directory name match → WRONG FILE

---

## Root Cause Analysis

### Primary Cause

**Naming Pattern Mismatch**: The dongle keymap file is named `aurora_corne_dongle_xiao.keymap` but ZMK's keymap selection algorithm cannot find it because:

1. It's located in `config/boards/shields/splitkb_aurora_corne/` (shield directory)
2. ZMK looks for user keymaps in `config/` FIRST (and only this location for user configs)
3. Shield directory keymaps are only defaults, always overridden by config directory

### Secondary Contributing Factors

1. **Misleading Directory Structure**: Placing custom keymaps in `config/boards/shields/` suggests they override default shield keymaps, but ZMK's precedence rules don't work this way

2. **Shield Directory Name Confusion**: The shield directory is named `splitkb_aurora_corne`, which matches an existing keymap in `config/`, causing an unintended match

3. **Undocumented Variant Suffix Behavior**: ZMK documentation doesn't clearly explain that only `_left`/`_right` are stripped, not custom variant suffixes like `_dongle_xiao`

### Fishbone Diagram

```
Materials (Files)                     Methods (Process)
    |                                     |
    |-- aurora_corne_dongle_xiao.keymap   |-- ZMK searches config/ first
    |   in WRONG location                 |-- Only strips _left/_right
    |   (shield dir not config/)          |-- Falls back to dir name match
    |                                     |
Technology (System)        →  →  →  WRONG KEYMAP LOADED  ←  ←  ← People (Understanding)
    |                                     |
    |-- Config dir precedence             |-- Unclear suffix stripping rules
    |-- Shield dir name matching          |-- Misleading directory structure
    |-- No custom suffix recognition      |-- Documentation gaps
```

---

## Evidence Documentation

### Build Log Excerpt
```
-- Shield(s): aurora_corne_dongle_xiao
-- ZMK Config Kconfig: /__w/zmk-aurora-corne/zmk-aurora-corne/config/aurora_corne_dongle_xiao.conf
-- Using keymap file: /__w/zmk-aurora-corne/zmk-aurora-corne/config/splitkb_aurora_corne.keymap
-- Found devicetree overlay: /__w/zmk-aurora-corne/zmk-aurora-corne/config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.overlay
-- Found devicetree overlay: /__w/zmk-aurora-corne/zmk-aurora-corne/config/splitkb_aurora_corne.keymap
```

### Main Keyboard Keymap (WRONG file being used)
**File**: `config/splitkb_aurora_corne.keymap`
**Line 12**: `&led_strip { chain-length = <24>; };` ← CAUSES BUILD FAILURE (dongle has no LEDs)

### Dongle Keymap (CORRECT file being ignored)
**File**: `config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap`
**Lines 14-21**: Empty bindings with `&none` for all positions ← CORRECT for dongle (no keys)

### Shield Configuration
**File**: `config/boards/shields/splitkb_aurora_corne/Kconfig.shield`
```kconfig
config SHIELD_AURORA_CORNE_DONGLE_XIAO
	def_bool $(shields_list_contains,aurora_corne_dongle_xiao)
```

### Dongle Config (Correctly Loaded)
**File**: `config/aurora_corne_dongle_xiao.conf`
This file IS loaded correctly, proving that config-level files work when properly named.

---

## Actionable Remediation Plan

### Priority 1: Immediate Fix (Resolve Build Failure)

**Action**: Move dongle keymap to config directory with exact shield name

```bash
mv config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap \
   config/aurora_corne_dongle_xiao.keymap
```

**Why**: This ensures ZMK finds the correct keymap in its primary search location.

**Expected Result**: Build should succeed with dongle keymap (empty bindings, no `&led_strip` reference)

**Risk**: Low - This aligns with ZMK's documented precedence rules

### Priority 2: Long-term Naming Consistency (Prevent Similar Issues)

**Option A - Consistent Base Naming** (Recommended):

Rename to match the shield directory base name pattern:
```bash
mv config/aurora_corne_dongle_xiao.keymap \
   config/splitkb_aurora_corne_dongle_xiao.keymap
```

**Rationale**:
- Keeps `splitkb_aurora_corne` as the base name (matches shield directory)
- Uses `_dongle_xiao` as a variant suffix (like `_left`/`_right`)
- Makes the naming pattern explicit and discoverable

**Option B - Separate Shield Directory** (Alternative):

Create a completely separate shield for the dongle:
```
config/boards/shields/aurora_corne_dongle/
├── aurora_corne_dongle.keymap
├── aurora_corne_dongle.overlay
├── Kconfig.shield
└── Kconfig.defconfig
```

**Rationale**: Treats dongle as a completely different hardware configuration

**Risk**: Higher - Requires more restructuring and testing

### Priority 3: Preventive Measures

1. **Documentation**: Add comments in keymap files explaining the naming requirements
   ```
   /*
    * IMPORTANT: This keymap file must be in config/ directory
    * and named to match the shield name for ZMK to find it.
    * Shield name: aurora_corne_dongle_xiao
    * Keymap location: config/aurora_corne_dongle_xiao.keymap
    */
   ```

2. **Build Validation**: Add a GitHub Actions check to verify keymap files are in correct locations

3. **Naming Convention Standard**: Document the expected pattern for variant shields:
   ```
   Pattern: {vendor}_{keyboard}_{variant}.keymap
   Example: splitkb_aurora_corne_dongle_xiao.keymap
   ```

### Priority 4: Testing Strategy

**Before Fix**:
1. Verify build fails with current structure ✅ (already confirmed)
2. Verify error message mentions `&led_strip` ✅ (already confirmed)

**After Priority 1 Fix**:
1. Build dongle firmware and verify:
   - Build succeeds without `&led_strip` errors
   - Correct keymap file is reported in build log
   - Generated firmware size is reasonable for empty keymap
2. Test on actual dongle hardware:
   - Dongle boots successfully
   - BLE central role functions correctly
   - Can connect to peripherals

**After Priority 2 Fix** (if implemented):
1. Verify new naming doesn't break existing builds
2. Verify both main keyboard and dongle builds work
3. Update CI/CD workflows if necessary

---

## Risk Assessment

**Current Impact**: HIGH
- Dongle firmware cannot be built
- Blocks testing of BLE central functionality
- Potential for wasted debugging time if issue recurs

**Urgency**: HIGH
- Blocking active development
- Simple fix available
- No workarounds possible with current structure

**Likelihood of Recurrence**: MEDIUM
- Similar issues could occur with other shield variants
- ZMK's naming conventions are not well documented
- Easy to misunderstand file precedence rules

**Mitigation Priority**: HIGH
- Implement Priority 1 fix immediately
- Consider Priority 2 for long-term maintainability
- Document findings for future reference

---

## Lessons Learned

1. **ZMK Config Precedence**: Config directory ALWAYS overrides shield directory for keymaps, regardless of file organization

2. **Suffix Stripping Limitations**: ZMK only recognizes `_left`/`_right` as strippable suffixes, not arbitrary variant names

3. **Directory Name Matching**: Shield directory names can influence keymap fallback selection in unexpected ways

4. **Overlay vs Keymap Selection**: Different selection algorithms for overlay files (exact match) vs keymap files (base name with suffix stripping)

5. **Documentation Gaps**: ZMK documentation doesn't clearly explain custom variant shield naming requirements

---

## References

- [ZMK Configuration Overview](https://zmk.dev/docs/config) - Config file precedence rules
- [ZMK Customization Guide](https://zmk.dev/docs/customization) - File organization
- [GitHub Issue #1144](https://github.com/zmkfirmware/zmk/issues/1144) - Shield revision keymap file precedence
- [Zephyr Devicetree Overlays](https://docs.zephyrproject.org/latest/build/dts/howtos.html) - Overlay file precedence

---

## Appendix: Five Whys Analysis

**Problem**: Dongle build uses wrong keymap file

**Why 1**: Why is the wrong keymap selected?
→ Because `config/splitkb_aurora_corne.keymap` exists and matches the shield directory name

**Why 2**: Why does the shield directory name matter for keymap selection?
→ Because ZMK falls back to shield directory name matching when exact shield name keymap is not found in config/

**Why 3**: Why is the exact shield name keymap not found?
→ Because `aurora_corne_dongle_xiao.keymap` is in `config/boards/shields/splitkb_aurora_corne/` instead of `config/`

**Why 4**: Why was the keymap placed in the shield directory instead of config/?
→ Because it was assumed that shield-specific keymaps should live with their shield definitions

**Why 5**: Why was this assumption incorrect?
→ Because ZMK's design separates "default shield keymaps" (in shield dirs) from "user keymaps" (in config/), with user keymaps ALWAYS taking precedence, making shield directory keymaps only defaults that are overridden

**Root Cause**: Misunderstanding of ZMK's two-tier keymap system (defaults vs user configs) and the file precedence rules

---

## Next Steps for Main Agent

1. ✅ **Move keymap file**:
   ```bash
   mv config/boards/shields/splitkb_aurora_corne/aurora_corne_dongle_xiao.keymap \
      config/aurora_corne_dongle_xiao.keymap
   ```

2. ✅ **Verify build**: Run CI build for dongle to confirm fix

3. ⚠️ **Consider renaming** (optional): Evaluate whether `splitkb_aurora_corne_dongle_xiao.keymap` is a better long-term name

4. ⚠️ **Update documentation**: Add comments explaining keymap location requirements

5. ⚠️ **Test on hardware**: Verify dongle functionality with corrected firmware

---

**Analysis Completed**: 2026-01-04
**Analyst**: Root Cause Analysis Agent
**Confidence**: HIGH (supported by official documentation and GitHub issue evidence)
