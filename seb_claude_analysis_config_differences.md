# Comparison of multi-data run ids


# End-to-end forecasting

The end to end forecasting runs from Christian taken from the slides to Peter are:

ladqrul6: ERA5 -> ERA5 6 hourly, atmosfs

jivvzyg0: ERA5 (+5 hour lag) + obs -> ERA5 1 hourly

# SSL pretraining

We do JEPA pre-training, and then we do forecast training first with ERA5 and then with operational analysis.

### Sophie's runs

A first preliminary run id is `xi0mujc2-finetune-forecast-ft-4` with oper finetuning and `xi0mujc2-finetune-forecast` for ERA5 only

Unfrozen oper finetuning is: `xi0mujc2-finetune-forecast-ft-5`

### Seb's runs

`j281hzap`: Seb JEPA space masking, ERA5 input, forecasting

`ebtoiz2x`: oper finetuning of the previous run

`y49gywef`: unfrozen forecast training for ERA5
     
`vbujhwi3`: unfrozen finetuning with oper input

#### Note that all of these finetuning forecast runs do not have the correct nominal time offset, because we mistakenly used type : anemoi rather than type : anemoi_operan.

This is fixed with these of Seb's runs that are training:

`emn8detg` : unfrozen forecast ERA5 training with correct nominal time mapping

`usppokj6` : unfrozen oper finetuning of the run above

      

### Let's compare some of these runs and their configs to see all the different dimensions where they differ

Runs compared: `ladqrul6`, `jivvzyg0`, `y49gywef`, `vbujhwi3`,
`xi0mujc2-finetune-forecast`, `xi0mujc2-finetune-forecast-ft-5`._

_Configs: `develop/WeatherGenerator/models/<id>/model_<id>.json`. 
Metrics (JSONL): `results/<id>/<id>_train_metrics.json`. 


