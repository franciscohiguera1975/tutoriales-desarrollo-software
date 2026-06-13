# Módulo 09 — Clustering: K-Means, DBSCAN y Clustering Jerárquico

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M08 (Evaluación de Clasificación)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Aprendizaje no supervisado — conceptos](#aprendizaje-no-supervisado)
3. [K-Means](#k-means)
4. [Cómo elegir K — Elbow y Silhouette](#cómo-elegir-k)
5. [K-Medoids y MiniBatch K-Means](#k-medoids-y-minibatch-k-means)
6. [DBSCAN — clustering basado en densidad](#dbscan)
7. [Clustering Jerárquico (Aglomerativo)](#clustering-jerárquico)
8. [Gaussian Mixture Models (GMM)](#gaussian-mixture-models)
9. [Evaluación de clustering](#evaluación-de-clustering)
10. [Clustering en la práctica — pipeline completo](#pipeline-completo)
11. [Checkpoint M09](#checkpoint-m09)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install scikit-learn-extra  # para K-Medoids (opcional)
python modulo-09-clustering.py
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
!pip install -q scikit-learn-extra
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Aprendizaje no supervisado

```
Supervisado:     X → [modelo entrenado con y] → ŷ
No supervisado:  X → [modelo descubre estructura] → grupos / representaciones

Clustering: agrupar muestras similares sin etiquetas conocidas.

Aplicaciones típicas:
  • Segmentación de clientes (marketing)
  • Detección de anomalías (fraude, fallos)
  • Compresión de datos (reducción de colores en imágenes)
  • Descubrimiento de temas en textos
  • Bioinformática (tipos de células, genes)
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import (make_blobs, make_moons, make_circles,
                               load_iris, fetch_openml)
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
import os
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

# ── Datasets de prueba ────────────────────────────────────────────────────────
np.random.seed(42)

# 1. Blobs esféricos — ideal para K-Means
X_blobs, y_blobs = make_blobs(n_samples=500, centers=4, cluster_std=0.8,
                               random_state=42)

# 2. Lunas — no convexo, K-Means falla
X_moons, y_moons = make_moons(n_samples=400, noise=0.07, random_state=42)

# 3. Círculos concéntricos — K-Means falla, DBSCAN funciona bien
X_circ, y_circ = make_circles(n_samples=400, noise=0.05, factor=0.5, random_state=42)

# 4. Dataset con ruido (outliers) — ideal para DBSCAN
X_noise = np.vstack([X_blobs, np.random.uniform(-6, 8, size=(50, 2))])

# Visualización de los datasets
fig, axes = plt.subplots(1, 4, figsize=(18, 4))
for ax, (X, y, titulo) in zip(axes, [
    (X_blobs, y_blobs, 'Blobs (K-Means ideal)'),
    (X_moons, y_moons, 'Lunas (K-Means falla)'),
    (X_circ,  y_circ,  'Círculos (K-Means falla)'),
    (X_noise, np.zeros(len(X_noise)), 'Blobs + ruido (DBSCAN)'),
]):
    ax.scatter(X[:, 0], X[:, 1], c=y, cmap='tab10', s=10, alpha=0.8)
    ax.set_title(titulo, fontsize=10)
    ax.axis('off')

plt.suptitle('Datasets para clustering', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m09_datasets.png', dpi=120)
plt.show()
```

---

## K-Means

### Algoritmo

```
Inicialización: elige K centroides al azar (o con K-Means++)
Iteración hasta convergencia:
  E-step: asigna cada punto al centroide más cercano
  M-step: recalcula los centroides como la media de los puntos asignados

Objetivo: minimizar la inercia (suma de distancias al cuadrado)
  inercia = Σᵢ Σⱼ ‖xᵢ − μⱼ‖² para todos los puntos xᵢ del cluster j
```

```python
from sklearn.cluster import KMeans

# ── K-Means básico ────────────────────────────────────────────────────────────
X_scaled = StandardScaler().fit_transform(X_blobs)

km = KMeans(n_clusters=4, init='k-means++', n_init=10, random_state=42)
km.fit(X_scaled)
etiquetas = km.labels_

print(f"K-Means (K=4):")
print(f"  Inercia: {km.inertia_:.2f}")
print(f"  Iteraciones: {km.n_iter_}")
print(f"  Tamaños de clusters: {np.bincount(etiquetas)}")

# Visualización con centroides
fig, axes = plt.subplots(1, 2, figsize=(13, 5))

axes[0].scatter(X_scaled[:, 0], X_scaled[:, 1], c=etiquetas,
                cmap='tab10', s=20, alpha=0.7)
axes[0].scatter(km.cluster_centers_[:, 0], km.cluster_centers_[:, 1],
                s=200, c='red', marker='★', zorder=5, label='Centroides')
axes[0].set_title(f'K-Means++ (K=4) — Inercia={km.inertia_:.1f}')
axes[0].legend()

# Problema con K-Means en datos no convexos
km_moons = KMeans(n_clusters=2, random_state=42)
km_moons.fit(X_moons)
axes[1].scatter(X_moons[:, 0], X_moons[:, 1], c=km_moons.labels_,
                cmap='bwr', s=20)
axes[1].scatter(km_moons.cluster_centers_[:, 0],
                km_moons.cluster_centers_[:, 1],
                s=200, c='black', marker='★', zorder=5)
axes[1].set_title('K-Means en Lunas — FALLA (no convexo)')

plt.tight_layout()
plt.savefig('outputs/m09_kmeans.png', dpi=120)
plt.show()
```

### K-Means: limitaciones

```
✓ Ventajas:
  • Rápido: O(n × K × iter)
  • Escala a millones de puntos (MiniBatch)
  • Fácil de implementar e interpretar

✗ Limitaciones:
  • Asume clusters ESFÉRICOS y de SIMILAR tamaño
  • Sensible a outliers (usa la media)
  • Necesita especificar K de antemano
  • Puede converger a mínimos locales (solución: n_init alto, K-Means++)
  • Falla con formas arbitrarias (lunas, círculos)
```

---

## Cómo elegir K

```python
# ── Método del codo (Elbow) ───────────────────────────────────────────────────
from sklearn.metrics import silhouette_score, davies_bouldin_score, calinski_harabasz_score

k_values = range(2, 11)
inercias, silhouettes, db_scores, ch_scores = [], [], [], []

for k in k_values:
    km_k = KMeans(n_clusters=k, init='k-means++', n_init=10, random_state=42)
    labels_k = km_k.fit_predict(X_scaled)

    inercias.append(km_k.inertia_)
    silhouettes.append(silhouette_score(X_scaled, labels_k))
    db_scores.append(davies_bouldin_score(X_scaled, labels_k))
    ch_scores.append(calinski_harabasz_score(X_scaled, labels_k))

fig, axes = plt.subplots(2, 2, figsize=(13, 9))

# Elbow
axes[0,0].plot(k_values, inercias, 'b-o', ms=7)
axes[0,0].set_xlabel('K'); axes[0,0].set_ylabel('Inercia')
axes[0,0].set_title('Elbow Method — busca el "codo"')
axes[0,0].axvline(4, color='red', ls='--', label='K óptimo (verdadero=4)')
axes[0,0].legend()

# Silhouette (maximizar → clusters bien separados y compactos)
axes[0,1].plot(k_values, silhouettes, 'g-o', ms=7)
axes[0,1].set_xlabel('K'); axes[0,1].set_ylabel('Silhouette Score')
axes[0,1].set_title('Silhouette Score ↑ mejor')
axes[0,1].axvline(k_values[np.argmax(silhouettes)], color='red', ls='--',
                   label=f'K óptimo: {k_values[np.argmax(silhouettes)]}')
axes[0,1].legend()

# Davies-Bouldin (minimizar → clusters más compactos y separados)
axes[1,0].plot(k_values, db_scores, 'r-o', ms=7)
axes[1,0].set_xlabel('K'); axes[1,0].set_ylabel('Davies-Bouldin ↓ mejor')
axes[1,0].set_title('Davies-Bouldin Index')
axes[1,0].axvline(k_values[np.argmin(db_scores)], color='blue', ls='--',
                   label=f'K óptimo: {k_values[np.argmin(db_scores)]}')
axes[1,0].legend()

# Calinski-Harabasz (maximizar → ratio varianza inter/intra cluster)
axes[1,1].plot(k_values, ch_scores, 'm-o', ms=7)
axes[1,1].set_xlabel('K'); axes[1,1].set_ylabel('Calinski-Harabasz ↑ mejor')
axes[1,1].set_title('Calinski-Harabasz Score')
axes[1,1].axvline(k_values[np.argmax(ch_scores)], color='blue', ls='--',
                   label=f'K óptimo: {k_values[np.argmax(ch_scores)]}')
axes[1,1].legend()

plt.suptitle('Métodos para elegir K óptimo', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m09_elbow_silhouette.png', dpi=120)
plt.show()

print(f"\nResumen para K_range={list(k_values)}:")
print(f"  Elbow: inspección visual")
print(f"  Silhouette máxima: K={k_values[np.argmax(silhouettes)]} ({max(silhouettes):.4f})")
print(f"  Davies-Bouldin mínimo: K={k_values[np.argmin(db_scores)]} ({min(db_scores):.4f})")
print(f"  Calinski-Harabasz máx: K={k_values[np.argmax(ch_scores)]}")
```

### Gráfico de Silhouette por muestra

```python
# ── Silhouette plot detallado ─────────────────────────────────────────────────
from matplotlib import cm

k_opt = k_values[np.argmax(silhouettes)]
km_opt = KMeans(n_clusters=k_opt, init='k-means++', n_init=10, random_state=42)
labels_opt = km_opt.fit_predict(X_scaled)

from sklearn.metrics import silhouette_samples
sil_vals = silhouette_samples(X_scaled, labels_opt)

fig, ax = plt.subplots(figsize=(8, 6))
y_lower = 10
colores = cm.tab10(np.linspace(0, 1, k_opt))

for i in range(k_opt):
    sil_i = sorted(sil_vals[labels_opt == i])
    y_upper = y_lower + len(sil_i)
    ax.fill_betweenx(np.arange(y_lower, y_upper), 0, sil_i,
                     facecolor=colores[i], edgecolor=colores[i], alpha=0.7)
    ax.text(-0.05, y_lower + len(sil_i) / 2, f'C{i}')
    y_lower = y_upper + 5

ax.axvline(silhouette_score(X_scaled, labels_opt), color='red', ls='--',
           label=f'Media={silhouette_score(X_scaled, labels_opt):.4f}')
ax.set_xlabel('Silhouette coefficient')
ax.set_title(f'Silhouette plot — K={k_opt}')
ax.legend()
plt.tight_layout()
plt.savefig('outputs/m09_silhouette_plot.png', dpi=120)
plt.show()
print("Barras anchas y hacia la derecha = clusters bien formados")
```

---

## K-Medoids y MiniBatch K-Means

```python
# ── MiniBatch K-Means: para datasets grandes ──────────────────────────────────
from sklearn.cluster import MiniBatchKMeans
import time

# Comparativa de velocidad
X_grande = np.random.randn(50_000, 10)

t0 = time.time()
km_full = KMeans(n_clusters=8, n_init=3, random_state=42)
km_full.fit(X_grande)
t_full = time.time() - t0

t0 = time.time()
km_mini = MiniBatchKMeans(n_clusters=8, batch_size=1024, n_init=3, random_state=42)
km_mini.fit(X_grande)
t_mini = time.time() - t0

print(f"\nK-Means full:       {t_full:.2f}s | Inercia: {km_full.inertia_:.0f}")
print(f"MiniBatch K-Means:  {t_mini:.2f}s | Inercia: {km_mini.inertia_:.0f}")
print(f"Speedup: {t_full/t_mini:.1f}x más rápido")

# ── K-Medoids: usa puntos reales como centroides (robusto a outliers) ─────────
try:
    from sklearn_extra.cluster import KMedoids

    km_med = KMedoids(n_clusters=4, random_state=42)
    labels_med = km_med.fit_predict(X_scaled)
    print(f"\nK-Medoids (K=4):")
    print(f"  Silhouette: {silhouette_score(X_scaled, labels_med):.4f}")
    print(f"  Los medoids son puntos reales del dataset → robusto a outliers")
except ImportError:
    print("\n(scikit-learn-extra no instalado — instalar: pip install scikit-learn-extra)")
```

---

## DBSCAN

### Conceptos

```
DBSCAN (Density-Based Spatial Clustering of Applications with Noise)

Parámetros:
  eps (ε):      radio de vecindad para considerar puntos cercanos
  min_samples:  mínimo de puntos en el radio para ser "core point"

Tipos de puntos:
  Core point:   tiene ≥ min_samples vecinos en radio ε
  Border point: está en la vecindad de un core point pero no es core
  Noise point:  no es core ni border (etiqueta = -1)

Ventajas vs K-Means:
  ✓ No necesita especificar K
  ✓ Detecta clusters de forma arbitraria
  ✓ Robusto a outliers (los etiqueta como ruido)
  ✗ Sensible a eps y min_samples
  ✗ Falla con densidades muy variables
  ✗ No escala bien a alta dimensión (maldición de la dimensionalidad)
```

```python
from sklearn.cluster import DBSCAN

# ── DBSCAN en los 4 datasets ──────────────────────────────────────────────────
datasets_dbscan = [
    (X_blobs,                 StandardScaler(), 0.3,  5, 'Blobs'),
    (X_moons,                 StandardScaler(), 0.15, 5, 'Lunas'),
    (X_circ,                  StandardScaler(), 0.15, 5, 'Círculos'),
    (X_noise[:, :2],          StandardScaler(), 0.4,  5, 'Blobs + Ruido'),
]

fig, axes = plt.subplots(2, 2, figsize=(13, 10))
axes = axes.ravel()

for ax, (X_d, scaler, eps, min_s, titulo) in zip(axes, datasets_dbscan):
    X_d_sc = scaler.fit_transform(X_d)
    db = DBSCAN(eps=eps, min_samples=min_s)
    labels_db = db.fit_predict(X_d_sc)

    n_clusters = len(set(labels_db)) - (1 if -1 in labels_db else 0)
    n_noise    = np.sum(labels_db == -1)

    # Colorear: ruido en negro
    colores = np.where(labels_db == -1, 0, labels_db + 1)
    ax.scatter(X_d_sc[:, 0], X_d_sc[:, 1], c=colores,
               cmap='tab10', s=15, alpha=0.8)
    ax.scatter(X_d_sc[labels_db == -1, 0], X_d_sc[labels_db == -1, 1],
               c='k', s=25, marker='x', label=f'Ruido ({n_noise})')
    ax.set_title(f'{titulo} — {n_clusters} clusters (ruido={n_noise})\n'
                 f'eps={eps}, min_samples={min_s}', fontsize=9)
    ax.legend(fontsize=8)

plt.suptitle('DBSCAN en distintos datasets', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m09_dbscan.png', dpi=120)
plt.show()
```

### Selección de eps con k-distance graph

```python
# ── k-distance graph para elegir eps ─────────────────────────────────────────
from sklearn.neighbors import NearestNeighbors

X_d_sc = StandardScaler().fit_transform(X_moons)
k = 5  # igual a min_samples

nbrs = NearestNeighbors(n_neighbors=k).fit(X_d_sc)
distancias, _ = nbrs.kneighbors(X_d_sc)
distancias_k  = np.sort(distancias[:, k-1])[::-1]

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(distancias_k, 'b-', lw=1.5)
ax.set_xlabel('Muestras ordenadas')
ax.set_ylabel(f'Distancia al {k}°-vecino más cercano')
ax.set_title(f'K-distance graph (k={k}) — el "codo" = eps óptimo')
ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/m09_kdistance.png', dpi=120)
plt.show()
print("Busca el punto donde la curva se 'dobla' — ese es el eps óptimo")
```

---

## Clustering Jerárquico

```
Aglomerativo (bottom-up):
  1. Cada punto comienza como su propio cluster
  2. Une iterativamente los dos clusters más cercanos
  3. Continúa hasta que queda un solo cluster
  Resultado: dendrograma — árbol de fusiones

Métricas de enlace (linkage):
  ward:     minimiza la varianza dentro de los clusters (más compactos, recomendado)
  complete: distancia máxima entre puntos de dos clusters
  average:  distancia media entre puntos de dos clusters
  single:   distancia mínima entre puntos (propenso a encadenamiento)
```

```python
from sklearn.cluster import AgglomerativeClustering
from scipy.cluster.hierarchy import dendrogram, linkage, fcluster
from scipy.spatial.distance import pdist

# ── Dendrograma ───────────────────────────────────────────────────────────────
X_dend = X_blobs[:80]  # submuestra para visualización legible
X_dend_sc = StandardScaler().fit_transform(X_dend)

Z = linkage(X_dend_sc, method='ward')

fig, axes = plt.subplots(1, 2, figsize=(15, 5))

# Dendrograma completo
dendrogram(Z, ax=axes[0], leaf_rotation=90, leaf_font_size=5)
axes[0].axhline(y=3, color='red', ls='--', label='Corte en 4 clusters')
axes[0].set_title('Dendrograma — Ward Linkage (80 muestras)')
axes[0].set_xlabel('Muestra')
axes[0].set_ylabel('Distancia de fusión')
axes[0].legend()

# Dendrograma truncado (para datasets grandes)
dendrogram(Z, ax=axes[1], truncate_mode='lastp', p=12,
           leaf_font_size=10, show_contracted=True)
axes[1].set_title('Dendrograma truncado (últimas 12 fusiones)')

plt.tight_layout()
plt.savefig('outputs/m09_dendrograma.png', dpi=120)
plt.show()

# ── AgglomerativeClustering con sklearn ──────────────────────────────────────
fig, axes = plt.subplots(1, 4, figsize=(18, 4))
linkages = ['ward', 'complete', 'average', 'single']

X_blobs_sc = StandardScaler().fit_transform(X_blobs)

for ax, link in zip(axes, linkages):
    agg = AgglomerativeClustering(n_clusters=4, linkage=link)
    labels_agg = agg.fit_predict(X_blobs_sc)
    sil = silhouette_score(X_blobs_sc, labels_agg)
    ax.scatter(X_blobs_sc[:, 0], X_blobs_sc[:, 1],
               c=labels_agg, cmap='tab10', s=10, alpha=0.8)
    ax.set_title(f'Linkage={link}\nSilhouette={sil:.4f}', fontsize=9)
    ax.axis('off')

plt.suptitle('AgglomerativeClustering — efecto del método de enlace', fontsize=12)
plt.tight_layout()
plt.savefig('outputs/m09_agg_linkage.png', dpi=120)
plt.show()
```

---

## Gaussian Mixture Models

```
GMM: en lugar de asignar cada punto a UN cluster (hard assignment como K-Means),
     calcula la PROBABILIDAD de que cada punto pertenezca a cada componente gaussiana.

Modelo:
  P(x) = Σₖ πₖ × N(x | μₖ, Σₖ)

  πₖ = peso de la componente k
  μₖ = media de la componente k
  Σₖ = covarianza de la componente k (puede ser esférica, diagonal o completa)

Ventajas sobre K-Means:
  ✓ Asignación probabilística (soft clustering)
  ✓ Permite elipses (no solo esferas)
  ✓ Parámetro covariance_type controla la forma
  ✓ AIC/BIC para seleccionar número de componentes
```

```python
from sklearn.mixture import GaussianMixture

# ── Tipos de covarianza ───────────────────────────────────────────────────────
fig, axes = plt.subplots(1, 4, figsize=(18, 4))
cov_types = ['spherical', 'diag', 'tied', 'full']

for ax, cov in zip(axes, cov_types):
    gmm = GaussianMixture(n_components=4, covariance_type=cov,
                          random_state=42, n_init=5)
    labels_gmm = gmm.fit_predict(X_blobs_sc)
    sil = silhouette_score(X_blobs_sc, labels_gmm)

    ax.scatter(X_blobs_sc[:, 0], X_blobs_sc[:, 1],
               c=labels_gmm, cmap='tab10', s=10, alpha=0.7)
    ax.set_title(f'{cov}\nSil={sil:.4f} AIC={gmm.aic(X_blobs_sc):.0f}', fontsize=9)
    ax.axis('off')

plt.suptitle('GMM — efecto del tipo de covarianza', fontsize=12)
plt.tight_layout()
plt.savefig('outputs/m09_gmm.png', dpi=120)
plt.show()

# ── Selección del número de componentes con BIC/AIC ──────────────────────────
n_components_range = range(1, 9)
bic_scores, aic_scores = [], []

for n in n_components_range:
    gmm_n = GaussianMixture(n_components=n, covariance_type='full',
                             random_state=42, n_init=5)
    gmm_n.fit(X_blobs_sc)
    bic_scores.append(gmm_n.bic(X_blobs_sc))
    aic_scores.append(gmm_n.aic(X_blobs_sc))

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(n_components_range, bic_scores, 'b-o', ms=6, label='BIC')
ax.plot(n_components_range, aic_scores, 'r-o', ms=6, label='AIC')
ax.axvline(n_components_range[np.argmin(bic_scores)], color='blue', ls='--',
           label=f'BIC óptimo: K={n_components_range[np.argmin(bic_scores)]}')
ax.set_xlabel('Número de componentes')
ax.set_ylabel('Score (menor = mejor)')
ax.set_title('GMM — Selección de K con BIC/AIC')
ax.legend()
plt.tight_layout()
plt.savefig('outputs/m09_gmm_bic.png', dpi=120)
plt.show()

# Probabilidades de pertenencia (soft clustering)
gmm_best = GaussianMixture(n_components=4, covariance_type='full',
                             random_state=42, n_init=5)
gmm_best.fit(X_blobs_sc)
proba_gmm = gmm_best.predict_proba(X_blobs_sc)
print(f"\nGMM probabilidades para los primeros 3 puntos:")
for i in range(3):
    print(f"  Punto {i}: {proba_gmm[i].round(4)}")
print("(Puntos cercanos al borde de un cluster → probabilidades mixtas)")
```

---

## Evaluación de clustering

```python
# ── Métricas internas (sin etiquetas reales) ──────────────────────────────────
from sklearn.metrics import silhouette_score, davies_bouldin_score, calinski_harabasz_score

def evaluar_clustering(X, labels, nombre='Modelo'):
    """Calcula métricas internas de clustering."""
    n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
    n_noise    = np.sum(labels == -1)
    mask       = labels != -1  # excluir ruido

    if n_clusters < 2 or mask.sum() < 10:
        print(f"{nombre}: clustering trivial o insuficientes puntos")
        return {}

    sil = silhouette_score(X[mask], labels[mask])
    db  = davies_bouldin_score(X[mask], labels[mask])
    ch  = calinski_harabasz_score(X[mask], labels[mask])

    print(f"  {nombre:<30} K={n_clusters} Ruido={n_noise:3d} "
          f"Sil={sil:.4f} DB={db:.4f} CH={ch:.1f}")
    return {'silhouette': sil, 'davies_bouldin': db, 'calinski_harabasz': ch}

# ── Métricas externas (cuando tenemos etiquetas reales) ───────────────────────
from sklearn.metrics import (adjusted_rand_score, normalized_mutual_info_score,
                              adjusted_mutual_info_score, fowlkes_mallows_score)

def evaluar_clustering_externo(y_true, y_pred, nombre='Modelo'):
    """Evalúa clustering comparando con etiquetas verdaderas."""
    ari  = adjusted_rand_score(y_true, y_pred)
    nmi  = normalized_mutual_info_score(y_true, y_pred)
    ami  = adjusted_mutual_info_score(y_true, y_pred)
    fm   = fowlkes_mallows_score(y_true, y_pred)
    print(f"  {nombre:<30} ARI={ari:.4f} NMI={nmi:.4f} AMI={ami:.4f} FM={fm:.4f}")
    return {'ARI': ari, 'NMI': nmi, 'AMI': ami, 'FM': fm}

# Comparativa en Iris (tenemos etiquetas reales)
iris = load_iris()
X_ir, y_ir = iris.data, iris.target
X_ir_sc = StandardScaler().fit_transform(X_ir)

print("\n─── Métricas internas ───")
for nombre, labels in [
    ('K-Means (K=3)',    KMeans(3, random_state=42, n_init=10).fit_predict(X_ir_sc)),
    ('GMM (K=3)',        GaussianMixture(3, random_state=42).fit_predict(X_ir_sc)),
    ('DBSCAN',           DBSCAN(eps=0.5, min_samples=5).fit_predict(X_ir_sc)),
    ('Agglomerative',    AgglomerativeClustering(3).fit_predict(X_ir_sc)),
]:
    evaluar_clustering(X_ir_sc, labels, nombre)

print("\n─── Métricas externas (vs etiquetas reales de Iris) ───")
for nombre, labels in [
    ('K-Means (K=3)',    KMeans(3, random_state=42, n_init=10).fit_predict(X_ir_sc)),
    ('GMM (K=3)',        GaussianMixture(3, random_state=42).fit_predict(X_ir_sc)),
    ('Agglomerative',    AgglomerativeClustering(3).fit_predict(X_ir_sc)),
]:
    evaluar_clustering_externo(y_ir, labels, nombre)

print("""
Métricas externas — interpretación:
  ARI  = 1 → agrupamiento perfecto, 0 → igual que azar, puede ser negativo
  NMI  ∈ [0, 1] → 1 = información mutua perfecta
  AMI  = versión ajustada por el azar de NMI
  FM   ∈ [0, 1] → media geométrica de Precision y Recall de pares
""")
```

---

## Pipeline completo

```python
# ── Segmentación de clientes — caso práctico ──────────────────────────────────
np.random.seed(42)

# Dataset sintético de clientes
n = 500
clientes = pd.DataFrame({
    'edad':           np.random.randint(18, 70, n),
    'ingreso_anual':  np.random.exponential(30000, n) + 15000,
    'gasto_mensual':  np.random.exponential(500, n) + 100,
    'frecuencia':     np.random.poisson(5, n),
    'satisfaccion':   np.random.uniform(1, 10, n),
    'tenure_meses':   np.random.randint(1, 60, n),
})
# Crear estructura real de clusters
clientes.loc[:100, 'gasto_mensual'] *= 3  # compradores frecuentes
clientes.loc[200:300, 'ingreso_anual'] *= 2  # alto ingreso

print(f"Dataset de clientes: {clientes.shape}")
print(clientes.describe().round(2))

# Preprocesamiento
sc = StandardScaler()
X_clientes = sc.fit_transform(clientes)

# Reducción de dimensionalidad para visualización
pca_2d = PCA(n_components=2, random_state=42)
X_2d = pca_2d.fit_transform(X_clientes)

print(f"\nVarianza explicada por PCA-2D: {pca_2d.explained_variance_ratio_.sum():.1%}")

# Elección de K
sils = []
for k in range(2, 8):
    km_k = KMeans(k, init='k-means++', n_init=10, random_state=42)
    sils.append(silhouette_score(X_clientes, km_k.fit_predict(X_clientes)))

K_opt = range(2, 8)[np.argmax(sils)]
print(f"K óptimo (Silhouette): {K_opt}")

# Modelo final
km_final = KMeans(n_clusters=K_opt, init='k-means++', n_init=20, random_state=42)
clientes['cluster'] = km_final.fit_predict(X_clientes)

# Perfil de cada cluster
print(f"\n─── Perfil de clusters ───")
perfil = clientes.groupby('cluster').mean().round(2)
print(perfil.to_string())

# Visualización
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

scatter = axes[0].scatter(X_2d[:, 0], X_2d[:, 1],
                           c=clientes['cluster'], cmap='tab10', s=15, alpha=0.8)
plt.colorbar(scatter, ax=axes[0])
axes[0].set_title(f'Segmentación de clientes — K={K_opt} (PCA-2D)')
axes[0].set_xlabel(f'PC1 ({pca_2d.explained_variance_ratio_[0]:.1%})')
axes[0].set_ylabel(f'PC2 ({pca_2d.explained_variance_ratio_[1]:.1%})')

# Radar chart de perfiles (normalizado)
perfil_norm = (perfil - perfil.min()) / (perfil.max() - perfil.min())
cats = perfil.columns.tolist()
N = len(cats)
angles = np.linspace(0, 2*np.pi, N, endpoint=False).tolist()
angles += angles[:1]

ax_polar = fig.add_subplot(122, polar=True)
colores_cl = plt.cm.tab10(np.linspace(0, 1, K_opt))

for i, row in perfil_norm.iterrows():
    vals = row.tolist() + row.tolist()[:1]
    ax_polar.plot(angles, vals, 'o-', lw=2, color=colores_cl[i], label=f'Cluster {i}')
    ax_polar.fill(angles, vals, alpha=0.1, color=colores_cl[i])

ax_polar.set_xticks(angles[:-1])
ax_polar.set_xticklabels(cats, size=8)
ax_polar.set_title('Perfil de clusters (normalizado)')
ax_polar.legend(loc='upper right', bbox_to_anchor=(1.3, 1.1))

plt.tight_layout()
plt.savefig('outputs/m09_segmentacion_clientes.png', dpi=120)
plt.show()

# Guardar resultados
clientes.to_csv('outputs/clientes_segmentados.csv', index=False)
import joblib
joblib.dump(km_final, 'models/kmeans_clientes.pkl')
joblib.dump(sc, 'models/scaler_clientes.pkl')
print("\nModelo y scaler guardados en models/")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RESUMEN M09 — Clustering                               │
├─────────────────┬───────────────────────────────────────────────────────────┤
│ Algoritmo       │ Características y cuándo usarlo                           │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ K-Means         │ Rápido, clusters esféricos, necesita K                     │
│ MiniBatch KM    │ K-Means escalable a millones de muestras                   │
│ K-Medoids       │ K-Means robusto a outliers (medoide = punto real)          │
│ DBSCAN          │ Forma arbitraria, detecta ruido, sin K                     │
│ Agglomerativo   │ Dendrograma para explorar estructura, O(n²)                │
│ GMM             │ Soft clustering, elipsoidal, selección con BIC             │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ Métricas int.   │ Silhouette ↑, Davies-Bouldin ↓, Calinski-Harabasz ↑       │
│ Métricas ext.   │ ARI, NMI, AMI, Fowlkes-Mallows (con etiquetas reales)     │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ Guía práctica   │ Empezar con K-Means → si falla (no convexo) → DBSCAN      │
│                 │ Para explorar estructura → Agglomerativo + dendrograma     │
│                 │ Para probabilidades → GMM                                  │
└─────────────────┴───────────────────────────────────────────────────────────┘
```

---

## Checkpoint M09

```python
# checkpoint_m09.py
"""
Checkpoint M09 — Clustering
Ejecutar: python checkpoint_m09.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import make_blobs, load_iris
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans, DBSCAN, AgglomerativeClustering
from sklearn.mixture import GaussianMixture
from sklearn.metrics import silhouette_score, adjusted_rand_score
import joblib, os

print("=" * 60)
print("CHECKPOINT M09 — Clustering")
print("=" * 60)

# 1. K-Means
X_blobs, y_blobs = make_blobs(n_samples=300, centers=4, cluster_std=0.8,
                               random_state=42)
X_sc = StandardScaler().fit_transform(X_blobs)

km = KMeans(n_clusters=4, init='k-means++', n_init=10, random_state=42)
labels_km = km.fit_predict(X_sc)
sil_km = silhouette_score(X_sc, labels_km)
ari_km = adjusted_rand_score(y_blobs, labels_km)

print(f"\n✓ K-Means (K=4): Silhouette={sil_km:.4f} | ARI={ari_km:.4f}")
assert sil_km > 0.60, f"Silhouette muy baja: {sil_km:.4f}"
assert ari_km > 0.90, f"ARI muy bajo: {ari_km:.4f}"

# 2. Selección de K (Silhouette)
sils = []
for k in range(2, 8):
    km_k = KMeans(k, n_init=5, random_state=42)
    sils.append(silhouette_score(X_sc, km_k.fit_predict(X_sc)))
k_opt = range(2, 8)[np.argmax(sils)]
print(f"✓ K óptimo por Silhouette: {k_opt} (verdadero=4)")
assert k_opt == 4, f"K óptimo incorrecto: {k_opt}"

# 3. DBSCAN
from sklearn.datasets import make_moons
X_moons, y_moons = make_moons(n_samples=300, noise=0.07, random_state=42)
X_moons_sc = StandardScaler().fit_transform(X_moons)

db = DBSCAN(eps=0.15, min_samples=5)
labels_db = db.fit_predict(X_moons_sc)
n_clusters = len(set(labels_db)) - (1 if -1 in labels_db else 0)
n_noise    = np.sum(labels_db == -1)
print(f"✓ DBSCAN en lunas: {n_clusters} clusters | ruido={n_noise}")
assert n_clusters == 2, f"DBSCAN debería encontrar 2 clusters: {n_clusters}"

# 4. GMM con BIC
iris = load_iris()
X_ir = StandardScaler().fit_transform(iris.data)
bics = [GaussianMixture(k, random_state=42).fit(X_ir).bic(X_ir)
        for k in range(1, 7)]
k_gmm = range(1, 7)[np.argmin(bics)]
gmm = GaussianMixture(k_gmm, random_state=42)
labels_gmm = gmm.fit_predict(X_ir)
ari_gmm = adjusted_rand_score(iris.target, labels_gmm)
print(f"✓ GMM en Iris: K_BIC={k_gmm} | ARI={ari_gmm:.4f}")
assert k_gmm in [2, 3, 4], f"K_GMM inesperado: {k_gmm}"

# 5. Agglomerativo
agg = AgglomerativeClustering(n_clusters=4, linkage='ward')
labels_agg = agg.fit_predict(X_sc)
sil_agg = silhouette_score(X_sc, labels_agg)
print(f"✓ AgglomerativeClustering (Ward, K=4): Silhouette={sil_agg:.4f}")
assert sil_agg > 0.55

# 6. Serialización
os.makedirs('models', exist_ok=True)
joblib.dump(km, 'models/checkpoint_m09_kmeans.pkl')
loaded = joblib.load('models/checkpoint_m09_kmeans.pkl')
assert np.array_equal(loaded.labels_, km.labels_)
print("✓ Serialización K-Means: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M09 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • K-Means++ con Silhouette para selección de K")
print("  • DBSCAN para datos no convexos con detección de ruido")
print("  • GMM con BIC para selección del número de componentes")
print("  • AgglomerativeClustering Ward")
print("  • ARI para evaluación externa")
print("=" * 60)
```

---

*Siguiente módulo → M10: Reglas de Asociación — Apriori y FP-Growth*
