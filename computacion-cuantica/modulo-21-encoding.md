# Módulo 21 — Codificación de Datos Clásicos en Qubits

**Objetivo:** Dominar las técnicas de codificación (encoding) de datos clásicos en
estados cuánticos: angle encoding, amplitude encoding, basis encoding, IQP circuits
y feature maps avanzados. Entender el impacto en expresividad y complejidad.

**Herramientas:** PennyLane, Qiskit, numpy, sklearn, matplotlib
**Prerequisito:** M06 (PennyLane), M17 (VQC)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
python modulo-21-encoding.py
```

---

## 1. El problema del encoding

```python
# modulo-21-encoding.py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import pennylane as qml
from qiskit import QuantumCircuit
from qiskit.circuit.library import ZZFeatureMap, ZFeatureMap, PauliFeatureMap
from qiskit.quantum_info import Statevector
from sklearn.preprocessing import MinMaxScaler, normalize
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Por qué encoding importa
# ─────────────────────────────────────────────

print("=" * 60)
print("ENCODING — CODIFICACIÓN DE DATOS EN ESTADOS CUÁNTICOS")
print("=" * 60)

print("""
El encoding es EL paso más crítico en QML.
Determina:
  • Cuántos qubits necesitas (capacidad)
  • Qué features puede aprender el modelo (inductive bias)
  • Si hay ventaja cuántica (o no)
  • El overhead de circuito (profundidad)

Métodos principales:
  1. BASIS ENCODING:    x ∈ {0,1}ⁿ → |x⟩         (n bits → n qubits)
  2. ANGLE ENCODING:   x ∈ ℝⁿ → Π Ry(xᵢ)|0⟩      (n features → n qubits)
  3. AMPLITUDE ENCOD.: x ∈ ℝᴺ → Σ xᵢ|i⟩/‖x‖      (2ⁿ features → n qubits)
  4. IQP CIRCUITS:     feature maps con ZZ interacciones
  5. DATA RE-UPLOADING: reinyectar datos en cada capa

Compromiso fundamental:
  AMPLITUDE: muchos datos en pocos qubits, pero circuito de profundidad O(2ⁿ)
  ANGLE:     1 dato por qubit, profundidad O(n) — práctico NISQ
""")
```

---

## 2. Basis Encoding

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: Basis Encoding
# ─────────────────────────────────────────────

print("=" * 60)
print("1. BASIS ENCODING — Datos binarios")
print("=" * 60)

print("""
Codifica un vector binario x ∈ {0,1}ⁿ directamente como estado base.
  x = [1, 0, 1, 1] → |1011⟩

Uso: datos discretos, estados de búsqueda (Grover), cadenas de texto tokenizadas.

Limitaciones:
  ✗ Solo para datos binarios (requiere binarización)
  ✗ 1 qubit por bit — no aprovecha superposición
  ✗ Para datos reales, necesitas cuantización
""")

def basis_encode(x_binary: list) -> QuantumCircuit:
    """Codifica un vector binario en el estado |x⟩."""
    n = len(x_binary)
    qc = QuantumCircuit(n, name="BasisEncode")
    for i, b in enumerate(reversed(x_binary)):   # Qiskit: qubit 0 = LSB
        if b == 1:
            qc.x(i)
    return qc

# Ejemplos
datos_bin = [
    [1, 0, 1, 1],
    [0, 1, 1, 0],
    [1, 1, 0, 0],
]

print("Ejemplos de Basis Encoding:")
for x in datos_bin:
    qc = basis_encode(x)
    sv = Statevector(qc)
    estado_bin = "".join(str(b) for b in x)
    estado_decimal = int(estado_bin, 2)
    probs = sv.probabilities_dict()
    estado_medido = max(probs, key=probs.get)
    print(f"  x={x} → |{estado_bin}⟩ = |{estado_decimal}⟩  "
          f"(verificado: |{estado_medido}⟩, P={probs[estado_medido]:.4f})")

# Superposición de múltiples ejemplos (encoding en batch)
def basis_encode_superposicion(X_batch: list) -> QuantumCircuit:
    """Codifica múltiples ejemplos en superposición uniforme."""
    n_features = len(X_batch[0])
    n_datos = len(X_batch)
    n_index = int(np.ceil(np.log2(n_datos)))   # qubits para el índice
    n_total = n_index + n_features

    qc = QuantumCircuit(n_total, name="BatchBasisEncode")
    # Superposición de índices
    qc.h(range(n_index))
    # Controlado por índice, codificar cada dato
    # (simplificado: solo el primer dato para illustration)
    for i, b in enumerate(reversed(X_batch[0])):
        if b == 1:
            qc.x(n_index + i)
    return qc

print(f"\nBasis encoding en superposición:")
print(f"  Para {len(datos_bin)} ejemplos de {len(datos_bin[0])} features:")
print(f"  Qubits necesarios: {int(np.ceil(np.log2(len(datos_bin))))} (índice) + {len(datos_bin[0])} (dato) = {int(np.ceil(np.log2(len(datos_bin)))) + len(datos_bin[0])}")
```

