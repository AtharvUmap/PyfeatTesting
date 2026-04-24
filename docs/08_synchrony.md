# 08 — Facial Expression Synchrony (two subjects)

## What this metric measures

How closely two subjects' facial expressions track each other over time. In the interaction / rapport literature, higher facial synchrony predicts stronger social connection and is a candidate mediator of trust formation in dyadic contexts.

This notebook requires **two** simultaneously-recorded videos, each run through `00_pipeline.ipynb` first to produce merged parquets. It computes:

| Measure | What it answers |
|---|---|
| Windowed Pearson correlation | *When* did synchrony happen, and on which AUs? |
| Lagged cross-correlation | Who leads whom, and by how much? |
| Overall synchrony score | One headline number for the dyad |

## Required setup

1. Record both subjects at the same time.
2. Process each video with `00_pipeline.ipynb`, producing two parquet files, e.g. `data/subjectA_merged.parquet` and `data/subjectB_merged.parquet`.
3. In this notebook, set `STEM_A = "player1"` and `STEM_B = "player2"`.

## Alignment

Even with simultaneous recording, small offsets in capture timing plus different `SKIP_FRAMES` settings can misalign samples. The notebook resamples both series to a common regularly-spaced grid using linear interpolation:

- Grid spacing = 1 / min(fpsA, fpsB) to avoid over-interpolating the slower stream.
- Grid covers the overlapping time range only (`max(start) … min(end)`).

Interpolation introduces a small low-pass smoothing effect; for short windows on high-fps signals, switch to nearest-sample resampling (not implemented here — change `np.interp` to `scipy.interpolate.interp1d(kind="previous")` if needed).

## Windowed Pearson correlation

For each signal (AU or emotion), slide a window over the aligned series and compute the Pearson correlation between A and B within each window. Parameters:

- `WINDOW_SECONDS` (default 5): window length. Shorter windows catch brief synchrony episodes but are noisier; longer windows smooth over transitions. 5 s is a common literature choice.
- `STEP_SECONDS` (default 1): how far the window advances between computations. Smaller step = denser time resolution but more redundant windows; 1 s gives good resolution without excessive compute.

The output is a 2D array of correlations (signal × time window) that becomes the synchrony heatmap.

### Why not a single global Pearson

Global Pearson returns one number per AU across the whole video — it averages moments of high and low synchrony together and loses the narrative of the interaction. Windowed Pearson preserves *when* synchrony happened.

## Lagged cross-correlation

For each signal, compute the Pearson correlation between A and B at a range of time lags, from −MAX_LAG_SECONDS to +MAX_LAG_SECONDS. Pick the lag where |correlation| is largest; report both that value and the lag.

**Sign convention used here:** **positive lag = B leads A**. At positive lag `k`, we compare `A[t]` with `B[t−k]` — B's earlier signal lines up with A's current signal, i.e. A is the follower.

**Why |correlation|** rather than raw max: a strong *negative* correlation (A smiles when B frowns) is informative and should be captured. Default to |r| but inspect the raw sign in the per-AU table.

**Edge-of-range warning:** if best_lag hits ±MAX_LAG_SECONDS, the true lag may be outside the search range. Consider widening `MAX_LAG_SECONDS`.

## Overall synchrony score

Mean of the top-K absolute correlations across all signals. Default K = 5.

**Why top-K:** synchrony concentrates in a few emotionally-loaded AUs. Averaging over all 20 AUs + 7 emotions dilutes the signal with muscles neither subject uses much. Top-K highlights the communicative AUs.

**Report K alongside the score.** A dyad with overall score 0.4 @ K=5 is different from the same 0.4 @ K=15.

## How to run it

1. Have two `<stem>_merged.parquet` files in `data/` from `00_pipeline.ipynb`.
2. Open `metrics/08_synchrony.ipynb`.
3. Set `STEM_A`, `STEM_B`. Optionally tune `WINDOW_SECONDS`, `STEP_SECONDS`, `MAX_LAG_SECONDS`, `TOP_K`.
4. Run all cells.

**Single-subject test mode:** if `STEM_A == STEM_B`, the notebook runs against the same video twice. All correlations come out as 1.0 at lag 0. This is useful for validating the code path, not for research.

## Outputs

- **Per-signal table** (`best_df`): `signal`, `best_corr`, `best_lag_s`, `corr_at_zero`. Saved to `data/<stemA>__vs__<stemB>_synchrony_per_au.parquet`.
- **Windowed correlations** (`wc`): long-format table, one row per (signal, window_center_time). Saved to `data/<stemA>__vs__<stemB>_synchrony_windowed.parquet`.
- **Heatmap plot**: AU × time grid, red/blue for positive/negative Pearson.
- **Top-K summary + overall score** printed at the end.

## Interpreting results

- **Strong positive correlation on AU12 (smile) + AU06 (cheek)** — classic rapport marker. Shared Duchenne smiling.
- **Strong positive correlation on AU04 (brow)** — shared concentration or shared concern.
- **Strong negative correlation on a reactive AU** — A consistently mirrors B's expression in opposite direction; unusual, worth inspecting frames.
- **Lag > 0.5 s** — one subject is reliably reactive to the other, not mirror-fast. Could indicate a "following" conversational dynamic.
- **Lag = 0** — simultaneous expression. Either genuine fast mirroring or shared external stimulus (both reacting to the same thing in the environment).

## Caveats

- **Correlation ≠ causation.** Both subjects reacting to the same external trigger will look like synchrony. For true coupling, consider surrogate-data tests (shuffle one subject's time series and check if the score drops).
- **Short windows × noisy signals = spurious correlations.** If AUs are mostly flat (rare activation), a 5 s window might see only noise in both streams, producing random high correlations. Report the fraction of time each AU was actually active.
- **Head pose and framing artifacts.** If one camera is at a different angle, AU detection noise differs between subjects, which can mask real synchrony. Use similar camera setups.
- **Single-signal-level — doesn't capture multimodal synchrony.** True interpersonal synchrony involves voice, gaze, posture, and facial expression together. This is one dimension.
- **Requires simultaneous recording.** Results on non-simultaneous video are meaningless.

## Downstream dependencies

None — this is typically a terminal analysis.
