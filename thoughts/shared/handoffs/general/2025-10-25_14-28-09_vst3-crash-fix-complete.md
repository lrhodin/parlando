---
date: 2025-10-25T14:28:09-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "VST3 Crash Fix Complete - IInfoListener Pointer Type Corrected"
tags: [implementation, vst3, nih-plug, crash-fix, iinfolistener, phase-4]
status: complete
last_updated: 2025-10-25
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: VST3 Crash Fix Complete - Track Name and Color Working

## Task(s)

**Status: VST3 CRASH FIXED ✅ | TRACK NAME & COLOR WORKING ✅ | TRACK TYPE LIMITATION IDENTIFIED ⚠️**

Implemented the fix for the VST3 crash identified in the previous handoff. The plugin now loads successfully in Bitwig Studio and correctly displays track name and color information.

**Completed:**
- ✅ Fixed IInfoListener pointer type in both vst3-sys and nih-plug
- ✅ Corrected IAttributeList key names to match VST3 spec
- ✅ Plugin loads without crashing in Bitwig Studio
- ✅ Track name displays and updates correctly
- ✅ Track color displays and updates correctly

**Identified Limitation:**
- ⚠️ VST3 IInfoListener spec does NOT provide track type flags (master/bus/return)
- ⚠️ Track type flags only work in CLAP, not VST3 (spec limitation, not implementation bug)

**Remaining:**
- ⏸️ Remove debug logging from production code
- ⏸️ Commit and push fixes to both repositories
- ⏸️ Update Cargo.toml to use proper vst3-sys fork (currently using local path)
- ⏸️ Test in additional DAWs (Logic Pro, Ableton Live)

**Previous Context:**
Starting from handoff at `thoughts/shared/handoffs/general/2025-10-25_10-08-22_vst3-crash-root-cause-identified.md` which identified the root cause but had not yet implemented the fix.

## Critical References

1. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-25_10-08-22_vst3-crash-root-cause-identified.md` - Root cause analysis
2. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phase 4 (lines 481-704)
3. **VST3 Spec**: vst3-sys `ivstchannelcontextinfo.rs` defines IInfoListener interface and key constants

## Recent Changes

### vst3-sys Repository (`/tmp/vst3-sys-temp/`)

**File**: `src/vst/ivstchannelcontextinfo.rs:1-11`
- Changed `IInfoListener::set_channel_context_infos` signature from `list: *mut c_void` to `list: SharedVstPtr<dyn IAttributeList>`
- Added imports: `SharedVstPtr`, `IAttributeList`
- Removed unused `c_void` import

### nih-plug Repository (`/tmp/nih-plug/`)

**File**: `src/wrapper/vst3/wrapper.rs:1900-1964`
- Updated `IInfoListener` implementation to use `SharedVstPtr<dyn IAttributeList>` parameter type
- Replaced unsafe pointer casts with `check_null_ptr!` and `.upgrade().unwrap()` pattern
- Fixed IAttributeList key names:
  - Was: `c"Steinberg.Vst.ChannelContext.ChannelName"` ❌
  - Now: `b"channel name\0".as_ptr() as *const _` ✅
- Applied correct keys for: `"channel name"`, `"channel color"`, `"channel index"`, `"channel uid"`
- Added debug logging (needs removal before final commit)

**File**: `Cargo.toml:117`
- Temporarily changed vst3-sys dependency to local path: `{ path = "/tmp/vst3-sys-temp", optional = true }`
- **Action needed**: Update to use forked repository before final commit

## Learnings

### Root Cause: Dual Pointer Type Bug

**Location**: `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:1900-1907` (original buggy code)

The crash was caused by TWO separate issues that both needed fixing:

1. **Incorrect Trait Signature** (vst3-sys):
   - The `IInfoListener` trait defined in vst3-sys used `*mut c_void` parameter type
   - This created a mismatch with VST3's COM calling convention

2. **Incorrect Implementation** (nih-plug):
   - Implementation used double pointer cast: `list as *const *const dyn IAttributeList`
   - Created invalid fat pointer with vtable pointing to 0x59
   - Result: `EXC_BAD_ACCESS (SIGSEGV)` on first dereference

**The Fix Pattern** (used throughout VST3 wrapper at lines 414, 501, 506):
```rust
unsafe fn method(&self, param: SharedVstPtr<dyn Interface>) -> tresult {
    check_null_ptr!(param);
    let param = param.upgrade().unwrap();
    // ... use param safely
}
```

### IAttributeList Key Names

**Critical Discovery**: VST3 uses simple key strings, NOT fully-qualified names.

**Wrong** (what we initially tried):
- `"Steinberg.Vst.ChannelContext.ChannelName"`
- `"Steinberg.Vst.ChannelContext.ChannelColor"`

**Correct** (from VST3 spec):
- `"channel name"`
- `"channel color"`
- `"channel index"`
- `"channel uid"`

These are defined as constants in `vst3-sys/src/vst/ivstchannelcontextinfo.rs:37-49` but we use inline byte strings to avoid dependency issues.

### VST3 Track Type Limitation

**Important**: VST3's `IInfoListener` interface does NOT provide track type information (master/bus/return flags).

**What VST3 IInfoListener provides**:
- ✅ Track name (`"channel name"`)
- ✅ Track color (`"channel color"` as ARGB int)
- ✅ Track index (`"channel index"`)
- ✅ Track UID (`"channel uid"`)
- ✅ Channel namespace (`"channel index namespace"`) - but no standard values
- ❌ Track type flags (master/bus/return) - **not in spec**

**What CLAP provides that VST3 doesn't**:
- `CLAP_TRACK_INFO_IS_FOR_MASTER`
- `CLAP_TRACK_INFO_IS_FOR_BUS`
- `CLAP_TRACK_INFO_IS_FOR_RETURN_TRACK`

**Attempted Solution**: Tried extracting `"channel index namespace"` to infer track type, but this caused crashes (possibly due to string allocation in callback). Reverted this change.

**Conclusion**: Track type flags will only work in CLAP hosts, not VST3. This is expected and correct behavior.

### Why CLAP Works But VST3 Didn't

- **CLAP**: Uses direct function pointers, simpler calling convention
- **VST3**: Uses COM interfaces with vtable indirection - pointer types matter critically
- The pointer type bug only manifested in VST3, not CLAP

## Artifacts

### Modified Repositories

**vst3-sys Fork** (`/tmp/vst3-sys-temp/`):
- Branch: `fix/drop-box-from-raw` (was already checked out)
- Modified: `src/vst/ivstchannelcontextinfo.rs`
- Status: Built successfully, needs commit and push
- Remote: Should fork to github.com/lrhodin/vst3-sys or submit PR to upstream

**nih-plug Fork** (`/tmp/nih-plug/`):
- Branch: `feature/track-info-support`
- Commit: `79e0fdb1` (Phase 4 complete - now with VST3 fix)
- Modified: `src/wrapper/vst3/wrapper.rs`, `Cargo.toml`
- Status: Built successfully, needs final cleanup and commit
- Remote: github.com/lrhodin/nih-plug

### Built Artifacts

**Installed Plugins**:
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (working)
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (working)

**Build Location**:
- Source: `/tmp/nih-plug/target/bundled/`
- Built with: `cargo xtask bundle track_info_display --release`

### Documentation

**Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-25_10-08-22_vst3-crash-root-cause-identified.md`
- Contains detailed crash analysis and root cause identification
- Includes crash logs and debugging steps
- Reference for understanding the bug history

**Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`
- Phase 4: Example Plugin and Testing (lines 481-704)
- Success criteria and testing procedures

**Research Document**: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md`
- Background on VST3 IInfoListener and CLAP track-info extension
- NIH-plug architecture patterns

## Action Items & Next Steps

### Immediate: Clean Up and Commit

1. **Remove debug logging** from `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs`:
   - Line 1904: Remove `nih_log!("VST3 IInfoListener::set_channel_context_infos called!");`
   - Lines 1956-1961: Remove or simplify verbose debug log

2. **Update Cargo.toml** at `/tmp/nih-plug/Cargo.toml:117`:
   ```toml
   # Current (temporary):
   vst3-sys = { path = "/tmp/vst3-sys-temp", optional = true }

   # Should be (after vst3-sys is pushed):
   vst3-sys = { git = "https://github.com/lrhodin/vst3-sys.git", branch = "fix/iinfolistener-pointer-type", optional = true }
   ```

3. **Commit vst3-sys changes**:
   ```bash
   cd /tmp/vst3-sys-temp
   git add src/vst/ivstchannelcontextinfo.rs
   git commit -m "Fix IInfoListener signature to use SharedVstPtr<dyn IAttributeList>

   The IInfoListener::set_channel_context_infos method was using an incorrect
   parameter type (*mut c_void) which caused crashes when the VST3 host called
   this method. The correct type is SharedVstPtr<dyn IAttributeList>, following
   the same pattern used by other VST3 COM interfaces like IComponent::set_state.

   This fixes a segfault at address 0x59 caused by invalid fat pointer dereferencing."

   git push origin fix/drop-box-from-raw
   # OR: Create new branch if fix/drop-box-from-raw shouldn't be modified
   ```

