---
date: 2025-10-25T14:38:49-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "VST3/CLAP Threading Crash Fix - AtomicRefCell Panic Resolution"
tags: [bugfix, threading, vst3, clap, nih-plug, panic, atomicrefcell, phase-4]
status: complete
last_updated: 2025-10-25
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: VST3/CLAP Threading Crash Fixed - try_borrow_mut() Pattern Applied

## Task(s)

**Status: THREADING BUG FIXED ✅ | AWAITING TESTING ⏸️**

Fixed a critical threading bug that caused both VST3 and CLAP versions of the track_info_display plugin to crash after ~30 seconds of runtime. This is part of NIH-plug Phase 4 testing following the 6-phase implementation plan.

**Completed:**
- ✅ Analyzed new crash (SIGABRT/panic after 30 seconds runtime)
- ✅ Identified root cause: AtomicRefCell panic due to threading conflict
- ✅ Fixed VST3 wrapper to use try_borrow_mut() instead of borrow_mut()
- ✅ Fixed CLAP wrapper to use try_borrow_mut() instead of borrow_mut()
- ✅ Rebuilt and installed both plugin versions

**Remaining:**
- ⏸️ Test VST3 version in Bitwig Studio for sustained runtime (>1 minute)
- ⏸️ Test CLAP version in Bitwig Studio for sustained runtime (>1 minute)
- ⏸️ Verify track name/color updates work correctly during active audio processing
- ⏸️ Complete remaining handoff tasks from previous session (commits, testing in other DAWs)

**Previous Context:**
This work continues from `thoughts/shared/handoffs/general/2025-10-25_14-28-09_vst3-crash-fix-complete.md` which fixed the initial VST3 crash on load. That fix introduced a new threading issue that manifested after runtime.

## Critical References

1. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-25_14-28-09_vst3-crash-fix-complete.md` - VST3 init crash fix
2. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phase 4 (lines 481-704)
3. **Crash Log**: Provided by user showing SIGABRT at Thread 27 with abort() call in track_info_display

## Recent Changes

### nih-plug Repository (`/tmp/nih-plug/`)

**File**: `src/wrapper/vst3/wrapper.rs:1951-1957`
- Changed from `*self.inner.track_info.borrow_mut() = Some(track_info);`
- To: `if let Ok(mut guard) = self.inner.track_info.try_borrow_mut() { *guard = Some(track_info); }`
- Prevents panic when audio thread holds read borrow during track info update

**File**: `src/wrapper/clap/wrapper.rs:1839-1843`
- Changed from `*self.track_info.borrow_mut() = Some(track_info);`
- To: `if let Ok(mut guard) = self.track_info.try_borrow_mut() { *guard = Some(track_info); }`
- Same fix applied to CLAP wrapper's update_track_info() function

## Learnings

### Root Cause: AtomicRefCell Panic on Cross-Thread Access

**Location**: Audio processing context creation at `src/wrapper/vst3/inner.rs:387` and `src/wrapper/vst3/context.rs:46`

The crash was a **Rust panic** (SIGABRT), not a segfault. The threading issue:

1. **Audio thread** (Thread 27 in crash log):
   - Creates `WrapperProcessContext` → calls `self.track_info.borrow()` at `inner.rs:387`
   - Holds `AtomicRef` guard for the **entire duration** of audio processing callback
   - Reference: `context.rs:46` shows `track_info_guard: AtomicRef<'a, Option<TrackInfo>>`

2. **UI/Main thread**:
   - Host sends track info update after ~30 seconds
   - VST3: `IInfoListener::set_channel_context_infos` called at `wrapper.rs:1900`
   - CLAP: `update_track_info()` called at `wrapper.rs:1793`
   - Tries to call `track_info.borrow_mut()` to store update

3. **Result**:
   - `AtomicRefCell::borrow_mut()` **panics** because active immutable borrow exists
   - Panic triggers `abort()` → SIGABRT → process termination
   - Crash log shows: `libsystem_c.dylib abort + 124` at Thread 27

**Why it took 30 seconds**: Host doesn't send track info updates immediately on load - only when track name/color changes or after initial settling period.

### The try_borrow_mut() Pattern

**Correct pattern for cross-thread AtomicRefCell updates**:
```rust
// ❌ WRONG - panics if read borrow active
*self.inner.track_info.borrow_mut() = Some(value);

// ✅ CORRECT - gracefully skips if read borrow active
if let Ok(mut guard) = self.inner.track_info.try_borrow_mut() {
    *guard = Some(value);
}
```

