# AudioManager

MicroPythonOS provides a centralized audio service called **AudioManager**, inspired by Android's architecture. It manages audio playback and recording across different hardware outputs with priority-based audio focus control and a device registry for routing.

## Overview

AudioManager provides:

- **Priority-based audio focus** - Higher priority streams interrupt lower priority ones
- **Device registry** - Register I2S outputs, PWM buzzers, and microphone inputs
- **Session control** - Player/Recorder sessions with start/stop/pause/resume
- **WAV file support** - 8/16/24/32-bit PCM, mono/stereo, auto-upsampling
- **WAV recording** - 16-bit mono PCM from I2S or ADC microphone
- **RTTTL ringtone support** - Full Ring Tone Text Transfer Language parser
- **Thread-safe** - Safe for concurrent access
- **Hardware-agnostic** - Apps work across all platforms without changes

## Supported Audio Devices

- **I2S Output**: Digital audio output for WAV file playback
- **I2S Input**: Microphone recording (I2S mic)
- **ADC Input**: Analog microphone recording via ADC (Fri3d 2026)
- **Buzzer**: PWM-based tone/ringtone playback

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
- **Encoding**: PCM (8/16/24/32-bit)
- **Channels**: Mono or stereo
- **Sample rate**: Any rate (auto-upsampled to ≥8000 Hz)

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
| **Linux/macOS** | ❌ | ✅ (simulated) | ✅ (simulated) | ❌ | Simulated recording for testing |

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
- **Encoding**: PCM only (not MP3, AAC)
- **Bit depth**: 8, 16, 24, or 32-bit
- **Channels**: Mono or stereo
- **Sample rate**: Any (auto-upsampled to ≥8000 Hz)

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [WidgetAnimator](widget-animator.md) - Smooth UI animations
- [LightsManager](lights-manager.md) - LED control for visual feedback
- [SharedPreferences](preferences.md) - Store audio preferences