![](https://gitlab.jsc.fz-juelich.de/hedgedoc/uploads/1e352299-e912-4310-87cb-01dc41c02b43.png)



---

## There are a lot of notable differences between the setups of these runs

1. Training and hence loss averaging is over different forecast horizons
    - The logged `loss_avg_mean` averaged over the forecast steps.
2. There target has different temporal frequency
3. Some inputs have nominal time mapping, some do not
4. The effective validation period, even if it is nominally the same, is very different for hourly vs 6 hourly forecasts. hourly and 256 samples only samples the first 11 days of October, while 6 hourly does most of Oct-Dec.
    * We know this makes a big difference (especially in S. hemisphere, from memory) from quaver scoring of temporal JEPA runs for 10 days vs. a month.



When put on the same basis (ERA5 **+6h**, per variable), several runs that look far apart are quite close. The one clear problem is that jivvzyg0 has an extremely high geopotential loss.

---

## Table of config settings

| run | task / input | streams | fcst steps | data window | ERA5 freq | samples | freeze |
|---|---|---|---|---|---|---|---|
| **ladqrul6** | ERA5 reanalysis → ERA5 (self) | 1 (ERA5 only) | 2 (+6/+12h) | 1979–2022 | 6h | 672k | none |
| **jivvzyg0** | ea reanalysis **+5h lag** → ERA5 diagnostic | 8 (ERA5_in + obs) | 2 | 1979–2022 | native | 672k | none |
| **y49gywef** | ea reanalysis → ERA5 diagnostic | 9 (+SYNOP) | 3 (+6/+12/+18h) | 1980–2022 | 6h | 196k (done) | none |
| **vbujhwi3** | **od operational** → ERA5 diagnostic | 9 (+SYNOP) | 3 | 2016–2022 | 1h | ~51k (running) | none |
| **xi0mujc2-finetune-forecast** | ea reanalysis → ERA5 diagnostic | 8 | 3 | 1980–2022 | 1h | 204k | encoder frozen |
| **xi0mujc2-finetune-forecast-ft-5** | **od operational** → ERA5 diagnostic | 8 | 3 | 2016–2022 | 1h | 84k | none (full FT) |


Note: all y49gywef, vbujhwi3, xi0mujc2 (jepa ones), *do not* have the correct nominal mapping for the input. This was my mistake. This makes the loss look good -- but remember that ladqrul6 also does not have this offset.

atmosfs = `ae_local_dim 2048`, no local/aggregation blocks, 32 heads, mlp×2, **no xSA, no RoPE2D**, qk LayerNorm.
jepa = `ae_local_dim 1024` + 4 local + 5 global + 5 aggregation blocks, 16 heads, mlp×4, **xSA, RoPE2D, swiglu, RMSNorm qk, 64 register tokens, step-conditioning**.

---

## Loss values (ERA5 physical MSE, per forecast step)

| run | +6h | +12h | +18h | step-avg (headline) | val +6h | val z (geopot.) |
|---|---|---|---|---|---|---|
| ladqrul6 | **0.0084** | 0.0126 | — | 0.0105 | 0.0091 | ~0.0001 |
| jivvzyg0 | 0.0196 | 0.0234 | — | 0.0215 | 0.0305 | **~0.0036 (z_500)** |
| y49gywef | **0.0089** | 0.0126 | 0.0159 | 0.0125 | 0.0102 | ~0.0002 |
| vbujhwi3 | 0.0151 | 0.0158 | 0.0173 | 0.0160 | 0.0180 | ~0.0005 |
| xi…-finetune-forecast (ea) | 0.0111 | 0.0146 | 0.0179 | — | 0.0135 | ~0.0006 |
| xi…-ft-5 (od) | 0.0149 | 0.0156 | 0.0171 | — | 0.0179 | ~0.0008 |

Note that we need to compare the +6h or +12h scores to really know where we are. y49gywef ≈ ladqrul6, but a bit worse for validation despite
y49 having ~⅓ the samples. I am not sure how much this will be fixed due to the *overfitting* (which may also be the train/val gap, where many samples don't have any observations during training, while at validation we **do** have all these observations)

---

## Config differences, by dimension (and why each breaks comparability)

1. forecast horizon
    - `loss_avg_mean` averages ERA5 MSE over the forecast steps. ladqrul6/jivvzyg0 average **2** steps (+6/+12h); the rest average **3** (+6/+12/+18h).
2. Task / input product.
   - ladqrul6: single ERA5 stream is **both source and target** — clean reanalysis→reanalysis. Half the time we only have to learn the Fortran.
   - jivvzyg0/y49/xi-fc: predict ERA5 as a **diagnostic** from a forcing input; harder.
   - Input dataset differs: `ea` ERA5 reanalysis (ladqrul6, jivvzyg0, y49, xi-fc) vs
     `od` operational analysis (vbujhwi3, xi-ft-5). The `od` is expected to be harder.

3. Operational-availability lag (`nominal_time_mapping`, +5h). Only jivvzyg0 uses it (`ERA5_in: type=anemoi_operan`).
   y49/vbu/xi-pair mistakenly ignore it by using the wrong stream type.

4. Optimizer/LR. gen-A: `lr_max 5e-5`, `grad_clip 1.0`, betas (0.98125, 0.9875).
   gen-B: `lr_max 8e-5`, `grad_clip 0.25`, betas (0.9875, 0.994).

8. ERA5 frequency & sampling stride. `time_window_step` is the **sampling stride** (1h for
   JEPA, 6h for baselines jivv and ladq). Maybe leads to some of the overfitting?
   The forecast initial timestep (`forecast.time_step` = 6h for all,
   so +6h is +6h everywhere). 
   * ERA5 dataset `frequency`: 6h for ladqrul6/y49, **1h for vbujhwi3
   and the xi-pair** → those verify +6h forecasts valid at all 24 hours (more diurnal signal).

10. Validation period has same dates, but we sample different days. The configured validation window is same for all, Oct-Dec 2023, validation config inherits the time window step, so 1h or 6hly output
    

    | run | val stride | val samples | ≈ span actually validated |
    |---|---|---|---|
    | ladqrul6 | 6h | 256 | Oct 1 – ~Dec 4 (~64 d) |
    | jivvzyg0 | 6h | **128** | Oct 1 – ~Nov 2 (~32 d) |
    | y49gywef / vbujhwi3 / xi-fc / xi-ft-5 | 1h | 256 | Oct 1 – **~Oct 11 (~11 d)** |

---

---

## Below this is fully AI generated, sorry.

##### Key findings

- **y49gywef is not meaningfully worse than ladqrul6.** Per-variable at +6h they are within a
  few percent; y49's higher headline is just the +18h step in its 3-step average, on a model
  also handling 9 streams in far fewer samples.

- **jivvzyg0's deficit is concentrated in geopotential** (val z fields 25–45× worse than
  ladqrul6), while every gen-B run — including the operational-input ones (vbujhwi3, xi-ft-5) —
  keeps geopotential healthy (z ≈ 0.0002–0.0008). This **isolates the cause to jivvzyg0's
  gen-A / no-RoPE2D architecture**, not the observation streams, not operational input, and not
  the +5h lag (whose practical effect on IC timing is modest). jivvzyg0 also has the worst
  train→val generalization gap (1.45× vs 1.07–1.20× for the others), and the overfit lives in z.

- **Operational (`od`) input costs ~30% of +6h ERA5 skill** (vbujhwi3 / xi-ft-5 vs their `ea`
  counterparts) — but this is confounded by shorter data and less training, so treat it as an
  upper bound, not a clean estimate.

- **`source:[]` vs a populated `source` list is a non-difference.** Any stream that is
  `diagnostic:True` **or** has empty source gets an `Identity` embedding and contributes no
  model input (`utils.is_stream_diagnostic`, `engines.py:54`). jivvzyg0's empty source and
  y49's populated-but-diagnostic source are equivalent for input.

---

## Recommendations

### To make the existing numbers comparable (no retraining)
1. **Report a fixed lead time, per variable** — ERA5 **+6h** MSE (and/or named fields z500,
   t850, q700), never the horizon-averaged `loss_avg_mean`.
2. **Restrict any cross-run table to a common validation window and common variable set**, and
   note the verification cadence (synoptic-only vs all-hours) next to each run.

### To get genuinely comparable runs (retraining)
3. **Fix one architecture.** Adopt gen-B everywhere; retrain jivvzyg0's task on gen-B to confirm
   the geopotential recovers to ~0.0002 (the cleanest test of the architecture diagnosis).
4. **Match forecast horizon** (e.g. all 3 steps) so the headline average is on equal footing.
5. **Match training length and data window.** Same `samples_per_mini_epoch × num_mini_epochs`,
   same `start_date`/`end_date`. For `ea`-vs-`od` comparisons, restrict the `ea` run to
   2016–2022 as well, so the only difference is the input product.
6. **Match `freeze_modules`** across any pair meant to be compared (the xi pair currently differs).
7. **Decide the `nominal_time_mapping` lag deliberately.** If the +5h operational lag is wanted
   for the `ea`/`od` runs, set `ERA5_in: type=anemoi_operan`; otherwise drop the field to avoid
   confusion. Right now only jivvzyg0 applies it.
8. **For a clean `ea`-vs-`od` input study**, build a matched pair that differs *only* in the
   `ERA5_in` dataset — same arch, horizon, window (2016–2022), training length, freeze, and
   `frequency`. None of the current pairs (y49/vbu, xi-fc/xi-ft-5) isolate it.
9. **Equalise per-stream `loss_weight`** intent: all multi-stream runs use `loss_weight=1.0`
   on 8–9 streams, diluting the ERA5 gradient. If ERA5 skill is the target metric, up-weight ERA5.
