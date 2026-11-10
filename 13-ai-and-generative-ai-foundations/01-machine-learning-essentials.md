# Machine Learning Essentials

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ● C ● · Status checked: 2026-11

## Why it matters

Generative AI didn't replace machine learning; it is machine learning. Every conversation about "will the model be accurate?", "do we have enough data?" or "why did it work in testing but not in production?" uses ML concepts: training vs inference, overfitting, evaluation metrics, data leakage. Engineers need them to build and evaluate systems. Architects need them to size and design. Consultants need them to set realistic expectations and to spot when a "GenAI" use case is really a classic ML problem (forecasting, churn prediction) that a cheaper model solves better.

## Key concepts

### What machine learning is

Machine learning builds a **model**, a function learned from data, that maps **inputs (features)** to **outputs (predictions)** without hand-coding the rules.

| Term | Meaning |
|---|---|
| **Feature** | An input variable (age, amount, word, pixel) |
| **Label / target** | The value to predict (fraud yes/no, price) |
| **Model** | The learned function, with **parameters** (weights) set during training |
| **Hyperparameters** | Settings chosen before training (learning rate, tree depth, number of layers) |
| **Training** | Adjusting parameters to minimise a **loss function** on training data |
| **Inference** | Using a trained model to make predictions on new data |

### Types of learning

| Type | Data | Goal | Examples |
|---|---|---|---|
| **Supervised** | Labelled examples (input + correct answer) | Predict labels for new inputs | Classification (spam, churn, document type), regression (price, demand) |
| **Unsupervised** | Unlabelled data | Find structure | Clustering (customer segments), dimensionality reduction (PCA), anomaly detection |
| **Self-supervised** | Unlabelled data where the label comes from the data itself | Learn general representations | Next-token prediction (how LLMs are pre-trained), masked-word prediction |
| **Reinforcement learning** | Rewards from an environment | Learn actions that maximise reward | Robotics, games; **RLHF** and RL fine-tuning to align and improve LLMs |
| **Semi-supervised / weak supervision** | Few labels, much unlabelled data | Use both | Labelling at scale |

### Classification vs regression vs clustering

| Task | Output | Typical algorithms |
|---|---|---|
| **Binary classification** | One of two classes | Logistic regression, gradient-boosted trees (LightGBM, XGBoost), neural networks |
| **Multiclass / multilabel** | One or more of many classes | Same, with softmax or one-vs-rest |
| **Regression** | A number | Linear regression, gradient-boosted trees, neural networks |
| **Clustering** | Group membership | k-means, DBSCAN, hierarchical clustering |
| **Time-series forecasting** | Future values | ARIMA, Prophet, gradient boosting with lag features, deep learning |

For tabular business data, **gradient-boosted trees** are often the strongest and cheapest option, frequently better than an LLM.

### Training vs inference

| | Training | Inference |
|---|---|---|
| When | Once or periodically | Every request |
| Compute | Heavy (GPUs for deep learning), batch | Lighter per call, latency-sensitive, scaled for traffic |
| Data | Historical, labelled | Live, unlabelled |
| Cost driver | GPU hours, data preparation | Requests × model size; for LLMs, **tokens** |
| On Azure | Azure Machine Learning jobs, Databricks, Fabric Data Science, Foundry fine-tuning | Managed online endpoints, Foundry Models deployments, batch endpoints |

For most generative AI projects you **don't train** a model. You run inference on a pre-trained foundation model and adapt it with prompting, retrieval (RAG) or fine-tuning (topics 16, 17).

### Data splits and generalisation

| Split | Purpose |
|---|---|
| **Training set** | Fit the parameters |
| **Validation set** | Tune hyperparameters and choose models |
| **Test set** | Final, untouched estimate of real-world performance |

**Cross-validation** rotates the validation fold for more reliable estimates on small data.

| Problem | Symptom | Fixes |
|---|---|---|
| **Overfitting** | Great on training data, poor on new data (high variance) | More data, regularisation, simpler model, early stopping, dropout |
| **Underfitting** | Poor everywhere (high bias) | More features, more capable model, train longer |
| **Data leakage** | Suspiciously high test scores; fails in production | Ensure features only use information available at prediction time; split by time or entity |
| **Distribution shift (drift)** | Performance decays after deployment | Monitor inputs and outcomes; retrain |

### Evaluation metrics

**Classification** (from the confusion matrix: TP, FP, TN, FN):

| Metric | Formula | Use when |
|---|---|---|
| **Accuracy** | (TP + TN) / all | Classes are balanced and errors cost the same |
| **Precision** | TP / (TP + FP) | False positives are expensive (flagging innocent customers) |
| **Recall** (sensitivity) | TP / (TP + FN) | False negatives are expensive (missing fraud or disease) |
| **F1 score** | 2 × (precision × recall) / (precision + recall) | You need a balance, especially with imbalanced classes |
| **ROC AUC** | Area under the TPR vs FPR curve over thresholds | Threshold-independent ranking quality |
| **PR AUC** | Area under the precision–recall curve | Heavily imbalanced data |
| **Log loss** | Penalises confident wrong probabilities | Probability quality matters |

A fraud model with 99.5% accuracy can be useless if only 0.5% of transactions are fraud: it could predict "not fraud" every time. Always look at precision and recall.

**Regression:**

| Metric | Meaning |
|---|---|
| **MAE** (mean absolute error) | Average absolute error, in the target's units; robust to outliers |
| **RMSE** (root mean squared error) | Penalises large errors more |
| **R²** (coefficient of determination) | Share of variance explained (1 is perfect, 0 is no better than the mean) |
| **MAPE** | Mean absolute percentage error; careful near zero values |