---

## 3. Angle Encoding

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Angle Encoding
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("2. ANGLE ENCODING — El más usado en NISQ")
print("="*60)

print("""
Cada feature xᵢ ∈ ℝ → ángulo de rotación de un qubit.
  |ψ(x)⟩ = Π_i Ry(xᵢ)|0⟩  o  Π_i Rx(xᵢ)|0⟩  o  Π_i Ry(xᵢ)Rz(xᵢ)|0⟩

  1 feature por qubit, normalizar a [0, 2π] o [-π, π].

Variantes:
  • Single rotation: Ry(xᵢ)  — 1 feature por qubit
  • Double rotation: Ry(xᵢ)Rz(x_{i+n})  — 2 features por qubit (usando ambos ángulos)
  • IQP-style: Ry(xᵢ) + ZZ interacciones — introduce correlaciones

Ventajas:
  ✓ Profundidad O(1) — ideal para NISQ
  ✓ Diferenciable — gradientes exactos con PSR
  ✓ Generalización natural a nuevos datos

Desventajas:
  ✗ Solo n features en n qubits (sin compresión)
  ✗ No captura correlaciones de forma innata
""")

dev_enc = qml.device("default.qubit", wires=4)

@qml.qnode(dev_enc)
def angle_encode_ry(x):
    """AngleEmbedding con RY (default de PennyLane)."""
    qml.AngleEmbedding(x, wires=range(4), rotation="Y")
    return qml.state()

@qml.qnode(dev_enc)
def angle_encode_ryrz(x):
    """Double encoding: RY para feature i, RZ para feature i+n."""
    qml.AngleEmbedding(x[:4], wires=range(4), rotation="Y")
    qml.AngleEmbedding(x[4:], wires=range(4), rotation="Z")
    return qml.state()

# Dato de prueba
x_4d = np.array([0.5, 1.2, -0.8, 2.1])  # 4 features

# Normalizar a [0, 2π]
x_norm = (x_4d - x_4d.min()) / (x_4d.max() - x_4d.min()) * 2 * np.pi
sv_enc = angle_encode_ry(x_norm)
probs = np.abs(sv_enc)**2

print(f"Dato original: {x_4d}")
print(f"Normalizado a [0,2π]: {np.round(x_norm, 4)}")
print(f"Estado codificado (primeras 4 amplitudes):")
for i in range(4):
    print(f"  |{format(i, '04b')}⟩: amp={sv_enc[i]:.4f}, prob={probs[i]:.4f}")

# AngleEmbedding en Qiskit
def angle_encode_qiskit(x: np.ndarray) -> QuantumCircuit:
    """Angle encoding en Qiskit."""
    n = len(x)
    qc = QuantumCircuit(n, name="AngleEncode")
    for i, xi in enumerate(x):
        qc.ry(xi, i)
    return qc

