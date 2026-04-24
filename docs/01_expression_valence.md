# 01 — Expression Valence

## What this metric measures

"Valence" is the positive-vs-negative dimension of facial affect. Positive expressions (smiles) correlate with warmth and approachability, which the trust literature links to higher perceived trustworthiness. Negative expressions (anger, brow-lowering, lid-tightening) correlate with dominance/threat cues and lower perceived trustworthiness.

This notebook quantifies valence across a video two ways:

1. **Emotion-classifier view:** Py-Feat's 7-class emotion probabilities tell you which emotion dominates at each moment. Useful for answering "what % of the video was the subject visibly happy vs angry vs neutral?"
2. **Action-Unit view:** AU12 (lip-corner pull = smile) and a 5-AU anger composite matching Ekman's EMFACS anger prototype — AU04 (brow lower), AU05 (upper-lid raise), AU07 (lid tighten), AU23 (lip tighten), AU24 (lip press). This gives a continuous intensity signal rather than a categorical label, covers both the upper-face (eye region) and lower-face (mouth) components of anger, and answers "how strong were the smiles, and how did valence change moment-to-moment?"

The two views usually agree, but disagree in informative ways. A subject can be "neutral" by the classifier while AU12 shows a faint smile — the AU view is more sensitive to sub-threshold affect.

## Why two signals instead of just one

- **Emotion probabilities** are end-to-end classifier outputs. Easy to interpret as "dominant emotion", but they collapse intensity and can mis-label mixed expressions.
- **AUs** are the atomic muscle movements underlying expressions. More granular, literature-standard for FACS-based analysis, and let you build composites tailored to your question (e.g. your anger composite could instead be AU04+AU23 for "controlled anger" — the signal is yours to recombine).

For trust research, the AU view is what most papers report, so we compute both and save the AU-derived valence as the reusable signal.

## How to run it

1. Make sure `data/<video>_merged.parquet` exists (produced by `00_pipeline.ipynb`).
2. Open `metrics/01_expression_valence.ipynb`.
3. Set `VIDEO_STEM` in the config cell to match your video's filename stem.
4. Run all cells.

## What you get

- **Summary table** — total seconds spent in each dominant emotion.
- **Stacked emotion timeline** — probability mix over time (100% stacked area).
- **Smile vs anger plot** — AU12 and anger-composite intensities overlaid.
- **Valence timeline** — `smile - anger_composite` per frame, with positive (green) and negative (red) regions filled.
- **Summary stats** — mean valence, time positive/negative, % positive.
- **Output file** — `data/<video>_valence.parquet` with columns `frame, timestamp, smile, anger_composite, valence`. Later notebooks (Duchenne detection, synchrony, relaxed-demeanor composite) can load this directly instead of re-deriving these columns.

## Interpreting results

- **Mean valence near 0** with high std = expressive but mixed. Not the same as "blank."
- **Mean valence positive, low std** = consistently warm, low dynamic range. Typical of polite/professional.
- **Mean valence negative, with spikes** = baseline tense with brief smiles. Often more diagnostic than "dominant emotion" summary.
- **Dominant = neutral for much of the video** is normal — the classifier is conservative. Use the AU view to see if there's sub-threshold affect the classifier is calling neutral.

## Caveats

- **AU intensities are 0–1** in Py-Feat 0.6.2, but calibration varies by subject. Don't compare raw AU means across different people without normalizing.
- **Anger composite is a simple unweighted mean** of AU04/05/07/23/24 — the five AUs in Ekman's EMFACS anger prototype. Upper face (AU04/05/07) and lower face (AU23/24) are both covered so subjects who express anger more with the mouth than the brow aren't penalized. Alternative definitions exist (weighted by detection reliability, max instead of mean, product, etc.) — reconsider for your specific study.
- **Valence is not the same as arousal.** A tight-lipped angry face and a surprised face can both be high-arousal but opposite valence. This notebook captures valence only.
- **Video quality matters.** AU detection degrades with occlusion (hand over mouth), extreme head pose, or poor lighting. Check the head-pose notebook (Ticket 4) for frames where the subject was turned away — those AU values are less reliable.

## Downstream dependencies

The saved `data/<video>_valence.parquet` is used by:
- **Ticket 2 (Duchenne):** uses smile events as input, then checks AU06.
- **Ticket 7 (Relaxed demeanor):** uses the anger composite as one of several components.
- **Ticket 8 (Synchrony):** uses valence as one of the signals to correlate across subjects.
