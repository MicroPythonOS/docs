# Supported File Formats

MicroPythonOS uses LVGL's built-in image decoders and its own audio stack. The formats below are supported out of the box for apps such as the Image Viewer and Music Player.

## Image formats

| Format | Extensions | Notes |
|--------|------------|-------|
| PNG | `.png` | Fully supported via LodePNG. |
| Baseline JPEG | `.jpg`, `.jpeg` | Fully supported via TJpgD. |
| Progressive JPEG | `.jpg`, `.jpeg` | **Not supported.** Files will decode as `0x0` and appear blank. |
| RAW | `.raw` | Supported by the Image Viewer app if named as `<name>_<w>x<h>_RGB565.raw` or `<name>_<w>x<h>_GRAY.raw`. |

## Audio formats

| Format | Extensions | Notes |
|--------|------------|-------|
| Plain PCM WAV | `.wav` | Standard RIFF/WAVE with `WAVE_FORMAT_PCM` (0x0001) or `WAVE_FORMAT_EXTENSIBLE` (0xFFFE) and 16-bit samples. |
| IMA ADPCM WAV | `.wav` | RIFF/WAVE with `WAVE_FORMAT_ADPCM` (0x0011), decoded by the built-in `adpcm_ima` module. Must have 4 or 16 bits per sample. |

### JPEG conversion

The built-in JPEG decoder only supports baseline JPEGs. The file extension is also treated case-sensitively, so `.JPG` or `.JPEG` will **not** be recognised.

To convert an image to a supported baseline JPEG:

```bash
convert input.jpg -interlace none output.jpg
```

`input.jpg` can be any image format that ImageMagick supports (including PNG), and `-interlace none` forces a non-progressive JPEG.
