# FontManager

FontManager is a singleton framework for loading, caching, and composing LVGL fonts — including built-in bitmap fonts, TrueType fonts, and emoji image fonts. It uses LVGL's imgfont fallback mechanism to render 32×32 emoji PNGs inline with text, with nearest-neighbour scaling to match any font size.

## Overview

FontManager centralizes all font concerns in a single class:

- **Unified API** - One call (`getFont`) to get any font, with or without emoji support
- **Emoji Compositing** - Transparently layers 32×32 emoji PNGs via LVGL's imgfont fallback, with nearest-neighbour scaling to match any font size
- **Lazy Caching** - Fonts and scaled image descriptors are cached on first use; no redundant work on subsequent calls
- **Android-Inspired** - Follows the same singleton/class-method pattern as other MicroPythonOS frameworks

## Quick Start

```python
from mpos import FontManager

# Get a built-in Montserrat font at 16px (emoji disabled by default)
font = FontManager.getFont(size=16, family="Montserrat")

# Get the same font with emoji support
emoji_font = FontManager.getFont(size=16, family="Montserrat", emoji=True)

# Load a TrueType font from a file
ttf_font = FontManager.getFont(size=42, ttf="M:apps/com.myapp/assets/MyFont.ttf")

# List all available built-in fonts (pass emojis=True for composed fonts)
for info in FontManager.listFonts():
    print(info["name"], info["size"])

# Get all available emoji codepoints (base codepoints only)
for cp in FontManager.getEmojiCodepoints():
    print(hex(cp))

# Get all available emoji strings (full sequences, including flag pairs)
for s in FontManager.getEmojiStrings():
    print(s)
```

## Architecture

FontManager is implemented as a singleton using class variables and class methods. No instance creation is needed.

### Font composition

When `emoji=True`, `getFont()` wraps the requested base font in an LVGL imgfont that renders emoji as scaled PNG images. The imgfont's `fallback` is set to the base font, so LVGL automatically falls through to the base font for any codepoint that is not an emoji.

```
getFont(size=16, emoji=True)
  └── base_font = font_montserrat_16          (builtin bitmap)
  └── emoji_font = lv.imgfont_create(...)     (PNG-based imgfont)
        └── emoji_font.fallback = base_font
  └── returns emoji_font
```

### Emoji size tiers

Emoji PNGs are stored in `builtin/res/emojis/`:

| Directory | Used for |
|-----------|----------|
| `32x32/`  | All fonts — emoji are rendered at 32×32 px and nearest-neighbour scaled up or down by LVGL as needed |

`_imgfont_path_cb` receives the rendering font's pixel height and returns the 32×32 emoji source. LVGL's software renderer performs nearest-neighbour scaling to fit the target font size.

### Caching layers

| Cache | Key | Value |
|-------|-----|-------|
| `_composed_font_cache` | `(font_id, emoji_size)` | Composed imgfont object |
| `_ttf_font_cache` | `(path, size)` | `lv.tiny_ttf_create_file` result |
| `_emoji_map` | (none) | `{codepoint: src_path}` dict |
| `_emoji_strings` | (none) | Sorted list of complete emoji strings |
| `_imgfont_scaled_src_cache` | `(src, target_height)` | Scaled `lv.image_dsc_t` or original src |
| `_imgfont_source_size_cache` | `src` | `(width, height)` tuple |
| `_imgfont_empty_src_cache` | `target_height` | 1×h transparent `lv.image_dsc_t` |

## API Reference

### `getFont(size=None, ttf=None, family=None, emoji=False)`

Return a font object suitable for use with `set_style_text_font()`.

**Parameters:**

- `size` (int, optional): Target pixel size. Snapped to the nearest available builtin size. Defaults to 12.
- `ttf` (str, optional): Path to a `.ttf` file (e.g. `"M:apps/myapp/assets/MyFont.ttf"`). When provided, `family` is ignored.
- `family` (str, optional): Font family name — `"Montserrat"` (default) or `"Unscii"`.
- `emoji` (bool, optional): If `True`, the returned font transparently renders emoji via a PNG imgfont fallback. Defaults to `False`. Pass `True` to enable inline emoji rendering (adds a small rendering cost per emoji glyph).

**Returns:** `lv.font_t` — the requested font, possibly wrapped with emoji support.

**Example:**

```python
from mpos import FontManager
import lvgl as lv

font = FontManager.getFont(size=24, family="Montserrat", emoji=True)
label = lv.label(screen)
label.set_style_text_font(font, lv.PART.MAIN)
label.set_text("Hello ❤️ 😀")
```

---

### `listFonts(emojis=False)`

Return a list of all available built-in fonts. By default returns raw base fonts; pass `emojis=True` to wrap each font with emoji composition.

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

for info in FontManager.listFonts(emojis=True):
    label = lv.label(screen)
    label.set_style_text_font(info["font"], lv.PART.MAIN)
    label.set_text(info["name"] + ": ABC 😀 ❤️")
```

---

### `getEmojiCodepoints()`

Return a sorted list of the base emoji codepoints available in the emoji map.

For multi-codepoint emoji such as flag sequences (e.g. `"🇸🇻"`), only the first codepoint is returned. Use `getEmojiStrings()` when you need complete, renderable emoji sequences.

**Returns:** list of int

**Example:**

```python
from mpos import FontManager

for cp in FontManager.getEmojiCodepoints():
    print(hex(cp), chr(cp))
```

---

### `getEmojiStrings()`

Return a sorted list of all available, complete emoji strings.

Unlike `getEmojiCodepoints()`, this returns full sequences: flag emoji include both regional indicators, and emoji with variation selectors keep their trailing selector. This is the preferred API for building a visual list of every supported emoji.

**Returns:** list of str

**Example:**

```python
from mpos import FontManager

for s in FontManager.getEmojiStrings():
    print(s)
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

To keep firmware image size small, the OS bundles ~50 of the most frequently-used emoji codepoints. Individual apps can bundle additional emoji PNGs in their own assets directory. Emoji PNGs are stored in `internal_filesystem/builtin/res/emojis/` and are included in the firmware image at `/builtin/res/emojis/` on the device filesystem:

```
builtin/res/emojis/
└── 32x32/       # Pre-rendered at 32×32 px
    ├── 1F600.png
    ├── 1F3CE-FE0F.png
    ├── 1F1F8-1F1FB.png
    └── ...
```

Files are named by their Unicode codepoint(s) in uppercase hex, with multiple codepoints joined by `-` (e.g. `1F600.png` for 😀, `1F3CE-FE0F.png` for 🏎️, `1F1F8-1F1FB.png` for 🇸🇻). FontManager scans the directory at runtime and builds a `{codepoint: path}` map.

### Adding new emoji

1. Add a PNG named `<CODEPOINT_HEX>.png` to `32x32/`:

```bash
cp original.png 32x32/CODEPOINT.png
```

2. The new emoji will be picked up automatically on next boot — no code changes needed.

### Emoji reference

For a canonical frequency-ordered listing of all available emoji codepoints, see [emoji_frequency_canonical.html](emoji_frequency_canonical.html).

## File Structure

```
internal_filesystem/
├── lib/mpos/ui/
│   └── font_manager.py      # FontManager class
└── builtin/res/emojis/
    └── 32x32/               # 32×32 px emoji PNGs
```

## Related Frameworks

- **[AppearanceManager](appearance-manager.md)** - Theme colors and light/dark mode
- **[DisplayMetrics](display-metrics.md)** - Display width, height, DPI

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Creating Apps](../apps/creating-apps.md)
