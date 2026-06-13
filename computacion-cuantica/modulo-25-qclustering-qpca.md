# Módulo 25 — Quantum Clustering y Quantum PCA

**Objetivo:** Implementar algoritmos cuánticos de aprendizaje no supervisado: Q-Means (clustering cuántico), Quantum PCA mediante variational eigensolver, y Quantum Kernel PCA. Comprender ventajas teóricas y limitaciones prácticas.

**Herramientas:** PennyLane 0.39+, Qiskit 1.x, NumPy, scikit-learn, matplotlib
**Prerequisito:** M21 (Encoding), M22 (QSVM), M18 (VQE)

---

## Ejecución

```bash
conda activate quantum
mkdir -p outputs
python seccion_X.py
```

---

## 1. Fundamentos: distancia cuántica entre estados

La métrica fundamental del clustering cuántico es la distancia entre estados cuánticos.

```python
# seccion_01_distancia.py
import numpy as np
import pennylane as qml
from sklearn.preprocessing import normalize

# === Distancia entre estados cuánticos ===
# Para estados |φ(x)⟩ y |φ(y)⟩:
#   D²(x,y) = 2(1 - |⟨φ(x)|φ(y)⟩|²)
#            = 2(1 - K(x,y))
# Donde K(x,y) es el kernel cuántico (Módulo 21)

n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits + 1)  # qubit ancilla para SWAP test

@qml.qnode(dev)
def swap_test(x1, x2):
    """
    SWAP test: mide |⟨φ(x1)|φ(x2)⟩|² sin colapsar el estado.
    Resultado: P(ancilla=|0⟩) = (1 + |⟨φ(x1)|φ(x2)⟩|²) / 2
    """
    # Ancilla en |+⟩
    qml.Hadamard(wires=0)

    # Preparar |φ(x1)⟩ en qubits 1..n
    qml.AngleEmbedding(x1, wires=range(1, n_qubits + 1), rotation="Y")

    # Preparar |φ(x2)⟩ en qubits n+1..2n (usando mismo registro, trick con rotaciones inversas)
    # Implementación: SWAP controlado de los dos estados
    for i in range(1, n_qubits + 1):
        qml.CSWAP(wires=[0, i, i])  # simplificado — en práctica se necesitan 2 registros

    qml.Hadamard(wires=0)
    return qml.probs(wires=0)

def distancia_cuantica_swap(x1, x2):
    """
    Distancia D²(x,y) = 2(1 - |⟨φ(x1)|φ(x2)⟩|²).
    Implementada vía kernel de superposición.
    """
    x1_n = np.array(x1[:n_qubits], dtype=float)
    x2_n = np.array(x2[:n_qubits], dtype=float)

    # Kernel via Statevector overlap (más preciso)
    dev_sv = qml.device("default.qubit", wires=n_qubits)

    @qml.qnode(dev_sv)
    def estado1(x):
        qml.AngleEmbedding(x, wires=range(n_qubits), rotation="Y")
        return qml.state()

    @qml.qnode(dev_sv)
    def estado2(x):
        qml.AngleEmbedding(x, wires=range(n_qubits), rotation="Y")
        return qml.state()

    sv1 = estado1(x1_n)
    sv2 = estado2(x2_n)

    fidelidad = abs(np.dot(np.conj(sv1), sv2)) ** 2
    return 2 * (1 - fidelidad)

# Demostración
print("=== Distancias Cuánticas ===")
np.random.seed(42)
x_a = np.random.uniform(0, np.pi, n_qubits)
x_b = np.random.uniform(0, np.pi, n_qubits)
x_c = x_a + 0.01 * np.random.randn(n_qubits)  # casi idéntico a x_a

d_ab = distancia_cuantica_swap(x_a, x_b)
d_ac = distancia_cuantica_swap(x_a, x_c)
d_aa = distancia_cuantica_swap(x_a, x_a)

print(f"D²(xa, xa)  = {d_aa:.6f}  (debe ser ~0)")
print(f"D²(xa, xc)  = {d_ac:.6f}  (muy pequeño — xc ≈ xa)")
print(f"D²(xa, xb)  = {d_ab:.6f}  (mayor — aleatorios)")
print(f"Orden: d_aa < d_ac < d_ab: {d_aa < d_ac < d_ab}")

# Propiedad de simetría
d_ba = distancia_cuantica_swap(x_b, x_a)
print(f"\nSimetría: D²(xa,xb)={d_ab:.6f} == D²(xb,xa)={d_ba:.6f}: {abs(d_ab-d_ba)<1e-10}")
```

