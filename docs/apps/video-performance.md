# Video Playback Performance

What it costs to play video on MicroPythonOS hardware, derived from the pixel
count up, and measured on a Fri3d 2026 badge (ESP32-S3, ST7789 320x240 over
SPI, SD card on the same SPI host). Use this to predict a player's framerate
before writing one, and to know which term to attack when it is slow.

The short version: **a 320x240 frame costs ~72 ms end to end, and the three
stages are serial.** That is 13.5 fps. No amount of timer tuning changes it;
only removing a stage does.

Two things cost framerate without showing up in the frame cost: the LVGL
refresh timer (§3) and the framework's fairness gap between task-handler
passes (§4). Removing both, plus three details of how you drive the passes, is
what took a real 320x240 player from 8.9 to 13.6-14.1 fps. Check those before
buying faster hardware.

Sound is a fourth term, and a cheap one if you feed it correctly: one clock,
a preallocated ring, and a sink that is never asked to block cost about
1.25 fps (9%) on an already saturated clip, of which only 0.4-2.3 ms per frame
is the audio code itself (§6).

## 1. Model a frame as four serial stages

A player does four things per frame, one after the other:

| Stage | What it moves | Bound by |
|-------|---------------|----------|
| Ingest | encoded bytes into RAM | storage or network throughput |
| Decode | JPEG → RGB565 | CPU (`jpegdec`, C) |
| Invalidate | `image.set_src()` | pointer swap, ~0 |
| Flush | RGB565 bytes to the panel | SPI bus + DMA |

Because they are serial, the frame time is the sum:

```
frame_ms = ingest_bytes / ingest_rate
         + decode_ms
         + flush_bytes / flush_rate
fps      = 1000 / frame_ms
```

Both byte terms are the whole frame: RGB565 is `w * h * 2` bytes, MJPEG is
whatever the encoder produced. Decode and flush scale with pixel count; ingest
scales with compression ratio, which is why ingest is the term a player can
actually tune.

**Nothing overlaps.** The flush is synchronous inside the loop, so while the
panel is drawing, the SD card is idle, and vice versa. Sharing one SPI host
therefore costs exactly the sum of the two transfers — there is no arbitration
penalty to hunt for. Measured: predicted 74.11 ms vs actual 74.11 ms at
320x240 (see §7). If you want overlap, you need a separate task or DMA chain
that the current design does not have.

## 2. Measured constants (Fri3d 2026, 320x240)

| Stage | 320x240 | 240x180 | 160x120 | Per byte / pixel |
|-------|---------|---------|---------|------------------|
| Ingest from SD (MJPEG) | 33.5 ms | 24.2 ms | 15.4 ms | ~0.5 MB/s effective in-loop |
| Decode | 13.8 ms | 10.1 ms | 5.3 ms | ~0.18 µs/pixel |
| Flush (RGB565) | 24.8 ms | 13.1 ms | 8.1 ms | ~6 MB/s effective at 80 MHZ SPI |
| **Total** | **72-74 ms** | **49 ms** | **30 ms** | |
| **Ceiling** | **13.5 fps** | **20.4 fps** | **33.2 fps** | |

Two sanity checks on those constants:

* **Flush**: 153,600 bytes in 24.8 ms = 6.2 MB/s. The bus is configured for
  80 MHz (10 MB/s theoretical), so the flush runs at ~60% of the wire rate.
  That is the real number to budget, not 10 MB/s.
* **Decode**: 76,800 pixels in 13.8 ms = 0.18 µs/pixel. Costs scale with area,
  so 160x120 is about a quarter of 320x240 — matches (5.3 vs 13.8 ms).

The SD card delivers only ~0.5-0.6 MB/s here, standalone and in-loop, and does
not improve with a higher SPI clock request (20 MHz is identical; 40 MHz fails
to mount). Treat 0.6 MB/s as the storage ceiling, not a tuning knob.

## 3. The 33 ms refresh trap

LVGL only refreshes on `LV_DEF_REFR_PERIOD` (33 ms) ticks. If you drive video
from an LVGL timer, your frame rate is quantized to that grid: a 42 ms frame
becomes 66 ms of wall time, and 24 fps is unreachable no matter how fast the
decode is.

Options, in order of preference:

1. **Drive the loop yourself and call `lv.refr_now()`** after `set_src`. This
   bypasses the timer and gives the real cost (24.8 ms flush at 320x240).
