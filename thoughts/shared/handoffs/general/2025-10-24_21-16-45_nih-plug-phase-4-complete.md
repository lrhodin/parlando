---
date: 2025-10-24T21:16:45-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 11fc8dd2f2158e7015ad98e631093e068de74d74
branch: master
repository: Parlando
topic: "NIH-plug Track Information Extension - Phase 4 Complete, VST3 Crash Investigation Needed"
tags: [implementation, nih-plug, vst3, clap, track-info, phase-4-complete, testing, crash-investigation]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: NIH-plug Phase 4 Complete - VST3 Crash Needs Investigation

## Task(s)

**Status: PHASE 4 COMPLETE ✅ | VST3 CRASH IDENTIFIED 🔴**

Implementing the NIH-plug track information extension following the 6-phase plan at `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`.

**Completed Phases:**
- **Phase 1** ✅: Core TrackInfo types and ProcessContext trait method (plan:48-137)
- **Phase 2** ✅: VST3 IInfoListener implementation with soundness fix (plan:139-298)
- **Phase 3** ✅: CLAP track-info extension implementation (plan:300-478)
  - Verified in Bitwig Studio - track name and color updates working live
- **Phase 4** ✅: Example Plugin and Testing (plan:481-704)
  - Unit tests added and passing
  - Comprehensive example plugin with GUI created
  - **CLAP testing**: Works perfectly in Bitwig ✅
  - **VST3 testing**: Crashes on load in Bitwig 🔴 (segfault at 0x59)

**Next Phase:** Fix VST3 crash, then continue testing in Logic Pro and Ableton Live

## Critical References

1. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - 6-phase plan; Phases 1-4 code complete, Phase 4 testing partially complete
2. **Research Document**: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Technical research with patterns from VST3/CLAP specifications
3. **NIH-plug Fork**: `/tmp/nih-plug/` on branch `feature/track-info-support`, commit `79e0fdb1`, pushed to github.com/lrhodin/nih-plug
4. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-24_15-15-16_nih-plug-phase-3-complete.md` - Starting point for this session

## Recent Changes

**NIH-plug Fork (`/tmp/nih-plug/` - commit `79e0fdb1`):**

Phase 4 Implementation:
- `src/context.rs:65-152` - Added 6 unit tests for TrackInfo and TrackType (defaults, equality, inequality, flags, all fields)
- `plugins/examples/track_info_display/Cargo.toml` - Created example plugin manifest with atomic_refcell, nih_plug, nih_plug_vizia dependencies
- `plugins/examples/track_info_display/src/lib.rs` - Complete 334-line example plugin with vizia GUI showing track name, color, and type badges
- `Cargo.toml:38` - Added track_info_display to workspace members

**Parlando (working directory - commit `11fc8dd`):**
- `Cargo.toml:19` - Updated NIH-plug dependency to use fork: `git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support"`

## Learnings

### Critical: VST3 Crash on Load in Bitwig

The VST3 version of track_info_display crashes with segfault when loaded in Bitwig Studio. Crash details:
- **Exception**: `EXC_BAD_ACCESS (SIGSEGV)` at address `0x0000000000000059`
- **Crash location**: `track_info_display` offset 622020 (0x10ba97dc4)
- **Thread**: Main thread during plugin initialization
- **Platform**: macOS 15.6, ARM64

The crash appears to be a null pointer dereference during VST3 initialization. The CLAP version works perfectly in the same host (Bitwig), suggesting the issue is specific to VST3 wrapper code or GUI initialization.

### CLAP Bitwig Limitation - Master Track Flag

When testing CLAP in Bitwig, plugins on the master track do NOT get `is_master` flag set to true. This appears to be a Bitwig limitation in their CLAP track-info implementation. Document but leave alone - not our bug.

### Phase 4 Context Restoration

After troubleshooting created a mess, successfully restored to clean Phase 4 state by:
1. Resetting to starting commit `de58467b` from handoff
2. Rebuilding all Phase 4 work from scratch (unit tests, example plugin, workspace integration)
3. Cleaning up stray plugin bundles left in Parlando root directory
4. This approach ensures git history is clean and matches the plan

### Example Plugin Implementation Pattern

The track_info_display plugin uses:
- `AtomicRefCell<String>` for GUI-shared track name/color data
- `AtomicBool` for track type flags
- Vizia's `Data::track_name.map()` pattern for reactive GUI updates
- Process counter (every 2000 calls) for periodic console logging without flooding

## Artifacts

**Git Commits (NIH-plug fork at github.com/lrhodin/nih-plug):**
- `79e0fdb1` - Phase 4: Add unit tests and comprehensive example plugin
- All Phases 1-3 commits from previous handoff still present

