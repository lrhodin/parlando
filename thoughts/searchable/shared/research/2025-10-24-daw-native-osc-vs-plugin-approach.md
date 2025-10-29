---
date: 2025-10-24T13:18:54-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 202ec0d983c263716458d4cf62ff9d31bc25beda
branch: master
repository: Parlando
topic: "DAW Native OSC Support vs Plugin Approach for Parlando"
tags: [research, osc, bitwig, ableton, logic, plugin-architecture, daw-compatibility]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
---

# Research: DAW Native OSC Support vs Plugin Approach for Parlando

**Date**: 2025-10-24T13:18:54-07:00
**Researcher**: Claude (Sonnet 4.5)
**Git Commit**: 202ec0d983c263716458d4cf62ff9d31bc25beda
**Branch**: master
**Repository**: Parlando

## Research Question

Can we use native OSC support in DAWs (particularly Bitwig, Ableton, and Logic) instead of developing a plugin? Would this approach provide track name access without setup requirements while maintaining compatibility across major DAWs?

## Executive Summary

**The plugin approach is strongly recommended.** None of the major DAWs have sufficient native OSC support to replace the Parlando plugin:

- **Bitwig**: No native OSC - requires third-party controller extension (DrivenByMoss)
- **Ableton Live**: No native OSC - requires Max for Live devices or remote scripts
- **Logic Pro**: Limited native OSC - receive-only, no track name queries, requires Control Surface framework

All three DAWs would require significant third-party additions and complex setup to achieve what the Parlando plugin provides out of the box. The plugin approach offers superior compatibility, zero configuration, and consistent behavior across all DAWs.

## Detailed Findings

### Bitwig Studio OSC Capabilities

#### Native Support
**Finding**: Bitwig has **NO native OSC support** built-in.

All OSC functionality requires installing the **DrivenByMoss** controller extension by Jürgen Moßgraber, which has become the de facto standard for OSC control in Bitwig.

#### Setup Requirements
1. Download and install DrivenByMoss extension (~3MB)
2. Configure virtual MIDI port (for note input)
3. Add "Open Sound Control" controller in Bitwig settings
4. Configure network ports (typically receive: 8000, send: 9000)
5. Restart extension after any configuration changes

#### Track Name Access
**Yes**, track names can be queried via OSC with DrivenByMoss:
- `/track/{1-8}/name` - Query specific track names
- `/track/selected/name` - Query selected track
- **Limitation**: Default bank size is 8 tracks (configurable)

#### MIDI Control via OSC
Supported through virtual MIDI keyboard commands:
- `/vkb_midi/{channel}/note/{note} {velocity}` - Note on/off
- `/vkb_midi/{channel}/cc/{number} {value}` - Control changes
- `/vkb_midi/{channel}/pitch {value}` - Pitch bend
- **Critical limitation**: Requires routing through virtual MIDI port (OSC → Virtual MIDI → Bitwig)

#### Transport State
Comprehensive transport control available:
- `/play`, `/stop`, `/record` - Transport control
- `/tempo/raw` - Tempo query/set
- `/beat/str` - Position in measures.beats
- `/time/signature` - Time signature

#### Fundamental Limitations
Controller extensions (including DrivenByMoss) operate at **"control rate"** not **"audio rate"**:
- Cannot process audio/MIDI in real-time
- No access to audio buffers or MIDI streams
- Cannot be inserted in the signal chain
- Performance impact with high message rates

### Ableton Live OSC Support

#### Native Support
**Finding**: Ableton Live has **NO native OSC support**.

All OSC functionality requires either:
- Max for Live devices (requires Suite edition or separate purchase)
- Third-party remote scripts (e.g., AbletonOSC)

#### Most Comprehensive Solution: AbletonOSC
**AbletonOSC** (third-party Python remote script):
- Requires Live 11 or higher
- Listens on port 11000, sends on port 11001

#### Track Name Access
**Yes**, with AbletonOSC:
```
/live/song/get/track_names [index_min, index_max]
```
Returns: `('1-MIDI', '2-MIDI', '3-Audio', '4-Audio')`

**Issues**:
- Confused by duplicate track names
- Problems with "#" characters in names
- Requires manual installation and configuration