2. Let the timer coalesce, and accept ≤30 fps quantization. Fine for 8-12 fps
   clips, wrong for anything near the panel limit.
3. Raise the refresh rate globally. Not recommended: it taxes every other app
   and does not remove the flush cost.

A dedicated loop with `refr_now` measured 13.5 fps at 320x240; the same work
driven by an LVGL timer in a real player (ClipTV 0.12.3) measured 9.9 fps. The
~35 ms/frame difference is scheduling, not hardware. Pinning that player's
timer period to 66 ms or 33 ms changed nothing (9.92 / 9.79 / 9.66 fps), which
is the signature of a cost that lives outside the timer.

## 4. The scheduler tax: your loop is not the only thing running

A dedicated loop with `refr_now` reaches 13.5 fps at 320x240 (§3), yet the same
pump inside a normal app measured **8.88 fps** on the badge. The missing 4.6 fps
was not the loop: it was the framework putting a gap between passes.

MicroPythonOS runs LVGL from one repeating pass (`task_handler.tick`). After
every pass it sleeps a share of the time that pass just took:

```python
gap_ms = (last_run_ms >> 2) + tick_slack    # fairness: leave the CPU to others
```

At 320x240 a pass costs ~72 ms, so the gap is ~18 ms, plus up to a 10 ms
FreeRTOS tick (`CONFIG_FREERTOS_HZ=100`) to notice it expired. The frame is
therefore ~29 ms late every time, and the pacer — correctly — drops source
frames to stay on wall-clock time.

The drop count is predictable. With per-pass overhead `O` the steady-state
number of dropped frames per presented frame is

```
d = (O - 1.6) / 39.9        p_fps = 15 / (1 + d)
```

At `O = 2 ms` (an idle loop) `d = 0.01`, i.e. real time. At the measured
`O = 29 ms` `d = 0.69`, so 15 / 1.69 = **8.88 fps** — the measured value
exactly. The overhead, not the hardware, was halving the framerate.

**The fix is to own the passes.** On device the player disables the framework
task handler, runs `lv.task_handler()` and the frame work from its own hardware
timer, and re-enables the framework when the clip ends. No fairness gap, no
tick quantization. Three details did the rest, and each one is worth 1-3 fps:

1. **Run the pass in the timer callback, not via `micropython.schedule`.**
   Scheduling hands the work to the main thread, which only runs it when it
   next yields. Measured dispatch delay: **25-75 ms** — most of a 160x120
   frame's 33 ms budget, and it showed up as repeated one-frame drops.
2. **Do not run an LVGL pass when no frame is due.** With a fixed 5 ms tick
   and a pass on every tick, the pass cost 3-5 ms of every frame for nothing;
   during playback there is nothing else to animate. Skipping the pass on
   idle ticks dropped 320x240's LVGL term from 4.9 to 1.4-1.8 ms per frame.
3. **Present every frame that is due per callback.** A fixed tick leaves up to
   a tick of dead time between two frames that are both already late. One
   callback presenting up to four due frames removed it: 320x240 15 fps went
   from 13.3 to **13.6-14.1 fps**, against a 13.5 fps ceiling for a 72 ms
   frame.

| Run | 320x240 15 fps | Per-pass overhead |
|-----|----------------|-------------------|
| Framework passes | 8.88 fps | ~29 ms |
| Own passes, scheduled, pass every tick | 13.3 fps | ~2-8 ms |
| Own passes, direct, pass only when due, 4 frames/pass | **13.6-14.1 fps** | ~1-2 ms |

At 13.6-14.1 fps the player is at the frame-cost ceiling: 72 ms of read +
decode + flush against a 66.7 ms period. It can sit slightly *above* the naive
13.5 figure because a frame that started on time is allowed to finish, and the
33 ms refresh period lets the tail of one flush overlap the start of the next.

The 160x120 30 fps clip gains the most from the last two fixes, because 25-75 ms
of dispatch latency is several of its frames: before them it measured 14-22 fps
with a drop every other frame, after them 24.5-27.0 fps.

Two caveats for anyone copying this:

* Hand the passes back in `finally`. A pump that raises while the framework is
  disabled freezes the whole UI — no traceback, no animation, no touch. This is
  a real failure mode, not a hypothetical: it showed up once during development.
* Only do this while the player is on screen, and only on hardware where you
  can afford a spinner timer. The desktop build keeps the framework passes.
