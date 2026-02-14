# Fontconfig Strategy

This document describes fontconfig control options and explores future improvements.

## Feature Flags

Three feature flags control font loading behavior:

| Feature | Behavior |
|---------|----------|
| `fontconfig` (default) | System fonts loaded automatically at startup |
| `fontconfig-explicit` | App must call `iced::font::configure()` before text rendering |
| Neither | No system fonts - bundled + explicitly loaded fonts only |

## Using `fontconfig-explicit`

When enabled, the application **must** configure the font system before any text rendering:

```rust
use iced::font::{configure, FontConfig};

fn main() -> iced::Result {
    // Configure BEFORE calling .run() or any text rendering
    configure(FontConfig {
        load_system_fonts: false,  // Fast startup, no system fonts
    });
    
    iced::application("MyApp", update, view).run()
}
```

### `FontConfig` Fields

| Field | Type | Description |
|-------|------|-------------|
| `load_system_fonts` | `bool` | Whether to scan system fonts via fontconfig |

## Performance Impact

On systems with many fonts (e.g., 7GB+ of font files), disabling system font loading can reduce startup time from seconds to milliseconds in debug builds.

## Future Enhancements

### 1. Lazy Font Loading

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

### 2. Custom Font Paths

Allow specifying custom font directories:

```rust
pub struct FontConfig {
    pub load_system_fonts: bool,
    pub font_paths: Vec<PathBuf>,  // Additional paths to scan
}
```

### 3. Font Preloading API

Load fonts on a background thread:

```rust
fn main() -> iced::Result {
    let loader = iced::font::Loader::new();
    loader.preload_system_fonts();  // Spawns background task
    
    my_app().run(iced::Settings::default())
}
```

### 4. Font Database Caching

Cache fontdb scan results to disk:

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

## Implementation Notes

### Related Code Locations

- `graphics/src/text.rs` - FontSystem initialization and configuration
- `wgpu/src/image/vector.rs` - SVG font loading (uses usvg's fontdb)
- `tiny_skia/src/vector.rs` - SVG font loading (uses usvg's fontdb)

### cosmic-text Feature Flags

cosmic-text 0.16 features relevant to fontconfig:
- `fontconfig` - enables `fontdb/fontconfig` for system font discovery
- `std` - required for most functionality
- `swash` - text rendering

Current configuration in workspace:
```toml
cosmic-text = { version = "0.16", default-features = false, features = ["std", "swash"] }
```

The `fontconfig` and `fontconfig-explicit` features enable the `fontconfig` feature on cosmic-text.