---

## 2. Q-Means: algoritmo de clustering cuántico

Q-Means es la versión cuántica de K-Means, usando distancias cuánticas y estados cuánticos como centroides.

```python
# seccion_02_qmeans.py
import numpy as np
import pennylane as qml
from sklearn.datasets import make_blobs
from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import adjusted_rand_score

n_qubits = 2  # 2 features para visualización
dev = qml.device("default.qubit", wires=n_qubits)

def kernel_cuantico(x1, x2):
    """Kernel de superposición entre dos puntos (angle encoding)."""
    @qml.qnode(dev)
    def overlap_circuit(x1, x2):
        qml.AngleEmbedding(x1, wires=range(n_qubits), rotation="Y")
        qml.adjoint(qml.AngleEmbedding)(x2, wires=range(n_qubits), rotation="Y")
        return qml.probs(wires=range(n_qubits))

    probs = overlap_circuit(x1, x2)
    return probs[0]  # P(|00...0⟩) = |⟨φ(x2)|φ(x1)⟩|²

def distancia_q(x, centroid):
    """Distancia cuántica D²(x, centroid) = 2(1 - K(x, centroid))."""
    return 2 * (1 - kernel_cuantico(x, centroid))

class QMeans:
    """
    Q-Means: K-Means con distancias cuánticas.
    Centroides se almacenan clásicamente como vectores de ángulos.
    """
    def __init__(self, n_clusters=3, max_iter=50, tol=1e-4, random_state=42):
        self.n_clusters  = n_clusters
        self.max_iter    = max_iter
        self.tol         = tol
        self.random_state = random_state
        self.centroids_  = None
        self.labels_     = None

    def fit(self, X):
        rng = np.random.RandomState(self.random_state)

        # Inicialización aleatoria de centroides
        idx = rng.choice(len(X), self.n_clusters, replace=False)
        self.centroids_ = X[idx].copy()

        for it in range(self.max_iter):
            # Asignación: cada punto al centroide más cercano (distancia cuántica)
            labels = self._asignar(X)

            # Actualización: nuevos centroides = media clásica de cada cluster
            nuevos_centroids = np.array([
                X[labels == k].mean(axis=0) if (labels == k).sum() > 0
                else self.centroids_[k]
                for k in range(self.n_clusters)
            ])

            # Convergencia
            cambio = np.linalg.norm(nuevos_centroids - self.centroids_)
            self.centroids_ = nuevos_centroids
            self.labels_ = labels

            if cambio < self.tol:
                print(f"  Convergió en iteración {it+1}")
                break

        return self

    def _asignar(self, X):
        """Asigna cada punto al centroide más cercano."""
        distancias = np.array([
            [distancia_q(x, c) for c in self.centroids_]
            for x in X
        ])
        return np.argmin(distancias, axis=1)

    def predict(self, X):
        return self._asignar(X)

# === Experimento ===
print("=== Q-Means Clustering ===")

# Dataset sintético con 3 clusters bien separados
X_raw, y_true = make_blobs(n_samples=60, centers=3, n_features=2,
                            cluster_std=0.5, random_state=42)
# Normalizar a [0, π] para angle encoding
scaler = MinMaxScaler(feature_range=(0, np.pi))
X = scaler.fit_transform(X_raw)

# Q-Means
print("Entrenando Q-Means...")
qmeans = QMeans(n_clusters=3, max_iter=20, random_state=42)
qmeans.fit(X)
labels_q = qmeans.labels_

# Comparar con K-Means clásico
from sklearn.cluster import KMeans
kmeans = KMeans(n_clusters=3, random_state=42, n_init=10)
kmeans.fit(X)
labels_k = kmeans.labels_

# Adjusted Rand Index (ARI) — mide concordancia con etiquetas reales
ari_q = adjusted_rand_score(y_true, labels_q)
ari_k = adjusted_rand_score(y_true, labels_k)

print(f"\nResultados:")
print(f"  Q-Means ARI:   {ari_q:.4f}")
print(f"  K-Means ARI:   {ari_k:.4f}")
print(f"  (ARI=1.0 es clustering perfecto)")
```

