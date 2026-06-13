# Módulo 22 — Quantum SVM y Kernels Cuánticos

**Objetivo:** Implementar Quantum SVM usando kernels cuánticos basados en el
ZZFeatureMap de Qiskit, comparar con SVM clásico y entender cuándo (y si)
los kernels cuánticos ofrecen ventaja.

**Herramientas:** Qiskit, sklearn, numpy, matplotlib
**Prerequisito:** M21 (Encoding), M17 (VQC)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
pip install scikit-learn  # si no está instalado
python modulo-22-qsvm.py
```

---

## 1. Kernels cuánticos

```python
# modulo-22-qsvm.py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
from sklearn.svm import SVC
from sklearn.datasets import make_circles, make_moons, make_blobs
from sklearn.preprocessing import MinMaxScaler
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report
from qiskit import QuantumCircuit
from qiskit.circuit.library import ZZFeatureMap, ZFeatureMap
from qiskit.quantum_info import Statevector
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Kernels cuánticos — teoría
# ─────────────────────────────────────────────

print("=" * 60)
print("QUANTUM SVM Y KERNELS CUÁNTICOS")
print("=" * 60)

print("""
SVM clásico:
  Encontrar hiperplano que maximiza el margen entre clases.
  Con kernel: proyectar datos a espacio de alta dimensión donde
  son linealmente separables.

  Decision function: f(x) = Σᵢ αᵢ yᵢ K(xᵢ, x) + b
  donde K(x, x') es el kernel.

Kernels clásicos populares:
  • Linear: K(x,x') = x·x'
  • RBF/Gaussiano: K(x,x') = exp(-‖x-x'‖²/2σ²)
  • Polinomial: K(x,x') = (x·x' + c)^d

Kernel cuántico:
  K_Q(x,x') = |⟨φ(x)|φ(x')⟩|² = |⟨0|U†(x')U(x)|0⟩|²

  Donde U(x) es el feature map (circuito de encoding).
  Se evalúa ejecutando el circuito:
    1. Preparar U(x)|0⟩
    2. Aplicar U†(x')
    3. Medir: P(|0⟩) = K_Q(x, x')

Ventaja teórica (Havlíček 2019):
  Para el ZZFeatureMap, el kernel cuántico NO puede calcularse
  eficientemente de forma clásica (asumiendo que PH-collapse no ocurre).
  → Potencial ventaja exponencial para ciertos datasets.

Realidad práctica (2024):
  No se ha demostrado ventaja cuántica práctica en datasets reales.
  Los kernels cuánticos NISQ son ruidosos y pequeños.
  Mejor en teoría que en práctica actualmente.
""")
```

---

## 2. Implementación del kernel cuántico

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: Kernel cuántico con Qiskit
# ─────────────────────────────────────────────

def kernel_cuantico(x1: np.ndarray, x2: np.ndarray,
                     feature_map: QuantumCircuit) -> float:
    """
    Calcula K_Q(x1, x2) = |⟨0|U†(x2)·U(x1)|0⟩|²
    usando superposición de estados.
    """
    # Unir los dos feature maps: U†(x2) · U(x1)
    n = feature_map.num_qubits
    # Preparar parámetros para los dos circuitos
    params = list(feature_map.parameters)
    n_params = len(params)
    half = n_params // 2

    # Si el FM tiene 2*n_features params (reps=2), dividir en mitades
    if n_params == 2 * n:
        qc = QuantumCircuit(n)
        # U(x1): asignar primera mitad de params a x1
        fm_x1 = feature_map.assign_parameters(np.tile(x1, feature_map.reps))
        fm_x2 = feature_map.assign_parameters(np.tile(x2, feature_map.reps))
        # Estado U(x1)|0⟩
        sv1 = Statevector(fm_x1)
        sv2 = Statevector(fm_x2)
        return abs(sv1.inner(sv2))**2
    else:
        fm_x1 = feature_map.assign_parameters(x1[:n_params])
        fm_x2 = feature_map.assign_parameters(x2[:n_params])
        sv1 = Statevector(fm_x1)
        sv2 = Statevector(fm_x2)
        return abs(sv1.inner(sv2))**2


class QuantumKernelSVM:
    """
    SVM con kernel cuántico.
    Implementa la interfaz de sklearn.
    """
    def __init__(self, feature_map: QuantumCircuit, C: float = 1.0):
        self.feature_map = feature_map
        self.C = C
        self._svm = None
        self.X_train = None

    def _compute_kernel_matrix(self, X1: np.ndarray,
                                X2: np.ndarray = None) -> np.ndarray:
        """Calcula la matriz de kernel entre X1 y X2."""
        if X2 is None:
            X2 = X1
        n1, n2 = len(X1), len(X2)
        K = np.zeros((n1, n2))
        for i in range(n1):
            for j in range(n2):
                K[i, j] = kernel_cuantico(X1[i], X2[j], self.feature_map)
        return K

    def fit(self, X: np.ndarray, y: np.ndarray):
        """Entrena el QSVM con la matriz de kernel pre-computada."""
        self.X_train = X.copy()
        K_train = self._compute_kernel_matrix(X)
        self._svm = SVC(kernel="precomputed", C=self.C)
        self._svm.fit(K_train, y)
        return self

    def predict(self, X: np.ndarray) -> np.ndarray:
        """Predice clases para nuevos datos."""
        K_test = self._compute_kernel_matrix(X, self.X_train)
        return self._svm.predict(K_test)

    def score(self, X: np.ndarray, y: np.ndarray) -> float:
        return accuracy_score(y, self.predict(X))

print("QuantumKernelSVM implementado.")
```

