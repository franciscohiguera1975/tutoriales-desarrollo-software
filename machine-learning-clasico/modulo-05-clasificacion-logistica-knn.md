# Tutorial 1 — Machine Learning Clásico con Python
## Módulo 5 — Clasificación: Logistic Regression y K-Nearest Neighbors

> **Objetivo:** Implementar los dos clasificadores más didácticos del ML:
> Regresión Logística (modelo paramétrico y probabilístico) y KNN (modelo
> no paramétrico basado en similitud). Dominar las métricas de clasificación,
> la frontera de decisión y la curva ROC.
>
> **Checkpoint final:** Clasificador binario y multiclase con reporte completo
> de métricas, curva ROC y frontera de decisión visualizada.

---

## Cómo ejecutar este módulo

**Script local:**
```bash
conda activate ml-clasico
python modulo_05.py
```

**Jupyter — primera celda:**
```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
```

**Google Colab — primera celda:**
```python
# Scikit-learn, NumPy, Pandas, Matplotlib ya vienen en Colab
!pip install -q seaborn
from google.colab import drive; drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
for d in ['data', 'outputs', 'models']: os.makedirs(f'{BASE}/{d}', exist_ok=True)
os.chdir(BASE)
```

---

## 5.1 Métricas de Clasificación — El Marco Completo

Antes de entrenar cualquier modelo, hay que entender cómo se mide
el rendimiento en clasificación. Las métricas son radicalmente distintas
a las de regresión.

### La Matriz de Confusión

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.metrics import (confusion_matrix, classification_report,
                              accuracy_score, precision_score, recall_score,
                              f1_score, roc_auc_score, roc_curve,
                              ConfusionMatrixDisplay)
import os

os.makedirs('outputs', exist_ok=True)

# Ejemplo ilustrativo — clasificador de enfermedades
# 1 = Enfermo (positivo), 0 = Sano (negativo)
y_real = np.array([1,1,1,1,1, 0,0,0,0,0, 1,1,0,0,1, 0,1,0,1,0])
y_pred = np.array([1,1,1,0,0, 0,0,0,1,1, 1,0,0,0,1, 0,0,0,1,1])

cm = confusion_matrix(y_real, y_pred)
tn, fp, fn, tp = cm.ravel()

print("MATRIZ DE CONFUSIÓN")
print(f"  TN (Verdaderos Negativos): {tn}  — sano predicho como sano ✓")
print(f"  FP (Falsos Positivos):     {fp}  — sano predicho como enfermo ✗ (Error Tipo I)")
print(f"  FN (Falsos Negativos):     {fn}  — enfermo predicho como sano ✗ (Error Tipo II)")
print(f"  TP (Verdaderos Positivos): {tp}  — enfermo predicho como enfermo ✓")

# Visualización
fig, ax = plt.subplots(figsize=(6, 5))
disp = ConfusionMatrixDisplay(confusion_matrix=cm,
                               display_labels=['Sano', 'Enfermo'])
disp.plot(ax=ax, cmap='Blues', colorbar=False)
ax.set_title('Matriz de Confusión')
plt.tight_layout()
plt.savefig('outputs/confusion_matrix_ejemplo.png', dpi=150)
plt.show()
```

### Todas las métricas derivadas

```python
# Calculadas a partir de TP, TN, FP, FN
accuracy  = (tp + tn) / (tp + tn + fp + fn)
precision = tp / (tp + fp)              # de los que predije positivos, ¿cuántos lo son?
recall    = tp / (tp + fn)              # de los positivos reales, ¿cuántos encontré?
f1        = 2 * precision * recall / (precision + recall)
specificity = tn / (tn + fp)           # recall para la clase negativa

print("MÉTRICAS DE CLASIFICACIÓN")
print(f"  Accuracy    (exactitud):   {accuracy:.4f}  — % de predicciones correctas")
print(f"  Precision   (precisión):   {precision:.4f}  — calidad de los positivos predichos")
print(f"  Recall      (sensibilidad):{recall:.4f}  — capacidad de encontrar positivos")
print(f"  F1-Score:                  {f1:.4f}  — media armónica de precision y recall")
print(f"  Specificity (especificidad):{specificity:.4f} — capacidad de identificar negativos")

