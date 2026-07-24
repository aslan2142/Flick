# Fosi Audio DS2 USB DAC Support — Change Overview

## Device

**Fosi Audio DS2** — USB DAC using a Savitech USB bridge chip.

- VID: 9770 (0x262A)
- PID: 1 (0x0001)
- Product name: "Fosi Audio DS2"

## Problem

The Fosi DS2's Savitech bridge has broken UAC2 clock control:

- **GET_RANGE** returns I/O error or zero subranges — the DAC advertises no sample rate ranges at all.
- **GET_CUR** always returns 0 Hz, even after a successful SET_CUR.
- **SET_CUR** is accepted silently and the DAC operates correctly at the requested rate, but never reports it back.

The 0 Hz readback poisoned every downstream code path:

1. `negotiate_android_direct_output_sample_rate()` returned `Some(0)` to callers.
2. `audio_prepare_engine(None)` resolved to `Some(0)`, which `create_audio_engine` filtered to 48000 Hz (the `unwrap_or` fallback).
3. `audio_play(44100)` called `resolve_track_playback_output_sample_rate(Some(44100))`, but `should_preserve_existing_rate` (DAP optimization) returned the existing engine's 48000 Hz instead of the track's 44100 Hz.
4. `set_effective_playback_format` stored 0 Hz in `DIRECT_USB_STATE.playback_format`.
5. `validate_android_direct_request` rejected 0 Hz vs preferred rate.
6. `android_direct_output_signature` returned `None` (preferred != stored), causing the engine to fall back to the android-shared (AudioFlinger) path.
7. `source.position_secs()` divided by zero sample rate → Infinity/NaN propagated to the Dart UI.
8. `create_audio_engine` clock rate resolution chain included 0 Hz values.

Net effect: any sample rate (44.1 kHz, 48 kHz, etc.) fell back to ExoPlayer at 48 kHz via AudioFlinger — no bit-perfect playback.

## Root Cause

Two root causes:

1. **`apply_sampling_frequency`** reported 0 Hz readback from GET_CUR, which propagated through the entire chain unchecked.
2. **`negotiate_android_direct_output_sample_rate`** returned `Some(format.sample_rate)` where `format` still had 0 Hz — even though `set_effective_playback_format` had corrected the stored rate. The function returned the pre-correction value, not the actual stored rate.

## Solution

### 1. Quirk database entry (`rust/src/uac2/quirk.rs`)

Added a Fosi DS2 entry (VID 9770, PID 1) with two quirks:

- **`SkipClockValidation`** — bypasses all clock verification gates (post-alt GET_CUR re-verification, clock verification refusal, output signature mismatch, validate_android_direct_request rate check).
- **`IgnoreInvalidSampleRate`** — tolerates reported rate mismatch (DAC reports 0 Hz when 44100 Hz expected).

Includes a unit test (`fosi_ds2_quirks_registered`) verifying both quirks are present.

### 2. 0 Hz readback replacement (`apply_sampling_frequency` in `android_direct.rs`)

When GET_CUR returns 0 Hz after SET_CUR, replace the reported rate with the requested rate. This is the fix at the source — the DAC operates correctly at the requested rate, it just doesn't report it.

### 3. 0 Hz storage refusal (`set_effective_playback_format` in `android_direct.rs`)

`set_effective_playback_format` now refuses to store 0 Hz. If the format has 0 Hz, it recovers the rate from `requested_playback_format`. This prevents 0 from poisoning `DIRECT_USB_STATE.playback_format` and all downstream consumers.

### 4. Negotiate readback fix (`negotiate_android_direct_output_sample_rate` in `android_direct.rs`) — THE critical fix

After `negotiate_android_direct_playback_format` runs, the return value is mapped through a readback: the function re-reads `DIRECT_USB_STATE.playback_format` (which was corrected by `set_effective_playback_format`) and returns that rate instead of the pre-correction `format.sample_rate`. This ensures callers like `audio_prepare_engine` and `audio_play` receive the actual effective rate (e.g. 44100), not 0.

### 5. USB DAC exclusion from DAP rate preservation (`resolve_track_playback_output_sample_rate` in `audio_api.rs`)

