# Fosi Audio DS2 USB DAC Bit-Perfect Playback — Implementation Documentation

## Device

**Fosi Audio DS2** — USB DAC using a Savitech USB bridge chip.

- VID: 9770 (0x262A, Savitech)
- PID: 1 (0x0001)
- Product name: "Fosi Audio DS2"

## Problem

The Fosi DS2's Savitech USB bridge has broken UAC2 clock control:

- **GET_RANGE** returns I/O error or zero subranges — the DAC advertises no sample rate ranges.
- **GET_CUR** always returns 0 Hz, even after a successful SET_CUR.
- **SET_CUR** is accepted silently and the DAC operates correctly at the requested rate, but never reports it back.

The 0 Hz readback poisoned every downstream code path, causing the engine to fall back to ExoPlayer (AudioFlinger) at 48 kHz instead of playing bit-perfect via the USB direct (libusb) path. This affected all sample rates, not just 44.1 kHz.

### Symptoms

- Playing a 44.1 kHz file as the first track after DAC connect: fell back to 48 kHz ExoPlayer, no audio heard (wrong output rate).
- Playing a 48 kHz file first, then a 44.1 kHz file: worked correctly (the engine was already initialized at a valid rate and could re-negotiate).
- `Infinity or NaN toInt` crash in the Dart UI when sample rate was 0.
- "Rust audio engine is not initialized" crash in some configurations.
- "No isochronous OUT endpoint can carry 44100 hz" fallback error.

## Root Cause Analysis

The root cause was identified through multiple iterations of debugging, using Rust engine logs captured from the device.

### Chain of failure

1. `negotiate_android_direct_output_sample_rate()` called `negotiate_android_direct_playback_format()`, which ran SET_CUR on the DAC. The DAC accepted SET_CUR but GET_CUR returned 0 Hz.

2. `apply_sampling_frequency()` reported 0 Hz as the `reported_sample_rate` from GET_CUR readback. This 0 propagated into `effective_format.sample_rate`.

3. `set_effective_playback_format()` stored 0 Hz in `DIRECT_USB_STATE.playback_format`.

4. `negotiate_android_direct_playback_format()` returned `Ok(effective_format)` with `sample_rate = 0`. The `.map(|format| Some(format.sample_rate))` in `negotiate_android_direct_output_sample_rate()` returned `Some(0)` to callers — even though `set_effective_playback_format` had internally corrected the stored rate. **This was the critical bug: the function returned the pre-correction value, not the actual stored rate.**

5. `audio_prepare_engine(None)` resolved to `Some(0)` via `negotiate(None)`. `create_audio_engine` received `Some(0)`, filtered it via `unwrap_or(48_000)`, and created the engine at 48000 Hz.

6. `audio_play(44100)` called `resolve_track_playback_output_sample_rate(Some(44100))`. However, `should_preserve_existing_rate` (a DAP optimization) returned the existing engine's 48000 Hz instead of calling `negotiate(44100)`, because USB DACs were not excluded from this check.

7. `validate_android_direct_request` rejected the mismatch between stored 0 Hz and preferred rate.

8. `android_direct_output_signature` returned `None` (preferred != stored rate), causing the engine to fall back to the android-shared (AudioFlinger) path.

9. `source.position_secs()` divided by zero sample rate, producing Infinity/NaN that propagated to the Dart UI.

### Why 48 kHz files worked

48 kHz files worked because the default fallback rate was 48000 Hz. When `negotiate(None)` returned `Some(0)` and `create_audio_engine` filtered it to 48000, the engine happened to be at the correct rate for 48 kHz files. The USB direct backend would then open at 48000 Hz (matching the effective format), and playback succeeded.

### Why 44.1 kHz worked after 48 kHz

After a successful 48 kHz playback, the USB session was active and the engine state was valid. When a 44.1 kHz track loaded, `audio_play` called `resolve_track_playback_output_sample_rate(Some(44100))`. With a valid existing engine, the rate change path worked correctly — `negotiate(Some(44100))` updated the formats, `ensure_rust_engine_async` detected the signature mismatch, and the engine was recreated at 44100.

## Solution

### Overview

The fix adds a device quirk for the Fosi DS2 and addresses the 0 Hz readback at multiple levels — from the source (GET_CUR readback) through the negotiation chain to downstream consumers.

### Changes by file

---