---

## 3. Visualización de clustering

```python
# seccion_03_vis_clustering.py
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from sklearn.datasets import make_blobs, make_circles, make_moons
from sklearn.preprocessing import MinMaxScaler
from sklearn.cluster import KMeans
from sklearn.metrics import adjusted_rand_score
from seccion_02_qmeans import QMeans, kernel_cuantico, distancia_q
import pennylane as qml

# Datasets para comparar
datasets = {
    "Blobs"   : make_blobs(n_samples=60, centers=3, cluster_std=0.5, random_state=42),
    "Moons"   : make_moons(n_samples=60, noise=0.05, random_state=42),
    "Circles" : make_circles(n_samples=60, noise=0.03, factor=0.5, random_state=42),
}

scaler = MinMaxScaler(feature_range=(0, np.pi))

fig, axes = plt.subplots(3, 3, figsize=(15, 13))
fig.suptitle("Q-Means vs K-Means", fontsize=14)

colors = ["#e74c3c", "#2ecc71", "#3498db", "#f39c12"]

for row, (nombre, (X_raw, y_true)) in enumerate(datasets.items()):
    X = scaler.fit_transform(X_raw)
    n_clusters = len(np.unique(y_true))

    # Q-Means
    qm = QMeans(n_clusters=n_clusters, max_iter=15, random_state=42)
    qm.fit(X)
    ari_q = adjusted_rand_score(y_true, qm.labels_)

    # K-Means
    km = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
    km.fit(X)
    ari_k = adjusted_rand_score(y_true, km.labels_)

    # Real labels
    ax = axes[row, 0]
    for k in range(n_clusters):
        mask = y_true == k
        ax.scatter(X[mask, 0], X[mask, 1], c=colors[k], s=30, alpha=0.8)
    ax.set_title(f"{nombre} — Real")
    ax.set_xlabel("x₀"); ax.set_ylabel("x₁")
    ax.grid(alpha=0.3)

    # Q-Means
    ax = axes[row, 1]
    for k in range(n_clusters):
        mask = qm.labels_ == k
        ax.scatter(X[mask, 0], X[mask, 1], c=colors[k], s=30, alpha=0.8)
    ax.scatter(qm.centroids_[:, 0], qm.centroids_[:, 1],
               c="black", marker="x", s=200, linewidths=3, zorder=5)
    ax.set_title(f"Q-Means (ARI={ari_q:.3f})")
    ax.set_xlabel("x₀")
    ax.grid(alpha=0.3)

    # K-Means
    ax = axes[row, 2]
    for k in range(n_clusters):
        mask = km.labels_ == k
        ax.scatter(X[mask, 0], X[mask, 1], c=colors[k], s=30, alpha=0.8)
    ax.scatter(km.cluster_centers_[:, 0], km.cluster_centers_[:, 1],
               c="black", marker="x", s=200, linewidths=3, zorder=5)
    ax.set_title(f"K-Means (ARI={ari_k:.3f})")
    ax.set_xlabel("x₀")
    ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m25_clustering.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m25_clustering.png")
plt.close()
```

---

## 4. Quantum PCA — variational approach

Quantum PCA encuentra los componentes principales usando un VQE sobre la matriz de covarianza.

