# 09 — Pupil Mimicry Feasibility Spike

## Purpose

This is a **spike**, not a metric. The goal is to decide whether webcam-recorded video can support **pupil mimicry** analysis — and if not, identify what hardware is needed. The full mimicry metric (cross-subject pupil correlation over time) is **not** implemented, and won't be until the feasibility question is answered.

## The iris-vs-pupil distinction

Pupil mimicry research measures **pupil** diameter — the black opening at the center of the iris that constricts and dilates. It responds to:

- **Light** (fast, large, ~1–4 mm changes) — irrelevant under constant lighting.
- **Arousal / cognitive load** (slow, small, ~0.1–0.5 mm) — the signal of interest.
- **Observer mimicry** (observer's pupil tracks the target's) — typically under 0.3 mm amplitude.

MediaPipe Face Mesh (via iris refinement, landmarks 468–477) gives us the **iris** — the colored part. The iris is biologically **fixed in size** (~11.7 mm horizontal diameter in adults) and does not respond to arousal. So MediaPipe cannot, directly, measure pupil diameter.

What MediaPipe can do is give us a **scale reference**: because iris diameter is constant, the iris-size-in-pixels tells us how many millimeters one pixel represents. This is the standard approach used in eye-tracking hardware too.

## What this notebook does

1. **Re-runs MediaPipe** on the video to extract all 10 iris landmarks per frame (the shared pipeline only saves the 2 iris centers).
2. **Computes per-frame iris diameter** two ways (max pairwise perimeter distance; 2× mean center-to-perimeter distance) for robustness.
3. **Measures noise** in the iris-diameter signal. Because iris diameter is biologically constant, any frame-to-frame variance is pure noise (landmark jitter + perspective drift).
4. **Converts pixels to millimeters** using the iris as a ruler: `mm_per_px = 11.7 / mean_iris_diameter_px`.
5. **Computes signal-to-noise** against expected pupil-mimicry amplitudes (~0.3 mm).
6. **Produces a recommendation** for the lab.

## How to run

1. Ensure the pipeline has been run for your video (`data/<stem>_merged.parquet` + meta sidecar exist).
2. Open `metrics/09_pupil_spike.ipynb`, set `VIDEO_STEM`.
3. Run all cells. Takes a couple of minutes for the MediaPipe re-pass.
4. Fill in the recommendation template in the final cell with your observed numbers.

## Why we can't skip the MediaPipe re-run

The shared `00_pipeline.ipynb` only stores **iris centers** (landmarks 468 and 473). For a diameter estimate we also need the **four perimeter points** per iris (469–472 left, 474–477 right). We could have stored them in the pipeline but the spike design is to keep the pipeline minimal — re-running is ~1 minute per video, only needed once per feasibility study.

## What a go / no-go verdict looks like

The notebook prints an SNR number at the end:

| SNR (mimicry / noise) | Verdict |
|---|---|
| < 2 | **Not viable.** Signal is below the noise floor. Any numbers you compute are landmark jitter, not biology. |
| 2–5 | **Marginal.** Detectable in aggregate with heavy averaging but not reliable per-frame. Proceed only if a dedicated eye tracker is out of reach. |
| ≥ 5 | **Viable in principle.** But still need to handle the iris-vs-pupil issue below. |

## The fundamental issue (independent of noise)

Even if the SNR calculation came back favorable, **we still measure iris, not pupil.** Iris geometry can only serve as a *ruler* — to measure actual pupil diameter we'd need a separate pupil-boundary detector operating on the pixel region inside the iris landmarks. That's a separate CV problem, and in visible-light video the pupil-iris contrast is often too low to segment reliably (especially for dark-eyed subjects).

## Hardware options if webcam isn't viable

| Option | Precision | Cost | Notes |
|---|---|---|---|
| **Tobii Pro Spectrum** | ~0.01 mm | USD ~$30k | Research-grade, 600 Hz |
| **Tobii Pro Fusion** | ~0.05 mm | USD ~$10k | Mid-tier, 250 Hz |
| **Pupil Labs Core (head-mounted)** | ~0.05 mm | USD ~$3k | Open hardware/software, 200 Hz |
| **SR Research EyeLink 1000+** | ~0.05 mm | USD ~$40k | Gold standard in psychophysics |
| **Custom: 4K camera + DeepVOG / PupilNet** | ~0.1 mm | USD ~$500 + engineering | Possible DIY build under controlled lighting |

All of the above image the pupil in IR where it appears as high-contrast dark against a bright iris — which sidesteps the visible-light segmentation problem entirely.

## Alternative: skip pupil mimicry entirely

If pupil mimicry isn't a hard deliverable for the study, the other eight metrics in this repo already cover the face/behavioral side of rapport and trust:

- Ticket 8 (**facial expression synchrony**) — the closest behavioral analogue to pupil mimicry; measures observer-following dynamics in facial muscles.
- Ticket 1 + 2 (**expression valence**, **Duchenne ratio**) — affective cues.
- Ticket 3 (**eye contact**) — the other major nonverbal trust cue.

Unless pupil mimicry is specifically requested by the research question, focusing the energy there is probably a better investment than adding a dedicated eye tracker.

## Actual result on the sample video

Smoke test run on `sample.mp4` (1280×720, head-and-shoulders framing):

| Measurement | Value |
|---|---|
| Mean iris diameter | 19.9 px |
| mm per pixel | 0.588 |
| Frame-to-frame noise std | 0.66 px = **0.385 mm** |
| Minimum detectable change (2σ) | 0.77 mm |
| Expected pupil-mimicry amplitude | ~0.30 mm |
| **SNR (mimicry / noise)** | **0.78** |

**Verdict for this recording setup: NOT VIABLE.** The expected signal is *smaller* than the per-frame noise floor. The iris in this video occupies only ~20 px — each pixel represents ~0.6 mm, meaning the landmark-quantization floor alone is twice the size of the signal we'd want to detect. No amount of post-processing recovers information that isn't captured in the first place.

Re-running the spike on a higher-resolution, closer-framed video (iris diameter of 60+ pixels) would give a more favorable SNR, but the **fundamental iris-vs-pupil issue** would still apply — even a noise-free iris measurement tells us nothing about pupil diameter on its own.

**Recommendation:** go with option 3 — skip pupil mimicry for this pipeline and rely on Ticket 8 (facial expression synchrony) as the behavioral analogue. If pupil mimicry specifically is a deliverable, the lab needs one of the IR eye-tracker options above.
