# Módulo 08 — Evaluación de Modelos de Clasificación

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M07 (Naive Bayes, Decision Tree, Random Forest)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Métricas de clasificación — marco completo](#métricas-de-clasificación)
3. [Curva ROC y AUC-ROC](#curva-roc-y-auc-roc)
4. [Curva Precision-Recall y Average Precision](#curva-precision-recall)
5. [Clasificación multiclase](#clasificación-multiclase)
6. [Datasets desbalanceados](#datasets-desbalanceados)
7. [Estrategias de validación cruzada](#estrategias-de-validación-cruzada)
8. [Análisis de errores](#análisis-de-errores)
9. [Pipeline de evaluación reutilizable](#pipeline-de-evaluación-reutilizable)
10. [Checkpoint M08](#checkpoint-m08)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install imbalanced-learn  # si no está instalado
python modulo-08-evaluacion-clasificacion.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
!pip install -q imbalanced-learn
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Métricas de clasificación

### La matriz de confusión — el punto de partida

```
                 PREDICHO
                 Negativo  Positivo
REAL  Negativo │    TN   │   FP   │  → Especificidad = TN/(TN+FP)
      Positivo │    FN   │   TP   │  → Sensibilidad  = TP/(TP+FN)

Accuracy    = (TP + TN) / Total                ← NO usar con datos desbalanceados
Precision   = TP / (TP + FP)                   ← de lo que predigo +, ¿cuánto es +?
Recall      = TP / (TP + FN)                   ← de lo real +, ¿cuánto capturo?
F1-Score    = 2 × Precision × Recall           ← media armónica P y R
              ─────────────────────
              Precision + Recall
Fβ-Score    = (1+β²) × P×R / (β²×P + R)       ← β>1: prioriza Recall; β<1: Precision
MCC         = (TP×TN − FP×FN) /                ← más robusto ante desbalance
              √((TP+FP)(TP+FN)(TN+FP)(TN+FN))
Cohen's κ   = (Acc − Acc_random) /             ← acuerdo más allá del azar
              (1 − Acc_random)
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_breast_cancer, load_iris, make_classification
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_validate, learning_curve)
from sklearn.preprocessing import StandardScaler, label_binarize
from sklearn.pipeline import Pipeline
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.metrics import (
    confusion_matrix, ConfusionMatrixDisplay,
    accuracy_score, precision_score, recall_score,
    f1_score, fbeta_score, matthews_corrcoef, cohen_kappa_score,
    roc_curve, roc_auc_score, auc,
    precision_recall_curve, average_precision_score,
    classification_report, brier_score_loss
)

# ── Dataset principal ─────────────────────────────────────────────────────────
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# Modelo de referencia para evaluación
pipe = Pipeline([
    ('sc', StandardScaler()),
    ('clf', RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1))
])
pipe.fit(X_train, y_train)
y_pred  = pipe.predict(X_test)
y_proba = pipe.predict_proba(X_test)[:, 1]


# ── Función: calcular todas las métricas ─────────────────────────────────────
def todas_las_metricas(y_true, y_pred, y_proba=None, nombre='Modelo'):
    """
    Calcula y muestra el conjunto completo de métricas de clasificación binaria.
    """
    tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()

    metricas = {
        'Accuracy':      accuracy_score(y_true, y_pred),
        'Precision':     precision_score(y_true, y_pred, zero_division=0),
        'Recall':        recall_score(y_true, y_pred),
        'F1-Score':      f1_score(y_true, y_pred),
        'F2-Score':      fbeta_score(y_true, y_pred, beta=2),
        'F0.5-Score':    fbeta_score(y_true, y_pred, beta=0.5),
        'Specificity':   tn / (tn + fp) if (tn + fp) > 0 else 0,
        'MCC':           matthews_corrcoef(y_true, y_pred),
        "Cohen's Kappa": cohen_kappa_score(y_true, y_pred),
        'TP':            tp, 'FP': fp, 'TN': tn, 'FN': fn,
    }

    if y_proba is not None:
        metricas['AUC-ROC'] = roc_auc_score(y_true, y_proba)
        metricas['Avg Precision'] = average_precision_score(y_true, y_proba)
        metricas['Brier Score'] = brier_score_loss(y_true, y_proba)

    print(f"\n{'─'*50}")
    print(f" Métricas: {nombre}")
    print(f"{'─'*50}")
    for k, v in metricas.items():
        if isinstance(v, float):
            print(f"  {k:<20} {v:.4f}")
        else:
            print(f"  {k:<20} {v}")
    return metricas


resultados = todas_las_metricas(y_test, y_pred, y_proba, 'Random Forest')
```

### Visualización de la matriz de confusión

```python
# ── Matriz de confusión extendida ─────────────────────────────────────────────
fig, axes = plt.subplots(1, 3, figsize=(15, 4))

# Counts
ConfusionMatrixDisplay.from_predictions(
    y_test, y_pred, display_labels=cancer.target_names,
    cmap='Blues', ax=axes[0]
)
axes[0].set_title('Conteos absolutos')

# Normalizada por filas (recall por clase)
ConfusionMatrixDisplay.from_predictions(
    y_test, y_pred, display_labels=cancer.target_names,
    normalize='true', cmap='Blues', ax=axes[1]
)
axes[1].set_title('Normalizada por fila (Recall)')

# Normalizada por columnas (precision por clase)
ConfusionMatrixDisplay.from_predictions(
    y_test, y_pred, display_labels=cancer.target_names,
    normalize='pred', cmap='Oranges', ax=axes[2]
)
axes[2].set_title('Normalizada por columna (Precision)')

plt.suptitle('Matriz de confusión — Random Forest (Breast Cancer)', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m08_confusion_matrices.png', dpi=120)
plt.show()
```

### El trade-off Precision vs Recall — efecto del threshold

```python
# ── Cómo el threshold cambia P, R, F1 ────────────────────────────────────────
thresholds = np.linspace(0.01, 0.99, 99)
precisions, recalls, f1s = [], [], []

for t in thresholds:
    y_pred_t = (y_proba >= t).astype(int)
    precisions.append(precision_score(y_test, y_pred_t, zero_division=0))
    recalls.append(recall_score(y_test, y_pred_t, zero_division=0))
    f1s.append(f1_score(y_test, y_pred_t, zero_division=0))

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

axes[0].plot(thresholds, precisions, 'b-', lw=2, label='Precision')
axes[0].plot(thresholds, recalls,    'r-', lw=2, label='Recall')
axes[0].plot(thresholds, f1s,        'g-', lw=2, label='F1')
best_t_f1 = thresholds[np.argmax(f1s)]
axes[0].axvline(best_t_f1, color='green', ls='--',
                 label=f'Mejor threshold (F1): {best_t_f1:.2f}')
axes[0].axvline(0.5, color='gray', ls=':', label='Default (0.5)')
axes[0].set_xlabel('Threshold de decisión')
axes[0].set_ylabel('Métrica')
axes[0].set_title('Threshold vs Precision / Recall / F1')
axes[0].legend()

# Trade-off P vs R
axes[1].plot(recalls, precisions, 'b-', lw=2)
axes[1].scatter([recalls[np.argmax(f1s)]], [precisions[np.argmax(f1s)]],
                 s=100, color='green', zorder=5, label=f'Mejor F1 (t={best_t_f1:.2f})')
axes[1].set_xlabel('Recall')
axes[1].set_ylabel('Precision')
axes[1].set_title('Curva Precision vs Recall (threshold como parámetro)')
axes[1].legend()

plt.tight_layout()
plt.savefig('outputs/m08_threshold_tradeoff.png', dpi=120)
plt.show()

print(f"\nThreshold óptimo para F1: {best_t_f1:.2f}")
print(f"  Precision: {precisions[np.argmax(f1s)]:.4f}")
print(f"  Recall:    {recalls[np.argmax(f1s)]:.4f}")
print(f"  F1:        {max(f1s):.4f}")
```

---

## Curva ROC y AUC-ROC

```python
# ── Curvas ROC para múltiples modelos ─────────────────────────────────────────
modelos_roc = {
    'Random Forest':      pipe,
    'Logistic Reg.':      Pipeline([('sc', StandardScaler()),
                                    ('clf', LogisticRegression(max_iter=1000))]),
    'SVM RBF':            Pipeline([('sc', StandardScaler()),
                                    ('clf', SVC(kernel='rbf', probability=True))]),
    'Gradient Boosting':  GradientBoostingClassifier(n_estimators=100, random_state=42),
}

fig, ax = plt.subplots(figsize=(8, 7))
ax.plot([0,1], [0,1], 'k--', lw=1, label='Random (AUC=0.50)')

for nombre, modelo in modelos_roc.items():
    modelo.fit(X_train, y_train)
    y_p = modelo.predict_proba(X_test)[:, 1]
    fpr, tpr, thres = roc_curve(y_test, y_p)
    roc_auc = auc(fpr, tpr)

    ax.plot(fpr, tpr, lw=2, label=f'{nombre} (AUC={roc_auc:.4f})')

    # Punto de Youden (maximiza TPR - FPR)
    j_scores = tpr - fpr
    best_idx = np.argmax(j_scores)
    ax.scatter(fpr[best_idx], tpr[best_idx], s=60, zorder=5)

ax.set_xlabel('FPR (1 − Especificidad)')
ax.set_ylabel('TPR (Sensibilidad / Recall)')
ax.set_title('Curvas ROC — Breast Cancer')
ax.legend(loc='lower right')
ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/m08_roc_curves.png', dpi=120)
plt.show()


# ── Índice de Youden y threshold óptimo ──────────────────────────────────────
fpr_rf, tpr_rf, thresholds_rf = roc_curve(y_test, y_proba)
j_scores = tpr_rf - fpr_rf
youden_idx = np.argmax(j_scores)
youden_threshold = thresholds_rf[youden_idx]

print(f"\nThreshold de Youden (RF): {youden_threshold:.4f}")
print(f"  TPR (Recall):       {tpr_rf[youden_idx]:.4f}")
print(f"  FPR (1-Especific.): {fpr_rf[youden_idx]:.4f}")

y_pred_youden = (y_proba >= youden_threshold).astype(int)
print(f"  F1 con Youden:      {f1_score(y_test, y_pred_youden):.4f}")
print(f"  F1 con default 0.5: {f1_score(y_test, y_pred):.4f}")
```

---

## Curva Precision-Recall

La curva PR es más informativa que ROC cuando las clases están **muy desbalanceadas**.

```python
# ── Curvas Precision-Recall ───────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(8, 6))

for nombre, modelo in modelos_roc.items():
    modelo.fit(X_train, y_train)
    y_p = modelo.predict_proba(X_test)[:, 1]
    prec, rec, _ = precision_recall_curve(y_test, y_p)
    ap = average_precision_score(y_test, y_p)
    ax.plot(rec, prec, lw=2, label=f'{nombre} (AP={ap:.4f})')

# Baseline: clasificador constante
baseline = np.sum(y_test) / len(y_test)
ax.axhline(baseline, color='gray', ls='--', label=f'Baseline (P={baseline:.2f})')

ax.set_xlabel('Recall')
ax.set_ylabel('Precision')
ax.set_title('Curvas Precision-Recall — Breast Cancer')
ax.legend()
ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/m08_pr_curves.png', dpi=120)
plt.show()

# Cuándo usar ROC vs PR:
print("""
ROC-AUC vs Average Precision (curva PR):
  ROC-AUC: útil cuando TP y TN importan igualmente, clases balanceadas
  AP/PR:   mejor cuando la clase positiva es rara (fraude, tumores, etc.)

  Regla: si TN es muy abundante → usar PR curve, no ROC
""")
```

---

## Clasificación multiclase

```python
# ── Dataset Iris (multiclase) ─────────────────────────────────────────────────
iris = load_iris()
X_ir, y_ir = iris.data, iris.target
X_tr_ir, X_te_ir, y_tr_ir, y_te_ir = train_test_split(
    X_ir, y_ir, test_size=0.25, stratify=y_ir, random_state=42
)

rf_ir = RandomForestClassifier(n_estimators=100, random_state=42)
rf_ir.fit(X_tr_ir, y_tr_ir)
y_pred_ir    = rf_ir.predict(X_te_ir)
y_proba_ir   = rf_ir.predict_proba(X_te_ir)

print("\n─── Clasificación Multiclase (Iris) ───")
print(classification_report(y_te_ir, y_pred_ir, target_names=iris.target_names))

# ── Estrategias de promediado F1 multiclase ───────────────────────────────────
print("F1 con diferentes estrategias de promediado:")
for avg in ['macro', 'micro', 'weighted']:
    score = f1_score(y_te_ir, y_pred_ir, average=avg)
    print(f"  F1 ({avg:<9}): {score:.4f}")

print("""
  macro:    promedio simple de F1 por clase → da igual peso a clases raras
  micro:    cuenta TP/FP/FN globalmente → favorece clases frecuentes
  weighted: ponderado por frecuencia de clase → útil con desbalance
  None:     devuelve F1 por clase (sin promedio)
""")

# ── AUC multiclase: OvR (One-vs-Rest) y OvO (One-vs-One) ────────────────────
# Binarización de etiquetas para calcular AUC por clase
y_bin = label_binarize(y_te_ir, classes=[0, 1, 2])

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Curvas ROC por clase
for i, cls_name in enumerate(iris.target_names):
    fpr, tpr, _ = roc_curve(y_bin[:, i], y_proba_ir[:, i])
    roc_auc_cls = auc(fpr, tpr)
    axes[0].plot(fpr, tpr, lw=2, label=f'{cls_name} (AUC={roc_auc_cls:.3f})')

axes[0].plot([0,1], [0,1], 'k--')
axes[0].set_xlabel('FPR')
axes[0].set_ylabel('TPR')
axes[0].set_title('ROC Curves — Iris (OvR por clase)')
axes[0].legend()

# AUC global
auc_ovr = roc_auc_score(y_te_ir, y_proba_ir, multi_class='ovr', average='macro')
auc_ovo = roc_auc_score(y_te_ir, y_proba_ir, multi_class='ovo', average='macro')
print(f"AUC macro (OvR): {auc_ovr:.4f}")
print(f"AUC macro (OvO): {auc_ovo:.4f}")

# Matriz de confusión multiclase
ConfusionMatrixDisplay.from_predictions(
    y_te_ir, y_pred_ir,
    display_labels=iris.target_names,
    cmap='Blues', ax=axes[1]
)
axes[1].set_title('Confusion Matrix — Iris (RF)')

plt.tight_layout()
plt.savefig('outputs/m08_multiclass_roc.png', dpi=120)
plt.show()
```

---

## Datasets desbalanceados

```python
# ── Creación de dataset muy desbalanceado (1:10) ──────────────────────────────
X_imb, y_imb = make_classification(
    n_samples=5000, n_features=10, n_informative=5,
    weights=[0.9, 0.1],  # 90% clase 0, 10% clase 1
    random_state=42
)
print(f"\nDistribución: {np.bincount(y_imb)} ({np.bincount(y_imb)/len(y_imb)*100}%)")

X_tr_imb, X_te_imb, y_tr_imb, y_te_imb = train_test_split(
    X_imb, y_imb, test_size=0.2, stratify=y_imb, random_state=42
)

# Demostración del problema: Accuracy engañosa
clf_naive = LogisticRegression(max_iter=1000)
clf_naive.fit(X_tr_imb, y_tr_imb)
y_pred_naive = clf_naive.predict(X_te_imb)

print("\nModelo SIN corrección de desbalance:")
print(f"  Accuracy: {accuracy_score(y_te_imb, y_pred_naive):.4f}  ← parece bien!")
print(f"  F1:       {f1_score(y_te_imb, y_pred_naive):.4f}")
print(f"  Recall:   {recall_score(y_te_imb, y_pred_naive):.4f}  ← terrible")
print(f"  MCC:      {matthews_corrcoef(y_te_imb, y_pred_naive):.4f}")

# ── Técnicas de balanceo ──────────────────────────────────────────────────────
from sklearn.utils.class_weight import compute_class_weight

# 1. class_weight='balanced'
clf_weighted = LogisticRegression(class_weight='balanced', max_iter=1000)
clf_weighted.fit(X_tr_imb, y_tr_imb)
y_pred_w = clf_weighted.predict(X_te_imb)

# 2. SMOTE (Synthetic Minority Oversampling)
try:
    from imblearn.over_sampling import SMOTE
    from imblearn.pipeline import Pipeline as ImbPipeline

    pipe_smote = ImbPipeline([
        ('smote', SMOTE(random_state=42)),
        ('sc',    StandardScaler()),
        ('clf',   LogisticRegression(max_iter=1000))
    ])
    pipe_smote.fit(X_tr_imb, y_tr_imb)
    y_pred_smote = pipe_smote.predict(X_te_imb)
    smote_ok = True
except ImportError:
    print("  (imbalanced-learn no instalado — ejecuta: pip install imbalanced-learn)")
    smote_ok = False

# 3. Random Forest con class_weight (muy efectivo)
rf_balanced = RandomForestClassifier(n_estimators=100, class_weight='balanced',
                                      random_state=42, n_jobs=-1)
rf_balanced.fit(X_tr_imb, y_tr_imb)
y_pred_rf_bal = rf_balanced.predict(X_te_imb)

# Comparativa
print("\n─── Comparativa de técnicas para desbalance ───")
header = f"{'Técnica':<30} {'Acc':>6} {'F1':>6} {'Recall':>8} {'MCC':>7}"
print(header)
print('─' * 60)

for nombre, y_p in [
    ('LR sin corrección',     y_pred_naive),
    ('LR class_weight=balanced', y_pred_w),
    ('RF class_weight=balanced', y_pred_rf_bal),
] + ([('LR + SMOTE',              y_pred_smote)] if smote_ok else []):
    acc = accuracy_score(y_te_imb, y_p)
    f1  = f1_score(y_te_imb, y_p)
    rec = recall_score(y_te_imb, y_p)
    mcc = matthews_corrcoef(y_te_imb, y_p)
    print(f"  {nombre:<28} {acc:.4f} {f1:.4f}   {rec:.4f}  {mcc:.4f}")

# ── Visualización ─────────────────────────────────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

for ax, (nombre, y_p) in zip(axes, [
    ('LR sin corrección', y_pred_naive),
    ('RF balanced',       y_pred_rf_bal),
]):
    ConfusionMatrixDisplay.from_predictions(
        y_te_imb, y_p, cmap='Blues', ax=ax
    )
    ax.set_title(f'{nombre}')

plt.suptitle('Desbalance: efecto en la matriz de confusión', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m08_imbalanced.png', dpi=120)
plt.show()
```

---

## Estrategias de validación cruzada

```python
# ── Comparativa de estrategias CV ────────────────────────────────────────────
from sklearn.model_selection import (
    KFold, StratifiedKFold, RepeatedStratifiedKFold,
    LeaveOneOut, cross_val_score
)

clf_eval = Pipeline([
    ('sc', StandardScaler()),
    ('clf', RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1))
])

estrategias = {
    'KFold-5':              KFold(n_splits=5, shuffle=True, random_state=42),
    'StratifiedKFold-5':    StratifiedKFold(5, shuffle=True, random_state=42),
    'StratifiedKFold-10':   StratifiedKFold(10, shuffle=True, random_state=42),
    'RepeatedStratKFold-5×3': RepeatedStratifiedKFold(n_splits=5, n_repeats=3,
                                                       random_state=42),
}

print("\n─── Estrategias de validación cruzada (Breast Cancer) ───\n")
for nombre, cv_strat in estrategias.items():
    scores = cross_val_score(clf_eval, X, y, cv=cv_strat,
                              scoring='roc_auc', n_jobs=-1)
    print(f"  {nombre:<30} AUC = {scores.mean():.4f} ± {scores.std():.4f} "
          f"(n={len(scores)} folds)")

print("""
Cuándo usar cada estrategia:
  KFold simple:        datos balanceados, sin estado (regresión)
  StratifiedKFold:     SIEMPRE en clasificación → mantiene distribución de clases
  RepeatedStratKFold:  estimación más estable (promedia múltiples KFold)
  LOOCV:               datasets muy pequeños (< 100 muestras), muy lento
""")
```

### Curvas de aprendizaje para diagnóstico

```python
# ── Curvas de aprendizaje ─────────────────────────────────────────────────────
from sklearn.model_selection import learning_curve

def plot_learning_curve(estimador, X, y, titulo='Learning Curve',
                         cv=5, n_jobs=-1):
    train_sizes = np.linspace(0.1, 1.0, 10)
    train_sz, train_sc, val_sc = learning_curve(
        estimador, X, y,
        train_sizes=train_sizes,
        cv=StratifiedKFold(cv, shuffle=True, random_state=42),
        scoring='roc_auc',
        n_jobs=n_jobs
    )

    train_mean = train_sc.mean(axis=1)
    train_std  = train_sc.std(axis=1)
    val_mean   = val_sc.mean(axis=1)
    val_std    = val_sc.std(axis=1)

    fig, ax = plt.subplots(figsize=(8, 5))
    ax.fill_between(train_sz, train_mean - train_std, train_mean + train_std, alpha=0.15, color='blue')
    ax.fill_between(train_sz, val_mean - val_std,   val_mean + val_std,   alpha=0.15, color='red')
    ax.plot(train_sz, train_mean, 'b-o', ms=4, label='Train AUC')
    ax.plot(train_sz, val_mean,   'r-o', ms=4, label='Val AUC')

    gap = train_mean[-1] - val_mean[-1]
    if val_mean[-1] < 0.85:
        diagnostico = 'SUBAJUSTE — agregar features, modelo más complejo'
    elif gap > 0.05:
        diagnostico = 'SOBREAJUSTE — regularizar, más datos, poda'
    elif val_mean[-1] > 0.95:
        diagnostico = 'Modelo EXCELENTE'
    else:
        diagnostico = 'Modelo BUENO'

    ax.set_title(f'{titulo}\nDiagnóstico: {diagnostico}', fontsize=12)
    ax.set_xlabel('Tamaño de entrenamiento')
    ax.set_ylabel('AUC-ROC')
    ax.legend()
    ax.grid(alpha=0.3)
    plt.tight_layout()
    return ax

# Comparar RF vs LR
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

for ax, (nombre, clf_lc) in zip(axes, [
    ('Random Forest',    Pipeline([('sc', StandardScaler()),
                                   ('clf', RandomForestClassifier(n_estimators=50, random_state=42))])),
    ('Logistic Reg.',    Pipeline([('sc', StandardScaler()),
                                   ('clf', LogisticRegression(C=1, max_iter=1000))])),
]):
    train_sz, train_sc, val_sc = learning_curve(
        clf_lc, X, y,
        train_sizes=np.linspace(0.1, 1.0, 10),
        cv=StratifiedKFold(5, shuffle=True, random_state=42),
        scoring='roc_auc', n_jobs=-1
    )
    train_m, val_m = train_sc.mean(1), val_sc.mean(1)
    train_s, val_s = train_sc.std(1), val_sc.std(1)

    ax.fill_between(train_sz, train_m - train_s, train_m + train_s, alpha=0.15, color='blue')
    ax.fill_between(train_sz, val_m   - val_s,   val_m   + val_s,   alpha=0.15, color='red')
    ax.plot(train_sz, train_m, 'b-o', ms=4, label='Train')
    ax.plot(train_sz, val_m,   'r-o', ms=4, label='Val')
    ax.set_title(f'Learning Curve — {nombre}')
    ax.set_xlabel('Muestras de entrenamiento')
    ax.set_ylabel('AUC-ROC')
    ax.legend()
    ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig('outputs/m08_learning_curves.png', dpi=120)
plt.show()
```

---

## Análisis de errores

```python
# ── Análisis detallado de errores ─────────────────────────────────────────────
import pandas as pd

# Predicciones con probabilidades
df_errores = pd.DataFrame({
    'y_real':     y_test,
    'y_pred':     y_pred,
    'proba_pos':  y_proba,
})
df_errores['error']    = df_errores['y_real'] != df_errores['y_pred']
df_errores['tipo']     = 'Correcto'
df_errores.loc[(df_errores['y_real']==0) & (df_errores['y_pred']==1), 'tipo'] = 'FP (Falso Positivo)'
df_errores.loc[(df_errores['y_real']==1) & (df_errores['y_pred']==0), 'tipo'] = 'FN (Falso Negativo)'

print(f"\n─── Análisis de errores ───")
print(df_errores['tipo'].value_counts())

# Distribución de probabilidades por tipo de predicción
fig, axes = plt.subplots(1, 2, figsize=(13, 4))

for tipo, color in [('FP (Falso Positivo)', 'orange'), ('FN (Falso Negativo)', 'red')]:
    mask = df_errores['tipo'] == tipo
    if mask.sum() > 0:
        axes[0].hist(df_errores[mask]['proba_pos'], bins=10, alpha=0.6,
                     color=color, label=f'{tipo} (n={mask.sum()})')

axes[0].hist(df_errores[df_errores['tipo']=='Correcto']['proba_pos'],
             bins=20, alpha=0.3, color='green', label='Correcto')
axes[0].axvline(0.5, color='black', ls='--', label='Threshold 0.5')
axes[0].set_xlabel('Probabilidad predicha (clase positiva)')
axes[0].set_ylabel('Frecuencia')
axes[0].set_title('Distribución de probabilidades por tipo de predicción')
axes[0].legend()

# Muestras de mayor incertidumbre (cerca del threshold)
df_errores['incertidumbre'] = np.abs(df_errores['proba_pos'] - 0.5)
df_dudosas = df_errores.nsmallest(20, 'incertidumbre')

axes[1].barh(range(len(df_dudosas)), df_dudosas['proba_pos'],
             color=['green' if c else 'red'
                    for c in df_dudosas['y_real'] == df_dudosas['y_pred']])
axes[1].axvline(0.5, color='black', ls='--')
axes[1].set_xlabel('Probabilidad predicha')
axes[1].set_title('Las 20 predicciones más inciertas\n(verde=correcto, rojo=error)')
axes[1].set_yticks([])

plt.tight_layout()
plt.savefig('outputs/m08_analisis_errores.png', dpi=120)
plt.show()

# Muestras más difíciles para el modelo
print("\nPredicciones más inciertas (probabilidad más cercana a 0.5):")
print(df_dudosas[['y_real', 'y_pred', 'proba_pos', 'tipo']].head(5).to_string())
```

---

## Pipeline de evaluación reutilizable

```python
# ── Función de evaluación completa ────────────────────────────────────────────
import joblib, os

def evaluar_clasificador(modelo, X_train, y_train, X_test, y_test,
                           nombre='Modelo', cv=10, guardar_en=None,
                           target_names=None):
    """
    Evaluación completa de un clasificador: entrena, evalúa, grafica y guarda.

    Parámetros
    ----------
    modelo      : estimador sklearn (puede ser Pipeline)
    X_train, y_train : datos de entrenamiento
    X_test, y_test   : datos de test
    nombre      : nombre del modelo para títulos y archivos
    cv          : número de folds StratifiedKFold
    guardar_en  : ruta para guardar el modelo (.pkl). None = no guarda
    target_names: lista de nombres de clases

    Retorna
    -------
    dict con todas las métricas
    """
    os.makedirs('outputs', exist_ok=True)

    # 1. Validación cruzada
    cv_strat = StratifiedKFold(cv, shuffle=True, random_state=42)
    cv_scores = cross_validate(
        modelo, X_train, y_train, cv=cv_strat,
        scoring=['accuracy', 'f1', 'roc_auc'],
        return_train_score=True, n_jobs=-1
    )

    # 2. Entrenamiento final y predicción
    modelo.fit(X_train, y_train)
    y_pred  = modelo.predict(X_test)
    y_proba = (modelo.predict_proba(X_test)[:, 1]
               if hasattr(modelo, 'predict_proba') else None)

    # 3. Métricas de test
    tn, fp, fn, tp = confusion_matrix(y_test, y_pred).ravel()
    metricas = {
        'cv_acc':       cv_scores['test_accuracy'].mean(),
        'cv_acc_std':   cv_scores['test_accuracy'].std(),
        'cv_f1':        cv_scores['test_f1'].mean(),
        'cv_auc':       cv_scores['test_roc_auc'].mean(),
        'test_acc':     accuracy_score(y_test, y_pred),
        'test_f1':      f1_score(y_test, y_pred),
        'test_recall':  recall_score(y_test, y_pred),
        'test_prec':    precision_score(y_test, y_pred),
        'test_mcc':     matthews_corrcoef(y_test, y_pred),
        'tp': tp, 'fp': fp, 'tn': tn, 'fn': fn,
    }
    if y_proba is not None:
        metricas['test_auc'] = roc_auc_score(y_test, y_proba)
        metricas['test_ap']  = average_precision_score(y_test, y_proba)
        metricas['brier']    = brier_score_loss(y_test, y_proba)

    # 4. Reporte
    print(f"\n{'═'*55}")
    print(f" EVALUACIÓN: {nombre}")
    print(f"{'═'*55}")
    print(f"  CV {cv}-fold  Accuracy: {metricas['cv_acc']:.4f} ± {metricas['cv_acc_std']:.4f}")
    print(f"  CV {cv}-fold  F1:       {metricas['cv_f1']:.4f}")
    print(f"  CV {cv}-fold  AUC-ROC:  {metricas['cv_auc']:.4f}")
    print(f"  ─── Test set ───")
    print(f"  Accuracy:  {metricas['test_acc']:.4f}  |  F1: {metricas['test_f1']:.4f}")
    print(f"  Precision: {metricas['test_prec']:.4f}  |  Recall: {metricas['test_recall']:.4f}")
    print(f"  MCC:       {metricas['test_mcc']:.4f}")
    if 'test_auc' in metricas:
        print(f"  AUC-ROC:   {metricas['test_auc']:.4f}  |  AP: {metricas['test_ap']:.4f}")
        print(f"  Brier:     {metricas['brier']:.4f}")
    print(f"  TP={tp} FP={fp} TN={tn} FN={fn}")
    print(classification_report(y_test, y_pred, target_names=target_names))

    # 5. Gráficas
    fig = plt.figure(figsize=(16, 10))
    gs = fig.add_gridspec(2, 3, hspace=0.4, wspace=0.35)

    # Confusion matrix
    ax1 = fig.add_subplot(gs[0, 0])
    ConfusionMatrixDisplay.from_predictions(
        y_test, y_pred, display_labels=target_names, cmap='Blues', ax=ax1
    )
    ax1.set_title('Confusion Matrix')

    # Curva ROC
    if y_proba is not None:
        ax2 = fig.add_subplot(gs[0, 1])
        fpr, tpr, _ = roc_curve(y_test, y_proba)
        ax2.plot(fpr, tpr, 'b-', lw=2,
                 label=f'AUC={metricas["test_auc"]:.4f}')
        ax2.plot([0,1],[0,1],'k--')
        ax2.set_xlabel('FPR'); ax2.set_ylabel('TPR')
        ax2.set_title('ROC Curve')
        ax2.legend()

        # Curva PR
        ax3 = fig.add_subplot(gs[0, 2])
        prec_c, rec_c, _ = precision_recall_curve(y_test, y_proba)
        ax3.plot(rec_c, prec_c, 'r-', lw=2,
                 label=f'AP={metricas["test_ap"]:.4f}')
        ax3.set_xlabel('Recall'); ax3.set_ylabel('Precision')
        ax3.set_title('Precision-Recall Curve')
        ax3.legend()

    # Learning curve
    ax4 = fig.add_subplot(gs[1, :2])
    train_sz, train_sc, val_sc = learning_curve(
        modelo, X_train, y_train,
        train_sizes=np.linspace(0.1, 1.0, 8),
        cv=StratifiedKFold(5, shuffle=True, random_state=42),
        scoring='roc_auc', n_jobs=-1
    )
    ax4.fill_between(train_sz, train_sc.mean(1)-train_sc.std(1),
                     train_sc.mean(1)+train_sc.std(1), alpha=0.15, color='blue')
    ax4.fill_between(train_sz, val_sc.mean(1)-val_sc.std(1),
                     val_sc.mean(1)+val_sc.std(1), alpha=0.15, color='red')
    ax4.plot(train_sz, train_sc.mean(1), 'b-o', ms=4, label='Train')
    ax4.plot(train_sz, val_sc.mean(1),   'r-o', ms=4, label='Val')
    ax4.set_xlabel('Muestras'); ax4.set_ylabel('AUC-ROC')
    ax4.set_title('Learning Curve'); ax4.legend()

    # CV scores distribution
    ax5 = fig.add_subplot(gs[1, 2])
    cv_aucs = cross_val_score(modelo, X_train, y_train,
                               cv=StratifiedKFold(10, shuffle=True, random_state=42),
                               scoring='roc_auc', n_jobs=-1)
    ax5.boxplot(cv_aucs, vert=True)
    ax5.scatter([1]*len(cv_aucs), cv_aucs, alpha=0.5, color='blue', s=20)
    ax5.set_title(f'CV-10 AUC dist.\n{cv_aucs.mean():.4f} ± {cv_aucs.std():.4f}')
    ax5.set_xticks([]); ax5.set_ylabel('AUC-ROC')

    plt.suptitle(f'Evaluación Completa — {nombre}', fontsize=14, fontweight='bold')
    nombre_archivo = nombre.lower().replace(' ', '_')
    plt.savefig(f'outputs/m08_eval_{nombre_archivo}.png', dpi=120)
    plt.show()

    # 6. Guardar modelo
    if guardar_en:
        os.makedirs(os.path.dirname(guardar_en) or '.', exist_ok=True)
        joblib.dump(modelo, guardar_en)
        print(f"Modelo guardado en: {guardar_en}")

    return metricas


# ── Uso de la función reutilizable ────────────────────────────────────────────
rf_final = Pipeline([
    ('sc', StandardScaler()),
    ('clf', RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1))
])

metricas_rf = evaluar_clasificador(
    rf_final, X_train, y_train, X_test, y_test,
    nombre='Random Forest',
    cv=10,
    guardar_en='models/random_forest_m08.pkl',
    target_names=cancer.target_names
)
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   RESUMEN M08 — Evaluación de Clasificación                 │
├──────────────────────┬──────────────────────────────────────────────────────┤
│ Herramienta          │ Cuándo usarla                                        │
├──────────────────────┼──────────────────────────────────────────────────────┤
│ Accuracy             │ Solo cuando las clases están balanceadas              │
│ F1-Score             │ Cuando importa tanto Precision como Recall            │
│ F2-Score (β>1)       │ Cuando Recall importa más (ej: diagnóstico médico)    │
│ F0.5-Score (β<1)     │ Cuando Precision importa más (ej: spam filter)        │
│ MCC                  │ Mejor métrica con desbalance extremo                  │
│ Cohen's Kappa        │ Acuerdo más allá del azar (clasificación multiclase)  │
│ AUC-ROC              │ Ranking de probabilidades, datos balanceados          │
│ Average Precision    │ Clases desbalanceadas (preferir sobre AUC-ROC)        │
│ Brier Score          │ Calibración de probabilidades (0=perfecto)            │
│ Curva ROC            │ Comparar modelos, elegir threshold con Youden's J     │
│ Curva PR             │ Clases raras: fraude, anomalías, diagnóstico          │
│ Learning Curve       │ Diagnosticar sub/sobreajuste                          │
│ StratifiedKFold      │ Validación cruzada: SIEMPRE estratificada             │
│ class_weight=balanced│ Corrección de desbalance sin modificar los datos      │
│ SMOTE                │ Oversampling sintético (imbalanced-learn)             │
└──────────────────────┴──────────────────────────────────────────────────────┘
```

---

## Checkpoint M08

```python
# checkpoint_m08.py
"""
Checkpoint M08 — Evaluación de Clasificación
Ejecutar: python checkpoint_m08.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_breast_cancer, make_classification
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_validate)
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import (
    accuracy_score, f1_score, roc_auc_score, matthews_corrcoef,
    precision_score, recall_score, average_precision_score
)
import joblib, os

print("=" * 60)
print("CHECKPOINT M08 — Evaluación de Clasificación")
print("=" * 60)

# 1. Dataset balanceado
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

pipe = Pipeline([
    ('sc', StandardScaler()),
    ('clf', RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1))
])
pipe.fit(X_train, y_train)
y_pred  = pipe.predict(X_test)
y_proba = pipe.predict_proba(X_test)[:, 1]

acc  = accuracy_score(y_test, y_pred)
f1   = f1_score(y_test, y_pred)
auc  = roc_auc_score(y_test, y_proba)
mcc  = matthews_corrcoef(y_test, y_pred)

print(f"\n✓ RF Accuracy: {acc:.4f} | F1: {f1:.4f} | AUC: {auc:.4f} | MCC: {mcc:.4f}")
assert acc  > 0.94, f"Accuracy muy baja: {acc:.4f}"
assert auc  > 0.97, f"AUC muy bajo: {auc:.4f}"

# 2. StratifiedKFold
cv = StratifiedKFold(10, shuffle=True, random_state=42)
cv_scores = cross_validate(pipe, X, y, cv=cv,
                             scoring=['accuracy', 'f1', 'roc_auc'], n_jobs=-1)
print(f"✓ CV-10 AUC: {cv_scores['test_roc_auc'].mean():.4f} ± {cv_scores['test_roc_auc'].std():.4f}")
assert cv_scores['test_roc_auc'].mean() > 0.96

# 3. Dataset desbalanceado — MCC vs Accuracy
X_imb, y_imb = make_classification(n_samples=2000, n_features=8,
                                     weights=[0.9, 0.1], random_state=42)
X_tr_imb, X_te_imb, y_tr_imb, y_te_imb = train_test_split(
    X_imb, y_imb, test_size=0.2, stratify=y_imb, random_state=42
)

lr_naive    = LogisticRegression(max_iter=1000)
lr_balanced = LogisticRegression(class_weight='balanced', max_iter=1000)
lr_naive.fit(X_tr_imb, y_tr_imb)
lr_balanced.fit(X_tr_imb, y_tr_imb)

mcc_naive = matthews_corrcoef(y_te_imb, lr_naive.predict(X_te_imb))
mcc_bal   = matthews_corrcoef(y_te_imb, lr_balanced.predict(X_te_imb))
rec_naive = recall_score(y_te_imb, lr_naive.predict(X_te_imb))
rec_bal   = recall_score(y_te_imb, lr_balanced.predict(X_te_imb))

print(f"\n✓ Desbalance — LR naive:    MCC={mcc_naive:.4f} Recall={rec_naive:.4f}")
print(f"✓ Desbalance — LR balanced: MCC={mcc_bal:.4f}   Recall={rec_bal:.4f}")
assert mcc_bal > mcc_naive, "class_weight=balanced debería mejorar MCC"
assert rec_bal > rec_naive, "class_weight=balanced debería mejorar Recall"

# 4. Serialización
os.makedirs('models', exist_ok=True)
joblib.dump(pipe, 'models/checkpoint_m08_clf.pkl')
loaded = joblib.load('models/checkpoint_m08_clf.pkl')
assert np.array_equal(loaded.predict(X_test), pipe.predict(X_test))
print("✓ Serialización: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M08 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • Métricas completas (Acc, F1, AUC, MCC, Precision, Recall)")
print("  • StratifiedKFold CV-10")
print("  • Corrección de desbalance con class_weight=balanced")
print("  • MCC como métrica robusta ante desbalance")
print("  • Serialización con joblib")
print("=" * 60)
```

---

*Siguiente módulo → M09: Clustering — K-Means, DBSCAN, Clustering Jerárquico*
