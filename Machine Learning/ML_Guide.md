# Machine Learning — Basics to Expert
## Complete Guide with Examples, Code, and Explanations

> 12 core ML concepts from first principles through advanced topics. Every concept explained in plain English with Python code, real-world examples, and practical notes.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| [1](#1-what-is-machine-learning) | What is Machine Learning? | Beginner |
| [2](#2-supervised-learning) | Supervised Learning | Beginner |
| [3](#3-unsupervised-learning) | Unsupervised Learning | Intermediate |
| [4](#4-regression) | Regression | Beginner |
| [5](#5-classification) | Classification | Beginner |
| [6](#6-decision-trees-and-ensemble-methods) | Decision Trees and Ensemble Methods | Intermediate |
| [7](#7-clustering) | Clustering (K-Means) | Intermediate |
| [8](#8-neural-networks) | Neural Networks | Advanced |
| [9](#9-deep-learning) | Deep Learning (CNN, RNN, Transformers) | Expert |
| [10](#10-model-evaluation) | Model Evaluation | Intermediate |
| [11](#11-overfitting-and-underfitting) | Overfitting and Underfitting | Intermediate |
| [12](#12-advanced-topics-and-roadmap) | Advanced Topics and Learning Roadmap | Expert |

---

## 1. What is Machine Learning?

### The core idea

Traditional programming: you write explicit rules. Machine learning: you give the machine examples and it learns the rules itself.

```
Traditional:  Data  +  Rules  →  Output
ML:           Data  +  Output →  Rules  (the model learns them)
```

> "Instead of writing a rule 'if has fins AND breathes underwater → fish', you show 10,000 pictures of fish and the model figures out what makes a fish."

### Three types of ML

| Type | What it needs | What it learns | Example |
|------|--------------|----------------|---------|
| **Supervised** | Labelled examples (input + answer) | Mapping from inputs to outputs | Spam detection, house price prediction |
| **Unsupervised** | Unlabelled data only | Hidden patterns and structure | Customer segmentation, anomaly detection |
| **Reinforcement** | Rewards and penalties | Actions that maximise reward | Game-playing AI, robotics |

### The ML pipeline — every project follows these steps

```
Collect data → Clean data → Choose model → Train → Evaluate → Tune → Deploy
```

Each step matters equally. Garbage data produces garbage models regardless of how sophisticated the algorithm is. Data scientists typically spend 60–80% of their time on data collection and cleaning.

---

## 2. Supervised Learning

### What it is

You provide input-output pairs (training data). The model learns a mapping function `f(x) = y` so it can predict outputs for new, unseen inputs.

> "Like a student studying past exam papers with answer keys. After seeing 1,000 examples, they can answer new questions they've never seen."

### How it works

```
Training: (house size, price), (house size, price), (house size, price)...
              ↓
Model learns: price ≈ size × Weight + Bias
              ↓
New input: 1,500 sqft → predicts £320,000
```

### Two flavours

| Regression | Classification |
|-----------|---------------|
| Output is a continuous number | Output is a discrete category |
| Examples: house price, temperature, salary | Examples: spam/not spam, cat/dog, disease/healthy |

### Python example

```python
from sklearn.linear_model import LinearRegression

# Training data: house size → price
X_train = [[800], [1000], [1200], [1500], [2000]]
y_train  = [160000, 200000, 240000, 300000, 400000]

model = LinearRegression()
model.fit(X_train, y_train)

# Predict a new house
model.predict([[1300]])
# Output: [260000]  — the model learned: price ≈ 200 × sqft
```

### Real-world examples

- Gmail spam filter
- Netflix and Spotify recommendations
- Google Translate
- Medical diagnosis from X-rays
- Credit card fraud detection

---

## 3. Unsupervised Learning

### What it is

You only give inputs — no correct answers. The model discovers structure, groupings, or relationships on its own.

> "Like sorting a box of mixed foreign coins into groups. You've never seen them before, but you group them by size, colour, and markings — no one told you the categories."

### Four main techniques

**Clustering** — group similar items together. Customer segmentation, document grouping, gene expression analysis.

**Dimensionality reduction** — compress 100 features into 2–3 while preserving the most information. Used for visualisation (PCA, t-SNE) and speeding up other algorithms.

**Anomaly detection** — learn what "normal" looks like, then flag anything unusual. Fraud detection, manufacturing defect detection, network intrusion detection.

**Association rules** — "people who buy X also buy Y." Market basket analysis drives "customers also bought" recommendations in e-commerce.

### Python example

```python
from sklearn.cluster import KMeans
import numpy as np

# Customer data: [spend_per_visit, visits_per_month]
customers = np.array([
    [10, 20], [12, 18], [9, 22],    # group A: frequent, low spend
    [80,  2], [90,  1], [75,  3],   # group B: rare, high spend
    [40,  8], [45,  9], [38, 10],   # group C: moderate
])

kmeans = KMeans(n_clusters=3, random_state=42)
kmeans.fit(customers)

print(kmeans.labels_)
# [0, 0, 0, 1, 1, 1, 2, 2, 2]  — three groups found automatically
```

---

## 4. Regression

### What it is

Regression finds the mathematical relationship between input features and a continuous output value.

### Types of regression

| Algorithm | When to use |
|-----------|-------------|
| **Linear regression** | When relationship is approximately linear |
| **Polynomial regression** | When the data follows a curve |
| **Ridge regression** | Linear + L2 regularisation to prevent overfitting |
| **Lasso regression** | Linear + L1 regularisation — also performs feature selection |
| **Logistic regression** | Despite the name, this is for classification (outputs a probability) |

### How to measure error

```python
# Mean Squared Error (MSE) — lower is better
predictions = [260000, 310000, 195000]
actuals      = [265000, 300000, 210000]

errors = [(p - a)**2 for p, a in zip(predictions, actuals)]
mse = sum(errors) / len(actuals)
# MSE = 116,666,667
# Root MSE (RMSE) ≈ £10,800 — average error per prediction
```

### Real-world regression examples

- Predicting house prices from size, location, number of rooms
- Forecasting energy consumption from temperature and time
- Estimating delivery time from distance and traffic data
- Predicting exam scores from study hours

---

## 5. Classification

### What it is

Given input features, predict which category the input belongs to. Binary classification has two classes; multi-class has many.

### Logistic regression for classification

Despite the name, logistic regression is a classifier. It outputs a probability between 0 and 1 via the sigmoid function, then applies a threshold.

```
Input features → Linear combination → Sigmoid → Probability → Class label
                                     (0 to 1)    (e.g. 0.85)   (spam)
```

### The threshold matters

- High threshold (0.8): fewer false alarms but misses more positives — good for spam filters
- Low threshold (0.3): catches more positives but more false alarms — good for fraud alerts

### Python example

```python
from sklearn.linear_model import LogisticRegression

# Email features: [word_count, exclamation_marks, has_link, caps_ratio]
X_train = [
    [120, 1, 0, 0.05],  # legitimate
    [ 30, 5, 1, 0.40],  # spam
    [200, 0, 0, 0.02],  # legitimate
    [ 15, 8, 1, 0.60],  # spam
]
y_train = [0, 1, 0, 1]  # 0=not spam, 1=spam

clf = LogisticRegression()
clf.fit(X_train, y_train)

# Predict a new email
clf.predict([[50, 6, 1, 0.45]])            # → [1]  spam!
clf.predict_proba([[50, 6, 1, 0.45]])      # → [0.15, 0.85]  85% spam
```

### Other classifiers

- **K-Nearest Neighbours (KNN)** — classify by looking at the K nearest training examples
- **Support Vector Machine (SVM)** — find the hyperplane that maximally separates classes
- **Naive Bayes** — fast, probabilistic, great for text classification

---

## 6. Decision Trees and Ensemble Methods

### Decision trees — if-then rules learned from data

A tree splits data at each node based on the feature that best separates the classes. The result is a flowchart of yes/no questions.

```
Salary > £30,000?
    ├── Yes → Employed > 2 years?
    │   ├── Yes → APPROVE ✓
    │   └── No  → Credit score > 650?
    │       ├── Yes → APPROVE ✓
    │       └── No  → REJECT ✗
    └── No  → REJECT ✗
```

> "Like a flowchart a doctor uses: 'Does the patient have a fever? → Yes: does the patient have a cough? → Yes: likely flu.'"

### The weakness: single trees overfit

A deep decision tree memorises the training data and performs poorly on new data. Ensemble methods fix this.

### Random forest — the workhorse

Build 100+ decision trees, each trained on a random subset of the data and features. Combine their votes by majority. More accurate and robust than any single tree.

```python
from sklearn.ensemble import RandomForestClassifier

# Loan features: [salary, years_employed, credit_score]
X = [[35000,3,720], [28000,1,580], [55000,8,800], [22000,0,500]]
y = [1, 0, 1, 0]  # 1=approve, 0=reject

rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X, y)

# Feature importance
for name, importance in zip(['salary','years','credit'], rf.feature_importances_):
    print(f"{name}: {importance:.2f}")
# salary: 0.42   years: 0.31   credit: 0.27
```

### Gradient boosting (XGBoost) — the competition winner

Build trees sequentially — each new tree learns to correct the errors of the previous trees. State of the art for structured/tabular data. Wins the majority of Kaggle competitions on structured data.

```python
from xgboost import XGBClassifier
xgb = XGBClassifier(n_estimators=100, learning_rate=0.1, max_depth=4)
xgb.fit(X_train, y_train)
```

---

## 7. Clustering (K-Means)

### What it is

K-Means partitions data into K groups where each data point belongs to the cluster with the nearest mean (centroid).

### Algorithm — step by step

1. **Initialise** — place K centroids randomly in the data space
2. **Assign** — assign each point to its nearest centroid (Euclidean distance)
3. **Move** — recalculate each centroid as the mean of all its assigned points
4. **Repeat** — repeat steps 2–3 until centroids stop moving (convergence)

### Choosing K — the elbow method

```python
import matplotlib.pyplot as plt
from sklearn.cluster import KMeans

inertias = []
K_range  = range(1, 11)

for k in K_range:
    km = KMeans(n_clusters=k, random_state=42)
    km.fit(X)
    inertias.append(km.inertia_)

plt.plot(K_range, inertias, 'o-')
plt.xlabel('K'); plt.ylabel('Inertia')
plt.title('Elbow Method — pick the bend in the curve')
# The "elbow" point is the optimal K
```

### Other clustering algorithms

| Algorithm | Strength |
|-----------|----------|
| **K-Means** | Fast, scalable, works well with round clusters |
| **DBSCAN** | Finds arbitrarily shaped clusters, handles noise/outliers |
| **Hierarchical** | Builds a dendrogram, no need to specify K in advance |
| **Gaussian Mixture** | Probabilistic — each point has soft membership in clusters |

---

## 8. Neural Networks

### What it is

Layers of interconnected nodes (neurons) inspired loosely by the brain. Each connection has a weight. Training adjusts the weights so the network makes better predictions.

> "Like a chain of filters — early layers detect simple features (edges), later layers detect complex abstractions (faces)."

### A single neuron

```
Inputs × Weights → Σ (sum) + Bias → Activation function → Output
x₁×w₁ + x₂×w₂ + x₃×w₃ + b → ReLU(result) → activated output
```

### Activation functions

| Function | Formula | Used for |
|----------|---------|----------|
| **ReLU** | max(0, x) | Hidden layers — most common |
| **Sigmoid** | 1 / (1 + e⁻ˣ) | Binary output layer |
| **Softmax** | eˣᵢ / Σeˣ | Multi-class output — probabilities sum to 1 |
| **Tanh** | (eˣ - e⁻ˣ) / (eˣ + e⁻ˣ) | Recurrent networks |

### How training works (backpropagation)

1. **Forward pass** — data flows through layers → prediction made
2. **Loss** — measure how wrong the prediction is (e.g. cross-entropy loss)
3. **Backpropagation** — compute gradient of loss with respect to each weight (chain rule)
4. **Gradient descent** — adjust weights in the direction that reduces loss
5. **Repeat** — for many batches of data (epochs)

### Python example with Keras

```python
import tensorflow as tf

model = tf.keras.Sequential([
    tf.keras.layers.Dense(64, activation='relu', input_shape=(10,)),
    tf.keras.layers.Dense(32, activation='relu'),
    tf.keras.layers.Dropout(0.3),           # prevent overfitting
    tf.keras.layers.Dense(1, activation='sigmoid'),  # binary output
])

model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)

model.fit(X_train, y_train, epochs=50, batch_size=32, validation_split=0.2)
```

---

## 9. Deep Learning

### What it is

"Deep" means many layers — sometimes hundreds. Each layer learns increasingly abstract representations of the input.

### CNN — Convolutional Neural Networks (images)

```
Input image → Conv layer (detect edges) → Pooling →
Conv layer (detect shapes) → Dense → Softmax (class probabilities)
```

```python
model = tf.keras.Sequential([
    tf.keras.layers.Conv2D(32, (3,3), activation='relu', input_shape=(28,28,1)),
    tf.keras.layers.MaxPooling2D(2, 2),
    tf.keras.layers.Conv2D(64, (3,3), activation='relu'),
    tf.keras.layers.MaxPooling2D(2, 2),
    tf.keras.layers.Flatten(),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(10, activation='softmax'),  # 10 digit classes
])
# Achieves ~99.2% accuracy on MNIST handwritten digits
```

### RNN / LSTM — Recurrent Networks (sequences)

Recurrent networks have memory — each output depends on the current input AND the previous hidden state. LSTMs solve the vanishing gradient problem with gating mechanisms. Used for text, audio, and time series.

### Transformers — the architecture that changed everything

"Attention is All You Need" (2017) introduced self-attention: every token can attend to every other token simultaneously. Parallelisable unlike RNNs. Powers GPT, Claude, BERT, Whisper. Dominates NLP and increasingly vision and audio.

### GAN — Generative Adversarial Networks

Two networks compete: a generator creates fake images, a discriminator tries to spot fakes. The generator improves until its outputs are indistinguishable from real ones. Creates photorealistic images, deepfakes, and artwork.

---

## 10. Model Evaluation

### Why accuracy alone is misleading

On a dataset where 1% of patients have cancer, a model that always predicts "no cancer" achieves 99% accuracy — but catches zero cancer cases. Use the right metric for your problem.

### The confusion matrix

```
                 Predicted Positive    Predicted Negative
Actual Positive    True Positive (TP)   False Negative (FN)
Actual Negative    False Positive (FP)  True Negative (TN)
```

### Key metrics

```python
from sklearn.metrics import classification_report, roc_auc_score

print(classification_report(y_test, y_pred))
# Output:
#               precision  recall  f1-score  support
# Class 0          0.92    0.96      0.94      500
# Class 1          0.89    0.81      0.85      200

# Precision = TP / (TP + FP)  — of predicted positives, how many are correct?
# Recall    = TP / (TP + FN)  — of actual positives, how many did we catch?
# F1 Score  = 2 × (P × R) / (P + R)  — harmonic mean of precision and recall
# ROC-AUC   — overall classifier quality across all thresholds
```

### Which metric to use?

| Situation | Best metric | Reason |
|-----------|------------|--------|
| Spam filter | Precision | False alarms (ham in spam) are annoying |
| Cancer detection | Recall | Missing a case (false negative) is dangerous |
| Balanced classes | F1 score | Equally penalises precision and recall errors |
| Class imbalance | ROC-AUC | Robust to imbalanced class distributions |
| Regression | RMSE / MAE | Measures average prediction error |

### Cross-validation — robust evaluation

```python
from sklearn.model_selection import cross_val_score

scores = cross_val_score(model, X, y, cv=5, scoring='f1')
print(f"Mean F1: {scores.mean():.3f} ± {scores.std():.3f}")
# Each fold uses different train/test split — reliable performance estimate
```

---

## 11. Overfitting and Underfitting

### The most important problem in ML

> "Overfitting is like a student who memorises every exam question word-for-word but can't answer a rephrased version. Underfitting is the student who didn't study at all."

### The bias-variance tradeoff

```
Complexity:   Low ─────────────────────► High
              Underfitting        Overfitting
              High bias           High variance
              Low variance        Low bias
                    ↑
              Sweet spot: good generalisation
```

### Diagnosing the problem

```python
# Plot learning curves to diagnose
from sklearn.model_selection import learning_curve

train_sizes, train_scores, val_scores = learning_curve(
    model, X, y, cv=5,
    train_sizes=np.linspace(0.1, 1.0, 10)
)

# Overfitting:   train accuracy high, val accuracy low
# Underfitting:  both accuracies low
# Good fit:      both accuracies similar and high
```

### Solutions

**For overfitting:**
- Add more training data — harder to memorise a million examples
- Regularisation — L1 (Lasso) or L2 (Ridge) penalises large weights
- Dropout — randomly disable neurons during training (neural networks)
- Early stopping — stop training when validation error starts rising
- Reduce model complexity — fewer layers or features

**For underfitting:**
- Increase model complexity — add more layers or features
- Train longer — more epochs, lower learning rate
- Feature engineering — create more informative input features
- Reduce regularisation — give the model more freedom

---

## 12. Advanced Topics and Roadmap

### Transfer learning

Take a model pre-trained on millions of examples (ImageNet, large corpora) and fine-tune it on your small dataset. Instead of training from scratch, you reuse learned representations. Reduces training time from weeks to hours.

```python
from tensorflow.keras.applications import VGG16

# Load pre-trained model (exclude top classification layer)
base_model = VGG16(weights='imagenet', include_top=False, input_shape=(224,224,3))
base_model.trainable = False  # freeze pre-trained weights

# Add your own classification head
model = tf.keras.Sequential([
    base_model,
    tf.keras.layers.GlobalAveragePooling2D(),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(5, activation='softmax'),  # your 5 classes
])
```

### Reinforcement learning

An agent learns by interacting with an environment — takes actions, receives rewards or penalties. Trained AlphaGo to beat world Go champions. Powers robotics, game AI, and RLHF in large language models.

### Attention and Transformers

Self-attention lets every token look at every other token simultaneously. Unlike RNNs, processing is fully parallelisable. "Attention is All You Need" (Vaswani et al., 2017) is the most influential ML paper of the decade. All major LLMs (GPT, Claude, Gemini, LLaMA) are based on the Transformer.

### Hyperparameter tuning

```python
import optuna

def objective(trial):
    lr  = trial.suggest_float('lr', 1e-5, 1e-1, log=True)
    n   = trial.suggest_int('n_estimators', 50, 500)
    model = RandomForestClassifier(n_estimators=n, random_state=42)
    return cross_val_score(model, X, y, cv=3).mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=50)
print("Best params:", study.best_params)
```

### MLOps — taking ML to production

- **Serving** — FastAPI or Flask REST endpoint wrapping your model
- **Model versioning** — MLflow or DVC tracks experiments and model versions
- **Drift detection** — monitor when real-world data diverges from training distribution
- **A/B testing** — compare model versions on live traffic
- **Retraining pipelines** — automated retraining when performance degrades

### Learning roadmap

| Stage | Tools and topics | Time investment |
|-------|-----------------|-----------------|
| **1 — Foundations** | Python, NumPy, Pandas, Matplotlib | 4–6 weeks |
| **2 — Classical ML** | scikit-learn: regression, classification, clustering, evaluation | 6–8 weeks |
| **3 — Deep Learning** | TensorFlow or PyTorch, CNNs, RNNs | 8–12 weeks |
| **4 — Practice** | Kaggle competitions, real datasets, personal projects | Ongoing |
| **5 — Advanced** | Hugging Face Transformers, LLMs, MLOps, cloud deployment | 3–6 months |

---

## Quick Reference — All 12 Topics

| Topic | One-line definition | Key algorithm |
|-------|---------------------|---------------|
| **What is ML** | Machine learns rules from examples, not programmed explicitly | — |
| **Supervised** | Learn from labelled input-output pairs | Linear regression, SVM |
| **Unsupervised** | Find hidden patterns in unlabelled data | K-Means, PCA |
| **Regression** | Predict a continuous number | Linear, Ridge, Lasso |
| **Classification** | Predict a discrete category | Logistic regression, Random Forest |
| **Decision trees** | Learn if-then rules from data | CART, XGBoost |
| **Clustering** | Group similar data points | K-Means, DBSCAN |
| **Neural networks** | Layered nodes that learn by adjusting weights | MLP, backpropagation |
| **Deep learning** | Many-layer networks for images, text, audio | CNN, LSTM, Transformer |
| **Evaluation** | Measure whether the model actually works | Precision, recall, F1, AUC |
| **Overfitting** | Model memorises training data, fails on new data | Regularisation, dropout |
| **Advanced** | Transfer learning, RL, Transformers, MLOps | Fine-tuning, attention |

---

*Python 3.10+ · scikit-learn 1.4 · TensorFlow 2.x / PyTorch 2.x · Verified examples*