---

## 3. Dataset y benchmark

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Benchmark en datasets de prueba
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("BENCHMARK: QSVM vs SVM CLÁSICO")
print("="*60)

# Generar dataset de círculos concéntricos (no linealmente separable)
np.random.seed(42)
n_muestras = 80  # pequeño para que el kernel sea computable rápido

X_circles, y_circles = make_circles(n_samples=n_muestras, noise=0.1, factor=0.3)
X_moons, y_moons = make_moons(n_samples=n_muestras, noise=0.1)

# Normalizar a [0, 2π]
scaler_c = MinMaxScaler(feature_range=(0, 2*np.pi))
scaler_m = MinMaxScaler(feature_range=(0, 2*np.pi))

X_c = scaler_c.fit_transform(X_circles)
X_m = scaler_m.fit_transform(X_moons)

# Split train/test
X_c_tr, X_c_te, y_c_tr, y_c_te = train_test_split(X_c, y_circles, test_size=0.25, random_state=42)
X_m_tr, X_m_te, y_m_tr, y_m_te = train_test_split(X_m, y_moons, test_size=0.25, random_state=42)

# Feature map de 2 qubits
fm_2q = ZZFeatureMap(feature_dimension=2, reps=1)

# Función helper para ejecutar todos los SVMs en un dataset
def benchmark_svms(X_tr, X_te, y_tr, y_te, nombre_dataset: str) -> dict:
    """Compara QSVM vs SVM clásico (linear, RBF, poly)."""
    print(f"\n  Dataset: {nombre_dataset}")
    print(f"  Train: {len(X_tr)}, Test: {len(X_te)}")
    resultados = {}

    # SVM clásicos
    kernels_clasicos = {
        "SVM-Linear": SVC(kernel="linear", C=1.0),
        "SVM-RBF": SVC(kernel="rbf", C=1.0, gamma="scale"),
        "SVM-Poly3": SVC(kernel="poly", degree=3, C=1.0),
    }
    for nombre, svm in kernels_clasicos.items():
        svm.fit(X_tr, y_tr)
        acc = accuracy_score(y_te, svm.predict(X_te))
        resultados[nombre] = acc
        print(f"  {nombre:<20}: accuracy = {acc:.4f}")

    # QSVM
    qsvm = QuantumKernelSVM(feature_map=ZZFeatureMap(2, reps=1), C=1.0)
    qsvm.fit(X_tr, y_tr)
    acc_q = qsvm.score(X_te, y_te)
    resultados["QSVM-ZZFeatureMap"] = acc_q
    print(f"  {'QSVM-ZZFeatureMap':<20}: accuracy = {acc_q:.4f}")

    return resultados