* A fixed tick means the callback sometimes has to wait before its frame is
  due. Sleeping out the wait (`time.sleep_ms(wait - tick)`) keeps the frame
  precise but blocks the `esp_timer` task — and every other `machine.Timer`
  callback — for up to a frame period. That is acceptable here because
  everything else that needs a timer is disabled for the duration, but do not
  copy the pattern into a player that has to keep other timers running: there,
  wake on the frame deadline instead.
* `disable()` stops the framework's *passes*, not LVGL's clock: `tick_inc` is
  called from the task handler's own 33 ms hardware timer, which keeps running.
  The player's pass still calls `lv.task_handler()`, so animations and
  `lv.async_call` work during playback, just at frame rate. The LVGL refresh
  timer does not cause a second flush per frame because the explicit
  `refr_now` resets it.

## 5. Dropping frames: read it or seek past it?

Drop-to-keep-time is cheap only if a dropped frame is cheap. MJPEG has no
index, so a player has two ways to advance past one:

* **Read and discard.** Costs the frame's bytes at ~0.55 MB/s. At 320x240 that
  is 23 KB ≈ 28 ms per dropped frame.
* **Seek.** `seek()` costs ~15 ms on this card, but the read that resynchronises
  on the next SOI also pulls in about one frame, so a seek costs roughly
  `15 ms + one frame read`.

Measured on 320x240, where the sink is already saturated:

| Source | Read and discard | Seek past drops |
|--------|------------------|-----------------|
| 30 fps (433 KB/s of source) | 3.86 fps | 7.6-7.7 fps |
| 60 fps (886 KB/s of source) | 0.46 fps | 7.2-7.3 fps |

The 30 fps source needs 433 KB/s of ingest to stay in real time, which is most
of the card's 550 KB/s, so reading every dropped frame leaves almost nothing for
the frames that are shown; the 60 fps source cannot be ingested at all. Seeking
is what makes those clips play at all — they still cannot run in real time, but
they move.

The same trick is poison when the sink is *not* saturated: an ungated seek
dropped a 160x120 30 fps clip from 26.7 to 17.4 fps, because the resync read
cost more than the frame it avoided. Hence two gates:

1. **Only seek when presenting alone cannot keep up** — when the mean
   *present* cost (read of the shown frame + decode + flush, excluding the time
   spent dropping) exceeds the clip's frame period.
2. **Only seek for two frames or more**, because one seek costs more than one
   frame read.

The saturation check must exclude dropped-frame time. If it includes it, the
first seek makes the next frame look slow, which justifies another seek: the
player locks itself into the slow path and stays there even though reading the
drops would have caught up. That feedback loop is why an earlier, simpler
version of this gate lost 10 fps on a clip that needed no seeking at all.

### Measured Stage 1 matrix (own passes, drop-to-keep-time, 1:1)

| Clip | Presented | Dropped / read | Read | Skip | Decode | Flush | LVGL | Idle |
|------|-----------|----------------|------|------|--------|-------|------|------|
| 320x240 15 fps | 13.6-14.1 fps | 5-6 of 150 | 25.9 | 0.5 | 13.7 | 24.4 | 1.7 | 5.0 |
| 240x180 15 fps | 14.3-14.4 fps | 1-2 of 150 | 17.4 | 0.0 | 10.0 | 12.3 | 3.3 | 25.9 |
| 160x120 15 fps | 14.4 fps | 1 of 150 | 10.8 | 0.0 | 4.7 | 7.9 | 3.6 | 41.7 |
| 160x120 30 fps | 24.5-27.0 fps | 24-47 of 300 | 12.2 | 0.4 | 4.7 | 7.7 | 2.7 | 11.7 |
| 320x240 30 fps | 7.6-7.7 fps | 13 of 91 (200 seek) | 75.5 | 5.7 | 13.8 | 24.3 | 1.8 | 9.3 |
| 320x240 60 fps | 7.2-7.3 fps | 37-40 of 112 (478 seek) | 74.3 | 14.6 | 13.8 | 24.3 | 1.7 | 8.7 |

All values are ms per presented frame, from the player's own counters, except
Presented which is derived from frames and elapsed time. Ranges are best and
worst across runs of this build: three for 320x240 15 fps and 160x120 30 fps,
two for the saturated clips.

Read the table this way:

* **Small clips are free.** 160x120 has 12 ms of idle per frame even at 30 fps;
  it is the first size that plays at (nearly) its nominal rate.