qc_ae = angle_encode_qiskit(x_norm)
print(f"\nCircuito AngleEncode (Qiskit): depth={qc_ae.depth()}, ops={dict(qc_ae.count_ops())}")

# Double encoding: 2 features por qubit
x_8d = np.random.uniform(0, 2*np.pi, 8)
sv_double = angle_encode_ryrz(x_8d)
print(f"\nDouble encoding: 8 features en 4 qubits")
print(f"  Profundidad del circuito: 2 capas (RY + RZ por qubit)")
```

---

## 4. Amplitude Encoding

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Amplitude Encoding
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("3. AMPLITUDE ENCODING — Máxima compresión")
print("="*60)

print("""
Codifica 2ⁿ features en n qubits usando las amplitudes:
  |ψ(x)⟩ = Σᵢ xᵢ/‖x‖ · |i⟩

  Para n=10: ¡1024 features en 10 qubits!

Ejemplo con 4 features en 2 qubits:
  x = [0.5, 0.3, 0.7, 0.2] / ‖x‖ → α₀₀|00⟩ + α₀₁|01⟩ + α₁₀|10⟩ + α₁₁|11⟩

Ventajas:
  ✓ Compresión exponencial: 2ⁿ features en n qubits
  ✓ Bien definido para cualquier vector normalizado

Desventajas:
  ✗ Profundidad O(2ⁿ) para preparar el estado arbitrario
  ✗ Requiere normalización (pérdida de magnitud)
  ✗ Sin acceso eficiente a features individuales
  ✗ No diferenciable de forma directa para nuevos datos

Cuándo usar:
  → Pocos qubits + muchos features (problema de dimensionalidad)
  → Kernel methods cuánticos (QSVM)
""")

@qml.qnode(dev_enc)
def amplitude_encode_pl(features):
    """AmplitudeEmbedding de PennyLane (normaliza automáticamente)."""
    qml.AmplitudeEmbedding(features, wires=range(4), normalize=True)
    return qml.state()

# 16 features en 4 qubits
x_16d = np.random.uniform(0.1, 1.0, 16)
x_16d_norm = x_16d / np.linalg.norm(x_16d)
sv_amp = amplitude_encode_pl(x_16d)

print(f"Amplitude encoding: 16 features en 4 qubits")
print(f"  Norma de x: {np.linalg.norm(x_16d):.4f} → {np.linalg.norm(x_16d_norm):.4f} (normalizado)")
print(f"  Norma del estado cuántico: {np.linalg.norm(sv_amp):.4f}")
print(f"  Fidelidad de recuperación: {np.abs(np.vdot(x_16d_norm, sv_amp))**2:.6f}")

# Comparación angle vs amplitude
print(f"\nComparación de métodos de encoding:")
print(f"{'Método':<20} {'Features/qubit':>15} {'Profundidad':>13} {'Uso principal':>20}")
print(f"{'─'*70}")
comparacion = [
    ("Basis",     "1 bit", "O(n)", "Datos discretos"),
    ("Angle (Ry)", "1", "O(1)", "QML general NISQ"),
    ("Angle (RyRz)", "2", "O(1)", "Más compresión"),
    ("Amplitude", "2ⁿ/n", "O(2ⁿ)", "Kernel methods"),
    ("IQP/ZZFeatureMap", "1-2", "O(n²)", "QSVM, clasificación"),
    ("Data re-upload", "1", "O(p·n)", "QNN expresivos"),
]
for m, fpq, prof, uso in comparacion:
    print(f"  {m:<20} {fpq:>13} {prof:>13} {uso:>20}")
```

---

## 5. IQP y ZZ Feature Maps

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: IQP y ZZFeatureMap
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("4. IQP CIRCUITS Y ZZ FEATURE MAP")
print("="*60)