#### 1. `rust/src/uac2/quirk.rs` (+22 lines)

Added a Fosi DS2 entry to the `QUIRK_DATABASE`:

- VID 9770, PID 1, empty product name (matches any name with this VID/PID)
- **`SkipClockValidation`** — bypasses all clock verification gates: post-alt GET_CUR re-verification, clock verification refusal, output signature mismatch, validate_android_direct_request rate check.
- **`IgnoreInvalidSampleRate`** — tolerates reported rate mismatch (DAC reports 0 Hz when 44100 Hz expected).

Added unit test `fosi_ds2_quirks_registered` verifying both quirks are present for VID 9770/PID 1.

---

#### 2. `rust/src/uac2/mod.rs` (+1/-1 lines)

Added `android_direct_usb_enabled` to the `pub use android_direct::{...}` re-export list, making it accessible from `audio_api.rs`.

Made `android_direct_usb_enabled()` `pub` in `android_direct.rs` (was `fn`, now `pub fn`).

---

#### 3. `rust/src/uac2/android_direct.rs` (+276/-91 lines)

This file contains the majority of the changes:

**`apply_sampling_frequency()` — 0 Hz readback replacement**

When GET_CUR returns 0 Hz after a successful SET_CUR, replace the reported rate with the requested rate. This is the fix at the source — the DAC operates correctly at the requested rate, it just doesn't report it.

```rust
if reported_sample_rate == 0 {
    reported_sample_rate = sample_rate;
}
```

**`set_effective_playback_format()` — 0 Hz storage refusal**

Changed signature from `fn set_effective_playback_format(playback_format: AndroidDirectUsbPlaybackFormat)` to take `mut playback_format`. If `sample_rate == 0`, recovers the rate from `requested_playback_format` before storing. Prevents 0 from poisoning `DIRECT_USB_STATE.playback_format` and all downstream consumers.

**`negotiate_android_direct_output_sample_rate()` — readback fix (THE critical fix)**

Changed the return mapping from:
```rust
negotiate_android_direct_playback_format(requested_format)
    .map(|format| Some(format.sample_rate))
```
to:
```rust
let negotiated = negotiate_android_direct_playback_format(requested_format);
negotiated.map(|format| {
    let actual_rate = DIRECT_USB_STATE
        .lock()
        .as_ref()
        .and_then(|state| state.playback_format.map(|f| f.sample_rate))
        .filter(|&rate| rate > 0)
        .unwrap_or(format.sample_rate);
    Some(actual_rate)
})
```

This reads back the corrected rate from `DIRECT_USB_STATE` (which `set_effective_playback_format` updated) instead of returning the pre-correction `format.sample_rate`. This ensures callers receive the actual effective rate (e.g. 44100), not 0.

**`android_direct_output_signature()` — quirk bypass**

When `preferred_sample_rate != Some(playback_format.sample_rate)`, checks the `SkipClockValidation` quirk. If active, returns a signature with the preferred rate instead of `None`. Without this, `None` causes the engine to fall back to the android-shared (AudioFlinger) path.

**`validate_android_direct_request()` — 0 Hz + quirk bypass**

Two additions:
1. If `playback_format.sample_rate == 0`, passes validation (broken DAC readback).
2. If `SkipClockValidation` quirk is active, accepts any stored rate (the stored format is unreliable for quirked devices).

**Post-alt GET_CUR re-verification — skip for quirked devices**

When `SkipClockValidation` is active, skips the post-alt-setting GET_CUR re-verification entirely. The DAC reports 0 Hz, so re-verification always fails — skipping avoids an unnecessary I/O error.

**Clock verification refusal — skip for quirked devices**

When clock verification fails and `SkipClockValidation` is active, trusts SET_CUR and continues. Sets `clock_verification_passed = true` and `reported_sample_rate = Some(playback_format.sample_rate)`.

**Rate mismatch refusal — tolerate for IgnoreInvalidSampleRate**

When reported rate != requested rate and `IgnoreInvalidSampleRate` is active, continues instead of refusing. The DAC reports 0 Hz when operating at the requested rate.

**`create_android_usb_backend_inner()` — 0 Hz guard**

If `preferred_sample_rate` arrives as 0, recovers from `requested_playback_format` or `playback_format`, falling back to 44100 if both are 0.

---

#### 4. `rust/src/api/audio_api.rs` (+4/-4 lines)