**Clustering:** silhouette score, Davies–Bouldin index, and business usefulness of the segments.

**Generative AI** adds its own metrics, such as groundedness, relevance, coherence, fluency, similarity, task completion and safety, usually measured with AI-assisted evaluators and human review (topic 22).

## How it works on Azure

| Need | Azure service |
|---|---|
| Code-first ML lifecycle (experiments, pipelines, registries, endpoints, MLOps) | **Azure Machine Learning** (topic 21) |
| Automated model selection and tuning | **Automated ML** in Azure ML |
| Notebooks and ML on lakehouse data | **Fabric Data Science**, **Azure Databricks** |
| Pre-built AI for common tasks (vision, speech, language, documents) | **Foundry Tools** (topic 20) |
| Foundation models and fine-tuning | **Microsoft Foundry** (topics 14–16) |

### A minimal classification example (scikit-learn)

```python
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.ensemble import HistGradientBoostingClassifier
from sklearn.metrics import classification_report, roc_auc_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)

model = HistGradientBoostingClassifier(max_depth=4, learning_rate=0.1)
model.fit(X_train, y_train)                       # training

proba = model.predict_proba(X_test)[:, 1]         # inference
pred = (proba >= 0.5).astype(int)                 # decision threshold is a business choice
print(classification_report(y_test, pred, digits=3))
print("ROC AUC:", round(roc_auc_score(y_test, proba), 3))
```

Change the threshold from 0.5 to 0.3 and watch recall rise and precision fall. Threshold selection is a business decision, not a technical one.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Classic ML vs LLM | Structured, tabular prediction with labelled history (churn, demand, credit risk) → classic ML. Unstructured language, documents, open-ended generation, few labels → LLMs. Often combine them: an LLM extracts features from text, and a classic model predicts |
| Metric choice | Pick the metric that reflects business cost; agree it with stakeholders before modelling |
| Simple vs complex model | Start with a simple baseline; add complexity only if it beats the baseline on held-out data |
| Interpretability | Regulated decisions (credit, hiring) need explainability (SHAP values, responsible AI dashboards) and may rule out opaque models |
| Batch vs real-time inference | Batch when predictions can be precomputed (nightly scores); real-time when the input is only known at request time |

## Hands-on

1. Run the scikit-learn example. Report precision, recall and F1 at thresholds 0.3, 0.5 and 0.7, and explain which you would choose for a medical screening use case.
2. Introduce leakage on purpose (add a feature derived from the label) and observe the unrealistically high score.
3. In Azure Machine Learning, run an **Automated ML** classification job on a public dataset and compare its best model with your baseline.
4. Write one paragraph explaining to a business stakeholder why accuracy alone is misleading for their fraud use case.

## Security, cost and reliability notes

- **Security:** training data can contain personal data; models can memorise it. Minimise and de-identify training data and control access to model artefacts.
- **Cost:** training is episodic; inference is continuous. Most lifetime cost is usually inference, so prefer smaller models that meet the metric.
- **Reliability:** monitor data drift and prediction quality in production and plan retraining. A model is a perishable asset.
- **Fairness:** check metrics per subgroup; a model can perform well overall and badly for a minority group (topic 23).

## Common mistakes

- Reporting accuracy on an imbalanced dataset.
- Tuning hyperparameters on the test set, then reporting the test score as unbiased.
- Random splits on time-series data, which leak the future into training.
- Using an LLM for a tabular prediction problem a gradient-boosted tree solves better and cheaper.
- Forgetting that production data differs from the training snapshot.

## Check yourself

1. What's the difference between training and inference?
2. Give an example each of supervised, unsupervised and reinforcement learning.
3. Define precision and recall, and give a case where recall matters more.
4. What is overfitting and how do you detect it?
5. What is data leakage?
6. Why are LLMs described as self-supervised during pre-training?
7. When would you recommend classic ML over an LLM?

<details>
<summary>Answers</summary>

1. Training learns model parameters from historical data; inference uses the trained model to make predictions on new inputs.
2. Supervised: classifying emails as spam from labelled examples. Unsupervised: clustering customers into segments. Reinforcement learning: learning a game strategy from rewards (or RLHF for LLM alignment).
3. Precision = TP/(TP+FP), the share of predicted positives that are correct. Recall = TP/(TP+FN), the share of actual positives found. Recall matters more in cancer screening or fraud detection, where missing a positive is costly.
4. The model fits training data (including noise) too closely and generalises poorly; detect it through a large gap between training and validation or test performance.
5. When information unavailable at prediction time (often derived from the label or the future) leaks into training features, inflating evaluation scores.
6. Their training labels come from the data itself: predicting the next token of text, so no human labelling is needed.
7. For structured, tabular prediction tasks with labelled history (churn, forecasting, risk scoring), where classic models are more accurate, cheaper, faster and more explainable.

</details>

## References

- [What is machine learning? (Azure)](https://azure.microsoft.com/resources/cloud-computing-dictionary/what-is-machine-learning-platform)
- [What is Azure Machine Learning?](https://learn.microsoft.com/azure/machine-learning/overview-what-is-azure-machine-learning)
- [Evaluate automated ML results (metrics explained)](https://learn.microsoft.com/azure/machine-learning/how-to-understand-automated-ml)
- [Fundamentals of machine learning (Microsoft Learn training)](https://learn.microsoft.com/training/modules/fundamentals-machine-learning/)
- [Responsible AI dashboard](https://learn.microsoft.com/azure/machine-learning/concept-responsible-ai-dashboard)
- [scikit-learn model evaluation](https://scikit-learn.org/stable/modules/model_evaluation.html)
