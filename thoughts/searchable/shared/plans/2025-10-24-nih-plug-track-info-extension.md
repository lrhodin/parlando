# NIH-plug Track Information Extension Implementation Plan

## Overview

Extend NIH-plug to support track/channel information from both VST3 (IInfoListener) and CLAP (track-info extension) standards, enabling Parlando to automatically sync track names from DAWs for musician identification.

## Current State Analysis

**NIH-plug Status:**
- No track information API exposed (confirmed in research)
- Supports transport info via `ProcessContext::transport()`
- Has extension patterns for both VST3 (COM interfaces) and CLAP (vtables)
- Uses trait-based context abstraction for plugin-host communication

**Parlando Status:**
- Phase 1 complete: VST3 plugin with OSC→MIDI conversion
- Phase 1.5 blocked: Requires track name for auto musician naming
- Currently must use fallback "Musician Number" parameter approach

### Key Discoveries:
- VST3 uses `#[VST3(implements(...))]` macro for COM interfaces (`/tmp/nih-plug/src/wrapper/vst3/wrapper.rs:41-49`)
- CLAP stores extension vtables as wrapper fields (`/tmp/nih-plug/src/wrapper/clap/wrapper.rs:169-239`)
- Thread safety via `AtomicRefCell` prevents audio thread blocking
- Example plugins serve as integration tests (no mock contexts)

## Desired End State

After completion, NIH-plug will expose track information through a unified `TrackInfo` struct accessible via `ProcessContext::track_info()`, supporting both VST3 IInfoListener and CLAP track-info extension.

### Verification:
- VST3 plugins receive track name/color via IInfoListener callback
- CLAP plugins query track info via track-info extension
- Example plugin demonstrates both formats working
- Parlando uses fork to implement Phase 1.5 auto musician naming

## What We're NOT Doing

- NOT adding track info to InitContext (keep scope minimal)
- NOT creating a separate TrackContext trait (avoid complexity)
- NOT implementing Standalone wrapper support (return None)
- NOT waiting for upstream PR approval before using in Parlando
- NOT implementing track modification capabilities (read-only)

## Implementation Approach

Fork NIH-plug, implement track info support following existing patterns, create example plugin for testing, submit PR to upstream, and use fork in Parlando immediately.

## Phase 1: Fork Setup and Core Types

### Overview
Fork NIH-plug repository, create feature branch, and implement the core `TrackInfo` struct and context trait method.

### Changes Required:

#### 1. Fork and Branch Setup
**Actions**: GitHub fork and local setup
```bash
# Fork https://github.com/robbert-vdh/nih-plug to lrhodin/nih-plug on GitHub
git clone https://github.com/lrhodin/nih-plug.git
cd nih-plug
git remote add upstream https://github.com/robbert-vdh/nih-plug.git
git checkout -b feature/track-info-support
```

#### 2. Core TrackInfo Types
**File**: `src/context.rs`
**Changes**: Add TrackInfo struct and TrackType flags

```rust
// Add after line 10 (other imports)
/// Information about the track/channel this plugin is inserted on
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

impl Default for TrackInfo {
    fn default() -> Self {
        Self {
            name: None,
            color: None,
            index: None,
            uid: None,
            track_type: TrackType::default(),
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Default)]
pub struct TrackType {
    pub is_master: bool,
    pub is_bus: bool,
    pub is_return: bool,
}
```

#### 3. ProcessContext Trait Extension
**File**: `src/context/process.rs`
**Changes**: Add track_info method to ProcessContext trait (around line 100)

```rust
pub trait ProcessContext<P: Plugin> {
    // ... existing methods ...

    /// Returns current track information if available from the host.
    ///
    /// This is supported for VST3 (via IInfoListener) and CLAP (via track-info extension).
    /// The standalone wrapper always returns None.
    fn track_info(&self) -> Option<&TrackInfo> {
        None // Default implementation for backwards compatibility
    }
}
```

### Success Criteria:

#### Automated Verification:
- [x] Fork exists at github.com/lrhodin/nih-plug
- [x] Feature branch created: `git branch | grep track-info-support`
- [x] Code compiles: `cargo build --all-features`
- [x] No existing tests broken: `cargo test --workspace`
- [x] Formatting correct: `cargo fmt -- --check`

#### Manual Verification:
- [ ] TrackInfo struct properly exported from context module
- [ ] Default trait implementation doesn't break existing plugins

---

## Phase 2: VST3 IInfoListener Implementation

