# Media Player

The Media Player plays video and audio from the SD card or internal storage. Video is MJPEG shown at the clip's recorded frame rate; audio is played through `AudioManager` with the system volume. It is preinstalled with the OS (`com.micropythonos.media_player`), so it needs no download; updating it goes through the App Store.

## Supported files

| Format | Extension | Video | Audio |
|-----------|-----------|-------|-------|
| AVI (RIFF) | `.avi` | MJPEG (fourcc `MJPG`) | none, PCM (8/16-bit) or IMA ADPCM |
| Raw MJPEG stream | `.mjpeg`, `.mjpg` | back-to-back JPEG frames | optional `.wav` sidecar with the same name |
| WAV | `.wav` | — | PCM (8/16/24/32-bit) or IMA ADPCM |
| MP3 | `.mp3` | — | MPEG-1/2/2.5 Layer III, CBR and VBR |
| RTTTL ringtone | `.rtttl` | — | tone sequence played on the piezo buzzer |

Audio in an AVI is played from inside the file, so one file is enough. A raw MJPEG stream has nowhere to carry sound: put a WAV next to it with the base name changed, e.g. `clip.mjpeg` + `clip.wav`. MP3 playback needs the `mp3dec` module, which ships in current firmware; the app hides `.mp3` files when the module is missing. RTTTL tunes need a buzzer output; on a board without one the app says so instead of failing silently.

Anything else is named rather than silently ignored:

- `not an AVI file (only AVI with MJPEG video is supported)` — e.g. an MP4 with the wrong extension
- `AMV files are not supported (convert to MJPEG AVI first)` — the format used by cheap MP5/MP4 players. It is already MJPEG with IMA ADPCM inside, so re-muxing it to a normal AVI makes it play; only the deliberately malformed container stops it
- `unsupported video codec: H264 (this player plays MJPEG)` — container is fine, the pictures are not
- an unsupported soundtrack (AC-3, MP3, MS ADPCM) still plays the pictures and prints why there is no sound

There is no H.264/H.265 or MPEG decode on the badge, and no scaling: frames are presented 1:1, so a clip larger than the panel is cropped by the display rather than shrunk. Resize to the panel size (320x240 on the Fri3d badges) when preparing clips.

## Where files go

The picker opens at `/data/video` (where the preloaded sample clip lives), then falls back to `/sdcard/video`, `/data/audio`, `/sdcard/audio`, `/sdcard/clips`, then `/sdcard`. Any file whose name ends in `.avi`, `.mjpeg`, `.mjpg`, `.wav`, `.mp3` or `.rtttl` is listed. The Media Player is also registered as a viewer for those extensions, so "Open with" from the file manager works (Music Player also offers itself for `.wav`, `.mp3` and `.rtttl`).

### Folders and playlists

Tapping a folder in the picker **enters** it, so nested folders are reachable. To play a folder as a playlist, long-press the folder to select it (or just press **Confirm** while inside it) and confirm: every media file in it plays in name order, which is why a naming scheme like `01_intro.avi`, `02_demo.avi` is worth using. Video and audio can be mixed in one folder.

When the last item ends, video playback closes the player, while audio stops and shows `Finished` so it can be replayed or replaced; with `loop` requested, the folder starts over from the first item. Each item is started like a fresh file, so a video gets its own frame rate from the container, its own soundtrack and its own timing report; a file that cannot be played stops the playlist with its message rather than being skipped silently.

### Preparing clips

Both video formats are plain MJPEG, which every desktop player opens without extra codecs. With `ffmpeg`:

```bash
# AVI with sound inside (PCM), sized to the badge panel
ffmpeg -i input.mp4 -vf "scale=320:240,fps=15" -c:v mjpeg -q:v 10 \
       -c:a pcm_s16le -ar 22050 -ac 1 clip.avi

# Raw MJPEG plus a WAV sidecar
ffmpeg -i input.mp4 -vf "scale=320:240,fps=15" -c:v mjpeg -q:v 10 -an clip.mjpeg
ffmpeg -i input.mp4 -vn -ac 1 -ar 22050 -c:a pcm_s16le clip.wav
```

`-q:v 10` is the recommended quality. It is ffmpeg's MJPEG quantiser multiplier, **not** a CRF: the scale runs 1-31 with **lower meaning better**, so `2` is the highest quality and `31` the lowest. `-q:v 10` lands near x264's `CRF 23`; do not copy a CRF value such as 18 into `-q:v`, it would look far worse there. Anything from **6 to 16 is acceptable** — 6 for high quality (about 1.3x the size of 10), 16 when space is tight (about 0.8x). Measure the size before shipping a clip into firmware, where space is the constraint the quality is traded against.

15 fps is close to the ceiling for a 320x240 clip: the player presents 13.6-14.1 fps of one, dropping 5 or 6 frames in 150. Use 12 fps if a clip must never lose a frame, and smaller frame sizes to buy frame rate — 160x120 reaches the high twenties. The measured cost model is in [Video Performance](video-performance.md).

## Controls

Tap the picture (or the music note, for audio) to pause and play. For video, a control bar appears while paused and for three seconds after any interaction; for audio it stays on screen. RTTTL has no sample position, so its bar shows restart and close only; pause and scrubbing do not apply.

| Control | Action |
|---------|--------|
| Scrub bar | Tap or drag to jump anywhere in the clip or track; the seek applies on release |
| `<<` / `>>` | Back or forward fifteen seconds |
| Restart | Play from the beginning |
| Play/pause | Pause or resume |
| Close | Leave the player |

On the badge: **A** pauses and resumes, **B** and **Y** seek back and forward fifteen seconds, **X** leaves. Seeking with the bar while a video is paused shows the frame at the new position and stays paused, so it doubles as a preview. Audio seeks are sample-accurate for WAV and land on the nearest MP3 frame.

## Behaviour

- Frames are dropped to keep real time: a clip that cannot be decoded fast enough still plays at the right speed, skipping pictures instead of slowing down.
- Sound and picture share one clock. If the pump falls behind, queued sound is dropped rather than allowed to drift out of sync, and the player reports how much it had to drop.
- Audio keeps playing when another app comes to the front, like the Music Player; the control timer resumes with the activity.
- Leaving is deferred by one LVGL pass, so the screen is never torn down from inside its own drawing callback.
- The app prints per-stage timings (read, decode, present, audio, LVGL pass) after video playback, which is how the numbers in [Video Performance](video-performance.md) were collected.

## See also

- [Video Performance](video-performance.md): why MJPEG, the cost model, measured frame rates, audio and A/V sync
- [App Performance](app-performance.md): general UI and rendering guidance