print("\n¿Cuándo priorizar cada métrica?")
print("  Recall alto:     diagnóstico médico (minimizar FN — no perder enfermos)")
print("  Precision alta:  spam (minimizar FP — no perder correos legítimos)")
print("  F1 alto:         balance general cuando las clases importan igual")
print("  Accuracy:        SOLO cuando las clases están balanceadas")
```

### El problema de la Accuracy con clases desbalanceadas

```python
# Dataset con 95% clase 0 y 5% clase 1
y_desbal = np.array([0]*95 + [1]*5)
y_pred_estupido = np.zeros(100, dtype=int)   # siempre predice 0

acc   = accuracy_score(y_desbal, y_pred_estupido)
rec   = recall_score(y_desbal, y_pred_estupido, zero_division=0)
f1_s  = f1_score(y_desbal, y_pred_estupido, zero_division=0)

print("Clasificador que SIEMPRE predice clase 0:")
print(f"  Accuracy:  {acc:.2f}  ← parece excelente pero es inútil")
print(f"  Recall:    {rec:.2f}   ← no detecta ningún positivo")
print(f"  F1-Score:  {f1_s:.2f}   ← revela el problema")
print("\nConclusion: en datasets desbalanceados usar F1, AUC-ROC o Recall")
```

---

## 5.2 Dataset — Social Network Ads

```python
import pandas as pd
import numpy as np
import os

os.makedirs('data', exist_ok=True)

# Generar dataset similar al Social Network Ads
np.random.seed(42)
n = 400

edad    = np.random.randint(18, 60, n)
salario = np.random.randint(15000, 150000, n)

# Regla no lineal: compra si es joven con buen salario O mayor con muy buen salario
prob = 1 / (1 + np.exp(-(
    -10
    + 0.15 * edad
    + 0.00004 * salario
    - 0.000001 * edad * salario
)))
compro = (np.random.rand(n) < prob).astype(int)

df = pd.DataFrame({
    'Age':             edad,
    'EstimatedSalary': salario,
    'Purchased':       compro
})
df.to_csv('data/Social_Network_Ads.csv', index=False)

print(f"Dataset: {df.shape}")
print(f"Compró:  {df['Purchased'].sum()} ({df['Purchased'].mean()*100:.1f}%)")
print(f"No compró: {(df['Purchased']==0).sum()} ({(df['Purchased']==0).mean()*100:.1f}%)")
print(df.describe().round(1))
```

### Preprocesamiento

```python
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

df = pd.read_csv('data/Social_Network_Ads.csv')
X  = df[['Age', 'EstimatedSalary']].values
y  = df['Purchased'].values

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.25, random_state=0
)

# Escalar — crítico para Logistic Regression y KNN
scaler   = StandardScaler()
X_train_sc = scaler.fit_transform(X_train)
X_test_sc  = scaler.transform(X_test)

print(f"Train: {X_train.shape} | Test: {X_test.shape}")
print(f"Proporción train: {np.bincount(y_train)/len(y_train)}")
```

---

## 5.3 Regresión Logística

### Concepto matemático

La regresión logística modela la **probabilidad** de la clase positiva:

```
P(y=1|x) = σ(w·x + b) = 1 / (1 + e^(-(w·x + b)))
```

La función sigmoide σ comprime cualquier valor real a [0, 1].
La frontera de decisión es lineal: `w·x + b = 0`.

**Función de pérdida — Binary Cross-Entropy (Log-Loss):**
```
L = -(1/n) Σ [yᵢ·log(ŷᵢ) + (1-yᵢ)·log(1-ŷᵢ)]
```

### Implementación

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import (accuracy_score, classification_report,
                              confusion_matrix, roc_auc_score, roc_curve)

# Modelo
log_reg = LogisticRegression(random_state=0, max_iter=1000)
log_reg.fit(X_train_sc, y_train)

# Predicciones
y_pred     = log_reg.predict(X_test_sc)
y_prob     = log_reg.predict_proba(X_test_sc)[:, 1]  # P(y=1)

# Métricas
print("REGRESIÓN LOGÍSTICA")
print("=" * 45)
print(f"Accuracy:   {accuracy_score(y_test, y_pred):.4f}")
print(f"AUC-ROC:    {roc_auc_score(y_test, y_prob):.4f}")
print(f"\nReporte completo:")
print(classification_report(y_test, y_pred,
                             target_names=['No compró', 'Compró']))

# Coeficientes (interpretabilidad)
print(f"\nCoeficientes:")
for feat, coef in zip(['Age', 'Salary'], log_reg.coef_[0]):
    print(f"  {feat:<15}: {coef:.4f}  "
          f"({'positivo' if coef>0 else 'negativo'} → "
          f"{'mayor' if coef>0 else 'menor'} probabilidad)")
print(f"  Intercepto:     {log_reg.intercept_[0]:.4f}")
```

