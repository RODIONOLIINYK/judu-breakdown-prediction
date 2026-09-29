# Dataset selection criteria for bus breakdown prediction

Updated: 2026-09-29

## Project objective

Build a reusable machine learning pipeline that estimates the probability that a bus develops **at least one component failure** within a future period, using information available at the prediction time. Any component failure counts toward the user's overall vehicle failure target. Component or system probabilities are optional additional outputs.

The eventual application is a proposal to JUDU in Vilnius. We currently have no internal JUDU/operator data, data dictionary, or contact. Their recorded mechanical fields are unknown. Development data may come from another country or operator. A transferable pipeline still needs field mapping, unit alignment, retraining, probability calibration, and validation when local data becomes available.

## Target definition

- Main target: `any_component_failure_within_horizon` for a vehicle that is operating at the prediction time.
- Candidate horizons: the next shift, 24 hours, and seven days. Choose horizons supported by actual event timing; these are design candidates, not fixed dataset requirements.
- Optional targets: the same probability for each component or system.
- A currently active fault is a separate state-detection task. Do not report detection of an existing fault as prediction of a future failure.
- Distinguish confirmed failures, diagnostic warnings, scheduled servicing, preventive replacements, and repairs whose cause is unknown. A repair is not automatically proof of a breakdown.
- Preserve whether an event stopped the vehicle or disrupted service as an additional field when available. This does not narrow the user's main target to service-stopping failures.
- Missing coverage of a component means unknown, not healthy. An overall label can be sufficient without individual component labels if its documented scope covers failures across the vehicle.

## Required properties for the main training dataset

| Criterion | Evidence needed |
|---|---|
| Relevant vehicles | Prefer city buses, coaches, minibuses, or trolleybuses; heavy road trucks are acceptable. Record powertrain and duty cycle. |
| Broad failure coverage | Recorded failures across multiple vehicle systems, or a documented vehicle-level failure target with broad coverage. A single component or subsystem does not satisfy the main objective. |
| Mechanical predictive inputs | Repeated physical measurements or diagnostic histories plausibly available before failure. GPS, delays, routes, and weather alone are insufficient for the intended mechanical model. |
| Temporal structure | Stable vehicle identifiers, ordered observation times, event times or usable relative time-to-event, and enough prior history to create prediction windows. |
| Positive and negative exposure | Both pre-failure histories and observed periods without failure. Observation boundaries must distinguish known negatives from incomplete follow-up. Fault-case snapshots alone are insufficient. |
| Usable labels | Label definitions, origin, timing, coverage, and relation to maintenance records must be documented. Evaluate warnings separately from confirmed component failure. |
| Actual access | Verify downloadable files, schema, and sample contents. A paper describing private data is an access lead, not an available training dataset. |
| Provenance | Identify the collecting organization, vehicles, period, and collection method. Separate real fleet data from simulations, injected faults, and synthetic examples. |
| Permitted use | Record the data license separately from any code license; identify restrictions relevant to research, redistribution in the public repository, and a later commercial pilot. |

Aircraft, generic manufacturing equipment, disks, and generic predictive-maintenance toy data are excluded as primary recommendations. Railway subsystem data may be discussed only as a clearly limited secondary reference, not a replacement for whole-bus data. Do not combine unrelated component datasets as if they describe the same vehicles.

## Desired input families

These are candidate fields, not a claim about JUDU's systems or a requirement that every dataset contain all of them.

| Input family | Examples |
|---|---|
| Vehicle specifications | Make/model, powertrain, age, component configuration |
| Usage | Odometer, engine hours, distance per day, idle time, starts/stops, load |
| Engine and cooling | RPM, engine load, coolant temperature, oil temperature/pressure, fuel consumption |
| Transmission | Oil temperature, selected gear, shift behaviour, diagnostic codes |
| Brakes and pneumatics | Air pressure, compressor activity, brake wear, ABS/EBS warnings |
| Electrical system | Battery/charging voltage, current, alternator or electrical fault history |
| Electric powertrain, where relevant | Battery temperature, state of charge/health, cell imbalance, motor/inverter temperature |
| Other systems | Doors, steering, suspension, HVAC, tires and their diagnostic/condition signals |
| Historical maintenance | Earlier faults, earlier repairs, time/distance since service, component replacements |
| Context | Ambient temperature, operating route and passenger/load estimates, where available |

Future repair descriptions, future downtime, and the target event's diagnostic code must not become inputs at an earlier prediction time. Previously recorded diagnostic codes may be valid features if their availability is established.

## Volume and suitability for deep learning

The pipeline should support experiments with a Temporal Fusion Transformer (TFT) or another sequence model. There is no universal row count that guarantees good performance.

Prioritize hundreds or thousands of vehicles, months to years of observation, and hundreds or thousands of independent failure events across several systems when available. These are search preferences, not established minimum sample-size thresholds. Smaller genuine datasets may be useful, but their limits must be explicit.