* **320x240 is ingest-bound at 15 fps and worse above it.** At 15 fps the read
  is already 26 ms of a 66.7 ms period and the panel flush is another 24.4, so
  the last 4.6 ms of the budget does not exist. At 30 fps the ingest alone
  (433 KB/s) is most of the card, which is why the dropped-frame count explodes.
* **The flush does not shrink with the source rate.** A 30 fps 320x240 clip and
  a 15 fps one cost the same 24.3 ms per presented frame; only the ingest term
  differs, and it grows. Note that a seek makes the *presented* frame's read
  expensive too: 75 ms instead of 26, because it has to scan for the next SOI.
* **The numbers are steady state.** A 60 s run of the 320x240 15 fps clip held
  14.26 fps over 900 frames (39 dropped), so the 10 s runs above are not a
  burst that decays once buffers warm up or the heap fragments.
* **160x120 at 30 fps varies between runs** (24.3-27.0 fps over six runs on the
  same build). The per-stage costs barely move, so the swing is in SD tail
  latency — single frames whose read takes 60-80 ms instead of 12. Real time is
  maintained either way; the drop count absorbs the variance.
* **The camera agrees, coarsely.** Filming the 320x240 15 fps clip at 720p30
  gave a clip advance of **15.34 fps (102% of nominal)** — the player is
  real-time — and 10.3-11.3 distinct pictures per second on screen. That last
  number is a lower bound, not a contradiction: a 30 Hz sampler watching a
  ~14 fps display misses displayed frames to aliasing, and it loses lock on
  ~5% of its own frames. Use the counters for the framerate; use the camera to
  prove there is no slow motion and no freeze.

### First-frame latency

From intent to the first presented frame: **230-550 ms**, dominated by
`SDCardManager.mount()` (160-400 ms on a cold card, 162 ms when it is already
mounted). With the card mounted, open + first read is ~11 ms. The overlays stay
visible with "loading..." until the first frame is ready, so the delay is not a
blank screen.

### Prefer a container: AVI over raw MJPEG

Raw `.mjpeg` has no index and no metadata: the frame rate can only come from
the file name, and dropping a frame means scanning for the next SOI. MJPEG in
AVI fixes both, and adds the soundtrack to the same file:

* one chunk per frame with its own size, so `skip(n)` is n header reads and
  seeks (~0.4 ms each) instead of n resync scans;
* `dwRate/dwScale` gives frames per second and `dwLength` the frame count, so
  the player no longer guesses from the name;
* an `auds` stream can carry 8/16-bit PCM or IMA ADPCM, which removes the
  sidecar `.wav` (and its same-basename convention) entirely;
* VLC plays MJPEG-in-AVI with LPCM out of the box, so one file can be checked
  on a desktop and then played on the badge unchanged.

The demuxer only walks what a player needs (the `hdrl` stream headers and the
`movi` chunk list). An unsupported video codec is reported by name rather than
played as garbage, and an unsupported soundtrack still plays the pictures: the
player shows "unsupported audio codec: MP3 (...)" and continues silently.

Measured on the badge (320x240 15 fps, `/sdcard/video`): with the sound carried
through the same chunk walk as the pictures, a frame costs **1.98-2.9 ms of
audio read** and 0.4% of the sound is dropped — against 7.5 ms and 19% when a
second cursor walked the interleave on its own. The frames are read with a
6-frame lookahead: the sound belonging to those frames is what the sink's lead
is primed from. Without a lookahead the interleave ties sound to pictures
one-to-one, so priming the sink would consume the whole queue and every slow
frame would starve the audio.

The same chunk walk also makes seeking cheap: a frame is one header plus its
payload, so a seek rewinds and steps over headers (0.3 ms each) instead of
scanning for SOI markers.

**Preloading to RAM loses on this board.** Reading the whole clip into a
`bytearray` and playing it from memory does what it says — 15.7 → 1.3 ms of read
per frame on a 1.1MB clip — but the start grows from 0.7 s to 2.3 s (the read
happens first) and the pump's own jitter rises from 7.5 to 19.4 ms/frame, so the
framerate drops from 13.9 to 11.9 fps. A 2.6MB 320x240 clip is worse still:
7.3 fps and 20% of the sound dropped. Holding megabytes on the PSRAM heap costs
more in GC and cache pressure than the SD reads it removes. The player keeps the
path (`preload` intent extra, capped at 3MB) but does not use it by default;
it would pay off on a board where the card, not the heap, is the bottleneck.

Two rules follow for anything that restarts playback (a loop, a seek):

