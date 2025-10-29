---
date: 2025-10-24T13:43:23-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
branch: master
repository: Parlando
topic: "NIH-plug Track Information Extension - Research Complete"
tags: [research, nih-plug, vst3, clap, track-info, implementation-strategy, open-source-contribution]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: NIH-plug Track Information Extension - Research Complete

## Task(s)

**Status: RESEARCH COMPLETE ✅**

Resumed from handoff `thoughts/shared/handoffs/general/2025-10-24_12-42-32_track-name-research-completion.md` to research NIH-plug implementation patterns for adding track information support.

**Completed:**
1. ✅ **NIH-plug Contribution Guidelines Research** - Discovered no formal CONTRIBUTING.md; inferred standards from codebase and CI/CD
2. ✅ **Context System Architecture Analysis** - Documented trait-based abstraction layer and wrapper implementations
3. ✅ **VST3 Wrapper Implementation Documentation** - Complete understanding of IComponent, IEditController, IAudioProcessor patterns
4. ✅ **CLAP Wrapper Implementation Documentation** - Complete understanding of extension registration and lifecycle
5. ✅ **Testing Patterns Research** - Embedded unit tests, example plugins as integration tests, CI matrix testing
6. ✅ **VST3/CLAP Track Info API Specifications** - Researched IInfoListener and track-info extension with official documentation links
7. ✅ **Complete Implementation Strategy** - Step-by-step guide for adding both VST3 and CLAP track info support

**Decision:** Proceeding with Option 1 from previous handoff - Extend NIH-plug via hybrid fork approach to enable auto track name sync.

## Critical References

1. **Research Document**: `thoughts/repos/Parlando/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Complete implementation strategy with code examples
2. **Phase 1.5 Specification**: `Parlando Plugin Phase 1.5 Update Guide.md:516-576` - Track name research findings and decision rationale
3. **OSC Protocol Spec**: `Parlando OSC Protocol Specification.md` - Bidirectional protocol for app communication
4. **Previous Handoff**: `thoughts/shared/handoffs/general/2025-10-24_12-42-32_track-name-research-completion.md` - Context on track name priority

## Recent Changes

**Documentation Created:**
- `thoughts/repos/Parlando/shared/research/2025-10-24-nih-plug-track-info-extension.md:1-637` - Comprehensive research document with implementation strategy
- `thoughts/repos/Parlando/shared/handoffs/general/2025-10-24_13-43-23_nih-plug-track-info-research.md:1-end` - This handoff

**No Code Changes:** This session focused entirely on research and planning. The Parlando plugin codebase remains at Phase 1 complete state.

## Learnings

### NIH-plug Architecture Patterns

1. **Context System Design** (`/tmp/nih-plug/src/context/`)
   - Three core traits: `InitContext`, `ProcessContext`, `GuiContext`
   - Each wrapper (VST3/CLAP/Standalone) implements these traits
   - Data flows through contexts to maintain plugin-host abstraction
   - Example: Transport info accessed via `ProcessContext::transport()`

2. **VST3 Extension Pattern** (`/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:41-49`)
   - Use `#[VST3(implements(...))]` macro to declare COM interfaces
   - Store shared state in `Arc<WrapperInner<P>>` with atomic synchronization
   - Example: `INoteExpressionController` at line 1710-1782

3. **CLAP Extension Pattern** (`/tmp/nih-plug/src/wrapper/clap/wrapper.rs:169-239`)
   - Store vtable structs as wrapper fields
   - Return from `get_extension()` based on plugin capabilities
   - Query host extensions in `init()` and cache in `AtomicRefCell<Option<ClapPtr<T>>>`
   - Example: `voice_info` extension for polyphonic modulation

4. **Thread Safety Strategy**
   - `AtomicCell` for simple copy-able data
   - `AtomicRefCell` for collections (prevents blocking audio thread)
   - `Arc` for shared ownership across threads
   - Task-based communication via `ArrayQueue<Task<P>>`

5. **Testing Approach**
   - Embedded unit tests in `#[cfg(test)]` modules
   - Example plugins serve as integration tests
   - No mock contexts - use dummy backend for testing without hardware
   - CI tests on Linux/macOS/Windows with locked dependencies

### VST3 IInfoListener Specifics