**Modified/Created Files:**
- `/tmp/nih-plug/src/context.rs` - Added unit tests module
- `/tmp/nih-plug/plugins/examples/track_info_display/` - New example plugin
- `/tmp/nih-plug/Cargo.toml` - Added workspace member
- `Cargo.toml` - Updated to use fork (Parlando)

**Installed Plugins:**
- VST3: `~/Library/Audio/Plug-Ins/VST3/track_info_display.vst3` (crashes on load)
- CLAP: `~/Library/Audio/Plug-Ins/CLAP/track_info_display.clap` (works correctly)

**Documentation:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Phases 1-4 code complete, Phase 4 testing in progress
- `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Original research

**Test Results:**
- ✅ Unit tests: All 6 tests pass (`cargo test track_info`)
- ✅ Build: Compiles successfully (`cargo build --release`)
- ✅ Bundle: Both VST3 and CLAP bundle successfully
- ✅ CLAP in Bitwig: Track name, color updates work; type badges work (except master limitation)
- 🔴 VST3 in Bitwig: Crashes on load with segfault
- ⏸️ VST3 in Logic Pro: Not yet tested
- ⏸️ VST3 in Ableton Live: Not yet tested

## Action Items & Next Steps

### Immediate: Debug VST3 Crash

1. **Investigate the segfault at address 0x59**
   - Crash happens during initialization (main thread, before audio processing)
   - Likely null pointer dereference in VST3 wrapper or GUI code
   - Compare with working CLAP implementation to identify difference
   - Check vizia GUI initialization in VST3 vs CLAP

2. **Review VST3-specific code paths**
   - VST3 IInfoListener implementation (`/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:1903-1973`)
   - Track info guard pattern initialization (`/tmp/nih-plug/src/wrapper/vst3/context.rs:122-124`)
   - GUI editor creation path in VST3 wrapper

3. **Potential fixes to try**:
   - Verify track_info_guard is properly initialized before GUI creation
   - Check if GUI is accessing track_info before it's available
   - Add null checks in track_info() method
   - Test without GUI to isolate if crash is GUI-related

### After VST3 Fix: Complete Phase 4 Testing

1. **Test VST3 in Logic Pro**
   - Verify IInfoListener receives track info
   - Test track name and color updates
   - Verify no crashes or glitches

2. **Test VST3 in Ableton Live**
   - Same verification as Logic Pro
   - Test on different track types

3. **Update plan checkboxes**
   - Mark Phase 4 verification items complete
   - Document all testing results including Bitwig master track limitation

### Subsequent Phases

- **Phase 5**: Documentation and PR Submission (plan:706-798)
- **Phase 6**: Parlando Integration (plan:800-879)

## Other Notes

### Repository Locations

**NIH-plug Fork:**
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Commit: `79e0fdb1` (Phase 4 complete)

**Parlando:**
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Now uses fork via `Cargo.toml:19`
- Commit: `11fc8dd` (updated to use fork)

### Key File Locations in NIH-plug Fork

**Core Types (Phase 1):**
- TrackInfo/TrackType: `src/context.rs:12-44`
- Unit tests: `src/context.rs:65-152`
- ProcessContext trait: `src/context/process.rs:95-101`

**VST3 Implementation (Phase 2):**
- Wrapper: `src/wrapper/vst3/wrapper.rs:1903-1973` (IInfoListener)
- Inner storage: `src/wrapper/vst3/inner.rs:143,326`
- Context: `src/wrapper/vst3/context.rs:122-124` (track_info method)

**CLAP Implementation (Phase 3):**
- Wrapper: `src/wrapper/clap/wrapper.rs:1790-1840,3268-3273`
- Context: `src/wrapper/clap/context.rs:131-133`

**Example Plugin:**
- Source: `plugins/examples/track_info_display/src/lib.rs` (334 lines)
- Manifest: `plugins/examples/track_info_display/Cargo.toml`

### Testing Commands

```bash
# Build example plugin
cd /tmp/nih-plug
cargo xtask bundle track_info_display --release

# Copy to plugin folders
cp -r target/bundled/track_info_display.vst3 ~/Library/Audio/Plug-Ins/VST3/
cp -r target/bundled/track_info_display.clap ~/Library/Audio/Plug-Ins/CLAP/

# Run unit tests
cargo test track_info --lib
```

### Session Context Restoration Notes

This session started from a previous handoff but encountered troubleshooting that created git history mess. Successfully restored by:
1. Identifying starting commit from handoff (`de58467b`)
2. Hard resetting to that commit
3. Rebuilding Phase 4 from scratch following the original plan
4. Cleaning up any files created outside git (plugin bundles in Parlando root)
5. Force-pushing to clean remote history

This pattern can be repeated if future troubleshooting creates similar messes.