* **Restart the sound through the normal path.** The sink still holds ~180 ms
  of the old sound, so priming on top of it mixes old and new positions.
  Closing and reopening costs the ring's drain (~370 ms) and is the same code
  path a fresh start uses; a seek then plays with 0% of the sound dropped.
* **Do it inside the pump.** The pump runs in a timer callback and is the only
  context allowed to touch the demuxer; seeking from a key handler meanwhile
  pulled the frame queue out from under it ("pop from empty list").

## 6. Audio: one clock, two streams

Audio is the second consumer of the same frame budget, and on this board the
naive ways to add it are all wrong.

**Do not use the shared player.** `AudioManager.player()` decodes in its own
reader thread, and that thread competes with the frame reader for the same SD
card. The video player therefore feeds `machine.I2S` itself, from the same
pump that presents frames.

**A separate audio thread would not help.** `machine.I2S.write()` does not
release the GIL (`ports/esp32/machine_i2s.c` has no `MP_THREAD_GIL_EXIT`), so
a thread would serialise against the pump anyway. The write has to be cheap.

### How the sink actually behaves

Measured on the badge (mono 16-bit, 22050 Hz, `ibuf` = ring size):

* A freshly opened channel queues a **full ring of silence** that drains in
  real time. Until it has drained, a write can only land as the ring empties:
  1 KB took ~22 ms, i.e. the write blocks for its own playing time.
* Once the ring has drained, a write of up to `ibuf` returns in **~90 us**.
* `ibuf` is rounded up to a power of two by the driver and must exceed
  `lead + one write`, or the top-up blocks again.

So the start sequence is: open the sink, preload the sidecar (the drain runs
during the preload), wait out the ring, then prime and present frame 0 from
the same instant. Priming before the ring drains costs 137 ms of wall time and
puts the sound 137 ms ahead of the picture.

### The clock

One byte budget drained at `byte_rate` from the prime instant:

* `lead` (180 ms) is how far ahead of the clock the sink is kept fed;
* `low` (120 ms) is the refill floor, so a top-up is at most 60 ms and always
  lands in free ring space instead of blocking;
* every byte count is rounded down to a whole sample frame. A queue that
  starts mid-frame plays left and right swapped, and a single odd byte once
  left the ring's write head one byte short of full, where a block decoder
  could never use the tail again: playback went silent and every remaining
  tick "resynced" past a clip that had stopped being read.

A refill never allocates: decoded PCM lands in one preallocated 32 KB ring,
compacted in place. Rebuilding the queue per refill cost **7 ms of PSRAM heap
churn per frame** and triggered GC inside the pump; the in-place refill costs
0.3 ms.

When the clock has passed the audio (a stalled pump), the player drops the
queued bytes and skips the source forward by the remainder, noting the skip on
the clock. Both streams therefore follow one clock and **drift is zero by
construction**; what a stall costs is the dropped sound, which the player
reports in milliseconds and percent.

### Format costs

The sidecar is 16-bit PCM or IMA ADPCM at the same rate. PCM is preloaded into
RAM when it fits (`AUDIO_PRELOAD_BYTES` = 512 KB); IMA is 4x smaller and is
decoded on the fly.

| Sidecar (10 s) | Size | CPU per frame | Note |
|----------------|------|---------------|------|
| PCM 16-bit mono 22050 | 441 KB | **0.4 ms** (0.26 read + 0.11 write) | memcpy from the preloaded buffer |
| IMA ADPCM mono 22050 | 111 KB | **2.3 ms** (2.15 decode) | 4 bits per sample in pure Python |

IMA decoding was 33.6 us per sample with the original per-sample `@native`
helpers (741 ms of CPU per second of audio, i.e. unusable). The same decoder
written as one `@micropython.viper` loop over byte-table copies of the step
index costs **1.9 us per sample** (42 ms per second of audio, 17x faster); a
320x240 15 fps frame is 2.3 ms of it. Keep decoder inner loops free of
per-sample function calls and tuple returns.

### Measured, 320x240 15 fps

| Audio | Presented | Pass | Idle | Audio/frame | Other | Sound dropped |
|-------|-----------|------|------|-------------|-------|---------------|
| none | 13.85-14.05 fps | 66.4-66.6 | 4.8-5.6 | - | 4.4-4.9 | - |
| PCM | 12.33-12.60 fps | 67.3-67.6 | 12.0-13.5 | 1.1-1.2 | 10.6-12.2 | 206-225 ms (2.1-2.3%), 1-2 resyncs |
| IMA | 11.97-12.44 fps | 68.5-70.1 | 11.9-13.5 | 2.7 | 8.6-10.3 | 151-389 ms (1.5-3.9%), 2 resyncs |