- **Interface**: `Steinberg::Vst::ChannelContext::IInfoListener`
- **Callback**: `setChannelContextInfos(IAttributeList* list)`
- **Data Available**: Track name, color (ARGB), index, UID, plugin location (pre/post fader)
- **Thread**: UI/main thread
- **Pattern**: Push-based (host calls plugin when changes occur)
- **SDK Issues**: Versions 3.7.0-3.7.1 have string copy bug

### CLAP track-info Extension Specifics

- **Extension ID**: `"clap.track-info/1"`
- **Host Interface**: `clap_host_track_info_t` with `get()` method
- **Plugin Interface**: `clap_plugin_track_info_t` with `changed()` callback
- **Data Available**: Track name, color (RGBA), channel count, track type flags (master/bus/return)
- **Thread**: Main thread
- **Pattern**: Pull-based (plugin queries host) with optional change notification

### Key Implementation Requirements

1. **Add `TrackInfo` struct** to `/tmp/nih-plug/src/context.rs` with optional fields for cross-standard compatibility
2. **VST3**: Add `IInfoListener` to implements macro, store in `WrapperInner`, implement callback
3. **CLAP**: Add vtable field, query host extension, implement `changed()` callback
4. **Context Method**: Add `track_info() -> Option<&TrackInfo>` to `ProcessContext` trait
5. **Thread Safety**: Use `AtomicRefCell` for storage, ensure audio thread can safely read
6. **Backwards Compatibility**: Default trait implementation returns `None`

### NIH-plug Contribution Standards (Inferred)

- **No formal CONTRIBUTING.md** - Standards derived from codebase patterns
- **Rust nightly required** - SIMD features mandate nightly toolchain
- **Testing required**: Must pass on Linux/macOS/Windows
- **Lockfile must be current**: CI uses `--locked` flag
- **VST3-free build must succeed**: GPL compliance check
- **Format code**: Uses rustfmt with minimal config (`.rustfmt.toml`)
- **Debug assertions**: Use `nih_debug_assert!` for validation
- **No allocations in DSP**: Use `assert_process_allocs` feature for testing

## Artifacts

**Research Document:**
- `thoughts/repos/Parlando/shared/research/2025-10-24-nih-plug-track-info-extension.md` - Complete implementation guide with:
  - Architecture analysis of NIH-plug context system
  - VST3 IInfoListener implementation strategy
  - CLAP track-info extension implementation strategy
  - Step-by-step code examples for both wrappers
  - Testing strategy and example plugin outline
  - Fork and contribution workflow
  - Official VST3 and CLAP documentation links

**Specifications (Existing):**
- `Parlando Plugin Phase 1.5 Update Guide.md` - Plugin updates needed before Phase 2
- `Parlando OSC Protocol Specification.md` - Complete bidirectional OSC protocol

**NIH-plug Clone:**
- `/tmp/nih-plug/` - Local clone at commit `28b149e` for research purposes

## Action Items & Next Steps

### Immediate: Fork and Begin Implementation

1. **Fork NIH-plug on GitHub**
   ```bash
   # Fork https://github.com/robbert-vdh/nih-plug to lrhodin/nih-plug
   git clone https://github.com/lrhodin/nih-plug.git
   cd nih-plug
   git remote add upstream https://github.com/robbert-vdh/nih-plug.git
   git checkout -b feature/track-info-support
   ```

2. **Implement Core TrackInfo Struct**
   - Add to `/tmp/nih-plug/src/context.rs`
   - Include fields: name, color (RGBA), index, UID, track_type flags
   - Add to `ProcessContext` trait: `fn track_info(&self) -> Option<&TrackInfo>`

3. **Implement VST3 IInfoListener**
   - Modify `wrapper.rs:41` - Add to `#[VST3(implements(...))]` macro
   - Modify `inner.rs:30-140` - Add `track_info: AtomicRefCell<Option<TrackInfo>>` field
   - Implement interface in `wrapper.rs` - Extract data from IAttributeList
   - Update `context.rs` - Implement `track_info()` method

4. **Implement CLAP track-info Extension**
   - Modify `wrapper.rs:169-239` - Add `clap_plugin_track_info` vtable field
   - Modify `wrapper.rs:1839-1860` - Query host extension in `init()`
   - Implement `ext_track_info_changed()` callback function
   - Update `context.rs` - Implement `track_info()` method

5. **Create Example Plugin**
   - Create `plugins/examples/track_info_display/`
   - Demonstrate accessing track info via `ProcessContext::track_info()`
   - Log track name and color changes
   - Export both VST3 and CLAP