res_circles = benchmark_svms(X_c_tr, X_c_te, y_c_tr, y_c_te, "Circles")
res_moons = benchmark_svms(X_m_tr, X_m_te, y_m_tr, y_m_te, "Moons")
```

---

## 4. Frontera de decisión

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Visualización de fronteras
# ─────────────────────────────────────────────

def frontera_decision(ax, clf, X_tr, y_tr, X_te, y_te,
                       titulo: str, es_qsvm: bool = False):
    """Dibuja la frontera de decisión de un clasificador."""
    h = 0.05
    x_min, x_max = X_tr[:, 0].min() - 0.1, X_tr[:, 0].max() + 0.1
    y_min, y_max = X_tr[:, 1].min() - 0.1, X_tr[:, 1].max() + 0.1
    xx, yy = np.meshgrid(np.arange(x_min, x_max, h),
                          np.arange(y_min, y_max, h))

    grid = np.c_[xx.ravel(), yy.ravel()]
    if es_qsvm:
        Z = clf.predict(grid)
    else:
        Z = clf.predict(grid)
    Z = Z.reshape(xx.shape)

    ax.contourf(xx, yy, Z, alpha=0.3, cmap="RdBu")
    ax.scatter(X_tr[y_tr==0, 0], X_tr[y_tr==0, 1], c="blue", s=20, alpha=0.7, marker="o")
    ax.scatter(X_tr[y_tr==1, 0], X_tr[y_tr==1, 1], c="red", s=20, alpha=0.7, marker="^")
    ax.scatter(X_te[y_te==0, 0], X_te[y_te==0, 1], c="blue", s=50, edgecolors="k", marker="o")
    ax.scatter(X_te[y_te==1, 0], X_te[y_te==1, 1], c="red", s=50, edgecolors="k", marker="^")
    acc_test = accuracy_score(y_te, clf.predict(X_te))
    ax.set_title(f"{titulo}\nAcc={acc_test:.3f}")

fig, axes = plt.subplots(2, 4, figsize=(18, 9))

datasets = [
    (X_c_tr, X_c_te, y_c_tr, y_c_te, "Circles"),
    (X_m_tr, X_m_te, y_m_tr, y_m_te, "Moons"),
]

for row, (X_tr_, X_te_, y_tr_, y_te_, dname) in enumerate(datasets):
    # SVM-RBF
    svm_rbf = SVC(kernel="rbf", C=1.0, gamma="scale")
    svm_rbf.fit(X_tr_, y_tr_)
    frontera_decision(axes[row, 0], svm_rbf, X_tr_, y_tr_, X_te_, y_te_,
                     f"{dname}: SVM-RBF")

    # SVM-Linear
    svm_lin = SVC(kernel="linear", C=1.0)
    svm_lin.fit(X_tr_, y_tr_)
    frontera_decision(axes[row, 1], svm_lin, X_tr_, y_tr_, X_te_, y_te_,
                     f"{dname}: SVM-Linear")

    # SVM-Poly
    svm_poly = SVC(kernel="poly", degree=3, C=1.0)
    svm_poly.fit(X_tr_, y_tr_)
    frontera_decision(axes[row, 2], svm_poly, X_tr_, y_tr_, X_te_, y_te_,
                     f"{dname}: SVM-Poly3")

    # QSVM
    qsvm_vis = QuantumKernelSVM(ZZFeatureMap(2, reps=1), C=1.0)
    qsvm_vis.fit(X_tr_, y_tr_)
    frontera_decision(axes[row, 3], qsvm_vis, X_tr_, y_tr_, X_te_, y_te_,
                     f"{dname}: QSVM-ZZ", es_qsvm=True)

plt.suptitle("QSVM vs SVM Clásico — Fronteras de Decisión",
             fontsize=13, fontweight="bold")
plt.tight_layout()
plt.savefig("outputs/m22_qsvm_fronteras.png", dpi=150, bbox_inches="tight")
plt.close()
print("\n→ Guardado: outputs/m22_qsvm_fronteras.png")
```

---

## 5. Ventaja cuántica: ¿real o ilusión?

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Discusión de ventaja cuántica
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("VENTAJA CUÁNTICA EN QSVM — ESTADO DEL ARTE")
print("="*60)

