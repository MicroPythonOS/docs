# AudioManager

MicroPythonOS provides a centralized audio service called **AudioManager**, inspired by Android's architecture. It manages audio playback and recording across different hardware outputs with priority-based audio focus control and a device registry for routing.

## Overview

AudioManager provides:

- **Priority-based audio focus** - Higher priority streams interrupt lower priority ones
- **Device registry** - Register I2S outputs, PWM buzzers, and microphone inputs
- **Session control** - Player/Recorder sessions with start/stop/pause/resume
- **WAV file support** - 8/16/24/32-bit PCM, IMA ADPCM (4-bit), mono/stereo, auto-upsampling
- **MP3 file support** - MPEG-1/2/2.5 Layer III, CBR and VBR, mono/stereo, 8-48 kHz
- **WAV recording** - 16-bit mono PCM from I2S or ADC microphone
- **RTTTL ringtone support** - Full Ring Tone Text Transfer Language parser
- **Thread-safe** - Safe for concurrent access
- **Hardware-agnostic** - Apps work across all platforms without changes

## Supported Audio Devices

- **I2S Output**: Digital audio output for WAV file playback
- **I2S Input**: Microphone recording (I2S mic)
- **ADC Input**: Analog microphone recording via ADC (Fri3d 2026)
- **Buzzer**: PWM-based ringtone playback, and WAV/MP3 playback through the
  piezo's average-voltage response on ESP32 (see
  [Playing PCM audio on a piezo buzzer](#playing-pcm-audio-on-a-piezo-buzzer-esp32))

## Quick Start

### Registering Devices

AudioManager uses a registry of device descriptors instead of direct hardware initialization.
Register devices once at startup (typically in your board init file):

```python
from mpos import AudioManager

# I2S speaker
AudioManager.add(
    AudioManager.Output(
        name="speaker",
        kind="i2s",
        channels=2,
        i2s_pins={"sck": 2, "ws": 47, "sd": 16, "mck": 2},
        preferred_sample_rate=44100,
    )
)

# I2S microphone
AudioManager.add(
    AudioManager.Input(
        name="mic",
        kind="i2s",
        i2s_pins={"sck_in": 17, "ws": 47, "sd_in": 15},
        preferred_sample_rate=16000,
    )
)

# Buzzer for RTTTL
AudioManager.add(
    AudioManager.Output(
        name="buzzer",
        kind="buzzer",
        buzzer_pin=46,
    )
)
```

### Playing WAV Files

```python
from mpos import AudioManager

player = AudioManager.player(
    file_path="M:/sdcard/music/song.wav",
    stream_type=AudioManager.STREAM_MUSIC,
    volume=80,
    on_complete=lambda msg: print(msg),
)
player.start()
```

**Supported formats:**
- **Encoding**: PCM (8/16/24/32-bit), IMA ADPCM (4-bit)
- **Channels**: Mono or stereo
- **Sample rate**: Any rate (auto-upsampled to ≥8000 Hz)
- **Repeat**: Pass `repeat_count` to loop playback N times

### Playing MP3 Files

MP3 files are handled by the same player API; the file extension picks the stream implementation:

```python
player = AudioManager.player(
    file_path="M:/sdcard/music/song.mp3",
    stream_type=AudioManager.STREAM_MUSIC,
    volume=80,
)
player.start()
```

**Supported formats:**
- **Encoding**: MPEG-1, MPEG-2 and MPEG-2.5 Layer III (CBR and VBR)
- **Channels**: Mono or stereo
- **Sample rate**: 8, 11.025, 12, 16, 22.05, 24, 32, 44.1 and 48 kHz
- **Repeat**: Pass `repeat_count` to loop playback N times

Decoding uses the `mp3dec` native module (the public-domain [minimp3](https://github.com/lieff/minimp3) decoder) so the LVGL UI keeps running while audio plays: the GIL is released around each frame decode. The native sample rate of the file is used directly, without resampling. ID3v2 tags (including cover art) and ID3v1 trailers are skipped, and reading the sample rate and duration for progress display works without decoding.

Files without an audio frame (or with free-format bitrate) are not supported.

### Playing PCM Audio on a Piezo Buzzer (ESP32)

A passive piezo disc is not a speaker, but it can still reproduce samples from a
WAV or MP3 file. The disc deflects in proportion to the *average* voltage on its
pin, so a PWM channel whose duty cycle is rewritten at the sample rate acts as
a very cheap DAC. On ESP32 builds MicroPythonOS ships a `buzzer` driver
(`c_mpos/src/buzzer.c`) that does exactly that, and the usual player API works
unchanged:

```python
from mpos import AudioManager

player = AudioManager.player(
    file_path="data/alert.wav",
    stream_type=AudioManager.STREAM_NOTIFICATION,
    volume=100,
    output=AudioManager.find_output_by_kind("buzzer"),
)
player.start()
```

Anything a buzzer output plays that is not RTTTL goes through this PCM path:
WAV (PCM/ADPCM) and MP3 are decoded by the same code as the I2S output, and
volume changes apply while playing.

Expect it to be **quiet and narrow-band**. The disc has a mechanical resonance
somewhere around 3-4 kHz and rolls off quickly below it, so pure low sine waves
are barely audible while harmonics (music, square-wave ringtones) come through
much better. A pure 440 Hz test tone, for example, is essentially inaudible;
what you hear is the PWM noise exciting the disc's resonance.

If the source is harmonic-poor (sine tones, voice, bass) you can overdrive the
samples before they reach the PWM, which turns them into near-square waves full
of harmonics that the disc can reproduce:

```python
output = AudioManager.find_output_by_kind("buzzer")
output.buzzer_gain = 600   # 100 = unchanged, 600 = 6x with hard clipping
```

Buzzer outputs default to `1600` (the disc is quiet enough that unity gain makes
most material inaudible); set it to `100` if you want the samples untouched.

It is set per output (also accepted as `buzzer_gain=` in
`AudioManager.Output(...)`) and applies to the next clip.

Apps that decode and pace their own PCM instead of using a Player (the video
player is one) can stream to any output through the small sink API:

```python
from mpos.audio.stream_buzzer import BuzzerSink

sink = BuzzerSink(AudioManager.find_output_by_kind("buzzer"), rate=22050, channels=1)
sink.open()
sink.write(pcm_bytes)   # blocks only if the ~370 ms ring is full
sink.close()
```

`BuzzerSink` mirrors the `open()/wait_ready()/write()/close()` shape of the
I2S sink, so an app can pick it when `output.kind == "buzzer"` and keep its
existing pacing.

Driver notes, in case you need to work on it:

- The PWM carrier runs at 78 kHz (80 MHz APB clock with 10-bit duty resolution)
  and the duty cycle is rewritten from a hardware timer interrupt at the file's
  sample rate (8-48 kHz). Samples are queued in an 8 KB ring buffer in internal
  RAM, so the playback thread can be delayed by a GC pause or a flash read
  without an audible gap.
- The ESP32 backend uses LEDC timer 3 / channel 7 and `machine.Timer(1)`, so it
  does not collide with `machine.PWM` (allocates from 0 upwards),
  the LVGL task handler (`machine.Timer(0)`) or the warm-output timer of the I2S
  stream (`machine.Timer(3)`).
- The timer interrupt is reserved when the board registers its buzzer output
  (`AudioManager.add(...)` at boot) and is kept for the lifetime of the
  firmware. Allocating it lazily at play time can fail: by then the rest of the
  firmware may have taken every free low/medium interrupt vector, and
  `esp_intr_alloc()` returns `ESP_ERR_NOT_FOUND`.
- On desktop and web the same decode -> mono -> volume chain runs into an
  emulated sink that writes a temporary WAV file, so it can be exercised and
  unit-tested without hardware.

### Playing RTTTL Ringtones

```python
from mpos import AudioManager

rtttl = "Nokia:d=4,o=5,b=225:8e6,8d6,8f#,8g#,8c#6,8b,d,8p,8b,8a,8c#,8e"
player = AudioManager.rtttl_player(
    rtttl,
    stream_type=AudioManager.STREAM_NOTIFICATION,
)
player.start()
```

### Recording Audio (I2S Mic)

```python
from mpos import AudioManager

recorder = AudioManager.recorder(
    file_path="data/my_recording.wav",
    duration_ms=10000,
    sample_rate=16000,
)
recorder.start()
```

### Recording Audio (ADC Mic)

```python
from mpos import AudioManager

AudioManager.record_wav_adc(
    file_path="data/adc_recording.wav",
    duration_ms=5000,
    sample_rate=16000,
    adc_pin=1,
)
```

### Desktop WAV Playback

On Linux and macOS desktop builds, WAV playback uses an external audio player subprocess instead of I2S hardware. `AudioManager` automatically detects one of the following players in order of preference:

1. `ffplay` (FFmpeg)
2. `aplay` (ALSA)
3. `paplay` (PulseAudio)
4. `afplay` (macOS)

MP3 playback prefers a player that can decode MP3 itself (`ffplay`, then `mpg123`, or `afplay` on macOS). When only `aplay`/`paplay` is available, the file is decoded to a temporary WAV first, so older systems without an MP3-capable player still work.

If none is installed, the player still runs in "timing simulation" mode so apps can test playback flow without sound.

```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# macOS
# afplay is pre-installed
```

Priority and volume controls apply conceptually on desktop, although the external player handles the actual output.

### ADPCM WAV Encoding

IMA ADPCM compresses 16-bit PCM to 4 bits per sample, roughly quartering WAV file size with minimal quality loss — ideal for storage-constrained devices.

`ffmpeg` can encode ADPCM (`-acodec adpcm_ima_wav`) but has two drawbacks: it prepends silence that lengthens the file, and it only supports the default 4-bit depth. MicroPythonOS currently decodes 4-bit IMA ADPCM only. For better encoding results than `ffmpeg`, use [adpcm-xq](https://github.com/dbry/adpcm-xq), which avoids the silence padding and additionally supports 2 and 3-bit ADPCM (still good quality, even smaller files). If there's demand for it, 2 and 3-bit ADPCM decoding support could be added to MicroPythonOS too.

```bash
# ffmpeg (adds silence, 4-bit only)
ffmpeg -i input.wav -acodec adpcm_ima_wav output.wav

# adpcm-xq (clean, supports 2, 3 and 4-bit)
adpcm-xq input.wav output.wav
```

## Audio Focus Priority

AudioManager implements a 3-tier priority-based audio focus system inspired by Android:

| Stream Type | Priority | Use Case | Behavior |
|-------------|----------|----------|----------|
| **STREAM_ALARM** | 2 (Highest) | Alarms, alerts | Interrupts all other streams |
| **STREAM_NOTIFICATION** | 1 (Medium) | Notifications, UI sounds | Interrupts music, rejected by alarms |
| **STREAM_MUSIC** | 0 (Lowest) | Music, podcasts | Interrupted by everything |

### Priority Rules

1. **Higher priority interrupts lower priority**: ALARM > NOTIFICATION > MUSIC
2. **Equal priority is rejected**: Can't play two alarms simultaneously
3. **Lower priority is rejected**: Can't start music while alarm is playing

## Device Routing and Conflicts

AudioManager prevents conflicts by tracking pin usage and sample rates:

- **Shared clocks**: If two I2S sessions share `ws`/`sck`, they must use the same sample rate.
- **Conflicts**: Starting a new session stops conflicting sessions automatically.

## Volume Control

```python
from mpos import AudioManager

AudioManager.set_volume(70)
volume = AudioManager.get_volume()
print(f"Current volume: {volume}")
```

Volume changes affect active sessions when the stream supports volume adjustment.

## API Reference

### Device Registration

**`AudioManager.add(device)`**

Register a device descriptor.

- **Parameters:**
  - `device` - `AudioManager.Output` or `AudioManager.Input`

**`AudioManager.get_outputs()`** / **`AudioManager.get_inputs()`**

Return a list of registered devices.

**`AudioManager.get_default_output()`** / **`AudioManager.get_default_input()`**

Return the default device for playback or recording.

**`AudioManager.set_default_output(output)`** / **`AudioManager.set_default_input(input_device)`**

Set the default device used when a session does not specify one.

### Playback Sessions

**`AudioManager.player(file_path=None, rtttl=None, stream_type=None, on_complete=None, output=None, sample_rate=None, volume=None)`**

Create a playback session. For I2S playback use `file_path`. For RTTTL use `rtttl`.

- **Returns:** `Player`

**`AudioManager.rtttl_player(rtttl, **kwargs)`**

Shortcut for creating an RTTTL player.

**`AudioManager.get_active_player(stream_type=None, file_path=None)`**

Return the active `Player` matching optional filters.

**`AudioManager.get_active_track(stream_type=None)`**

Return file path for the active track (if any).

### Recording Sessions

**`AudioManager.recorder(file_path, input=None, sample_rate=None, on_complete=None, duration_ms=None, **adc_config)`**

Create a recording session. Use an `AudioManager.Input` descriptor for I2S or ADC input.

- **Returns:** `Recorder`

**`AudioManager.record_wav_adc(file_path, duration_ms=None, sample_rate=None, adc_pin=None, on_complete=None, **adc_config)`**

Start an ADC recording session immediately.

### Global Control

**`AudioManager.stop()`**

Stop all active playback and recording sessions.

**`AudioManager.set_volume(volume)`** / **`AudioManager.get_volume()`**

Set or get the global volume (0–100).

### `Player` Session Methods

- `start()` / `stop()` / `pause()` / `resume()`
- `is_playing()` / `is_active()`
- `get_progress_percent()` / `get_progress_ms()` / `get_duration_ms()`

### `Recorder` Session Methods

- `start()` / `stop()` / `pause()` / `resume()`
- `is_recording()` / `is_active()`
- `get_duration_ms()`

## Hardware Support Matrix

| Board | I2S Output | I2S Mic | ADC Mic | Buzzer | Notes |
|-------|------------|--------|---------|--------|-------|
| **Fri3d 2024 Badge** | ✅ | ✅ | ❌ | ✅ | Full audio support |
| **Fri3d 2026 Badge** | ✅ | ❌ | ✅ | ✅ | ADC mic via `adc_mic` |
| **Waveshare ESP32-S3** | ✅ | ❌ | ❌ | ❌ | I2S output only |
| **Linux/macOS** | ✅ (external player) | ✅ (simulated) | ✅ (simulated) | ❌ | WAV playback via `ffplay`/`aplay`/`paplay`/`afplay` |

## Troubleshooting

### Playback Rejected

**Symptom**: New playback is rejected or interrupted.

**Cause**: Higher-priority stream is active or device/pin conflict.

**Solution**:
1. Use `AudioManager.get_active_player()` to inspect active streams
2. Stop the active session with `AudioManager.stop()` if appropriate
3. Use a higher priority stream type

### WAV File Not Playing

**Requirements:**
- **Encoding**: PCM or IMA ADPCM (not MP3, AAC)
- **Bit depth**: 8, 16, 24, or 32-bit (PCM); 4-bit (ADPCM)
- **Channels**: Mono or stereo
- **Sample rate**: Any (auto-upsampled to ≥8000 Hz)

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [WidgetAnimator](widget-animator.md) - Smooth UI animations
- [LightsManager](lights-manager.md) - LED control for visual feedback
- [SharedPreferences](preferences.md) - Store audio preferences