6. **Add Tests**
   - Unit tests in `context.rs` for TrackInfo equality and defaults
   - Ensure CI passes on all platforms

### Testing Phase

7. **Build and Test in Multiple DAWs**
   - Logic Pro (VST3 IInfoListener support)
   - Ableton Live (both formats)
   - Reaper (excellent CLAP support)
   - Bitwig Studio (CLAP native)

8. **Verify Track Info Updates**
   - Track name changes
   - Track color changes
   - Track type detection (CLAP: master/bus/return)

### Contribution Phase

9. **Submit Pull Request to Upstream**
   - Ensure all CI checks pass
   - Write clear PR description referencing VST3/CLAP specs
   - Include example plugin demonstrating feature
   - Link to official documentation for both standards

10. **Use Fork in Parlando During Review**
    - Update `Cargo.toml`: `nih_plug = { git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support" }`
    - Implement Phase 1.5 musician naming with track info access
    - Transition to upstream when PR merges

### After NIH-plug Extension

11. **Implement Parlando Phase 1.5** (as specified in update guide)
    - Use track info for musician naming if available
    - Fallback to "Musician Number" parameter if not
    - Complete UUID generation, discovery protocol, primary election

## Other Notes

### Project Context

**Current Status:**
- **Phase 1**: ✅ COMPLETE - VST3 plugin with OSC→MIDI conversion
- **Phase 1.5**: 📋 SPECIFIED - Blocked on track name detection
- **Phase 2**: 📋 PARTIALLY SPECIFIED - OSC protocol complete, app implementation TODO

**User Preference:** Strongly desires auto track name sync ("I really want to make the auto track name -> musician name sync work")

### Key NIH-plug Directories

**Core Framework:**
- `/tmp/nih-plug/src/context/` - Context trait definitions
- `/tmp/nih-plug/src/wrapper/vst3/` - VST3 wrapper implementation
- `/tmp/nih-plug/src/wrapper/clap/` - CLAP wrapper implementation
- `/tmp/nih-plug/src/wrapper/util/` - Shared utilities

**Testing:**
- `/tmp/nih-plug/plugins/examples/` - Example plugins
- Embedded tests in source files with `#[cfg(test)]` modules

**Build System:**
- `/tmp/nih-plug/.github/workflows/` - CI configuration
- `/tmp/nih-plug/xtask/` - Custom build tasks

### Important Parallels in Codebase

1. **Transport Info** (`context/process.rs:104-338`) - Similar pattern to track info (read-only host data)
2. **Voice Info** (CLAP only, `wrapper/clap/wrapper.rs:238`) - Example of conditional extension
3. **Note Expressions** (VST3, `wrapper/vst3/note_expressions.rs`) - Example of bidirectional event translation

### Official Documentation Links (from research)

**VST3:**
- IInfoListener: https://steinbergmedia.github.io/vst3_doc/vstinterfaces/classSteinberg_1_1Vst_1_1ChannelContext_1_1IInfoListener.html
- SDK: https://github.com/steinbergmedia/vst3sdk

**CLAP:**
- track-info header: https://github.com/free-audio/clap/blob/main/include/clap/ext/track-info.h
- Main repo: https://github.com/free-audio/clap

### Estimated Timeline

- **NIH-plug extension**: 1-2 weeks full-time (per previous handoff estimate)
- **PR review**: 1-2 weeks (typical for NIH-plug PRs)
- **Parlando Phase 1.5**: 1-2 days after NIH-plug fork is usable

### Open Questions for Implementation

1. Should `TrackInfo` be available in `InitContext` too? (Some hosts provide it during initialization)
2. Should we create a dedicated `TrackContext` trait? (Cleaner separation vs. added complexity)
3. How to handle Standalone wrapper? (Currently would return None; could add user-configurable name)

These can be discussed in the NIH-plug PR based on maintainer feedback.

### Success Criteria

- ✅ VST3 plugins can receive track name and color via IInfoListener
- ✅ CLAP plugins can query track info via track-info extension
- ✅ Plugins access data through unified `TrackInfo` struct
- ✅ Example plugin demonstrates feature in both formats
- ✅ All CI tests pass on Linux/macOS/Windows
- ✅ PR accepted to upstream NIH-plug
- ✅ Parlando uses feature for automatic musician naming