**`resolve_track_playback_output_sample_rate()` — USB DAC exclusion**

Added `&& !usb_direct_active` to the `should_preserve_existing_rate` condition. This DAP optimization (keeps the engine at its current rate for DAPs with bit-perfect disabled) was incorrectly returning the existing engine's 48000 Hz for USB DACs instead of calling `negotiate(44100)`.

```rust
#[cfg(feature = "uac2")]
let usb_direct_active = crate::uac2::android_direct_usb_enabled();
#[cfg(not(feature = "uac2"))]
let usb_direct_active = false;
```

---

#### 5. `rust/src/audio/engine.rs` (+4/-1 lines)

**`create_audio_engine()` — preferred_sample_rate 0 filter**

```rust
preferred_sample_rate.filter(|&rate| rate > 0).unwrap_or(48_000)
```

If `Some(0)` arrives (from negotiate before the readback fix, or any other path), filters it out and falls back to 48000.

**Clock rate resolution chain — 0 filters**

Each step in the `.or()` chain (clock_reported, playback_format, requested) is filtered with `.filter(|&rate| rate > 0)` so 0 values are skipped.

---

#### 6. `rust/src/audio/source.rs` (+5/-1 lines)

**`position_secs()` — division by zero guard**

```rust
let rate = self.info.output_sample_rate;
if rate == 0 {
    return 0.0;
}
frames as f64 / rate as f64
```

---

#### 7. `lib/services/rust_audio_service.dart` (+9/-2 lines)

**`_updateProgress()` — Infinity/NaN guard**

```dart
final positionSecs = progress.positionSecs;
if (!positionSecs.isFinite || positionSecs < 0) {
    _updateState();
    return;
}
```

And for duration:
```dart
if (progress.durationSecs != null &&
    progress.durationSecs!.isFinite &&
    progress.durationSecs! >= 0) {
```

---

#### 8. `lib/services/player_service.dart` (+5/-1 lines)

**`_ensureRustEngineInitialized()` — preferredSampleRate override removal**

Removed `preferredSampleRate = directUsbFormat?.sampleRate;` for USB DAC experimental mode. The stored DAC format may be stale (hardcoded 48000 from `_ensureUsbDacRegistered`) or report 0 Hz. Passing `null` lets the Rust engine use the track's actual sample rate via `requested_playback_format`.

---

## Debugging process

### Iteration 1: Quirk entry + basic bypasses

Added Fosi DS2 to quirk database. Added bypasses in `validate_android_direct_request` and `android_direct_output_signature` for `SkipClockValidation` devices. Built and tested — 0 Hz still propagating, `Infinity or NaN toInt` crash.

### Iteration 2: 8-layer defense-in-depth

Applied 0 Hz guards at 8 layers: `apply_sampling_frequency`, `set_effective_playback_format`, `source.position_secs`, `rust_audio_service._updateProgress`, `create_android_usb_backend_inner` entry guard, format selection fallback, engine `preferred_sample_rate` filter, and PRE-STREAM OVERRIDE. Built and tested — no longer falling back, but output rate was 48 kHz instead of 44.1 kHz (no audio heard).

### Iteration 3: Dart-side preferredSampleRate

Removed `preferredSampleRate = directUsbFormat?.sampleRate` override in Dart. Built and tested — same issue (stale Dart kernel cached, needed `flutter clean`).

### Iteration 4: should_preserve_existing_rate exclusion

Discovered that `should_preserve_existing_rate` (DAP optimization) was returning 48000 Hz for USB DACs, bypassing `negotiate(44100)` entirely. Added `&& !usb_direct_active` exclusion. Also added format mismatch quirk fallback in `create_android_usb_backend_inner`. Built and tested — USB backend opened at 44100 Hz but stream died after 15ms, fell back to ExoPlayer.

### Iteration 5: Debug logging

Added `[RATE]` debug logging (using `dev_eprintln!` to match the codebase convention) in `resolve_track_playback_output_sample_rate`, `audio_prepare_engine`, and `audio_play`. After several attempts with wrong logging macros (`eprintln!`, `log_warn!`), got the logs visible in logcat.

### Iteration 6: Root cause identified

Debug logs revealed:
- `audio_prepare_engine: preferred=None resolved=Some(0)` — negotiate returned 0
- `audio_play: track_rate=Some(44100) resolved_rate=Some(0)` — negotiate returned 0 again
- `should_preserve_existing_rate: preserve=false` — the exclusion fix was working, but negotiate itself was returning 0

