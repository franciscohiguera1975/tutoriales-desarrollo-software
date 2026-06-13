# Módulo 06 — Clasificación: SVM y Kernel SVM

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M05 (Clasificación — Logistic Regression & KNN)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Intuición geométrica de SVM](#intuición-geométrica-de-svm)
3. [Hard Margin vs Soft Margin](#hard-margin-vs-soft-margin)
4. [El truco del Kernel](#el-truco-del-kernel)
5. [SVM Lineal con Scikit-learn](#svm-lineal-con-scikit-learn)
6. [Kernel SVM — los 4 kernels](#kernel-svm---los-4-kernels)
7. [Ajuste de hiperparámetros con GridSearchCV](#ajuste-de-hiperparámetros-con-gridsearchcv)
8. [SVM Multiclase (One-vs-Rest / One-vs-One)](#svm-multiclase)
9. [Probabilidades y calibración (CalibratedClassifierCV)](#probabilidades-y-calibración)
10. [SVM vs Logistic Regression — comparativa final](#svm-vs-logistic-regression)
11. [Checkpoint M06](#checkpoint-m06)

---

## Cómo ejecutar este módulo

> Consulta M01 para la guía completa de entornos. Aquí el resumen rápido.

### Opción A — Script local

```bash
conda activate ml-clasico
python modulo-06-clasificacion-svm.py
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
# numpy, pandas, sklearn, matplotlib ya están disponibles en Colab
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Intuición geométrica de SVM

**Support Vector Machine** (SVM) busca el **hiperplano de máximo margen** que separa dos clases.

```
              Clase A (+)
         ×  ×  ×
        ×  ×  ×   ← margen
    ═══════════════ ← hiperplano óptimo
        ○  ○  ○   ← margen
         ○  ○  ○
              Clase B (−)

    ← margen máximo →
```

Conceptos clave:

| Término          | Definición                                                    |
|------------------|---------------------------------------------------------------|
| Hiperplano       | Frontera de decisión: `w·x + b = 0`                          |
| Margen           | Distancia entre el hiperplano y los puntos más cercanos      |
| Support Vectors  | Los puntos que definen el margen (los más difíciles de clasificar) |
| Máximo margen    | SVM optimiza para que este margen sea lo más grande posible  |

La función de decisión es:

```
f(x) = w·x + b

si f(x) ≥ +1  →  clase +1
si f(x) ≤ −1  →  clase −1
```

```python
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.colors import ListedColormap

# ── Utilidad de visualización de fronteras de decisión ───────────────────────
def plot_frontera(clf, X, y, titulo='Frontera de decisión', ax=None):
    """
    Dibuja la frontera de decisión de cualquier clasificador sklearn.
    Funciona con SVM, Logistic Regression, KNN, etc.
    """
    if ax is None:
        fig, ax = plt.subplots(figsize=(8, 6))

    x_min, x_max = X[:, 0].min() - 0.5, X[:, 0].max() + 0.5
    y_min, y_max = X[:, 1].min() - 0.5, X[:, 1].max() + 0.5
    h = 0.02

    xx, yy = np.meshgrid(np.arange(x_min, x_max, h),
                         np.arange(y_min, y_max, h))
    Z = clf.predict(np.c_[xx.ravel(), yy.ravel()])
    Z = Z.reshape(xx.shape)

    colores_mapa  = ListedColormap(['#FFAAAA', '#AAFFAA', '#AAAAFF'])
    colores_punto = ListedColormap(['#FF0000', '#00AA00', '#0000FF'])

    ax.contourf(xx, yy, Z, alpha=0.3, cmap=colores_mapa)
    scatter = ax.scatter(X[:, 0], X[:, 1], c=y, s=40,
                         cmap=colores_punto, edgecolors='k', linewidths=0.5)
    ax.set_title(titulo, fontsize=13, fontweight='bold')
    ax.set_xlabel('Feature 1')
    ax.set_ylabel('Feature 2')

    # Dibuja vectores de soporte (solo disponible en SVM)
    if hasattr(clf, 'support_vectors_'):
        ax.scatter(clf.support_vectors_[:, 0],
                   clf.support_vectors_[:, 1],
                   s=150, facecolors='none', edgecolors='k',
                   linewidths=2, label='Support Vectors')
        ax.legend()

    return ax


# ── Dataset de ejemplo: dos lunas (no linealmente separable) ─────────────────
from sklearn.datasets import make_moons, make_circles, make_classification

np.random.seed(42)
X_moons, y_moons = make_moons(n_samples=300, noise=0.2, random_state=42)

plt.figure(figsize=(6, 4))
plt.scatter(X_moons[:, 0], X_moons[:, 1], c=y_moons,
            cmap='bwr', edgecolors='k', s=30)
plt.title('Dataset: make_moons — no separable linealmente')
plt.tight_layout()
plt.savefig('outputs/m06_dataset.png', dpi=120)
plt.show()
print("Dataset moons: forma =", X_moons.shape)
```

---

## Hard Margin vs Soft Margin

```python
# ── Demostración visual del efecto del parámetro C ───────────────────────────
from sklearn.svm import SVC
from sklearn.datasets import make_classification
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

np.random.seed(0)
X_lin, y_lin = make_classification(n_samples=200, n_features=2,
                                    n_redundant=0, n_clusters_per_class=1,
                                    random_state=42)

fig, axes = plt.subplots(1, 3, figsize=(18, 5))
valores_C = [0.01, 1.0, 100.0]
descripciones = ['C=0.01 → margen amplio\n(alto sesgo, baja varianza)',
                 'C=1.0 → equilibrio',
                 'C=100 → margen estrecho\n(bajo sesgo, alta varianza)']

for ax, C, desc in zip(axes, valores_C, descripciones):
    clf = Pipeline([
        ('scaler', StandardScaler()),
        ('svc', SVC(kernel='linear', C=C))
    ])
    clf.fit(X_lin, y_lin)
    plot_frontera(clf, X_lin, y_lin, titulo=f'SVM Lineal {desc}', ax=ax)

plt.tight_layout()
plt.savefig('outputs/m06_efecto_C.png', dpi=120)
plt.show()
```

### ¿Qué hace C?

```
C pequeño (0.01)  → toleramos muchos errores dentro del margen → margen AMPLIO
                    Regularización alta — modelo más simple (puede subajustar)

C grande (100)    → penalizamos fuertemente los errores → margen ESTRECHO
                    Regularización baja — modelo más complejo (puede sobreajustar)
```

Analogía:
- **Hard Margin** (C → ∞): el profesor no permite NINGÚN error en el margen. Solo funciona si los datos son perfectamente separables.
- **Soft Margin** (C finito): permite algunos errores (violaciones del margen) a cambio de generalizar mejor.

---

## El truco del Kernel

El **truco del kernel** permite que SVM aprenda fronteras NO lineales sin calcular explícitamente la transformación al espacio de alta dimensión.

```
Espacio original (2D)          Espacio transformado (∞D)
  ○  ×  ×  ○                     ○  ○  |  × ×
  × (no separable)       →       (separable linealmente)

En lugar de calcular ϕ(x) explícitamente, usamos K(x,z) = ϕ(x)·ϕ(z)
```

### Los 4 kernels principales

| Kernel      | Fórmula                          | Parámetros       | Uso típico                        |
|-------------|----------------------------------|------------------|-----------------------------------|
| `linear`    | K(x,z) = x·z                    | C                | Datos linealmente separables, texto |
| `poly`      | K(x,z) = (γx·z + r)^d          | C, γ, degree, r  | Datos con interacciones polinomiales |
| `rbf`       | K(x,z) = exp(−γ‖x−z‖²)         | C, γ             | Uso general (default), datos complejos |
| `sigmoid`   | K(x,z) = tanh(γx·z + r)        | C, γ, r          | Similar a redes neuronales, menos común |

```python
# ── Demostración visual de los 4 kernels ─────────────────────────────────────
X_sc = StandardScaler().fit_transform(X_moons)

kernels = ['linear', 'poly', 'rbf', 'sigmoid']
fig, axes = plt.subplots(2, 2, figsize=(14, 10))
axes = axes.ravel()

for ax, kernel in zip(axes, kernels):
    clf = SVC(kernel=kernel, C=1.0, gamma='scale', random_state=42)
    clf.fit(X_sc, y_moons)
    acc = clf.score(X_sc, y_moons)
    plot_frontera(clf, X_sc, y_moons,
                  titulo=f'Kernel={kernel} | Acc={acc:.3f}', ax=ax)

plt.suptitle('Efecto de los distintos kernels en make_moons', fontsize=14, y=1.01)
plt.tight_layout()
plt.savefig('outputs/m06_kernels.png', dpi=120)
plt.show()
```

### ¿Qué es gamma (γ)?

```python
# ── Efecto de gamma en el kernel RBF ─────────────────────────────────────────
fig, axes = plt.subplots(1, 4, figsize=(20, 4))
gammas = [0.01, 0.1, 1.0, 10.0]

for ax, g in zip(axes, gammas):
    clf = SVC(kernel='rbf', C=1.0, gamma=g)
    clf.fit(X_sc, y_moons)
    acc = clf.score(X_sc, y_moons)
    n_sv = len(clf.support_vectors_)
    plot_frontera(clf, X_sc, y_moons,
                  titulo=f'γ={g} | Acc={acc:.3f}\nSVs={n_sv}', ax=ax)

plt.suptitle('Efecto de gamma (RBF kernel) — γ grande = frontera más compleja', y=1.02)
plt.tight_layout()
plt.savefig('outputs/m06_efecto_gamma.png', dpi=120)
plt.show()
```

```
γ pequeño (0.01) → función de influencia AMPLIA → frontera suave (subajuste)
γ grande  (10.0) → función de influencia ESTRECHA → frontera muy compleja (sobreajuste)

Regla práctica:
  gamma='scale'  →  γ = 1 / (n_features × Var(X))  ← sklearn default
  gamma='auto'   →  γ = 1 / n_features
```

---

## SVM Lineal con Scikit-learn

```python
# ── Dataset real: Breast Cancer Wisconsin ────────────────────────────────────
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, StratifiedKFold, cross_validate
from sklearn.metrics import (classification_report, confusion_matrix,
                              ConfusionMatrixDisplay, roc_auc_score,
                              f1_score, accuracy_score)
import joblib

cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
feature_names = cancer.feature_names

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

print(f"Train: {X_train.shape} | Test: {X_test.shape}")
print(f"Clases: {cancer.target_names} | Balance: {np.bincount(y)}")

# ── SVM Lineal con Pipeline ───────────────────────────────────────────────────
pipe_linear = Pipeline([
    ('scaler', StandardScaler()),
    ('svc', SVC(kernel='linear', C=1.0, random_state=42))
])

pipe_linear.fit(X_train, y_train)
y_pred = pipe_linear.predict(X_test)

print("\n─── SVM Lineal ───")
print(classification_report(y_test, y_pred, target_names=cancer.target_names))

# Matriz de confusión
fig, ax = plt.subplots(figsize=(6, 5))
ConfusionMatrixDisplay.from_predictions(
    y_test, y_pred,
    display_labels=cancer.target_names,
    cmap='Blues', ax=ax
)
ax.set_title('SVM Lineal — Breast Cancer')
plt.tight_layout()
plt.savefig('outputs/m06_cm_linear.png', dpi=120)
plt.show()
```

### Vectores de soporte y su significado

```python
# ── Análisis de los support vectors ──────────────────────────────────────────
svc_model = pipe_linear.named_steps['svc']

print(f"Número total de support vectors: {svc_model.n_support_.sum()}")
print(f"  Clase 0 ({cancer.target_names[0]}): {svc_model.n_support_[0]} SVs")
print(f"  Clase 1 ({cancer.target_names[1]}): {svc_model.n_support_[1]} SVs")
print(f"Ratio SVs / total: {svc_model.n_support_.sum() / len(X_train):.2%}")

# Importancia de features vía coeficientes del hiperplano (solo kernel lineal)
coef = np.abs(svc_model.coef_[0])
top_idx = np.argsort(coef)[::-1][:10]

fig, ax = plt.subplots(figsize=(10, 4))
ax.barh(range(10), coef[top_idx], color='steelblue')
ax.set_yticks(range(10))
ax.set_yticklabels(feature_names[top_idx])
ax.set_xlabel('|Coeficiente del hiperplano|')
ax.set_title('Top 10 features más importantes — SVM Lineal')
ax.invert_yaxis()
plt.tight_layout()
plt.savefig('outputs/m06_coef_linear.png', dpi=120)
plt.show()
```

---

## Kernel SVM — los 4 kernels

```python
# ── Comparativa de kernels en Breast Cancer ───────────────────────────────────
from sklearn.metrics import roc_curve, auc

kernels_config = {
    'linear':  SVC(kernel='linear',  C=1.0, probability=True, random_state=42),
    'poly':    SVC(kernel='poly',    C=1.0, degree=3, gamma='scale',
                   probability=True, random_state=42),
    'rbf':     SVC(kernel='rbf',     C=1.0, gamma='scale',
                   probability=True, random_state=42),
    'sigmoid': SVC(kernel='sigmoid', C=1.0, gamma='scale',
                   probability=True, random_state=42),
}

resultados = {}
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

for nombre, svc in kernels_config.items():
    pipe = Pipeline([('scaler', StandardScaler()), ('svc', svc)])

    # Cross-validation
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    scores = cross_validate(pipe, X_train, y_train, cv=cv,
                             scoring=['accuracy', 'f1', 'roc_auc'],
                             return_train_score=True)

    # Ajuste final y curva ROC
    pipe.fit(X_train, y_train)
    y_proba = pipe.predict_proba(X_test)[:, 1]
    fpr, tpr, _ = roc_curve(y_test, y_proba)
    roc_auc = auc(fpr, tpr)

    resultados[nombre] = {
        'val_acc':   scores['test_accuracy'].mean(),
        'val_f1':    scores['test_f1'].mean(),
        'val_auc':   scores['test_roc_auc'].mean(),
        'test_acc':  accuracy_score(y_test, pipe.predict(X_test)),
        'test_auc':  roc_auc,
    }

    axes[1].plot(fpr, tpr, label=f'{nombre} (AUC={roc_auc:.3f})')

# Tabla de resultados
import pandas as pd
df_res = pd.DataFrame(resultados).T.round(4)
print("\n─── Comparativa de kernels ───")
print(df_res.to_string())

# Curvas ROC
axes[1].plot([0,1], [0,1], 'k--', lw=1)
axes[1].set_xlabel('FPR (1 - Especificidad)')
axes[1].set_ylabel('TPR (Sensibilidad)')
axes[1].set_title('Curvas ROC por kernel')
axes[1].legend()

# Bar chart de métricas
metricas = ['val_acc', 'val_f1', 'val_auc']
x = np.arange(len(metricas))
width = 0.2
for i, (nombre, vals) in enumerate(resultados.items()):
    yvals = [vals[m] for m in metricas]
    axes[0].bar(x + i * width, yvals, width, label=nombre)

axes[0].set_xticks(x + width * 1.5)
axes[0].set_xticklabels(['CV Accuracy', 'CV F1', 'CV AUC-ROC'])
axes[0].set_ylim(0.88, 1.0)
axes[0].set_title('Métricas de validación cruzada por kernel')
axes[0].legend()

plt.tight_layout()
plt.savefig('outputs/m06_kernels_comparativa.png', dpi=120)
plt.show()
```

---

## Ajuste de hiperparámetros con GridSearchCV

```python
# ── Grid Search para RBF SVM ──────────────────────────────────────────────────
from sklearn.model_selection import GridSearchCV

param_grid_rbf = {
    'svc__C':     [0.1, 1, 10, 100],
    'svc__gamma': ['scale', 'auto', 0.001, 0.01, 0.1, 1],
}

pipe_rbf = Pipeline([
    ('scaler', StandardScaler()),
    ('svc', SVC(kernel='rbf', probability=True, random_state=42))
])

grid_rbf = GridSearchCV(
    pipe_rbf, param_grid_rbf,
    cv=StratifiedKFold(n_splits=5, shuffle=True, random_state=42),
    scoring='roc_auc',
    n_jobs=-1, verbose=1
)
grid_rbf.fit(X_train, y_train)

print(f"\nMejores parámetros (RBF): {grid_rbf.best_params_}")
print(f"Mejor AUC CV:             {grid_rbf.best_score_:.4f}")

# Evaluación en test
y_pred_best  = grid_rbf.predict(X_test)
y_proba_best = grid_rbf.predict_proba(X_test)[:, 1]
print(f"\nTest Accuracy: {accuracy_score(y_test, y_pred_best):.4f}")
print(f"Test AUC-ROC:  {roc_auc_score(y_test, y_proba_best):.4f}")
print(f"Test F1:       {f1_score(y_test, y_pred_best):.4f}")

# ── Mapa de calor del GridSearch (C vs gamma) ─────────────────────────────────
import matplotlib.cm as cm

# Filtramos solo los valores numéricos de gamma para el heatmap
cv_results = grid_rbf.cv_results_
scores_matrix = cv_results['mean_test_score'].reshape(
    len(param_grid_rbf['svc__C']),
    len(param_grid_rbf['svc__gamma'])
)

fig, ax = plt.subplots(figsize=(10, 5))
im = ax.imshow(scores_matrix, cmap='YlOrRd', aspect='auto',
               vmin=scores_matrix.min(), vmax=scores_matrix.max())
ax.set_xticks(range(len(param_grid_rbf['svc__gamma'])))
ax.set_xticklabels(param_grid_rbf['svc__gamma'], rotation=45)
ax.set_yticks(range(len(param_grid_rbf['svc__C'])))
ax.set_yticklabels(param_grid_rbf['svc__C'])
ax.set_xlabel('gamma')
ax.set_ylabel('C')
ax.set_title('GridSearch AUC-ROC — SVM RBF (Breast Cancer)')
plt.colorbar(im, ax=ax, label='AUC-ROC (CV)')

# Marcamos el mejor
idx_flat = np.argmax(scores_matrix)
iy, ix = np.unravel_index(idx_flat, scores_matrix.shape)
ax.add_patch(plt.Rectangle((ix-0.5, iy-0.5), 1, 1,
                             fill=False, edgecolor='blue', lw=3))
plt.tight_layout()
plt.savefig('outputs/m06_gridsearch_heatmap.png', dpi=120)
plt.show()
```

### Grid Search para kernel polinomial

```python
# ── Grid Search para kernel polinomial ───────────────────────────────────────
param_grid_poly = {
    'svc__C':      [0.1, 1, 10],
    'svc__degree': [2, 3, 4, 5],
    'svc__gamma':  ['scale', 'auto'],
    'svc__coef0':  [0.0, 1.0],
}

pipe_poly = Pipeline([
    ('scaler', StandardScaler()),
    ('svc', SVC(kernel='poly', probability=True, random_state=42))
])

grid_poly = GridSearchCV(
    pipe_poly, param_grid_poly,
    cv=StratifiedKFold(n_splits=5, shuffle=True, random_state=42),
    scoring='roc_auc', n_jobs=-1
)
grid_poly.fit(X_train, y_train)

print(f"\nMejores parámetros (Poly): {grid_poly.best_params_}")
print(f"Mejor AUC CV:              {grid_poly.best_score_:.4f}")
```

---

## SVM Multiclase

```python
# ── SVM con Iris: estrategias One-vs-One y One-vs-Rest ───────────────────────
from sklearn.datasets import load_iris
from sklearn.multiclass import OneVsRestClassifier, OneVsOneClassifier

iris = load_iris()
X_ir, y_ir = iris.data[:, :2], iris.target  # 2 features para visualizar

# SVC en sklearn usa OvO por defecto cuando hay >2 clases
print("Estrategias multiclase:")
print("  OvO (One-vs-One):  nC*(nC-1)/2 clasificadores =", 3*2//2, "clases=3")
print("  OvR (One-vs-Rest): nC clasificadores =", 3, "clases=3")

fig, axes = plt.subplots(1, 3, figsize=(18, 5))

configuraciones = [
    ('SVC (OvO por defecto)',
     SVC(kernel='rbf', C=10, gamma='scale', random_state=42)),
    ('OneVsOneClassifier(SVC)',
     OneVsOneClassifier(SVC(kernel='rbf', C=10, gamma='scale', random_state=42))),
    ('OneVsRestClassifier(SVC)',
     OneVsRestClassifier(SVC(kernel='rbf', C=10, gamma='scale', random_state=42))),
]

for ax, (titulo, clf) in zip(axes, configuraciones):
    pipe_ir = Pipeline([('scaler', StandardScaler()), ('clf', clf)])
    pipe_ir.fit(X_ir, y_ir)
    acc = pipe_ir.score(X_ir, y_ir)
    plot_frontera(pipe_ir, X_ir, y_ir,
                  titulo=f'{titulo}\nAcc={acc:.3f}', ax=ax)

plt.suptitle('Iris — SVM multiclase (2 features para visualizar)', y=1.02)
plt.tight_layout()
plt.savefig('outputs/m06_multiclase.png', dpi=120)
plt.show()
```

---

## Probabilidades y calibración

Por defecto, `SVC` no produce probabilidades. Hay dos formas de obtenerlas:

```python
# ── Opción 1: probability=True (Platt scaling — más lento) ───────────────────
svc_prob = SVC(kernel='rbf', C=10, gamma='scale',
               probability=True, random_state=42)
pipe_prob = Pipeline([('scaler', StandardScaler()), ('svc', svc_prob)])
pipe_prob.fit(X_train, y_train)

proba = pipe_prob.predict_proba(X_test)[:, 1]
print("Con probability=True:")
print(f"  AUC-ROC: {roc_auc_score(y_test, proba):.4f}")
print(f"  Rango de probabilidades: [{proba.min():.3f}, {proba.max():.3f}]")

# ── Opción 2: CalibratedClassifierCV (mejor calibración) ─────────────────────
from sklearn.calibration import CalibratedClassifierCV, CalibrationDisplay

svc_base = SVC(kernel='rbf', C=10, gamma='scale', random_state=42)
svc_calibrado = CalibratedClassifierCV(
    Pipeline([('scaler', StandardScaler()), ('svc', svc_base)]),
    method='sigmoid',  # 'sigmoid' (Platt) o 'isotonic'
    cv=5
)
svc_calibrado.fit(X_train, y_train)

proba_cal = svc_calibrado.predict_proba(X_test)[:, 1]
print("\nCon CalibratedClassifierCV:")
print(f"  AUC-ROC: {roc_auc_score(y_test, proba_cal):.4f}")

# Curva de calibración
fig, ax = plt.subplots(figsize=(6, 6))
CalibrationDisplay.from_predictions(
    y_test, proba, n_bins=10, ax=ax, name='SVC (probability=True)'
)
CalibrationDisplay.from_predictions(
    y_test, proba_cal, n_bins=10, ax=ax, name='CalibratedClassifierCV'
)
ax.set_title('Curva de calibración — SVM')
plt.tight_layout()
plt.savefig('outputs/m06_calibracion.png', dpi=120)
plt.show()
```

### ¿Cuándo necesitas probabilidades?

| Situación                              | Recomendación                          |
|----------------------------------------|----------------------------------------|
| Solo necesitas la clase predicha       | `SVC(probability=False)` ← más rápido |
| Necesitas AUC-ROC o ranking            | `SVC(probability=True)` o `LinearSVC` + `CalibratedClassifierCV` |
| Necesitas probabilidades bien calibradas | `CalibratedClassifierCV(method='isotonic')` |
| Dataset muy grande                     | `LinearSVC` (más eficiente que `SVC(kernel='linear')`) |

---

## SVM vs Logistic Regression

```python
# ── Comparativa final: SVM (mejores params) vs Logistic Regression ───────────
from sklearn.linear_model import LogisticRegression

modelos = {
    'LR (L2, C=1)':      Pipeline([('sc', StandardScaler()),
                                    ('clf', LogisticRegression(C=1, max_iter=1000,
                                                               random_state=42))]),
    'LR (L1, C=0.1)':    Pipeline([('sc', StandardScaler()),
                                    ('clf', LogisticRegression(C=0.1, penalty='l1',
                                                               solver='liblinear',
                                                               max_iter=1000))]),
    'SVM Linear (C=1)':  Pipeline([('sc', StandardScaler()),
                                    ('clf', SVC(kernel='linear', C=1,
                                                probability=True, random_state=42))]),
    'SVM RBF (best)':    grid_rbf.best_estimator_,
    'SVM Poly (best)':   grid_poly.best_estimator_,
}

cv_final = StratifiedKFold(n_splits=10, shuffle=True, random_state=42)
metricas_final = {}

for nombre, modelo in modelos.items():
    scores = cross_validate(modelo, X, y, cv=cv_final,
                             scoring=['accuracy', 'f1', 'roc_auc'],
                             return_train_score=False, n_jobs=-1)
    metricas_final[nombre] = {
        'Accuracy':  f"{scores['test_accuracy'].mean():.4f} ± {scores['test_accuracy'].std():.4f}",
        'F1':        f"{scores['test_f1'].mean():.4f} ± {scores['test_f1'].std():.4f}",
        'AUC-ROC':   f"{scores['test_roc_auc'].mean():.4f} ± {scores['test_roc_auc'].std():.4f}",
    }

df_final = pd.DataFrame(metricas_final).T
print("\n─── Comparativa final: SVM vs Logistic Regression ───")
print(df_final.to_string())
```

### Guía de decisión: ¿SVM o Logistic Regression?

```
                ┌─────────────────────────────────────────────────────┐
                │  ¿Los datos son linealmente separables (o casi)?     │
                └───────────────────────┬─────────────────────────────┘
                                        │
              SÍ                        │                    NO
        ┌─────┴─────┐                   │            ┌────────────────┐
        ▼           ▼                   │            ▼                ▼
   ¿Dataset     Dataset            SVM (RBF/Poly)  ¿Necesitas     ¿Dataset
   pequeño?     grande?              funciona      probabilidades  grande?
        │           │                muy bien          bien?          │
        ▼           ▼                                  │              ▼
  SVM Linear  LinearSVC         SÍ → LogReg + Calib   NO → SVM   KernelApprox
  LogReg L2   LogReg L1/L2                                         + SGD

Reglas prácticas:
  • n < 10_000:   SVM RBF suele ganar
  • n > 100_000:  LinearSVC o LogReg (SVM RBF muy lento: O(n²) a O(n³))
  • Alta dimensión (texto/NLP): LinearSVC, LogReg L1/L2
  • Necesitas explicabilidad: LogReg (coeficientes interpretables)
  • Máxima precisión, datos pequeños: SVM con GridSearch
```

---

## Linearización para grandes datasets: LinearSVC y SGD

```python
# ── LinearSVC: SVM lineal mucho más eficiente ────────────────────────────────
from sklearn.svm import LinearSVC

pipe_linear_svc = Pipeline([
    ('scaler', StandardScaler()),
    ('svc', LinearSVC(C=1.0, max_iter=5000, random_state=42))
])

scores_linear = cross_validate(
    pipe_linear_svc, X, y, cv=cv_final,
    scoring=['accuracy', 'f1'], n_jobs=-1
)
print(f"\nLinearSVC: Acc={scores_linear['test_accuracy'].mean():.4f} "
      f"| F1={scores_linear['test_f1'].mean():.4f}")

# ── Aproximación de kernel para datos masivos: Nyström + LinearSVC ───────────
from sklearn.kernel_approximation import Nystroem
from sklearn.pipeline import Pipeline as Pipe

pipe_nystroem = Pipe([
    ('scaler', StandardScaler()),
    ('feature_map', Nystroem(kernel='rbf', n_components=100, gamma='scale',
                              random_state=42)),
    ('svc', LinearSVC(C=1.0, max_iter=5000, random_state=42))
])

scores_ny = cross_validate(
    pipe_nystroem, X, y, cv=cv_final,
    scoring=['accuracy', 'f1'], n_jobs=-1
)
print(f"Nyström + LinearSVC: Acc={scores_ny['test_accuracy'].mean():.4f} "
      f"| F1={scores_ny['test_f1'].mean():.4f}")
print("(Nyström aproxima el kernel RBF, escala a millones de muestras)")
```

---

## Guardar el mejor modelo

```python
# ── Serialización del modelo ganador ─────────────────────────────────────────
mejor_modelo = grid_rbf.best_estimator_
mejor_modelo.fit(X_train, y_train)

import os
os.makedirs('models', exist_ok=True)
joblib.dump(mejor_modelo, 'models/svm_rbf_best.pkl')
print("Modelo guardado en models/svm_rbf_best.pkl")

# Verificación de carga
modelo_cargado = joblib.load('models/svm_rbf_best.pkl')
y_pred_cargado = modelo_cargado.predict(X_test)
assert np.array_equal(y_pred_cargado, grid_rbf.predict(X_test)), "¡El modelo cargado no coincide!"
print("Verificación OK: modelo cargado produce las mismas predicciones")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          RESUMEN M06 — SVM                                  │
├─────────────────────┬───────────────────────────────────────────────────────┤
│ Concepto            │ Punto clave                                            │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Hiperplano          │ w·x + b = 0 → maximiza el margen geométrico           │
│ Support Vectors     │ Los puntos en el margen que definen el hiperplano      │
│ Parámetro C         │ C↑ → margen estrecho (baja regularización)             │
│                     │ C↓ → margen amplio (alta regularización)               │
│ Kernel lineal       │ Datos separables, NLP, alta dimensión                  │
│ Kernel RBF          │ Uso general, default, no lineal                        │
│ Kernel poly         │ Interacciones entre features, grado ajustable          │
│ Parámetro gamma     │ γ↑ → frontera compleja; γ↓ → frontera suave            │
│ Probabilidades      │ probability=True (Platt) o CalibratedClassifierCV      │
│ Multiclase          │ OvO (default sklearn) o OvR                            │
│ Escalabilidad       │ LinearSVC o Nyström para n > 100k                      │
└─────────────────────┴───────────────────────────────────────────────────────┘

Cuándo usar SVM:
  ✓ Datasets medianos (< 50k muestras)
  ✓ Alta dimensionalidad con pocas muestras
  ✓ Datos no lineales (RBF kernel)
  ✓ Cuando la precisión importa más que la velocidad de predicción

Cuándo evitar SVM:
  ✗ Datasets muy grandes (lento en entrenamiento)
  ✗ Cuando necesitas explicabilidad simple
  ✗ Cuando necesitas actualización online (prefiere SGDClassifier)
```

---

## Checkpoint M06

Guarda el siguiente script como `checkpoint_m06.py` y ejecútalo para verificar que dominas los conceptos:

```python
# checkpoint_m06.py
"""
Checkpoint M06 — SVM y Kernel SVM
Ejecutar: python checkpoint_m06.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, StratifiedKFold, cross_validate
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.svm import SVC, LinearSVC
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
from sklearn.model_selection import GridSearchCV
import joblib, os

print("=" * 60)
print("CHECKPOINT M06 — SVM y Kernel SVM")
print("=" * 60)

# 1. Carga de datos
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
print(f"\n✓ Dataset: {X.shape[0]} muestras, {X.shape[1]} features")
print(f"  Train/Test: {len(X_train)}/{len(X_test)}")

# 2. SVM lineal
pipe_lin = Pipeline([('sc', StandardScaler()),
                     ('svc', SVC(kernel='linear', C=1, random_state=42))])
pipe_lin.fit(X_train, y_train)
acc_lin = accuracy_score(y_test, pipe_lin.predict(X_test))
print(f"\n✓ SVM Lineal (C=1) — Test Accuracy: {acc_lin:.4f}")
assert acc_lin > 0.92, f"Accuracy muy baja: {acc_lin:.4f}"

# 3. SVM RBF
pipe_rbf = Pipeline([('sc', StandardScaler()),
                     ('svc', SVC(kernel='rbf', C=10, gamma='scale',
                                  probability=True, random_state=42))])
pipe_rbf.fit(X_train, y_train)
y_proba = pipe_rbf.predict_proba(X_test)[:, 1]
auc = roc_auc_score(y_test, y_proba)
acc_rbf = accuracy_score(y_test, pipe_rbf.predict(X_test))
print(f"✓ SVM RBF   (C=10) — Test Accuracy: {acc_rbf:.4f} | AUC: {auc:.4f}")
assert acc_rbf > 0.94, f"Accuracy muy baja: {acc_rbf:.4f}"
assert auc > 0.97, f"AUC muy bajo: {auc:.4f}"

# 4. Support vectors
n_sv = pipe_rbf.named_steps['svc'].n_support_.sum()
print(f"✓ Número de support vectors: {n_sv}")
assert 0 < n_sv < len(X_train), "Número de SVs fuera de rango"

# 5. GridSearchCV (mini-grid para rapidez)
mini_grid = GridSearchCV(
    Pipeline([('sc', StandardScaler()),
              ('svc', SVC(kernel='rbf', probability=True, random_state=42))]),
    {'svc__C': [1, 10], 'svc__gamma': ['scale', 0.01]},
    cv=3, scoring='roc_auc', n_jobs=-1
)
mini_grid.fit(X_train, y_train)
print(f"✓ GridSearch RBF — Mejores params: {mini_grid.best_params_}")
print(f"  Mejor AUC CV: {mini_grid.best_score_:.4f}")

# 6. LinearSVC (eficiencia)
pipe_lsvc = Pipeline([('sc', StandardScaler()),
                      ('svc', LinearSVC(C=1, max_iter=5000, random_state=42))])
cv_scores = cross_validate(pipe_lsvc, X, y,
                            cv=StratifiedKFold(5, shuffle=True, random_state=42),
                            scoring='accuracy', n_jobs=-1)
print(f"✓ LinearSVC CV Accuracy: {cv_scores['test_score'].mean():.4f} "
      f"± {cv_scores['test_score'].std():.4f}")

# 7. Serialización
os.makedirs('models', exist_ok=True)
joblib.dump(pipe_rbf, 'models/checkpoint_m06_svm.pkl')
loaded = joblib.load('models/checkpoint_m06_svm.pkl')
assert np.array_equal(loaded.predict(X_test), pipe_rbf.predict(X_test))
print("✓ Serialización y carga del modelo: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M06 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • SVM lineal y RBF implementados correctamente")
print("  • Métricas de evaluación dentro del rango esperado")
print("  • GridSearchCV funcional")
print("  • LinearSVC para escalabilidad")
print("  • Serialización con joblib")
print("=" * 60)
```

---

*Siguiente módulo → M07: Clasificación — Naive Bayes, Decision Tree y Random Forest*
