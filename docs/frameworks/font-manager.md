# FontManager

FontManager is a singleton framework for loading, caching, and composing LVGL fonts — including built-in bitmap fonts, TrueType fonts, and emoji image fonts. It automatically selects the best-quality pre-rendered emoji asset for the requested size and uses LVGL's imgfont fallback mechanism to render emoji inline with text.

## Overview

FontManager centralizes all font concerns in a single class:

- **Unified API** - One call (`getFont`) to get any font, with or without emoji support
- **Emoji Compositing** - Transparently layers an emoji imgfont on top of any base font via LVGL's `fallback` mechanism
- **Size-tiered Assets** - Picks the closest pre-rendered emoji PNG directory so LVGL never has to scale further than necessary
- **Lazy Caching** - Fonts and scaled image descriptors are cached on first use; no redundant work on subsequent calls
- **Android-Inspired** - Follows the same singleton/class-method pattern as other MicroPythonOS frameworks

## Quick Start

```python
from mpos import FontManager

# Get a built-in Montserrat font at 16px, with emoji support (default)
font = FontManager.getFont(size=16, family="Montserrat")

# Get the same font without emoji (e.g. for glyph enumeration)
base_font = FontManager.getFont(size=16, family="Montserrat", emoji=False)

# Load a TrueType font from a file
ttf_font = FontManager.getFont(size=42, ttf="M:apps/com.myapp/assets/MyFont.ttf")

# List all available built-in fonts (each already has emoji composed in)
for info in FontManager.listFonts():
    print(info["name"], info["size"])

# Get all available emoji codepoints (union across all size tiers)
for cp in FontManager.getEmojiCodepoints():
    print(hex(cp))
```

## Architecture

FontManager is implemented as a singleton using class variables and class methods. No instance creation is needed.

### Font composition

When `emoji=True` (the default), `getFont()` wraps the requested base font in an LVGL imgfont that renders emoji as scaled PNG images. The imgfont's `fallback` is set to the base font, so LVGL automatically falls through to the base font for any codepoint that is not an emoji.

```
getFont(size=16)
  └── base_font = font_montserrat_16          (builtin bitmap)
  └── emoji_font = lv.imgfont_create(...)     (PNG-based imgfont)
        └── emoji_font.fallback = base_font
  └── returns emoji_font
```

### Emoji size tiers

Emoji PNGs are stored in size-specific directories under `builtin/res/emojis/`:

| Directory | Max target height | Used for |
|-----------|------------------|----------|
| `20x20/`  | ≤ 20 px          | Montserrat 8–16 (line heights ≤ 20 px) |
| `56x56/`  | any              | Larger fonts |

`_imgfont_path_cb` receives the rendering font's pixel height and picks the smallest tier whose `max_height` is ≥ the target. This minimises (or eliminates) the nearest-neighbour downscale performed by LVGL's software renderer.

### Caching layers

| Cache | Key | Value |
|-------|-----|-------|
| `_composed_font_cache` | `(font_id, emoji_size)` | Composed imgfont object |
| `_ttf_font_cache` | `(path, size)` | `lv.tiny_ttf_create_file` result |
| `_emoji_maps` | `dir_name` | `{codepoint: src_path}` dict |
| `_imgfont_scaled_src_cache` | `(src, target_height)` | Scaled `lv.image_dsc_t` or original src |
| `_imgfont_source_size_cache` | `src` | `(width, height)` tuple |
| `_imgfont_empty_src_cache` | `target_height` | 1×h transparent `lv.image_dsc_t` |

## API Reference

### `getFont(size=None, ttf=None, family=None, emoji=True)`

Return a font object suitable for use with `set_style_text_font()`.

**Parameters:**

