# 08 — Ground-truth analysis

After a run has been downloaded and stacked, the `sar_pipeline.analysis` package answers three questions
with labelled fields (ground-truth polygons):

1. **Are the labels usable?** Geometry checks and a time-series review of every field.
2. **Which features separate the classes?** Per-pixel features compared with spatial cross-validation.
3. **What does a map look like?** An exploratory class map of an AOI.
4. **How good is a tuned model on fields it never saw?** The model phase: holdout test and a validated map.

Nothing here talks to Earth Engine: every step reads local run outputs only.

---

## 1. Setup

Add the ground truth and an `analysis:` section to your config (see `config/pipeline.example.yaml`):

```yaml
gcps:
  path: "data/gcps/<folder>/<polygons>.shp"   # local, gitignored
  class_field: <class-name-field>              # e.g. crop name
  id_field: <class-code-field>                 # numeric code per class (used in rasters)
analysis:
  target_class: "<class-name>"
  bins: {kind: half_month, days: 15, start: null}
  ...
```

Class names and codes are always read from the ground-truth file; nothing is hard-coded.

---

## 2. Steps

All steps: `python -m sar_pipeline.analysis <step> --config config/<aoi>_<season>.yaml --run <run_id>`
(`--run` defaults to the newest run). Outputs go to `processed/<aoi>/<season>/analysis/<run_id>/`.

| Step | What it does | Main outputs |
|---|---|---|
| `gcp-qc` | geometry checks: validity, overlaps, area, pure pixels, share inside the AOI | `gcp_vector_qc.csv`, `gcp_overlaps.csv` |
| `gcp-eda` | per-field mean VV/VH per date, class curves, outlier flags, review list | `gcp_timeseries.csv`, `class_curves.png`, `gcp_outlier_flags.csv`, `gcp_review.csv` |
| `gcp-qc --write-copy` | copies the ground truth with `qc_status`, `qc_reason`, `use_in_analysis` | `<polygons>_qc.gpkg` next to the original |
| `features` | per-pixel features of the analysed fields | `<name>.parquet` + `<name>.json` |
| `evaluate` | spatial CV of every feature set in the file | `cv_summary_<name>.csv`, `confusion_<set>.csv`, optional `oof_<set>.parquet` |
| `cv-layers --set S` | CV results for QGIS | `cv_fields_<set>.gpkg`, `cv_pixels_<set>.tif` + `.qml` |
| `class-map --set S --map-config C` | class map of the AOI of config `C` | `rf_<set>_classes.tif` + `.qml`, `rf_<set>_<target>_prob.tif`, `rf_<set>_classes.json` |
| `model [--map-config C]` | holdout, tuning, threshold, holdout test, validated map (section 5) | `model_<set>/` folder; map `model_<rf\|xgb>_<set>_*` in the analysis folder of `C` |

A typical session:

```bash
python -m sar_pipeline.analysis gcp-qc   --config config/my_aoi.yaml
python -m sar_pipeline.analysis gcp-eda  --config config/my_aoi.yaml
# look at gcp_review.csv (and the fields in QGIS), edit it if needed, then:
python -m sar_pipeline.analysis gcp-qc   --config config/my_aoi.yaml --write-copy
python -m sar_pipeline.analysis features --config config/my_aoi.yaml
python -m sar_pipeline.analysis evaluate --config config/my_aoi.yaml --save-oof
python -m sar_pipeline.analysis cv-layers --config config/my_aoi.yaml --set hm_w5_s0501
python -m sar_pipeline.analysis class-map --config config/my_aoi.yaml --set hm_w5_s0501 --map-config config/other_aoi.yaml
python -m sar_pipeline.analysis model     --config config/my_aoi.yaml --map-config config/other_aoi.yaml
```

### Field QC never deletes anything

The review flags are suggestions. The original file is never modified; the copy keeps every field and only
marks `check_label` and `mixed_pixels` as `use_in_analysis = False`. `note_water` / `note_builtup` are kept
(they are real members of a catch-all class). Check flagged fields on optical imagery before relabelling.

