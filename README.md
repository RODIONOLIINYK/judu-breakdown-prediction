# JUDU breakdown prediction

Predict maintenance risk from vehicle histories and specifications. The first experiment uses **GRU + attention pooling + specification embeddings + a survival head**. Dataset inspection is complete; preprocessing, training and evaluation are not implemented yet.

This README is the project reference. [plan.md](plan.md) gives the implementation order, the reason for each step and the completion checks.

## Scope and time convention

The current benchmark is **SCANIA Component X, release 3**: trucks and the first repair/replacement of one anonymized component. A repair is the target event, not proof of a roadside breakdown. Predicting failures across JUDU buses will require bus data, broader event labels and fleet validation.

**Project assumption: 0.2 dataset units = 1 day.** This is our chosen interpretation, not an author-confirmed conversion. Keep raw files unchanged and convert all readout times, observation endpoints and intervals consistently:

```text
time_days = time_step × 5
initial prediction horizon = 30 days = 6 dataset units
```

Timestamps represent elapsed time since operation began, not readout numbers. Preserve irregular intervals; do not invent daily sensor readings. Operational values are anonymized and scaled, so their physical meanings remain unknown.

Inspection found timestamps on a 0.2-unit grid, a median readout gap of 4.4 units and a most frequent gap of 6.0 units. Under our assumption, these become 1-day resolution, a 22-day median gap and a 30-day most frequent gap. Calendar-like intervals motivated the assumption; changes in anonymized counter values do not independently confirm it.

Training observation endpoints have a median of 218.2 units and a maximum of 510 units: **1,091 and 2,550 assumed days** since operation began. These are endpoint ages, not necessarily the span between the first and last recorded readouts.

## Data

Join by `vehicle_id` and order each vehicle's readouts by `time_step`.

| Files | Contents |
|---|---|
| `{train,validation,test}_operational_readouts.csv` | 105 numerical fields: 8 cumulative counters and 97 bins across 6 histograms. |
| `{train,validation,test}_specifications.csv` | 8 categorical specification fields. |
| `train_tte.csv` | Observation endpoint (`length_of_study_time_step`) and repair indicator (`in_study_repair`). |
| `{validation,test}_labels.csv` | Event-proximity classes at the final available readout. |

Training contains **23,550 vehicles, 1,122,452 readouts, 2,272 observed repairs and 21,278 censored vehicles**. Complete histories have a median of 43 readouts and a maximum of 303. Validation/test use different vehicles and truncated histories.

Histogram bins already occupy fixed columns; retain every position, including zeros. Bin boundaries and measured quantities are unknown. The dataset has no full history of repeated repairs, servicing or replacements; do not fabricate these inputs.

The counter columns are `171_0`, `666_0`, `427_0`, `837_0`, `309_0`, `835_0`, `370_0` and `100_0`. Histogram groups and widths are **167: 10, 272: 10, 291: 11, 158: 10, 459: 20, 397: 36**. Bin values are not necessarily counts of sensor observations; they can represent accumulated quantities.

Inspection found all `171_0` values on a 15-unit grid and histogram-total changes for groups 272 and 158 close to a 13:12 ratio. These encoding patterns do not identify physical units or justify treating either feature as a clock.

Observed training category counts for `Spec_0` through `Spec_7` are **3, 29, 21, 4, 2, 5, 17, 9**. Category names have no known physical or ordinal meaning.

## Inputs and model

At prediction time `t`, supply **all available readouts at or before `t`**, plus the vehicle's specifications. Exclude future readings and the target repair.

The initial readout vector has **228 features**:

- **105** original numerical values.
- **8** optional counter rates: adjacent change / elapsed **days**.
- **113** missingness flags for values and rates, distinguishing missing data from measured zeros.
- **2** timing values: elapsed days since operation began and days since the previous readout.

Compute rates before scaling. Mark first-record rates, missing endpoints and negative/reset-like increments as unavailable. Fit numerical imputation and scaling only on training vehicles; retain the missingness flags. Compare models with and without rates.

Without rates and their eight missingness flags, the comparison model has **212 input features**. Rates are optional engineering features; GRU can also learn changes from the original sequence.

```text
Readouts [L × 228] → one-layer GRU → saved states [L × 64]
                                  → attention pooling → history [64]
8 specification fields → 8 learned embedding tables → specifications [32]

Concatenate [64 + 32] → dense layers → PC-Hazard survival head
                                    → repair probability within 30 days
```

`L` varies: 48 and 200 readouts work with the same model. Use true sequence lengths for GRU batching and mask padding during pooling. Start with four embedding coordinates per specification field and reserve an unknown-category entry. Both branches and the head learn together through one loss.

