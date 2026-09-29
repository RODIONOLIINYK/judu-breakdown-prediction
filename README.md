# JUDU breakdown prediction

## Project goal

The main goal of this project is to develop a machine learning model that estimates how likely a bus is to break down within a specified future period, using features extracted from vehicle specifications and operational history. The intended outcome is a probability that helps a fleet operator prioritize inspections and plan maintenance before a vehicle fails.

The first stage uses the **SCANIA Component X Dataset** to develop and evaluate the prediction method. This dataset contains SCANIA **trucks** and events for one anonymized engine component. Its repair and replacement records serve as failure labels. Applying the model to JUDU buses will require relevant bus data and validation; performance on this dataset alone cannot establish accuracy for bus breakdowns or failures of other components. [Dataset paper](https://doi.org/10.1038/s41597-025-04802-6)

## What the model should predict

For a vehicle at its latest available readout, estimate the probability of a Component X event within a chosen prediction horizon, using only information available at that readout. Start with horizons of 6, 12, 24, and 48 dataset time steps so the results can be compared with the supplied evaluation labels.

An illustrative output would be: “Vehicle 123 has an estimated 18% probability of a Component X event within the next 24 time steps.” This is an example of the intended output, not a model result.

Time is anonymized in this dataset. Time steps should not be presented as hours or days without a verified conversion. Vehicles without an observed repair are censored: their eventual failure time is unknown. [Technical paper](https://arxiv.org/abs/2401.15199)

## Dataset files

The project contains official release **3**, published April 9, 2025, under **CC BY 4.0**. This release includes test labels and the final published article. The nine CSV files are kept in their original format. [Official dataset record](https://researchdata.se/en/catalogue/dataset/2024-34)

All paths below are relative to this project folder:

```text
README.md
data/scania_component_x/
  raw/
    train_operational_readouts.csv
    train_specifications.csv
    train_tte.csv
    validation_operational_readouts.csv
    validation_specifications.csv
    validation_labels.csv
    test_operational_readouts.csv
    test_specifications.csv
    test_labels.csv
  docs/
    2024_IDA_challenge_v2.pdf
    Scania_Component_X.pdf
  download_manifest.json
```

Operational tables provide repeated numerical counters and histogram bins. Specifications provide vehicle categories. `train_tte.csv` supplies observation duration and repair status; validation and test labels describe proximity to failure. Tables connect through `vehicle_id`. [Technical paper](https://arxiv.org/abs/2401.15199)

The download manifest records each file's source URL, byte size, and locally calculated SHA-256 checksum.

## Get the project from GitHub

The three large operational tables are stored with Git LFS. Install [Git LFS](https://git-lfs.com/), then clone the repository and retrieve the full dataset:

```sh
git lfs install
git clone https://github.com/RODIONOLIINYK/judu-breakdown-prediction.git
cd judu-breakdown-prediction
git lfs pull
```

The download is approximately 1.54 GiB. If Git LFS is unavailable, the manifest links directly to every original file in the official dataset release.

## Proposed development approach

1. Inspect missing values, observation intervals, counter resets, and label distributions.
2. Extract features such as the latest readings, counter changes per time step, historical trends, normalized histogram distributions, and vehicle categories. Treat anonymized feature meanings as unknown.
3. Train a baseline probability model and compare it with a survival model that accounts for censoring. Exclude future readings and target information from input features. For horizon-based training, handle observations whose follow-up ends before the horizon as censored rather than assigning them a negative label.
4. Preserve the supplied vehicle splits. Fit preprocessing on training data, tune and calibrate probabilities using validation data, and reserve the test set for final evaluation.
5. Evaluate probability calibration, Brier score, precision and recall, and the maintenance cost of missed failures and unnecessary inspections. Use censoring-aware metrics where needed and assess each prediction horizon separately.
6. Validate on representative bus records before using predictions to support JUDU maintenance decisions.

Current status: dataset and project definition prepared. Model development has not started.

## Sources and attribution

- [Official dataset, version 3 — Lindgren et al.](https://doi.org/10.5878/bnh5-ka77)
- [Scientific Data publication — Kharazian et al. (2025)](https://doi.org/10.1038/s41597-025-04802-6)
- [arXiv technical paper](https://arxiv.org/abs/2401.15199)
- [Dataset license: Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/)
- [Related survival modelling repository](https://github.com/pboneza/ssl-survival-scania-component-x). The originally supplied shorter GitHub URL returned 404; this is the available repository. Dataset files were downloaded from the official research data service.
