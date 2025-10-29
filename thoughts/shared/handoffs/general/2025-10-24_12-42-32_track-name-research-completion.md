---
date: 2025-10-24T12:42:32-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 202ec0d983c263716458d4cf62ff9d31bc25beda
branch: master
repository: Parlando
topic: "Track Name Detection Research & Phase 2 Planning Strategy"
tags: [research, phase-1.5, nih-plug, track-names, vst3, clap, implementation-strategy]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: Track Name Detection Research & NIH-plug Extension Strategy

## Task(s)

**Status: COMPLETE ✅**

Resumed from handoff `thoughts/shared/handoffs/general/2025-10-23_16-24-00_phase-2-technical-specification.md` to research track name detection capabilities for Phase 1.5 plugin updates.

**Completed:**
1. ✅ **Track Name Detection Research** - Comprehensive investigation of NIH-plug, VST3, and CLAP APIs
2. ✅ **Implementation Decision** - Documented decision to use "Musician Number" parameter strategy
3. ✅ **Extension Strategy Discussion** - Designed hybrid fork approach for adding track name support to NIH-plug
4. ✅ **Repository Setup** - Configured GitHub remote, cleaned up local files, committed specifications

**Key Finding:** While VST3 (IInfoListener) and CLAP (track-info extension) fully support track name detection, NIH-plug does not currently expose these APIs to plugin developers.

## Critical References

1. **Parlando Plugin Phase 1.5 Update Guide.md** - Updated with complete track name research findings and implementation decision
2. **Parlando OSC Protocol Specification.md** - OSC protocol for Phase 1.5 (UUID collision detection, primary election)
3. Original handoff: `thoughts/shared/handoffs/general/2025-10-23_16-24-00_phase-2-technical-specification.md`

## Recent Changes

**Documentation:**
- `Parlando Plugin Phase 1.5 Update Guide.md:516-576` - Added complete track name research section
- `Parlando Plugin Phase 1.5 Update Guide.md:153-276` - Simplified musician naming implementation (removed track name detection code)
- `.gitignore:30-34` - Added `thoughts/` and `scripts/` exclusions

**Repository Management:**
- Deleted `Parlando/` directory (unused Obsidian vault)
- Moved utility scripts from `../humanlayer 2/hack/` to `scripts/` (gitignored)
- Configured GitHub remote: `https://github.com/lrhodin/Parlando.git`
- Pushed commit `202ec0d` with Phase 2 specifications

**No code changes** - Plugin remains at Phase 1 complete state

## Learnings

### Track Name Detection Research

1. **VST3 Support (But Not Exposed):**
   - Interface: `Vst::ChannelContext::IInfoListener` (since VST 3.6.5, 2015)
   - Method: `setChannelContextInfos(IAttributeList*)`
   - Provides: `kChannelNameKey`, `kChannelColorKey`, `kChannelIndexKey`, `kChannelUIDKey`
   - **NIH-plug status:** ❌ NOT IMPLEMENTED (confirmed by searching entire codebase)

2. **CLAP Support (But Not Exposed):**
   - Extension ID: `"clap.track-info/1"`
   - Method: `clap_host_track_info_t::get()`
   - Provides: Track name, color, channel count, track type flags
   - **NIH-plug status:** ❌ NOT IMPLEMENTED (no references found)

3. **NIH-plug Context APIs:**
   - Examined `/tmp/nih-plug/src/context/init.rs` and `context/process.rs`
   - `InitContext` only provides: `plugin_api()`, `execute()`, `set_latency_samples()`, `set_current_voice_capacity()`
   - No track/channel name methods exist in the framework

4. **Implementation Decision:**
   - Use "Musician Number" parameter (1-999) formatted as "Musician {N}"
   - Simple, predictable, DAW-agnostic behavior
   - User has explicit control over numbering
   - Phase 3 app can add custom naming UI

### NIH-plug Extension Strategy

**Hybrid Fork Approach (Recommended):**
1. Fork NIH-plug to `lrhodin/nih-plug`
2. Create feature branch: `feature/track-info-support`
3. Implement VST3 IInfoListener + CLAP track-info extension
4. Update Parlando `Cargo.toml` to use fork: `nih_plug = { git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support" }`
5. Submit PR to upstream NIH-plug
6. Switch back to upstream when PR merges

**Effort Estimate:** 1-2 weeks full-time, 1 month part-time

**Benefits:** Unblocks Parlando development, contributes to community, clean long-term solution

## Artifacts

**Specifications:**
1. `Parlando OSC Protocol Specification.md` (20.7 KB) - Complete bidirectional OSC protocol
2. `Parlando Plugin Phase 1.5 Update Guide.md` (22.3 KB) - Plugin updates with track name research

**Key Sections:**
- `Parlando Plugin Phase 1.5 Update Guide.md:518-576` - Track name research findings and decision
- `Parlando Plugin Phase 1.5 Update Guide.md:153-276` - Musician naming implementation
- `Parlando OSC Protocol Specification.md:67-143` - UUID + sequence collision detection
- `Parlando OSC Protocol Specification.md:220-323` - Message specifications