- `size` (int, optional): Target pixel size. Snapped to the nearest available builtin size. Defaults to 12.
- `ttf` (str, optional): Path to a `.ttf` file (e.g. `"M:apps/myapp/assets/MyFont.ttf"`). When provided, `family` is ignored.
- `family` (str, optional): Font family name — `"Montserrat"` (default) or `"Unscii"`.
- `emoji` (bool): If `True` (default), the returned font transparently renders emoji via a PNG imgfont fallback. Pass `False` to get the raw base font (useful for `get_glyph_dsc` enumeration).

**Returns:** `lv.font_t` — the requested font, possibly wrapped with emoji support.

**Example:**

```python
from mpos import FontManager
import lvgl as lv

font = FontManager.getFont(size=24, family="Montserrat")
label = lv.label(screen)
label.set_style_text_font(font, lv.PART.MAIN)
label.set_text("Hello ❤️ 😀")
```

---

### `listFonts()`

Return a list of all available built-in fonts, each already composed with emoji support.

**Returns:** list of dicts, each with keys:

- `"name"` (str): Human-readable name, e.g. `"Montserrat 16"`
- `"family"` (str): Family name, e.g. `"Montserrat"`
- `"size"` (int): Nominal point size
- `"font"` (lv.font_t): Composed font (with emoji)
- `"base_font"` (lv.font_t): Raw base font (without emoji)

**Example:**

```python
from mpos import FontManager
import lvgl as lv

for info in FontManager.listFonts():
    label = lv.label(screen)
    label.set_style_text_font(info["font"], lv.PART.MAIN)
    label.set_text(info["name"] + ": ABC 😀 ❤️")
```

---

### `getEmojiCodepoints()`

Return a sorted list of all emoji codepoints available across all size tiers.

**Returns:** list of int

**Example:**

```python
from mpos import FontManager

for cp in FontManager.getEmojiCodepoints():
    print(hex(cp), chr(cp))
```

---

### `normalizeEmojiText(text)`

Strip Unicode variation selectors (U+FE0E text selector, U+FE0F emoji selector) from a string. Useful before storing or comparing text that may have been pasted from a source that appends these codepoints.

**Parameters:**

- `text` (str): Input string

**Returns:** str — cleaned string

**Example:**

```python
from mpos import FontManager

clean = FontManager.normalizeEmojiText("❤️")  # removes U+FE0F after ❤
```

## Emoji Assets

Emoji PNGs are stored in `internal_filesystem/builtin/res/emojis/` and are included in the firmware image at `/builtin/res/emojis/` on the device filesystem. Each subdirectory is a size tier:

```
builtin/res/emojis/
├── 20x20/       # Pre-rendered at 20×20 px (Lanczos-downscaled from 56×56)
│   ├── 1F600.png
│   ├── 263A.png
│   └── ...
└── 56x56/       # Cropped from original 72×72 OpenMoji PNGs
    ├── 1F600.png
    ├── 263A.png
    └── ...
```

Files are named by their Unicode codepoint in uppercase hex (e.g. `1F600.png` for 😀). FontManager scans each directory at runtime and builds a `{codepoint: path}` map.

### Adding new emoji

1. Add a PNG named `<CODEPOINT_HEX>.png` to both `56x56/` and `20x20/` (generate the 20×20 version with ImageMagick):

```bash
convert original.png -gravity center -crop 56x56+0+0 +repage -filter Lanczos -resize 20x20 20x20/CODEPOINT.png
```

2. The new emoji will be picked up automatically on next boot — no code changes needed.

## File Structure

```
internal_filesystem/
├── lib/mpos/ui/
│   └── font_manager.py      # FontManager class
└── builtin/res/emojis/
    ├── 20x20/               # 20×20 px emoji PNGs
    └── 56x56/               # 56×56 px emoji PNGs
```

## Related Frameworks

- **[AppearanceManager](appearance-manager.md)** - Theme colors and light/dark mode
- **[DisplayMetrics](display-metrics.md)** - Display width, height, DPI

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Creating Apps](../apps/creating-apps.md)
