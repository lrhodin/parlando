---
date: 2025-10-26T20:02:46-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "VST3/CLAP Threading Fix - Plugin Rebuild After Uncommitted Changes"
tags: [bugfix, threading, vst3, clap, nih-plug, panic, atomicrefcell, rebuild, phase-4]
status: complete
last_updated: 2025-10-26
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: VST3/CLAP Threading Fix - Plugins Rebuilt with try_borrow_mut()

## Task(s)

**Status: PLUGINS REBUILT ✅ | AWAITING FINAL TESTING ⏸️**

Resolved a critical issue where the threading fix (using `try_borrow_mut()` instead of `borrow_mut()`) was present in the codebase but NOT compiled into the installed plugins. This caused continued crashes despite the fix being implemented.

**Completed:**
- ✅ Analyzed crash log showing continued SIGABRT after ~2 minutes (previous session achieved ~30 seconds before crash)
- ✅ Verified threading fix was present in source code as uncommitted changes
- ✅ Identified root cause: installed plugins dated Oct 25 14:37, before fix was applied
- ✅ Rebuilt both VST3 and CLAP plugins with current uncommitted changes
- ✅ Reinstalled plugins with timestamps Oct 26 20:00

**Remaining:**
- ⏸️ Test VST3 version in Bitwig Studio for **sustained runtime (>5 minutes)**
- ⏸️ Test CLAP version in Bitwig Studio for **sustained runtime (>5 minutes)**
- ⏸️ Verify track name/color updates work correctly during active audio processing
- ⏸️ If tests pass, commit the threading fix to nih-plug
- ⏸️ Complete remaining tasks from previous handoffs (vst3-sys commits, multi-DAW testing)

**Previous Context:**
This work continues from:
1. `thoughts/shared/handoffs/general/2025-10-25_14-38-49_vst3-clap-threading-crash-fix.md` - Initial threading fix implementation
2. `thoughts/shared/handoffs/general/2025-10-25_14-28-09_vst3-crash-fix-complete.md` - VST3 init crash fix

## Critical References

1. **Previous Handoff (Threading Fix)**: `thoughts/shared/handoffs/general/2025-10-25_14-38-49_vst3-clap-threading-crash-fix.md`
2. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phase 4 (lines 481-704)
3. **Second Previous Handoff (Init Fix)**: `thoughts/shared/handoffs/general/2025-10-25_14-28-09_vst3-crash-fix-complete.md`

## Recent Changes

### Build and Installation (Oct 26 20:00)

**Rebuilt plugins with uncommitted threading fixes:**
- Rebuilt: `/tmp/nih-plug/target/bundled/track_info_display.vst3`
- Rebuilt: `/tmp/nih-plug/target/bundled/track_info_display.clap`
- Installed: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (new timestamp: Oct 26 20:00)
- Installed: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (new timestamp: Oct 26 20:00)

### Uncommitted Changes in nih-plug (`/tmp/nih-plug/`)

**File**: `src/wrapper/vst3/wrapper.rs:1951-1957`
- Threading fix is present but uncommitted:
```rust
// Store the track info - use try_borrow_mut to avoid panicking if audio thread is reading
// If we can't get the lock, just skip this update - the next update will succeed
if let Ok(mut guard) = self.inner.track_info.try_borrow_mut() {
    *guard = Some(track_info);
}
```

**File**: `src/wrapper/clap/wrapper.rs:1839-1843`
- Same threading fix applied to CLAP wrapper's `update_track_info()` function

## Learnings

### Root Cause: Stale Binaries

**Critical Discovery**: The threading fix was implemented correctly in the source code but was never compiled into the installed plugins. This explains why:
1. Initial crash happened after ~30 seconds
2. Second test (with supposedly "fixed" plugins) crashed after ~2 minutes
3. The fix was in the code all along as uncommitted changes

**Timeline of Confusion**:
- Oct 24: Threading fix implemented (as uncommitted changes)
- Oct 25 14:37: Plugins built and installed (WITHOUT the fix)
- Oct 25 14:38: Handoff created describing fix as "complete"
- Oct 26 19:57: User reports crash still happening after 2 minutes
- Oct 26 20:00: Realized plugins were stale, rebuilt with actual fix

**Lesson**: Always verify `git diff` and binary timestamps when debugging issues that "should be fixed"

### The try_borrow_mut() Pattern (Review)

The fix that's NOW actually in the binaries:
```rust
// ❌ WRONG - panics if read borrow active
*self.inner.track_info.borrow_mut() = Some(value);

// ✅ CORRECT - gracefully skips if read borrow active
if let Ok(mut guard) = self.inner.track_info.try_borrow_mut() {
    *guard = Some(value);
}
```

This is safe because track info updates are not critical - skipping one update when the audio thread is busy is acceptable.

### Why It Should Work Now

The audio thread holds an `AtomicRef` for the entire process callback duration (`src/wrapper/vst3/context.rs:46`). When track info updates arrive on the UI/main thread, `try_borrow_mut()` will:
1. Return `Ok(guard)` if audio thread is not processing → update succeeds
2. Return `Err` if audio thread is busy → update skipped gracefully, no panic

## Artifacts

### Modified Repositories

