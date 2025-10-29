---
date: 2025-10-24T14:05:58-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
branch: master
repository: Parlando
topic: "NIH-plug Track Information Extension - Implementation Plan Created"
tags: [implementation, planning, nih-plug, vst3, clap, track-info, parlando-phase-1.5]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: NIH-plug Implementation Plan Created

## Task(s)

**Status: PLANNING COMPLETE ✅**

Resumed from handoff `thoughts/shared/handoffs/general/2025-10-24_13-43-23_nih-plug-track-info-research.md` which contained complete research on extending NIH-plug. User chose Option A (implement NIH-plug extension vs fallback strategy) and requested a detailed implementation plan.

**Completed:**
1. ✅ Reviewed previous research document thoroughly
2. ✅ Verified current Parlando codebase state (Phase 1 complete, no uncommitted changes)
3. ✅ Created comprehensive 6-phase implementation plan
4. ✅ Defined success criteria for each phase (automated + manual verification)
5. ✅ Synced plan to thoughts repository

**Next Steps:** Begin implementation starting with Phase 1 (fork setup).

## Critical References

1. **Implementation Plan**: `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - Complete 6-phase plan ready for execution
2. **Research Document**: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Technical research with code examples
3. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-24_13-43-23_nih-plug-track-info-research.md` - Context on research completion

## Recent Changes

**Documentation Created:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md:1-end` - Complete implementation plan with 6 phases
- `thoughts/shared/handoffs/general/2025-10-24_14-05-58_nih-plug-implementation-plan-created.md:1-end` - This handoff

**No Code Changes:** This session focused entirely on planning. The Parlando codebase remains unchanged at Phase 1 complete state (git clean).

## Learnings

### Implementation Strategy

1. **Phased Approach Chosen:**
   - Phase 1: Fork setup and core types
   - Phase 2: VST3 IInfoListener implementation
   - Phase 3: CLAP track-info extension
   - Phase 4: Example plugin and testing
   - Phase 5: Documentation and PR submission
   - Phase 6: Parlando integration

2. **Key Technical Decisions:**
   - Use `AtomicRefCell` for thread-safe storage (avoids blocking audio thread)
   - Unified `TrackInfo` struct works for both VST3 and CLAP
   - Default trait implementation returns `None` for backwards compatibility
   - Fork can be used in Parlando immediately (no need to wait for upstream PR)

3. **Success Criteria Pattern:**
   - Each phase has **Automated Verification** (compile, tests, CI)
   - Each phase has **Manual Verification** (DAW testing, feature validation)
   - Manual testing required before proceeding to next phase

4. **Scope Boundaries:**
   - NOT adding to InitContext (keep minimal)
   - NOT creating separate TrackContext trait (avoid complexity)
   - NOT implementing Standalone support (return None)
   - NOT implementing write capabilities (read-only)

### Parlando Context

**Current State:**
- Using upstream NIH-plug: `git = "https://github.com/robbert-vdh/nih-plug.git"` (Cargo.toml:19)
- Phase 1 complete (OSC→MIDI conversion working)
- Phase 1.5 blocked on track name detection
- User strongly desires auto track name sync

**Integration Plan:**
- Update Cargo.toml to point to fork: `git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support"`
- Add track info handling in `src/lib.rs` process() method
- Fallback to "Musician Number" parameter when track info unavailable
- Send musician name updates via OSC to app

## Artifacts

**Implementation Plan:**
- `thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md` - 6-phase plan with:
  - Detailed code changes for each phase
  - File paths and line number references
  - Success criteria (automated + manual)
  - Testing strategy
  - Performance considerations
  - Migration notes

**Supporting Documents:**
- `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Technical research document
- `Parlando Plugin Phase 1.5 Update Guide.md:516-576` - Phase 1.5 specification with track name requirements

**NIH-plug Reference:**
- `/tmp/nih-plug/` - Local clone at commit `28b149e` (still available for reference)

## Action Items & Next Steps

### Immediate: Begin Phase 1 Implementation

1. **Fork NIH-plug on GitHub**
   - Go to https://github.com/robbert-vdh/nih-plug
   - Click "Fork" button
   - Fork to `lrhodin/nih-plug`

2. **Clone and Setup Fork**
   ```bash
   git clone https://github.com/lrhodin/nih-plug.git
   cd nih-plug
   git remote add upstream https://github.com/robbert-vdh/nih-plug.git
   git checkout -b feature/track-info-support
   ```

3. **Implement Core TrackInfo Types**
   - Add `TrackInfo` struct to `src/context.rs`
   - Add `TrackType` flags struct
   - Add `track_info()` method to `ProcessContext` trait in `src/context/process.rs:100`
   - See plan Phase 1 for exact code

4. **Verify Phase 1 Success Criteria**
   - Run automated checks (compile, tests, format)
   - Verify default implementation doesn't break existing plugins
   - Complete Phase 1 before proceeding to Phase 2

### Subsequent Phases

5. **Phase 2**: Implement VST3 IInfoListener (see plan:71-183)
6. **Phase 3**: Implement CLAP track-info extension (see plan:185-295)
7. **Phase 4**: Create example plugin and tests (see plan:297-403)
8. **Phase 5**: Documentation and PR submission (see plan:405-475)
9. **Phase 6**: Integrate fork into Parlando (see plan:477-556)

## Other Notes

### Estimated Timeline

- **Phase 1-4 (Implementation)**: 1-2 weeks
- **Phase 5 (PR submission)**: 1-2 days
- **Upstream PR review**: 1-2 weeks (can proceed with Phase 6 during this)
- **Phase 6 (Parlando integration)**: 1-2 days
- **Total for Parlando to use feature**: ~1-2 weeks (don't need to wait for upstream)

### Important File Locations

**NIH-plug (to be modified in fork):**
- Core types: `src/context.rs`
- ProcessContext trait: `src/context/process.rs`
- VST3 wrapper: `src/wrapper/vst3/wrapper.rs`
- VST3 inner: `src/wrapper/vst3/inner.rs`
- VST3 context: `src/wrapper/vst3/context.rs`
- CLAP wrapper: `src/wrapper/clap/wrapper.rs`
- CLAP context: `src/wrapper/clap/context.rs`

**Parlando (to be updated in Phase 6):**
- Main plugin: `src/lib.rs`
- Dependencies: `Cargo.toml`

### Testing Environments Needed

- **VST3 Testing**: Logic Pro, Ableton Live
- **CLAP Testing**: Reaper, Bitwig Studio
- **Platforms**: macOS (primary), Linux (CI), Windows (CI)

### Relevant API Documentation

- VST3 IInfoListener: https://steinbergmedia.github.io/vst3_doc/vstinterfaces/classSteinberg_1_1Vst_1_1ChannelContext_1_1IInfoListener.html
- CLAP track-info: https://github.com/free-audio/clap/blob/main/include/clap/ext/track-info.h

### Project Context

This work directly enables Parlando Phase 1.5 which adds:
- UUID generation for plugin instances
- Discovery protocol (port 30000)
- Primary election mechanism
- **Auto musician naming from track names** (requires this NIH-plug extension)
- Fallback to "Musician Number" parameter

User strongly prefers the auto naming feature, which is why we're implementing the NIH-plug extension rather than using the simpler fallback-only approach.