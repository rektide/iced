# Fontconfig Strategy

This document explores future options for giving applications control over fontconfig/system font loading behavior.

## Current State

The `fontconfig` feature flag controls whether system fonts are loaded via fontconfig on Linux:
- **Enabled (default)**: All system fonts in `/usr/share/fonts` and other fontconfig paths are scanned at startup
- **Disabled**: Only bundled fonts (Iced-Icons, optionally Fira Sans) and explicitly loaded fonts are available

The slowdown issue is tracked in [issue #2455](https://github.com/iced-rs/iced/issues/2455).

## Future Options

### 1. Application-Controlled Font System Initialization

Allow applications to provide their own `FontSystem` configuration:

```rust
// In iced_core or iced_graphics
pub struct FontConfig {
    /// Whether to load system fonts via fontconfig
    pub load_system_fonts: bool,
    /// Custom font paths to scan (beyond fontconfig)
    pub font_paths: Vec<PathBuf>,
    /// Embedded fonts to include
    pub embedded_fonts: Vec<&'static [u8]>,
}

impl Default for FontConfig {
    fn default() -> Self {
        Self {
            load_system_fonts: cfg!(feature = "fontconfig"),
            font_paths: vec![],
            embedded_fonts: vec![],
        }
    }
}

// Application can configure before first text render
iced::Settings {
    fonts: FontConfig {
        load_system_fonts: false,
        embedded_fonts: vec![MY_FONT_BYTES],
        ..Default::default()
    },
    ..Default::default()
}
```

### 2. Lazy Font Loading

Defer font scanning until a font is actually needed:

```rust
// Don't scan fonts at FontSystem creation
// Instead, lazily query fontconfig when a specific font family is requested
// This requires changes to cosmic-text's fontdb
```

Pros:
- Zero startup cost for simple apps using default/bundled fonts
- Full fontconfig support available when needed

Cons:
- First text render with a system font has latency
- Requires upstream changes to cosmic-text/fontdb

### 3. Font Preloading API

Let applications pre-load fonts on a background thread:

```rust
// In application init, optionally spawn background font loading
fn main() -> iced::Result {
    let font_loader = iced::font::Loader::new();
    
    // Preload fonts in background
    font_loader.preload_system_fonts();
    
    // Continue with UI init - fonts will be ready when needed
    my_app().run(iced::Settings::default())
}
```

### 4. Caching Font Database

Cache the fontdb scan results to disk:

```rust
// On first run, scan and cache fontdb to XDG cache dir
// On subsequent runs, load from cache if still valid
// Cache invalidated by mtime check on font directories
```

Pros:
- Fast startup after first run
- Transparent to applications

Cons:
- Cache invalidation complexity
- Stale cache issues
- Cross-platform cache location handling

### 5. Selective Fontconfig Directories

Allow specifying which fontconfig directories to scan:

```rust
pub enum FontSource {
    /// Use system fontconfig (all directories)
    System,
    /// Only scan specific directories
    Paths(Vec<PathBuf>),
    /// Only use embedded fonts
    Embedded,
}

iced::Settings {
    font_source: FontSource::Paths(vec![
        "/usr/share/fonts/truetype/dejavu".into(),
    ]),
    ..Default::default()
}
```

### 6. Fontconfig Query-Only Mode

Use fontconfig to query font paths but don't pre-scan:

```rust
// When rendering text, query fontconfig for the specific font
// Load only that font file, not all system fonts
```

This would require fontdb to support on-demand loading rather than batch scanning.

## Recommended Path Forward

1. **Short-term** (current implementation): Feature flag to disable fontconfig entirely
2. **Medium-term**: Option 1 - Application-controlled FontConfig in Settings
3. **Long-term**: Option 2 - Lazy loading with upstream cosmic-text changes

The medium-term solution provides the best balance of:
- Application control
- No upstream changes required
- Sensible defaults for common cases
- Escape hatch for performance-sensitive applications

## Implementation Notes

### Related Code Locations

- `graphics/src/text.rs:115-133` - FontSystem initialization
- `wgpu/src/image/vector.rs:54-60` - SVG font loading (uses usvg's fontdb)
- `tiny_skia/src/vector.rs:90-96` - SVG font loading (uses usvg's fontdb)

### cosmic-text Feature Flags

cosmic-text 0.16 features relevant to fontconfig:
- `fontconfig` - enables `fontdb/fontconfig` for system font discovery
- `std` - required for most functionality
- `swash` - text rendering

Current configuration in workspace:
```toml
cosmic-text = { version = "0.16", default-features = false, features = ["std", "swash"] }
```

With `fontconfig` feature enabling the additional `fontconfig` feature.
