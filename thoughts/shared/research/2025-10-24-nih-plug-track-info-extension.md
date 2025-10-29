---
date: 2025-10-24T13:36:37-07:00
researcher: Claude (Sonnet 4.5)
git_commit: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
branch: master
repository: Parlando
topic: "Extending NIH-plug to Support Track Information (VST3 IInfoListener & CLAP track-info)"
tags: [research, nih-plug, vst3, clap, track-info, plugin-development, open-source]
status: complete
last_updated: 2025-10-24
last_updated_by: Claude (Sonnet 4.5)
---

# Research: Extending NIH-plug to Support Track Information

**Date**: 2025-10-24T13:36:37-07:00
**Researcher**: Claude (Sonnet 4.5)
**Git Commit**: 0e69fd381d9785fbe30074539e31e1f67f20d6fb
**Branch**: master
**Repository**: Parlando

## Research Question

How can we extend NIH-plug to support track/channel information from both VST3 (IInfoListener) and CLAP (track-info extension) standards? What are the existing patterns in the codebase, and how should this feature be implemented to align with NIH-plug's architecture and contribution standards?

## Summary

NIH-plug currently does not expose track/channel information APIs despite both VST3 and CLAP standards providing this capability. To implement this feature properly:

1. **VST3** requires implementing the `IInfoListener` interface (available since VST 3.6.5) which provides track name, color, index, and other metadata via a callback mechanism
2. **CLAP** requires implementing the `track-info` extension which provides similar data through a query-based approach
3. The implementation should follow NIH-plug's existing patterns: trait-based context abstraction, wrapper-specific implementations, and thread-safe shared state
4. Testing should follow NIH-plug's embedded unit test pattern and leverage example plugins for integration testing

## Detailed Findings

### Current NIH-plug Architecture

#### Context System (`/tmp/nih-plug/src/context/`)
The framework provides three core context traits that abstract plugin-host communication:

- **`InitContext`** - Used during plugin initialization
- **`ProcessContext`** - Used during audio processing
- **`GuiContext`** - Used for GUI interactions

Each wrapper (VST3, CLAP, Standalone) implements these traits:
- VST3: `/tmp/nih-plug/src/wrapper/vst3/context.rs`
- CLAP: `/tmp/nih-plug/src/wrapper/clap/context.rs`
- Standalone: `/tmp/nih-plug/src/wrapper/standalone/context.rs`

#### VST3 Wrapper Architecture (`/tmp/nih-plug/src/wrapper/vst3/`)
The VST3 wrapper uses a `#[VST3(implements(...))]` macro to implement COM interfaces:

```rust
#[VST3(implements(
    IComponent,
    IEditController,
    IAudioProcessor,
    IMidiMapping,
    INoteExpressionController,
    IProcessContextRequirements,
    IUnitInfo
))]
pub struct Wrapper<P: Vst3Plugin> {
    inner: Arc<WrapperInner<P>>,
}
```

The `WrapperInner` struct (`inner.rs`) contains all shared state using appropriate synchronization primitives.

#### CLAP Wrapper Architecture (`/tmp/nih-plug/src/wrapper/clap/`)
The CLAP wrapper stores extension vtables as struct fields and returns them from `get_extension()`:

```rust
pub struct Wrapper<P: ClapPlugin> {
    // ... other fields ...
    clap_plugin_params: clap_plugin_params,
    clap_plugin_latency: clap_plugin_latency,
    clap_plugin_gui: clap_plugin_gui,
    // ... more extensions ...
}
```

Extensions are conditionally returned based on plugin capabilities (e.g., GUI extension only if editor exists).

### Track Information APIs

#### VST3 IInfoListener
- **Interface**: `Steinberg::Vst::ChannelContext::IInfoListener`
- **Method**: `setChannelContextInfos(IAttributeList* list)`
- **Available Data**:
  - Track name (`kChannelNameKey`)
  - Track color (`kChannelColorKey` - ARGB)
  - Channel index (`kChannelIndexKey`)
  - Channel UID (`kChannelUIDKey`)
  - Plugin location (`kChannelPluginLocationKey` - pre/post fader)

