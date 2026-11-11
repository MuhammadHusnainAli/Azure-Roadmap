# Automated ML

## Why it matters

Many predictive problems don't need a bespoke model architecture: they need a solid baseline fast. **Automated ML (AutoML)** in Azure ML tries many algorithms, featurisation steps and hyperparameters in parallel, ranks them on your chosen metric, and produces a deployable model with explanations. It's how analysts and engineers get a credible churn, demand or defect model in hours, and how data scientists establish a baseline before custom work. It's also the Microsoft-recommended successor for **Custom Vision** image classification and detection ([Topic 20 file 02](../20-foundry-tools/02-azure-vision.md)).

## Key concepts

### Supported tasks

| Task | Examples | Notes |
|---|---|---|
| **Classification** | Fraud, churn, defect yes/no | Class balancing, deep neural network featurisation for text columns |
| **Regression** | Price, demand quantity | |
| **Time-series forecasting** | Sales, energy demand, inventory | Holiday featurisation, lags, rolling windows, rolling-origin cross-validation, many-models/grouping, learners such as Auto-ARIMA, Prophet, ForecastTCN |
| **Computer vision** | Multi-class and multi-label image classification, object detection, instance segmentation | SDK-based; integrates with data labelling |
| **NLP** | Text classification (multi-class, multi-label), named entity recognition | BERT-based models, 104 languages, distributed training |

### How it works

1. Choose the task and the **primary metric** (for example AUC weighted, normalised RMSE).
2. Provide labelled training data (`MLTable` for tabular), optional validation and test data.
3. Configure limits (trials, timeout, concurrency), allowed/blocked algorithms, featurisation and ensembling.
4. AutoML runs many trials in parallel (pipelines of featurisation + algorithm + hyperparameters).
5. Review the leaderboard, metrics, explanations and the best model (a pickled pipeline including preprocessing); register or deploy it.

### Featurisation and guardrails

AutoML automatically handles missing values, categorical encoding, scaling and text featurisation, and runs **data guardrails** (class imbalance, missing values, high cardinality, leakage hints). You can customise or disable featurisation.

### Validation

| Option | Use |
|---|---|
| Train/validation split or cross-validation | Default model selection |
| Rolling-origin cross-validation | Forecasting |
| **Test data** | Unbiased evaluation of the recommended model (Preview) |

### Explainability and RAI

Best models include feature importance; you can generate a **Responsible AI dashboard** for error analysis and fairness (file 10).

## How it works on Azure

Tabular classification (Python SDK v2):

```python
from azure.ai.ml import MLClient, Input, automl
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), "<sub>", "rg-ml", "mlw-churn-dev")
job = automl.classification(
    experiment_name="churn-automl",
    training_data=Input(type="mltable", path="azureml:churn-mltable:1"),
    target_column_name="churned",
    primary_metric="AUC_weighted",
    n_cross_validations=5,
    enable_model_explainability=True)
job.set_limits(max_trials=40, max_concurrent_trials=4, timeout_minutes=90, enable_early_termination=True)
job.set_training(blocked_training_algorithms=["KNN"], enable_onnx_compatible_models=True)
returned = ml.jobs.create_or_update(job)     # serverless if no compute given
print(returned.studio_url)
```

Forecasting adds `automl.forecasting(...)` with `set_forecast_settings(time_column_name=..., forecast_horizon=..., time_series_id_column_names=[...])`. Vision tasks use `automl.image_classification`, `automl.image_object_detection` and so on, typically on GPU compute.

No-code alternative: **Automated ML** in studio walks through data, task, settings and compute.

## Design decisions and trade-offs

### When AutoML is good enough

| Good fit | Poor fit |
|---|---|
| Tabular data with a clear target | Novel architectures or research-grade models |
| Need a strong baseline quickly | Complex custom loss functions or constraints |
| Teams with limited data science capacity | Very large-scale deep learning beyond supported tasks |
| Forecasting many related series | Strict latency/size constraints needing hand-optimised models (though ONNX export helps) |
| Replacing Custom Vision | Precise control over every training detail |

### Other choices

| Decision | Guidance |
|---|---|
| **Metric** | Match the business cost (AUC weighted for imbalance; precision/recall at threshold; normalised RMSE for forecasts). |
| **Budget** | Limit trials and time; enable early termination; often 20–50 trials find most of the gain. |
| **Explainability** | Turn it on for regulated use; review top features for leakage. |
| **ONNX** | Enable ONNX-compatible models for portable, fast inference. |
| **AutoML vs LLM** | For tabular prediction, AutoML/classic ML usually beats prompting an LLM on accuracy, cost and determinism. |

## Hands-on

1. Register a churn CSV as an `MLTable` data asset.
2. Run the classification job above; inspect the leaderboard and data guardrails.
3. Open the best model's explanations; check for leakage (for example a "cancellation_date" column).
4. Remove the leaking column and rerun; compare metrics.
5. Optional: run an AutoML image classification job on labelled product photos as a Custom Vision migration test.

## Security, cost and reliability notes

- **Data access** via identity-based datastores; sensitive columns minimised before training.
- **Cost**: trials × compute; vision and NLP need GPUs; set limits.
- **Fairness and bias**: AutoML optimises a metric, not fairness; assess with the RAI dashboard.
- **Reproducibility**: record data version, settings and best run ID; register the model with tags.

## Common mistakes

| Mistake | Fix |
|---|---|
| Accepting the top model without checking leakage | Review explanations and features |
| Accuracy on imbalanced data | Use AUC weighted or precision/recall |
| No test set | Hold out test data |
| Unlimited trials | Set limits and early termination |
| Random splits for time series | Use forecasting task with rolling-origin CV |

## Check yourself

1. Which task types does AutoML support?
2. What does AutoML automate?
3. Name three forecasting-specific features.
4. What are data guardrails?
5. When is AutoML not the right choice?
6. Why is AutoML relevant to Custom Vision customers?

<details>
<summary>Answers</summary>

1. Classification, regression, time-series forecasting, computer vision (image classification, object detection, instance segmentation) and NLP (text classification, NER).
2. Algorithm selection, featurisation, hyperparameter tuning, validation and ensembling.
3. Any three: holiday featurisation, lags, rolling window aggregates, rolling-origin cross-validation, grouping/many models, Auto-ARIMA/Prophet/ForecastTCN learners.
4. Automatic checks on data issues such as class imbalance, missing values and high cardinality.
5. For novel architectures, custom losses, research-grade deep learning or when every training detail must be controlled.
6. Custom Vision is retiring (September 2028) and Microsoft recommends AutoML for custom image classification and object detection.

</details>

## References

- [What is automated ML?](https://learn.microsoft.com/azure/machine-learning/concept-automated-ml)
- [Set up AutoML training for tabular data](https://learn.microsoft.com/azure/machine-learning/how-to-configure-auto-train)
- [Set up AutoML for time-series forecasting](https://learn.microsoft.com/azure/machine-learning/how-to-auto-train-forecast)
- [Set up AutoML for computer vision](https://learn.microsoft.com/azure/machine-learning/how-to-auto-train-image-models)