print("""
Resultado teórico (Havlíček 2019):
  Construyeron un dataset sintético donde el ZZFeatureMap
  clasifica perfectamente, pero ningún clasificador clásico
  eficiente puede hacerlo (bajo suposiciones de complejidad).

Realidad práctica:
  • El dataset fue diseñado para el quantum kernel — no es natural
  • Para datos reales, SVM-RBF o redes neurales suelen ganar
  • QSVM con hardware NISQ tiene error → degrada el kernel
  • Evaluación del kernel: O(n²) circuitos para n datos → no escala

Resultado negativo (Kübler 2021):
  Para cualquier ventaja cuántica en kernels, el dataset debe tener
  una "estructura cuántica específica" que el feature map explota.
  Para datos tabulares reales, raramente existe esta estructura.

¿Cuándo podría funcionar QSVM?
  ✓ Datos que tienen estructura cuántica natural (física cuántica, química)
  ✓ Datasets diseñados con la estructura correcta
  ✓ Cuando el hardware mejore y el ruido sea manejable

Conclusión actual (2024-2025):
  QSVM es una demostración de QML, no una herramienta productiva todavía.
  El campo busca activamente problemas donde kernels cuánticos ganen.
""")

# Comparación de costes
print("Comparación de costes computacionales:")
print(f"""
  Dataset: n muestras, d features

  SVM-RBF:
    Entrenamiento: O(n²·d) a O(n³·d)
    Predicción: O(n_sv · d)  (n_sv = support vectors)

  QSVM:
    Evaluación kernel: O(p · n²) circuitos  (p = profundidad fm)
    Con hardware real: incluye tiempo de cola (minutos a horas!)
    Para n=1000: 1M evaluaciones de circuito → inviable hoy

  Aceleración cuántica necesaria para ventaja:
    El circuito cuántico debe ser EXPONENCIALMENTE más rápido de ejecutar
    que calcular el kernel clásico equivalente.
    Esto solo ocurre si el feature map NO tiene simulación clásica eficiente.
""")

# Graficar escalabilidad
fig, ax = plt.subplots(figsize=(9, 5))
n_range = np.array([10, 50, 100, 200, 500, 1000])
costo_rbf = n_range**2       # O(n²) evaluaciones de kernel
costo_qsvm = n_range**2 * 10  # mismo orden pero con overhead de hardware

ax.loglog(n_range, costo_rbf, "b-o", linewidth=2, markersize=8,
         label="SVM-RBF (evaluar kernel clásico)")
ax.loglog(n_range, costo_qsvm, "r-s", linewidth=2, markersize=8,
         label="QSVM (circuitos cuánticos)")

ax.set_xlabel("Número de muestras n")
ax.set_ylabel("Evaluaciones de kernel")
ax.set_title("Escalabilidad: QSVM vs SVM-RBF\n(ambos O(n²), QSVM tiene mayor constante)")
ax.legend(fontsize=9)
ax.grid(True, alpha=0.3, which="both")
ax.text(0.6, 0.3, "Sin ventaja cuántica\nen escalabilidad (2024)",
       transform=ax.transAxes, ha="center", fontsize=10,
       bbox=dict(boxstyle="round", facecolor="wheat", alpha=0.8))

plt.tight_layout()
plt.savefig("outputs/m22_qsvm_escalabilidad.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m22_qsvm_escalabilidad.png")
print("\n✓ Módulo 22 completado")
```

---

## Checkpoint M22

```python
# checkpoint_m22.py
"""Verificaciones del Módulo 22 — QSVM"""
import numpy as np

def test_kernel_cuantico_identidad():
    """K(x, x) = 1."""
    from qiskit.circuit.library import ZZFeatureMap
    from qiskit.quantum_info import Statevector
    fm = ZZFeatureMap(2, reps=1)
    x = np.array([0.5, 1.2])
    fm_bound = fm.assign_parameters(np.tile(x, 1))
    sv = Statevector(fm_bound)
    k = abs(sv.inner(sv))**2
    assert abs(k - 1.0) < 1e-8, f"K(x,x) debe ser 1, got {k}"
    print("✓ test_kernel_cuantico_identidad")