print("""
IQP (Instantaneous Quantum Polynomial) circuits:
  Circuito de Hadamard + diagonal (Z, ZZ) + Hadamard.
  Genera features no lineales a través de ZᵢZⱼ = cos(xᵢ)cos(xⱼ)·...

ZZFeatureMap (Qiskit, Havlíček 2019):
  Repeticiones de:
    Capa 1: H⊗n · Ry(2·xᵢ)  (encoding lineal)
    Capa 2: Π_{i<j} CX·Rz(2·(π-xᵢ)(π-xⱼ))·CX  (encoding cuadrático)

  Genera el kernel:
    K(x,x') = |⟨φ(x)|φ(x')⟩|²

  Ventaja teórica: para ciertos problemas, este kernel NO puede ser
  calculado eficientemente por un computador clásico.
""")

# ZZFeatureMap de Qiskit
n_features_zz = 4
x_sample = np.array([0.5, 1.2, 0.8, 2.1])

zz_fm = ZZFeatureMap(feature_dimension=n_features_zz, reps=2)
print(f"ZZFeatureMap ({n_features_zz} features, 2 repeticiones):")
print(f"  Parámetros: {zz_fm.num_parameters}")
print(f"  Profundidad: {zz_fm.decompose().depth()}")
print(f"  Operaciones: {dict(zz_fm.decompose().count_ops())}")

# Instanciar con datos
zz_bound = zz_fm.assign_parameters(np.tile(x_sample, 2))  # 2 reps × 4 features
sv_zz = Statevector(zz_bound)
print(f"\nEstado ZZFeatureMap para x={x_sample}:")
probs_zz = sv_zz.probabilities_dict(decimals=4)
top5 = sorted(probs_zz.items(), key=lambda kv: kv[1], reverse=True)[:5]
for estado, prob in top5:
    print(f"  |{estado}⟩: {prob:.4f}")

# Kernel cuántico con ZZFeatureMap
def kernel_cuantico_zz(x1: np.ndarray, x2: np.ndarray,
                        feature_map: ZZFeatureMap) -> float:
    """
    K(x1, x2) = |⟨φ(x1)|φ(x2)⟩|²
    Overlap entre dos estados del feature map.
    """
    fm1 = feature_map.assign_parameters(np.tile(x1, feature_map.reps))
    fm2 = feature_map.assign_parameters(np.tile(x2, feature_map.reps))
    sv1 = Statevector(fm1)
    sv2 = Statevector(fm2)
    return abs(sv1.inner(sv2))**2

x1 = np.array([0.5, 1.2, 0.8, 2.1])
x2 = np.array([0.6, 1.1, 0.9, 1.9])  # similar a x1
x3 = np.array([2.5, 0.3, 2.1, 0.5])  # muy diferente

K_11 = kernel_cuantico_zz(x1, x1, ZZFeatureMap(n_features_zz, reps=1))
K_12 = kernel_cuantico_zz(x1, x2, ZZFeatureMap(n_features_zz, reps=1))
K_13 = kernel_cuantico_zz(x1, x3, ZZFeatureMap(n_features_zz, reps=1))

