# Dataset selection criteria

Updated: 2026-09-29

## Goal

Predict the chance that **at least one part of a bus fails** within a future period, such as the next shift or 24 hours. Any component failure counts, even if the bus can still move. Predictions for individual parts are optional.

We want to develop the pipeline now and later adapt it to JUDU/operator data. We do not yet know which mechanical fields they record.

## What the dataset must contain

| Requirement | What to look for |
|---|---|
| **Suitable vehicles** | Prefer buses, coaches, minibuses or trolleybuses. Heavy road trucks are acceptable. Any country is fine. |
| **Failures across the vehicle** | Failures from several systems, or an overall failure label covering the vehicle. Records for just one component are insufficient. |
| **Mechanical measurements** | Sensor readings or diagnostic histories recorded before failures. Delays and GPS alone are insufficient. |
| **History over time** | Vehicle IDs, ordered measurements, and failure times, so we can connect earlier conditions to later failures. |
| **Normal operation too** | Periods with and without failures, with clear observation boundaries. Missing records must not be treated as proof that nothing failed. |
| **Clear, real outcomes** | Explain what counts as a failure. Keep warnings, scheduled servicing and preventive replacements separate. |
| **Accessible, documented files** | Verify actual files, their source, column meanings and license, including restrictions on sharing and commercial use. |

Useful inputs include mileage, engine hours, RPM, oil pressure, coolant temperature, brake air pressure, battery voltage, previous faults and repairs, and vehicle age/model. **Not every field is required.**

## Is it large enough?

We want to test TFT or another deep learning model. Prefer many vehicles, months or years of history, and many separate failure events across systems.

Count **vehicles and failure events**, not just rows. Millions of readings around a few breakdowns are still only a few examples of failure. No dataset size guarantees good performance.

## Can we test it fairly?

- Use only information available before each prediction.
- Keep separate vehicles, later periods and failure episodes for testing where possible; avoid overlapping training and test windows.
- Measure missed failures, false alarms, warning time, and whether predicted probabilities match actual outcomes.
- Compare deep learning with a simpler model on the same task.

## What does not meet the main goal?

- Aircraft, generic factory equipment, or synthetic toy datasets.
- Single-component datasets, including SCANIA Component X and bus DPF data.
- Current fault detection presented as future failure prediction.
- Fault-only snapshots without earlier history and normal operating periods.
- Private data described in a paper but unavailable for use. Record it as a potential access lead.

## How to assess a candidate

Record its **source/download link, access and license, vehicle type, fleet size, duration, mechanical fields, failure types/counts, and main gaps**. Inspect a sample and mark unknown facts as unknown.

Verdict: **suitable**, **partial match**, **requires access**, or **unsuitable**.

Current status: no full match has been verified. Existing leads and their limitations are in the [Gemini research prompt](GEMINI_DEEP_RESEARCH_PROMPT.md). SCANIA remains a limited benchmark. Moving to JUDU data will require field mapping, retraining and validation.