Ranges are across runs in which the clip was the first or second launch after
a reset. Launching the player three or four times in one boot makes the IMA
runs collapse (9.2 fps, 18% dropped) while PCM holds — the decode allocates a
few kilobytes per refill, and by then the heap is fragmented. Measure from a
fresh boot, as with the Stage 1 matrix.

Read this way:

* Audio costs about **1.25 fps (9%)** on a clip that is already
  saturated — but only 1.2-2.7 ms/frame is the audio code. The rest is in
  `other` (the pump not being serviced), which grows from 4.9 to 8.6-10.6 ms;
  audio changes *when* the pump is called, not how long a frame takes.
* The dropped sound is one or two stalls of 100-200 ms per 10 s clip, not a
  steady leak: a lead of 180 ms absorbs ordinary frame jitter, and the two
  resyncs are the outliers. Absorbing those too would need a 400 ms ring,
  which does not fit in the 32 KB of internal DMA RAM.
* First frame with audio is 466-780 ms against 390 ms without. Opening the
  sink as soon as the stream headers are read means the read, the buffer
  allocation and the first decode all happen inside the ring's drain instead
  of after it; opening it later cost 970 ms.

### What was and was not measured

The sound is verified end to end. With a cable from the badge's headset jack to
the host's line input, the 1 kHz sidecar beeps are recorded at full scale
(0.5 of full scale in the 1 kHz band) and land **1.0002 s apart**, drift
+0.16 ms/s across five of them: the sink plays the file at its own rate, without
gaps and without resampling. An earlier estimate of the sample rate from DMA
drain timing ("2.6-3.1% fast") was an artefact of that method, not a property of
the hardware — the recorded beeps are the better instrument.

The **A/V offset itself is still not measured**, and one webcam + line-in
capture cannot measure it as it stands:

* the two capture devices run in different clock domains — here v4l2 reports
  `CLOCK_MONOTONIC` (uptime, 323881 s) while ALSA reports the wall clock
  (1794307262 s), and ffmpeg rebases each stream to zero, which throws their
  relative alignment away; `-copyts` keeps the alignment but the absolute
  values overflow Matroska;
* the UVC stream arrives with seconds of buffered frames, so its first frame is
  not the first frame captured.

A rig that can measure it needs one marker present in both streams: a clap
board, or an LED driven by the same event as the beep. The synthetic clips
already flash and beep on the same instants for exactly this purpose.

What bounds the offset today is the code path, not a measurement: frame 0 is
flushed and sample 0 is primed from the same instant (the prime is what starts
the sink, and it is issued right after the flush), the flush itself costs
~24 ms, and the sink's DMA starts within a few milliseconds. So the two should
be inside the ~80 ms at which an offset becomes visible — but that is an
argument, and it is worth confirming on a bench with audio wired.

## 7. The isolating matrix

To attribute time, run the same loop six ways: source (SD vs RAM) x display
(every frame, 1 in 10, never). Only the source and the flush change.

320x240, ms per frame:

| Run | fps | frame | read | decode | draw |
|-----|-----|-------|------|--------|------|
| A full pipeline | 13.49 | 74.11 | 33.46 | 13.84 | 24.78 |
| B SD + decode | 18.95 | 52.77 | 33.58 | 13.98 | 2.25 |
| C decode + flush | 23.73 | 42.14 | 0.16 | 14.94 | 24.76 |
| D decode + 10% flush | 53.50 | 18.69 | 0.26 | 13.67 | 2.43 |
| E SD, never draw | 19.83 | 50.42 | 34.88 | 13.81 | — |
| F decode only | 62.69 | 15.95 | 1.53 | 13.66 | — |

Read the table like this:

* A ≈ E + draw, C ≈ F + draw: **flush and ingest are additive**.
* E ≈ F + read: **ingest is additive**.
* Draw 1 frame in 10 adds ~1-2 ms/frame on average: **skipping the flush is
  cheap**. Skipping the *read* is not — see §5, where a dropped frame costs its
  full ingest time. Drop-to-keep-time should still default ON, but the price is
  the wasted read.
* `other` (loop overhead, GC, scheduling) is 1-3 ms in a dedicated loop. If you
  see more, your architecture — not the hardware — is the problem.

