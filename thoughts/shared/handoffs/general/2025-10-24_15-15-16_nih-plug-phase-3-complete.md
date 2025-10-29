---
date: 2025-10-24T15:15:16-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
branch: master
repository: Parlando
topic: "NIH-plug Track Information Extension - Phases 1-3 Complete, Ready for Phase 4"
tags: [implementation, nih-plug, vst3, clap, track-info, phase-3-complete, verified]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: NIH-plug Track Information Extension - Phase 3 Complete & Verified

## Task(s)

**Status: PHASES 1-3 COMPLETE ✅ | READY FOR PHASE 4**

Implementing the NIH-plug track information extension following the 6-phase plan at `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`.

**Completed Phases:**

- **Phase 1** ✅: Core TrackInfo types and ProcessContext trait method (plan:48-137)
- **Phase 2** ✅: VST3 IInfoListener implementation with soundness fix (plan:139-298)
- **Phase 3** ✅: CLAP track-info extension implementation (plan:300-478)
  - All automated verification passed
  - **Verified in Bitwig Studio** - track name and color updates working live

**Next Phase:** Phase 4 - Example Plugin and Testing (plan:481-704)

## Critical References

1. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - 6-phase plan; Phases 1-3 complete, Phase 4 starts at line 481
2. **Research Document**: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Technical research with patterns from VST3/CLAP specifications
3. **NIH-plug Fork**: `/tmp/nih-plug/` on branch `feature/track-info-support`, commit `de58467b`, pushed to github.com/lrhodin/nih-plug

## Recent Changes

**NIH-plug Fork (`/tmp/nih-plug/` - commits `73dd526d`, `7a3eeb10`, `64c2b9ed`, `4112b24c`, `9b9a4aca`, `de58467b`):**

Phase 1 (commit 73dd526d):
- `src/context.rs:12-44` - Added TrackInfo struct and TrackType flags
- `src/context/process.rs:95-101` - Added track_info() method to ProcessContext trait with default None implementation

Phase 2 (commits 7a3eeb10, 64c2b9ed):
- `src/wrapper/vst3/wrapper.rs:42-50` - Added IInfoListener to implements macro
- `src/wrapper/vst3/wrapper.rs:1-30` - Added imports for IInfoListener, IAttributeList, TrackInfo
- `src/wrapper/vst3/wrapper.rs:1903-1973` - Implemented IInfoListener::set_channel_context_infos()
- `src/wrapper/vst3/inner.rs:18` - Added TrackInfo import
- `src/wrapper/vst3/inner.rs:143` - Added track_info field to WrapperInner (AtomicRefCell)
- `src/wrapper/vst3/inner.rs:326` - Initialize track_info in WrapperInner::new()
- `src/wrapper/vst3/context.rs:1,8` - Added AtomicRef and TrackInfo imports
- `src/wrapper/vst3/context.rs:46` - Added track_info_guard field to WrapperProcessContext
- `src/wrapper/vst3/context.rs:122-124` - Implemented track_info() using guard pattern (soundness fix)
- `src/wrapper/vst3/inner.rs:387` - Initialize track_info_guard in make_process_context()

Phase 3 (commit 4112b24c):
- `src/wrapper/clap/wrapper.rs:50-55,64,91` - Added CLAP track-info extension imports
- `src/wrapper/clap/wrapper.rs:252-255` - Added clap_plugin_track_info, host_track_info, track_info fields
- `src/wrapper/clap/wrapper.rs:696-700` - Initialize extension fields in Wrapper::new()
- `src/wrapper/clap/wrapper.rs:1876-1882` - Query host extension and initial track info in init()
- `src/wrapper/clap/wrapper.rs:1790-1840` - Implemented update_track_info() helper method
- `src/wrapper/clap/wrapper.rs:3268-3273` - Implemented ext_track_info_changed() callback
- `src/wrapper/clap/wrapper.rs:2363-2364` - Registered extension in get_extension()
- `src/wrapper/clap/context.rs:1,10` - Added AtomicRef and TrackInfo imports
- `src/wrapper/clap/context.rs:45` - Added track_info_guard field to WrapperProcessContext
- `src/wrapper/clap/context.rs:131-133` - Implemented track_info() method
- `src/wrapper/clap/wrapper.rs:773` - Initialize track_info_guard in make_process_context()

Test Plugin (commit 9b9a4aca):
- `plugins/examples/track_info_test/Cargo.toml` - Created test plugin manifest
- `plugins/examples/track_info_test/src/lib.rs` - Minimal CLAP plugin that logs track info every 2000 process calls
- `Cargo.toml:37` - Added track_info_test to workspace members