### Overview
Implement the VST3 IInfoListener interface to receive track information callbacks from VST3 hosts.

### Changes Required:

#### 1. Add IInfoListener to Wrapper Declaration
**File**: `src/wrapper/vst3/wrapper.rs`
**Changes**: Add IInfoListener to implements macro (line 41)

```rust
#[VST3(implements(
    IComponent,
    IEditController,
    IAudioProcessor,
    IMidiMapping,
    INoteExpressionController,
    IProcessContextRequirements,
    IUnitInfo,
    IInfoListener  // Add this line
))]
pub struct Wrapper<P: Vst3Plugin> {
    inner: Arc<WrapperInner<P>>,
}
```

#### 2. Add Track Info Storage to WrapperInner
**File**: `src/wrapper/vst3/inner.rs`
**Changes**: Add track_info field (around line 130)

```rust
pub struct WrapperInner<P: Vst3Plugin> {
    // ... existing fields around line 30-129 ...

    /// Current track information from host (VST3 IInfoListener)
    pub track_info: AtomicRefCell<Option<TrackInfo>>,
}

// In WrapperInner::new() around line 250
track_info: AtomicRefCell::new(None),
```

#### 3. Import Required Types
**File**: `src/wrapper/vst3/wrapper.rs`
**Changes**: Add imports at top of file

```rust
use crate::context::{TrackInfo, TrackType};
use vst3_sys::vst::{IInfoListener, IAttributeList, kResultOk};
use vst3_sys::base::{kInvalidArgument, tresult};
use std::ffi::CStr;
```

#### 4. Implement IInfoListener Interface
**File**: `src/wrapper/vst3/wrapper.rs`
**Changes**: Add implementation after other interface implementations (around line 1800)

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
        let name_key = CStr::from_bytes_with_nul_unchecked(
            b"Steinberg.Vst.ChannelContext.ChannelName\0"
        );
        if list.get_string(
            name_key.as_ptr(),
            name_u16.as_mut_ptr(),
            name_u16.len() as i32,
        ) == kResultOk {
            // Find null terminator
            let len = name_u16.iter().position(|&c| c == 0).unwrap_or(name_u16.len());
            track_info.name = Some(String::from_utf16_lossy(&name_u16[..len]));
        }

        // Extract track color (ARGB format)
        let mut color: i64 = 0;
        let color_key = CStr::from_bytes_with_nul_unchecked(
            b"Steinberg.Vst.ChannelContext.ChannelColor\0"
        );
        if list.get_int(color_key.as_ptr(), &mut color) == kResultOk {
            let color_u32 = color as u32;
            track_info.color = Some((
                ((color_u32 >> 16) & 0xFF) as u8, // R
                ((color_u32 >> 8) & 0xFF) as u8,  // G
                (color_u32 & 0xFF) as u8,         // B
                ((color_u32 >> 24) & 0xFF) as u8, // A
            ));
        }

        // Extract channel index
        let mut index: i64 = 0;
        let index_key = CStr::from_bytes_with_nul_unchecked(
            b"Steinberg.Vst.ChannelContext.ChannelIndex\0"
        );
        if list.get_int(index_key.as_ptr(), &mut index) == kResultOk {
            track_info.index = Some(index as i32);
        }

        // Extract channel UID
        let mut uid_u16 = [0u16; 128];
        let uid_key = CStr::from_bytes_with_nul_unchecked(
            b"Steinberg.Vst.ChannelContext.ChannelUID\0"
        );
        if list.get_string(
            uid_key.as_ptr(),
            uid_u16.as_mut_ptr(),
            uid_u16.len() as i32,
        ) == kResultOk {
            let len = uid_u16.iter().position(|&c| c == 0).unwrap_or(uid_u16.len());
            track_info.uid = Some(String::from_utf16_lossy(&uid_u16[..len]));
        }

        // Store the track info
        *self.inner.track_info.borrow_mut() = Some(track_info);

        kResultOk
    }
}
```

#### 5. Update VST3 ProcessContext Implementation
**File**: `src/wrapper/vst3/context.rs`
**Changes**: Implement track_info method (around line 400)

```rust
impl<P: Vst3Plugin> ProcessContext<P> for WrapperProcessContext<'_, P> {
    // ... existing methods ...

    fn track_info(&self) -> Option<&TrackInfo> {
        // SAFETY: We're returning a reference that's valid for the lifetime of the borrow.
        // The AtomicRefCell ensures thread-safe access.
        unsafe {
            let info_ref = self.inner.track_info.borrow();
            std::mem::transmute(info_ref.as_ref())
        }
    }
}
```

### Success Criteria:

#### Automated Verification:
- [x] VST3 wrapper compiles: `cargo build --features vst3`
- [x] No clippy warnings: `cargo clippy --features vst3`
- [x] VST3 tests pass: `cargo test --features vst3`

#### Manual Verification:
- [ ] IInfoListener properly registered in VST3 COM interface
- [ ] Track info storage thread-safe with AtomicRefCell
- [ ] No memory leaks or unsafe access patterns

---

## Phase 3: CLAP track-info Extension Implementation

### Overview
Implement the CLAP track-info extension to query and receive track information from CLAP hosts.

### Changes Required:

#### 1. Add CLAP Extension Dependencies
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Add imports and types (top of file)

```rust
// Add to imports section
use clap_sys::ext::track_info::{
    clap_plugin_track_info, clap_host_track_info, clap_track_info,
    CLAP_EXT_TRACK_INFO, CLAP_TRACK_INFO_HAS_TRACK_NAME,
    CLAP_TRACK_INFO_HAS_TRACK_COLOR, CLAP_TRACK_INFO_IS_FOR_MASTER,
    CLAP_TRACK_INFO_IS_FOR_BUS, CLAP_TRACK_INFO_IS_FOR_RETURN_TRACK,
};
use crate::context::{TrackInfo, TrackType};
```

#### 2. Add Extension Fields to Wrapper
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Add fields to Wrapper struct (around line 200)

```rust
pub struct Wrapper<P: ClapPlugin> {
    // ... existing fields ...

