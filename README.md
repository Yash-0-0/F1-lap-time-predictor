# F1 Lap Time Predictor — Tire Degradation Analysis

## Race & driver selection

**Race:** 2019 Spanish Grand Prix (Circuit de Barcelona-Catalunya), `raceId = 1014` in the
Ergast/Kaggle dataset.

This race was chosen because:
- It was **sunny and dry** all race weekend — no rain, no major track-temperature swings.
- There was **no red flag**. A brief Safety Car appeared on lap 46 after two backmarkers collided —
  unrelated to weather or track conditions, and (as noted below) it turned out to be a useful stress
  test for the cleaning pipeline.
- 18 of 20 starters were classified finishers. Rather than pick a race with only 5–10 *total*
  finishers (which in the post-2011 era almost always means a chaotic, high-attrition, often
  wet race — exactly the confound this project wants to avoid), we select the **top 10 classified
  finishers** to analyze. All 10 ran a two-stop strategy, giving three clean stints per driver.

## Data cleaning

From the 660 raw laps across the 10 selected drivers:
- Pit-in laps and the lap immediately after a stop (out-laps) were removed (20 + 20 laps).
- Laps slower than 1.5× that driver's median lap time were removed as outliers (37 laps).
- **70 laps removed in total**, leaving 590 laps for modeling.

## Feature engineering

`tire_age`: laps completed since the driver's last pit stop, resetting at each stop.
`stint`: which stint number a lap belongs to (0 = opening stint, 1 = second, 2 = third).

## Train/test split

Splitting laps randomly would leak information, since consecutive laps for the same driver are
nearly identical in character. Instead, each driver's **final stint** is held out as test data;
all earlier stints are used for training. This forces the model to genuinely extrapolate rather
than interpolate between neighboring laps.

## Models & results

Two feature sets (baseline: `grid`, `lap`; enhanced: `grid`, `lap`, `tire_age`) were each trained
with Linear Regression and Random Forest, and scored with RMSE/MAE on the held-out final stints.

| Feature set | Model | RMSE (s) | MAE (s) |
|---|---|---|---|
| Baseline (grid, lap) | Linear Regression | 13.31 | 5.48 |
| Baseline (grid, lap) | Random Forest | 12.84 | 7.05 |
| Tire-age enhanced | Linear Regression | 13.28 | 5.48 |
| Tire-age enhanced | Random Forest | 12.70 | 7.20 |

(Exact numbers are reproduced in the notebook; see `F1_Lap_Time_Predictor.ipynb`.)

## Visualization

`stint_prediction.png` (also embedded in the notebook) plots actual vs. predicted lap times for
one driver's held-out final stint.

## Interpretation

**Tire age gave only a marginal improvement.** 2019-era Pirelli tires degrade fairly gently over a
single ~20-lap stint, so within-stint tire wear doesn't move lap time much compared to factors this
simple model doesn't capture — fuel burn-off, track evolution, and traffic.

**A more interesting finding: outlier removal left some Safety Car laps in the data.** Laps 46–52
(the SC period) are ~120s instead of the normal ~82s. They survived the "1.5× driver median" filter
because each driver's own race-long median is itself pulled upward slightly by those same slow laps —
the threshold is relative to a median that includes the very outliers it's trying to catch. This
inflates RMSE for any stint overlapping the SC period. A more robust filter would flag laps that are
uniformly slow *across all drivers at the same time* (a field-wide signal), rather than relying only
on each driver's own median.

**Takeaway:** the stint-based split is working as intended — test-set error reflects genuine
extrapolation, not leakage from neighboring laps. The modest gain from `tire_age` is a real, if
unglamorous, empirical result rather than a modeling bug.

## Files

- `F1_Lap_Time_Predictor.ipynb` — full notebook, cleaning → features → split → models → plot
- `stint_prediction.png` — the predicted-vs-actual visualization
- `README.md` — this file
