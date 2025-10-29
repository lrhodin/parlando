# Parlando

**AI-powered musical conductor for virtual instruments**

Parlando enables AI agents to conduct and control virtual instruments in real-time through MIDI, creating expressive musical performances with dynamic phrasing, articulation, and emotional nuance.

## Repository Structure

This is a monorepo containing multiple related projects:

```
Parlando/
├── ParlandoPlugin/      # VST3/CLAP audio plugin (submodule)
├── ParlandoApp/         # Standalone desktop application (submodule, WIP)
├── nih-plug/            # NIH-plug framework fork with track info support (submodule)
├── vst3-sys/            # VST3 bindings fork (submodule)
├── thoughts/            # Specs, plans, research, and handoffs
└── docs/                # Documentation
```

### Submodules

- **[ParlandoPlugin](https://github.com/lrhodin/parlando-plugin)** - The core VST3/CLAP plugin
- **[ParlandoApp](https://github.com/lrhodin/parlando-app)** - Future standalone application
- **[nih-plug](https://github.com/lrhodin/nih-plug)** (branch: `feature/track-info-support`) - Fork with VST3/CLAP track information support
- **[vst3-sys](https://github.com/lrhodin/vst3-sys)** - VST3 bindings fork

## Getting Started

### Clone with Submodules

```bash
git clone --recursive https://github.com/lrhodin/parlando.git
cd parlando
```

### Or Initialize Submodules After Cloning

```bash
git clone https://github.com/lrhodin/parlando.git
cd parlando
git submodule update --init --recursive
```

### Building the Plugin

```bash
cd ParlandoPlugin
cargo xtask bundle parlando --release
```

The compiled plugins will be in `ParlandoPlugin/target/bundled/`:
- `parlando.vst3` - VST3 format
- `parlando.clap` - CLAP format

## Development

### Working on Submodules

Each submodule is a full git repository. To make changes:

```bash
cd ParlandoPlugin  # or nih-plug, vst3-sys, etc.
git checkout -b my-feature
# Make changes...
git commit -m "My changes"
git push origin my-feature
```

Then update the parent repo to track the new commit:

```bash
cd ..
git add ParlandoPlugin
git commit -m "Update ParlandoPlugin to latest"
```

### Using Local Dependencies

During development, ParlandoPlugin uses path dependencies to the local nih-plug fork:

```toml
# ParlandoPlugin/Cargo.toml
nih_plug = { path = "../nih-plug", features = ["assert_process_allocs"] }
```

## Documentation

- **[thoughts/shared/plans/](thoughts/shared/plans/)** - Implementation plans and specifications
- **[thoughts/shared/research/](thoughts/shared/research/)** - Research documents
- **[thoughts/shared/handoffs/](thoughts/shared/handoffs/)** - Progress handoffs and status updates

### Key Documents

- **[NIH-plug Track Info Extension](thoughts/shared/plans/2025-10-24-nih-plug-track-info-extension.md)** - Complete implementation plan for VST3/CLAP track information support
- **[Latest Handoff](thoughts/shared/handoffs/general/)** - Most recent progress and status

## Project Status

### ParlandoPlugin ✅
- **Phase 1**: Core VST3 plugin with OSC→MIDI conversion - Complete
- **Phase 1.5**: Track info integration - In Progress
- **Phase 2**: Multi-instance support (10K simultaneous instances) - Complete

### NIH-plug Track Info Support 🚧
- VST3 IInfoListener implementation - Complete
- CLAP track-info extension - Complete
- Threading fixes (3 bugs resolved) - Complete
- Testing in multiple DAWs - In Progress
- Upstream PR submission - Pending

## License

GPL-3.0-or-later

## Contact

Ludvig Rhodin - [rhodin.ludvig@gmail.com](mailto:rhodin.ludvig@gmail.com)

GitHub: [@lrhodin](https://github.com/lrhodin)