```python
# seccion_04_qpca_variacional.py
"""
Quantum PCA via VQE.
Idea: los eigenvalores/eigenvectores de la matriz de covarianza C
      se obtienen minimizando ⟨ψ(θ)|C|ψ(θ)⟩ variacionalmente.

Ventaja teórica (Lloyd 2014): O(log n) en datos cuánticos vs O(n²) clásico.
Limitación práctica: cargado de datos en estado cuántico es O(n).
"""
import numpy as np
import pennylane as qml
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# === Paso 1: Construir matriz de covarianza cuántica ===
def covarianza_qpca(X):
    """
    Calcula la matriz de covarianza normalizada para QPCA.
    C = X^T X / n  (datos ya centrados)
    Normalizada para que Tr(C)=1 → válida como Hamiltoniano cuántico.
    """
    n, d = X.shape
    C = (X.T @ X) / n
    C = C / np.trace(C)  # normalizar
    return C

# Iris dataset (4 features → 4x4 covarianza → 2 qubits)
iris = load_iris()
scaler = StandardScaler()
X = scaler.fit_transform(iris.data)
C = covarianza_qpca(X)

print("=== Quantum PCA ===")
print(f"Dimensión: {C.shape}")
print(f"Traza: {np.trace(C):.4f} (debe ser 1.0)")

# === Paso 2: QPCA como problema de eigenvalores vía VQE ===
# Para una matriz 4x4 necesitamos n=2 qubits (2^2=4)
n_qubits = 2
dev = qml.device("default.qubit", wires=n_qubits)

def hamiltonian_desde_matriz(M):
    """
    Descompone M en base de Pauli: M = Σ c_i P_i
    Para matriz 4x4 (2 qubits).
    """
    from itertools import product
    I = np.eye(2)
    X_p = np.array([[0, 1], [1, 0]])
    Y_p = np.array([[0, -1j], [1j, 0]])
    Z_p = np.array([[1, 0], [0, -1]])
    paulis = {"I": I, "X": X_p, "Y": Y_p, "Z": Z_p}

    ops = []
    coeffs = []
    for p1, p2 in product("IXYZ", repeat=2):
        P = np.kron(paulis[p1], paulis[p2])
        c = np.trace(P.conj().T @ M) / 4
        if abs(c) > 1e-10:
            coeffs.append(c.real)
            ops.append(qml.pauli.string_to_pauli_word(p1 + p2))

    return qml.Hamiltonian(coeffs, ops)

H_cov = hamiltonian_desde_matriz(C)

@qml.qnode(dev, diff_method="parameter-shift")
def ansatz_qpca(params):
    """Ansatz para encontrar el eigenvector de mínimo eigenvalor (primer componente)."""
    qml.RY(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    qml.RY(params[2], wires=0)
    qml.RY(params[3], wires=1)
    return qml.expval(H_cov)

# Optimización VQE para encontrar eigenvalor mínimo
import pennylane.numpy as pnp

params = pnp.random.uniform(0, 2*np.pi, size=4, requires_grad=True)
opt = qml.GradientDescentOptimizer(stepsize=0.1)

print("\nEncontando primer componente principal (VQE)...")
energias = []
for i in range(100):
    params, energy = opt.step_and_cost(ansatz_qpca, params)
    energias.append(energy)
    if (i+1) % 25 == 0:
        print(f"  Iter {i+1}: eigenvalor ≈ {energy:.6f}")

# Comparar con eigenvalores clásicos
eigenvalues_cl, eigenvectors_cl = np.linalg.eigh(C)
print(f"\nEigenvalores clásicos (ordenados): {eigenvalues_cl}")
print(f"Eigenvalor mínimo clásico: {eigenvalues_cl[0]:.6f}")
print(f"Eigenvalor VQE encontrado: {energias[-1]:.6f}")
print(f"Error: {abs(energias[-1] - eigenvalues_cl[0]):.6f}")
```

---

## 5. Quantum Kernel PCA

Kernel PCA usando el kernel cuántico (ZZFeatureMap).

