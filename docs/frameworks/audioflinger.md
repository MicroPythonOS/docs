# AudioFlinger

MicroPythonOS provides a centralized audio service called **AudioFlinger**, inspired by Android's architecture. It manages audio playback across different hardware outputs with priority-based audio focus control.

## Overview

AudioFlinger provides:

- **Priority-based audio focus** - Higher priority streams interrupt lower priority ones
- **Multiple audio devices** - I2S digital audio, PWM buzzer, or both
- **Background playback** - Runs in separate thread
- **WAV file support** - 8/16/24/32-bit PCM, mono/stereo, auto-upsampling
- **RTTTL ringtone support** - Full Ring Tone Text Transfer Language parser
- **Thread-safe** - Safe for concurrent access
- **Hardware-agnostic** - Apps work across all platforms without changes

## Supported Audio Devices

- **I2S**: Digital audio output for WAV file playback (Fri3d badge, Waveshare board)
- **Buzzer**: PWM-based tone/ringtone playback (Fri3d badge only)
- **Both**: Simultaneous I2S and buzzer support
- **Null**: No audio (desktop/Linux)

## Quick Start

### Playing WAV Files

```python
from mpos.app.activity import Activity
import mpos.audio.audioflinger as AudioFlinger

class MusicPlayerActivity(Activity):
    def onCreate(self):
        # Play a music file
        success = AudioFlinger.play_wav(
            "M:/sdcard/music/song.wav",
            stream_type=AudioFlinger.STREAM_MUSIC,
            volume=80,
            on_complete=lambda msg: print(msg)
        )

        if not success:
            print("Playback rejected (higher priority stream active)")
```

**Supported formats:**
- **Encoding**: PCM (8/16/24/32-bit)
- **Channels**: Mono or stereo
- **Sample rate**: Any rate (auto-upsampled to ≥22050 Hz)

### Playing RTTTL Ringtones

```python
# Play notification sound via buzzer
rtttl = "Nokia:d=4,o=5,b=225:8e6,8d6,8f#,8g#,8c#6,8b,d,8p,8b,8a,8c#,8e"
AudioFlinger.play_rtttl(
    rtttl,
    stream_type=AudioFlinger.STREAM_NOTIFICATION
)
```

**RTTTL format example:**
```
name:settings:notes
Nokia:d=4,o=5,b=225:8e6,8d6,8f#,8g#
```

- `d=4` - Default duration (quarter note)
- `o=5` - Default octave
- `b=225` - Beats per minute
- Notes: `8e6` = 8th note, E in octave 6

### Volume Control

```python
# Set volume (0-100)
AudioFlinger.set_volume(70)

# Get current volume
volume = AudioFlinger.get_volume()
print(f"Current volume: {volume}")
```

### Stopping Playback

```python
# Stop currently playing audio
AudioFlinger.stop()
```

## Audio Focus Priority

AudioFlinger implements a 3-tier priority-based audio focus system inspired by Android:

| Stream Type | Priority | Use Case | Behavior |
|-------------|----------|----------|----------|
| **STREAM_ALARM** | 2 (Highest) | Alarms, alerts | Interrupts all other streams |
| **STREAM_NOTIFICATION** | 1 (Medium) | Notifications, UI sounds | Interrupts music, rejected by alarms |
| **STREAM_MUSIC** | 0 (Lowest) | Music, podcasts | Interrupted by everything |

### Priority Rules

1. **Higher priority interrupts lower priority**: ALARM > NOTIFICATION > MUSIC
2. **Equal priority is rejected**: Can't play two alarms simultaneously
3. **Lower priority is rejected**: Can't start music while alarm is playing

### Example: Priority in Action

```python
# Start playing music (priority 0)
AudioFlinger.play_wav("music.wav", stream_type=AudioFlinger.STREAM_MUSIC)

# Notification sound (priority 1) interrupts music
AudioFlinger.play_rtttl("beep:d=4:8c", stream_type=AudioFlinger.STREAM_NOTIFICATION)
# Music stops, notification plays

# Try to play another notification while first is playing
success = AudioFlinger.play_rtttl("beep:d=4:8d", stream_type=AudioFlinger.STREAM_NOTIFICATION)
# Returns False - equal priority rejected

# Alarm (priority 2) interrupts notification
AudioFlinger.play_wav("alarm.wav", stream_type=AudioFlinger.STREAM_ALARM)
# Notification stops, alarm plays
```

## Hardware Support Matrix

| Board | I2S | Buzzer | Notes |
|-------|-----|--------|-------|
| **Fri3d 2024 Badge** | ✅ GPIO 2, 47, 16 | ✅ GPIO 46 | Both devices available |
| **Waveshare ESP32-S3** | ✅ GPIO 2, 47, 16 | ❌ | I2S only |
| **Linux/macOS** | ❌ | ❌ | No audio (desktop builds) |

