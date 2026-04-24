# 04 — Head Movement Variability

## What this metric measures

How much and how unpredictably the subject's head moves during the video. The trust literature associates moderate head movement with engaged conversation and very high or erratic head movement with nervousness, distraction, or cognitive load. Very low movement can indicate disengagement or freezing.

The notebook computes **three complementary measures**, each answering a different question:

| Measure | Question it answers | Output |
|---|---|---|
| Rolling-window std | *When* was the head moving, on which axes? | Time series per axis |
| Angular path length | *How much* total head motion was there? | One scalar per axis + total |
| Sample entropy | *How predictable / structured* was the motion? | One scalar per axis |

Using all three protects you from misleading conclusions: a subject can have high path length but low entropy (slow steady tracking), low path length but high entropy (tiny jittery twitches), or any combination.

## Head pose recap

Py-Feat reports three rotation angles, all in **degrees**:

- **Pitch** — nodding up/down
- **Yaw** — turning left/right
- **Roll** — tilting sideways (ear to shoulder)

0° on all three means head pointing straight at the camera.

## Measure 1: rolling-window std

Standard deviation computed over a sliding window (default 2 s) around each frame, per axis.

**Intuition:** at each moment, "how bouncy has the head been in the last 2 seconds on this axis?"

**Why rolling instead of global:** a single global std collapses everything to one number, losing temporal structure. Rolling std preserves *when* movement happened — you can see quiet stretches followed by bursts.

**Parameter:** `WINDOW_SECONDS` (default 2.0). Shorter windows are more responsive but noisier; longer windows are smoother but hide short bursts. 2 s aligns with typical head-nod cadence in conversation.

## Measure 2: angular path length

Treat each frame's `(pitch, yaw, roll)` as a point in 3D pose space. The path length is the sum of Euclidean distances between consecutive frames. Also reported per-axis.

**Intuition:** "if I traced the head-orientation trajectory through space, how far did it travel?"

**Why normalize by duration:** report `path_per_second` to compare videos of different lengths.

**Per-axis path lengths** reveal *which* kind of motion dominated: all-pitch means lots of nodding, all-yaw means lots of turning, etc.

**Gotcha:** path length is computed on *sampled* frames (every Nth frame of the source, per `SKIP_FRAMES` in the pipeline). Motion that happens entirely between samples is missed. Keep `SKIP_FRAMES` constant when comparing videos.

## Measure 3: sample entropy

Sample entropy (Richman & Moorman, 2000) measures how unpredictable a time series is. Concretely, `SampEn(m, r, N)` asks: given the last `m` points, how often does the next point also match a previous pattern?

Algorithm (for embedding dimension `m`, tolerance `r`, length `N`):
1. Build all length-`m` subsequences.
2. Count pairs that match within `r` on every coordinate → count `B`.
3. Do the same for length-`m+1` subsequences → count `A`.
4. `SampEn = −ln(A / B)`.

Low sample entropy → many length-`m` matches also extend to length-`m+1` → signal is self-similar / predictable.

High sample entropy → pattern rarely continues → signal is irregular / unpredictable.

**Standard parameter choices (used here):**
- `m = 2` — embedding dimension. Larger m needs much more data.
- `r = 0.2 × std(signal)` — tolerance scaled to each signal's own variability, so subjects with different motion ranges are comparable.

**Rough interpretation for head pose:**
- `~0.0–0.4` — very regular motion (still, or rhythmic).
- `~0.5–1.0` — typical natural conversation.
- `~1.2+` — erratic, hard-to-predict motion.

These ranges are heuristics, not hard thresholds — calibrate against your corpus.

**Why sample entropy and not FFT / periodicity measures:** sample entropy is nonparametric and doesn't assume any underlying periodic structure. Head motion is rarely periodic but often "structured" in harder-to-define ways; entropy captures that.

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/04_head_variability.ipynb`.
3. Set `VIDEO_STEM`. Optionally adjust `WINDOW_SECONDS`.
4. Run all cells.

## Output

- **Summary stats table** — per-axis mean angle, global std, path length, and sample entropy.
- **3-axis time-series plot** — raw pose on top, rolling std on bottom.
- **`data/<video>_head_variability.parquet`** — per-frame raw pose + rolling-std columns for reuse.

## Caveats

- **Frame sampling matters.** All three measures are computed on the sampled frame rate. Keep `SKIP_FRAMES` constant across comparisons.
- **Detection failures show up as NaN.** We drop those rows before computing; if a video has many dropouts the entropy and path length become less reliable.
- **Head pose ≠ gaze.** A subject who moves their eyes a lot but holds their head still will score low here. Use Ticket 5 (gaze variability) for the complementary measure.
- **Degrees, not radians.** Thresholds like "15°" from eye-contact detection are directly comparable; don't accidentally treat them as radians.
- **Absolute vs relative.** Our path length uses raw degree deltas. For cross-subject comparison in studies, consider z-scoring each subject's pose time series first to normalize for baseline motion style.
- **Low-signal entropy trap.** If an axis barely moves (e.g. yaw span of only a few degrees), sample entropy is dominated by landmark noise rather than real motion and will read *high*. Always cross-check entropy against the std / path-length on the same axis — if the axis has near-zero path, its entropy value is not meaningful.

## Downstream dependencies

- **Ticket 7 (Relaxed demeanor):** uses head-motion jerk (second derivative of pose) as one component. Shares the pose series from this notebook.
- **Ticket 8 (Synchrony):** head-movement time series can be correlated across subjects.
