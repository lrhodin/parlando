---
date: 2025-10-23T16:24:00-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 9c1a50cdcfd376c4f275f1e0d075474505c60cbb
branch: master
repository: Parlando
topic: "Phase 2 Technical Specification Planning"
tags: [phase-2, architecture, osc-protocol, plugin-updates, technical-specification]
status: in_progress
last_updated: 2025-10-23
last_updated_by: Claude (Sonnet 4.5)
type: implementation_strategy
---

# Handoff: Phase 2 Technical Specification & Architecture Planning

## Task(s)

**Status: In Progress - Phase 2 Planning**

Working on comprehensive technical specifications for Parlando Phase 2 (App Development). Following Option A approach: create detailed technical specifications before implementing the app.

**Completed:**
1. ✅ **OSC Communication Protocol Specification** - Complete bidirectional protocol between app and plugin
2. ✅ **Plugin Phase 1.5 Update Guide** - Required plugin changes before app development
3. ✅ **Initial audit** - Verified Phase 1 plugin aligns with technical guide

**In Progress:**
4. 🚧 **App Technical Implementation Guide** - Data structures, threading model, LLM integration (not started)

**Planned:**
5. ⏳ File format specifications (instrument definitions, sessions)
6. ⏳ LLM integration architecture detailed design
7. ⏳ Threading and concurrency model specifications
8. ⏳ Error handling specifications

## Critical References

1. **Parlando OSC Protocol Specification.md** - Complete OSC message definitions, discovery protocol, primary election
2. **Parlando Plugin Phase 1.5 Update Guide.md** - Plugin changes required before Phase 2
3. **Parlando Technical Implementation Guide.md** - Original Phase 1 technical guide (reference for style/structure)

## Recent Changes

**New Documents Created:**
- `Parlando OSC Protocol Specification.md:1-end` - Complete OSC protocol with collision detection
- `Parlando Plugin Phase 1.5 Update Guide.md:1-end` - Plugin update implementation guide

**No Code Changes:** This session focused entirely on specification and planning documents.

## Learnings

### Key Architecture Decisions

1. **UUID + Sequence Numbers for Collision Detection**
   - Each plugin generates random 8-char hex UUID (4 bytes)
   - Sequence numbers distinguish plugin restart from UUID collision
   - App can detect: collision (sequence goes backward, plugin alive) vs restart (sequence reset, plugin timed out)
   - See: `Parlando OSC Protocol Specification.md:45-125`

2. **Simplified Port Strategy**
   - Discovery: Fixed port 30000 (all plugins announce here)
   - Transport: Fixed port 20000 (only primary plugin sends here)
   - Plugin RX: Random ports 9000-19999 (no inter-plugin coordination needed)
   - Eliminates need for per-plugin TX ports
   - See: `Parlando OSC Protocol Specification.md:26-43`

3. **Primary Plugin Election**
   - Only one plugin sends transport data (eliminates redundancy)
   - App designates first registered plugin as primary
   - Heartbeat monitoring detects failures (15s timeout)
   - App automatically promotes new primary on failure
   - See: `Parlando OSC Protocol Specification.md:152-186`

4. **Musician Naming Strategy** (after discussion)
   - Priority 1: Try to detect track name from DAW (research needed)
   - Priority 2: Use "Musician Number" parameter → "Musician 1", "Musician 2", etc.
   - Priority 3: Phase 3 app feature for custom renaming
   - Rejected dropdown approach (too limiting, user wants simple number parameter)
   - See: `Parlando Plugin Phase 1.5 Update Guide.md:186-290`

5. **Persistent Announcements**
   - Plugins re-announce every 5 seconds until acknowledged
   - Works regardless of launch order (app before plugins or vice versa)
   - Switches to heartbeat mode after registration
   - See: `Parlando OSC Protocol Specification.md:130-151`

### Important Context

- **User won't launch until all phases complete** - Landing page/user guide describe full v1.0, not just Phase 1
- **Phase 1 plugin is excellent** - Perfectly matches technical spec, production-ready
- **Phase 1.5 is prerequisite** - Plugin must be updated before Phase 2 app development begins
- **Track name detection uncertain** - NIH-plug may not expose track names; needs research before implementation