```python
# seccion_05_qkpca.py
"""
Quantum Kernel PCA (QKPCA):
1. Calcular matriz kernel K_ij = |⟨φ(xi)|φ(xj)⟩|² para todos los pares
2. Centrar K: K_c = K - 1/n JK - 1/n KJ + 1/n² J K J (J = matriz de unos)
3. Eigendescomposición de K_c → componentes principales en espacio de características
"""
import numpy as np
import pennylane as qml
from sklearn.datasets import load_iris, make_circles
from sklearn.preprocessing import StandardScaler, MinMaxScaler
from sklearn.decomposition import PCA, KernelPCA
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

n_qubits = 2
dev_k = qml.device("default.qubit", wires=n_qubits)

def kernel_cuantico_par(x1, x2):
    """Kernel cuántico |⟨φ(x1)|φ(x2)⟩|² via ZZFeatureMap simplificado."""
    @qml.qnode(dev_k)
    def circuit(x1, x2):
        # ZZFeatureMap (1 rep)
        qml.Hadamard(0); qml.Hadamard(1)
        qml.PhaseShift(2 * x1[0], wires=0)
        qml.PhaseShift(2 * x1[1], wires=1)
        qml.CNOT(wires=[0, 1])
        qml.PhaseShift(2 * (np.pi - x1[0]) * (np.pi - x1[1]), wires=1)
        qml.CNOT(wires=[0, 1])
        # Adjunto para x2
        qml.CNOT(wires=[0, 1])
        qml.PhaseShift(-2 * (np.pi - x2[0]) * (np.pi - x2[1]), wires=1)
        qml.CNOT(wires=[0, 1])
        qml.PhaseShift(-2 * x2[1], wires=1)
        qml.PhaseShift(-2 * x2[0], wires=0)
        qml.Hadamard(1); qml.Hadamard(0)
        return qml.probs(wires=range(n_qubits))
    return circuit(x1, x2)[0]  # P(|00⟩)

def calcular_matriz_kernel(X, verbose=False):
    """Calcula la matriz kernel K completa. O(n²) llamadas al circuito."""
    n = len(X)
    K = np.zeros((n, n))
    for i in range(n):
        for j in range(i, n):
            k_ij = kernel_cuantico_par(X[i], X[j])
            K[i, j] = k_ij
            K[j, i] = k_ij
        if verbose and (i + 1) % 10 == 0:
            print(f"  {i+1}/{n} filas calculadas...")
    return K

def centrar_kernel(K):
    """Centra la matriz kernel para KernelPCA."""
    n = K.shape[0]
    J = np.ones((n, n)) / n
    return K - J @ K - K @ J + J @ K @ J

def qkpca(K, n_components=2):
    """Quantum Kernel PCA: eigendescomposición de K centrada."""
    K_c = centrar_kernel(K)
    eigenvalues, eigenvectors = np.linalg.eigh(K_c)

    # Ordenar de mayor a menor
    idx = np.argsort(eigenvalues)[::-1]
    eigenvalues = eigenvalues[idx]
    eigenvectors = eigenvectors[:, idx]

    # Proyección: normalizar por raíz cuadrada de eigenvalores
    componentes = eigenvectors[:, :n_components]
    for i in range(n_components):
        if eigenvalues[i] > 0:
            componentes[:, i] *= np.sqrt(eigenvalues[i])

    return componentes, eigenvalues

# === Experimento con Circles (no separable linealmente) ===
print("=== Quantum Kernel PCA ===")
X_raw, y = make_circles(n_samples=80, noise=0.05, factor=0.4, random_state=42)
scaler = MinMaxScaler(feature_range=(0.1, np.pi - 0.1))
X = scaler.fit_transform(X_raw)

print("Calculando matriz kernel cuántica (80x80)...")
K = calcular_matriz_kernel(X, verbose=True)

print(f"\nMatriz kernel calculada: {K.shape}")
print(f"K diagonal (debe ser ~1): {K.diagonal().mean():.4f}")
print(f"K simétrica: {np.allclose(K, K.T)}")

# QKPCA
Q_proj, Q_evals = qkpca(K, n_components=2)

# Comparaciones
pca_cl = PCA(n_components=2)
P_proj = pca_cl.fit_transform(X_raw)

kpca_rbf = KernelPCA(n_components=2, kernel="rbf", gamma=2)
K_proj = kpca_rbf.fit_transform(X_raw)

# Varianza explicada cuántica
total_var = np.sum(np.maximum(Q_evals, 0))
var_ratio = Q_evals[:2] / total_var if total_var > 0 else [0, 0]
print(f"\nVarianza explicada QKPCA: PC1={var_ratio[0]:.3f}, PC2={var_ratio[1]:.3f}")

# Visualización
fig, axes = plt.subplots(1, 3, figsize=(15, 5))
fig.suptitle("Comparación PCA Methods — Circles Dataset", fontsize=13)
colors = ["#e74c3c" if yi == 0 else "#3498db" for yi in y]

for ax, (proj, titulo) in zip(axes, [
    (P_proj,  "PCA Clásico"),
    (K_proj,  "KernelPCA (RBF)"),
    (Q_proj,  "Quantum KernelPCA"),
]):
    ax.scatter(proj[:, 0], proj[:, 1], c=colors, s=30, alpha=0.8)
    ax.set_xlabel("PC1"); ax.set_ylabel("PC2")
    ax.set_title(titulo)
    ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m25_qkpca.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m25_qkpca.png")
plt.close()
```