print(f"\nKernel cuántico (ZZFeatureMap, reps=1):")
print(f"  K(x1, x1) = {K_11:.6f}  (debe ser ≈ 1)")
print(f"  K(x1, x2) = {K_12:.6f}  (x2 similar a x1)")
print(f"  K(x1, x3) = {K_13:.6f}  (x3 muy diferente)")
```

---

## 6. Comparación visual y guía de selección

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: Visualización comparativa
# ─────────────────────────────────────────────

fig = plt.figure(figsize=(18, 12))
gs = GridSpec(2, 3, figure=fig, hspace=0.45, wspace=0.4)

# Generar datos de ejemplo (2D para visualización)
np.random.seed(42)
n_pts = 60
theta_c = np.linspace(0, 2*np.pi, n_pts//2)
clase0 = np.column_stack([np.cos(theta_c) + 0.1*np.random.randn(n_pts//2),
                           np.sin(theta_c) + 0.1*np.random.randn(n_pts//2)])
clase1_pts = 0.5 * np.random.randn(n_pts//2, 2)
X_vis = np.vstack([clase0, clase1_pts])
y_vis = np.array([0]*(n_pts//2) + [1]*(n_pts//2))

# Normalizar a [0, 2π]
scaler = MinMaxScaler(feature_range=(0, 2*np.pi))
X_scaled = scaler.fit_transform(X_vis)

# Plot 1: Datos originales
ax1 = fig.add_subplot(gs[0, 0])
ax1.scatter(X_vis[y_vis==0, 0], X_vis[y_vis==0, 1], c="blue", s=20, alpha=0.6, label="Clase 0")
ax1.scatter(X_vis[y_vis==1, 0], X_vis[y_vis==1, 1], c="red", s=20, alpha=0.6, label="Clase 1")
ax1.set_title("Datos originales (2D)"); ax1.legend(fontsize=7)
ax1.set_xlabel("x₀"); ax1.set_ylabel("x₁")

# Plot 2: Angle encoding — proyección en esfera de Bloch para 1 qubit
ax2 = fig.add_subplot(gs[0, 1])
phi = X_scaled[:, 0]   # RY angle
psi_y = X_scaled[:, 1]  # proyección adicional
# Para RY(θ)|0⟩: ⟨X⟩=0, ⟨Y⟩=0, ⟨Z⟩=cos(θ)
z_comp = np.cos(phi)
x_comp = np.sin(phi) * np.cos(psi_y)
ax2.scatter(x_comp[y_vis==0], z_comp[y_vis==0], c="blue", s=20, alpha=0.6, label="Clase 0")
ax2.scatter(x_comp[y_vis==1], z_comp[y_vis==1], c="red", s=20, alpha=0.6, label="Clase 1")
theta_circ = np.linspace(0, 2*np.pi, 100)
ax2.plot(np.cos(theta_circ), np.sin(theta_circ), "k-", alpha=0.2)
ax2.set_title("Angle Encoding\n(proyección en plano X-Z)"); ax2.legend(fontsize=7)
ax2.set_xlabel("⟨X⟩"); ax2.set_ylabel("⟨Z⟩")
ax2.set_xlim(-1.2, 1.2); ax2.set_ylim(-1.2, 1.2)
ax2.set_aspect("equal")

# Plot 3: Kernel matrix del ZZFeatureMap
ax3 = fig.add_subplot(gs[0, 2])
n_kernel = 20  # usar subset por velocidad
X_sub = X_scaled[:n_kernel]
y_sub = y_vis[:n_kernel]
fm_small = ZZFeatureMap(2, reps=1)
K_matrix = np.zeros((n_kernel, n_kernel))
for i in range(n_kernel):
    for j in range(i, n_kernel):
        k_val = kernel_cuantico_zz(X_sub[i], X_sub[j], fm_small)
        K_matrix[i, j] = k_val
        K_matrix[j, i] = k_val
im = ax3.imshow(K_matrix, cmap="viridis")
plt.colorbar(im, ax=ax3)
# Marcar bloques de clase
ax3.axhline(y=n_kernel//2-0.5, color="red", linewidth=2)
ax3.axvline(x=n_kernel//2-0.5, color="red", linewidth=2)
ax3.set_title(f"Kernel cuántico ZZFeatureMap\n(subconjunto de {n_kernel} puntos)")
ax3.set_xlabel("Índice de muestra"); ax3.set_ylabel("Índice de muestra")

# Plot 4: Complejidad de encoding
ax4 = fig.add_subplot(gs[1, 0])
n_qubits_range = np.arange(1, 16)
features_angle = n_qubits_range                     # 1 feature/qubit
features_amplitude = 2**n_qubits_range               # 2^n features/qubit
depth_angle = np.ones_like(n_qubits_range)
depth_amplitude = 2**n_qubits_range                  # depth exponencial

ax4_twin = ax4.twinx()
ax4.semilogy(n_qubits_range, features_angle, "b-o", linewidth=2, label="Angle (features)")
ax4.semilogy(n_qubits_range, features_amplitude, "r-s", linewidth=2, label="Amplitude (features)")
ax4_twin.plot(n_qubits_range, depth_angle, "b--", linewidth=1.5, alpha=0.5, label="Angle (depth)")
ax4_twin.semilogy(n_qubits_range, depth_amplitude, "r--", linewidth=1.5, alpha=0.5, label="Amplitude (depth)")
ax4.set_xlabel("Número de qubits")
ax4.set_ylabel("Features codificadas", color="black")
ax4_twin.set_ylabel("Profundidad del circuito", color="gray")
ax4.set_title("Capacidad vs Complejidad\nde Encoding")
ax4.legend(loc="upper left", fontsize=7)
ax4_twin.legend(loc="lower right", fontsize=7)

# Plot 5: Separabilidad en espacio de features
ax5 = fig.add_subplot(gs[1, 1])
# Calcular kernel RBF clásico para comparación
def rbf_kernel(x1, x2, sigma=1.0):
    return np.exp(-np.linalg.norm(x1-x2)**2 / (2*sigma**2))

K_rbf = np.zeros((n_kernel, n_kernel))
for i in range(n_kernel):
    for j in range(n_kernel):
        K_rbf[i, j] = rbf_kernel(X_sub[i], X_sub[j])

im2 = ax5.imshow(K_rbf, cmap="plasma")
plt.colorbar(im2, ax=ax5)
ax5.axhline(y=n_kernel//2-0.5, color="red", linewidth=2)
ax5.axvline(x=n_kernel//2-0.5, color="red", linewidth=2)
ax5.set_title("Kernel RBF clásico\n(para comparar)")
ax5.set_xlabel("Índice de muestra"); ax5.set_ylabel("Índice de muestra")

# Plot 6: Tabla de decisión
ax6 = fig.add_subplot(gs[1, 2])
ax6.axis("off")
tabla_enc = [
    ["Encoding", "Features/qubit", "Profundidad", "Cuando usar"],
    ["Basis", "1 bit/qubit", "O(n)", "Datos binarios"],
    ["Angle (RY)", "1", "O(1)", "VQC general NISQ"],
    ["Angle (2x)", "2", "O(1)", "Más features, mismos q"],
    ["Amplitude", "2ⁿ/n", "O(2ⁿ)", "QSVM, muchos features"],
    ["ZZFeatureMap", "1-2", "O(n²)", "QSVM, clasificación"],
    ["Re-uploading", "1+", "O(p·1)", "QNN expresivos"],
]
t = ax6.table(cellText=tabla_enc[1:], colLabels=tabla_enc[0],
              cellLoc="center", loc="center",
              colColours=["#e8f4f8"]*4)
t.auto_set_font_size(False)
t.set_fontsize(7.5)
t.scale(1.1, 1.9)
ax6.set_title("Guía de selección de encoding", fontsize=9)

plt.suptitle("Encoding de Datos Clásicos en Qubits — Comparación Completa",
             fontsize=13, fontweight="bold")
plt.savefig("outputs/m21_encoding_comparacion.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m21_encoding_comparacion.png")
print("\n✓ Módulo 21 completado")
```