### Probabilidades y umbral de decisión

```python
# Por defecto threshold = 0.5 — se puede ajustar según el problema
thresholds = [0.3, 0.4, 0.5, 0.6, 0.7]

print(f"{'Threshold':>12} {'Accuracy':>10} {'Precision':>10} {'Recall':>8} {'F1':>8}")
print("-" * 55)
for t in thresholds:
    y_pred_t = (y_prob >= t).astype(int)
    acc  = accuracy_score(y_test, y_pred_t)
    prec = precision_score(y_test, y_pred_t, zero_division=0)
    rec  = recall_score(y_test, y_pred_t, zero_division=0)
    f1   = f1_score(y_test, y_pred_t, zero_division=0)
    print(f"{t:>12.1f} {acc:>10.4f} {prec:>10.4f} {rec:>8.4f} {f1:>8.4f}")

print("\nUmbral bajo  → más recall (detectar más positivos)")
print("Umbral alto  → más precision (menos falsos positivos)")
```

### Curva ROC y AUC

```python
fpr, tpr, thresholds_roc = roc_curve(y_test, y_prob)
auc = roc_auc_score(y_test, y_prob)

fig, ax = plt.subplots(figsize=(7, 6))
ax.plot(fpr, tpr, color='darkorange', linewidth=2,
        label=f'ROC curve (AUC = {auc:.4f})')
ax.plot([0, 1], [0, 1], color='navy', linestyle='--',
        linewidth=1, label='Clasificador aleatorio (AUC=0.5)')
ax.fill_between(fpr, tpr, alpha=0.1, color='darkorange')

# Punto óptimo — Youden's J = TPR - FPR
j_scores   = tpr - fpr
idx_optimo = np.argmax(j_scores)
ax.scatter(fpr[idx_optimo], tpr[idx_optimo], color='red', s=100,
           zorder=5, label=f'Umbral óptimo = {thresholds_roc[idx_optimo]:.3f}')

ax.set_xlabel('Tasa de Falsos Positivos (FPR)')
ax.set_ylabel('Tasa de Verdaderos Positivos (TPR / Recall)')
ax.set_title('Curva ROC — Regresión Logística')
ax.legend(loc='lower right')
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/roc_logistica.png', dpi=150)
plt.show()

print(f"AUC = {auc:.4f}")
print("AUC = 1.0: clasificador perfecto")
print("AUC = 0.5: clasificador aleatorio (sin valor predictivo)")
print(f"Umbral óptimo (Youden): {thresholds_roc[idx_optimo]:.3f}")
```

### Frontera de decisión

```python
def plot_frontera(modelo, X, y, titulo, scaler=None, ax=None):
    """Visualiza la frontera de decisión de cualquier clasificador 2D."""
    if ax is None:
        fig, ax = plt.subplots(figsize=(7, 5))

    h = 0.02
    x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
    y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
    xx, yy = np.meshgrid(np.arange(x_min, x_max, h),
                          np.arange(y_min, y_max, h))

    grid = np.c_[xx.ravel(), yy.ravel()]
    if scaler:
        grid = scaler.transform(grid)

    Z = modelo.predict(grid).reshape(xx.shape)
    ax.contourf(xx, yy, Z, alpha=0.3, cmap='RdYlBu')
    ax.scatter(X[:, 0], X[:, 1], c=y, cmap='RdYlBu',
               edgecolors='k', s=40, zorder=5)
    ax.set_xlabel('Age'); ax.set_ylabel('Estimated Salary')
    ax.set_title(titulo)
    return ax

fig, ax = plt.subplots(figsize=(8, 6))
plot_frontera(log_reg, X_test, y_test,
              f'Logistic Regression — Frontera de Decisión\nAUC={auc:.4f}',
              scaler=scaler, ax=ax)
plt.tight_layout()
plt.savefig('outputs/frontera_logistica.png', dpi=150)
plt.show()
```