    /// Track info extension vtable
    clap_plugin_track_info: clap_plugin_track_info,

    /// Host track info extension (cached on init)
    host_track_info: AtomicRefCell<Option<ClapPtr<clap_host_track_info>>>,

    /// Current track information
    track_info: AtomicRefCell<Option<TrackInfo>>,
}
```

#### 3. Initialize Extension in Constructor
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Initialize fields in Wrapper::new() (around line 500)

```rust
// In Wrapper::new()
clap_plugin_track_info: clap_plugin_track_info {
    changed: Some(Self::ext_track_info_changed),
},
host_track_info: AtomicRefCell::new(None),
track_info: AtomicRefCell::new(None),
```

#### 4. Query Host Extension on Init
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Query host in init() function (around line 1850)

```rust
// In init() function, after other host extension queries
*wrapper.host_track_info.borrow_mut() = query_host_extension::<clap_host_track_info>(
    &wrapper.host_callback,
    CLAP_EXT_TRACK_INFO,
);

// Query initial track info if available
wrapper.update_track_info();
```

#### 5. Implement Extension Functions
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Add implementation methods (around line 3000)

```rust
impl<P: ClapPlugin> Wrapper<P> {
    /// Update track info by querying the host
    fn update_track_info(&self) {
        if let Some(host_track_info) = &*self.host_track_info.borrow() {
            let mut info = clap_track_info {
                flags: 0,
                name: [0; CLAP_NAME_SIZE],
                color: clap_color { red: 0, green: 0, blue: 0, alpha: 255 },
                audio_channel_count: 0,
                audio_port_type: std::ptr::null(),
            };

            // SAFETY: Calling host extension with valid pointers
            if unsafe {
                clap_call!(host_track_info=>get(&*self.host_callback, &mut info))
            } {
                let mut track_info = TrackInfo::default();

                // Extract track name
                if info.flags & CLAP_TRACK_INFO_HAS_TRACK_NAME != 0 {
                    // SAFETY: CLAP guarantees null-terminated string
                    track_info.name = Some(unsafe {
                        CStr::from_ptr(info.name.as_ptr())
                            .to_string_lossy()
                            .into_owned()
                    });
                }

                // Extract track color
                if info.flags & CLAP_TRACK_INFO_HAS_TRACK_COLOR != 0 {
                    track_info.color = Some((
                        info.color.red,
                        info.color.green,
                        info.color.blue,
                        info.color.alpha,
                    ));
                }

                // Extract track type
                track_info.track_type = TrackType {
                    is_master: info.flags & CLAP_TRACK_INFO_IS_FOR_MASTER != 0,
                    is_bus: info.flags & CLAP_TRACK_INFO_IS_FOR_BUS != 0,
                    is_return: info.flags & CLAP_TRACK_INFO_IS_FOR_RETURN_TRACK != 0,
                };

                *self.track_info.borrow_mut() = Some(track_info);
            }
        }
    }