Root cause: `negotiate_android_direct_output_sample_rate` returned `Some(format.sample_rate)` where `format` still had 0 Hz from the GET_CUR readback, even though `set_effective_playback_format` had internally corrected the stored rate to 44100. The function returned the pre-correction value.

### Iteration 7: Negotiate readback fix (THE fix)

Changed the return mapping in `negotiate_android_direct_output_sample_rate` to read back the corrected rate from `DIRECT_USB_STATE.playback_format` instead of using the pre-correction `format.sample_rate`. Built and tested — **everything works**.

### Iteration 8: Cleanup

Removed redundant defense-in-depth layers that were no longer needed after the root cause fix:
- Format mismatch quirk fallback in `create_android_usb_backend_inner`
- PRE-STREAM OVERRIDE
- All `[RATE]` debug logging

Verified: cargo check clean, 4/4 quirk tests pass, APK builds, all sample rates play correctly.

## Testing

Verified on real Fosi DS2 hardware:

- 44.1 kHz file as first track after DAC connect: plays at 44100 Hz bit-perfect (USB direct, no ExoPlayer fallback).
- 48 kHz file as first track: plays at 48000 Hz bit-perfect.
- 44.1 kHz after 48 kHz: plays correctly (engine recreates at correct rate).
- No `Infinity or NaN toInt` crashes.
- 4/4 quirk unit tests pass.
- `cargo check` clean (0 errors, 13 pre-existing warnings).

## Files changed

| File | Lines |
|------|-------|
| `rust/src/uac2/quirk.rs` | +22 |
| `rust/src/uac2/mod.rs` | +1/-1 |
| `rust/src/uac2/android_direct.rs` | +276/-91 |
| `rust/src/api/audio_api.rs` | +4/-4 |
| `rust/src/audio/engine.rs` | +4/-1 |
| `rust/src/audio/source.rs` | +5/-1 |
| `lib/services/rust_audio_service.dart` | +9/-2 |
| `lib/services/player_service.dart` | +5/-1 |
| `android/.../MusicNotificationService.kt` | +35 |
| **Total** | **+370/-98** |

## Architecture note

The Fosi DS2 uses the USB direct (libusb) path, which bypasses AudioFlinger entirely — the same approach UAPP (USB Audio Player PRO) uses for USB DACs. The quirk fix is correct for this device class. The 0 Hz readback is a USB protocol issue (broken Savitech bridge), not an AudioFlinger issue.

For internal DACs (MixerBitPerfect path), a quirk cannot fix missing sample rate information because AudioFlinger owns the device. Extending the existing DSD ALSA-direct module to PCM would be the approach for that case — a separate, future improvement.

## Post-fix: screen-off audio stop on track change

### Symptom

After the quirk fix, the first track played fine with the screen off, but when the second track started (after the first one ended naturally), audio would stop ~2 seconds in. The app still showed "playing" state. Turning the screen back on resumed audio immediately.

### Root cause

The quirk changes caused the Rust audio engine to be **recreated on every track change** — the `!usb_direct_active` exclusion in `should_preserve_existing_rate` (audio_api.rs) and the `SkipClockValidation` quirk in `android_direct_output_signature` (android_direct.rs) returning a rate-specific signature instead of `None`. Before the quirk fix, the engine was reused across tracks and the USB isochronous stream never stopped.

When the engine is recreated, the old USB session is torn down and a new one starts with fresh SCHED_FIFO isochronous transfer threads. These threads need the CPU to stay awake to meet their 1ms real-time deadlines. But the app never acquired a `PARTIAL_WAKE_LOCK` — the `WAKE_LOCK` permission was in the manifest but unused. ExoPlayer/just_audio acquire their own wakelocks internally; the custom Rust USB direct path does not.

The first track survived screen-off because the stream was already stable before the screen turned off. The second track died because its freshly-created USB session had no wakelock, and Doze throttled the CPU enough to starve the isochronous transfers.

### Fix

Added a `PowerManager.WakeLock` (PARTIAL_WAKE_LOCK, non-reference-counted) to `MusicNotificationService`:

- `updateWakeLock()` is called on every `onStartCommand` — acquires when `isPlaying` is true, releases when false.
- Released in `onDestroy` and `shutdownForTaskRemoval()`.
