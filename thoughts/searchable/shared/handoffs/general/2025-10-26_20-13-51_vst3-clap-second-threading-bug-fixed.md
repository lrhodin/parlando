---
date: 2025-10-26T20:13:51-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "VST3/CLAP Threading Fix - Second Bug in Example Plugin Fixed"
tags: [bugfix, threading, vst3, clap, nih-plug, panic, atomicrefcell, gui, phase-4]
status: complete
last_updated: 2025-10-26
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: VST3/CLAP Threading Fix - Second Bug Discovered and Fixed

## Task(s)

**Status: TWO THREADING BUGS FIXED ✅ | PLUGINS REBUILT ✅ | AWAITING FINAL TESTING ⏸️**

Discovered and fixed a SECOND critical threading bug after the previous handoff's fix failed to prevent crashes. The plugin crashed 96 seconds after the previous rebuild when the user changed the track name.

**Completed:**
- ✅ Analyzed new crash log showing SIGABRT on main thread via timer callback
- ✅ Identified Bug #1 (wrapper) was already fixed with `try_borrow_mut()`
- ✅ Discovered Bug #2: Example plugin's `process()` function used `borrow_mut()` on GUI-shared state
- ✅ Fixed example plugin to use `try_borrow()` / `try_borrow_mut()` for all GUI-shared `AtomicRefCell` access
- ✅ Rebuilt and reinstalled both VST3 and CLAP plugins with BOTH fixes (Oct 26 20:10)

**Remaining:**
- ⏸️ Test VST3 version alone for sustained runtime (>5 minutes)
- ⏸️ Test CLAP version alone for sustained runtime (>5 minutes)
- ⏸️ Verify track name/color updates work correctly without crashes
- ⏸️ If tests pass, commit both threading fixes to nih-plug
- ⏸️ Complete remaining tasks from original handoff (vst3-sys commits, multi-DAW testing)