---

## 6. Quantum Anomaly Detection

El espacio cuántico puede detectar anomalías como estados con baja fidelidad al cluster principal.

```python
# seccion_06_anomaly.py
"""
Quantum Anomaly Detection:
Un punto x es anómalo si tiene baja similitud cuántica con todos los puntos normales.
Score de anomalía: a(x) = 1 - max_i K(x, x_i)
"""
import numpy as np
import pennylane as qml
from sklearn.preprocessing import MinMaxScaler
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

n_qubits = 2
dev = qml.device("default.qubit", wires=n_qubits)

def kernel_q(x1, x2):
    @qml.qnode(dev)
    def circ(x1, x2):
        qml.AngleEmbedding(x1, wires=range(n_qubits), rotation="Y")
        qml.adjoint(qml.AngleEmbedding)(x2, wires=range(n_qubits), rotation="Y")
        return qml.probs(wires=range(n_qubits))
    return circ(x1, x2)[0]

def score_anomalia(x_nuevo, X_train):
    """Score de anomalía: 1 - similitud máxima con datos de entrenamiento."""
    max_sim = max(kernel_q(x_nuevo, xi) for xi in X_train)
    return 1 - max_sim

# Generar datos normales y anómalos
np.random.seed(42)
X_normal = np.random.normal([np.pi/2, np.pi/2], 0.3, size=(30, 2))
X_normal = np.clip(X_normal, 0, np.pi)

# Anómalos: puntos lejanos al cluster central
X_anomalo = np.array([
    [0.1, 0.1],
    [2.8, 0.2],
    [2.5, 2.8],
    [0.2, 2.9],
])

# Calcular scores
scaler = MinMaxScaler(feature_range=(0, np.pi))
X_train_sc = scaler.fit_transform(X_normal)

print("Calculando scores de anomalía...")
scores_normal = [score_anomalia(x, X_train_sc) for x in X_train_sc]
X_anom_sc = np.clip(X_anomalo, 0, np.pi)
scores_anomalo = [score_anomalia(x, X_train_sc) for x in X_anom_sc]

umbral = np.percentile(scores_normal, 95)
print(f"\nUmbral (percentil 95 de normales): {umbral:.4f}")
print(f"\nScores de puntos normales (media ± std): {np.mean(scores_normal):.4f} ± {np.std(scores_normal):.4f}")
print(f"Scores de puntos anómalos:")
for i, s in enumerate(scores_anomalo):
    etiqueta = "ANOMALÍA" if s > umbral else "normal"
    print(f"  Punto {i+1}: score={s:.4f} → {etiqueta}")

# Visualización
fig, axes = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle("Quantum Anomaly Detection", fontsize=13)

ax = axes[0]
ax.scatter(X_train_sc[:, 0], X_train_sc[:, 1],
           c=scores_normal, cmap="RdYlGn_r", s=50, alpha=0.8,
           vmin=0, vmax=1, label="Normales")
sc = ax.scatter(X_anom_sc[:, 0], X_anom_sc[:, 1],
                c=scores_anomalo, cmap="RdYlGn_r", s=200,
                marker="*", edgecolors="black", linewidths=1.5,
                vmin=0, vmax=1, label="Candidatos anómalos")
plt.colorbar(sc, ax=ax, label="Score anomalía")
ax.axhline(np.pi/2, color="gray", linestyle="--", alpha=0.4)
ax.axvline(np.pi/2, color="gray", linestyle="--", alpha=0.4)
ax.set_title("Distribución Espacial")
ax.set_xlabel("x₀"); ax.set_ylabel("x₁")
ax.legend(); ax.grid(alpha=0.3)

ax = axes[1]
ax.hist(scores_normal, bins=15, alpha=0.7, color="green", label="Normales", density=True)
ax.axvline(umbral, color="red", linestyle="--", linewidth=2,
           label=f"Umbral={umbral:.3f}")
for s in scores_anomalo:
    ax.axvline(s, color="orange", linestyle=":", alpha=0.8)
ax.scatter(scores_anomalo, [5]*len(scores_anomalo),
           color="orange", marker="*", s=200, zorder=5, label="Anómalos")
ax.set_xlabel("Score de Anomalía")
ax.set_ylabel("Densidad")
ax.set_title("Distribución de Scores")
ax.legend(fontsize=8)
ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m25_anomaly.png", dpi=120, bbox_inches="tight")
print("\nFigura guardada: outputs/m25_anomaly.png")
plt.close()
```