def test_kernel_cuantico_positivo():
    """K(x1, x2) ∈ [0, 1] para cualquier par de puntos."""
    from qiskit.circuit.library import ZZFeatureMap
    from qiskit.quantum_info import Statevector
    fm = ZZFeatureMap(2, reps=1)
    np.random.seed(42)
    for _ in range(10):
        x1 = np.random.uniform(0, 2*np.pi, 2)
        x2 = np.random.uniform(0, 2*np.pi, 2)
        sv1 = Statevector(fm.assign_parameters(x1))
        sv2 = Statevector(fm.assign_parameters(x2))
        k = abs(sv1.inner(sv2))**2
        assert 0 <= k <= 1 + 1e-8, f"Kernel debe estar en [0,1], got {k}"
    print("✓ test_kernel_cuantico_positivo")

def test_qsvm_entrena():
    """QSVM puede ajustarse a datos de entrenamiento."""
    from sklearn.datasets import make_circles
    from sklearn.preprocessing import MinMaxScaler
    from sklearn.svm import SVC
    from qiskit.circuit.library import ZZFeatureMap
    from qiskit.quantum_info import Statevector
    np.random.seed(42)
    X, y = make_circles(n_samples=20, noise=0.1, factor=0.3)
    scaler = MinMaxScaler(feature_range=(0, 2*np.pi))
    X = scaler.fit_transform(X)
    fm = ZZFeatureMap(2, reps=1)

    def kernel(X1, X2):
        K = np.zeros((len(X1), len(X2)))
        for i, x1 in enumerate(X1):
            for j, x2 in enumerate(X2):
                sv1 = Statevector(fm.assign_parameters(x1))
                sv2 = Statevector(fm.assign_parameters(x2))
                K[i, j] = abs(sv1.inner(sv2))**2
        return K

    K_train = kernel(X, X)
    svm = SVC(kernel="precomputed", C=1.0)
    svm.fit(K_train, y)
    acc = svm.score(K_train, y)  # training accuracy
    assert acc > 0.5, f"QSVM accuracy en train debe ser > 0.5, got {acc:.4f}"
    print(f"✓ test_qsvm_entrena (train acc={acc:.4f})")

def test_svm_rbf_circles():
    """SVM-RBF resuelve bien el dataset de círculos."""
    from sklearn.datasets import make_circles
    from sklearn.preprocessing import MinMaxScaler
    from sklearn.svm import SVC
    from sklearn.model_selection import train_test_split
    np.random.seed(42)
    X, y = make_circles(n_samples=100, noise=0.05, factor=0.3)
    X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.25)
    svm = SVC(kernel="rbf", C=1.0, gamma="scale")
    svm.fit(X_tr, y_tr)
    acc = svm.score(X_te, y_te)
    assert acc > 0.85, f"SVM-RBF accuracy en circles debe ser > 0.85, got {acc:.4f}"
    print(f"✓ test_svm_rbf_circles (acc={acc:.4f})")

def test_kernel_matrix_simetrica():
    """La matriz de kernel cuántico es simétrica."""
    from qiskit.circuit.library import ZZFeatureMap
    from qiskit.quantum_info import Statevector
    fm = ZZFeatureMap(2, reps=1)
    np.random.seed(0)
    X = np.random.uniform(0, 2*np.pi, (5, 2))
    K = np.zeros((5, 5))
    for i in range(5):
        for j in range(5):
            sv_i = Statevector(fm.assign_parameters(X[i]))
            sv_j = Statevector(fm.assign_parameters(X[j]))
            K[i, j] = abs(sv_i.inner(sv_j))**2
    assert np.allclose(K, K.T, atol=1e-8), "Matriz de kernel debe ser simétrica"
    print("✓ test_kernel_matrix_simetrica")

if __name__ == "__main__":
    print("Ejecutando checkpoint M22 — QSVM\n")
    test_kernel_cuantico_identidad()
    test_kernel_cuantico_positivo()
    test_qsvm_entrena()
    test_svm_rbf_circles()
    test_kernel_matrix_simetrica()
    print("\n✓ Todos los tests de M22 pasaron")
```
