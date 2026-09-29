# Gemini Deep Research prompt

Copy the prompt below into Gemini Deep Research. If file attachments are supported, also attach `DATASET_SELECTION_CRITERIA.md` from this folder.

---

Find and verify a dataset for predicting **overall bus mechanical failure risk**. Do a thorough research search, inspect primary sources and actual downloadable files, and recommend only datasets whose limitations you have checked.

## Project context

We want to develop a reusable machine learning architecture and pipeline, potentially using a Temporal Fusion Transformer (TFT) or another deep learning sequence model. We eventually intend to propose it to JUDU in Vilnius for use with local bus operators' internal data. We currently have no access to their internal data, no data dictionary, and no contact. Do not claim to know what sensors they record.

The development dataset need not be Lithuanian. The priority is a mechanically relevant road-vehicle fleet with useful measurements and real failure labels. Prefer city buses, coaches, minibuses or trolleybuses; heavy road trucks are acceptable. Include diesel, hybrid or electric fleets, explaining which vehicle types a candidate supports.

## Exact prediction objective

At prediction time t, use only observations available by t to estimate the probability that **at least one component fails** within a future horizon H. Any component failure counts toward the overall vehicle failure target; optional outputs identify component or system risks. We are not restricting the target to catastrophic or service-stopping failures, although that distinction is useful additional metadata.

Candidate horizons are the next shift, 24 hours, or seven days, depending on the event timing actually available. Current fault detection is a separate task and must not be described as future failure prediction.

A documented overall vehicle-failure flag can be sufficient even without individual component labels. The crucial requirement is broad failure coverage, not that every part has its own column. Single-component labels cannot validate whole-vehicle failure prediction.

## Dataset requirements

1. Real buses or comparable heavy road vehicles, ideally from normal fleet operation.
2. Failures across multiple systems, such as engine/cooling, transmission, brakes/pneumatics, electrical systems, doors, suspension, steering or HVAC; or a documented broad vehicle-level failure target.
3. Mechanical inputs: sensor histories, diagnostics, operational counters, usage and maintenance history. Examples include oil pressure, coolant temperature, RPM, engine load, brake pressure, charging voltage, mileage and earlier fault codes. Not every example field is required.
4. Stable vehicle identifiers, ordered measurements, meaningful timestamps or usable relative time-to-event, and histories preceding failures.
5. Normal operating periods as well as failure periods, with follow-up boundaries. A table containing only visits for faults cannot automatically support an absolute future breakdown probability.
6. Enough independent vehicles and failure events for meaningful sequence-model training and testing. Prefer hundreds/thousands of vehicles and events and months/years of history where possible, but do not invent a universal minimum. Report smaller useful candidates honestly.
7. Actual accessible files and a clear data license. Separate immediate downloads, application/request-only data, commercial offerings, and private datasets mentioned in papers.

## Exclusions and common traps

- Do not recommend aircraft datasets, including NGAFID or NASA C-MAPSS, as the primary solution.
- Do not recommend SCANIA Component X as the solution: we already have it, and it labels only one anonymized component.
- SCANIA APS does not solve this either: it distinguishes APS-related failures from other failures and is not a complete future vehicle-failure history.
- GPS, tardiness, route schedules, and NYC bus delay records alone do not satisfy the mechanical-input requirement.
- Railway compressor datasets, isolated batteries/bearings, and a bus DPF-only dataset are partial subsystem benchmarks, not whole-bus solutions.
- Reject undocumented random synthetic tables and datasets with model-generated or threshold-generated “failure_probability”/RUL labels as evidence of real predictive performance. Synthetic data may be listed separately for software testing only.
- Diagnostic warnings, repairs and planned replacements are not automatically confirmed failures. Identify the label's actual meaning.
- Do not claim data availability because a paper or GitHub notebook is public. Inspect the data files and their labels.
- Do not confuse millions of adjacent rows with millions of independent examples; count vehicles and failure episodes.
- Do not merge unrelated component datasets into fictitious vehicles or treat missing component labels as healthy states.
- Do not promise that a pretrained model will work when arbitrary JUDU data is loaded. Pipeline reuse still requires mapping, retraining, calibration and local validation.

## Where and how to search

Search primary research repositories, transportation and university data portals, supplementary files, fleet-maintenance challenges, OEM research releases, Zenodo, Mendeley Data, Figshare, Dataverse/Borealis, and original author repositories. Follow papers to their data-availability statements and datasets. Search internationally, including relevant non-English sources. Kaggle and GitHub mirrors may help discovery, but verify original provenance.

Do not stop after generic “predictive maintenance dataset” results. Look for combinations of vehicle CAN/J1939/OBD telemetry, workshop repair histories, diagnostic trouble codes, multiple systems, road calls, and normal-operation histories.

Known leads to investigate independently:

- Istanbul: “Data-Driven Fault Detection and Feature Selection from CAN-Bus Diagnostics in Commercial Vehicle,” https://doi.org/10.61969/jai.1862835. Study describes 120 buses over two years. Public sample: https://github.com/nardanesi/canbus-dataset. Our inspection found only 500 rows, no timestamps and empty alarm columns; look for a fuller release without misrepresenting this sample.
- Chinese Vehicle Fleet Fault Association Dataset: https://doi.org/10.17632/r98g7g6k5t.1. Verify actual schema, joins, pre-failure sensor history and normal observations; its description alone is insufficient.
- “Maintenance and reliability of a municipal bus fleet – a data-centered assessment”: https://doi.org/10.1186/s12544-026-00829-x. Describes 53 real buses, fault records and fuel consumption. The paper says data may be available on reasonable request. Check for a subsequent release, feature detail and training suitability; do not describe it as an existing public download.
- EngineAD: https://github.com/Armanfard-Lab/EngineAD. It appears restricted to engine anomaly segments; investigate related broader releases, but do not present it as all-system forecasting data.
- Hybrid bus DPF dataset: https://doi.org/10.17632/3sk43brs4p.1. It is a single-subsystem dataset; only a lead to potentially broader underlying fleet data.

## Verification and output

For each serious candidate, report a table with:

- Dataset title, organization, publication/release date, primary paper/DOI.
- Dataset landing page, direct download link, access status and data license.
- Real versus synthetic provenance, vehicle type, fleet size and collection duration.
- Row count, sampling frequency, sequence length, missingness and file size.
- Actual mechanical features and meaning/units; note anonymization or PCA transformation.
- Failure systems, target definitions, event timing and independent event counts per system.
- Normal exposure and observation boundaries; ability to construct future-horizon labels.
- Feasibility of vehicle-held-out and forward-in-time validation.
- Suitability for TFT or another deep learning model, and what limits that judgment.
- A verdict: main training candidate, partial benchmark, access lead, or rejected.

Open files or inspect a sample when possible. Cite evidence for consequential claims and distinguish “verified in files” from “reported by authors” and “unknown.” Do not invent missing counts, download URLs, licenses or dataset capabilities. An inaccessible file means unverified, not nonexistent.

Conclude with:

1. The best verified main dataset, if one exists, and why it meets the whole-vehicle target.
2. Up to five meaningful alternatives or access leads, with concrete missing requirements.
3. A minimal input/output schema and feasible validation design for the strongest candidate.
4. If no adequate public dataset is found, state that plainly. Identify the closest genuine bus/heavy-road-vehicle data holders and their public access procedures. You may draft a data request, but do not contact anyone, create accounts, accept agreements, or buy data.

We prefer an honest finding of “no verified full match” over another recommendation that quietly changes the task to a single component, delays, aircraft, or synthetic performance.