    /// Called when track info changes
    unsafe extern "C" fn ext_track_info_changed(plugin: *const clap_plugin) {
        check_null_ptr!((), plugin, (*plugin).plugin_data);
        let wrapper = &*((*plugin).plugin_data as *const Self);

        wrapper.update_track_info();

        // Optionally notify GUI if needed
        // wrapper.schedule_gui(Task::TrackInfoChanged);
    }
}
```

#### 6. Register Extension in get_extension
**File**: `src/wrapper/clap/wrapper.rs`
**Changes**: Add to get_extension() (around line 2500)

```rust
// In get_extension() function
} else if id == CLAP_EXT_TRACK_INFO {
    &wrapper.clap_plugin_track_info as *const _ as *const c_void
```

#### 7. Update CLAP ProcessContext Implementation
**File**: `src/wrapper/clap/context.rs`
**Changes**: Implement track_info method (around line 300)

```rust
impl<P: ClapPlugin> ProcessContext<P> for WrapperProcessContext<'_, P> {
    // ... existing methods ...

    fn track_info(&self) -> Option<&TrackInfo> {
        // SAFETY: We're returning a reference that's valid for the lifetime of the borrow
        unsafe {
            let info_ref = self.wrapper.track_info.borrow();
            std::mem::transmute(info_ref.as_ref())
        }
    }
}
```

### Success Criteria:

#### Automated Verification:
- [x] CLAP wrapper compiles: `cargo build --features clap`
- [x] No clippy warnings: `cargo clippy --features clap`
- [x] CLAP tests pass: `cargo test --features clap`

#### Manual Verification:
- [x] track-info extension properly registered (Bitwig recognized extension)
- [x] Host extension queried on init (Initial track info received)
- [x] Changed callback properly handles updates (Name/color changes detected live)
- [x] Thread-safe access to track info (Guard pattern working correctly)

---

## Phase 4: Example Plugin and Testing

### Overview
Create an example plugin demonstrating track info usage and add unit tests for the new functionality.

### Changes Required:

#### 1. Create Example Plugin Directory
**Action**: Create new example plugin
```bash
mkdir -p plugins/examples/track_info_display/src
```

#### 2. Example Plugin Implementation
**File**: `plugins/examples/track_info_display/src/lib.rs`
**Changes**: Complete plugin implementation

```rust
use nih_plug::prelude::*;
use std::sync::Arc;

struct TrackInfoDisplay {
    params: Arc<TrackInfoParams>,
}

#[derive(Params)]
struct TrackInfoParams {
    #[id = "track_name_display"]
    track_name: StringParam,

    #[id = "track_color_display"]
    track_color: StringParam,
}

impl Default for TrackInfoDisplay {
    fn default() -> Self {
        Self {
            params: Arc::new(TrackInfoParams::default()),
        }
    }
}

impl Default for TrackInfoParams {
    fn default() -> Self {
        Self {
            track_name: StringParam::new(
                "Track Name",
                "No track info".to_string(),
            ).with_readonly(true),

            track_color: StringParam::new(
                "Track Color",
                "Unknown".to_string(),
            ).with_readonly(true),
        }
    }
}

impl Plugin for TrackInfoDisplay {
    const NAME: &'static str = "Track Info Display";
    const VENDOR: &'static str = "NIH-plug";
    const URL: &'static str = "https://github.com/robbert-vdh/nih-plug";
    const EMAIL: &'static str = "info@example.com";
    const VERSION: &'static str = env!("CARGO_PKG_VERSION");
    const AUDIO_IO_LAYOUTS: &'static [AudioIOLayout] = &[
        AudioIOLayout {
            main_input_channels: NonZeroU32::new(2),
            main_output_channels: NonZeroU32::new(2),
            ..AudioIOLayout::const_default()
        },
    ];
    const SAMPLE_ACCURATE_AUTOMATION: bool = true;
    type SysExMessage = ();
    type BackgroundTask = ();

    fn params(&self) -> Arc<dyn Params> {
        self.params.clone()
    }

    fn process(
        &mut self,
        buffer: &mut Buffer,
        _aux: &mut AuxiliaryBuffers,
        context: &mut impl ProcessContext<Self>,
    ) -> ProcessStatus {
        // Update track info display
        if let Some(track_info) = context.track_info() {
            // Update track name display
            if let Some(name) = &track_info.name {
                self.params.track_name.set_string(name.clone());
                nih_log!("Track name: {}", name);
            }

            // Update track color display
            if let Some((r, g, b, a)) = track_info.color {
                let color_str = format!("RGBA({},{},{},{})", r, g, b, a);
                self.params.track_color.set_string(color_str.clone());
                nih_log!("Track color: {}", color_str);
            }

            // Log track type
            if track_info.track_type.is_master {
                nih_log!("This is the master track");
            }
            if track_info.track_type.is_bus {
                nih_log!("This is a bus track");
            }
            if track_info.track_type.is_return {
                nih_log!("This is a return track");
            }
        }

        // Pass through audio
        ProcessStatus::Normal
    }
}