For each candidate report:

1. Distinct vehicles, fleet diversity, and observation duration per vehicle.
2. Sensor rows, sampling intervals, sequence lengths, missingness, and coverage gaps.
3. Distinct failure episodes overall and per system; distinguish events from repeated alarms.
4. Number of normal operating periods with adequate follow-up.
5. Number of independent vehicles and events left for validation and testing.
6. Whether any published performance used synthetic labels, random overlapping windows, or other information leakage.

Millions of rows from one vehicle or a few breakdowns do not establish fleet-level generalization. Oversampling and synthetic augmentation do not create additional real failure evidence. File size is not a measure of statistical adequacy.

## Pipeline and model comparison

Use a configurable data adapter for each source, producing linked vehicle, measurement, fault, maintenance, and observation-coverage tables. Keep the target definition, available information, prediction cutoff, and evaluation cases consistent across architectures.

Models may use different representations: summary features for trees and ordered windows for a sequence model. Inputs do not have to be identical tensors. Across different source datasets, feature mappings may differ; unknown or unavailable signals must be represented explicitly. Do not invent mappings for anonymized SCANIA features.

For the future multi-component output, the user prefers the independence baseline:

`P(any failure within H | history) = 1 - product_i(1 - p_i(H | history))`.

This requires a common horizon and a conditional independence assumption. Components can share causes; assess calibration against observed overall outcomes. This formula covers only the modelled components. Also evaluate a directly trained overall-failure output when labels support it. Do not add overlapping component probabilities as a general probability formula.

## Validation requirements

- Use vehicle-held-out evaluation where identifiers permit, and forward-in-time evaluation where timestamps permit. Report limitations when either is impossible.
- Keep all windows around a failure episode in one partition and prevent overlapping history/target windows from crossing split boundaries.
- Fit imputation, scaling, feature selection, and calibration without using the final test set.
- Treat insufficient follow-up as unknown/censored; do not silently label it as no failure.
- Evaluate event-level precision and recall, false alerts per vehicle-day, warning lead time, PR-AUC, and probability calibration/Brier score, with uncertainty estimates where feasible.
- Preserve the real event prevalence for probability evaluation. Event-enriched samples require additional calibration evidence before making everyday fleet-risk claims.
- Compare TFT/deep learning with a reasonable simpler baseline. Good accuracy on a highly imbalanced dataset is insufficient evidence.

## Candidate assessment format

For every candidate record: primary source and DOI, direct data URL, access status, license, vehicle type, real/synthetic provenance, fleet size, observation duration, sampling, feature schema, failure systems, event counts, normal exposure, label semantics, download size, split feasibility, and main limitations.

Use one of these outcomes:

- **Main training candidate:** meets the core scope and has enough verified data for a meaningful evaluation.
- **Partial benchmark:** useful for a narrower mechanical task, with the missing requirements stated.
- **Access lead:** a relevant dataset exists in a study, but usable files are not publicly available or require a request.
- **Rejected:** incompatible target, unsuitable domain, insufficient labels/history, or unreliable provenance.

Write “not reported” for unknown facts. Do not turn a promising description into a recommendation without checking the underlying files. If no full match is found, state that conclusion without claiming that such data cannot exist.

## Findings already established

- [SCANIA Component X](https://doi.org/10.1038/s41597-025-04802-6): a partial benchmark; its target records one anonymized component. It cannot establish whole-vehicle failure performance.
- [Istanbul bus CAN study](https://doi.org/10.61969/jai.1862835): a relevant access lead involving 120 buses over two years. The [public sample](https://github.com/nardanesi/canbus-dataset) was inspected: 500 rows, 56 columns, no timestamp column, and all four alarm columns empty. The sample does not support the required forecast; do not equate it with the full private study data.
- [164 hybrid diesel bus dataset](https://data.mendeley.com/datasets/3sk43brs4p/1): real bus data, but the target is duration in DPF soot zones. It remains a single-subsystem task.
- [EngineAD](https://github.com/Armanfard-Lab/EngineAD): commercial-vehicle engine anomaly detection, with PCA-transformed signals and segment labels. A partial benchmark, not verified broad vehicle failure forecasting data.
- [Chinese Vehicle Fleet Fault Association Dataset](https://data.mendeley.com/datasets/r98g7g6k5t/1): potentially relevant fault/repair records. Full suitability remains unverified; examine vehicle linkage, continuous pre-failure history, and normal exposure before recommending it.
- [Autosan municipal bus fleet study](https://doi.org/10.1186/s12544-026-00829-x): an access lead involving 53 buses, fault records and fuel consumption. The paper says data may be available from the authors upon reasonable request. No full public download was verified; mechanical sensor histories and suitability for deep learning remain unverified.

Current decision: no replacement dataset has yet been verified against all core requirements. The existing SCANIA files are retained as previously downloaded, not endorsed as the final dataset for the broader target.
