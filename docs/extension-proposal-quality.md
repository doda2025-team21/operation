# Extension Proposal: Quality Checks for ML Models

## Shortcomings of current implementaion

While the project already has automated pipelines for training and releasing models but release quality is currently binary (success or failure depending on training success). There is no notion of "model quality" at release time. A model currently can be released even if it performs worse than the previous model, degrades latency, or behaves unstably on edge cases. 

In production-ready ML systems, the quality of ML models is equally as important to the release-engineering practices surrounding the release. This is a release-engineering concern that couples software engineering problems in ML systems.

## Proposed Extension

### Core Idea

Introduce an automatically generated quality-gate pipeline that:

1. Runs **after model training**

2. Evaluates model quality against **explicit acceptance criteria**

3. **Blocks or allows** the release automatically

### Design Overview

1. #### Quality Gate as a Pipeline Stage (Not a Script)

    Instead of embedding quality checks directly in training code, introduce a dedicated pipeline stage:

    ```bash
    train-model → evaluate-model → quality-gate → release-model
    ```
    This separation is key:

    - training answers *“can we produce a model?”*
    - quality gates answer *“is this model releasable?”*

2. #### Declarative Quality Criteria

    Define quality requirements such as accuracy, precision, F1-score, latency, etc. The metrics need to be defined based on the business goals of the model: correctness, stability, performance, failure behaviour, to name a few.
    This file must be independent of training logic, model-agnostic, version-controlled, and usable/parsable by a Github Actions pipeline. 

3. #### GitHub Action–Generated Quality Gate Pipeline

    A GitHub Action dynamically generates and executes the quality gate. The pipeline behaviour include:
    1. Load evaluation metrics produced during training

    2. Fetch metrics of the previous released model

    3. Compare metrics according to gate definitions

    4. Emit a pass/fail result

    5. Fail the workflow if gates are violated: If the gate fails, the model artifact is not released and no downstream deployment can occur. 


4. #### Automatic Baseline Comparison

    To avoid hard-coded thresholds only, the pipeline must support absolute gates (e.g. accuracy ≥ 0.92)and relative gates (e.g. no regression vs last release)
    This is critical for ML release engineering, where absolute performance plateaus.

## Implementation Plan (1-5 days)

## Expected outcomes

## Sources


[Application Release Engineering - Best Practices and Tools](https://www.xenonstack.com/insights/application-release-engineering)