#### CLAP track-info Extension
- **Extension ID**: `"clap.track-info/1"`
- **Host Interface**: `clap_host_track_info_t` with `get()` method
- **Plugin Interface**: `clap_plugin_track_info_t` with `changed()` callback
- **Available Data**:
  - Track name (UTF-8 string)
  - Track color (R,G,B,A struct)
  - Track type flags (master, bus, return)
  - Audio channel count
  - Audio port type

### NIH-plug Coding Standards

Based on analysis of the codebase:

1. **No formal CONTRIBUTING.md** - Standards are inferred from existing code
2. **Rust nightly required** - For SIMD features
3. **Testing approach**:
   - Embedded unit tests in `#[cfg(test)]` modules
   - Example plugins serve as integration tests
   - CI tests on Linux, macOS, Windows
4. **Threading patterns**:
   - `Arc<WrapperInner>` for shared state
   - `AtomicCell` for simple data
   - `AtomicRefCell` for collections
   - Task-based communication between threads
5. **Debug assertions** - Use `nih_debug_assert!` for validation

## Implementation Strategy

### 1. Add Track Info to Context Traits

Create a new `TrackInfo` struct in `/tmp/nih-plug/src/context.rs`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct TrackInfo {
    /// Track name from the DAW
    pub name: Option<String>,
    /// Track color as RGBA (0-255 per channel)
    pub color: Option<(u8, u8, u8, u8)>,
    /// Channel index within its namespace
    pub index: Option<i32>,
    /// Unique identifier for the track
    pub uid: Option<String>,
    /// Track type flags (master, bus, return)
    pub track_type: TrackType,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct TrackType {
    pub is_master: bool,
    pub is_bus: bool,
    pub is_return: bool,
}
```

Add method to `ProcessContext` trait:

```rust
pub trait ProcessContext<P: Plugin> {
    // ... existing methods ...

    /// Returns current track information if available
    fn track_info(&self) -> Option<&TrackInfo> {
        None // Default implementation for backwards compatibility
    }
}
```

### 2. VST3 Implementation

#### Add IInfoListener to Wrapper

Modify `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs`:

```rust
#[VST3(implements(
    IComponent,
    IEditController,
    IAudioProcessor,
    IMidiMapping,
    INoteExpressionController,
    IProcessContextRequirements,
    IUnitInfo,
    IInfoListener  // Add this
))]
pub struct Wrapper<P: Vst3Plugin> {
    inner: Arc<WrapperInner<P>>,
}
```

#### Add Track Info Storage to WrapperInner

In `/tmp/nih-plug/src/wrapper/vst3/inner.rs`:

```rust
pub struct WrapperInner<P: Vst3Plugin> {
    // ... existing fields ...