4. **Commit nih-plug changes**:
   ```bash
   cd /tmp/nih-plug
   git add src/wrapper/vst3/wrapper.rs Cargo.toml
   git commit -m "Fix VST3 IInfoListener implementation and IAttributeList key names

   Fixed two issues preventing VST3 track info from working:

   1. Updated IInfoListener::set_channel_context_infos to use correct parameter
      type (SharedVstPtr<dyn IAttributeList>) matching the vst3-sys trait fix

   2. Corrected IAttributeList key names to match VST3 spec:
      - Use simple keys like 'channel name' not 'Steinberg.Vst.ChannelContext.ChannelName'
      - Fixes: channel name, channel color, channel index, channel uid

   Plugin now successfully loads in Bitwig Studio and displays track name and
   color information. Track type flags (master/bus/return) are not available
   in VST3 IInfoListener spec - these only work in CLAP.

   Requires vst3-sys fork with IInfoListener signature fix."

   git push origin feature/track-info-support
   ```

### Testing in Multiple DAWs

After committing, test the VST3 plugin in:

1. **Bitwig Studio** ✅ (already tested - working)
   - Track name updates ✅
   - Track color updates ✅
   - No crash ✅

2. **Logic Pro** (macOS)
   - Verify IInfoListener receives track info
   - Test track name and color updates
   - Verify no crashes

3. **Ableton Live** (macOS)
   - Same verification as Logic Pro
   - Test on different track types

4. **CLAP Version** (for comparison)
   - Verify track type flags work correctly in CLAP
   - Document which hosts support CLAP track-info extension

### Subsequent Steps

1. **Update Parlando** (main repository):
   - The dependency at `Cargo.toml:19` already points to the fork
   - After nih-plug fix is merged, rebuild Parlando
   - Test Parlando plugin with track info support

2. **Complete Phase 4**:
   - Mark Phase 4 testing complete in plan document
   - Document VST3 track type limitation (not a bug, spec limitation)
   - Update success criteria to reflect VST3 vs CLAP differences

3. **Phase 5**: Documentation and PR submission
   - Write comprehensive PR description for upstream nih-plug
   - Reference VST3 and CLAP specifications
   - Include testing results from multiple DAWs
   - Consider submitting vst3-sys fix as separate PR to upstream vst3-sys

4. **Phase 6**: Parlando integration
   - Implement auto musician naming using track info
   - Use CLAP for full track type support if needed

## Other Notes

### Repository Locations

**vst3-sys Fork**:
- Local: `/tmp/vst3-sys-temp/`
- Remote: Need to push to github.com/lrhodin/vst3-sys (or submit PR to upstream)
- Branch: `fix/drop-box-from-raw` (existing branch, or create new branch)
- Modified file: `src/vst/ivstchannelcontextinfo.rs`

**nih-plug Fork**:
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Commit: `79e0fdb1` (will change after new commit)

**Parlando Main Repository**:
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Uses nih-plug fork via `Cargo.toml:19`
- Current commit: `11fc8dd`
- No changes needed until nih-plug fix is complete

### Testing Commands

```bash
# Build example plugin
cd /tmp/nih-plug
cargo xtask bundle track_info_display --release

# Install plugins
cp -r target/bundled/track_info_display.vst3 ~/Library/Audio/Plug-Ins/VST3/
cp -r target/bundled/track_info_display.clap ~/Library/Audio/Plug-Ins/CLAP/

# Run unit tests
cargo test track_info --lib

# Check formatting
cargo fmt --check

# Run clippy
cargo clippy --all-features --workspace
```

### Console Output Verification

When plugin loads successfully, console should show:
```
[INFO] VST3 IInfoListener::set_channel_context_infos called!
[INFO] VST3 Track info stored - name: Some("Audio 1"), color: Some((r, g, b, a)), index: Some(0)
```

When track name/color is changed, the logs update accordingly.

### Known Good Patterns in Codebase

Reference these VST3 wrapper patterns for similar implementations:
- `wrapper.rs:414-470` - `set_state()` and `get_state()` with `SharedVstPtr<dyn IBStream>`
- `wrapper.rs:501-516` - IEditController methods with SharedVstPtr
- `wrapper.rs:688-691` - `set_component_handler()` with upgrade pattern

All VST3 COM interface methods follow this pattern - `IInfoListener` should too.

### Important Reminders

1. **VST3 track type limitation is NOT a bug** - it's a spec limitation
2. **Track type flags work in CLAP** - test CLAP version to verify this
3. **Debug logging must be removed** before final commit
4. **Both repositories need to be committed** - vst3-sys first, then nih-plug
5. **Cargo.toml path must be updated** to use git dependency instead of local path
