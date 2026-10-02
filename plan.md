# JUDU implementation plan

## Goal and starting assumptions

Build a reproducible experiment that predicts **Component X repair probability within 30 days**, using past readouts and vehicle specifications. [README.md](README.md) contains the project facts, architecture and dataset limitations; this document describes the work to do and why.

- Use the project assumption **0.2 dataset units = 1 day** consistently.
- Start with **GRU + attention pooling + specification embeddings + PC-Hazard**.
- Work locally on the MacBook Air M5. Consider Azure only after measuring a local bottleneck.
- Keep raw data unchanged. The first version excludes repeated repair histories, which this dataset does not provide.

## Implementation steps: what to do and why

### 1. Establish a small, reproducible project

Create separate data preparation, model, training and evaluation modules. Record dependencies and put model settings in one configuration file. Ignore generated caches and checkpoints in Git.

**Why:** preprocessing and model settings must be reproducible; temporary outputs should not clutter the repository.

### 2. Separate fitting and evaluation vehicles

Split the official training vehicles into **80% fitting and 20% internal validation**, using seed 42 and stratifying by repair indicator. Save the vehicle IDs.

Keep the supplied validation and test datasets separate.

**Why:** the internal holdout provides exact repair/censoring endpoints for survival evaluation. The supplied validation labels provide event-window benchmarks. Keeping vehicles separate prevents histories of the same vehicle appearing on both sides.

### 3. Prepare numerical and categorical inputs

Sort readouts by vehicle and timestamp; convert times and endpoints to days.

Construct the agreed **228-feature readout vector**: 105 original values, 8 counter rates, 113 missingness flags and 2 timing values. Preserve histogram positions and irregular intervals.

Calculate rates before scaling. Mark first-record, missing and negative/reset-like rates as unavailable. Fit median imputation and standard scaling on fitting vehicles only. Map specification categories to integer IDs, reserving an unknown category.

**Why:** the model needs consistent numerical scales, explicit missingness and elapsed time. Specification IDs will select learned embedding rows.

### 4. Build and inspect training examples

For each prediction point, include only readouts up to that point:

```text
Inputs:
  history           [L, 228]
  specifications    [8 category IDs]

Targets:
  remaining_days    endpoint_days − prediction_days
  event_flag        1 = observed repair; 0 = censored
```

Use prediction points strictly before the endpoint. Sample up to four uniformly selected eligible points per fitting vehicle each epoch. Fix internal-validation prediction points using seed 42.

Average losses within each vehicle, then across vehicles.

**Why:** this simulates prediction from partial histories without leaking future information or letting long histories dominate.

**First milestone:** manually verify one observed-repair example and one censored example.

### 5. Implement batching and the model

Pad histories within each batch and retain their true lengths. Use packed sequences for GRU processing and mask padding during attention pooling.

Start with:

- One GRU layer with **64 hidden features**.
- Learned attention scores followed by masked softmax and weighted pooling.
- Eight embedding tables with **4 coordinates per category**, giving 32 specification features.
- Concatenation into 96 features, then a 64-feature dense layer with ReLU and dropout 0.1.
- A survival head producing positive hazards through softplus.

**Why:** variable-length histories become fixed-width summaries, while both input branches learn from the same prediction loss.

### 6. Implement censoring-aware learning

Use one-day hazard intervals covering the fitting follow-up range; extend the final hazard constantly when evaluating longer follow-up.

Calculate the loss through integrated hazard to avoid numerical underflow:

```text
loss = integrated_hazard(u) − event_flag × log(hazard(u))
risk within 30 days = 1 − exp(−integrated_hazard(30))
```

**Why:** censored examples contribute their known repair-free observation period without asserting that their unknown future is healthy. Daily output intervals do not require daily sensor measurements.

### 7. Run a small training check, then the full experiment

First confirm finite losses, working gradients and decreasing training loss on a small subset containing both outcome types.

Then train with seed 42, batch size 16 and Adam at learning rate 0.001. Allow up to 50 epochs, stopping after five epochs without internal-validation loss improvement. Save the best checkpoint, preprocessing and configuration.

Use MPS when supported; otherwise use CPU.

**Why:** a small run catches pipeline mistakes cheaply. Validation determines when to stop rather than training loss alone.

### 8. Evaluate, compare and expose predictions

On the internal holdout, report survival loss, censoring-aware Brier scores and concordance. Assess 30-day ranking and calibration while handling censored horizon outcomes correctly.

Use official validation classes for their supplied event-window benchmark. Compare the initial model with:

1. The same GRU using its final state instead of attention pooling.
2. The same architecture without counter rates.

Keep splits, evaluation examples and training settings consistent. Open test labels only after selecting the approach.

Provide an inference entry point accepting a vehicle history and specifications, returning vehicle ID, prediction time and 30-day repair probability.

**Why:** these comparisons establish whether pooling and engineered rates help. A repeatable inference path makes the trained experiment usable.

## Required checks and completion criteria

- No vehicle overlap between fitting and evaluation partitions.
- No future readings or target endpoints included as input features.
- Correct time conversion, missingness flags and reset handling.
- Histories of different lengths batch correctly; padding does not change predictions.
- Censored examples contribute loss and gradients.
- Survival probabilities remain between zero and one and decrease over time.
- Saved preprocessing and checkpoints reproduce predictions.
- Evaluation clearly distinguishes supplied benchmark labels from exact survival outcomes.

The first version is complete when preparation, training, evaluation and inference run reproducibly, with recorded results and limitations. Its results describe the SCANIA component benchmark; suitability for JUDU buses remains a later validation task.

**Current status:** this is the implementation roadmap. No pipeline has been built or trained yet.
