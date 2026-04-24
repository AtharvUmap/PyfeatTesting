# 02 — Duchenne Smile Detection

## What this metric measures

A **Duchenne smile** is a smile that activates both:
- **AU12** (*zygomaticus major*) — pulls the lip corners up. This is the mouth part of the smile.
- **AU06** (*orbicularis oculi pars orbitalis*) — raises the cheeks and produces crow's-feet wrinkles around the outer eye.

Duchenne smiles are tied to genuine felt positive emotion. Smiles that show AU12 alone — without AU06 — are called **non-Duchenne** (or "social", "polite", or "Pan-Am" smiles); they can be produced voluntarily and are common in masking contexts.

For trust research, the **Duchenne ratio** (fraction of smiles that were Duchenne) tends to be a stronger predictor of perceived warmth and trustworthiness than raw smile time.

## Why AU06 specifically

*Orbicularis oculi pars orbitalis* is difficult — though not impossible — to activate voluntarily. It is the classical FACS marker for "felt" smiles. The distinction goes back to 19th-century neurologist Duchenne de Boulogne, who noticed that "the sweet emotions of the soul activate the pars lateralis; the muscle around the eye is not obedient to the will."

Contemporary caveats:
- Some individuals can voluntarily activate AU06, so the marker is probabilistic, not deterministic.
- Very intense non-Duchenne smiles can mechanically raise the cheeks enough to trigger AU06 incidentally (Krumhuber & Manstead, 2009). For high-stakes claims, filter on AU06 duration or intensity in addition to presence.

## How events are detected

The notebook treats each contiguous stretch of frames where `AU12 > threshold` as one smile event. Key parameters (all configurable in the config cell):

| Parameter | Default | What it controls |
|---|---|---|
| `AU12_THRESHOLD` | 0.5 | AU12 intensity above which a frame counts as smiling. Py-Feat 0.6.2 AUs are 0–1 and bimodal (either ~0 or ~0.9 when active), so any value between 0.3 and 0.7 gives similar results. |
| `AU06_THRESHOLD` | 0.5 | Peak AU06 within an event that qualifies it as Duchenne. |
| `MIN_EVENT_FRAMES` | 2 | Drops events shorter than this. At 6 effective fps, this is ~0.3 s — below the lower bound of natural smile duration. |
| `MERGE_GAP_FRAMES` | 1 | Merges events separated by 1 below-threshold frame. Smooths over single-frame detection dropouts (e.g. a quick blink mid-smile). |

Each event is classified by the **peak** AU06 within its time window (not mean) — a smile might activate AU06 only at its apex, and peak catches that.

## How to run it

1. Ensure `data/<video>_merged.parquet` exists (from `00_pipeline.ipynb`).
2. Open `metrics/02_duchenne.ipynb`.
3. Set `VIDEO_STEM` in the config cell to match your video's filename stem.
4. Run all cells.

## What you get

- **Per-event table** — one row per detected smile, with `start_time`, `end_time`, `duration_s`, `peak_AU12`, `peak_AU06`, `mean_AU06`, and `type` (duchenne / non-duchenne).
- **Summary stats** — total smile count, Duchenne/non-Duchenne counts, ratio, total time in each.
- **Overlaid plot** — AU12 and AU06 time series with events shaded green (Duchenne) or red (non-Duchenne).
- **Output file** — `data/<video>_duchenne_events.parquet` for reuse.

## Interpreting results

- **Duchenne ratio ≈ 1.0** — most smiles were genuine. Typical of relaxed, enjoyable interactions.
- **Duchenne ratio ≈ 0.0** — most smiles were polite/social. Typical of strained, professional, or deceptive contexts.
- **Few smiles total** — the ratio is unreliable. Report raw counts alongside.
- **All events Duchenne** on a well-lit, high-engagement video is expected; don't over-interpret.
- **Tune thresholds first** if results look wrong. A subject with dramatic expressions and clean detections might need a higher threshold; a subject with subtle expressions might need lower.

## Caveats

- **This is a facial-muscle classifier, not a lie detector.** A non-Duchenne smile doesn't prove absence of felt emotion; a Duchenne smile doesn't prove honesty.
- **Detection depends on visibility.** If the subject's eyes are occluded (hand on face, hair, strong shadows), AU06 can read as absent even during genuine smiles. Cross-check flagged non-Duchenne events against the source video.
- **Head pose matters.** Extreme yaw/pitch degrades AU detection for both AU12 and AU06. Consider filtering events where the pipeline's head-pose columns exceed, say, 20° on any axis.
- **Scale caveat.** Py-Feat 0.6.2 returns AU intensities in 0–1; older papers and the FACS manual use 0–5. Don't port thresholds between the two without rescaling.

## Downstream dependencies

The saved `data/<video>_duchenne_events.parquet` is used by:
- **Ticket 8 (Synchrony)** — as a discrete event stream for cross-subject smile alignment analysis.