#### MIDI Control via OSC
With AbletonOSC, can add MIDI notes to clips:
```
/live/clip/add/notes track_id, clip_id, pitch, start_time, duration, velocity, mute
```

**Note**: This adds notes to arrangement clips, not real-time MIDI input like the plugin provides.

#### Official Solution: Ableton Connection Kit
Free Max for Live pack providing:
- OSC Send - Parameter to OSC output
- OSC MIDI Send - MIDI to OSC output
- OSC TouchOSC - Controller mapping

**Limitations**:
- Output-focused (sending from Live, not receiving)
- Requires Max for Live knowledge for customization
- No track name querying

#### Setup Complexity
1. Install remote script or Max for Live devices
2. Configure in Preferences > Link/Tempo/MIDI
3. Set up port configuration
4. Create or load OSC devices on tracks
5. Map parameters individually

### Logic Pro OSC Capabilities

#### Native Support
**Finding**: Logic Pro has **limited native OSC support** (since v9.1.2).

**Characteristics**:
- Receive-only for most parameters
- No bidirectional feedback without plugins
- UDP/IPv4 only (no TCP, no IPv6)
- Default port: 7000

#### Track Name Access
**No native track name queries via OSC**.

**Workarounds**:
1. **OSCulator plugin** - Adds `/logic/track/name` support
2. **Mackie Control Protocol** - Parse MCU display output (max 31 chars)
3. **AppleScript UI automation** - Query UI elements (fragile)

#### MIDI Control via OSC
**Yes**, but with limitations:
- Must manually define OSC paths (no learning mode)
- Requires Bonjour/Zeroconf device announcement
- Devices must identify as TouchOSC-compatible

#### Transport Control
Available with OSCulator plugin:
- `/logic/transport/play`, `/stop`, `/record`
- `/logic/transport/time/beats` - Position info
- **Note**: Native Logic OSC is primarily receive-only

#### Setup Requirements
1. Install OSCulator plugin ($39 commercial) or similar
2. Configure Control Surface in Logic
3. Manually define OSC message paths (Cmd+L on parameters)
4. Set up Bonjour service announcement

### Comparison: Plugin vs Native OSC

| Feature | Parlando Plugin | Bitwig (DrivenByMoss) | Ableton (AbletonOSC) | Logic (OSCulator) |
|---------|-----------------|------------------------|----------------------|-------------------|
| **Native Support** | Works in any DAW | Requires extension | Requires script/M4L | Limited native |
| **Track Names** | Via parameter¹ | Yes (8 tracks/bank) | Yes (with issues) | Workarounds only |
| **Setup Complexity** | Drop in plugin | Complex (5+ steps) | Complex (manual install) | Complex (paid plugin) |
| **MIDI Input** | Direct to engine | Via virtual MIDI | To clips only² | Manual mapping |
| **Transport Sync** | Bidirectional | Yes | With AbletonOSC | One-way only |
| **Multi-Instance** | 10,000 instances | Single controller | Script limitations | Unknown |
| **Configuration** | Zero | Port setup required | Port setup required | Manual paths |
| **Cost** | Free | Free (extension) | Free/M4L cost | $39 (OSCulator) |
| **Consistency** | Same everywhere | Bitwig-specific | Ableton-specific | Logic-specific |

¹ Plugin uses "Musician Number" parameter since NIH-plug doesn't expose track names
² AbletonOSC adds notes to arrangement, not real-time MIDI like plugin

### Architecture: Native OSC Approach

If using native OSC, the architecture would be dramatically different for each DAW:

#### Bitwig Architecture
```
Parlando App
    ↓ OSC to port 8000
DrivenByMoss Extension
    ↓ Controller messages
    ↓ Virtual MIDI port (for notes)
Bitwig Audio Engine
```

**Issues**:
- Controller rate, not audio rate
- Extra latency through virtual MIDI
- Single instance only

#### Ableton Architecture
```
Parlando App
    ↓ OSC to port 11000
AbletonOSC Remote Script
    ↓ Python API calls
    ↓ Clip note additions
Ableton Session/Arrangement
```

**Issues**:
- No real-time MIDI input
- Adds to clips, not live performance
- Complex state management

#### Logic Architecture
```
Parlando App
    ↓ OSC to port 7000
    ↓ Must announce as TouchOSC device
Logic Control Surface Framework
    ↓ OSCulator plugin (if bidirectional needed)
    ↓ Manual parameter mappings
Logic Audio Engine
```

