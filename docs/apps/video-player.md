# Video Player

The Video Player plays MJPEG video from the SD card with its sound, at the clip's recorded frame rate. It is a built-in app (`com.micropythonos.video_player`), so it ships with the OS and needs no download.

## Supported files

| Container | Extension | Video | Audio |
|-----------|-----------|-------|-------|
| AVI (RIFF) | `.avi` | MJPEG (fourcc `MJPG`) | none, PCM (8/16-bit) or IMA ADPCM |
| Raw MJPEG stream | `.mjpeg`, `.mjpg` | back-to-back JPEG frames | optional `.wav` sidecar with the same name |

Audio in an AVI is played from inside the file, so one file is enough. A raw MJPEG stream has nowhere to carry sound: put a WAV next to it with the base name changed, e.g. `clip.mjpeg` + `clip.wav`.

Anything else is named rather than silently ignored:

- `not an AVI file (only AVI with MJPEG video is supported)` — e.g. an MP4 with the wrong extension
- `unsupported video codec: H264 (this player plays MJPEG)` — container is fine, the pictures are not
- an unsupported soundtrack (AC-3, MP3, MS ADPCM) still plays the pictures and prints why there is no sound

There is no H.264/H.265 or MPEG decode on the badge, and no scaling: frames are presented 1:1, so a clip larger than the panel is cropped by the display rather than shrunk. Resize to the panel size (320x240 on the Fri3d badges) when preparing clips.

## Where clips go

The picker opens at `/sdcard/video` and falls back to `/sdcard/clips`, then `/sdcard`. Any clip whose name ends in `.avi`, `.mjpeg` or `.mjpg` is listed. The video player is also registered as a viewer for those extensions, so "Open with" from the file manager works.

### Preparing clips

Both formats are plain MJPEG, which every desktop player opens without extra codecs. With `ffmpeg`:

```bash
# AVI with sound inside (PCM), sized to the badge panel
ffmpeg -i input.mp4 -vf "scale=320:240,fps=15" -c:v mjpeg -q:v 6 \
       -c:a pcm_s16le -ar 22050 -ac 1 clip.avi

# Raw MJPEG plus a WAV sidecar
ffmpeg -i input.mp4 -vf "scale=320:240,fps=15" -c:v mjpeg -q:v 6 -an clip.mjpeg
ffmpeg -i input.mp4 -vn -ac 1 -ar 22050 -c:a pcm_s16le clip.wav
```

Smaller frame sizes buy frame rate: at 320x240 the panel and the SD card together cap the player near 12 fps, while 160x120 clips reach the high twenties. The measured cost model is in [Video Performance](video-performance.md).

## Controls

Tap the picture to pause and play. A control bar appears while paused and for three seconds after any interaction:

| Control | Action |
|---------|--------|
| Scrub bar | Tap or drag to jump anywhere in the clip; the seek applies on release |
| `<<` / `>>` | Back or forward fifteen seconds |
| Restart | Play from the beginning |
| Play/pause | Pause or resume |
| Close | Leave the player |

On the badge: **A** pauses and resumes, **B** and **Y** seek back and forward fifteen seconds, **X** leaves. Seeking with the bar while paused shows the frame at the new position and stays paused, so it doubles as a preview.

## Behaviour

- Frames are dropped to keep real time: a clip that cannot be decoded fast enough still plays at the right speed, skipping pictures instead of slowing down.
- Sound and picture share one clock. If the pump falls behind, queued sound is dropped rather than allowed to drift out of sync, and the player reports how much it had to drop.
- Leaving is deferred by one LVGL pass, so the screen is never torn down from inside its own drawing callback.
- The app prints per-stage timings (read, decode, present, audio, LVGL pass) after playback, which is how the numbers in [Video Performance](video-performance.md) were collected.

## See also

- [Video Performance](video-performance.md): why MJPEG, the cost model, measured frame rates, audio and A/V sync
- [App Performance](app-performance.md): general UI and rendering guidance