## Artifacts

**Specifications Created:**
1. `Parlando OSC Protocol Specification.md` - Complete protocol (discovery, performance, transport messages)
2. `Parlando Plugin Phase 1.5 Update Guide.md` - Plugin implementation guide with code examples

**Existing Documents Reviewed:**
- `Parlando Landing Page.md` - Marketing copy (for v1.0 launch)
- `Parlando User Guide.md` - End-user documentation (for v1.0)
- `Parlando Technical Implementation Guide.md` - Original Phase 1 spec
- `IMPLEMENTATION_STATUS.md` - Project status tracking
- `README.md` - Plugin README
- `src/lib.rs` - Phase 1 plugin implementation (459 lines)
- `src/osc.rs` - OSC communication layer (127 lines)
- `test-harness/src/main.rs` - Test suite (227 lines)

## Action Items & Next Steps

### Immediate Next Steps

1. **Research Track Name Detection**
   - Investigate NIH-plug API for track name access
   - Test in multiple DAWs (Logic, Ableton, FL Studio, Cubase)
   - Document findings in Phase 1.5 guide
   - Decide: implement track name detection or use parameter fallback

2. **Create App Technical Implementation Guide**
   - Complete data structures (all fields defined, not pseudocode)
   - Plugin discovery & registration logic
   - Collision detection implementation
   - Musician management system
   - Threading model (main UI thread, async LLM, real-time OSC)
   - State management pattern

3. **Design LLM Integration Architecture**
   - API choice: X.AI Grok-4-Fast-Reasoning vs alternatives
   - Request/response format for teach mode
   - Request/response format for perform mode
   - Structured output strategy (function calling vs JSON mode)
   - Context window management
   - Error handling and retry logic

4. **Specify File Formats**
   - `.parlando` instrument definition format (JSON structure)
   - Session save format
   - Performance history storage
   - Storage locations (`Documents/Parlando/...`)

5. **Define Performance Generation Pipeline**
   - LLM output → math expressions conversion
   - Expression evaluation using `meval` crate
   - CC curve generation from functions
   - Note sequence building
   - Real-time OSC streaming strategy

### Before Implementation

- Complete all specification documents (aim for Phase 1 level of detail)
- Prototype key uncertainties (LLM structured output, egui+async)
- Get user approval on architectures
- Then begin Phase 1.5 plugin updates
- Finally start Phase 2 app development

## Other Notes

### Project Structure Context

- **Phase 1 Complete** ✅ - VST3 plugin with OSC→MIDI conversion
- **Phase 1.5 Required** - Plugin updates for discovery protocol
- **Phase 2 Planned** - Standalone app with LLM integration
- **Phase 3 Future** - Hierarchical musicians, Composer AI

### Code Quality Observations

- Phase 1 plugin is exceptionally well-implemented
- Clean architecture, proper error handling, thread-safe
- Test harness comprehensive (6 test suites)
- Build system professional (NIH-plugin-template integration)
- Ready foundation for Phase 2

### Key Files to Understand

**Plugin Implementation:**
- `src/lib.rs:1-459` - Main plugin, will need updates for Phase 1.5
- `src/osc.rs:1-129` - OSC layer, reusable for app

**Documentation:**
- `Parlando OSC Protocol Specification.md` - **START HERE** for protocol understanding
- `Parlando Plugin Phase 1.5 Update Guide.md` - Plugin changes needed
- `Parlando Technical Implementation Guide.md:84-212` - Phase 2 section (needs expansion)

### User Preferences

- Wants track name auto-detection first (research required)
- Prefers simple "Musician Number" parameter over dropdown
- Wants to delay launch until all features complete
- Following Option A: spec first, then implement
- Emphasized importance of detailed specs (like Phase 1 had)

### Open Questions to Resolve

1. Does NIH-plug expose track names? (requires research)
2. What's the exact LLM API format for Grok-4? (need to test)
3. How should we structure CC curve generation? (math expressions vs keyframes vs curve types)
4. What egui state management pattern? (Elm architecture vs ad-hoc)
5. How to handle scheduled events? (Phase 2 or later?)