Style Fixes (commit de58467b):
- `src/wrapper/vst3/wrapper.rs:1913,1928,1941,1948` - Replaced CStr::from_bytes_with_nul_unchecked with c"" literals
- `src/wrapper/vst3/wrapper.rs:2` - Removed unused CStr import

**Parlando (working directory):**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:469-477` - Marked Phase 3 verification complete

## Learnings

### Critical: Guard Pattern for Soundness

**The Pattern**: To safely return references to AtomicRefCell data in ProcessContext:
1. Add a guard field to the context struct: `track_info_guard: AtomicRef<'a, Option<TrackInfo>>`
2. Initialize guard in make_process_context(): `track_info_guard: self.track_info.borrow()`
3. Return reference from guard: `self.track_info_guard.as_ref()` - NO unsafe code needed

This pattern is used in both VST3 and CLAP implementations and matches existing NIH-plug patterns (input_events_guard, output_events_guard). **Never use unsafe transmute to extend lifetimes.**

### VST3 IInfoListener Implementation Details

1. **COM Interface Signature**: Methods use `*mut c_void` not `SharedVstPtr`
   - Must cast: `&*(list as *const *const dyn IAttributeList)` then `&**list`
   - Always null-check before dereferencing

2. **VST3 String Type**: Uses `i16` (tchar) not `u16`
   - Arrays: `[0i16; 128]`
   - Convert: `.map(|&c| c as u16)` before `String::from_utf16_lossy()`
   - Size parameter is `u32`

3. **Thread Safety**: `set_channel_context_infos()` called on main thread only (VST3 spec)
   - Safe to use `borrow_mut()` without audio thread conflicts
   - Audio thread only reads via `borrow()` in track_info()

### CLAP track-info Extension Details

1. **Extension Constants**: From clap-sys 0.5.0
   - Extension ID: `CLAP_EXT_TRACK_INFO` (`"clap.track-info/1"`)
   - Flags: `CLAP_TRACK_INFO_HAS_TRACK_NAME`, `CLAP_TRACK_INFO_HAS_TRACK_COLOR`, etc.
   - Import: `use clap_sys::string_sizes::CLAP_NAME_SIZE;` required

2. **Data Structure**: `clap_track_info` struct
   - `name: [c_char; CLAP_NAME_SIZE]` - null-terminated C string
   - `color: clap_color` with separate r,g,b,a u8 fields
   - `flags: u64` - bitfield for has_name, has_color, is_master, is_bus, is_return

3. **Lifecycle**: Query host in `init()`, update via `changed()` callback
   - Host extension: `query_host_extension::<clap_host_track_info>(&wrapper.host_callback, CLAP_EXT_TRACK_INFO)`
   - Call `update_track_info()` immediately after querying to get initial state

### Modern Rust Idioms

Use `c"string"` literals instead of `CStr::from_bytes_with_nul_unchecked(b"string\0")` to avoid clippy warnings. This is available in Rust nightly (NIH-plug uses nightly).

## Artifacts

**Git Commits (NIH-plug fork at github.com/lrhodin/nih-plug):**
- `73dd526d` - Phase 1: Add core TrackInfo types and ProcessContext trait method
- `7a3eeb10` - Phase 2: Implement VST3 IInfoListener for track information
- `64c2b9ed` - Fix soundness issue in track_info() lifetime management
- `4112b24c` - Phase 3: Implement CLAP track-info extension for track information
- `9b9a4aca` - Add track_info_test example plugin for CLAP verification
- `de58467b` - Fix clippy style warnings: use c"" literals

**Modified Files:**
- `/tmp/nih-plug/src/context.rs` - Core TrackInfo types
- `/tmp/nih-plug/src/context/process.rs` - ProcessContext trait method
- `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs` - VST3 IInfoListener implementation
- `/tmp/nih-plug/src/wrapper/vst3/inner.rs` - VST3 storage field
- `/tmp/nih-plug/src/wrapper/vst3/context.rs` - VST3 ProcessContext implementation
- `/tmp/nih-plug/src/wrapper/clap/wrapper.rs` - CLAP track-info extension implementation
- `/tmp/nih-plug/src/wrapper/clap/context.rs` - CLAP ProcessContext implementation
- `/tmp/nih-plug/plugins/examples/track_info_test/` - Test plugin for verification

