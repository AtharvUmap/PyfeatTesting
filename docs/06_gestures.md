# 06 — Gesture Rate and Speed

## What this metric measures

How often and how fast the subject uses hand gestures. The trust literature ties gesture tempo to perceived confidence and engagement — faster, more frequent gestures are often rated as more trustworthy in interview contexts, up to a point where excessive gesturing reads as erratic.

The notebook reports:

| Measure | Question it answers |
|---|---|
| Gestures per minute | How frequently does the subject gesture? |
| Mean peak velocity | How fast are the gestures, on average? |
| Total time gesturing | How much of the video is in active gesture? |
| Per-event table | Start/end/duration/peak-velocity of every gesture |

## Framing caveat — read this first

MediaPipe Pose **always** emits coordinates for every landmark, even when the landmark is not visible in the frame. For out-of-frame body parts, it extrapolates a likely position from the visible body. Those extrapolated coordinates have very low **visibility** scores (the `_v` field, range 0–1), typically below 0.1.

**If the subject's hands aren't in the camera frame** (common for head-and-shoulders interview framing), this metric is **not meaningful**. The notebook gates on visibility and warns when it sees this condition. Possible remedies:

- Reshoot with wider framing that includes the hands.
- Skip this metric for videos where hands are out of frame.
- Focus on videos where you can explicitly see the hands.

## Signal source

- `mp_pose_l_wrist_x`, `_y`, `_v` (left wrist normalized x/y + visibility)
- `mp_pose_r_wrist_x`, `_y`, `_v` (right wrist)

Normalized units: `x` and `y` are fractions of the frame width/height (0 = left/top edge, 1 = right/bottom edge).

## Algorithm

1. **Visibility gate.** For each frame and wrist, if `visibility < MIN_VISIBILITY` (default 0.5), drop that wrist's coordinate (set to NaN). Don't count it as either moving or still.
2. **Per-wrist velocity.** For each wrist: `v_t = sqrt((Δx)² + (Δy)²) / Δt` in normalized-units/sec.
3. **Combined gesture velocity.** `vel = max(vel_L, vel_R)` — a gesture counts if either hand is moving.
4. **Event detection.** Contiguous runs where `vel > VELOCITY_THRESHOLD`.
5. **Duration filter.** Drop events shorter than `MIN_DURATION_MS` (default 200 ms) — real gestures last at least 200–300 ms; shorter runs are landmark jitter.
6. **Per-event peak.** For each surviving event, record the max velocity — this is the standard "gesture speed" metric in the literature.

## Parameters

| Parameter | Default | What it controls |
|---|---|---|
| `VELOCITY_THRESHOLD` | 0.3 | normalized-units/sec cutoff for \"moving\" |
| `MIN_DURATION_MS` | 200 | events shorter than this are noise |
| `MIN_VISIBILITY` | 0.5 | drop frames where the wrist isn't confidently detected |

The ticket-spec defaults (0.3, 200) work for a subject fully in frame with modest/normal gesturing. If your corpus features larger gestures (e.g. animated speakers), raise the threshold; if very subtle gestures, lower it.

## Why use max(L, R) instead of summing

Summing double-counts bilateral gestures (both hands moving together) and obscures asymmetric gestures (one dominant hand). `max` collapses both hands into a single "is anything moving fast" signal, matching how a human observer perceives gesture presence. For handedness studies, split the analysis per hand using the `vel_l` / `vel_r` columns saved to the output parquet.

## Why 2D (x, y) and not 3D (x, y, z)

MediaPipe's `z` coordinate is a monocular-depth estimate; it's noisier than x/y, especially for extremities. Gesture literature is defined on what a viewer sees — frame-plane motion. 2D is the defensible choice.

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/06_gestures.ipynb`.
3. Set `VIDEO_STEM`. Run all cells.
4. **Check Section 1's visibility diagnostic first.** If `pct_visible` is below ~30% for both wrists, stop — the metric is unreliable for this video.

## Outputs

- **Summary stats** — gestures/min, mean peak velocity, total gesturing time, % time gesturing, parameters used.
- **Per-event table** — one row per gesture with start, end, duration, peak velocity.
- **Plot** — per-wrist velocity traces on top; combined velocity + threshold + event shading on bottom.
- **`data/<video>_gestures.parquet`** — per-frame `vel_l`, `vel_r`, `vel`, `is_gesturing`.

## Interpreting results

- **High gestures/min with high mean peak velocity** — energetic, animated speaker.
- **Low gestures/min with high mean peak** — infrequent but emphatic gestures.
- **High gestures/min with low mean peak** — fidgety or constant small gestures.
- **Low gestures/min with low mean peak** — stationary, maybe reserved or disengaged.

## Caveats

- **Framing is everything.** No hands in frame = no data. Nothing the algorithm can do about it.
- **Normalized units.** Velocity is in frame-widths per second, not meters/sec. A subject closer to the camera will appear to gesture \"faster\" than one farther away, for the same real-world motion. Control for camera setup when comparing across videos.
- **Sampling rate undercounts fast micro-gestures.** At 6 effective fps, gestures lasting <330 ms (two frames) are near the resolution floor. Real beat gestures are often in this range.
- **Gesture type isn't classified.** This measures presence and speed, not what the gesture means (iconic, deictic, beat, metaphoric). Discourse-level analysis requires human coding or a specialized gesture classifier.
- **Resting wrist still moves a little.** Even a perfectly still subject produces some wrist velocity from breathing and tiny adjustments. The threshold is specifically there to filter this; inspect the velocity histogram if results look off.

## Downstream dependencies

- **Ticket 7 (Relaxed demeanor):** could optionally fold gesture frequency into the composite (not currently wired in).
- **Ticket 8 (Synchrony):** gesture-presence time series can be correlated across subjects.