### Extensión multiclase — Softmax

```python
from sklearn.datasets import load_iris
from sklearn.linear_model import LogisticRegression

iris = load_iris()
X_iris = iris.data
y_iris = iris.target

X_tr_i, X_te_i, y_tr_i, y_te_i = train_test_split(
    X_iris, y_iris, test_size=0.3, random_state=42, stratify=y_iris
)

# multi_class='multinomial' → usa softmax (suma de probabilidades = 1)
log_multi = LogisticRegression(
    multi_class='multinomial',
    solver='lbfgs',
    max_iter=1000,
    random_state=42
)
log_multi.fit(X_tr_i, y_tr_i)

print("REGRESIÓN LOGÍSTICA MULTICLASE (Iris)")
print(classification_report(y_te_i, log_multi.predict(X_te_i),
                             target_names=iris.target_names))

# Probabilidades por clase (3 columnas, suman 1 por fila)
probs = log_multi.predict_proba(X_te_i[:5])
print("Probabilidades primeras 5 muestras:")
df_probs = pd.DataFrame(probs, columns=iris.target_names).round(3)
df_probs['Predicción'] = iris.target_names[log_multi.predict(X_te_i[:5])]
print(df_probs)
```

### Regularización en Logistic Regression

```python
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score

# C = inverso de la regularización (C alto → menos regularización)
valores_C = [0.001, 0.01, 0.1, 1, 10, 100, 1000]

print(f"{'C':>8} {'L1 CV-5':>10} {'L2 CV-5':>10}")
print("-" * 32)
for C in valores_C:
    cv_l1 = cross_val_score(
        LogisticRegression(C=C, penalty='l1', solver='liblinear',
                           max_iter=1000),
        X_train_sc, y_train, cv=5, scoring='f1'
    ).mean()
    cv_l2 = cross_val_score(
        LogisticRegression(C=C, penalty='l2', solver='lbfgs',
                           max_iter=1000),
        X_train_sc, y_train, cv=5, scoring='f1'
    ).mean()
    print(f"{C:>8} {cv_l1:>10.4f} {cv_l2:>10.4f}")
```

---

## 5.4 K-Nearest Neighbors (KNN)

### Concepto

KNN es el clasificador más intuitivo: para clasificar un punto nuevo,
busca los `k` puntos de entrenamiento más cercanos y asigna la clase
mayoritaria entre ellos.

**No hay entrenamiento real** — KNN memoriza todo el dataset.
La predicción requiere calcular distancias a todos los puntos.

**Distancias:**
```
Euclidiana: d(a,b) = √Σ(aᵢ - bᵢ)²    ← más común
Manhattan:  d(a,b) = Σ|aᵢ - bᵢ|
Minkowski:  d(a,b) = (Σ|aᵢ-bᵢ|ᵖ)^(1/p) ← generalización
```

### Implementación

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.metrics import accuracy_score, classification_report

# k=5 es el punto de partida estándar
knn = KNeighborsClassifier(n_neighbors=5, metric='minkowski', p=2)
knn.fit(X_train_sc, y_train)

y_pred_knn = knn.predict(X_test_sc)
y_prob_knn = knn.predict_proba(X_test_sc)[:, 1]

print("K-NEAREST NEIGHBORS (k=5)")
print("=" * 45)
print(f"Accuracy:  {accuracy_score(y_test, y_pred_knn):.4f}")
print(f"AUC-ROC:   {roc_auc_score(y_test, y_prob_knn):.4f}")
print(f"\nReporte:")
print(classification_report(y_test, y_pred_knn,
                             target_names=['No compró', 'Compró']))
```

### Efecto de k — El hiperparámetro más importante

```python
import matplotlib.pyplot as plt
import numpy as np