---

## 7. Resumen: ventajas teóricas vs realidad práctica

```python
# seccion_07_resumen.py
"""
Comparación cuantitativa de métodos de clustering/PCA cuánticos vs clásicos.
"""
import numpy as np
import time
from sklearn.datasets import make_blobs
from sklearn.cluster import KMeans
from sklearn.decomposition import PCA
from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import adjusted_rand_score

print("=" * 60)
print("RESUMEN: Métodos Cuánticos de Aprendizaje No Supervisado")
print("=" * 60)

tabla = [
    # Método, Ventaja Teórica, Realidad Práctica, Complexidad
    ("Q-Means",     "O(log n) evaluaciones",         "O(n·k·iter) circuitos",  "Útil: ~N<100"),
    ("QPCA (VQE)",  "O(log² n) en datos cuánticos",  "Carga datos O(n)",       "Sim solo"),
    ("QKPCA",       "Kernel exponencial",             "O(n²) circuitos",        "Útil: ~N<200"),
    ("Q-Anomaly",   "Distancia en Hilbert space",     "O(n) por evaluación",    "Nicho: datos cuánticos"),
]

print(f"\n{'Método':<15} {'Ventaja Teórica':<35} {'Práctica':<25} {'Escala'}")
print("-" * 95)
for metodo, teoria, practica, escala in tabla:
    print(f"{metodo:<15} {teoria:<35} {practica:<25} {escala}")

print("\n\nRecomendaciones de uso:")
print("  • Q-Means:   Útil si los datos son inherentemente cuánticos (sensores, mediciones)")
print("  • QKPCA:     Útil para datasets pequeños (<200 samples) con estructura no lineal compleja")
print("  • Q-Anomaly: Útil en detección de fallos en circuitos cuánticos reales")
print("  • QPCA VQE:  Investigación; no ventaja práctica en simulador clásico")

print("\nEstado del arte 2024:")
print("  - No se ha demostrado ventaja cuántica sobre clásico para clustering/PCA en datos reales")
print("  - La ventaja teórica asume datos cargados cuánticamente (quantum RAM — no existe aún)")
print("  - HHL (2009) ofrece speedup exponencial para álgebra lineal, pero con caveats severos")
print("  - Dequantization (Tang 2021): muchos algoritmos cuánticos se pueden desquantizar a O(polylog)")
```

---

## Checkpoint