    /// Current track information from host
    pub track_info: AtomicRefCell<Option<TrackInfo>>,
}
```

#### Implement IInfoListener

In `/tmp/nih-plug/src/wrapper/vst3/wrapper.rs`:

```rust
impl<P: Vst3Plugin> IInfoListener for Wrapper<P> {
    unsafe fn set_channel_context_infos(
        &self,
        list: SharedVstPtr<dyn IAttributeList>,
    ) -> tresult {
        check_null_ptr!(kInvalidArgument, list);

        let mut track_info = TrackInfo::default();

        // Extract track name
        let mut name_u16 = [0u16; 128];
        if list.get_string(
            CStr::from_bytes_with_nul_unchecked(b"Steinberg.Vst.ChannelContext.ChannelName\0").as_ptr(),
            name_u16.as_mut_ptr(),
            name_u16.len() as i32,
        ) == kResultOk {
            track_info.name = Some(String::from_utf16_lossy(&name_u16));
        }

        // Extract track color
        let mut color: i64 = 0;
        if list.get_int(
            CStr::from_bytes_with_nul_unchecked(b"Steinberg.Vst.ChannelContext.ChannelColor\0").as_ptr(),
            &mut color,
        ) == kResultOk {
            let color_u32 = color as u32;
            track_info.color = Some((
                ((color_u32 >> 16) & 0xFF) as u8, // R
                ((color_u32 >> 8) & 0xFF) as u8,  // G
                (color_u32 & 0xFF) as u8,         // B
                ((color_u32 >> 24) & 0xFF) as u8, // A
            ));
        }

        // Store the track info
        *self.inner.track_info.borrow_mut() = Some(track_info);

        // Schedule a task to notify the plugin if needed
        let task = Task::TrackInfoChanged;
        match self.inner.schedule_gui(task) {
            Ok(_) => kResultOk,
            Err(_) => kResultOk, // Don't fail the call
        }
    }
}
```

#### Update ProcessContext Implementation

In `/tmp/nih-plug/src/wrapper/vst3/context.rs`:

```rust
impl<P: Vst3Plugin> ProcessContext<P> for WrapperProcessContext<'_, P> {
    // ... existing methods ...

    fn track_info(&self) -> Option<&TrackInfo> {
        unsafe {
            let info_ref = self.inner.track_info.borrow();
            std::mem::transmute(info_ref.as_ref())
        }
    }
}
```

### 3. CLAP Implementation

#### Add track-info Extension

In `/tmp/nih-plug/src/wrapper/clap/wrapper.rs`:

```rust
use clap_sys::ext::track_info::{
    clap_plugin_track_info, clap_host_track_info,
    clap_track_info_t, CLAP_EXT_TRACK_INFO,
};

pub struct Wrapper<P: ClapPlugin> {
    // ... existing fields ...

    /// Track info extension vtable
    clap_plugin_track_info: clap_plugin_track_info,

    /// Host track info extension
    host_track_info: AtomicRefCell<Option<ClapPtr<clap_host_track_info>>>,

    /// Current track information
    track_info: AtomicRefCell<Option<TrackInfo>>,
}
```

#### Initialize in Constructor

In `Wrapper::new()`:

```rust
clap_plugin_track_info: clap_plugin_track_info {
    changed: Some(Self::ext_track_info_changed),
},
host_track_info: AtomicRefCell::new(None),
track_info: AtomicRefCell::new(None),
```

#### Query Host Extension

In `init()` function:

```rust
*wrapper.host_track_info.borrow_mut() = query_host_extension::<clap_host_track_info>(
    &wrapper.host_callback,
    CLAP_EXT_TRACK_INFO,
);

// Query initial track info
wrapper.update_track_info();
```

#### Implement Extension Functions

```rust
unsafe extern "C" fn ext_track_info_changed(plugin: *const clap_plugin) {
    check_null_ptr!((), plugin, (*plugin).plugin_data);
    let wrapper = &*((*plugin).plugin_data as *const Self);

    wrapper.update_track_info();

    // Schedule GUI update if needed
    wrapper.schedule_gui(Task::TrackInfoChanged);
}