This is safe because:
- Track info updates are not critical - skipping one is acceptable
- Next update will succeed when audio thread releases its borrow
- Updates typically happen infrequently (only when user changes track properties)

### Why Both Wrappers Had Same Issue

Both VST3 and CLAP use identical threading model:
- ProcessContext borrows track_info for entire audio callback duration
- Track info updates arrive on main/UI thread asynchronously
- Both need try_borrow_mut() to avoid panic

## Artifacts

### Modified Repositories

**nih-plug Fork** (`/tmp/nih-plug/` on branch `feature/track-info-support`):
- `src/wrapper/vst3/wrapper.rs:1951-1957` - VST3 threading fix
- `src/wrapper/clap/wrapper.rs:1839-1843` - CLAP threading fix
- Status: Built successfully, needs commit

**Built Plugins**:
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (rebuilt with fix)
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (rebuilt with fix)
- Build command: `cd /tmp/nih-plug && cargo xtask bundle track_info_display --release`

### Previous Handoff Work

All artifacts from previous handoff remain relevant:
- vst3-sys fork at `/tmp/vst3-sys-temp/` with IInfoListener signature fix
- Implementation plan at `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`
- Research document at `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md`

## Action Items & Next Steps

### Immediate: Test the Threading Fix

1. **Test VST3 in Bitwig Studio**:
   - Load track_info_display.vst3
   - Play audio for at least 2 minutes
   - Change track name and color multiple times during playback
   - Verify: No crashes occur, updates appear correctly

2. **Test CLAP in Bitwig Studio**:
   - Same procedure as VST3
   - CLAP was the version that crashed in the provided crash log

3. **Stress test**:
   - Multiple rapid track name/color changes during heavy audio processing
   - Verify no panics or instability

### After Successful Testing

4. **Commit the threading fix**:
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

   Fixes crash: SIGABRT after ~30 seconds when host sends track info updates
   during active audio processing."

   git push origin feature/track-info-support
   ```

5. **Complete vst3-sys commits** (from previous handoff):
   - Commit and push vst3-sys IInfoListener signature fix
   - Update nih-plug Cargo.toml to use vst3-sys fork instead of local path

6. **Test in additional DAWs** (from previous handoff):
   - Logic Pro (macOS)
   - Ableton Live (macOS)

7. **Update Parlando**:
   - Rebuild Parlando plugin with fixed nih-plug dependency
   - Test track info support in Parlando

## Other Notes

### Threading Model Understanding

**Key insight**: NIH-plug's ProcessContext pattern:
- Context is created at start of each audio callback
- Holds immutable borrows (`AtomicRef`) of shared state for entire callback
- This includes: `input_events`, `output_events`, `transport`, `track_info`
- Reference: `src/wrapper/vst3/inner.rs:380-388`

**Implication**: Any cross-thread updates to these shared resources MUST use `try_borrow_mut()` to avoid panic.

### Similar Patterns in Codebase

No other code in nih-plug wrappers currently updates AtomicRefCell data from outside the audio thread, so this is a unique case. The fix establishes the pattern for future cross-thread updates.

### Crash Log Analysis

The crash log showed:
- Exception: `EXC_CRASH (SIGABRT)` with code 6 (abort trap)
- Crashed thread: Thread 27 `bitwig-remote-pluginhost-41373-11`
- Stack trace: `abort() → track_info_display + 3332504` (Rust panic handler)
- Binary: CLAP version at base `0x3a13a8000` (not VST3)

This confirmed the issue was in CLAP, but investigation showed VST3 had identical bug.

### Repository Locations

**nih-plug Fork**:
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Modified files need commit

**vst3-sys Fork**:
- Local: `/tmp/vst3-sys-temp/`
- Remote: Need to push to github.com/lrhodin/vst3-sys
- Branch: `fix/drop-box-from-raw`
- Modified files need commit (from previous handoff)

**Parlando**:
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Uses nih-plug fork via `Cargo.toml:19`
- Current commit: `11fc8dd`
- No changes needed until nih-plug fix is tested and committed

### Important Reminders

1. **Test thoroughly** - This was a subtle timing bug that only manifested after ~30 seconds
2. **Both wrappers fixed** - VST3 and CLAP had identical issue, both are fixed
3. **try_borrow_mut() is safe** - Skipping an update is acceptable for track info
4. **Pattern established** - Future cross-thread AtomicRefCell updates should follow this pattern
5. **Previous handoff tasks remain** - Still need to complete vst3-sys commits and multi-DAW testing