**I2S Pins:**
- **BCLK** (Bit Clock): GPIO 2
- **WS** (Word Select): GPIO 47
- **DOUT** (Data Out): GPIO 16

**Buzzer Pin:**
- **PWM**: GPIO 46 (Fri3d badge only)

## Configuration

Audio device preference is configured in the Settings app under **"Advanced Settings"**:

- **Auto-detect**: Use available hardware (default)
- **I2S (Digital Audio)**: Digital audio only
- **Buzzer (PWM Tones)**: Tones/ringtones only
- **Both I2S and Buzzer**: Use both devices
- **Disabled**: No audio

**⚠️ Important**: Changing the audio device requires a **restart** to take effect.

### How Auto-Detect Works

1. Checks if I2S hardware is available
2. Checks if Buzzer hardware is available
3. Selects the best available option:
   - Both devices available → Use "Both"
   - Only I2S → Use "I2S"
   - Only Buzzer → Use "Buzzer"
   - Neither → Use "Null" (no audio)

## Complete Example: Music Player with Controls

```python
from mpos.app.activity import Activity
import mpos.audio.audioflinger as AudioFlinger
import lvgl as lv

class SimpleMusicPlayerActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()

        # Play button
        play_btn = lv.button(self.screen)
        play_btn.set_size(100, 50)
        play_btn.set_pos(10, 50)
        play_label = lv.label(play_btn)
        play_label.set_text("Play")
        play_label.center()
        play_btn.add_event_cb(lambda e: self.play_music(), lv.EVENT.CLICKED, None)

        # Stop button
        stop_btn = lv.button(self.screen)
        stop_btn.set_size(100, 50)
        stop_btn.set_pos(120, 50)
        stop_label = lv.label(stop_btn)
        stop_label.set_text("Stop")
        stop_label.center()
        stop_btn.add_event_cb(lambda e: AudioFlinger.stop(), lv.EVENT.CLICKED, None)

        # Volume slider
        volume_label = lv.label(self.screen)
        volume_label.set_text("Volume:")
        volume_label.set_pos(10, 120)

        volume_slider = lv.slider(self.screen)
        volume_slider.set_size(200, 10)
        volume_slider.set_pos(10, 150)
        volume_slider.set_range(0, 100)
        volume_slider.set_value(AudioFlinger.get_volume(), False)
        volume_slider.add_event_cb(
            lambda e: AudioFlinger.set_volume(volume_slider.get_value()),
            lv.EVENT.VALUE_CHANGED,
            None
        )

        self.setContentView(self.screen)

    def play_music(self):
        success = AudioFlinger.play_wav(
            "M:/sdcard/music/song.wav",
            stream_type=AudioFlinger.STREAM_MUSIC,
            volume=AudioFlinger.get_volume(),
            on_complete=lambda msg: print(f"Playback finished: {msg}")
        )

        if not success:
            print("Playback rejected - higher priority audio active")
```

## API Reference

### Functions

**`play_wav(path, stream_type=STREAM_MUSIC, volume=None, on_complete=None)`**

Play a WAV file.

- **Parameters:**
  - `path` (str): Path to WAV file (e.g., `"M:/sdcard/music/song.wav"`)
  - `stream_type` (int): Stream type constant (STREAM_ALARM, STREAM_NOTIFICATION, STREAM_MUSIC)
  - `volume` (int, optional): Volume 0-100. If None, uses current volume
  - `on_complete` (callable, optional): Callback function called when playback completes

- **Returns:** `bool` - `True` if playback started, `False` if rejected (higher/equal priority active)

**`play_rtttl(rtttl, stream_type=STREAM_NOTIFICATION)`**

Play an RTTTL ringtone.

- **Parameters:**
  - `rtttl` (str): RTTTL format string (e.g., `"Nokia:d=4,o=5,b=225:8e6,8d6"`)
  - `stream_type` (int): Stream type constant

- **Returns:** `bool` - `True` if playback started, `False` if rejected

**`set_volume(volume)`**

Set playback volume.

- **Parameters:**
  - `volume` (int): Volume level 0-100

**`get_volume()`**

Get current playback volume.

- **Returns:** `int` - Current volume (0-100)

**`stop()`**

Stop currently playing audio.

### Stream Type Constants

- `AudioFlinger.STREAM_ALARM` - Highest priority (value: 2)
- `AudioFlinger.STREAM_NOTIFICATION` - Medium priority (value: 1)
- `AudioFlinger.STREAM_MUSIC` - Lowest priority (value: 0)

## Troubleshooting

### Playback Rejected

**Symptom**: `play_wav()` or `play_rtttl()` returns `False`, no sound plays

**Cause**: Higher or equal priority stream is currently active