### Comparing settings in one pass

`features` accepts several values at once, and every combination becomes a *feature set*:

```bash
python -m sar_pipeline.analysis features --config config/my_aoi.yaml \
    --bins half_month 8 12 --starts 2026-04-01 2026-05-01 --windows 5 --name bin_test
python -m sar_pipeline.analysis evaluate --config config/my_aoi.yaml --name bin_test
```

Set names read `<bins>_w<window>_s<start MMDD>`, e.g. `hm_w5_s0501` = half-month bins, 5×5 window, from 1 May.
All sets share exactly the same pixels, so their scores are directly comparable.

---

## 3. How the features are built

For each pixel, track and time bin: **VH**, **VV** and **VH−VV** (dB).

- **Time bins.** Acquisitions are grouped into calendar half-months (1–15, 16–end). One track revisits every
  ~12 days, so a 15-day bin almost always holds one acquisition per track and every field gets the same
  columns. Shorter bins leave many bins empty; longer bins blur fast crop changes.
- **Averaging in linear power.** Values are averaged as power and converted to dB at the end. Averaging dB
  values would bias the mean low.
- **5×5 window.** Speckle remains even after filtering. Averaging a 5×5 window (50 m) around each pixel
  steadies the value; nodata pixels in the window are ignored.
- **Rain rule.** Rain wets soil and leaves and raises backscatter for a day or two without any crop change. An
  acquisition with ≥ `rain_mm_24h` in the previous 24 h is dropped if its bin also has a dry one.
- **Empty bins** are linearly interpolated in time (the first/last value is repeated at the ends); the number
  of interpolated bins per set and track is written to the `.json` file.
- **Non-finite values** (for example log of zero power) are never silently kept or dropped: rows with any
  non-finite value are removed and counted in the log and the `.json` file.
- **Pixel cap.** At most `max_pixels_per_field` pixels per field (fixed seed), so a few very large fields do not
  dominate training.

### Why data from 1 May onward is the default window

Start-date tests on the reference data showed that **starting later than mid-May costs about 0.08 F1 for the
target crop**, because the early growth stage is lost, while **adding April gains nothing measurable** (April
is mostly bare or flooded soil for all classes). The recommended setting is therefore half-month bins from
1 May to the latest acquisition. The example config keeps `start: null` (season start) because the right
first date depends on each region's crop calendar: choose the first half-month before planting.

---

## 4. How the evaluation works

- **Spatial cross-validation.** Neighbouring fields share soil, weather and management, so they look alike. A
  random split would test on near-copies of training fields and inflate scores. Fields are grouped into
  `group_cell_m` cells (5 km by default), and all fields of a cell are always in the same fold.
- **Metrics.** Precision, recall and F1 of the target class (per pixel and by majority vote per field), per-class
  F1, accuracy and macro F1. Use target-class F1 as the headline number: accuracy is dominated by the largest
  classes.
- **Why a fixed Random Forest.** The goal is to compare *features*, so the model and its settings stay fixed
  (`analysis.rf`). RF needs no scaling, handles correlated features and is hard to overfit badly, which makes
  it a stable diagnostic baseline. It is **not** a tuned production model; tuning is the model phase
  (section 5).

---

## 5. Model phase

The `model` step turns the chosen feature set (the one in `analysis.bins` / `window_px`) into a tested model
and a map. It reuses `--name <file>.parquet` when that file holds the same feature set, otherwise it extracts
the features once. Every stage writes its result to `analysis/<run_id>/model_<set>/` and is skipped on a rerun.

**1. Lock away a holdout.** About 20 % of the fields are set aside *before* anything is tuned. Whole 5 km
blocks (`group`) go to the holdout, never single fields, so holdout fields have no neighbours in training.
The block choice is a seeded search: many random block orders are tried and the one is kept whose class
shares are closest to the full data, with at least 15 holdout fields per class. `holdout_split.csv` lists
every field's block and split; `holdout_composition.csv` gives fields and pixels per class. The code raises
an error if a holdout field ever reaches tuning or threshold selection.