**nih-plug Fork** (`/tmp/nih-plug/` on branch `feature/track-info-support`):
- `src/wrapper/vst3/wrapper.rs:1951-1957` - VST3 threading fix (UNCOMMITTED)
- `src/wrapper/clap/wrapper.rs:1839-1843` - CLAP threading fix (UNCOMMITTED)
- Last commit: `79e0fdb1` (Phase 4 complete)
- Status: Threading fixes need to be committed after successful testing

**Built and Installed Plugins** (Oct 26 20:00):
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` ← NOW contains fix
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` ← NOW contains fix

**Previous Artifacts** (from earlier handoffs):
- vst3-sys fork at `/tmp/vst3-sys-temp/` with IInfoListener signature fix (needs commit)
- Implementation plan: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`
- Research document: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md`

## Action Items & Next Steps

### Immediate: Test the Actually-Fixed Plugins

**CRITICAL**: The plugins installed on Oct 26 20:00 are the FIRST builds that actually contain the threading fix.

1. **Test VST3 in Bitwig Studio**:
   - Load track_info_display.vst3
   - Play audio for **at least 5 minutes** (previous tests: 30sec → 2min → crash)
   - Change track name and color **multiple times** during playback
   - Verify: No crashes occur, updates appear correctly

2. **Test CLAP in Bitwig Studio**:
   - Same procedure as VST3
   - CLAP was the version shown in the crash logs

3. **Stress test** (if 5-minute test passes):
   - Multiple rapid track name/color changes during heavy audio processing
   - Leave running for 15+ minutes
   - Verify no panics or instability

### After Successful Testing

4. **Commit the threading fix to nih-plug**:
   ```bash
   cd /tmp/nih-plug
   git add src/wrapper/vst3/wrapper.rs src/wrapper/clap/wrapper.rs
   git commit -m "Fix threading panic in track info updates

   Changed both VST3 and CLAP wrappers to use try_borrow_mut() instead of
   borrow_mut() when updating track info. This prevents panic when the audio
   thread holds a read borrow during ProcessContext creation.

   The audio thread borrows track_info for the entire process callback, so
   UI thread updates must use try_borrow_mut() to avoid AtomicRefCell panic.
   Skipping an update is acceptable since the next update will succeed.

   Fixes crash: SIGABRT after variable runtime (30sec-2min) when host sends
   track info updates during active audio processing."

   git push origin feature/track-info-support
   ```

5. **Complete vst3-sys commits** (from Oct 25 handoff):
   - Commit and push vst3-sys IInfoListener signature fix
   - Update nih-plug Cargo.toml to use vst3-sys fork instead of local path

6. **Test in additional DAWs**:
   - Logic Pro (macOS)
   - Ableton Live (macOS)

7. **Update Parlando**:
   - Rebuild Parlando plugin with fixed nih-plug dependency
   - Test track info support in Parlando

### If Testing Reveals Issues

If crashes still occur despite the rebuild:
- Check if there are additional `borrow_mut()` calls on `track_info` that need the try_borrow pattern
- Look for other `AtomicRefCell` instances that might have similar cross-thread access patterns
- Review the ProcessContext creation and destruction timing

## Other Notes

### Repository Status

**nih-plug Fork**:
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Uncommitted changes: VST3 and CLAP threading fixes (MUST commit after testing)

**vst3-sys Fork**:
- Local: `/tmp/vst3-sys-temp/`
- Remote: Need to push to github.com/lrhodin/vst3-sys
- Branch: `fix/drop-box-from-raw`
- Uncommitted changes: IInfoListener signature fix (from Oct 25 handoff)

**Parlando**:
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Uses nih-plug fork via `Cargo.toml:19`
- Current commit: `11fc8dd`
- No changes needed until nih-plug fix is tested and committed

### Build Commands for Reference

```bash
# Rebuild example plugins
cd /tmp/nih-plug
cargo xtask bundle track_info_display --release

# Install plugins
cp -r target/bundled/track_info_display.vst3 ~/Library/Audio/Plug-Ins/VST3/
cp -r target/bundled/track_info_display.clap ~/Library/Audio/Plug-Ins/CLAP/

# Verify timestamps
ls -la ~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3/Contents/MacOS/track_info_display
ls -la ~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap/Contents/MacOS/track_info_display
```

### Important Reminders

1. **Test thoroughly** - This was a subtle timing bug, extended testing is critical
2. **Verify timestamps** - Always check binary modification times match expected build times
3. **Both wrappers fixed** - VST3 and CLAP had identical issue, both are now fixed
4. **Uncommitted changes** - Don't forget to commit after successful testing
5. **Previous handoff tasks remain** - Still need vst3-sys commits and multi-DAW testing

### Threading Model (Review)

NIH-plug's ProcessContext pattern (why try_borrow_mut is necessary):
- Context is created at start of each audio callback (`src/wrapper/vst3/inner.rs:380-388`)
- Holds immutable borrows (`AtomicRef`) of shared state for entire callback
- This includes: `input_events`, `output_events`, `transport`, `track_info`
- Any cross-thread updates to these shared resources MUST use `try_borrow_mut()`

### Expected Test Outcome

With properly compiled binaries containing the fix:
- Plugins should run indefinitely without crashes
- Track info updates should work smoothly
- Occasional missed updates (when audio thread is busy) are invisible to user
- System should be stable under sustained operation and rapid changes
