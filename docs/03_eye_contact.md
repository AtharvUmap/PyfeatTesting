# 03 — Eye Contact Detection

## What this metric measures

How long the subject is making eye contact with the camera — a proxy for social engagement. Eye contact is one of the strongest nonverbal cues for perceived trustworthiness; more eye contact generally correlates with higher ratings of warmth, attentiveness, and honesty.

The notebook produces:
- a per-frame `eye_contact` boolean,
- total seconds and percentage of the video spent in eye contact,
- a timeline plot showing why each frame was or wasn't counted,
- an offset scatter plot for visualizing threshold tuning.

## Why two signals (iris + head pose)

Head pose alone can't tell you where the eyes are pointing — someone can look sideways with their eyes while facing forward. Iris position alone can't tell you where the head is pointing — someone can have eyes centered in their sockets while their entire head is turned 30° away.

Combining both closes both loopholes:

| Head pose | Iris offset | Result |
|---|---|---|
| frontal | centered | **eye contact** |
| frontal | off-center | looking away with eyes only |
| turned | centered (relative to turned head) | looking away with head |
| turned | off-center | looking away in both |

Only the first row counts.

## How iris offset is computed

For each eye, MediaPipe Face Mesh gives us the iris center landmark and four eye-socket corners. We compute:

- **horizontal offset** = `(iris_x − eye_mid_x) / eye_width`, where `eye_mid_x = (outer_corner_x + inner_corner_x)/2` and `eye_width = |outer_corner_x − inner_corner_x|`.
- **vertical offset** = `(iris_y − eye_mid_y) / eye_height`, where `eye_mid_y = (top_y + bot_y)/2` and `eye_height = |top_y − bot_y|`.

Dividing by eye dimensions makes the offset **scale-invariant** — 0.2 means 20% of an eye-width off center, whether the subject is close to or far from the camera.

The two eyes' offsets are averaged per frame (reduces landmark noise roughly √2×), with a sign flip on the left eye so both eyes share a convention: `offset_x > 0` means the subject is looking toward their own right.

## Calibration

Iris "centered" is subject-specific. Even when looking directly at a camera, individuals' irises sit slightly off geometric center due to eye asymmetry, head-tilt habits, and camera placement. The notebook's calibration step fixes this by:

1. Taking a user-specified timestamp range during which the subject is looking at the camera.
2. Computing the mean `offset_x` and `offset_y` in that window.
3. Using those as the **baseline** against which future frames are compared.

If `CALIBRATION_WINDOW = None` the baseline defaults to (0, 0) — only accurate when the camera is directly in front of the subject and the subject has perfectly centered eyes.

Practical tip: 1–3 seconds of calibration is plenty. Instruct the subject to look at the camera for that window, or pick a portion of the video where they clearly are (e.g. a greeting).

## Head pose — pitch, yaw, roll

Py-Feat reports head orientation as three rotation angles in **degrees**, borrowed from aviation. 0° on all three means the head is pointing straight at the camera.

| Axis | Motion | In plain English |
|---|---|---|
| **Pitch** | nodding up/down | "yes" gesture — chin up (positive) or chin down (negative) |
| **Yaw** | turning left/right | "no" gesture — face turning toward your own left or right shoulder |
| **Roll** | tilting sideways | "what?" gesture — ear tipping toward shoulder |

```
     PITCH (nodding)
        ↑
     chin up
        │
     ───┼───  → YAW (turning)
        │     face turns left/right
     chin down
        ↓

   ROLL (tilting)
   ear to shoulder ↺↻
```

For eye contact we gate on **pitch** and **yaw** only — both indicate the head is pointing somewhere other than the camera. **Roll is intentionally not gated**: a tilted head (leaning on a hand, casually reclined) doesn't break eye contact.

## Classification rule

A frame is counted as eye contact iff **all four** conditions pass:

```
|offset_x − baseline_x| < OFFSET_THRESHOLD   (default 0.15 eye-widths)
|offset_y − baseline_y| < OFFSET_THRESHOLD
|pf_Pitch|              < HEAD_PITCH_MAX     (default 15°)
|pf_Yaw|                < HEAD_YAW_MAX       (default 15°)
```

Roll (head tilt) is intentionally not gated — tilting the head doesn't break eye contact.

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/03_eye_contact.ipynb`.
3. Set `VIDEO_STEM`, optionally set `CALIBRATION_WINDOW` to a `(start_s, end_s)` tuple.
4. Run all cells.
5. Inspect the scatter plot and timeline. Tune thresholds if the classification looks off.

## Output

- `data/<video>_eye_contact.parquet` — one row per frame with `frame, timestamp, offset_x, offset_y, pf_Pitch, pf_Yaw, eye_contact`.
- Printed summary: total eye-contact seconds, % of video, calibration baseline, thresholds used.

## Caveats

- **Head pose from Py-Feat is via img2pose** — accurate to a few degrees on front-facing faces, worse on extreme angles. A 15° threshold is conservative for typical webcam footage.
- **Landmark noise.** MediaPipe iris landmarks jitter by ~1–2 pixels frame to frame even on a still face. Normalizing by eye width reduces this but doesn't eliminate it — very brief "glances away" that last one frame may be noise.
- **Single-eye fallback.** If only one eye is detected on a frame, the notebook still averages what it has. Accuracy degrades for those frames.
- **Camera vs gaze target.** This measures contact with the **camera**, not necessarily with an interviewer's eyes. If the subject is looking at a laptop screen above the webcam, they'll register as \"not looking at camera\" even though they're looking at a relevant target. Flag this when designing your recording setup.
- **Calibration drift.** If the subject moves significantly after the calibration window, baseline becomes stale. For long recordings, consider re-calibrating on multiple windows and interpolating.

## Downstream dependencies

The saved `data/<video>_eye_contact.parquet` is used by:
- **Ticket 5 (Gaze variability)** — shares the offset columns for variability stats.
- **Ticket 8 (Synchrony)** — eye-contact time series can be correlated across subjects.