**2. Tune on the rest (dev).** Two model families, each with a small grid, scored with grouped 5-fold CV on the
dev fields only:

| Model | Grid |
|---|---|
| Random Forest | `n_estimators` 200/500, `max_features` sqrt/0.3, `min_samples_leaf` 1/2/5, `class_weight` balanced/none |
| XGBoost (`hist`) | `max_depth` 4/6/8, `learning_rate` 0.05/0.1, `n_estimators` 300/600, `subsample` 0.8, `colsample_bytree` 0.6/0.9, balanced sample weights on/off |

The winner has the best target-class F1; ties (to 3 decimals) go to the better macro F1. All grid scores are
in `tuning_results.csv`. To save time, settings that differ only in `n_estimators` are fitted once with the
larger number and scored with the first N trees (RF) or boosting rounds (XGBoost), which gives exactly the
same predictions as fitting the smaller model separately.

**3. Pick a target threshold.** By default a pixel gets the class with the highest probability (*argmax*).
When the target class is often confused with one neighbour, a lower or higher bar for the target trades
precision against recall. The rule: *predict the target when its probability is ≥ t, otherwise the most
likely other class*. `t` from 0.20 to 0.80 (step 0.05) is scored on the winner's dev out-of-fold
probabilities and the t with the best target F1 is kept (`threshold_curve.csv`, `winner.json`).

**4. Test once on the holdout.** The winner is fitted on all dev fields and scored on the holdout with both
rules (argmax and threshold): per-class precision/recall/F1, confusion matrices, field majority-vote accuracy
and target F1, and a 95 % interval of target precision/recall/F1 from 1000 bootstrap resamples of *fields*
(pixels of one field are not independent). Files: `holdout_metrics.json`, `holdout_per_class.csv`,
`holdout_confusion_<rule>.csv`. This is the number to quote. It is computed once: if `holdout_metrics.json`
exists it is not recomputed, so the holdout cannot be tuned against by rerunning.

**5. Map.** The winner is refitted on *all* fields (dev + holdout: more data, same settings) and applied to
the AOI of `--map-config` with the threshold rule. Rain flags come from the training run, as in `class-map`.
Outputs in that AOI's analysis folder: `model_<rf|xgb>_<set>_classes.tif` (+ `.qml`, nodata 0),
`model_<rf|xgb>_<set>_<target>_prob.tif` (percent, nodata 255) and a `.json` with class area shares under
both rules.

To start over (new fields, new features), move the `model_<set>/` folder away: the step refuses to reuse it
when the holdout split no longer matches the data.

---

## 6. Caveats — read before quoting numbers

- **Scores describe field interiors.** Ground-truth polygons are drawn inside fields. Real maps also contain
  field edges, roads, trees and mixed pixels, where accuracy is lower.
- **Class balance.** Training uses `class_weight: balanced`; the class shares in the ground truth are not the
  shares in the landscape. Precision in a real map depends on how common each class really is.
- **The class map is not a validated product.** It is trained on all fields and applied to every AOI pixel,
  including places the CV never tested. Use it to spot patterns and errors, not to report areas. The model
  phase map has a holdout score behind it, but that score still describes field interiors (above), and the
  holdout is small: read its bootstrap interval, not just the point value.
- **The threshold is tuned for the ground-truth class mix.** Fields are sampled evenly per class, not in
  landscape proportions, so a threshold that maximises F1 on them can over- or under-map the target in a real
  AOI. Compare the argmax and threshold area shares in the map `.json`.
- **Rain flags of a map come from the training run.** Rain is averaged over each run's AOI, so a different AOI
  could flag different dates as wet. `class-map` uses the training run's flags so every bin picks the same
  acquisitions the model was trained on, and it stops if the two runs have different band layouts.
- **Probability raster.** `rf_<set>_<target>_prob.tif` stores 0–100 % with nodata 255, separately from the
  class raster (nodata 0), so a 0 % probability is not confused with "no data".