**Previous Context:**
This work continues from:
1. `thoughts/shared/handoffs/general/2025-10-26_20-02-46_vst3-clap-threading-fix-rebuild.md` - Previous rebuild (only Bug #1 fixed)
2. `thoughts/shared/handoffs/general/2025-10-25_14-38-49_vst3-clap-threading-crash-fix.md` - Initial threading fix (Bug #1)
3. `thoughts/shared/handoffs/general/2025-10-25_14-28-09_vst3-crash-fix-complete.md` - VST3 init crash fix

## Critical References

1. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-26_20-02-46_vst3-clap-threading-fix-rebuild.md`
2. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phase 4 (lines 481-704)
3. **First Threading Fix Handoff**: `thoughts/shared/handoffs/general/2025-10-25_14-38-49_vst3-clap-threading-crash-fix.md`

## Recent Changes

### nih-plug Example Plugin (`/tmp/nih-plug/`)

**File**: `plugins/examples/track_info_display/src/lib.rs:253-264`
- Changed track name update from `borrow()` and `borrow_mut()` to `try_borrow()` and `try_borrow_mut()`
- Added explicit lock dropping before acquiring write lock

**File**: `plugins/examples/track_info_display/src/lib.rs:267-278`
- Changed track color update from `borrow_mut()` to `try_borrow_mut()`

**File**: `plugins/examples/track_info_display/src/lib.rs:315-332`
- Changed reset logic from `borrow()` and `borrow_mut()` to `try_borrow()` and `try_borrow_mut()`

### Rebuilt and Installed Plugins (Oct 26 20:10)

- Rebuilt: `/tmp/nih-plug/target/bundled/track_info_display.vst3`
- Rebuilt: `/tmp/nih-plug/target/bundled/track_info_display.clap`
- Installed: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (timestamp: Oct 26 20:10)
- Installed: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (timestamp: Oct 26 20:10)

## Learnings

### Two Separate Threading Bugs

**Bug #1: NIH-plug Wrapper** (Fixed Oct 25, present in Oct 26 20:00 build):
- **Location**: `src/wrapper/vst3/wrapper.rs:1953` and `src/wrapper/clap/wrapper.rs:1841`
- **Issue**: Wrapper used `borrow_mut()` to store track info from host while audio thread held immutable borrow
- **Fix**: Changed to `try_borrow_mut()` to gracefully skip updates when lock unavailable

**Bug #2: Example Plugin GUI State** (Fixed Oct 26, NEW in this session):
- **Location**: `plugins/examples/track_info_display/src/lib.rs:256, 263, 305-306`
- **Issue**: Plugin's `process()` function (audio thread) called `borrow_mut()` on GUI-shared state
- **Root Cause**: GUI timer callbacks on main thread tried to read `track_name` and `track_color` while audio thread held write lock
- **Evidence**: Crash stack showed `__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__` on Thread 0 (main thread)
- **Fix**: Changed ALL `borrow()` and `borrow_mut()` calls in `process()` to `try_borrow()` and `try_borrow_mut()`

### Why Previous Fix Didn't Work

The Oct 26 20:00 rebuild only contained Bug #1 fix (wrapper). Plugin crashed at 20:01:36 (96 seconds later) because:
1. User loaded BOTH VST3 and CLAP on same track
2. User changed track name
3. GUI timer callback tried to read `track_name` (main thread)
4. Audio thread held `borrow_mut()` lock on `track_name`
5. `AtomicRefCell` panic → SIGABRT

### The try_borrow Pattern for GUI-Shared State

Example plugin now follows correct cross-thread pattern:

```rust
// WRONG - panics if GUI thread is reading:
*self.track_name.borrow_mut() = name.clone();

// CORRECT - gracefully skips if GUI thread is reading:
if let Ok(current_name_guard) = self.track_name.try_borrow() {
    if *current_name_guard != *name {
        drop(current_name_guard); // Release read lock
        if let Ok(mut guard) = self.track_name.try_borrow_mut() {
            *guard = name.clone();
        }
    }
}
```

This is safe because:
- Skipping an update when GUI is reading is acceptable
- Next update (few milliseconds later) will succeed
- GUI gets slightly stale data briefly, which is imperceptible

## Artifacts

### Modified Files (uncommitted)

**nih-plug Repository** (`/tmp/nih-plug/` on branch `feature/track-info-support`):
- `src/wrapper/vst3/wrapper.rs:1951-1955` - VST3 wrapper threading fix (Bug #1)
- `src/wrapper/clap/wrapper.rs:1839-1843` - CLAP wrapper threading fix (Bug #1)
- `plugins/examples/track_info_display/src/lib.rs:253-332` - Example plugin GUI state threading fix (Bug #2)
- Last commit: `79e0fdb1` (Phase 4 complete, before threading fixes)
- Status: All fixes need to be committed after successful testing

### Built and Installed Plugins (Oct 26 20:10)

**Now contain BOTH Bug #1 and Bug #2 fixes:**
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3`
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap`
- Binary size: 7,754,032 bytes (slightly larger than previous build)

### Previous Artifacts (from earlier handoffs)

- vst3-sys fork at `/tmp/vst3-sys-temp/` with IInfoListener signature fix (needs commit)
- Implementation plan: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`
- Research document: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md`

## Action Items & Next Steps

### Immediate: Test the Fully-Fixed Plugins

**CRITICAL**: The plugins installed Oct 26 20:10 are the FIRST builds that contain BOTH threading fixes.

**IMPORTANT**: Test with only ONE plugin instance at a time to isolate any remaining issues:

1. **Test VST3 alone**:
   - Load ONLY track_info_display.vst3 (not CLAP)
   - Play audio for at least 5 minutes
   - Change track name and color multiple times during playback
   - Verify: No crashes, updates appear correctly

2. **Test CLAP alone**:
   - Remove VST3, load ONLY track_info_display.clap
   - Same procedure as VST3
   - CLAP was the version in both crash logs

3. **If both pass individually**, then test with both loaded simultaneously

4. **Stress test** (if 5-minute tests pass):
   - Multiple rapid track name/color changes during heavy audio processing
   - Leave running for 15+ minutes
   - Verify no panics or instability

### After Successful Testing

5. **Commit BOTH threading fixes to nih-plug**:
   ```bash
   cd /tmp/nih-plug
   git add src/wrapper/vst3/wrapper.rs src/wrapper/clap/wrapper.rs plugins/examples/track_info_display/src/lib.rs
   git commit -m "Fix threading panics in track info implementation

   Fixed two separate threading bugs that caused AtomicRefCell panics:

   1. Wrapper bug: Changed track info storage in VST3 and CLAP wrappers to
      use try_borrow_mut() instead of borrow_mut(). This prevents panic when
      audio thread holds a read borrow during ProcessContext creation.

   2. Example plugin bug: Changed track_info_display example plugin to use
      try_borrow() and try_borrow_mut() for all GUI-shared state (track_name,
      track_color). This prevents panic when GUI timer callbacks on main thread
      access the same AtomicRefCell while audio thread is updating values.

   Both bugs manifested as SIGABRT crashes after variable runtime (30sec-2min)
   when host sent track info updates during active audio processing. Skipping
   an update when locks are unavailable is acceptable since the next update
   will succeed."

   git push origin feature/track-info-support
   ```

6. **Complete vst3-sys commits** (from Oct 25 handoff):
   - Commit and push vst3-sys IInfoListener signature fix
   - Update nih-plug Cargo.toml to use vst3-sys fork instead of local path

7. **Test in additional DAWs**:
   - Logic Pro (macOS)
   - Ableton Live (macOS)

8. **Update Parlando**:
   - Rebuild Parlando plugin with fixed nih-plug dependency
   - Test track info support in Parlando

### If Testing Reveals Issues

If crashes still occur despite both fixes:
- Examine crash logs to identify which thread and which variable is involved
- Search for other `borrow_mut()` calls on shared `AtomicRefCell` instances
- Consider if there are other cross-thread access patterns not yet identified
- Check if the crash is related to a different component entirely

## Other Notes

### Repository Status

**nih-plug Fork**:
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Uncommitted changes: VST3 wrapper, CLAP wrapper, and example plugin threading fixes
- These MUST be committed together after successful testing

**vst3-sys Fork**:
- Local: `/tmp/vst3-sys-temp/`
- Remote: Need to push to github.com/lrhodin/vst3-sys
- Branch: `fix/drop-box-from-raw`
- Uncommitted changes: IInfoListener signature fix (from Oct 25 handoff)

**Parlando**:
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Uses nih-plug fork via `Cargo.toml:19`
- Current commit: `11fc8dd`
- No changes needed until nih-plug fixes are tested and committed

### Build Commands for Reference

```bash
# Rebuild example plugin
cd /tmp/nih-plug
cargo xtask bundle track_info_display --release

# Install plugins
cp -r target/bundled/track_info_display.vst3 ~/Library/Audio/Plug-Ins/VST3/
cp -r target/bundled/track_info_display.clap ~/Library/Audio/Plug-Ins/CLAP/

# Verify timestamps (should be Oct 26 20:10 or later)
ls -la ~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3/Contents/MacOS/track_info_display
ls -la ~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap/Contents/MacOS/track_info_display
```

### Why Both Bugs Were Necessary to Fix

The threading issue has two access patterns that both need `try_borrow_mut()`:

1. **Host → Wrapper → Plugin** (Bug #1):
   - Host sends track info update (main/UI thread)
   - Wrapper stores in `self.inner.track_info`
   - Audio thread borrows for ProcessContext
   - Race condition if update arrives during audio callback

2. **Plugin → GUI** (Bug #2):
   - Plugin updates GUI-shared state in `process()` (audio thread)
   - GUI timer callbacks read state for display (main thread)
   - Race condition if timer fires during process callback

Both patterns created `AtomicRefCell` panics but manifested differently:
- Bug #1: Crashed when host sent updates during audio processing
- Bug #2: Crashed when GUI timer tried to update display during audio processing

### Testing History

- Oct 25 14:37: Built without any fixes → crashed after 30 seconds
- Oct 26 20:00: Built with Bug #1 fix only → crashed after 96 seconds
- Oct 26 20:10: Built with BOTH fixes → awaiting testing

### Important Reminders

1. **Test with single instances first** - Loading both VST3 and CLAP may complicate debugging
2. **Both fixes are required** - Either bug alone can cause crashes
3. **GUI timer callbacks are the trigger** - Bug #2 only manifests when GUI is active
4. **Uncommitted changes span 3 files** - Don't forget any when committing
5. **Previous handoff tasks remain** - Still need vst3-sys commits and multi-DAW testing