The same matrix at 240x180 (A 20.4 fps) and 160x120 (A 33.2 fps) scales as the
byte and pixel models predict.

## 8. Scaling is the most expensive mistake

`image.set_scale()` resamples on every draw. A 160x120 frame scaled 2x to fill
320x240 costs **86.9 ms/frame** versus 8.9 ms at 1:1 — roughly a 10x penalty,
and enough on its own to drop a clip below real time.

Consequences:

* Play clips 1:1. Center them if they are smaller, letterbox if needed.
* `set_size()` + `set_scale()` are acceptable once at clip load.
* If an app must fill the panel from a small clip, decode to the target size
  offline or pre-generate the larger asset; never scale per frame.
* Rotation is the same trap: a permanent `set_rotation()` puts every frame on
  the per-pixel transform path. Ship pre-rotated content.

## 9. What this means for a player design

Derived from the model, not from taste:

1. **One pump loop**, not LVGL timers: `read frame → decode → set_src →
   refr_now → pace`. Get the loop cost under 3 ms, and own the LVGL passes:
   a framework pass costs a quarter of its duration again in fairness gap
   (§4).
2. **Pace against a wall clock**, not against "how long the last frame took".
   Drift accumulates otherwise. A one-frame clock (`next_deadline += 1000/fps`)
   with drop-when-behind is enough.
3. **Drop to keep time.** If `now > deadline + frame_interval`, skip to the
   next due frame. Cost of dropping is a frame read (~28 ms at 320x240), not
   1 ms, because MJPEG has no index; seek past the drops only when presenting
   alone cannot keep up (§5). Cost of not dropping is a permanently
   slower-than-real-time clip.
4. **Reuse one decode buffer.** `jpegdec.decode(src, rgb)` writes into a fixed
   `bytearray(w*h*2)`; point the `image_dsc_t` at it once. Do not reallocate per
   frame — that is GC pressure and fragmentation.
5. **Read exactly one frame per iteration.** Scan for SOI/EOI in a small
   sliding buffer. A reader that scans a 256 KB buffer per frame measured
   204 ms/frame; the same SD card with a 64 KB window measured 32 ms. The
   buffer size, not the card, was the bug. In AVI (see below) every chunk
   carries its length, so a frame is one header read plus its payload and a
   drop is a seek past the header — strictly cheaper than scanning.
6. **Never call `set_src` for an identical frame** and skip redraws you do not
   need; invalidation is cheap but the flush is not.
7. **Feed audio from the pump, on one clock.** Audio must not block the pump:
   refill from a preallocated ring and only when the sink has free space, so a
   write costs microseconds instead of its playing time. A separate thread buys
   nothing because `I2S.write()` holds the GIL (§6).
8. **Do not preload by default.** It looks free on paper (a 160x120 clip is
   ~0.9 MB) but on this board the read is replaced by GC and cache pressure and
   the framerate *drops* (§6). Stream, and use the card's sequential bandwidth.

## 10. Would a network player be faster than SD?

In principle, yes — but bounded, and in this environment it was not.

The network only replaces the **ingest** term. Decode (13.8 ms) and flush
(24.8 ms) are unchanged, so the hard ceiling becomes 23.7 fps at 320x240
(row C: decode + flush with no storage at all). SD costs 33.5 ms on top of
that, which is why the full pipeline is 13.5 fps. So the *maximum* win from
streaming is 13.5 → 23.7 fps at 320x240, and only if the network is faster
than the card.

Measured here (badge → Fritz!Box AP, RSSI -36 to -46 dBm, 2.4 GHz, HTTP from a
wired host):

| Metric | Result |
|--------|--------|
| Bulk ingest, 16 KB reads | 0.22-0.35 MB/s (1.9-3.0 Mbit/s) |
| Per-read latency | ~11 ms mean, p95 15-20 ms |
| Tail | occasional 1.5 s stalls, even with power save off (`pm=0`) |
| Frame-sized reads (15.8 KB) | 9.3-12.3 frames/s (0.14-0.18 MB/s) |
| Time to first byte | 5-16 ms |

That is **not better than the SD card** (0.6 MB/s, no multi-second stalls).
The link rate is the limit, not the chip's theoretical WiFi capability; a
better AP was not measured, so treat this as "a normal 2.4 GHz router at close
range" rather than a ceiling.