impl ClapPlugin for TrackInfoDisplay {
    const CLAP_ID: &'static str = "com.nih-plug.track-info-display";
    const CLAP_DESCRIPTION: Option<&'static str> =
        Some("Example plugin demonstrating track info API");
    const CLAP_MANUAL_URL: Option<&'static str> = None;
    const CLAP_SUPPORT_URL: Option<&'static str> = None;
    const CLAP_FEATURES: &'static [ClapFeature] = &[
        ClapFeature::AudioEffect,
        ClapFeature::Utility,
    ];
}

impl Vst3Plugin for TrackInfoDisplay {
    const VST3_CLASS_ID: [u8; 16] = *b"TrackInfoDisplay";
    const VST3_SUBCATEGORIES: &'static [Vst3SubCategory] = &[
        Vst3SubCategory::Tools,
    ];
}

nih_export_clap!(TrackInfoDisplay);
nih_export_vst3!(TrackInfoDisplay);
```

#### 3. Example Plugin Cargo.toml
**File**: `plugins/examples/track_info_display/Cargo.toml`
**Changes**: Create Cargo manifest

```toml
[package]
name = "track_info_display"
version = "0.1.0"
edition = "2021"
authors = ["NIH-plug Contributors"]
license = "ISC"

[lib]
crate-type = ["cdylib"]

[dependencies]
nih_plug = { path = "../../../" }
```

#### 4. Add Unit Tests
**File**: `src/context.rs`
**Changes**: Add test module (at end of file)

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
        assert!(!info.track_type.is_bus);
        assert!(!info.track_type.is_return);
    }

    #[test]
    fn track_info_equality() {
        let mut info1 = TrackInfo::default();
        let mut info2 = TrackInfo::default();

        info1.name = Some("Track 1".to_string());
        info2.name = Some("Track 1".to_string());

        info1.color = Some((255, 0, 0, 255));
        info2.color = Some((255, 0, 0, 255));

        assert_eq!(info1, info2);
    }

    #[test]
    fn track_type_flags() {
        let mut track_type = TrackType::default();
        assert!(!track_type.is_master);

        track_type.is_master = true;
        track_type.is_bus = true;

        assert!(track_type.is_master);
        assert!(track_type.is_bus);
        assert!(!track_type.is_return);
    }
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Example plugin builds: `cargo xtask bundle track_info_display --release`
- [ ] Unit tests pass: `cargo test track_info`
- [ ] All features compile: `cargo build --all-features --workspace`
- [ ] CI passes: `cargo test --locked --workspace --all-features`

#### Manual Verification:
- [ ] Example plugin loads in Logic Pro (VST3)
- [ ] Example plugin loads in Reaper (CLAP)
- [ ] Track names display correctly
- [ ] Track colors update when changed
- [ ] Master/bus/return flags work correctly

---

## Phase 5: Documentation and PR Submission

### Overview
Add documentation, ensure all tests pass, and submit pull request to upstream NIH-plug.

### Changes Required:

#### 1. Add Documentation to lib.rs
**File**: `src/lib.rs`
**Changes**: Add documentation section (around line 300)

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

#### 2. Update CHANGELOG.md
**File**: `CHANGELOG.md`
**Changes**: Add entry at top

```markdown
## [Unreleased]

### Added
- Track information support via `ProcessContext::track_info()` for both VST3 (IInfoListener) and CLAP (track-info extension). Plugins can now access track name, color, index, and type information from the host.
```

#### 3. Run Final Tests
```bash
# Format code
cargo fmt

# Run clippy
cargo clippy --all-features --workspace

# Run all tests
cargo test --all-features --workspace

# Build without VST3 (GPL check)
cargo build --no-default-features