k_range     = range(1, 31)
acc_train   = []
acc_test    = []
f1_test     = []

for k in k_range:
    knn_k = KNeighborsClassifier(n_neighbors=k)
    knn_k.fit(X_train_sc, y_train)
    acc_train.append(accuracy_score(y_train, knn_k.predict(X_train_sc)))
    y_pred_k = knn_k.predict(X_test_sc)
    acc_test.append(accuracy_score(y_test, y_pred_k))
    f1_test.append(f1_score(y_test, y_pred_k, zero_division=0))

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

axes[0].plot(k_range, acc_train, 'o-', label='Train Accuracy', color='blue')
axes[0].plot(k_range, acc_test,  'o-', label='Test Accuracy',  color='orange')
axes[0].axvline(x=k_range[np.argmax(acc_test)], color='red', linestyle='--',
                label=f'k óptimo = {k_range[np.argmax(acc_test)]}')
axes[0].set_xlabel('k (número de vecinos)')
axes[0].set_ylabel('Accuracy')
axes[0].set_title('KNN — Accuracy vs k\n(k=1: overfitting, k→∞: underfitting)')
axes[0].legend(); axes[0].grid(True, alpha=0.3)

axes[1].plot(k_range, f1_test, 'o-', color='green')
axes[1].axvline(x=k_range[np.argmax(f1_test)], color='red', linestyle='--',
                label=f'k óptimo F1 = {k_range[np.argmax(f1_test)]}')
axes[1].set_xlabel('k'); axes[1].set_ylabel('F1-Score')
axes[1].set_title('KNN — F1-Score vs k')
axes[1].legend(); axes[1].grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('outputs/knn_k_optimo.png', dpi=150)
plt.show()

k_opt = k_range[np.argmax(f1_test)]
print(f"k óptimo por F1-Score: {k_opt}")
print(f"k=1:  Train={acc_train[0]:.4f}, Test={acc_test[0]:.4f}  ← overfitting")
print(f"k={k_opt}: Train={acc_train[k_opt-1]:.4f}, Test={acc_test[k_opt-1]:.4f}  ← balance")
print(f"k=30: Train={acc_train[-1]:.4f}, Test={acc_test[-1]:.4f}  ← underfitting")
```

### Buscar k óptimo con GridSearchCV

```python
from sklearn.model_selection import GridSearchCV

param_grid_knn = {
    'n_neighbors': list(range(1, 31)),
    'metric':      ['euclidean', 'manhattan', 'minkowski'],
    'weights':     ['uniform', 'distance'],
}

gs_knn = GridSearchCV(
    KNeighborsClassifier(),
    param_grid_knn,
    cv=5, scoring='f1', n_jobs=-1
)
gs_knn.fit(X_train_sc, y_train)

print(f"Mejores parámetros: {gs_knn.best_params_}")
print(f"Mejor F1 (CV-5):   {gs_knn.best_score_:.4f}")

knn_opt = gs_knn.best_estimator_
y_pred_opt = knn_opt.predict(X_test_sc)
print(f"\nKNN óptimo en test:")
print(classification_report(y_test, y_pred_opt,
                             target_names=['No compró', 'Compró']))
```

### Frontera de decisión KNN vs Logística

```python
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

plot_frontera(log_reg, X_test, y_test,
              'Logistic Regression\n(frontera lineal)',
              scaler=scaler, ax=axes[0])

plot_frontera(knn_opt, X_test, y_test,
              f'KNN (k={gs_knn.best_params_["n_neighbors"]})\n(frontera no lineal)',
              scaler=scaler, ax=axes[1])

plt.tight_layout()
plt.savefig('outputs/frontera_log_vs_knn.png', dpi=150)
plt.show()

print("Logística:  frontera SIEMPRE lineal")
print("KNN:        frontera puede ser arbitrariamente compleja")
print("Tradeoff:   KNN más flexible pero más lento en predicción")
```

---

## 5.5 Comparación y Selección de Modelo

```python
from sklearn.model_selection import cross_validate, StratifiedKFold
import pandas as pd
import numpy as np

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