**Solution**:
1. Check if higher priority audio is playing
2. Wait for higher priority stream to complete
3. Use `AudioFlinger.stop()` to force stop current playback (use sparingly)
4. Use equal or higher priority stream type

```python
# Check if playback was rejected
success = AudioFlinger.play_wav("sound.wav", stream_type=AudioFlinger.STREAM_MUSIC)
if not success:
    print("Higher priority audio is playing")
    # Option 1: Wait and retry
    # Option 2: Stop current playback (if appropriate)
    AudioFlinger.stop()
    AudioFlinger.play_wav("sound.wav", stream_type=AudioFlinger.STREAM_MUSIC)
```

### WAV File Not Playing

**Symptom**: File exists but doesn't play, or plays with distortion

**Cause**: Unsupported WAV format

**Requirements:**
- **Encoding**: PCM only (not MP3, AAC, or other compressed formats)
- **Bit depth**: 8, 16, 24, or 32-bit
- **Channels**: Mono or stereo
- **Sample rate**: Any (auto-upsampled to ≥22050 Hz)

**Solution**: Convert WAV file to supported format using ffmpeg or audacity:

```bash
# Convert to 16-bit PCM, 44100 Hz, stereo
ffmpeg -i input.wav -acodec pcm_s16le -ar 44100 -ac 2 output.wav

# Convert to 16-bit PCM, 22050 Hz, mono (smaller file)
ffmpeg -i input.wav -acodec pcm_s16le -ar 22050 -ac 1 output.wav
```

### No Sound on Device

**Symptom**: Playback succeeds but no sound comes out

**Possible causes:**

1. **Volume set to 0**
   ```python
   AudioFlinger.set_volume(50)  # Set to 50%
   ```

2. **Wrong audio device selected**
   - Check Settings → Advanced Settings → Audio Device
   - Try "Auto-detect" or manually select device
   - **Restart required** after changing audio device

3. **Hardware not available (desktop)**
   ```python
   # Desktop builds have no audio hardware
   # AudioFlinger will return success but produce no sound
   ```

4. **I2C/Speaker not connected (Fri3d badge)**
   - Check if speaker is properly connected
   - Verify I2S pins are not used by other peripherals

### Audio Cuts Out or Stutters

**Symptom**: Playback starts but stops unexpectedly or has gaps

**Cause**: Higher priority stream interrupting, or insufficient memory

**Solution**:
1. **Check for interrupting streams:**
   ```python
   # Avoid notifications during music playback
   # Or use callbacks to resume after interruption
   ```

2. **Reduce memory usage:**
   - Close unused apps
   - Use lower bitrate WAV files (22050 Hz instead of 44100 Hz)
   - Free up memory with `gc.collect()`

3. **Check SD card speed:**
   - Use Class 10 or faster SD card for smooth WAV playback

### Restart Required After Configuration Change

**Symptom**: Changed audio device in Settings but still using old device

**Cause**: Audio device is initialized at boot time

**Solution**: Restart the device after changing audio device preference

```python
# After changing setting, restart is required
# Settings app should display a message: "Restart required to apply changes"
```

### RTTTL Not Playing on Waveshare

**Symptom**: RTTTL tones don't play on Waveshare board

**Cause**: Waveshare board has no buzzer (PWM speaker)

**Solution**:
- Use WAV files instead of RTTTL on Waveshare
- RTTTL requires hardware buzzer (Fri3d badge only)
- I2S cannot produce RTTTL tones

## Performance Tips

### Optimizing WAV Files

- **File size**: Use 22050 Hz instead of 44100 Hz to reduce file size by 50%
- **Channels**: Use mono instead of stereo if positional audio isn't needed
- **Bit depth**: Use 16-bit for good quality with reasonable size

```bash
# Small file, good quality (recommended)
ffmpeg -i input.wav -acodec pcm_s16le -ar 22050 -ac 1 output.wav

# High quality, larger file
ffmpeg -i input.wav -acodec pcm_s16le -ar 44100 -ac 2 output.wav
```

### Background Playback

AudioFlinger runs in a separate thread, so playback doesn't block the UI:

```python
# This doesn't block - returns immediately
AudioFlinger.play_wav("long_song.wav", stream_type=AudioFlinger.STREAM_MUSIC)

# UI remains responsive while audio plays
# Use on_complete callback to know when finished
AudioFlinger.play_wav(
    "song.wav",
    on_complete=lambda msg: print("Playback finished")
)
```

### Memory Management

- WAV files are streamed from SD card (not loaded into RAM)
- Buffers are allocated during playback and freed after
- Multiple simultaneous streams not supported (priority system prevents this)

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [LightsManager](lights-manager.md) - LED control for visual feedback
- [SharedPreferences](preferences.md) - Store audio preferences