# Build example plugin
cargo xtask bundle track_info_display --release
```

#### 4. Submit Pull Request
**Action**: Create PR on GitHub
- Title: "Add track information support (VST3 IInfoListener & CLAP track-info)"
- Description: Include:
  - Motivation (plugins need track context)
  - Implementation overview
  - Links to VST3/CLAP specifications
  - Example plugin demonstration
  - Testing performed

### Success Criteria:

#### Automated Verification:
- [ ] All CI checks pass on GitHub
- [ ] No merge conflicts with upstream
- [ ] Documentation builds correctly

#### Manual Verification:
- [ ] PR description clear and complete
- [ ] Example plugin demonstrates feature
- [ ] Code follows NIH-plug patterns
- [ ] No breaking changes to existing plugins

---

## Phase 6: Parlando Integration

### Overview
Use the forked NIH-plug in Parlando to implement Phase 1.5 auto musician naming.

### Changes Required:

#### 1. Update Parlando's Cargo.toml
**File**: `Cargo.toml`
**Changes**: Point to fork

```toml
[dependencies]
# Use our fork with track info support
nih_plug = { git = "https://github.com/lrhodin/nih-plug.git", branch = "feature/track-info-support", features = ["assert_process_allocs"] }
```

#### 2. Update Parlando Plugin
**File**: `src/lib.rs`
**Changes**: Add track info handling in process()

```rust
fn process(
    &mut self,
    buffer: &mut Buffer,
    aux: &mut AuxiliaryBuffers,
    context: &mut impl ProcessContext<Self>,
) -> ProcessStatus {
    // Check for track name updates
    if let Some(track_info) = context.track_info() {
        if let Some(name) = &track_info.name {
            // Update musician name from track name
            let current_name = self.musician_name.borrow().clone();
            if current_name != *name {
                *self.musician_name.borrow_mut() = name.clone();

                // Send name update to app
                if self.acknowledged.load(Ordering::SeqCst) {
                    self.send_name_update(name.clone());
                }

                nih_log!("Musician name updated from track: {}", name);
            }
        }
    } else {
        // Fallback to musician number parameter
        let number = self.params.musician_number.value();
        let fallback_name = format!("Musician {}", number);
        *self.musician_name.borrow_mut() = fallback_name;
    }

    // ... rest of processing ...
}
```

#### 3. Test Integration
```bash
# Build Parlando plugin
cargo xtask bundle parlando --release

# Test in DAW
# 1. Load in Logic Pro
# 2. Rename track
# 3. Verify musician name updates
# 4. Check OSC messages
```

### Success Criteria:

#### Automated Verification:
- [ ] Parlando builds with fork: `cargo build --release`
- [ ] Tests pass: `cargo test`
- [ ] Bundle creates successfully: `cargo xtask bundle parlando`

#### Manual Verification:
- [ ] Track names sync automatically in Logic Pro
- [ ] Track names sync automatically in Ableton Live
- [ ] Fallback to musician number when no track info
- [ ] OSC messages contain correct musician names

---

## Testing Strategy

### Unit Tests:
- TrackInfo struct creation and equality
- TrackType flags behavior
- Default implementations return None

### Integration Tests:
- Example plugin receives track info in VST3 hosts
- Example plugin receives track info in CLAP hosts
- Thread safety under concurrent access
- Memory safety with host callbacks

### Manual Testing Steps:
1. Load example plugin in Logic Pro (VST3)
2. Rename track → verify name updates
3. Change track color → verify color updates
4. Load example plugin in Reaper (CLAP)
5. Test master/bus/return track types
6. Test Parlando with auto naming in multiple DAWs

## Performance Considerations

- Track info stored in `AtomicRefCell` to avoid blocking audio thread
- Updates happen on main/UI thread only
- Cache track info to avoid repeated host queries
- No allocations in audio thread (info pre-allocated)

## Migration Notes

- This change is backwards compatible
- Existing plugins continue to work without modification
- Default trait implementation returns None
- Opt-in via `ProcessContext::track_info()` method

## References

- Original handoff: `thoughts/shared/handoffs/general/2025-10-24_13-43-23_nih-plug-track-info-research.md`
- Research document: `thoughts/shared/research/2025-10-24-nih-plug-track-info-extension.md`
- Phase 1.5 spec: `Parlando Plugin Phase 1.5 Update Guide.md`
- VST3 IInfoListener: https://steinbergmedia.github.io/vst3_doc/vstinterfaces/classSteinberg_1_1Vst_1_1ChannelContext_1_1IInfoListener.html
- CLAP track-info: https://github.com/free-audio/clap/blob/main/include/clap/ext/track-info.h