**Issues**:
- No track name queries
- One-way communication
- Requires commercial plugin for full features

### Architecture: Plugin Approach (Current)

```
Parlando App
    ↓ OSC (standardized protocol)
Parlando Plugin (VST3/CLAP)
    ↓ Direct MIDI injection
    ↓ Transport feedback
    ↓ Multi-instance support
Any DAW's Audio Engine
```

**Advantages**:
- Consistent across all DAWs
- Audio-rate processing
- Direct MIDI injection
- Bidirectional transport sync
- Zero configuration

## Key Capabilities Comparison

### What the Plugin Provides That Native OSC Cannot

1. **Unified Protocol**: Same OSC messages work in every DAW
2. **Audio-Rate Processing**: Plugin runs in audio thread, not control thread
3. **MIDI Merging**: Combines OSC-generated MIDI with pass-through MIDI
4. **True Multi-Instance**: 10,000 simultaneous instances with automatic port allocation
5. **Zero Configuration**: No setup wizards, port configuration, or extensions
6. **Sample-Accurate Timing**: Preserves timing from OSC to MIDI output
7. **Transport Sync**: Bidirectional DAW state reporting (most DAWs can't do this natively)
8. **Future Discovery Protocol**: Phase 1.5 adds sophisticated instance coordination

### What Native OSC Provides

1. **No Plugin Required**: One less component to install (but requires other components)
2. **Track Names** (Bitwig/Ableton): Can query actual track names (with limitations)
3. **DAW-Specific Features**: Access to DAW-specific functionality

However, each benefit comes with significant trade-offs in setup complexity and cross-DAW compatibility.

## Recommendations

### Stick with the Plugin Approach

The plugin approach is strongly recommended for the following reasons:

1. **Compatibility**: Works identically in every VST3/CLAP host
2. **Simplicity**: Users install one plugin, no configuration required
3. **Performance**: Direct audio engine integration, no control-rate limitations
4. **Reliability**: No dependency on third-party extensions or scripts
5. **Maintenance**: One codebase to maintain vs. three different DAW integrations
6. **Future-Proof**: Plugin API more stable than undocumented OSC implementations

### If Track Names Are Critical

If automatic track name detection is essential, consider:

1. **Hybrid Fork Strategy** (from handoff): Extend NIH-plug to expose track name APIs
2. **Fallback System**: Use "Musician Number" parameter now, add track names when available
3. **Phase 3 Enhancement**: Add custom naming in the Parlando app UI

### Architecture Impact of Native OSC

If you were to pursue native OSC:

**Development Impact**:
- 3x codebase (different for each DAW)
- Complex DAW detection and routing logic
- Extensive testing matrix (versions × DAWs × configurations)

**User Impact**:
- Complex setup instructions per DAW
- Potential paid plugins (Logic: OSCulator $39)
- Inconsistent behavior across DAWs
- Support burden for configuration issues

**Performance Impact**:
- Controller-rate limitations (Bitwig)
- Virtual MIDI routing latency
- No sample-accurate timing

## Conclusion

While native DAW OSC support initially seems attractive for avoiding plugin development, the reality is that **none of the major DAWs provide sufficient native OSC capabilities** for the Parlando use case. Each would require third-party additions, complex setup, and would still have significant limitations compared to the plugin approach.

The Parlando plugin provides a consistent, zero-configuration solution that works identically across all DAWs. The Phase 1.5 improvements (UUID, discovery protocol, primary election) will make it even more sophisticated than what native OSC could provide.

**Recommended Path**: Continue with the plugin approach, implementing Phase 1.5 improvements, and consider extending NIH-plug for track name support as a future enhancement.

## Related Research

- `thoughts/shared/handoffs/general/2025-10-24_12-42-32_track-name-research-completion.md` - NIH-plug track name research
- `Parlando Plugin Phase 1.5 Update Guide.md` - Planned plugin enhancements
- `Parlando OSC Protocol Specification.md` - Complete protocol specification

## Open Questions

1. Could a hybrid approach work? (Plugin for most DAWs, native for specific ones)
2. Is the "Musician Number" parameter sufficient for v1.0?
3. Should we contribute track name support to NIH-plug upstream?