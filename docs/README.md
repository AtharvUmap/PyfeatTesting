# PyfeatTesting — Facial Behavior Analysis Pipeline

A project for extracting facial expression, head pose, eye gaze, and body-pose signals from video, organized for studying trust-related social behaviors.

## What this project does

Given a video of one subject, this pipeline extracts a rich per-frame feature set and makes it available to a library of small, focused "metric" notebooks. Each metric notebook computes one specific behavioral measure (e.g. eye contact duration, head movement variability, Duchenne smile ratio) on top of that shared feature set.

This design means **the slow extraction step runs once per video**, and every downstream metric notebook is fast and operates on identical data, so results stay comparable across metrics.

## Tool stack

| Tool | Purpose |
|---|---|
| [**Py-Feat**](https://py-feat.org/) | Action Units (AUs), 7 basic emotions, head pose (pitch/yaw/roll) |
| [**MediaPipe Face Mesh**](https://developers.google.com/mediapipe) (with iris) | iris landmarks for true eye-gaze direction |
| **MediaPipe Pose** | body landmarks (shoulders, wrists, hips, etc.) |
| **MediaPipe Hands** | 21 landmarks per hand |

## Folder layout

```
PyfeatTesting/
├── .venv/                          # Python 3.11 virtual environment
├── sample.mp4                      # your input video(s) go at the project root
├── metrics/
│   └── 00_pipeline.ipynb           # the shared preprocessor (see below)
├── data/                           # outputs land here (gitignored by convention)
│   ├── <video>_merged.parquet      # per-frame merged feature table
│   └── <video>_merged.meta.json    # sidecar with fps, dims, skip_frames
├── docs/
│   └── README.md                   # this file
└── 04_fex_analysis.ipynb           # original Py-Feat exploration notebook
```

Future metric notebooks will live next to `00_pipeline.ipynb` and follow the naming convention `NN_<metric>.ipynb` (e.g. `01_eye_contact.ipynb`, `02_head_variability.ipynb`).

## Setup

```bash
# from the project root
source .venv/bin/activate

# if the env is fresh, install:
pip install py-feat "mediapipe==0.10.14" "numpy<1.24" "scipy<1.14" pyarrow opencv-python
```

**Version pins that matter:**
- `mediapipe==0.10.14` — later versions dropped the `mp.solutions` API on ARM64 Mac.
- `numpy<1.24` and `scipy<1.14` — required by `nltools`/Py-Feat.

## How to use it

### Step 1 — Drop your video in

Put your video at the project root, e.g. `/Users/.../PyfeatTesting/my_video.mp4`.

### Step 2 — Run the pipeline once per video

Open `metrics/00_pipeline.ipynb` and edit the **Config** cell:

```python
VIDEO_PATH = "../my_video.mp4"   # relative to the notebook (which is in metrics/)
SKIP_FRAMES = 5                  # process every 5th frame (good default)
```

Then **Run All**. It will:
1. Read video metadata (fps, frame count, dimensions).
2. Run Py-Feat's `Detector` on every Nth frame → AUs, emotions, pose.
3. Run MediaPipe Face Mesh + Pose + Hands in a single video pass → landmarks.
4. Merge both outputs by frame number into one DataFrame.
5. Save `data/my_video_merged.parquet` + `data/my_video_merged.meta.json`.

Expect a few minutes of runtime (Py-Feat is the bottleneck — roughly real-time at `SKIP_FRAMES=5`).

### Step 3 — Run any metric notebook

Any downstream notebook starts by loading the parquet:

```python
import pandas as pd
df = pd.read_parquet("../data/my_video_merged.parquet")
meta = pd.read_json("../data/my_video_merged.meta.json", typ="series")
FPS = float(meta["effective_fps"])   # frames-per-second of the sampled series
```

From there, each metric notebook does its own analysis and produces numbers + plots.

## What's in the merged parquet

| Column prefix | Source | Contents |
|---|---|---|
| `frame` | — | integer frame index (the merge key) |
| `timestamp` | — | frame / fps, in seconds |
| `pf_*` | Py-Feat | `pf_AU01_r` ... `pf_AU45_r` (AU intensities), `pf_anger` ... `pf_neutral` (emotion probs), `pf_Pitch`, `pf_Yaw`, `pf_Roll`, 68 facial landmarks (`pf_x_0`..`pf_y_67`), face bounding box, identity embedding |
| `mp_face_*` | MediaPipe Face Mesh | iris centers (`mp_face_l_iris_x/y/z`, `mp_face_r_iris_x/y/z`) and eye corners/top/bottom (normalized 0–1 coords) |
| `mp_pose_*` | MediaPipe Pose | nose, shoulders, elbows, wrists, hips — each with x/y/z/visibility |
| `mp_hand_left_*` / `mp_hand_right_*` | MediaPipe Hands | 21 landmarks per hand, x/y/z |

Rows with no detection on a given extractor will have NaN for that extractor's columns — handle with `.dropna(subset=[...])` as appropriate.

## Ticket-based development

Metrics are implemented one at a time. See the running list of tickets (Ticket 0 = this pipeline; Tickets 1–9 = individual metrics like eye contact, head variability, Duchenne detection, etc.). Each metric notebook reads the parquet and writes nothing back to the shared data — it produces its own plots and summary numbers in the notebook itself.

## Troubleshooting

- **`Could not open ../sample.mp4`** — video path is relative to the notebook's location (`metrics/`), not the project root. Use `../your_video.mp4`.
- **`reset_index` column-collision error** — Py-Feat 0.6.2 returns an index named `frame` *and* a column named `frame`. The normalization cell in `00_pipeline.ipynb` handles this; make sure you're running the current version from disk (reload the notebook if your IDE cached an older copy).
- **`mp.solutions` AttributeError** — you have a newer mediapipe. Downgrade: `pip install "mediapipe==0.10.14"`.
- **Extraction is slow** — raise `SKIP_FRAMES`. Going from 5 to 10 halves runtime; time resolution drops from 6 fps to 3 fps effective.

## Limitations to be aware of

- **Pupil diameter / pupil mimicry** from webcam video is not reliable. Sub-pixel iris diameter changes are swamped by lighting noise. A dedicated IR eye tracker (Tobii, Pupil Labs) is likely required if this is a core metric.
- **Head pose ≠ eye gaze.** Py-Feat gives head pitch/yaw/roll only. True gaze direction comes from the MediaPipe iris landmarks (which is why we run both).
- **One subject per video.** `max_num_faces=1` / `max_num_hands=2`. For dyadic synchrony studies, run each subject's video through the pipeline separately and merge by timestamp downstream.
- **No audio analysis.** This pipeline is visual-only.