**Repository:**
- `.gitignore:30-34` - Local development exclusions
- Commit: `202ec0d983c263716458d4cf62ff9d31bc25beda`
- Remote: `https://github.com/lrhodin/Parlando`

## Action Items & Next Steps

### Decision Point: Implementation Path

The user expressed strong interest in auto track name sync. Three paths forward:

#### Option 1: Extend NIH-plug (Hybrid Fork Strategy)
**Timeline:** 2 weeks full-time / 1 month part-time

**Steps:**
1. Fork NIH-plug on GitHub
2. Design track info API (`TrackContext` trait, `TrackInfo` struct)
3. Implement VST3 `IInfoListener` interface in wrapper
4. Implement CLAP `track-info` extension in wrapper
5. Test in Logic, Ableton, Bitwig, Reaper
6. Submit PR to upstream
7. Use fork in Parlando during review period
8. Transition to upstream when merged

**API Design Preview:**
```rust
pub trait TrackContext {
    fn track_info(&self) -> Option<TrackInfo>;
}

pub struct TrackInfo {
    pub name: Option<String>,
    pub color: Option<Color>,
    pub index: Option<i32>,
}
```

#### Option 2: Proceed with Musician Number Parameter
**Timeline:** Ready to implement now

**Steps:**
1. Begin Phase 1.5 plugin implementation
2. Add UUID generation and collision detection
3. Add musician number parameter
4. Implement announcement/heartbeat protocol
5. Test multi-instance behavior
6. Launch v1.0 with manual naming
7. Add auto track names in v1.1 (after NIH-plug extension)

#### Option 3: Continue Phase 2 App Specifications
**Timeline:** 1-2 weeks for complete app spec

**Steps:**
1. Create "App Technical Implementation Guide"
2. Design LLM integration architecture (Grok-4-Fast-Reasoning)
3. Define file formats (`.parlando`, sessions)
4. Specify performance generation pipeline
5. Design threading model (UI, async LLM, real-time OSC)

### Recommended: Option 1 (Extend NIH-plug)

User expressed: "I really want to make the auto track name -> musician name sync work"

This indicates auto track names are a priority. The hybrid fork strategy provides:
- ✅ Best UX (tracks show as "Bass Synth" not "Musician 5")
- ✅ Contributes to open source community
- ✅ Clean long-term solution (no fork maintenance once merged)
- ✅ Doesn't block Parlando development (use fork immediately)

## Other Notes

### Project Status

**Phase 1:** ✅ COMPLETE
- VST3 plugin with OSC→MIDI conversion (src/lib.rs:459 lines)
- Multi-instance support (10,000 simultaneous instances)
- Transport sync, MIDI pass-through
- Production-ready and well-tested

**Phase 1.5:** 📋 SPECIFIED (Ready to implement)
- UUID generation and collision detection
- Random RX port binding (9000-19999)
- Discovery protocol with persistent announcements
- Musician naming strategy (parameter-based or auto-detected)
- Primary plugin election system

**Phase 2:** 📋 PARTIALLY SPECIFIED
- OSC protocol complete
- Plugin updates specified
- App implementation guide TODO
- LLM integration architecture TODO

### Repository Structure

```
Parlando/
├── src/
│   ├── lib.rs (459 lines) - Phase 1 plugin
│   └── osc.rs (127 lines) - OSC layer
├── Parlando OSC Protocol Specification.md - Complete protocol
├── Parlando Plugin Phase 1.5 Update Guide.md - Implementation guide
├── Parlando Technical Implementation Guide.md - Original Phase 1 spec
├── IMPLEMENTATION_STATUS.md - Project tracking
├── README.md - Plugin documentation
└── .gitignore - Excludes thoughts/, scripts/
```

### NIH-plug Fork Implementation Details

If proceeding with extension, key files to modify:

**New files:**
- `src/context/track.rs` - TrackContext trait and TrackInfo struct

**VST3 wrapper:**
- `src/wrapper/vst3/wrapper.rs` - Implement IInfoListener
- `src/wrapper/vst3/inner.rs` - Query channel context

**CLAP wrapper:**
- `src/wrapper/clap/wrapper.rs` - Query track-info extension
- `src/wrapper/clap/plugin.rs` - Handle extension lifecycle

**Context implementations:**
- `src/context/init.rs` - Add TrackContext to InitContext
- `src/context/process.rs` - Add TrackContext to ProcessContext

### Resources

- NIH-plug repo: https://github.com/robbert-vdh/nih-plug
- VST3 IInfoListener docs: https://steinbergmedia.github.io/vst3_dev_portal/pages/Technical+Documentation/Change+History/3.6.5/IInfoListener.html
- CLAP track-info spec: https://github.com/free-audio/clap/blob/main/include/clap/ext/track-info.h
- Parlando GitHub: https://github.com/lrhodin/Parlando

### User Preferences

- Wants auto track name sync to work ("really want to make this work")
- Understands NIH-plug extension is significant effort but valuable
- Interested in hybrid fork strategy (use fork, contribute upstream, transition back)
- Following "Option A" approach: spec first, then implement
- Won't launch until all phases complete (wants complete v1.0)
