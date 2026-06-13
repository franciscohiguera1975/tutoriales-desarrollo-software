# Módulo 07 — Clasificación: Naive Bayes, Decision Tree y Random Forest

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M06 (SVM y Kernel SVM)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Naive Bayes — Clasificación probabilística](#naive-bayes)
3. [Decision Tree — Árboles de clasificación](#decision-tree)
4. [Random Forest — Bosques aleatorios](#random-forest)
5. [Gradient Boosting y XGBoost para clasificación](#gradient-boosting-y-xgboost)
6. [Comparativa final de todos los clasificadores](#comparativa-final)
7. [Guía de selección de clasificador](#guía-de-selección)
8. [Checkpoint M07](#checkpoint-m07)

---

## Cómo ejecutar este módulo

> Consulta M01 para la guía completa de entornos.

### Opción A — Script local

```bash
conda activate ml-clasico
python modulo-07-naive-bayes-dt-rf.py
```

### Opción B — Jupyter Notebook

```python
# Primera celda
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
# Primera celda
!pip install -q xgboost
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Naive Bayes

### ¿Qué es Naive Bayes?

Naive Bayes aplica el **teorema de Bayes** con la suposición ("naive") de que todas las features son **independientes entre sí** dado la clase:

```
P(clase | X) ∝ P(clase) × P(X₁|clase) × P(X₂|clase) × ... × P(Xₙ|clase)
               ─────────   ───────────────────────────────────────────────
               prior       likelihood (producto porque asumimos independencia)
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
from matplotlib.colors import ListedColormap
import warnings
warnings.filterwarnings('ignore')

# ── Datasets ──────────────────────────────────────────────────────────────────
from sklearn.datasets import (load_breast_cancer, load_iris, fetch_20newsgroups,
                               make_classification)
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_validate, GridSearchCV)
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.pipeline import Pipeline
from sklearn.metrics import (accuracy_score, f1_score, roc_auc_score,
                              classification_report, ConfusionMatrixDisplay)

# Dataset principal: Breast Cancer
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
print(f"Dataset: {X.shape} | Train: {len(X_train)} | Test: {len(X_test)}")
```

### Las 3 variantes de Naive Bayes

| Variante            | Distribución asumida           | Uso típico                               |
|---------------------|--------------------------------|------------------------------------------|
| `GaussianNB`        | Normal (continua)              | Features continuas (datos numéricos)     |
| `MultinomialNB`     | Multinomial (conteos)          | Clasificación de texto (bag of words)    |
| `BernoulliNB`       | Bernoulli (binaria 0/1)        | Texto con presencia/ausencia de palabras |
| `ComplementNB`      | Complemento de MultinomialNB   | Texto desbalanceado                      |

```python
from sklearn.naive_bayes import GaussianNB, MultinomialNB, BernoulliNB, ComplementNB

# ── 1. GaussianNB — datos continuos ──────────────────────────────────────────
gnb = GaussianNB()
gnb.fit(X_train, y_train)
y_pred_gnb = gnb.predict(X_test)
acc_gnb = accuracy_score(y_test, y_pred_gnb)
print(f"\nGaussianNB Accuracy: {acc_gnb:.4f}")
print(classification_report(y_test, y_pred_gnb, target_names=cancer.target_names))

# El prior estimado automáticamente
print("Prior de clases estimado por GaussianNB:")
for cls, prior in zip(cancer.target_names, gnb.class_prior_):
    print(f"  {cls}: P(clase) = {prior:.3f}")

# ── Ajuste del var_smoothing (evita P=0 por división por cero) ───────────────
from sklearn.model_selection import GridSearchCV

param_gnb = {'var_smoothing': np.logspace(-9, 0, 20)}
grid_gnb  = GridSearchCV(GaussianNB(), param_gnb, cv=5, scoring='roc_auc')
grid_gnb.fit(X_train, y_train)
print(f"\nMejor var_smoothing: {grid_gnb.best_params_['var_smoothing']:.2e}")
print(f"Mejor AUC CV:        {grid_gnb.best_score_:.4f}")
```

### Visualización de distribuciones Gaussianas por clase

```python
# ── Distribuciones por clase (primeras 4 features) ───────────────────────────
fig, axes = plt.subplots(2, 2, figsize=(12, 8))
axes = axes.ravel()
feature_names = cancer.feature_names

for i, ax in enumerate(axes):
    for cls_idx, cls_name in enumerate(cancer.target_names):
        mask = y_train == cls_idx
        datos = X_train[mask, i]
        mu, sigma = gnb.theta_[cls_idx, i], np.sqrt(gnb.var_[cls_idx, i])
        x_range = np.linspace(datos.min(), datos.max(), 200)
        from scipy.stats import norm
        pdf = norm.pdf(x_range, mu, sigma)
        ax.hist(datos, bins=20, alpha=0.4, density=True, label=f'{cls_name} (datos)')
        ax.plot(x_range, pdf, lw=2, label=f'{cls_name} N(μ={mu:.2f}, σ={sigma:.2f})')
    ax.set_title(feature_names[i], fontsize=9)
    ax.legend(fontsize=7)

plt.suptitle('GaussianNB — Distribuciones aprendidas por clase', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m07_gnb_distribuciones.png', dpi=120)
plt.show()
```

### Naive Bayes para texto (MultinomialNB)

```python
# ── NB para clasificación de texto: 20 Newsgroups ────────────────────────────
from sklearn.datasets import fetch_20newsgroups
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer

# 4 categorías para simplificar
categorias = ['sci.space', 'sci.med', 'rec.sport.hockey', 'comp.graphics']
newsgroups = fetch_20newsgroups(subset='train', categories=categorias,
                                 remove=('headers', 'footers', 'quotes'))
news_test  = fetch_20newsgroups(subset='test',  categories=categorias,
                                 remove=('headers', 'footers', 'quotes'))

print(f"\nNewsgroups: {len(newsgroups.data)} docs train | {len(news_test.data)} test")
print(f"Clases: {newsgroups.target_names}")

# MultinomialNB con CountVectorizer (bag of words)
pipe_mnb = Pipeline([
    ('vect', CountVectorizer(max_features=10000, stop_words='english')),
    ('clf',  MultinomialNB(alpha=0.1))   # alpha = Laplace smoothing
])
pipe_mnb.fit(newsgroups.data, newsgroups.target)
acc_mnb = pipe_mnb.score(news_test.data, news_test.target)
print(f"MultinomialNB + CountVectorizer Accuracy: {acc_mnb:.4f}")

# ComplementNB generalmente supera a MultinomialNB en texto
pipe_cnb = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=10000, stop_words='english')),
    ('clf',   ComplementNB(alpha=0.1))
])
pipe_cnb.fit(newsgroups.data, newsgroups.target)
acc_cnb = pipe_cnb.score(news_test.data, news_test.target)
print(f"ComplementNB + TF-IDF Accuracy:           {acc_cnb:.4f}")

# Predicción de nuevo texto
nuevos = ["The shuttle launch was delayed due to fuel issues",
          "The doctor prescribed penicillin for the infection"]
preds = pipe_mnb.predict(nuevos)
for txt, pred in zip(nuevos, preds):
    print(f"  '{txt[:50]}...' → {newsgroups.target_names[pred]}")
```

---

## Decision Tree

### ¿Cómo divide el árbol?

Un árbol de decisión divide el espacio de features recursivamente eligiendo el **split que maximiza la pureza** de los hijos:

```
Métricas de impureza:
  Gini    = 1 − Σ pᵢ²          (más rápido, default en sklearn)
  Entropía = −Σ pᵢ log₂(pᵢ)   (conceptualmente más limpio)

La ganancia de información = impureza_padre − Σ (peso_hijo × impureza_hijo)
```

```python
from sklearn.tree import DecisionTreeClassifier, export_text, plot_tree

# ── Árbol simple y visualizable (usando Iris) ─────────────────────────────────
iris = load_iris()
X_ir, y_ir = iris.data, iris.target

X_tr_ir, X_te_ir, y_tr_ir, y_te_ir = train_test_split(
    X_ir, y_ir, test_size=0.25, stratify=y_ir, random_state=42
)

dt_iris = DecisionTreeClassifier(max_depth=3, criterion='gini', random_state=42)
dt_iris.fit(X_tr_ir, y_tr_ir)
print(f"\nDecision Tree (Iris, depth=3) Accuracy: {dt_iris.score(X_te_ir, y_te_ir):.4f}")

# Representación de texto del árbol
print("\nEstructura del árbol:")
print(export_text(dt_iris, feature_names=list(iris.feature_names)))

# Visualización gráfica
fig, ax = plt.subplots(figsize=(18, 8))
plot_tree(dt_iris, feature_names=iris.feature_names,
          class_names=iris.target_names, filled=True,
          rounded=True, fontsize=9, ax=ax)
plt.title('Decision Tree (Iris, max_depth=3)', fontsize=14)
plt.tight_layout()
plt.savefig('outputs/m07_dt_iris.png', dpi=120)
plt.show()
```

### Control de complejidad del árbol

```python
# ── Efecto de max_depth sobre accuracy y overfitting ─────────────────────────
depths = range(1, 21)
train_scores, test_scores = [], []

for d in depths:
    dt = DecisionTreeClassifier(max_depth=d, random_state=42)
    dt.fit(X_train, y_train)
    train_scores.append(accuracy_score(y_train, dt.predict(X_train)))
    test_scores.append(accuracy_score(y_test, dt.predict(X_test)))

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

axes[0].plot(depths, train_scores, 'b-o', ms=4, label='Train')
axes[0].plot(depths, test_scores, 'r-o', ms=4, label='Test')
axes[0].axvline(depths[np.argmax(test_scores)], color='green',
                 ls='--', label=f'Óptimo depth={depths[np.argmax(test_scores)]}')
axes[0].set_xlabel('max_depth')
axes[0].set_ylabel('Accuracy')
axes[0].set_title('Efecto de max_depth (Breast Cancer)')
axes[0].legend()

# Importancia de features
best_depth = depths[np.argmax(test_scores)]
dt_best = DecisionTreeClassifier(max_depth=best_depth, random_state=42)
dt_best.fit(X_train, y_train)

importancias = dt_best.feature_importances_
idx_sorted   = np.argsort(importancias)[::-1][:10]

axes[1].barh(range(10), importancias[idx_sorted], color='steelblue')
axes[1].set_yticks(range(10))
axes[1].set_yticklabels(cancer.feature_names[idx_sorted])
axes[1].set_xlabel('Importancia (Gini)')
axes[1].set_title(f'Top 10 features — DT (depth={best_depth})')
axes[1].invert_yaxis()

plt.tight_layout()
plt.savefig('outputs/m07_dt_depth_importancia.png', dpi=120)
plt.show()

print(f"\nMejor depth: {best_depth}")
print(f"  Train Accuracy: {train_scores[best_depth-1]:.4f}")
print(f"  Test  Accuracy: {test_scores[best_depth-1]:.4f}")
```

### Hiperparámetros principales del árbol

```python
# ── GridSearch completo para DecisionTree ────────────────────────────────────
param_dt = {
    'max_depth':        [3, 5, 7, 10, None],
    'min_samples_split': [2, 5, 10, 20],
    'min_samples_leaf':  [1, 2, 5, 10],
    'criterion':         ['gini', 'entropy'],
}

grid_dt = GridSearchCV(
    DecisionTreeClassifier(random_state=42), param_dt,
    cv=StratifiedKFold(5, shuffle=True, random_state=42),
    scoring='f1', n_jobs=-1
)
grid_dt.fit(X_train, y_train)

print(f"\nMejores parámetros DT: {grid_dt.best_params_}")
print(f"Mejor F1 CV: {grid_dt.best_score_:.4f}")
print(f"Test F1:     {f1_score(y_test, grid_dt.predict(X_test)):.4f}")
```

### Poda del árbol (Cost-Complexity Pruning)

```python
# ── Poda post-entrenamiento: ccp_alpha ────────────────────────────────────────
# Encuentra el rango de alphas óptimos
path = DecisionTreeClassifier(random_state=42).cost_complexity_pruning_path(X_train, y_train)
ccp_alphas = path.ccp_alphas

# Evalúa varios alphas
acc_train_ccp, acc_test_ccp, n_nodes = [], [], []
for alpha in ccp_alphas[::5]:  # submuestreo para rapidez
    dt_ccp = DecisionTreeClassifier(ccp_alpha=alpha, random_state=42)
    dt_ccp.fit(X_train, y_train)
    acc_train_ccp.append(dt_ccp.score(X_train, y_train))
    acc_test_ccp.append(dt_ccp.score(X_test, y_test))
    n_nodes.append(dt_ccp.tree_.node_count)

fig, axes = plt.subplots(1, 2, figsize=(13, 4))
alphas_sub = ccp_alphas[::5]

axes[0].plot(alphas_sub, acc_train_ccp, 'b-o', ms=4, label='Train')
axes[0].plot(alphas_sub, acc_test_ccp, 'r-o', ms=4, label='Test')
axes[0].set_xlabel('ccp_alpha')
axes[0].set_ylabel('Accuracy')
axes[0].set_title('Cost-Complexity Pruning')
axes[0].legend()

axes[1].plot(alphas_sub, n_nodes, 'g-o', ms=4)
axes[1].set_xlabel('ccp_alpha')
axes[1].set_ylabel('Número de nodos')
axes[1].set_title('Complejidad del árbol vs ccp_alpha')

plt.tight_layout()
plt.savefig('outputs/m07_dt_pruning.png', dpi=120)
plt.show()

# Mejor alpha
mejor_alpha = alphas_sub[np.argmax(acc_test_ccp)]
print(f"\nMejor ccp_alpha: {mejor_alpha:.6f}")
```

---

## Random Forest

### ¿Por qué un bosque supera a un árbol?

```
Árbol único:
  Ventaja: muy fácil de interpretar
  Problema: alta varianza (sobreajusta fácilmente)

Random Forest (n árboles con bootstrap + feature subsampling):
  Idea: "Wisdom of crowds" — muchos árboles mediocres pero independientes
        → su promedio es mucho más estable

  Dos fuentes de aleatoriedad:
  1. Bootstrap: cada árbol se entrena en una muestra con reemplazamiento
     (~63.2% muestras únicas, ~36.8% OOB — Out-of-Bag)
  2. Feature subsampling: en cada split solo se consideran √n_features features
     (reduce la correlación entre árboles)
```

```python
from sklearn.ensemble import (RandomForestClassifier, GradientBoostingClassifier,
                               ExtraTreesClassifier, BaggingClassifier)

# ── Efecto del número de árboles ──────────────────────────────────────────────
n_estimators_list = [1, 5, 10, 25, 50, 100, 200, 500]
acc_train_rf, acc_test_rf, oob_scores = [], [], []

for n in n_estimators_list:
    rf = RandomForestClassifier(n_estimators=n, oob_score=(n >= 5),
                                  random_state=42, n_jobs=-1)
    rf.fit(X_train, y_train)
    acc_train_rf.append(rf.score(X_train, y_train))
    acc_test_rf.append(rf.score(X_test, y_test))
    oob_scores.append(rf.oob_score_ if hasattr(rf, 'oob_score_') and n >= 5 else np.nan)

fig, ax = plt.subplots(figsize=(10, 5))
ax.plot(n_estimators_list, acc_train_rf, 'b-o', ms=5, label='Train')
ax.plot(n_estimators_list, acc_test_rf, 'r-o', ms=5, label='Test')
ax.plot(n_estimators_list, oob_scores, 'g--s', ms=5, label='OOB Score')
ax.set_xlabel('n_estimators')
ax.set_ylabel('Accuracy')
ax.set_title('Random Forest — Efecto del número de árboles (Breast Cancer)')
ax.legend()
ax.set_xscale('log')
plt.tight_layout()
plt.savefig('outputs/m07_rf_n_estimators.png', dpi=120)
plt.show()
print("Nota: el score se estabiliza — más árboles no sobreajustan pero tampoco mejoran indefinidamente")
```

### Importancia de features en Random Forest

```python
# ── Feature importance: Mean Decrease Impurity (MDI) vs Permutation ───────────
from sklearn.inspection import permutation_importance

rf_100 = RandomForestClassifier(n_estimators=100, oob_score=True,
                                  random_state=42, n_jobs=-1)
rf_100.fit(X_train, y_train)

print(f"\nRandom Forest (100 árboles):")
print(f"  Train Accuracy: {rf_100.score(X_train, y_train):.4f}")
print(f"  Test  Accuracy: {rf_100.score(X_test, y_test):.4f}")
print(f"  OOB   Score:    {rf_100.oob_score_:.4f}")

# MDI — Mean Decrease in Impurity (rápido pero puede sobreestimar cardinales altas)
mdi_imp = rf_100.feature_importances_

# Permutation Importance (más robusto, más lento)
perm_result = permutation_importance(rf_100, X_test, y_test,
                                       n_repeats=10, random_state=42, n_jobs=-1)
perm_imp = perm_result.importances_mean

fig, axes = plt.subplots(1, 2, figsize=(14, 6))
n_top = 10

for ax, importancias, titulo in zip(
    axes,
    [mdi_imp, perm_imp],
    ['MDI (Mean Decrease Impurity)', 'Permutation Importance']
):
    idx = np.argsort(importancias)[::-1][:n_top]
    ax.barh(range(n_top), importancias[idx], color='steelblue')
    ax.set_yticks(range(n_top))
    ax.set_yticklabels(cancer.feature_names[idx])
    ax.set_title(f'RF — {titulo}')
    ax.set_xlabel('Importancia')
    ax.invert_yaxis()

plt.tight_layout()
plt.savefig('outputs/m07_rf_importancia.png', dpi=120)
plt.show()

# Las features más importantes según ambas métricas
top_mdi  = set(cancer.feature_names[np.argsort(mdi_imp)[::-1][:5]])
top_perm = set(cancer.feature_names[np.argsort(perm_imp)[::-1][:5]])
print(f"\nTop 5 MDI:  {top_mdi}")
print(f"Top 5 Perm: {top_perm}")
print(f"Coincidencia: {top_mdi & top_perm}")
```

### Hiperparámetros clave de Random Forest

```python
# ── GridSearch para Random Forest ────────────────────────────────────────────
from sklearn.model_selection import RandomizedSearchCV

param_rf = {
    'n_estimators':      [50, 100, 200],
    'max_depth':         [None, 5, 10, 20],
    'min_samples_split': [2, 5, 10],
    'min_samples_leaf':  [1, 2, 4],
    'max_features':      ['sqrt', 'log2', 0.5],  # sqrt es el default para clasificación
    'bootstrap':         [True, False],
}

rand_rf = RandomizedSearchCV(
    RandomForestClassifier(random_state=42, n_jobs=-1),
    param_rf, n_iter=30,
    cv=StratifiedKFold(5, shuffle=True, random_state=42),
    scoring='roc_auc', random_state=42, n_jobs=-1
)
rand_rf.fit(X_train, y_train)

print(f"\nRF RandomizedSearch — Mejores parámetros:")
for k, v in rand_rf.best_params_.items():
    print(f"  {k}: {v}")
print(f"Mejor AUC CV: {rand_rf.best_score_:.4f}")

y_proba_rf = rand_rf.predict_proba(X_test)[:, 1]
print(f"Test AUC-ROC: {roc_auc_score(y_test, y_proba_rf):.4f}")
print(f"Test F1:      {f1_score(y_test, rand_rf.predict(X_test)):.4f}")
```

### ExtraTrees — aún más aleatorio

```python
# ── ExtraTreesClassifier: umbral de split aleatorio (más rápido) ──────────────
et = ExtraTreesClassifier(n_estimators=100, random_state=42, n_jobs=-1)
et.fit(X_train, y_train)
print(f"\nExtraTrees Accuracy: {et.score(X_test, y_test):.4f}")
print("(ExtraTrees elige el threshold de split al azar → más rápido que RF)")
```

---

## Gradient Boosting y XGBoost

```
Random Forest: árboles en PARALELO — cada árbol es independiente
Gradient Boosting: árboles en SERIE — cada árbol corrige los errores del anterior

Algoritmo (simplificado):
  1. Inicializa con predicción constante (media de y)
  2. Para t = 1..T:
     a. Calcula los residuos rᵢ = yᵢ − F_{t-1}(xᵢ)
     b. Ajusta un árbol débil hₜ a los residuos
     c. F_t = F_{t-1} + η × hₜ    (η = learning_rate)
```

```python
import xgboost as xgb

# ── Sklearn GradientBoosting ──────────────────────────────────────────────────
from sklearn.ensemble import GradientBoostingClassifier, HistGradientBoostingClassifier

gb = GradientBoostingClassifier(n_estimators=100, learning_rate=0.1,
                                  max_depth=3, random_state=42)
gb.fit(X_train, y_train)
print(f"\nGradientBoosting  Accuracy: {gb.score(X_test, y_test):.4f}")

# HistGradientBoosting: versión rápida (LightGBM-style, sklearn)
hgb = HistGradientBoostingClassifier(max_iter=100, learning_rate=0.1,
                                       max_depth=3, random_state=42)
hgb.fit(X_train, y_train)
print(f"HistGradientBoost Accuracy: {hgb.score(X_test, y_test):.4f}")

# ── XGBoost ───────────────────────────────────────────────────────────────────
xgb_clf = xgb.XGBClassifier(
    n_estimators=100,
    learning_rate=0.1,
    max_depth=3,
    subsample=0.8,
    colsample_bytree=0.8,
    use_label_encoder=False,
    eval_metric='logloss',
    random_state=42,
    n_jobs=-1
)
xgb_clf.fit(X_train, y_train,
            eval_set=[(X_test, y_test)],
            verbose=False)

y_proba_xgb = xgb_clf.predict_proba(X_test)[:, 1]
print(f"XGBoost           Accuracy: {accuracy_score(y_test, xgb_clf.predict(X_test)):.4f}")
print(f"XGBoost           AUC-ROC:  {roc_auc_score(y_test, y_proba_xgb):.4f}")

# ── XGBoost con early stopping ────────────────────────────────────────────────
X_tr2, X_val, y_tr2, y_val = train_test_split(X_train, y_train, test_size=0.2,
                                                stratify=y_train, random_state=42)

xgb_es = xgb.XGBClassifier(
    n_estimators=500, learning_rate=0.05, max_depth=4,
    subsample=0.8, colsample_bytree=0.8,
    eval_metric='logloss', random_state=42, n_jobs=-1
)
xgb_es.fit(X_tr2, y_tr2,
           eval_set=[(X_val, y_val)],
           early_stopping_rounds=20,
           verbose=False)

print(f"\nXGBoost (early stopping):")
print(f"  Árboles óptimos: {xgb_es.best_iteration}")
print(f"  Mejor score val: {xgb_es.best_score:.4f}")
print(f"  Test Accuracy:   {accuracy_score(y_test, xgb_es.predict(X_test)):.4f}")

# Importancia de features XGBoost
fig, ax = plt.subplots(figsize=(10, 5))
xgb.plot_importance(xgb_clf, ax=ax, max_num_features=10, importance_type='gain')
ax.set_title('XGBoost — Feature Importance (Gain)')
plt.tight_layout()
plt.savefig('outputs/m07_xgb_importance.png', dpi=120)
plt.show()
```

---

## Comparativa final

```python
# ── Comparativa exhaustiva de todos los clasificadores ────────────────────────
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import KNeighborsClassifier
from sklearn.svm import SVC
from sklearn.naive_bayes import GaussianNB
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import (RandomForestClassifier, ExtraTreesClassifier,
                               GradientBoostingClassifier,
                               HistGradientBoostingClassifier)
import xgboost as xgb
import time

clasificadores = {
    'Logistic Regression': Pipeline([
        ('sc', StandardScaler()),
        ('clf', LogisticRegression(C=1, max_iter=1000, random_state=42))
    ]),
    'KNN (k=5)': Pipeline([
        ('sc', StandardScaler()),
        ('clf', KNeighborsClassifier(n_neighbors=5, n_jobs=-1))
    ]),
    'SVM RBF': Pipeline([
        ('sc', StandardScaler()),
        ('clf', SVC(kernel='rbf', C=10, gamma='scale',
                    probability=True, random_state=42))
    ]),
    'Naive Bayes (Gaussian)': GaussianNB(),
    'Decision Tree': DecisionTreeClassifier(max_depth=5, random_state=42),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1),
    'Extra Trees': ExtraTreesClassifier(n_estimators=100, random_state=42, n_jobs=-1),
    'Gradient Boosting': GradientBoostingClassifier(n_estimators=100, random_state=42),
    'HistGradient Boost': HistGradientBoostingClassifier(max_iter=100, random_state=42),
    'XGBoost': xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                                   random_state=42, n_jobs=-1, verbosity=0),
}

cv_strat = StratifiedKFold(n_splits=10, shuffle=True, random_state=42)
resultados_final = {}

print("\n─── Comparativa de clasificadores (CV-10, Breast Cancer) ───\n")
for nombre, clf in clasificadores.items():
    t0 = time.time()
    scores = cross_validate(clf, X, y, cv=cv_strat,
                             scoring=['accuracy', 'f1', 'roc_auc'],
                             n_jobs=-1, return_train_score=False)
    elapsed = time.time() - t0

    resultados_final[nombre] = {
        'Accuracy':  scores['test_accuracy'].mean(),
        'F1':        scores['test_f1'].mean(),
        'AUC-ROC':   scores['test_roc_auc'].mean(),
        'Acc±std':   scores['test_accuracy'].std(),
        'Tiempo(s)': elapsed,
    }
    print(f"  {nombre:<25} Acc={scores['test_accuracy'].mean():.4f} "
          f"F1={scores['test_f1'].mean():.4f} "
          f"AUC={scores['test_roc_auc'].mean():.4f} ({elapsed:.1f}s)")

df_final = pd.DataFrame(resultados_final).T.round(4)
print("\n")
print(df_final[['Accuracy', 'F1', 'AUC-ROC', 'Acc±std', 'Tiempo(s)']].to_string())

# Gráfica de comparativa
fig, axes = plt.subplots(1, 3, figsize=(18, 6))
metricas_plot = ['Accuracy', 'F1', 'AUC-ROC']
colores = plt.cm.tab10(np.linspace(0, 1, len(clasificadores)))

for ax, metrica in zip(axes, metricas_plot):
    nombres = list(resultados_final.keys())
    valores  = [resultados_final[n][metrica] for n in nombres]
    std_vals  = [resultados_final[n]['Acc±std'] for n in nombres]

    y_pos = np.arange(len(nombres))
    bars = ax.barh(y_pos, valores, color=colores, alpha=0.8)
    ax.errorbar(valores, y_pos, xerr=std_vals, fmt='none', color='black',
                 capsize=3, lw=1.5)
    ax.set_yticks(y_pos)
    ax.set_yticklabels(nombres, fontsize=9)
    ax.set_xlabel(metrica)
    ax.set_title(f'{metrica} — CV-10')
    # Mejor modelo resaltado
    best_idx = np.argmax(valores)
    bars[best_idx].set_edgecolor('gold')
    bars[best_idx].set_linewidth(2.5)
    ax.set_xlim(min(valores) * 0.99, 1.01)

plt.suptitle('Comparativa de clasificadores — Breast Cancer (CV-10)', fontsize=14)
plt.tight_layout()
plt.savefig('outputs/m07_comparativa_final.png', dpi=120)
plt.show()
```

---

## Guía de selección de clasificador

```
¿Cuántos datos tienes?
│
├── < 1,000 muestras
│   ├── Features continuas y ruido → Naive Bayes Gaussian (simple, rápido)
│   ├── Necesitas interpretación  → Decision Tree (max_depth controlado)
│   └── Máxima precisión          → SVM RBF con GridSearch
│
├── 1,000 – 100,000 muestras
│   ├── Baseline rápido           → Logistic Regression / Naive Bayes
│   ├── No lineal, interpretable  → Random Forest
│   └── Máxima precisión          → XGBoost / HistGradientBoosting
│
└── > 100,000 muestras
    ├── Tabular data              → XGBoost / LightGBM (no incluido en sklearn)
    ├── Alta dimensión (texto)    → LinearSVC / ComplementNB / LogReg L1
    └── Necesitas actualización   → SGDClassifier (online learning)

¿Necesitas explicabilidad?
  Alta:   Logistic Regression, Decision Tree, Naive Bayes
  Media:  Random Forest (feature importance, SHAP)
  Baja:   SVM RBF, XGBoost (caja negra, pero SHAP ayuda)

¿Datos desbalanceados?
  → Todos: class_weight='balanced' o BalancedRandomForest (imbalanced-learn)
  → NB: ajustar priors manualmente

¿Datos con ruido?
  Random Forest y Gradient Boosting son más robustos que Decision Tree solo
  Naive Bayes: sensible a features correlacionadas (viola independencia)
```

```python
# ── Resumen de resultados ─────────────────────────────────────────────────────
print("\n─── RESUMEN: mejor modelo por criterio ───")
df_sorted_auc = df_final.sort_values('AUC-ROC', ascending=False)
print(f"\n  Mejor AUC-ROC:   {df_sorted_auc.index[0]} ({df_sorted_auc['AUC-ROC'].iloc[0]:.4f})")
df_sorted_f1  = df_final.sort_values('F1', ascending=False)
print(f"  Mejor F1:        {df_sorted_f1.index[0]} ({df_sorted_f1['F1'].iloc[0]:.4f})")
df_sorted_vel = df_final.sort_values('Tiempo(s)')
print(f"  Más rápido:      {df_sorted_vel.index[0]} ({df_sorted_vel['Tiempo(s)'].iloc[0]:.2f}s)")

# Guardar mejor modelo
import joblib, os
best_name = df_sorted_auc.index[0]
best_clf  = clasificadores[best_name]
best_clf.fit(X_train, y_train)
os.makedirs('models', exist_ok=True)
joblib.dump(best_clf, 'models/mejor_clasificador_m07.pkl')
print(f"\nModelo '{best_name}' guardado en models/mejor_clasificador_m07.pkl")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     RESUMEN M07 — NB, DT, RF, XGBoost                      │
├─────────────────────────┬───────────────────────────────────────────────────┤
│ Algoritmo               │ Características clave                             │
├─────────────────────────┼───────────────────────────────────────────────────┤
│ GaussianNB              │ Bayesiano, rápido, funciona bien con poco datos    │
│ MultinomialNB           │ Ideal para texto (conteos/frecuencias)             │
│ ComplementNB            │ MultinomialNB mejorado para clases desbalanceadas  │
│ Decision Tree           │ Interpretable, sobreajusta sin poda               │
│ Random Forest           │ Ensemble paralelo, robusto, feature importance     │
│ Extra Trees             │ Más rápido que RF, threshold aleatorio             │
│ GradientBoosting        │ Ensemble serial, alta precisión, más lento        │
│ HistGradientBoosting    │ GB rápido (LightGBM-style), maneja NaN nativamente │
│ XGBoost                 │ Gold standard tabular, early stopping, GPU         │
└─────────────────────────┴───────────────────────────────────────────────────┘

Orden recomendado al enfrentar un problema de clasificación:
  1. Logistic Regression    ← baseline interpretable
  2. Random Forest          ← robusto sin mucho tuning
  3. XGBoost/HistGB         ← máxima precisión tabular
  4. SVM RBF (< 50k datos) ← útil cuando RF no alcanza
  5. Naive Bayes            ← texto o datos muy escasos
```

---

## Checkpoint M07

```python
# checkpoint_m07.py
"""
Checkpoint M07 — Naive Bayes, Decision Tree, Random Forest
Ejecutar: python checkpoint_m07.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_breast_cancer, load_iris
from sklearn.model_selection import train_test_split, StratifiedKFold, cross_validate
from sklearn.naive_bayes import GaussianNB
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, roc_auc_score, f1_score
import xgboost as xgb
import joblib, os

print("=" * 60)
print("CHECKPOINT M07 — NB, DT, RF, XGBoost")
print("=" * 60)

cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

cv = StratifiedKFold(5, shuffle=True, random_state=42)

# 1. Gaussian Naive Bayes
gnb = GaussianNB()
gnb.fit(X_train, y_train)
acc_gnb = accuracy_score(y_test, gnb.predict(X_test))
print(f"\n✓ GaussianNB Accuracy: {acc_gnb:.4f}")
assert acc_gnb > 0.90, f"GNB muy bajo: {acc_gnb:.4f}"
assert len(gnb.class_prior_) == 2, "GNB no aprendió 2 clases"

# 2. Decision Tree
dt = DecisionTreeClassifier(max_depth=5, random_state=42)
dt.fit(X_train, y_train)
acc_dt = accuracy_score(y_test, dt.predict(X_test))
print(f"✓ Decision Tree (depth=5) Accuracy: {acc_dt:.4f}")
assert acc_dt > 0.88, f"DT muy bajo: {acc_dt:.4f}"
assert dt.get_depth() <= 5, "Árbol más profundo de lo configurado"

# 3. Feature importance DT
assert dt.feature_importances_.sum() > 0.99, "Feature importances no suman ~1"
top_feat = cancer.feature_names[np.argmax(dt.feature_importances_)]
print(f"✓ Feature más importante (DT): {top_feat}")

# 4. Random Forest
rf = RandomForestClassifier(n_estimators=100, oob_score=True,
                              random_state=42, n_jobs=-1)
rf.fit(X_train, y_train)
acc_rf  = accuracy_score(y_test, rf.predict(X_test))
auc_rf  = roc_auc_score(y_test, rf.predict_proba(X_test)[:, 1])
print(f"✓ Random Forest Accuracy: {acc_rf:.4f} | AUC: {auc_rf:.4f}")
assert acc_rf  > 0.94, f"RF Acc muy bajo: {acc_rf:.4f}"
assert auc_rf  > 0.97, f"RF AUC muy bajo: {auc_rf:.4f}"
assert rf.oob_score_ > 0.93, f"OOB muy bajo: {rf.oob_score_:.4f}"
print(f"✓ OOB Score: {rf.oob_score_:.4f}")

# 5. XGBoost
xgb_clf = xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                               random_state=42, verbosity=0, n_jobs=-1)
xgb_clf.fit(X_train, y_train)
acc_xgb = accuracy_score(y_test, xgb_clf.predict(X_test))
auc_xgb = roc_auc_score(y_test, xgb_clf.predict_proba(X_test)[:, 1])
print(f"✓ XGBoost Accuracy: {acc_xgb:.4f} | AUC: {auc_xgb:.4f}")
assert acc_xgb > 0.94, f"XGB Acc muy bajo: {acc_xgb:.4f}"

# 6. Comparativa rápida
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

lr = Pipeline([('sc', StandardScaler()),
               ('clf', LogisticRegression(max_iter=1000, random_state=42))])

mejores = {'GNB': acc_gnb, 'DT': acc_dt, 'RF': acc_rf, 'XGB': acc_xgb}
mejor_nombre = max(mejores, key=mejores.get)
print(f"\n✓ Mejor clasificador en test: {mejor_nombre} ({mejores[mejor_nombre]:.4f})")

# 7. Serialización
os.makedirs('models', exist_ok=True)
joblib.dump(rf, 'models/checkpoint_m07_rf.pkl')
loaded = joblib.load('models/checkpoint_m07_rf.pkl')
assert np.array_equal(loaded.predict(X_test), rf.predict(X_test))
print("✓ Serialización RF: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M07 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • GaussianNB (priors, var_smoothing)")
print("  • Decision Tree (max_depth, feature_importances)")
print("  • Random Forest (OOB score, feature importance)")
print("  • XGBoost (n_estimators, AUC-ROC)")
print("  • Serialización con joblib")
print("=" * 60)
```

---

*Siguiente módulo → M08: Evaluación de Modelos de Clasificación*
