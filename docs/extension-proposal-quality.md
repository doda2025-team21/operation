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

1. **_Define Quality Gate Metrics:_** Define the quality metrics that should be used to evaluate the ML model based on business use case. The quality metrics could be decision of thresholds, relative regression checks to a baseline model, etc. Schema ould be done implemented in `quality-gates.yaml` file.

2. **_Extend Output to Provide Metrics:_** We need to extend the existing training model workflow to export evaluation metrics and inference metrics (that can be scraped/used by the evaluation pipeline) in-order to make model quality objectively measurable. The extension should not affect model behaviour. Modification must output a `metrics.json` file that can be parsed. 

3. **_Implement Quality Gate Action:_** This task involves building the steps that will parse the metrics, compare them to the existing baseline or thresholds, and succeed/fail depending on the result of the comparison. It should emit a failure message if the quality of the model is below par, and succeed if the quality of the model is satisfied. This could be implemented using a `Github Actions` that would:
    - Loads `quality-gates.yaml`
    - Loads current model metrics
    - Loads previous release metrics
    - Evaluates each gate

4. **_Integrate the steps into Release Pipeline:_** Insert the quality step before model release to ensure only quality-approved models are released. Make all the release steps depend on passing gate and the workflow needs to be updated with configuring the dependencies correctly. 

5. **_Testing the Gates' Validity (Optional):_** We could perform synthetic gate validation by adding a dedicated job in the workflow that bypasses training and injects synthetic metrics (to evaluate good and bad models). These synthetic metrics are evaluated by quality-gate logic and we'd be able to test the functionality and correctness of the quality gates.

6. **_Documentation & Visualization:_** As a final addition to measure and visualize the metrics: model quality at release, release blocked or allowed, which metric caused failure, etc. This allows us decipher the reason for failure and interpret the results. This can be done by exposing metrics in Prometheus, consequently gathering and visualizing in Grafana (additional dashboard).

## Expected outcomes

### Release Engineering Improvements
| Before                            | After                                     |
| --------------------------------- | ----------------------------------------- |
| Any trained model can be released | Only quality-approved models are released |
| Quality evaluated post-hoc        | Quality enforced pre-release              |
| Silent regressions possible       | Regressions blocked automatically         |
| Release = artifact exists         | Release = artifact meets policy           |

### How to Measure the Effect (Continuous Experimentation)

#### Hypothesis

Automated quality gates reduce the number of low-quality models entering deployment.

#### Measurement

Compare the ML model releases before vs after introducing gates. We need to draw conclusions from how many times the release pipeline rejected a release based on the quality gate failing. This directly measures release quality. 


## Sources

[Application Release Engineering - Best Practices and Tools](https://www.xenonstack.com/insights/application-release-engineering)

[Azure Machine Learning model monitoring](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring?view=azureml-api-2)

[Evaluating machine learning models: Establishing quality gates](https://www.codecentric.de/en/knowledge-hub/blog/evaluating-machine-learning-models-quality-gates)