modelos_cls = {
    'Logistic Regression (L2)': LogisticRegression(C=1, max_iter=1000,
                                                    random_state=42),
    'Logistic Regression (L1)': LogisticRegression(C=1, penalty='l1',
                                                    solver='liblinear',
                                                    max_iter=1000,
                                                    random_state=42),
    f'KNN (k={gs_knn.best_params_["n_neighbors"]})': knn_opt,
}

resultados_cls = []
for nombre, modelo in modelos_cls.items():
    res = cross_validate(
        modelo, X_train_sc, y_train,
        cv=cv,
        scoring=['accuracy', 'f1', 'roc_auc', 'precision', 'recall'],
        n_jobs=-1
    )
    resultados_cls.append({
        'Modelo':    nombre,
        'Accuracy':  res['test_accuracy'].mean().round(4),
        'F1':        res['test_f1'].mean().round(4),
        'AUC-ROC':   res['test_roc_auc'].mean().round(4),
        'Precision': res['test_precision'].mean().round(4),
        'Recall':    res['test_recall'].mean().round(4),
    })

df_cls = pd.DataFrame(resultados_cls).set_index('Modelo')
print("COMPARACIÓN — CV-5 Estratificado")
print("=" * 70)
print(df_cls.to_string())

# Visualización radar-style (bar chart comparativo)
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

metricas = ['Accuracy', 'F1', 'AUC-ROC']
x = np.arange(len(df_cls))
w = 0.25

colores = ['steelblue', 'orange', 'green']
for i, metrica in enumerate(metricas):
    axes[0].bar(x + i*w, df_cls[metrica], w, label=metrica,
                color=colores[i], edgecolor='white')

axes[0].set_xticks(x + w)
axes[0].set_xticklabels(df_cls.index, rotation=15, ha='right', fontsize=9)
axes[0].set_ylabel('Score')
axes[0].set_title('Accuracy vs F1 vs AUC-ROC')
axes[0].legend(); axes[0].grid(True, alpha=0.3, axis='y')
axes[0].set_ylim(0, 1)

axes[1].bar(x - w/2, df_cls['Precision'], w, label='Precision',
            color='purple', edgecolor='white')
axes[1].bar(x + w/2, df_cls['Recall'], w, label='Recall',
            color='crimson', edgecolor='white')
axes[1].set_xticks(x)
axes[1].set_xticklabels(df_cls.index, rotation=15, ha='right', fontsize=9)
axes[1].set_ylabel('Score')
axes[1].set_title('Precision vs Recall')
axes[1].legend(); axes[1].grid(True, alpha=0.3, axis='y')
axes[1].set_ylim(0, 1)

plt.tight_layout()
plt.savefig('outputs/comparacion_log_knn.png', dpi=150)
plt.show()
```

### Cuándo elegir cada uno

| Criterio | Logistic Regression | KNN |
|---|---|---|
| **Interpretabilidad** | ✅ Coeficientes claros | ❌ Caja negra |
| **Velocidad de predicción** | ✅ O(1) | ❌ O(n·d) |
| **Velocidad de entrenamiento** | ✅ Rápido | ✅ Instantáneo |
| **Frontera de decisión** | ❌ Solo lineal | ✅ No lineal |
| **Datasets grandes** | ✅ Escala bien | ❌ Lento |
| **Datos con ruido** | ✅ Robusto | ❌ Sensible |
| **Requiere scaling** | ✅ Sí | ✅ Sí (imprescindible) |
| **Probabilidades calibradas** | ✅ Sí | ⚠️ Aproximadas |

---

## ✅ Checkpoint Módulo 5

```python
# checkpoint_m05.py
import numpy as np
import pandas as pd
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import (accuracy_score, f1_score,
                              roc_auc_score, classification_report)
from sklearn.datasets import load_breast_cancer
import warnings
warnings.filterwarnings('ignore')

print("Checkpoint Módulo 5 — Clasificación: Logística + KNN")
print("=" * 55)

# Dataset: Breast Cancer (clasificación binaria real)
bc = load_breast_cancer()
X, y = bc.data, bc.target

X_tr, X_te, y_tr, y_te = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)
scaler = StandardScaler()
X_tr_sc = scaler.fit_transform(X_tr)
X_te_sc  = scaler.transform(X_te)