impl<P: ClapPlugin> Wrapper<P> {
    fn update_track_info(&self) {
        if let Some(host_track_info) = &*self.host_track_info.borrow() {
            let mut info = clap_track_info_t {
                flags: 0,
                name: [0; CLAP_NAME_SIZE],
                color: clap_color_t { red: 0, green: 0, blue: 0, alpha: 255 },
                audio_channel_count: 0,
                audio_port_type: std::ptr::null(),
            };

            if unsafe { clap_call!(host_track_info=>get(&*self.host_callback, &mut info)) } {
                let mut track_info = TrackInfo::default();

                if info.flags & CLAP_TRACK_INFO_HAS_TRACK_NAME != 0 {
                    track_info.name = Some(
                        CStr::from_ptr(info.name.as_ptr())
                            .to_string_lossy()
                            .into_owned()
                    );
                }

                if info.flags & CLAP_TRACK_INFO_HAS_TRACK_COLOR != 0 {
                    track_info.color = Some((
                        info.color.red,
                        info.color.green,
                        info.color.blue,
                        info.color.alpha,
                    ));
                }

                track_info.track_type = TrackType {
                    is_master: info.flags & CLAP_TRACK_INFO_IS_FOR_MASTER != 0,
                    is_bus: info.flags & CLAP_TRACK_INFO_IS_FOR_BUS != 0,
                    is_return: info.flags & CLAP_TRACK_INFO_IS_FOR_RETURN_TRACK != 0,
                };

                *self.track_info.borrow_mut() = Some(track_info);
            }
        }
    }
}
```

#### Register Extension

In `get_extension()`:

```rust
} else if id == CLAP_EXT_TRACK_INFO {
    &wrapper.clap_plugin_track_info as *const _ as *const c_void
```

#### Update ProcessContext

Similar to VST3, add `track_info()` method implementation.

### 4. Testing Strategy

#### Unit Tests

Add tests in `/tmp/nih-plug/src/context.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn track_info_default() {
        let info = TrackInfo::default();
        assert_eq!(info.name, None);
        assert_eq!(info.color, None);
        assert_eq!(info.index, None);
        assert_eq!(info.uid, None);
        assert!(!info.track_type.is_master);
    }

    #[test]
    fn track_info_equality() {
        let mut info1 = TrackInfo::default();
        let mut info2 = TrackInfo::default();

        info1.name = Some("Track 1".to_string());
        info2.name = Some("Track 1".to_string());

        assert_eq!(info1, info2);
    }
}
```

#### Example Plugin

Create `/tmp/nih-plug/plugins/examples/track_info_display/`:

```rust
use nih_plug::prelude::*;
use std::sync::Arc;

struct TrackInfoDisplay {
    params: Arc<TrackInfoParams>,
    current_track_info: Arc<Mutex<Option<TrackInfo>>>,
}

#[derive(Params)]
struct TrackInfoParams {}

impl Plugin for TrackInfoDisplay {
    fn process(
        &mut self,
        buffer: &mut Buffer,
        _aux: &mut AuxiliaryBuffers,
        context: &mut impl ProcessContext<Self>,
    ) -> ProcessStatus {
        // Get track info from context
        if let Some(track_info) = context.track_info() {
            *self.current_track_info.lock() = Some(track_info.clone());

            nih_log!("Track: {:?}, Color: {:?}",
                     track_info.name,
                     track_info.color);
        }

        ProcessStatus::Normal
    }
}

nih_export_clap!(TrackInfoDisplay);
nih_export_vst3!(TrackInfoDisplay);
```

#### Integration Testing

1. Build the example plugin
2. Load in various DAWs (Logic, Ableton, Reaper, Bitwig)
3. Verify track name appears in logs
4. Change track color and verify update
5. Test on different track types (master, bus, return)

### 5. Documentation

Add to `/tmp/nih-plug/src/lib.rs`:

```rust
//! ## Track Information
//!
//! Plugins can access track/channel information from the host through the
//! `ProcessContext::track_info()` method. This includes track name, color,
//! and type information when available.
//!
//! ```
//! fn process(
//!     &mut self,
//!     buffer: &mut Buffer,
//!     aux: &mut AuxiliaryBuffers,
//!     context: &mut impl ProcessContext<Self>,
//! ) -> ProcessStatus {
//!     if let Some(track_info) = context.track_info() {
//!         // Use track name, color, etc.
//!         if let Some(name) = &track_info.name {
//!             nih_log!("Processing on track: {}", name);
//!         }
//!     }
//!     ProcessStatus::Normal
//! }
//! ```
//!
//! Track information is supported in VST3 (via IInfoListener) and CLAP
//! (via track-info extension). The standalone wrapper returns None.
```

## Fork and Development Process

### 1. Fork NIH-plug

```bash
# Fork on GitHub first, then:
git clone https://github.com/lrhodin/nih-plug.git
cd nih-plug
git remote add upstream https://github.com/robbert-vdh/nih-plug.git
git checkout -b feature/track-info-support
```

### 2. Development Setup

```bash
# Install Rust nightly (required for SIMD)
rustup toolchain install nightly
rustup default nightly