**Documentation:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phases 1-3 marked complete (lines 127-131, 290-292, 469-477)
- `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Original research

**Verification Results:**
- ✅ `cargo build` - Compiles successfully
- ✅ `cargo clippy` - No errors, only unrelated warnings
- ✅ `cargo test` - All 64 tests passed
- ✅ Bitwig Studio test - Track name and color updates working live

## Action Items & Next Steps

### Immediate: Begin Phase 4 - Example Plugin and Testing

Following `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:481-704`:

**Note**: We already have a minimal test plugin (`track_info_test`) that was used to verify Phase 3. Phase 4 calls for a more complete example with GUI and comprehensive testing.

1. **Decide on Example Plugin Scope**
   - Option A: Enhance existing `track_info_test` with GUI and more features
   - Option B: Create new full-featured example per plan (plugins/examples/track_info_display)
   - Recommended: Option A (iterate on working test plugin)

2. **Add GUI to Display Track Info** (if pursuing full Phase 4)
   - Show track name in plugin UI
   - Show track color as visual indicator
   - Display track type badges (master/bus/return)
   - Update UI in real-time when track info changes

3. **Add Unit Tests** (`src/context.rs`)
   - TrackInfo default values test
   - TrackInfo equality test
   - TrackType flags test

4. **Comprehensive Testing**
   - Test in multiple DAWs (Logic Pro for VST3, Bitwig/Reaper for CLAP)
   - Verify both VST3 and CLAP implementations
   - Test on master/bus/return tracks
   - Verify track name/color changes update live

5. **Update Plan Checkboxes**
   - Mark automated verification items complete
   - Document manual testing results

### Subsequent Phases

- **Phase 5**: Documentation and PR Submission (plan:706-798)
  - Add documentation to lib.rs
  - Update CHANGELOG.md
  - Submit PR to upstream NIH-plug
  - Reference VST3 and CLAP specs in PR description

- **Phase 6**: Parlando Integration (plan:800-879)
  - Update Parlando's Cargo.toml to use fork
  - Implement Phase 1.5 auto musician naming from track names
  - Test in real performance scenario with multiple instances
  - Consider fallback when track info unavailable

## Other Notes

### Repository Locations

**NIH-plug Fork:**
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Commit: `de58467b` (all phases 1-3 complete with style fixes)

**Parlando:**
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Will use fork via Cargo.toml:19 (currently points to upstream)
- Test harness: `cargo run --package test-harness --release`

### Key File Locations in NIH-plug Fork

**Core Types (Phase 1):**
- TrackInfo/TrackType: `src/context.rs:12-44`
- ProcessContext trait: `src/context/process.rs:95-101`

**VST3 Implementation (Phase 2):**
- Wrapper: `src/wrapper/vst3/wrapper.rs`
- Inner storage: `src/wrapper/vst3/inner.rs`
- Context: `src/wrapper/vst3/context.rs`

**CLAP Implementation (Phase 3):**
- Wrapper: `src/wrapper/clap/wrapper.rs`
- Context: `src/wrapper/clap/context.rs`

**Test Plugin:**
- Minimal example: `plugins/examples/track_info_test/`
- Built artifact: `/tmp/nih-plug/target/bundled/track_info_test.clap`

### Testing the Current Implementation

To verify the implementation works:

```bash
# Build test plugin
cd /tmp/nih-plug
cargo xtask bundle track_info_test --release

# Copy to CLAP plugins folder
cp -r target/bundled/track_info_test.clap ~/Library/Audio/Plug-Ins/CLAP/

# Load in Bitwig/Reaper and check console output
# Plugin logs track info every 2000 process calls (~few seconds)
```

### Bitwig Verification Results

From actual testing session:
```
=== CLAP TRACK INFO RECEIVED ===
  Track Name: Audio 2
  Track Color: RGBA(217, 157, 15, 255)
  Track Type: master=false, bus=false, return=false
================================

[After renaming track]

=== CLAP TRACK INFO RECEIVED ===
  Track Name: test
  Track Color: RGBA(87, 97, 198, 255)
  Track Type: master=false, bus=false, return=false
================================
```

This confirms:
- Extension properly registered
- Track name received and updates
- Track color received and updates
- Track type flags working
- No crashes or memory issues

### Phase 4 Considerations

The plan calls for a more comprehensive example plugin with:
- GUI displaying track info
- String parameters showing track name/color
- Logging to demonstrate usage
- Support for both VST3 and CLAP

However, the minimal `track_info_test` plugin already validates the core functionality. Phase 4 could either:
1. Enhance the test plugin with GUI
2. Create a new plugin per plan specification
3. Skip to Phase 5 (documentation) since implementation is verified

Recommend discussing with user which approach to take.

### NIH-plug Build Requirements

- Rust nightly toolchain required (for SIMD features)
- CLAP is always compiled (not behind feature flag)
- VST3 behind `--features vst3` (enabled by default)
- Test with: `cargo build`, `cargo clippy`, `cargo test`

### Important Patterns to Follow

1. **Guard Pattern**: Always hold AtomicRef guards as context fields, never use unsafe transmute
2. **Thread Safety**: Main thread for updates, audio thread for read-only access via borrow()
3. **Null Checks**: Always check for None before accessing track_info
4. **Backwards Compatibility**: Default trait implementation returns None, opt-in feature