# 1. Logistic Regression
lr = LogisticRegression(max_iter=10000, random_state=42)
lr.fit(X_tr_sc, y_tr)
acc_lr  = accuracy_score(y_te, lr.predict(X_te_sc))
f1_lr   = f1_score(y_te, lr.predict(X_te_sc))
auc_lr  = roc_auc_score(y_te, lr.predict_proba(X_te_sc)[:, 1])
assert acc_lr > 0.90, f"Accuracy LR baja: {acc_lr}"
print(f"✓ Logistic Regression — Acc={acc_lr:.4f} F1={f1_lr:.4f} AUC={auc_lr:.4f}")

# 2. KNN
knn = KNeighborsClassifier(n_neighbors=7)
knn.fit(X_tr_sc, y_tr)
acc_knn = accuracy_score(y_te, knn.predict(X_te_sc))
f1_knn  = f1_score(y_te, knn.predict(X_te_sc))
assert acc_knn > 0.85, f"Accuracy KNN baja: {acc_knn}"
print(f"✓ KNN (k=7)           — Acc={acc_knn:.4f} F1={f1_knn:.4f}")

# 3. Cross-Validation
cv_lr  = cross_val_score(lr,  X_tr_sc, y_tr, cv=5, scoring='roc_auc').mean()
cv_knn = cross_val_score(knn, X_tr_sc, y_tr, cv=5, scoring='roc_auc').mean()
assert cv_lr > 0.95
print(f"✓ AUC CV-5: LR={cv_lr:.4f} | KNN={cv_knn:.4f}")

# 4. Comparación
mejor = 'Logística' if cv_lr >= cv_knn else 'KNN'
print(f"✓ Mejor modelo en Breast Cancer: {mejor}")

# 5. Umbral personalizado
y_prob = lr.predict_proba(X_te_sc)[:, 1]
y_pred_t03 = (y_prob >= 0.3).astype(int)
recall_t03  = f1_score(y_te, y_pred_t03, pos_label=0)
print(f"✓ Con umbral=0.3: Recall clase maligna = {recall_t03:.4f}")

print("\n✓ CHECKPOINT M05 COMPLETADO")
```

### Archivos generados

| Archivo | Descripción |
|---|---|
| `data/Social_Network_Ads.csv` | Dataset de clasificación binaria |
| `outputs/confusion_matrix_ejemplo.png` | Matriz de confusión ilustrada |
| `outputs/roc_logistica.png` | Curva ROC con AUC y umbral óptimo |
| `outputs/frontera_logistica.png` | Frontera de decisión Logística |
| `outputs/knn_k_optimo.png` | Accuracy y F1 vs k |
| `outputs/frontera_log_vs_knn.png` | Comparación visual de fronteras |
| `outputs/comparacion_log_knn.png` | Benchmark Logística vs KNN |

### Checklist

| Técnica | Implementada |
|---|---|
| Matriz de confusión: TP, TN, FP, FN | ✅ |
| Accuracy, Precision, Recall, F1, Specificity | ✅ |
| Problema de accuracy con clases desbalanceadas | ✅ |
| Regresión Logística binaria con probabilidades | ✅ |
| Ajuste de umbral de decisión | ✅ |
| Curva ROC y AUC-ROC | ✅ |
| Umbral óptimo (Youden's J) | ✅ |
| Logística multiclase (softmax, Iris) | ✅ |
| Regularización L1 y L2 en Logística | ✅ |
| KNN con distintos valores de k | ✅ |
| GridSearchCV para k óptimo y métrica | ✅ |
| Frontera de decisión visualizada | ✅ |
| Comparación con CV-5 estratificado | ✅ |

---

## Resumen

| Concepto | Estado |
|---|---|
| Arsenal completo de métricas de clasificación | ✅ |
| Regresión Logística binaria y multiclase | ✅ |
| Ajuste de umbral y curva ROC | ✅ |
| KNN — efecto de k, métricas de distancia | ✅ |
| Comparación objetiva con validación cruzada | ✅ |

**Siguiente módulo →** M06: Support Vector Machine (SVM) y Kernel SVM
