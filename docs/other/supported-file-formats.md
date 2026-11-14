# Supported File Formats

MicroPythonOS uses LVGL's built-in image decoders and its own audio stack. The formats below are supported out of the box for apps such as the Image Viewer, Media Player and Music Player.

## Image formats

| Format | Extensions | Notes |
|--------|------------|-------|
| BMP | `.bmp` | Supports RGB565 and RGB888 |
| PNG | `.png` | Fully supported via LodePNG. |
| Baseline JPEG | `.jpg`, `.jpeg` | Fully supported via TJpgD. |
| Progressive JPEG | `.jpg`, `.jpeg` | **Not supported.** Files will decode as `0x0` and appear blank. |
| RAW | `.raw` | Supported by the Image Viewer app if named as `<name>_<w>x<h>_RGB565.raw` or `<name>_<w>x<h>_GRAY.raw`. |

## Audio formats

| Format | Extensions | Notes |
|--------|------------|-------|
| Plain PCM WAV | `.wav` | Standard RIFF/WAVE with `WAVE_FORMAT_PCM` (0x0001) or `WAVE_FORMAT_EXTENSIBLE` (0xFFFE); 8, 16, 24 or 32-bit samples, mono or stereo. |
| IMA ADPCM WAV | `.wav` | RIFF/WAVE with `WAVE_FORMAT_ADPCM` (0x0011), decoded by the built-in `adpcm_ima` module. Must have 4 or 16 bits per sample. |
| MP3 | `.mp3` | MPEG-1/2/2.5 Layer III, constant or variable bitrate, 8–48 kHz, mono or stereo, through the native `mp3dec` module. |

### JPEG conversion

The built-in JPEG decoder only supports baseline JPEGs. The file extension is also treated case-sensitively, so `.JPG` or `.JPEG` will **not** be recognised.

To convert an image to a supported baseline JPEG:

```bash
convert input.jpg -interlace none output.jpg
```

`input.jpg` can be any image format that ImageMagick supports (including PNG), and `-interlace none` forces a non-progressive JPEG.