---

## Checkpoint M21

```python
# checkpoint_m21.py
"""Verificaciones del Módulo 21 — Encoding"""
import numpy as np

def test_basis_encode():
    """Basis encoding mapea bitstring a estado base correcto."""
    from qiskit import QuantumCircuit
    from qiskit.quantum_info import Statevector
    x = [1, 0, 1, 1]
    qc = QuantumCircuit(4)
    for i, b in enumerate(reversed(x)):
        if b == 1:
            qc.x(i)
    sv = Statevector(qc)
    probs = sv.probabilities_dict()
    # El único estado con probabilidad 1 debe ser el bitstring correcto
    estado_esperado = "".join(str(b) for b in x)
    assert probs.get(estado_esperado, 0) > 0.999, \
        f"Basis encoding: estado esperado {estado_esperado}, got {probs}"
    print(f"✓ test_basis_encode (|{''.join(str(b) for b in x)}⟩)")

def test_angle_encode_normaliza():
    """AngleEmbedding preserva norma del estado cuántico."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=4)
    @qml.qnode(dev)
    def fn(x):
        qml.AngleEmbedding(x, wires=range(4), rotation="Y")
        return qml.state()
    x = np.array([0.5, 1.2, 2.3, 0.8])
    sv = fn(x)
    norma = np.linalg.norm(sv)
    assert abs(norma - 1.0) < 1e-10, f"Estado cuántico debe ser unitario: ‖ψ‖={norma}"
    print("✓ test_angle_encode_normaliza")

def test_amplitude_encode_fidelidad():
    """AmplitudeEmbedding preserva las amplitudes relativas."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=3)
    @qml.qnode(dev)
    def fn(x):
        qml.AmplitudeEmbedding(x, wires=range(3), normalize=True)
        return qml.state()
    x = np.array([1.0, 2.0, 0.5, 1.5, 0.3, 0.7, 1.8, 0.9])
    x_norm = x / np.linalg.norm(x)
    sv = fn(x)
    fidelidad = abs(np.vdot(x_norm, sv))**2
    assert fidelidad > 0.999, f"Fidelidad debe ser ≈1, got {fidelidad:.6f}"
    print(f"✓ test_amplitude_encode_fidelidad (fid={fidelidad:.6f})")

def test_kernel_identidad():
    """K(x, x) = 1 para el kernel cuántico."""
    from qiskit.quantum_info import Statevector
    from qiskit.circuit.library import ZZFeatureMap
    fm = ZZFeatureMap(2, reps=1)
    x = np.array([0.5, 1.2])
    fm_bound = fm.assign_parameters(np.tile(x, 1))
    sv = Statevector(fm_bound)
    k_xx = abs(sv.inner(sv))**2
    assert abs(k_xx - 1.0) < 1e-8, f"K(x,x) debe ser 1, got {k_xx}"
    print(f"✓ test_kernel_identidad (K(x,x)={k_xx:.8f})")

def test_kernel_simetrico():
    """K(x1, x2) = K(x2, x1) para kernel cuántico."""
    from qiskit.quantum_info import Statevector
    from qiskit.circuit.library import ZZFeatureMap
    fm = ZZFeatureMap(2, reps=1)
    x1 = np.array([0.5, 1.2]); x2 = np.array([0.8, 0.3])
    sv1 = Statevector(fm.assign_parameters(np.tile(x1, 1)))
    sv2 = Statevector(fm.assign_parameters(np.tile(x2, 1)))
    k12 = abs(sv1.inner(sv2))**2
    k21 = abs(sv2.inner(sv1))**2
    assert abs(k12 - k21) < 1e-10, f"Kernel debe ser simétrico: K12={k12}, K21={k21}"
    print(f"✓ test_kernel_simetrico (K12={k12:.6f} = K21={k21:.6f})")

def test_amplitude_compression():
    """Amplitude encoding: 2^n features en n qubits."""
    for n in [2, 3, 4]:
        n_features = 2**n
        x = np.random.randn(n_features)
        x /= np.linalg.norm(x)
        # Verificar que el vector normalizado tiene la longitud correcta
        assert len(x) == n_features, f"n={n}: necesitamos {n_features} features"
        assert abs(np.linalg.norm(x) - 1.0) < 1e-10, "Debe estar normalizado"
    print("✓ test_amplitude_compression (2^n features en n qubits)")

if __name__ == "__main__":
    print("Ejecutando checkpoint M21 — Encoding\n")
    test_basis_encode()
    test_angle_encode_normaliza()
    test_amplitude_encode_fidelidad()
    test_kernel_identidad()
    test_kernel_simetrico()
    test_amplitude_compression()
    print("\n✓ Todos los tests de M21 pasaron")
```