An embedding is a trainable lookup table, not a separate pretrained model. For example, `Spec_0` has three known categories; adding an unknown category gives `nn.Embedding(4, 4)`. Backpropagation updates the selected embedding rows along with the GRU and prediction head.

**Start with attention pooling, without self-attention.** Pooling learns nonnegative weights summing to one and takes a weighted sum of saved states. It can emphasize earlier states without comparing every pair of records. Compare it against the final GRU state alone; consider transformer/TFT experiments if evaluation justifies them.

Pooling produces a new summary rather than adding a correction to the final state. Attention does not guarantee perfect memory or establish causal relationships between sensors. This compact GRU is the first experiment because it suits the local compute budget; relative performance must be measured.

## Training and outputs

One example is a vehicle history ending at an eligible readout with raw timestamp `t`, strictly before its observation endpoint. Initially sample up to four eligible readouts per vehicle per training pass, rotating them without selecting by future outcomes. Average losses within vehicles, then across vehicles, so long histories do not dominate.

```text
u = (observation endpoint − t) × 5 days
δ = 1 for an observed repair; 0 for observation ending without repair
```

Use positive piecewise-constant repair rates, initially in **one-day hazard bins** covering the training follow-up range. This defines output resolution, not sensor sampling frequency. With survival probability `S(u)` and repair rate `λ(u)`:

```text
loss = −log S(u) − δ log λ(u)
risk within H days = 1 − S(H)
```

Observed repairs contribute both loss terms. Censored vehicles contribute `−log S(u)`: their known repair-free follow-up is useful, while their later outcome remains unknown.

The planned head uses softplus for positive hazards and integrated hazard for stable loss calculation. Beyond the fitted interval range, extend the final hazard constantly; predictions there are extrapolations.

For a 30-day prediction, a repair within the horizon is positive. A known later repair or repair-free follow-up through 30 days is negative. Censoring before 30 days leaves the horizon outcome unknown; survival training still uses its observed follow-up. Optional time-to-repair quantiles come from the survival curve and are not exact breakdown dates.

## Evaluation and compute

Keep whole vehicles separate between fitting and evaluation. Create a vehicle-level holdout within training data for tuning and exact event/censoring-time metrics; fit preprocessing only on the fitting partition. Use supplied validation labels for benchmark checks and reserve test labels for final evaluation.

Official validation histories stop at a randomly selected prediction readout; later readouts are withheld. Their labels give event-time ranges, not exact repair or censoring endpoints. This is why the survival experiment also needs an internal holdout with targets from `train_tte.csv`.

Native boundaries **6, 12, 24 and 48 units** become **30, 60, 120 and 240 assumed days**. Evaluate ranking, precision/recall, calibration and maintenance cost. Use censoring-aware Brier scores and concordance where exact follow-up is available. Multiple sampled histories do not create additional independent repairs.

Start on the **MacBook Air M5** with 64 GRU hidden features, small batches and cached prepared sequences. Use PyTorch MPS if supported, otherwise CPU. Measure runtime and memory before expanding the model. Azure is optional if local limits become a bottleneck; keep any future cloud experiment within the available **$200** budget.

For a future JUDU dataset, add timestamped component repairs, servicing, mileage and engine hours where available. Learn how replacements affect subsequent risk from records rather than assuming that replaced parts are less reliable. Tree-boosting models are outside the current experiment; transformer/TFT comparisons are deferred.

## Repository and setup

```text
README.md                                     Project facts and current decisions
plan.md                                       Implementation steps and reasons
data/scania_component_x/raw/                   Original nine CSVs
data/scania_component_x/docs/                  Official PDFs
data/scania_component_x/download_manifest.json URLs, sizes and SHA-256 checksums
docs/DATASET_SELECTION_CRITERIA.md              Future bus-dataset requirements
```

Operational files use Git LFS (approximately 1.54 GiB in total):

```sh
git lfs install
git clone https://github.com/RODIONOLIINYK/judu-breakdown-prediction.git
cd judu-breakdown-prediction
git lfs pull
```

The download manifest provides original file URLs if Git LFS is unavailable. This README records the current modelling plan; training commands will be added when the model is implemented.

## Sources

- [Official SCANIA Component X dataset, release 3](https://researchdata.se/en/catalogue/dataset/2024-34/3), licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- [Dataset paper — Kharazian et al., Scientific Data (2025)](https://doi.org/10.1038/s41597-025-04802-6).
- [PC-Hazard methodology — Kvamme and Borgan](https://arxiv.org/abs/1910.06724).
