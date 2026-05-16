# Insect Passage Timeline — Poschiavino River RODI

Analysis notebook for RODI (Riverine Organism Detection Instrument) insect-passage data collected at the Poschiavino River during pre- and post-flood deployments in May–June 2025.

## What the notebook does

The notebook turns raw RODI classifications into FP-corrected, 30-minute-binned insect-passage timelines and produces four figures:

| Figure | Description |
|--------|-------------|
| `insect_timeline_30min.png` | Stacked subplot per dataset showing estimated true insects per 30-min bin, net-cleaning shading, and FP rate on a secondary axis |
| `up_vs_downstream_30min.png` | Upstream vs downstream count comparison on dual y-axes for pre- and post-flood periods |
| `diurnal_pattern_30min.png` | Mean ± SEM insect count by time-of-day, averaged across all days |
| `heatmap_day_hour.png` | 2-D day × 30-min-slot activity heatmap per dataset |

Figures are written to the `figures/` folder.

---

## Datasets

Four deployments, two stations, two flood periods:

| Dataset key | Station | Position | Period | Dates |
|-------------|---------|----------|--------|-------|
| `250528_post` | RFDO1 | Downstream | Pre-flood | 28 May – 2 Jun 2025 |
| `250529_post` | RFUP1 | Upstream | Pre-flood | 28 May – 3 Jun 2025 |
| `250609_post` | RFDO1 | Downstream | Post-flood | 9–15 Jun 2025 |
| `250609_RFUP1` | RFUP1 | Upstream | Post-flood | 9–15 Jun 2025 |

---

## Repository structure

```
data/
├── notebook_cache/
│   ├── timeline_250528_post.csv      ← pre-computed annotated timelines (fast path)
│   ├── timeline_250529_post.csv
│   ├── timeline_250609_post.csv
│   └── timeline_250609_RFUP1.csv
├── fp_rates/
│   ├── 250528_post/250528_post_sampling_summary.csv
│   ├── 250529_post/sampling_summary_post.csv
│   ├── 250609_post/250609_post_sampling_summary.csv
│   └── 250609_RFUP1/sampling_summary.csv
└── Group_2_Net_Cleaning.csv
figures/                              ← generated output (not tracked by git)
insect_timeline_analysis.ipynb
```

---

## CSV file descriptions

### `data/notebook_cache/timeline_<dataset>.csv`

Pre-computed, fully annotated 30-minute timelines — one file per dataset. The notebook loads these directly and skips all raw-data processing. Columns:

| Column | Description |
|--------|-------------|
| `bin_start` | Start of the 30-min bin (local time, tz-naive) |
| `bin_end` | End of the 30-min bin |
| `raw_count` | Total RODI-classified insect ROIs in the bin |
| `tp_rate` | True-positive rate (1 − FP rate) for the enclosing 6-h block |
| `fp_rate` | False-positive rate for the enclosing 6-h block |
| `est_true` | Estimated true insects = `round(raw_count × tp_rate)` |
| `tag` | Dataset key (e.g. `250528_post`) |
| `station` | `RFDO1` or `RFUP1` |
| `position` | `Downstream` or `Upstream` |
| `period` | `Pre-flood` or `Post-flood` |
| `status` | `normal`, `cleaning`, `no_net`, or `unknown` (see below) |

### `data/fp_rates/<dataset>/<file>.csv`

False-positive review results. Each row is one 6-hour block in which 400 randomly sampled ROIs (or all ROIs if fewer than 400) were manually reviewed. Separator is `;` for pre-flood datasets, `,` for post-flood.

| Column | Description |
|--------|-------------|
| `dataset` | Dataset key |
| `block` | 6-hour block label, e.g. `2025-05-29_06h-12h` |
| `total_insects` | Total insect ROIs in that 6-h window |
| `sampled` | Number of ROIs manually reviewed |
| `false_positive` | Count of reviewed ROIs judged to be false positives |

The FP rate for a block is `false_positive / sampled`. This rate is assumed **uniform** across all 30-min sub-bins within the 6-h block.

### `data/Group_2_Net_Cleaning.csv`

Logbook of net-cleaning events at RFDO1 (station RFDO_1). Separator is `;`. Columns:

| Column | Description |
|--------|-------------|
| `Location ID` | Station identifier (`RFDO_1`) |
| `Day` | Calendar day, e.g. `28-May` |
| `Time before cleaning` | Time net was removed for cleaning (`HH:MM`) |
| `Time after cleaning` | Time net was reinstalled (`HH:MM`) |
| `Note before cleaning` / `Note after cleaning` | Free-text operator notes |

A row with only `Time before cleaning` means the net was removed and not yet reinstalled in the same log entry (extended no-net period). A row with only `Time after cleaning` marks the end of such a period.

---

## What the notebook includes and excludes

**Included in counts:**
- All bins with `status = normal` (net in place, outside cleaning windows)
- Bins with `status = no_net` (extended no-net/repair periods) — plotted with hatching to flag reduced data quality but still summed
- Bins with `status = unknown` (RFUP1 datasets, no cleaning CSV available) — plotted at reduced opacity

**Excluded from counts:**
- Bins with `status = cleaning` — short routine cleaning windows (~10–20 min each); these bins are set to NaN in the heatmap (shown grey)

**Excluded from the diurnal pattern (section 10) and summary table (section 11):**
- Only `cleaning` bins are excluded; `no_net` and `unknown` bins are included

---

## Data transformations

1. **Binning**: Insect ROI timestamps are aggregated into 30-minute bins using `pandas.resample('30min')`.
2. **FP correction**: Each bin's raw count is multiplied by the `tp_rate` of the 6-h block it falls in. Result is rounded to the nearest integer.
3. **Cleaning window padding**: Each cleaning event (from `Time before cleaning` to `Time after cleaning`) is extended by 5 minutes on the right to account for reinstallation time.
4. **Flood gap (hardcoded)**: The RFDO1 net is assumed absent from `2025-06-01 12:14` (last pre-flood cleaning) to `2025-06-12 11:50` (first post-flood reinstallation log entry). No removal event was logged for this gap.

---

## Assumptions

- **FP rate is uniform within each 6-h block.** The 400-sample review is treated as representative of the entire block regardless of time-of-day variation within it.
- **Flood gap (RFDO1).** The net was absent throughout the entire June 1–12 flood interval. No field log of removal exists; this is an engineering assumption based on the flood event timing and the first reinstallation record.
- **RFUP1 has no cleaning log.** Both RFUP1 datasets (`250529_post`, `250609_RFUP1`) lack a net-cleaning CSV. All their bins are labelled `unknown` and the plot background is tinted yellow as a reminder that cleaning windows are not corrected for.
- **Timezone.** All timestamps are treated as local time (tz-naive). UTC offsets are stripped if present.
- **No double-counting.** The notebook deduplicates ROIs by `image_name` before building the timestamp index (when running from raw data). The cached timelines already reflect deduplicated counts.
- **6-h block boundaries are wall-clock hours** (00h, 06h, 12h, 18h). Bins whose `bin_start` falls exactly on a boundary belong to the block that starts at that hour.

---

## Requirements

```
pandas
numpy
matplotlib
```

Run the notebook from the repository root so that relative paths (`data/`, `figures/`) resolve correctly.
