# Módulo 11 — Reducción de Dimensionalidad: PCA, LDA, t-SNE y UMAP

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M09 (Clustering)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [La maldición de la dimensionalidad](#la-maldición-de-la-dimensionalidad)
3. [PCA — Análisis de Componentes Principales](#pca)
4. [LDA — Análisis Discriminante Lineal](#lda)
5. [t-SNE — Visualización de datos de alta dimensión](#t-sne)
6. [UMAP — Uniform Manifold Approximation](#umap)
7. [Comparativa y guía de selección](#comparativa-y-guía)
8. [Reducción como paso de preprocesamiento](#reducción-como-preprocesamiento)
9. [Checkpoint M11](#checkpoint-m11)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install umap-learn  # UMAP no está en sklearn
python modulo-11-reduccion-dimensionalidad.py
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
!pip install -q umap-learn
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## La maldición de la dimensionalidad

```
Con pocas dimensiones:    datos densos, distancias significativas
Con muchas dimensiones:   datos escasos, distancias pierden sentido

Problema:
  En alta dimensión, todos los puntos tienden a estar a distancias similares.
  → Los algoritmos basados en distancia (KNN, SVM, clustering) degradan.
  → El sobreajuste aumenta con n_features >> n_samples.

Regla empírica:
  n_samples ≥ 5–10 × n_features para modelos estables

Reducción de dimensionalidad:
  • Extracción de features: transforma a un espacio de menor dimensión
  • Selección de features:  elige las features más informativas (M15)
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import warnings
warnings.filterwarnings('ignore')
import os
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

from sklearn.datasets import (load_iris, load_breast_cancer, load_digits,
                               fetch_openml, make_classification)
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.pipeline import Pipeline
from sklearn.decomposition import PCA
from sklearn.discriminant_analysis import LinearDiscriminantAnalysis as LDA
from sklearn.manifold import TSNE
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_validate)
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

# ── Datasets ───────────────────────────────────────────────────────────────────
iris  = load_iris()
X_ir, y_ir = iris.data, iris.target

digits = load_digits()
X_dg, y_dg = digits.data, digits.target  # 64 features (imágenes 8x8)

cancer = load_breast_cancer()
X_ca, y_ca = cancer.data, cancer.target

print(f"Iris:     {X_ir.shape} → target: {np.unique(y_ir)}")
print(f"Digits:   {X_dg.shape} → target: {np.unique(y_dg)}")
print(f"Cancer:   {X_ca.shape} → target: {np.unique(y_ca)}")
```

---

## PCA

### ¿Qué hace PCA?

```
PCA (Principal Component Analysis) encuentra las direcciones de MÁXIMA VARIANZA
en los datos y proyecta a ese subespacio.

Matemáticamente:
  1. Centra los datos: X_c = X - μ
  2. Calcula la matriz de covarianza: C = X_cᵀ × X_c / (n-1)
  3. Descomposición en valores propios: C = V Λ Vᵀ
     Vᵢ = eigenvectors (componentes principales)
     λᵢ = eigenvalues (varianza capturada por cada componente)
  4. Proyecta: Z = X_c × V[:, :k]   (k componentes)

Propiedades:
  ✓ Los componentes son ORTOGONALES entre sí (sin correlación)
  ✓ El primero captura la máxima varianza, el segundo la siguiente, etc.
  ✓ Lineal: útil para relaciones lineales; t-SNE/UMAP para no lineales
  ✗ Las componentes no son interpretables directamente
  ✗ Sensible a escala → siempre StandardScaler antes de PCA
```

```python
# ── PCA paso a paso ───────────────────────────────────────────────────────────
scaler = StandardScaler()
X_ir_sc = scaler.fit_transform(X_ir)

# PCA completo (todas las componentes)
pca_full = PCA()
pca_full.fit(X_ir_sc)

print("Varianza explicada por componente (Iris):")
for i, (var, ratio) in enumerate(zip(pca_full.explained_variance_,
                                      pca_full.explained_variance_ratio_)):
    print(f"  PC{i+1}: {var:.4f} ({ratio:.1%}) — acumulado: {sum(pca_full.explained_variance_ratio_[:i+1]):.1%}")

# Scree plot
fig, axes = plt.subplots(1, 2, figsize=(13, 5))

axes[0].bar(range(1, len(pca_full.explained_variance_ratio_)+1),
            pca_full.explained_variance_ratio_, color='steelblue', label='Individual')
axes[0].plot(range(1, len(pca_full.explained_variance_ratio_)+1),
             np.cumsum(pca_full.explained_variance_ratio_), 'r-o', ms=6, label='Acumulada')
axes[0].axhline(0.95, color='green', ls='--', label='95% varianza')
axes[0].set_xlabel('Componente Principal')
axes[0].set_ylabel('Varianza explicada')
axes[0].set_title('Scree Plot — Iris')
axes[0].legend()

# PCA 2D — visualización
pca_2d = PCA(n_components=2)
X_ir_pca = pca_2d.fit_transform(X_ir_sc)

colores = ['red', 'green', 'blue']
for i, cls in enumerate(iris.target_names):
    mask = y_ir == i
    axes[1].scatter(X_ir_pca[mask, 0], X_ir_pca[mask, 1],
                    c=colores[i], s=30, alpha=0.8, label=cls)

axes[1].set_xlabel(f'PC1 ({pca_2d.explained_variance_ratio_[0]:.1%})')
axes[1].set_ylabel(f'PC2 ({pca_2d.explained_variance_ratio_[1]:.1%})')
axes[1].set_title(f'PCA 2D — Iris (varianza total: {pca_2d.explained_variance_ratio_.sum():.1%})')
axes[1].legend()

plt.tight_layout()
plt.savefig('outputs/m11_pca_iris.png', dpi=120)
plt.show()
```

### PCA en Digits (64 → 2D)

```python
# ── PCA en dataset de alta dimensión: Digits ──────────────────────────────────
X_dg_sc = StandardScaler().fit_transform(X_dg)

pca_dg = PCA(n_components=2, random_state=42)
X_dg_pca = pca_dg.fit_transform(X_dg_sc)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

scatter = axes[0].scatter(X_dg_pca[:, 0], X_dg_pca[:, 1],
                           c=y_dg, cmap='tab10', s=5, alpha=0.7)
plt.colorbar(scatter, ax=axes[0], label='Dígito')
axes[0].set_xlabel(f'PC1 ({pca_dg.explained_variance_ratio_[0]:.1%})')
axes[0].set_ylabel(f'PC2 ({pca_dg.explained_variance_ratio_[1]:.1%})')
axes[0].set_title('PCA 2D — Digits (64 features → 2)')

# Varianza acumulada para elegir n_components
pca_full_dg = PCA()
pca_full_dg.fit(X_dg_sc)
cum_var = np.cumsum(pca_full_dg.explained_variance_ratio_)

axes[1].plot(range(1, len(cum_var)+1), cum_var, 'b-')
for threshold in [0.90, 0.95, 0.99]:
    n_comp = np.argmax(cum_var >= threshold) + 1
    axes[1].axhline(threshold, color='red', ls='--', alpha=0.6)
    axes[1].axvline(n_comp, color='green', ls='--', alpha=0.6)
    axes[1].text(n_comp + 0.5, threshold - 0.02, f'{threshold:.0%}→{n_comp} PCs',
                 fontsize=8)

axes[1].set_xlabel('Número de componentes')
axes[1].set_ylabel('Varianza explicada acumulada')
axes[1].set_title('¿Cuántas componentes necesito? (Digits)')
axes[1].grid(alpha=0.3)

plt.tight_layout()
plt.savefig('outputs/m11_pca_digits.png', dpi=120)
plt.show()

# Regla: varianza ≥ 95%
n_95 = np.argmax(cum_var >= 0.95) + 1
print(f"\nDigits: {n_95} componentes capturan el 95% de la varianza (de {X_dg.shape[1]})")
print(f"Reducción: {X_dg.shape[1]} → {n_95} ({100*(1 - n_95/X_dg.shape[1]):.1f}% menos features)")
```

### PCA como preprocesador — efecto en clasificación

```python
# ── PCA + clasificador: ¿mejora el desempeño? ─────────────────────────────────
cv_strat = StratifiedKFold(5, shuffle=True, random_state=42)
X_ca_sc_tr, X_ca_sc_te, y_ca_tr, y_ca_te = train_test_split(
    StandardScaler().fit_transform(X_ca), y_ca,
    test_size=0.2, stratify=y_ca, random_state=42
)

# Sin PCA
rf_sin_pca = RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1)
sc_sin = cross_validate(rf_sin_pca, StandardScaler().fit_transform(X_ca), y_ca,
                         cv=cv_strat, scoring='accuracy', n_jobs=-1)

# Con PCA (95% varianza)
for n_components in [2, 5, 10, 20, 'mle']:
    try:
        pca_n = PCA(n_components=n_components)
        pipe_pca = Pipeline([
            ('pca', pca_n),
            ('clf', RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1))
        ])
        sc_n = cross_validate(pipe_pca, X_ca_sc_tr, y_ca_tr,
                               cv=StratifiedKFold(5, shuffle=True, random_state=42),
                               scoring='accuracy', n_jobs=-1)
        print(f"  PCA n={str(n_components):<4}: Acc={sc_n['test_score'].mean():.4f} "
              f"± {sc_n['test_score'].std():.4f}")
    except Exception as e:
        print(f"  PCA n={n_components}: Error — {e}")

print(f"  Sin PCA:       Acc={sc_sin['test_score'].mean():.4f} "
      f"± {sc_sin['test_score'].std():.4f}")
print("\nNota: PCA no siempre mejora clasificadores de árbol (ya son invariantes a escala)")
```

---

## LDA

```
LDA (Linear Discriminant Analysis):
  Objetivo: encontrar la proyección que MAXIMIZA la separación entre clases
            (a diferencia de PCA que maximiza varianza total)

  Matemáticamente:
    Maximizar: Sw⁻¹ Sb   (ratio varianza inter-clase / intra-clase)
    donde Sw = within-class scatter matrix
          Sb = between-class scatter matrix

  Limitaciones:
    • Máximo K-1 componentes para K clases
    • Asume distribución gaussiana por clase
    • Asume covarianzas iguales entre clases (puede violarse en la práctica)

  LDA vs PCA:
    PCA: sin supervisión (ignora etiquetas) → compresión
    LDA: supervisado (usa etiquetas)        → mejor para clasificación
```

```python
# ── LDA en Iris (4 features → 2 componentes, max K-1=2) ─────────────────────
lda = LDA(n_components=2)
X_ir_lda = lda.fit_transform(X_ir_sc, y_ir)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Proyección LDA
for i, cls in enumerate(iris.target_names):
    mask = y_ir == i
    axes[0].scatter(X_ir_lda[mask, 0], X_ir_lda[mask, 1],
                    c=colores[i], s=40, alpha=0.8, label=cls)

axes[0].set_xlabel(f'LD1 (explicado: {lda.explained_variance_ratio_[0]:.1%})')
axes[0].set_ylabel(f'LD2 (explicado: {lda.explained_variance_ratio_[1]:.1%})')
axes[0].set_title(f'LDA 2D — Iris (total: {lda.explained_variance_ratio_.sum():.1%})')
axes[0].legend()

# Comparativa PCA vs LDA (proyección de Iris)
axes[1].scatter(X_ir_pca[:, 0], X_ir_pca[:, 1], c=y_ir, cmap='Set1',
                s=30, alpha=0.5, label='PCA', marker='o')
# Normalizar LDA para comparar escala visual
X_lda_n = (X_ir_lda - X_ir_lda.mean(0)) / X_ir_lda.std(0)
axes[1].scatter(X_lda_n[:, 0], X_lda_n[:, 1], c=y_ir, cmap='Set1',
                s=30, alpha=0.5, label='LDA', marker='^')
axes[1].set_title('PCA vs LDA — Iris (círculo=PCA, triángulo=LDA)')
axes[1].legend()

plt.tight_layout()
plt.savefig('outputs/m11_lda_iris.png', dpi=120)
plt.show()

# LDA también sirve como clasificador
lda_clf = LDA()
lda_clf.fit(X_ca_sc_tr, y_ca_tr)
print(f"\nLDA como clasificador (Cancer):")
print(f"  Test Accuracy: {accuracy_score(y_ca_te, lda_clf.predict(X_ca_sc_te)):.4f}")

sc_lda = cross_validate(LDA(), StandardScaler().fit_transform(X_ca), y_ca,
                         cv=cv_strat, scoring='accuracy', n_jobs=-1)
print(f"  CV Accuracy:   {sc_lda['test_score'].mean():.4f} ± {sc_lda['test_score'].std():.4f}")
```

---

## t-SNE

```
t-SNE (t-distributed Stochastic Neighbor Embedding):
  Objetivo: VISUALIZACIÓN en 2D/3D — preserva RELACIONES LOCALES de alta dimensión

  Algoritmo (simplificado):
    1. Calcula P(j|i): probabilidad de que j sea vecino de i en el espacio original
       (distribución Gaussiana con bandwidth σᵢ adaptativo)
    2. En 2D: calcula Q(j|i) con distribución t-Student (colas más pesadas)
    3. Optimiza posiciones 2D para que Q ≈ P (minimiza KL-divergencia)

  La distribución t-Student en 2D:
    → "empuja" los puntos muy alejados (mapea clusters distantes)
    → "atrae" los vecinos cercanos (preserva estructura local)

  Hiperparámetro perplexity ≈ número de vecinos a preservar
    5–50 típico; 30 es el default

IMPORTANTE:
  ✗ t-SNE no preserva distancias GLOBALES (solo locales)
  ✗ NO usar para reducción + clasificación (solo visualización)
  ✗ Lento con n > 10,000 (usar perplexity bajo o submuestrear)
  ✗ No reproducible sin random_state fijo
  ✓ Revela clusters y estructura que PCA no puede mostrar
```

```python
# ── t-SNE en Digits (64D → 2D) ────────────────────────────────────────────────
# Primero reducimos con PCA para acelerar t-SNE
pca_50 = PCA(n_components=50, random_state=42)
X_dg_pca50 = pca_50.fit_transform(X_dg_sc)

print("Ejecutando t-SNE... (puede tardar 1-2 min)")
tsne = TSNE(n_components=2, perplexity=30, learning_rate='auto',
            init='pca', random_state=42, n_jobs=-1)
X_dg_tsne = tsne.fit_transform(X_dg_pca50)

fig, ax = plt.subplots(figsize=(10, 8))
scatter = ax.scatter(X_dg_tsne[:, 0], X_dg_tsne[:, 1],
                     c=y_dg, cmap='tab10', s=8, alpha=0.8)
plt.colorbar(scatter, ax=ax, label='Dígito')
ax.set_title('t-SNE 2D — Digits (64 features → 2D)\nperplexity=30')
ax.axis('off')
plt.tight_layout()
plt.savefig('outputs/m11_tsne_digits.png', dpi=120)
plt.show()

# ── Efecto de la perplexity ───────────────────────────────────────────────────
# Usar submuestra para rapidez
idx_sub = np.random.choice(len(X_dg), 500, replace=False)
X_sub, y_sub = X_dg_pca50[idx_sub], y_dg[idx_sub]

perplexidades = [5, 30, 100]
fig, axes = plt.subplots(1, 3, figsize=(18, 5))

for ax, perp in zip(axes, perplexidades):
    t = TSNE(n_components=2, perplexity=perp, random_state=42, n_iter=500)
    Z = t.fit_transform(X_sub)
    ax.scatter(Z[:, 0], Z[:, 1], c=y_sub, cmap='tab10', s=10, alpha=0.8)
    ax.set_title(f'perplexity={perp}', fontsize=11)
    ax.axis('off')

plt.suptitle('Efecto de perplexity en t-SNE (500 muestras de Digits)', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m11_tsne_perplexity.png', dpi=120)
plt.show()
print("perplexity demasiado baja → muchos clusters pequeños")
print("perplexity demasiado alta → estructura global aplastada")
```

---

## UMAP

```
UMAP (Uniform Manifold Approximation and Projection):
  Más rápido que t-SNE y preserva mejor la ESTRUCTURA GLOBAL.

  Idea: modela los datos como puntos en una variedad (manifold) de Riemannian
  y construye una representación simplicial (grafo de vecindades) en alta y baja dim.

  Parámetros clave:
    n_neighbors:  número de vecinos para construir el grafo (15 default)
                  bajo → estructura local; alto → estructura global
    min_dist:     distancia mínima entre puntos en la proyección (0.1 default)
                  bajo → clusters compactos; alto → distribución más uniforme
    metric:       métrica de distancia (euclidean, cosine, etc.)

  UMAP vs t-SNE:
    ✓ UMAP es mucho más rápido
    ✓ UMAP preserva mejor la estructura global
    ✓ UMAP puede usarse como preprocesador (reproducible, más estable)
    ✗ UMAP tiene más hiperparámetros a ajustar
```

```python
# ── UMAP en Digits ────────────────────────────────────────────────────────────
try:
    import umap

    reducer_umap = umap.UMAP(n_components=2, n_neighbors=15, min_dist=0.1,
                              random_state=42, n_jobs=-1)
    X_dg_umap = reducer_umap.fit_transform(X_dg_sc)

    fig, axes = plt.subplots(1, 2, figsize=(14, 6))

    scatter1 = axes[0].scatter(X_dg_tsne[:, 0], X_dg_tsne[:, 1],
                                c=y_dg, cmap='tab10', s=8, alpha=0.8)
    plt.colorbar(scatter1, ax=axes[0])
    axes[0].set_title('t-SNE 2D — Digits')
    axes[0].axis('off')

    scatter2 = axes[1].scatter(X_dg_umap[:, 0], X_dg_umap[:, 1],
                                c=y_dg, cmap='tab10', s=8, alpha=0.8)
    plt.colorbar(scatter2, ax=axes[1])
    axes[1].set_title('UMAP 2D — Digits')
    axes[1].axis('off')

    plt.suptitle('Comparativa t-SNE vs UMAP — Digits (64 features → 2D)', fontsize=13)
    plt.tight_layout()
    plt.savefig('outputs/m11_tsne_vs_umap.png', dpi=120)
    plt.show()

    # UMAP como preprocesador para clasificación
    pca_then_rf = Pipeline([
        ('pca', PCA(n_components=20, random_state=42)),
        ('clf', RandomForestClassifier(n_estimators=50, random_state=42))
    ])
    sc_pca_rf = cross_validate(pca_then_rf, X_dg_sc, y_dg, cv=5, scoring='accuracy')

    # Nota: UMAP en pipeline requiere n_jobs=1 para evitar problemas con joblib
    X_dg_umap_30 = umap.UMAP(n_components=30, random_state=42).fit_transform(X_dg_sc)
    rf_umap = RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1)
    from sklearn.model_selection import cross_val_score
    sc_umap_rf = cross_val_score(rf_umap, X_dg_umap_30, y_dg, cv=5, scoring='accuracy')

    print(f"\nDigits — Comparativa de reducción + RF:")
    print(f"  PCA-20 + RF:   {sc_pca_rf['test_score'].mean():.4f}")
    print(f"  UMAP-30 + RF:  {sc_umap_rf.mean():.4f}")

    sc_full = cross_val_score(
        RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1),
        X_dg_sc, y_dg, cv=5, scoring='accuracy'
    )
    print(f"  Sin reducción: {sc_full.mean():.4f}")

except ImportError:
    print("\numap-learn no instalado — ejecuta: pip install umap-learn")
    print("Continuando sin UMAP...")
```

---

## Comparativa y guía

```python
# ── Comparativa visual en Iris (4D → 2D) ─────────────────────────────────────
fig, axes = plt.subplots(2, 2, figsize=(13, 10))
axes = axes.ravel()

metodos = [
    ('PCA',   PCA(n_components=2), X_ir_sc, False),
    ('LDA',   LDA(n_components=2), X_ir_sc, True),
    ('t-SNE', TSNE(n_components=2, perplexity=30, random_state=42), X_ir_sc, False),
]

# UMAP si disponible
try:
    import umap
    metodos.append(('UMAP', umap.UMAP(n_components=2, random_state=42), X_ir_sc, False))
except ImportError:
    metodos.append(('PCA (UMAP no instalado)',
                    PCA(n_components=2), X_ir_sc, False))

for ax, (nombre, reductor, X_in, supervisado) in zip(axes, metodos):
    if supervisado:
        Z = reductor.fit_transform(X_in, y_ir)
    else:
        Z = reductor.fit_transform(X_in)

    for i, cls in enumerate(iris.target_names):
        mask = y_ir == i
        ax.scatter(Z[mask, 0], Z[mask, 1], c=colores[i], s=30, alpha=0.8, label=cls)

    ax.set_title(f'{nombre} — Iris', fontsize=11)
    ax.legend(fontsize=8)
    ax.axis('off')

plt.suptitle('Comparativa PCA / LDA / t-SNE / UMAP — Iris', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m11_comparativa.png', dpi=120)
plt.show()
```

```
Guía de selección:
─────────────────────────────────────────────────────────────────────────────
Objetivo                      Método recomendado
─────────────────────────────────────────────────────────────────────────────
Preprocesamiento → clasificación  PCA (mle o % varianza) → pipeline
Clasificación supervisada         LDA (máximo K-1 componentes)
Visualización rápida              PCA-2D primero, luego t-SNE/UMAP
Visualización de alta calidad     UMAP (preserva global + local)
Datos con estructura local        t-SNE (perplexity=30–50)
Datasets muy grandes (>50k)       UMAP (mucho más rápido que t-SNE)
Interpretabilidad                 PCA (loadings, varianza explicada)
─────────────────────────────────────────────────────────────────────────────

NUNCA usar t-SNE/UMAP como preprocesadores para ML → solo visualización
SIEMPRE StandardScaler antes de PCA, LDA, t-SNE, UMAP
```

---

## Reducción como preprocesamiento

```python
# ── Pipeline completo: PCA + clasificador en Breast Cancer ────────────────────
from sklearn.model_selection import GridSearchCV

pipe_pca_clf = Pipeline([
    ('scaler', StandardScaler()),
    ('pca',    PCA(random_state=42)),
    ('clf',    RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1))
])

param_grid = {
    'pca__n_components': [5, 10, 15, 20, 25, 30],
}

grid = GridSearchCV(pipe_pca_clf, param_grid,
                    cv=StratifiedKFold(5, shuffle=True, random_state=42),
                    scoring='roc_auc', n_jobs=-1)
grid.fit(X_ca, y_ca)

print(f"\nBreast Cancer — PCA + RF (GridSearch)")
print(f"  Mejor n_components: {grid.best_params_['pca__n_components']}")
print(f"  Mejor AUC CV:       {grid.best_score_:.4f}")

# Visualización de loadings (qué features contribuyen a cada PC)
pca_visual = PCA(n_components=3)
pca_visual.fit(StandardScaler().fit_transform(X_ca))

fig, axes = plt.subplots(1, 3, figsize=(16, 4))
for i, ax in enumerate(axes):
    loadings = pca_visual.components_[i]
    idx = np.argsort(np.abs(loadings))[::-1][:10]
    colors_bar = ['red' if l < 0 else 'steelblue' for l in loadings[idx]]
    ax.barh(range(10), loadings[idx], color=colors_bar)
    ax.set_yticks(range(10))
    ax.set_yticklabels(cancer.feature_names[idx], fontsize=8)
    ax.set_title(f'PC{i+1} loadings ({pca_visual.explained_variance_ratio_[i]:.1%})')
    ax.axvline(0, color='black', lw=0.5)
    ax.invert_yaxis()

plt.suptitle('Loadings de las primeras 3 PCs — Breast Cancer', fontsize=12)
plt.tight_layout()
plt.savefig('outputs/m11_pca_loadings.png', dpi=120)
plt.show()

# Guardar pipeline con PCA
import joblib
best_pipe = grid.best_estimator_
best_pipe.fit(X_ca, y_ca)
joblib.dump(best_pipe, 'models/pipeline_pca_rf.pkl')
print("Pipeline PCA+RF guardado en models/pipeline_pca_rf.pkl")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               RESUMEN M11 — Reducción de Dimensionalidad                    │
├──────────────┬──────────────────────────────────────────────────────────────┤
│ Método       │ Uso principal                                                 │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ PCA          │ Preprocesamiento, compresión, eliminación de ruido            │
│              │ Scree plot para elegir n_components (≥ 95% varianza)          │
│ LDA          │ Supervisado, maximiza separación de clases, también clasifica │
│              │ Máximo K-1 componentes, asume gaussianidad por clase          │
│ t-SNE        │ Visualización 2D/3D de datos complejos                       │
│              │ Preserva estructura LOCAL, no usar para ML pipeline           │
│ UMAP         │ Visualización + preprocesamiento (más estable que t-SNE)      │
│              │ Preserva LOCAL y GLOBAL, mucho más rápido que t-SNE          │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Reglas       │ SIEMPRE escalar antes de reducir                              │
│              │ PCA mle = estimación automática del número de componentes     │
│              │ Pipeline: scaler → PCA → clasificador                         │
│              │ Loadings = contribución de features originales a cada PC      │
└──────────────┴──────────────────────────────────────────────────────────────┘
```

---

## Checkpoint M11

```python
# checkpoint_m11.py
"""
Checkpoint M11 — Reducción de Dimensionalidad
Ejecutar: python checkpoint_m11.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_iris, load_digits, load_breast_cancer
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.discriminant_analysis import LinearDiscriminantAnalysis as LDA
from sklearn.manifold import TSNE
from sklearn.pipeline import Pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score, StratifiedKFold
from sklearn.metrics import accuracy_score
import joblib, os

print("=" * 60)
print("CHECKPOINT M11 — Reducción de Dimensionalidad")
print("=" * 60)

# 1. PCA en Iris
iris = load_iris()
X_ir_sc = StandardScaler().fit_transform(iris.data)

pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_ir_sc)
var_total = pca.explained_variance_ratio_.sum()
print(f"\n✓ PCA 2D — Iris: varianza explicada = {var_total:.2%}")
assert var_total > 0.90, f"PCA varianza muy baja: {var_total:.2%}"
assert X_pca.shape == (150, 2)

# 2. Número de componentes para 95% varianza en Digits
digits = load_digits()
X_dg_sc = StandardScaler().fit_transform(digits.data)
pca_full = PCA()
pca_full.fit(X_dg_sc)
cum_var = np.cumsum(pca_full.explained_variance_ratio_)
n_95 = np.argmax(cum_var >= 0.95) + 1
print(f"✓ Digits: {n_95} PCs para 95% varianza (de 64 features)")
assert 20 <= n_95 <= 50, f"n_95 fuera de rango esperado: {n_95}"

# 3. LDA en Iris — debe separar mejor que PCA para clasificación
lda = LDA(n_components=2)
X_lda = lda.fit_transform(X_ir_sc, iris.target)
var_lda = lda.explained_variance_ratio_.sum()
print(f"✓ LDA 2D — Iris: varianza discriminante = {var_lda:.2%}")
assert var_lda > 0.95, f"LDA varianza muy baja: {var_lda:.2%}"
assert len(lda.explained_variance_ratio_) == 2  # max K-1 = 2

# 4. LDA como clasificador
cancer = load_breast_cancer()
X_ca = StandardScaler().fit_transform(cancer.data)
cv_scores = cross_val_score(LDA(), X_ca, cancer.target,
                             cv=StratifiedKFold(5, shuffle=True, random_state=42),
                             scoring='accuracy', n_jobs=-1)
print(f"✓ LDA como clasificador (Cancer): Acc={cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
assert cv_scores.mean() > 0.93, f"LDA Acc muy baja: {cv_scores.mean():.4f}"

# 5. PCA en pipeline + clasificador
pipe = Pipeline([
    ('sc', StandardScaler()),
    ('pca', PCA(n_components=20, random_state=42)),
    ('clf', RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1))
])
sc_pipe = cross_val_score(pipe, digits.data, digits.target,
                           cv=5, scoring='accuracy', n_jobs=-1)
print(f"✓ PCA-20 + RF (Digits): Acc={sc_pipe.mean():.4f}")
assert sc_pipe.mean() > 0.90

# 6. Serialización del pipeline
os.makedirs('models', exist_ok=True)
pipe.fit(digits.data, digits.target)
joblib.dump(pipe, 'models/checkpoint_m11_pca_pipeline.pkl')
loaded = joblib.load('models/checkpoint_m11_pca_pipeline.pkl')
assert accuracy_score(digits.target, loaded.predict(digits.data)) > 0.99
print("✓ Pipeline PCA+RF serializado y verificado")

print("\n" + "=" * 60)
print("CHECKPOINT M11 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • PCA con varianza explicada")
print("  • Selección de n_components (95% varianza)")
print("  • LDA supervisado como reductor y clasificador")
print("  • Pipeline scaler → PCA → clasificador")
print("  • Serialización del pipeline completo")
print("=" * 60)
```

---

*Siguiente módulo → M12: NLP Clásico — Bag of Words, TF-IDF, Análisis de Sentimiento*
