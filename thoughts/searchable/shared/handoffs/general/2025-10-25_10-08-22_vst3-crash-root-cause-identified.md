---
date: 2025-10-25T10:08:22-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "VST3 Crash Root Cause Identified - IInfoListener Pointer Type Bug"
tags: [debugging, nih-plug, vst3, crash-analysis, iinfolistener, phase-4]
status: in_progress
last_updated: 2025-10-25
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: VST3 Crash Root Cause Identified - Fix Ready to Implement

## Task(s)

**Status: ROOT CAUSE IDENTIFIED ✅ | FIX NOT YET IMPLEMENTED ⏸️**

Debugging VST3 crash that occurs when loading the `track_info_display` plugin in Bitwig Studio. This is part of NIH-plug Phase 4 testing following the 6-phase implementation plan at `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`.

**Previous Context:**
- Phases 1-3 complete: Core types, VST3 IInfoListener, CLAP track-info extension
- Phase 4 testing: CLAP works perfectly ✅, VST3 crashes on load 🔴
- Previous session incorrectly suspected GUI initialization - **this was a red herring**

**Current Status:**
- ✅ Root cause identified: Incorrect pointer type in IInfoListener implementation
- ⏸️ Fix designed but not implemented
- ⏸️ Testing in Bitwig pending
- ⏸️ Testing in Logic Pro and Ableton Live pending

## Critical References

1. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-24_21-16-45_nih-plug-phase-4-complete.md` - Starting point with crash log
2. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phase 4 (lines 481-704)
3. **NIH-plug Fork**: `/tmp/nih-plug/` at commit `79e0fdb1` on branch `feature/track-info-support`

## Recent Changes

**No code changes made in this session** - analysis only.

## Learnings

### Critical Bug: IInfoListener Pointer Type Mismatch

**Location**: `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:1900-1907`

**The Problem**:
The `IInfoListener::set_channel_context_infos` implementation uses an incorrect pointer cast that creates an invalid fat pointer:

```rust
unsafe fn set_channel_context_infos(&self, list: *mut c_void) -> tresult {
    // ❌ WRONG: Double pointer cast creates invalid vtable reference
    let list = &*(list as *const *const dyn IAttributeList);
    let list = &**list;  // Dereferences at address 0x59 → SEGFAULT
```

**Why This Crashes**:
- The cast `*mut c_void as *const *const dyn IAttributeList` creates a fat pointer (data + vtable)
- The vtable pointer ends up pointing to address 0x59 (89 bytes offset)
- When dereferenced, CPU tries to read vtable info from invalid memory
- Result: `EXC_BAD_ACCESS (SIGSEGV) at 0x0000000000000059`

**The Correct Pattern** (used in other VST3 COM methods at wrapper.rs:414, 467, 501, 506, etc.):
```rust
unsafe fn set_channel_context_infos(
    &self,
    list: SharedVstPtr<dyn IAttributeList>  // ✅ Correct type
) -> tresult {
    check_null_ptr!(list);
    let list = list.upgrade().unwrap();  // ✅ Safe conversion
    // ... rest of implementation works correctly ...
}
```

**Why CLAP Works But VST3 Doesn't**:
- CLAP uses a different extension mechanism with direct function pointers
- VST3 uses COM interfaces with vtable indirection
- The pointer type matters critically for COM, but CLAP doesn't have this issue

**Why This Wasn't Caught Earlier**:
1. Code compiles successfully - signature mismatch doesn't cause compile error
2. Crash happens during initialization, not audio processing
3. The soundness fix in commit `64c2b9ed` addressed lifetime issues but not this pointer bug
4. No VST3 testing was done until Phase 4

### Crash Location Analysis

From crash log Thread 0:
```
0   track_info_display         0x10ba97dc4 0x10ba00000 + 622020
1   track_info_display         0x10baa937c 0x10ba00000 + 693116
```

These offsets correspond to the IInfoListener implementation being called during VST3 plugin initialization, before audio processing starts.

### GUI Is NOT the Problem

Previous session suspected GUI initialization. **This was incorrect**. The crash is in COM interface initialization, completely unrelated to the Vizia GUI code. The GUI should remain unchanged.

## Artifacts

**NIH-plug Fork (`/tmp/nih-plug/` - commit `79e0fdb1`):**
- `src/wrapper/vst3/wrapper.rs:1900-1965` - Buggy IInfoListener implementation (needs fix)
- `src/wrapper/vst3/wrapper.rs:414,467,501,506` - Examples of correct SharedVstPtr usage
- `plugins/examples/track_info_display/src/lib.rs` - Example plugin (works in CLAP, crashes VST3)

**Installed Plugins:**
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (crashes on load)
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (works perfectly)

**Documentation:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Original plan
- `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Research document

## Action Items & Next Steps

### Immediate: Implement the Fix

1. **Update IInfoListener signature** (`/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:1900`)
   - Change parameter from `list: *mut c_void` to `list: SharedVstPtr<dyn IAttributeList>`
   - Replace unsafe pointer casts with `.upgrade().unwrap()` pattern
   - Follow the pattern from other VST3 COM methods in the same file

2. **Test the fix**
   ```bash
   cd /tmp/nih-plug
   cargo xtask bundle track_info_display --release
   cp -r target/bundled/track_info_display.vst3 ~/Library/Audio/Plug-Ins/VST3/
   # Load in Bitwig Studio and verify:
   # - Plugin loads without crash
   # - Track name displays correctly
   # - Track color updates when changed
   # - Type badges work for different track types
   ```

3. **Commit and push fix**
   ```bash
   git add src/wrapper/vst3/wrapper.rs
   git commit -m "Fix VST3 IInfoListener pointer type causing segfault on load"
   git push origin feature/track-info-support
   ```

### After VST3 Fix Verification

1. **Test VST3 in Logic Pro**
   - Verify IInfoListener receives track info
   - Test track name and color updates
   - Verify no crashes or glitches

2. **Test VST3 in Ableton Live**
   - Same verification as Logic Pro
   - Test on different track types

3. **Update Parlando**
   - Dependency already points to fork in `Cargo.toml:19`
   - Rebuild Parlando after NIH-plug fix is merged

4. **Complete Phase 4**
   - Mark Phase 4 testing complete in plan document
   - Document Bitwig master track limitation (CLAP doesn't set `is_master` flag)

### Subsequent Phases

- **Phase 5**: Documentation and PR submission to upstream NIH-plug
- **Phase 6**: Parlando integration with auto musician naming

## Other Notes

### Repository Locations

**NIH-plug Fork:**
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Commit: `79e0fdb1` (Phase 4 complete - but with VST3 bug)

**Parlando:**
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Uses fork via `Cargo.toml:19`
- Commit: `11fc8dd` (updated to use fork)

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

### Known Good Patterns in Codebase

Reference these for the correct implementation:
- `wrapper.rs:414-470` - `set_state()` and `get_state()` with `SharedVstPtr<dyn IBStream>`
- `wrapper.rs:501-516` - IEditController methods with SharedVstPtr
- `wrapper.rs:688-691` - `set_component_handler()` with upgrade pattern

### Crash Report Summary

- **Exception**: `EXC_BAD_ACCESS (SIGSEGV)` at address `0x0000000000000059`
- **Thread**: Main thread during plugin initialization
- **Platform**: macOS 15.6 ARM64, Bitwig Studio
- **Binary**: `track_info_display.vst3`
- **Root Cause**: Invalid fat pointer dereference in IInfoListener COM interface

### Important Reminders

1. **Do NOT remove the GUI** - it's not related to the crash
2. **The CLAP version is a good reference** - it proves the core logic works
3. **SharedVstPtr is the standard** - all VST3 COM interfaces use this pattern
4. **The fix is simple** - just change the function signature and upgrade pattern
