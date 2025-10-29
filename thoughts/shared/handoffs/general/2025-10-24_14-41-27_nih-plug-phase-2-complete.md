---
date: 2025-10-24T14:41:27-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
branch: master
repository: Parlando
topic: "NIH-plug Track Information Extension - Phase 2 Complete"
tags: [implementation, nih-plug, vst3, track-info, phase-2, soundness-fix]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: NIH-plug Track Information Extension - Phase 2 Complete

## Task(s)

**Status: PHASE 2 COMPLETE ✅ (with critical soundness fix applied)**

Implementing the NIH-plug track information extension following the 6-phase plan at `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md`.

**Phase 2 Completed:**
1. ✅ Added IInfoListener to VST3 wrapper's implements macro
2. ✅ Added track_info storage field to WrapperInner (AtomicRefCell)
3. ✅ Implemented set_channel_context_infos() to extract track name, color, index, UID
4. ✅ Updated VST3 ProcessContext to implement track_info() method
5. ✅ All automated verification passed (build, clippy, tests)
6. ✅ **CRITICAL: Fixed soundness issue** with lifetime management
7. ✅ Committed and pushed to GitHub

**Next Phase:** Phase 3 - CLAP track-info Extension Implementation

## Critical References

1. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - 6-phase plan; Phase 2 (lines 139-298) now complete, Phase 3 starts at line 300
2. **Research Document**: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Technical research with patterns
3. **NIH-plug Fork**: `/tmp/nih-plug/` - Local clone on branch `feature/track-info-support`, commit `64c2b9ed`

## Recent Changes

**NIH-plug Fork (`/tmp/nih-plug/` - commits `7a3eeb10` and `64c2b9ed`):**

Initial implementation (commit 7a3eeb10):
- `src/wrapper/vst3/wrapper.rs:42-50` - Added IInfoListener to implements macro
- `src/wrapper/vst3/wrapper.rs:1-30` - Added imports (CStr, IInfoListener, IAttributeList, TrackInfo)
- `src/wrapper/vst3/wrapper.rs:1899-1971` - Implemented IInfoListener::set_channel_context_infos()
- `src/wrapper/vst3/inner.rs:18` - Added TrackInfo import
- `src/wrapper/vst3/inner.rs:143` - Added track_info field to WrapperInner
- `src/wrapper/vst3/inner.rs:326` - Initialized track_info in WrapperInner::new()

Soundness fix (commit 64c2b9ed):
- `src/wrapper/vst3/context.rs:1` - Added AtomicRef import
- `src/wrapper/vst3/context.rs:8` - Added TrackInfo import
- `src/wrapper/vst3/context.rs:46` - Added track_info_guard field to WrapperProcessContext
- `src/wrapper/vst3/context.rs:122-124` - Fixed track_info() to use guard (removed unsafe transmute)
- `src/wrapper/vst3/inner.rs:387` - Initialize track_info_guard in make_process_context()

**Parlando (working directory):**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:290-292` - Marked Phase 2 automated verification as complete

## Learnings

### Critical Soundness Issue Found and Fixed

**The Problem**: Initial implementation had a use-after-free bug in `track_info()` method:
- Used `unsafe { transmute(self.inner.track_info.borrow().as_ref()) }`
- The `Ref` guard from `.borrow()` was dropped at function exit
- Returned reference outlived the guard's lifetime → **undefined behavior**

**The Fix**: Follow NIH-plug's established pattern for shared state access:
- Added `track_info_guard: AtomicRef<'a, Option<TrackInfo>>` to `WrapperProcessContext` (src/wrapper/vst3/context.rs:46)
- Guard is held for the entire `'a` lifetime of the context (initialized in make_process_context())
- Safe implementation: `self.track_info_guard.as_ref()` - no unsafe code needed
- Matches pattern used for `input_events_guard` and `output_events_guard`

### VST3 IInfoListener Patterns

1. **COM Interface Signature**: VST3 methods use `*mut c_void` not `SharedVstPtr`:
   - Must cast: `&*(list as *const *const dyn IAttributeList)` then `&**list`
   - Always null-check before dereferencing

2. **VST3 String Type**: Uses `i16` (tchar) not `u16`:
   - Arrays must be `[0i16; 128]` not `[0u16; 128]`
   - Convert with `.map(|&c| c as u16)` before `String::from_utf16_lossy()`
   - Size parameter is `u32` not `i32`

3. **Thread Safety**: `set_channel_context_infos()` is called on **main thread only** (per VST3 spec):
   - Safe to use `borrow_mut()` without audio thread conflicts
   - Audio thread only reads via `borrow()` in track_info()

## Artifacts

**Git Commits:**
- NIH-plug fork commit `7a3eeb10` - "Phase 2: Implement VST3 IInfoListener for track information"
- NIH-plug fork commit `64c2b9ed` - "Fix soundness issue in track_info() lifetime management"
- Both pushed to: github.com/lrhodin/nih-plug branch `feature/track-info-support`