The `should_preserve_existing_rate` optimization (keeps the engine at its current rate for DAPs with bit-perfect disabled) now excludes USB DACs via `&& !android_direct_usb_enabled()`. Without this, `audio_play(44100)` would return the existing engine's 48000 Hz instead of calling `negotiate(44100)`, bypassing the rate change entirely.

### 6. Engine 0 Hz guards (`rust/src/audio/engine.rs`)

- `preferred_sample_rate.filter(|&rate| rate > 0).unwrap_or(48_000)` — if `Some(0)` arrives (from negotiate before the readback fix was applied, or from any other path), filter it out and fall back to 48000 instead of using 0.
- Clock rate resolution `.or()` chain: each step (clock_reported, playback_format, requested) is filtered with `.filter(|&rate| rate > 0)` so 0 values are skipped.

### 7. Position division guard (`rust/src/audio/source.rs`)

`position_secs()` now guards `output_sample_rate == 0` and returns 0.0 instead of dividing by zero.

### 8. Dart Infinity/NaN guard (`lib/services/rust_audio_service.dart`)

`_updateProgress()` checks `positionSecs.isFinite && positionSecs >= 0` and `durationSecs.isFinite && durationSecs >= 0` before converting to milliseconds. Prevents the `Infinity or NaN toInt` crash.

### 9. Dart preferredSampleRate fix (`lib/services/player_service.dart`)

In `_ensureRustEngineInitialized`, the `preferredSampleRate = directUsbFormat?.sampleRate` override was removed for USB DAC experimental mode. The stored DAC format may be stale (hardcoded 48000 from `_ensureUsbDacRegistered`) or report 0 Hz. Passing `null` lets the Rust engine use the track's actual sample rate via `requested_playback_format`.

### 10. Module re-export (`rust/src/uac2/mod.rs`)

`android_direct_usb_enabled` made `pub` and added to the `pub use` re-export list so `audio_api.rs` can call `crate::uac2::android_direct_usb_enabled()`.

### 11. Bypass gates (all in `android_direct.rs`)

For `SkipClockValidation` quirked devices:

- **`android_direct_output_signature`** — when preferred != stored rate, returns a signature with the preferred rate instead of `None` (which would fall back to android-shared).
- **`validate_android_direct_request`** — accepts any stored rate (not just 0) when the quirk is active, since the stored format is unreliable.
- **Post-alt GET_CUR re-verification** — skipped entirely (avoids I/O error from broken GET_CUR).
- **Clock verification refusal** — when verification fails, the quirk trusts SET_CUR and continues.
- **Rate mismatch refusal** — when reported rate != requested rate, `IgnoreInvalidSampleRate` allows continuing.
- **`create_android_usb_backend_inner` 0 Hz guard** — if `preferred_sample_rate` arrives as 0, recovers from `requested_playback_format` or `playback_format`, falling back to 44100 if both are 0.

## What was removed (redundant layers)

During development, several defense-in-depth layers were added that turned out to be unnecessary once the root cause (negotiate readback) was fixed:

- **Format mismatch quirk fallback** in `create_android_usb_backend_inner` — when both effective and requested formats had valid non-zero rates but neither matched `preferred_sample_rate`, a `SkipClockValidation` branch accepted the effective format. With the negotiate readback fix, `preferred` now matches `effective`, so this branch never fires.
- **PRE-STREAM OVERRIDE** — last-resort brute-force of 0 Hz to `preferred_sample_rate`. Upstream guards (`set_effective_playback_format` + negotiate readback) prevent 0 from reaching this point.
- **Debug `[RATE]` logging** — temporary `dev_eprintln!` calls in `audio_api.rs` used during diagnosis.

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
| **Total** | **+335/-98** |

## Testing

Verified on real Fosi DS2 hardware:

- 44.1 kHz file as first track after DAC connect: plays at 44100 Hz bit-perfect (USB direct, no ExoPlayer fallback).
- 48 kHz file as first track: plays at 48000 Hz bit-perfect.
- 44.1 kHz after 48 kHz: plays correctly (engine recreates at correct rate).
- No `Infinity or NaN toInt` crashes.
- 4/4 quirk unit tests pass.
- `cargo check` clean (0 errors, 13 pre-existing warnings).