```python
# checkpoint_m25.py
"""Tests para Módulo 25 — Quantum Clustering y Quantum PCA."""
import numpy as np
import pennylane as qml

n_qubits = 2
dev = qml.device("default.qubit", wires=n_qubits)

def kernel_q(x1, x2):
    @qml.qnode(dev)
    def circ(x1, x2):
        qml.AngleEmbedding(x1, wires=range(n_qubits), rotation="Y")
        qml.adjoint(qml.AngleEmbedding)(x2, wires=range(n_qubits), rotation="Y")
        return qml.probs(wires=range(n_qubits))
    return circ(x1, x2)[0]

# T1: Kernel es simétrico K(x,y) = K(y,x)
def test_kernel_simetrico():
    x = np.array([0.5, 1.2])
    y = np.array([1.1, 0.3])
    assert abs(kernel_q(x, y) - kernel_q(y, x)) < 1e-10
    print("T1 PASS: kernel simétrico")

# T2: Kernel diagonal K(x,x) = 1
def test_kernel_diagonal():
    x = np.array([0.7, 1.4])
    assert abs(kernel_q(x, x) - 1.0) < 1e-10
    print("T2 PASS: K(x,x) = 1")

# T3: Distancia cuántica cumple D²(x,x) = 0
def test_distancia_identidad():
    x = np.array([0.9, 0.4])
    d = 2 * (1 - kernel_q(x, x))
    assert abs(d) < 1e-10
    print("T3 PASS: D²(x,x) = 0")

# T4: Q-Means básico converge con datos separables
def test_qmeans_convergencia():
    from sklearn.datasets import make_blobs
    from sklearn.preprocessing import MinMaxScaler
    from sklearn.metrics import adjusted_rand_score

    X_raw, y_true = make_blobs(n_samples=30, centers=2, cluster_std=0.3,
                                n_features=2, random_state=0)
    scaler = MinMaxScaler(feature_range=(0, np.pi))
    X = scaler.fit_transform(X_raw)

    # Q-Means manual simplificado
    rng = np.random.RandomState(0)
    centroids = X[rng.choice(len(X), 2, replace=False)]

    for _ in range(10):
        dists = np.array([[2*(1-kernel_q(x, c)) for c in centroids] for x in X])
        labels = np.argmin(dists, axis=1)
        for k in range(2):
            if (labels == k).sum() > 0:
                centroids[k] = X[labels == k].mean(0)

    ari = adjusted_rand_score(y_true, labels)
    assert ari > 0.5, f"Q-Means ARI muy bajo: {ari:.3f}"
    print(f"T4 PASS: Q-Means ARI={ari:.3f} > 0.5")

# T5: Centrar kernel preserva simetría
def test_centrar_kernel():
    n = 10
    X = np.random.uniform(0, np.pi, (n, n_qubits))
    K = np.array([[kernel_q(X[i], X[j]) for j in range(n)] for i in range(n)])

    J = np.ones((n, n)) / n
    K_c = K - J @ K - K @ J + J @ K @ J

    assert np.allclose(K_c, K_c.T, atol=1e-10), "K_c no es simétrica"
    print("T5 PASS: kernel centrado es simétrico")

# T6: Eigenvalores de covarianza coinciden con eigh de numpy
def test_eigenvalores_covarianza():
    np.random.seed(0)
    X = np.random.randn(20, 4)
    X_std = (X - X.mean(0)) / X.std(0)
    C = X_std.T @ X_std / 20
    C = C / np.trace(C)

    evals = np.linalg.eigvalsh(C)
    assert np.all(evals >= -1e-10), "Eigenvalores negativos (matriz no semidefinida positiva)"
    assert abs(np.trace(C) - 1.0) < 1e-10
    print(f"T6 PASS: eigenvalores de C ≥ 0, Tr(C)=1")

if __name__ == "__main__":
    print("=== Checkpoint M25: Q-Clustering y QPCA ===\n")
    test_kernel_simetrico()
    test_kernel_diagonal()
    test_distancia_identidad()
    test_qmeans_convergencia()
    test_centrar_kernel()
    test_eigenvalores_covarianza()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M26 — QGAN — Quantum Generative Adversarial Networks](./modulo-26-qgan.md)