**Modified Files:**
- `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs` - IInfoListener implementation
- `/tmp/nih-plug/src/wrapper/vst3/inner.rs` - Storage field and initialization
- `/tmp/nih-plug/src/wrapper/vst3/context.rs` - ProcessContext implementation with guard

**Documentation:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:290-292` - Phase 2 automated verification checkboxes marked complete

**Verification Results:**
- ✓ `cargo build --features vst3` - Compiles successfully
- ✓ `cargo clippy --features vst3` - No errors (minor style warnings only)
- ✓ `cargo test --features vst3` - All 64 tests passed

## Action Items & Next Steps

### Immediate: Begin Phase 3 - CLAP track-info Extension Implementation

Following `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:300-478`:

1. **Add CLAP Extension Dependencies** (`/tmp/nih-plug/src/wrapper/clap/wrapper.rs`)
   - Import track-info types from clap_sys
   - Import TrackInfo from crate::context

2. **Add Extension Fields to Wrapper** (wrapper.rs:~200)
   - Add `clap_plugin_track_info: clap_plugin_track_info` vtable
   - Add `host_track_info: AtomicRefCell<Option<ClapPtr<clap_host_track_info>>>`
   - Add `track_info: AtomicRefCell<Option<TrackInfo>>`

3. **Initialize Extension** (wrapper.rs:~500)
   - Initialize vtable with changed callback
   - Initialize storage fields

4. **Query Host Extension** (wrapper.rs:~1850)
   - Query for CLAP_EXT_TRACK_INFO in init()
   - Call update_track_info() to get initial data

5. **Implement Extension Functions** (wrapper.rs:~3000)
   - Add update_track_info() helper
   - Add ext_track_info_changed() callback
   - Parse clap_track_info flags for name, color, track type

6. **Register Extension** (wrapper.rs:~2500)
   - Add CLAP_EXT_TRACK_INFO to get_extension()

7. **Update CLAP ProcessContext** (`/tmp/nih-plug/src/wrapper/clap/context.rs:~300`)
   - Add track_info_guard field to WrapperProcessContext
   - Implement track_info() method (same pattern as VST3)

8. **Verify Phase 3 Success Criteria**
   - `cargo build --features clap`
   - `cargo clippy --features clap`
   - `cargo test --features clap`

### Subsequent Phases

- **Phase 4**: Create example plugin demonstrating track info (plan:480-704)
- **Phase 5**: Add documentation and submit PR to upstream (plan:706-798)
- **Phase 6**: Integrate into Parlando for auto musician naming (plan:800-879)

## Other Notes

### Repository Locations

**NIH-plug Fork:**
- Local: `/tmp/nih-plug/`
- Remote: github.com/lrhodin/nih-plug
- Branch: `feature/track-info-support`
- Commit: `64c2b9ed` (with soundness fix)

**Parlando:**
- Working directory: `/Users/ludvig/Desktop/Parlando`
- Using fork via Cargo.toml:19
- Test harness: `cargo run --package test-harness --release`

### Key File Locations in NIH-plug Fork

**For Phase 3 (CLAP):**
- Wrapper: `src/wrapper/clap/wrapper.rs`
- Context: `src/wrapper/clap/context.rs`
- Pattern reference: Look at how VST3 was implemented in Phase 2

**Core Types (already done in Phase 1):**
- TrackInfo/TrackType: `src/context.rs:12-44`
- ProcessContext trait: `src/context/process.rs:95-101`

### Testing Strategy for Phase 3

**Automated checks:**
- Build with CLAP features
- Clippy for CLAP code
- Run CLAP-specific tests

**Manual checks (defer to later phases):**
- Load in Bitwig (CLAP host)
- Verify track name updates
- Test track color changes
- Verify master/bus/return flags

### Important Pattern to Follow

The guard pattern is **critical** for soundness:
```rust
// In ProcessContext struct - hold the guard as a field:
pub(super) track_info_guard: AtomicRef<'a, Option<TrackInfo>>,

// In make_process_context() - initialize the guard:
track_info_guard: self.track_info.borrow(),

// In track_info() method - use the guard:
self.track_info_guard.as_ref()  // Safe, no unsafe code needed!
```

Do NOT use `transmute` to extend lifetimes - always hold guards in the context struct.

### Rustup Toolchain Note

NIH-plug requires Rust nightly. If clippy isn't installed:
```bash
cd /tmp/nih-plug
rustup component add clippy
```

### Timeline Estimate

From plan (remaining phases):
- Phase 3 (CLAP): ~2-3 days
- Phase 4 (Example): ~1 day
- Phase 5 (Docs/PR): ~1 day
- Phase 6 (Parlando): ~1 day
- **Total remaining**: ~5-6 days to full Parlando integration
