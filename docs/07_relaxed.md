# 07 — Relaxed Demeanor Index (exploratory)

## ⚠ This is an exploratory metric

There is **no single validated definition** of "relaxed demeanor" in the FACS / affect literature. The composite built here is a reasonable first pass that combines upper-face muscle tension, lower-face muscle tension, and head-motion smoothness. The specific component choices and their equal weighting should be validated against rated labels on your own corpus before drawing strong trust-research conclusions.

Treat outputs as a **within-video relative signal**, useful for questions like "was the subject more relaxed in the first half than the second?" Treat them with caution for absolute or cross-subject comparisons.

## What this metric measures

A single 0–1 score per frame where **higher = more relaxed**. A "relaxed demeanor" in the trust literature is associated with higher perceived trustworthiness and approachability — but what constitutes relaxation in nonverbal behavior is fuzzier than, say, smile detection.

## Components

Five tension indicators, each normalized and inverted so larger values mean *less* tension:

| Component | Source | What it measures |
|---|---|---|
| `AU04` | Py-Feat | Brow lowerer (*corrugator supercilii*) — furrowed brow, concentration, tension. |
| `AU07` | Py-Feat | Lid tightener (*orbicularis oculi palpebral*) — narrowed, tense eyes. |
| `AU23` | Py-Feat | Lip tightener (*orbicularis oris*) — thinned, compressed lips. |
| `AU24` | Py-Feat | Lip pressor (*orbicularis oris*) — firmly pressed lips. |
| `head_jerk` | Py-Feat pose | 3D magnitude of the 2nd time-derivative of (pitch, yaw, roll). Captures how *abruptly* the head changes direction. Relaxed heads move smoothly (low jerk); startled or tense heads snap. |

### Why these four AUs

They span both halves of the face. Upper-face only (AU04, AU07) would miss subjects who express tension through their mouth; lower-face only (AU23, AU24) would miss brow-furrowers. All four covers the common tension patterns.

### Why head jerk, not head velocity or head std

Velocity and std count *how much* the head moved. A relaxed subject can still move their head a lot (slow, deliberate nods). What distinguishes relaxed from tense head motion is **smoothness** — the second derivative. A sharp snap of the head shows up in jerk but is averaged out in velocity / std over a window.

Note: the ticket names this "jerk" informally. Strictly, mechanical jerk is the third derivative of position (second derivative of velocity). Here it's the second derivative of angular position, which is angular acceleration. The name "jerk" is retained for continuity with the ticket wording.

## Algorithm

1. Extract raw `AU04`, `AU07`, `AU23`, `AU24` from Py-Feat.
2. Compute per-axis 2nd derivatives of `pf_Pitch`, `pf_Yaw`, `pf_Roll`; combine into 3D magnitude = `head_jerk`.
3. Smooth `head_jerk` over a 1-second window (rolling mean) to suppress single-frame landmark noise.
4. **Min-max normalize** each component across the video to [0, 1].
5. **Invert**: `component_rel = 1 − normalized_component`. Now higher = more relaxed.
6. **Average** the five inverted components unweighted → per-frame `relaxed` score.
7. Video-level score = mean of per-frame `relaxed`.

## Parameters

| Parameter | Default | What it controls |
|---|---|---|
| `SMOOTH_WINDOW_S` | 1.0 | Rolling-mean window (s) applied to head jerk before normalization. Shorter = more responsive, noisier; longer = more stable, may miss real snaps. |

## Important caveats about the normalization

- **Within-video relative, not absolute.** A video where the subject is tense throughout still produces full-range 0–1 component scores because min-max pins the within-video min to 0. The composite is meaningful for within-video comparison; for cross-video comparison use the printed min/max values to convert to a shared scale, or z-score against a reference corpus.
- **Outlier-sensitive.** Because min-max uses the raw min and max, a single detection glitch can shift the whole normalization. For production use, consider replacing min-max with a robust version (e.g. [5th, 95th] percentile clipping) — not done here in order to match the ticket spec literally.

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/07_relaxed.ipynb`.
3. Set `VIDEO_STEM`. Run all cells.

## Outputs

- **Per-video mean** — the single headline number (0 = tense, 1 = relaxed, within this video).
- **Component bar chart** — each component's mean score, so you can see which dimensions dominate.
- **Per-frame timeline** — composite + individual components over time.
- **`data/<video>_relaxed.parquet`** — per-frame raw components, inverted-normalized components, and composite.

## Interpreting results

- **Composite mean ≈ 0.5** — subject's tension was average across the observed range. Without a reference corpus this tells you little on its own.
- **All bars near 0.5** — tension dispersed across components equally. Composite is a fair summary.
- **One component very low** (e.g. AU04_rel = 0.2) — that muscle group was consistently tense. Check the source video — is this real (subject was furrowing) or an artifact (Py-Feat's AU04 runs high for this subject's neutral face)?
- **Composite rises or falls across the video** — compelling narrative signal; line up with video timestamps to see what caused the shift.

## Caveats

- **No validated ground truth.** Rated relaxation is the correct benchmark; the notebook does not assume one exists.
- **Head jerk conflates relaxation with activity.** A subject mid-sentence making a natural head gesture will spike jerk, even if they're relaxed. Pair with Ticket 4 (head variability) to distinguish "lots of smooth motion" from "occasional sharp snaps."
- **Py-Feat AU noise floor.** Individual AUs can have detection artifacts that create spurious high values on an otherwise calm face. Inspect the individual component traces before trusting the composite.
- **Small head jerk on head-and-shoulders video.** If the camera is locked on a mostly-still face, head jerk is driven by landmark noise. The smoothing helps but can't fully compensate. Consider dropping the jerk component in this case (re-define `components` in the notebook) and averaging only the four AUs.
- **Single-subject assumption.** Like every notebook, this one runs on one face at a time.

## Downstream dependencies

None yet. The per-frame `relaxed` score could feed Ticket 8 (synchrony) as another cross-subject signal.
