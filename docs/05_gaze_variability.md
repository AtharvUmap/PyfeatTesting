# 05 — Gaze Variability

## What this metric measures

How much and how erratically the subject's eyes move. Related trust findings: high, erratic gaze variability (fidgety scanning) is often associated with lower perceived trustworthiness; steady or rhythmically-shifting gaze reads as calmer and more engaged.

The notebook reports three complementary views:

| Measure | Question it answers |
|---|---|
| Iris-offset std (global and rolling) | How much total spread was there in gaze position? |
| Fixation count + mean duration | How often and how long did the subject hold their gaze steady? |
| Saccade rate | How frequently did the subject make fast gaze shifts? |

## Temporal-resolution caveat (important)

Real saccades last **20–100 ms**. At our default pipeline sampling of ~6 effective fps (one frame every ~167 ms), we **cannot resolve individual saccades**. A single-frame "saccade" in our data conflates a real saccade with fixation time around it. The fixation/saccade numbers here are **coarse activity indicators** — useful for comparing videos processed with identical settings, but not precise oculomotor measurements.

For rigorous saccade analysis you need one of:
- A high-frame-rate recording (60+ fps) and a lower `SKIP_FRAMES` in the pipeline.
- A dedicated eye tracker (Tobii, Pupil Labs) sampling at 60–1200 Hz.

## Iris-offset formula (shared with Ticket 3)

For each eye, normalize iris position by eye geometry to get scale-invariant offsets:

- `offset_x = (iris_x − eye_mid_x) / eye_width`
- `offset_y = (iris_y − eye_mid_y) / eye_height`

Then average both eyes (with left-eye `offset_x` sign-flipped) so both eyes share a convention: `offset_x > 0` means the subject is looking toward their own right.

Units: **eye-widths** (horizontal) and **eye-heights** (vertical). 1.0 = one full eye width of displacement.

## The I-VT algorithm

I-VT = Identification by Velocity Threshold. Oldest and simplest eye-movement classifier; still the default choice for coarse analysis because it has one tunable parameter and is robust to noise.

1. Compute frame-to-frame 2D gaze velocity: `v = sqrt(dx² + dy²) / dt` in eye-widths per second.
2. Classify each sample: `fixation` if `v < threshold`, else `saccade`.
3. Merge consecutive same-class samples into events (runs).
4. Discard fixations shorter than `MIN_FIXATION_MS` (real fixations are ≥100 ms).
5. Optionally discard single-frame saccades if you're being conservative (`MIN_SACCADE_FRAMES`).

### Tuning the velocity threshold

Inspect the velocity histogram in the notebook. A well-chosen threshold falls in a low-count "valley" between the fixation peak (low velocity) and the saccade peak (high velocity). If you don't see two clear peaks:
- The subject may not have been doing many saccades (quiet video).
- The temporal resolution is hiding saccade structure (see caveat above).

The ticket's default of 30 eye-widths/sec is a starting point; a typical webcam-scale value is 10–50 eye-widths/sec depending on how centered the face is in frame.

## Parameters summary

| Parameter | Default | What it controls |
|---|---|---|
| `VELOCITY_THRESHOLD` | 30 | eye-widths/sec cutoff for I-VT |
| `MIN_FIXATION_MS` | 100 | drop fixations shorter than this |
| `MIN_SACCADE_FRAMES` | 1 | drop saccades shorter than this |

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/05_gaze_variability.ipynb`.
3. Set `VIDEO_STEM`. Run all cells.
4. Inspect the velocity histogram — if the threshold line falls badly, adjust `VELOCITY_THRESHOLD` and re-run.

## Outputs

- **Summary stats** — duration, offset stds, n_fixations, n_saccades, mean fixation duration, saccades/minute, % time fixating.
- **Time-series plot** — offsets, velocity, state strip, fixation shading.
- **Velocity histogram** — for threshold tuning.
- **`data/<video>_gaze_variability.parquet`** — per-frame offsets, velocity, and fixation/saccade booleans.

## Interpreting results

- **High fixation %, few saccades** — steady gaze, likely on a single target (camera / screen / nothing).
- **Many short fixations, high saccade rate** — scanning behavior. Literature typically associates this with lower perceived engagement / higher cognitive load.
- **Long mean fixation duration (>600 ms)** — deep attention, or zoning out — check against other signals.
- **Very short mean fixation (< 150 ms)** — at our sampling rate, likely an artifact of landmark noise, not real behavior.

## Caveats

- **Noise floor.** MediaPipe iris landmarks jitter by roughly 1–2 pixels even on a still face. On a small/far face, that's a non-trivial fraction of eye-width and will inflate the velocity signal. Solution: process higher-resolution video or move the camera closer.
- **Normalization gotcha.** Offsets are in eye-widths, not degrees of visual angle. Our thresholds and std values are not directly comparable to published eye-tracker literature that uses degrees. For degree-scale thresholds, you need either a calibrated camera-to-face distance or a full 3D gaze vector (OpenFace / MediaPipe Tasks).
- **Head motion bleeds in.** Iris offset is normalized by eye geometry, not by head pose, so a head turn can shift iris-relative-to-socket even when the eyes are fixed. For pure eye motion, combine this notebook's output with Ticket 4's head-pose series and subtract head contribution — not implemented here, but a possible upgrade.
- **Single-subject assumption.** The pipeline assumes one face; works on close-up interview video.

## Downstream dependencies

- **Ticket 8 (Synchrony):** can correlate the `is_fixation` boolean or velocity time series between subjects.
- **Ticket 7 (Relaxed demeanor):** gaze velocity could be folded in as a restlessness component, though the current Ticket 7 design uses only head jerk + anger AUs.