Bandwidth needed is small: 15.8 KB x 15 fps = **0.24 MB/s** for 320x240 at
15 fps. The measured average is just above that, but the p95 latency and the
1.5 s stalls mean an unbuffered player would stutter. Network playback needs a
read-ahead buffer of a second or more, which competes with the frame cache for
the same MicroPython heap.

Practical conclusions:

* Streaming cannot beat decode+flush, so it cannot exceed ~23.7 fps at 320x240
  on this hardware.
* On a slow or jittery link it is worse than the card, because tail latency is
  unbounded while SD latency is stable.
* If you build it anyway: buffer ahead, use plain HTTP on a LAN (TLS adds CPU
  and handshake latency), and keep the buffer in PSRAM-backed allocations.
* The win, when it exists, is at small sizes: 160x120 needs only 0.1 MB/s, so
  the network stops being the bottleneck and the 33.2 fps ceiling is reachable.

## 11. How these numbers were obtained

* **Camera**: `video_samples/tools/capture_measure.py` records the badge with a
  webcam and decodes a binary frame-index strip from the screen. It reports
  `presented_fps` (distinct pictures actually seen) separately from
  `clip_rate_fps` (how fast the clip timeline advanced), so "the player is
  dropping frames to keep time" is visible rather than misread as a slow clip.
* **Counters**: per-stage `time.ticks_us()` around read, decode, `set_src`, and
  `refr_now`, plus the LVGL pass, aggregated over a run. Stage 1 numbers come
  from the player in `internal_filesystem/apps/com.micropythonos.video_player`,
  which prints one line per clip of exactly these values.
* **Matrix**: `video_samples/tools/probe_matrix.py`, run from a fresh boot
  (heap fragmentation after a long uptime makes 1 MB allocations fail).
* Always measure the display with `lv.refr_now()`: the 33 ms refresh timer
  otherwise flattens everything to ~30 fps and hides the real cost.

Camera caveat: 720p30 capture is exact for display rates up to ~15 fps. Above
that, use player counters; a 60 fps capture mode is not deliverable over USB 2.

## 12. Quick reference

| Question | Answer |
|----------|--------|
| Why is my 320x240 clip stuck at 10 fps? | Two causes: a pass cost over 70 ms (hardware limit), or the scheduler in §4 adding ~29 ms per pass. Measure both. |
| Will a faster timer help? | No. The cost is outside the timer; pinning it changes nothing. |
| Does the shared SPI bus add overhead? | No. Costs are additive: ingest + decode + flush. |
| Can I play 30 fps at 320x240? | Not with MJPEG from SD: 13.5 fps full pipeline, 23.7 fps with the source in RAM. A 30 fps 320x240 source plays at 7-8 fps. |
| What does a drop cost? | A frame read: ~28 ms at 320x240, ~11 ms at 160x120. Seeking past drops costs ~15 ms + a resync read, so only do it when the sink is saturated. |
| My loop measures 72 ms/frame but the player shows 9 fps. Why? | The framework's fairness gap: a pass sleeps a quarter of its own duration before the next one. Own the passes. |
| Why did 160x120 get slower when I scaled it up? | LVGL scaling: 86.9 ms vs 8.9 ms at 1:1. Never scale in the draw path. |
| Is WiFi streaming the answer? | Only bounded: it removes the 33 ms ingest term, so 23.7 fps max; measured here it was slower than SD. |
| How much does audio cost? | 1.2-2.7 ms/frame for the audio code, ~1.25 fps overall at 320x240 15 fps. Use PCM (0.4 ms) over IMA (2.3 ms) when RAM allows. |
| My audio starts late, or the first write takes 137 ms. | The I2S ring is pre-filled with silence that must drain first. Open the sink, wait out `ibuf`, then prime; priming early adds the drain to the picture's lead. |
| Should I ship `.mjpeg` or `.avi`? | AVI: the container carries the frame rate and the sound, dropping a frame is a chunk-header seek instead of an SOI scan, and VLC opens it unchanged. `.mjpeg` + WAV sidecar remains the fixture for the low-level tests. |
| Audio goes silent part-way through the clip. | A block decoder cannot use a short tail in the ring: if the read cursor parks at the write head, every refill returns 0. Empty the ring when it is drained to the end, and compact when a refill does not fit. |
| Why does audio drop 200 ms at a time? | The pump stalled longer than the lead, and the resync skipped the sound forward to keep it with the pictures. Look at `other` (timer latency) in the report, not at the audio code. |