# Install Linux dependencies (if on Linux)
sudo apt-get install -y libasound2-dev libgl-dev libjack-dev \
  libx11-xcb-dev libxcb1-dev libxcb-dri2-0-dev libxcb-icccm4-dev \
  libxcursor-dev libxkbcommon-dev libxcb-shape0-dev libxcb-xfixes0-dev
```

### 3. Testing During Development

```bash
# Run tests
cargo test --workspace --features "simd,standalone,zstd"

# Build example plugin
cargo xtask bundle track_info_display --release

# Check formatting
cargo fmt -- --check

# Run CI checks locally
cargo test --locked --workspace --features "simd,standalone,zstd"
cargo build --no-default-features
```

### 4. Use Fork in Parlando

Update Parlando's `Cargo.toml`:

```toml
[dependencies]
nih_plug = { git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support" }
```

### 5. Submit Pull Request

1. Ensure all tests pass
2. Add documentation
3. Create example plugin demonstrating feature
4. Submit PR with clear description of changes
5. Reference VST3 and CLAP specifications in PR

## Key Implementation Considerations

### Thread Safety
- Track info updates arrive on main/UI thread
- Must be safely accessible from audio thread
- Use `AtomicRefCell` for storage, not `Mutex` (avoid blocking audio thread)

### Backwards Compatibility
- Default implementation in trait returns `None`
- Existing plugins continue to work without changes
- Feature is opt-in via `track_info()` method

### Error Handling
- Never panic in wrapper code
- Use `nih_debug_assert_failure!` for unexpected conditions
- Gracefully handle missing host extensions

### Performance
- Cache track info, don't query host repeatedly
- Only update when host sends change notification
- Minimize allocations in audio thread

## Related NIH-plug Patterns

### Similar Extension: Polyphonic Modulation
- CLAP: `voice_info` extension (`wrapper.rs:238`)
- Only registered if `P::CLAP_POLY_MODULATION_CONFIG.is_some()`
- Good example of conditional extension support

### Similar Data: Transport
- Already exposed via `ProcessContext::transport()`
- Shows pattern for read-only host data
- Located in `/tmp/nih-plug/src/context/process.rs`

### Task System
- Use `Task` enum for cross-thread notifications
- Example: `Task::ParameterValueChanged`
- Add `Task::TrackInfoChanged` for this feature

## Historical Context

From the handoff document (`thoughts/shared/handoffs/general/2025-10-24_12-42-32_track-name-research-completion.md`):
- Initial research found NIH-plug lacks track name support
- User strongly desires auto track name sync
- Hybrid fork strategy recommended for contributing back to upstream
- Effort estimate: 1-2 weeks full-time, 1 month part-time

## Open Questions

1. **Should track info be available in InitContext too?**
   - Some hosts provide it during initialization
   - Would require storing before audio processing starts

2. **Should we add a dedicated TrackContext trait?**
   - Cleaner separation of concerns
   - But adds complexity to context system

3. **How to handle Standalone wrapper?**
   - Currently would return None
   - Could add user-configurable track name

## Conclusion

Implementing track information support in NIH-plug requires:
1. Adding a `TrackInfo` struct and context method
2. Implementing VST3 `IInfoListener` interface
3. Implementing CLAP `track-info` extension
4. Following NIH-plug's patterns for thread-safe shared state
5. Testing with example plugins across multiple DAWs

The implementation is straightforward and follows existing patterns in the codebase. The main complexity lies in proper thread synchronization and ensuring backwards compatibility. The feature would benefit many NIH-plug users and aligns with the framework's goal of providing comprehensive plugin API support.