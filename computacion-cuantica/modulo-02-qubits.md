# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 2 — Qubits, Superposición y Entrelazamiento

> **Objetivo:** Implementar qubits, estados de superposición y entrelazamiento usando Qiskit. Visualizar estados cuánticos en la esfera de Bloch, medir estadísticas de colapso y verificar las paradojas del entrelazamiento (Bell states).
>
> **Herramientas:** Qiskit + Qiskit Aer (simulador). NumPy para verificaciones matemáticas.
>
> **Prerequisito:** M01 — Mecánica Cuántica para Programadores (notación Dirac, álgebra lineal compleja, postulados).

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-02-qubits.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-02-qubits.ipynb
```

### Opción C — Google Colab
```python
# Celda 1: instalación
!pip install qiskit qiskit-aer pylatexenc -q

# Celda 2: pegar el código de cada sección
```

### Opción D — IBM Quantum Cloud (hardware real)
```python
from qiskit_ibm_runtime import QiskitRuntimeService
service = QiskitRuntimeService(channel="ibm_quantum", token="TU_TOKEN")
backend = service.least_busy(operational=True, simulator=False)
```
> Los circuitos de este módulo son pequeños (≤ 3 qubits) — ideales para hardware real.

---

## 2.1 El Qubit con Qiskit

### Crear y medir un qubit en estado |0⟩

```python
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import numpy as np
import matplotlib.pyplot as plt

# Simulador de estado vectorial — mantiene la amplitud compleja
sim_sv  = AerSimulator(method='statevector')
# Simulador de disparos — mide y colapsa (como hardware real)
sim_qasm = AerSimulator(method='qasm')

def ejecutar_sv(qc: QuantumCircuit) -> np.ndarray:
    """Devuelve el statevector (amplitudes complejas) del circuito."""
    qc_sv = qc.copy()
    qc_sv.save_statevector()
    job = sim_sv.run(transpile(qc_sv, sim_sv))
    return np.array(job.result().get_statevector())

def ejecutar_shots(qc: QuantumCircuit, shots: int = 4096) -> dict:
    """Devuelve conteos de medición tras 'shots' disparos."""
    qc_m = qc.copy()
    qc_m.measure_all()
    job = sim_qasm.run(transpile(qc_m, sim_qasm), shots=shots)
    return job.result().get_counts()

# --- Qubit en estado |0⟩ (estado por defecto) ---
qc0 = QuantumCircuit(1)
sv0 = ejecutar_sv(qc0)
print(f"Statevector |0⟩: {sv0}")          # [1.+0j  0.+0j]
print(f"P(|0⟩) = {abs(sv0[0])**2:.2f}")  # 1.00
print(f"P(|1⟩) = {abs(sv0[1])**2:.2f}")  # 0.00

# --- Qubit en estado |1⟩ (puerta X = NOT cuántico) ---
qc1 = QuantumCircuit(1)
qc1.x(0)
sv1 = ejecutar_sv(qc1)
print(f"\nStatevector |1⟩: {sv1}")         # [0.+0j  1.+0j]
```

### La esfera de Bloch — visualización geométrica

```python
from qiskit.visualization import plot_bloch_multivector
from qiskit.quantum_info import Statevector

def bloch(sv_array: np.ndarray, titulo: str = ""):
    """Grafica la esfera de Bloch de un statevector."""
    sv = Statevector(sv_array)
    fig = plot_bloch_multivector(sv, title=titulo)
    return fig

# Polo norte (+Z) = |0⟩
sv_norte = np.array([1+0j, 0+0j])
fig1 = bloch(sv_norte, "|0⟩ — polo norte")
fig1.savefig("outputs/bloch_ket0.png", dpi=150, bbox_inches='tight')

# Polo sur (-Z) = |1⟩
sv_sur = np.array([0+0j, 1+0j])
fig2 = bloch(sv_sur, "|1⟩ — polo sur")
fig2.savefig("outputs/bloch_ket1.png", dpi=150, bbox_inches='tight')

# Ecuador (+X) = |+⟩ = (|0⟩ + |1⟩) / √2
sv_mas = np.array([1/np.sqrt(2)+0j, 1/np.sqrt(2)+0j])
fig3 = bloch(sv_mas, "|+⟩ — ecuador +X")
fig3.savefig("outputs/bloch_ket_plus.png", dpi=150, bbox_inches='tight')

print("Esferas de Bloch guardadas en outputs/")
```

---

## 2.2 Superposición — La Puerta Hadamard

### H transforma |0⟩ → |+⟩ y |1⟩ → |−⟩

```python
# Estado |+⟩ = H|0⟩ = (|0⟩ + |1⟩) / √2
qc_H0 = QuantumCircuit(1)
qc_H0.h(0)
sv_H0 = ejecutar_sv(qc_H0)
print(f"H|0⟩ = {sv_H0}")
# [0.707+0j  0.707+0j]

# Verificar normalización
norma = np.sum(np.abs(sv_H0)**2)
print(f"Norma = {norma:.6f}")  # 1.000000

# Estado |−⟩ = H|1⟩ = (|0⟩ - |1⟩) / √2
qc_H1 = QuantumCircuit(1)
qc_H1.x(0)   # preparar |1⟩
qc_H1.h(0)   # aplicar H
sv_H1 = ejecutar_sv(qc_H1)
print(f"\nH|1⟩ = {sv_H1}")
# [0.707+0j  -0.707+0j]  — fase negativa en |1⟩

# Dibujar circuito
print(qc_H0.draw())
```

### Verificar la superposición con disparos de medición

```python
import matplotlib.pyplot as plt

def graficar_histograma(conteos: dict, titulo: str, ax=None):
    estados = sorted(conteos.keys())
    valores = [conteos[e] for e in estados]
    total   = sum(valores)
    probs   = [v / total for v in valores]

    if ax is None:
        fig, ax = plt.subplots(figsize=(5, 3))
    ax.bar(estados, probs, color='steelblue', edgecolor='white')
    ax.set_ylim(0, 1.05)
    ax.set_ylabel("Probabilidad")
    ax.set_title(titulo)
    ax.axhline(0.5, color='red', linestyle='--', linewidth=0.8, alpha=0.7)
    for i, (e, p) in enumerate(zip(estados, probs)):
        ax.text(i, p + 0.02, f"{p:.3f}", ha='center', fontsize=9)
    return ax

fig, axes = plt.subplots(1, 3, figsize=(14, 4))

# |0⟩ → siempre mide 0
conteos_0 = ejecutar_shots(QuantumCircuit(1), shots=4096)
graficar_histograma(conteos_0, "Estado |0⟩", axes[0])

# |1⟩ → siempre mide 1
qc_ket1 = QuantumCircuit(1); qc_ket1.x(0)
conteos_1 = ejecutar_shots(qc_ket1, shots=4096)
graficar_histograma(conteos_1, "Estado |1⟩", axes[1])

# |+⟩ → ~50% 0, ~50% 1
qc_plus = QuantumCircuit(1); qc_plus.h(0)
conteos_H = ejecutar_shots(qc_plus, shots=4096)
graficar_histograma(conteos_H, "Estado |+⟩ = H|0⟩", axes[2])

plt.tight_layout()
plt.savefig("outputs/histogramas_superposicion.png", dpi=150, bbox_inches='tight')
plt.show()
print("Histogramas guardados.")
```

### Fase cuántica — H·H = I (reversibilidad)

```python
# H es su propia inversa: H·H = I
qc_HH = QuantumCircuit(1)
qc_HH.h(0)   # |0⟩ → |+⟩
qc_HH.h(0)   # |+⟩ → |0⟩
sv_HH = ejecutar_sv(qc_HH)
print(f"H·H|0⟩ = {sv_HH}")  # [1.+0j  0.+0j] = |0⟩ ✓

# La fase GLOBAL no es observable — la fase RELATIVA sí
# |−⟩ y |+⟩ tienen la misma distribución de probabilidades
# pero son estados DISTINTOS (ortogonales)
print(f"\n⟨+|−⟩ = {np.dot(sv_H0.conj(), sv_H1):.6f}")  # ≈ 0 — ortogonales
```

---

## 2.3 Puertas de Fase — S, T, Rz

```python
# Las puertas de fase no cambian P(|0⟩) ni P(|1⟩)
# sino la FASE RELATIVA entre amplitudes

# Puerta S = Rz(π/2): añade fase i al estado |1⟩
# S = [[1, 0], [0, i]]
qc_S = QuantumCircuit(1)
qc_S.h(0)   # |+⟩ = (|0⟩ + |1⟩)/√2
qc_S.s(0)   # (|0⟩ + i|1⟩)/√2 = |i+⟩
sv_S = ejecutar_sv(qc_S)
print(f"S|+⟩ = {sv_S}")
# [0.707+0j  0.+0.707j] — fase i en |1⟩

# Puerta T = Rz(π/4): fase e^{iπ/4}
qc_T = QuantumCircuit(1)
qc_T.h(0)
qc_T.t(0)
sv_T = ejecutar_sv(qc_T)
print(f"T|+⟩ = {sv_T}")

# Puerta Rz(θ) — rotación arbitraria alrededor del eje Z
theta = np.pi / 3   # 60 grados
qc_Rz = QuantumCircuit(1)
qc_Rz.h(0)
qc_Rz.rz(theta, 0)
sv_Rz = ejecutar_sv(qc_Rz)
print(f"Rz(60°)|+⟩ = {sv_Rz}")

# Visualizar los 4 estados en Bloch
from qiskit.visualization import plot_bloch_multivector
from qiskit.quantum_info import Statevector

fig, axes = plt.subplots(1, 4, subplot_kw={'projection': '3d'}, figsize=(16, 4))
plt.suptitle("Efecto de puertas de fase en la esfera de Bloch", fontsize=12)

estados = [
    ("|+⟩",  sv_H0),
    ("S|+⟩", sv_S),
    ("T|+⟩", sv_T),
    (f"Rz(60°)|+⟩", sv_Rz),
]
# Nota: plot_bloch_multivector devuelve una figura completa
# Para comparación visual, guardar cada uno por separado
for nombre, sv in estados:
    f = plot_bloch_multivector(Statevector(sv), title=nombre)
    fname = nombre.replace("/", "").replace("°","deg").replace("|","ket_").replace("⟩","")
    f.savefig(f"outputs/bloch_{fname}.png", dpi=120, bbox_inches='tight')
    plt.close(f)
print("Esferas de Bloch guardadas en outputs/")
```

---

## 2.4 Sistemas Multi-Qubit — Producto Tensorial

### Estado de dos qubits

```python
# Con 2 qubits tenemos espacio de 4 dimensiones: |00⟩, |01⟩, |10⟩, |11⟩
# El vector de estado tiene 2^n componentes

# Estado |00⟩ (estado por defecto con 2 qubits)
qc2 = QuantumCircuit(2)
sv_00 = ejecutar_sv(qc2)
print(f"|00⟩ = {sv_00}")
# [1.+0j  0.+0j  0.+0j  0.+0j]

# Convención de Qiskit: qubit 0 es el más a la DERECHA del string
# |01⟩ significa q1=0, q0=1 → índice binario: q1q0 = 01 = 1
qc_01 = QuantumCircuit(2)
qc_01.x(0)   # flip qubit 0
sv_01 = ejecutar_sv(qc_01)
print(f"|01⟩ = {sv_01}")
# [0  1  0  0] — índice 1 = '01' en binario

# |10⟩ — qubit 1 flipado
qc_10 = QuantumCircuit(2)
qc_10.x(1)
sv_10 = ejecutar_sv(qc_10)
print(f"|10⟩ = {sv_10}")
# [0  0  1  0] — índice 2 = '10' en binario

# Producto tensorial manual: |0⟩ ⊗ |+⟩
ket_0 = np.array([1+0j, 0+0j])
ket_plus = np.array([1/np.sqrt(2)+0j, 1/np.sqrt(2)+0j])
producto = np.kron(ket_0, ket_plus)   # ⊗ en numpy = kron
print(f"\n|0⟩ ⊗ |+⟩ = {producto}")
# Qiskit ordena: kron(q[n-1], ..., q[1], q[0])
producto_qiskit = np.kron(ket_plus, ket_0)   # q1=|+⟩, q0=|0⟩... hmm
# Verificar con Qiskit
qc_check = QuantumCircuit(2)
qc_check.h(0)   # q0 = |+⟩, q1 = |0⟩
sv_check = ejecutar_sv(qc_check)
print(f"Qiskit q1=|0⟩, q0=|+⟩ = {sv_check}")
```

> **Convención de Qiskit:** el qubit con índice 0 está a la **derecha** del string de bits. `|01⟩` significa q₁=0, q₀=1. El statevector indexa los estados como `|q_{n-1}...q_1 q_0⟩`.

---

## 2.5 Entrelazamiento — Estados de Bell

### Los 4 estados de Bell

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Estado |Φ+⟩ = (|00⟩ + |11⟩) / √2  ← el más famoso
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
def estado_bell(i: int, j: int) -> QuantumCircuit:
    """
    Prepara el estado de Bell según (i, j):
      (0,0) → |Φ+⟩ = (|00⟩ + |11⟩)/√2
      (0,1) → |Φ-⟩ = (|00⟩ - |11⟩)/√2
      (1,0) → |Ψ+⟩ = (|01⟩ + |10⟩)/√2
      (1,1) → |Ψ-⟩ = (|01⟩ - |10⟩)/√2
    """
    qc = QuantumCircuit(2, name=f"Bell({i},{j})")
    if i == 1:
        qc.x(0)
    qc.h(0)        # superposición en q0
    qc.cx(0, 1)    # CNOT: entrelaza q0 (control) con q1 (target)
    if j == 1:
        qc.z(0)
    return qc

# Crear y verificar los 4 estados de Bell
bell_nombres = ["Φ+", "Φ-", "Ψ+", "Ψ-"]
bell_params   = [(0,0), (0,1), (1,0), (1,1)]
bell_teoricos = [
    np.array([1/np.sqrt(2), 0, 0, 1/np.sqrt(2)]),   # |Φ+⟩
    np.array([1/np.sqrt(2), 0, 0, -1/np.sqrt(2)]),  # |Φ-⟩
    np.array([0, 1/np.sqrt(2), 1/np.sqrt(2), 0]),   # |Ψ+⟩
    np.array([0, 1/np.sqrt(2), -1/np.sqrt(2), 0]),  # |Ψ-⟩
]

print("Estado de Bell | Statevector (Qiskit) | Fidelidad con teórico")
print("-" * 65)
for nombre, (i, j), sv_teo in zip(bell_nombres, bell_params, bell_teoricos):
    qc_b = estado_bell(i, j)
    sv_b = ejecutar_sv(qc_b)
    fidelidad = abs(np.dot(sv_teo.conj(), sv_b))**2
    print(f"|{nombre}⟩ ({i},{j}) | {np.round(sv_b, 3)} | F={fidelidad:.6f}")

# Visualizar circuito del estado Φ+
qc_phi_plus = estado_bell(0, 0)
print("\nCircuito para |Φ+⟩:")
print(qc_phi_plus.draw())
```

### Verificar el entrelazamiento — correlaciones perfectas

```python
# En |Φ+⟩ = (|00⟩ + |11⟩)/√2:
# Cuando medimos q0=0 → q1=0 SIEMPRE
# Cuando medimos q0=1 → q1=1 SIEMPRE
# No existe correlación para |+⟩ ⊗ |+⟩ (estado separable)

SHOTS = 8192

# Estado separable: q0=|+⟩, q1=|+⟩ — probabilidades independientes
qc_sep = QuantumCircuit(2, 2)
qc_sep.h(0)
qc_sep.h(1)
qc_sep.measure([0, 1], [0, 1])
conteos_sep = sim_qasm.run(transpile(qc_sep, sim_qasm), shots=SHOTS).result().get_counts()

# Estado entrelazado: |Φ+⟩
qc_ent = estado_bell(0, 0)
qc_ent.measure_all()
conteos_ent = sim_qasm.run(transpile(qc_ent, sim_qasm), shots=SHOTS).result().get_counts()

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
graficar_histograma(conteos_sep, "Separable |+⟩⊗|+⟩\n(4 resultados equiprobables)", axes[0])
graficar_histograma(conteos_ent, "Entrelazado |Φ+⟩\n(solo 00 y 11)", axes[1])
plt.tight_layout()
plt.savefig("outputs/correlaciones_bell.png", dpi=150, bbox_inches='tight')
plt.show()

# Calcular correlación de resultados
def correlacion_bits(conteos: dict, shots: int) -> float:
    """Calcula E[q0·q1] - E[q0]·E[q1] (covarianza normalizada)."""
    suma_00, suma_01, suma_10, suma_11 = 0, 0, 0, 0
    for bits, cnt in conteos.items():
        # Qiskit devuelve el string en orden q_{n-1}...q_0
        q0 = int(bits[-1])  # último caracter = qubit 0
        q1 = int(bits[-2]) if len(bits) >= 2 else 0
        if q0 == 0 and q1 == 0: suma_00 += cnt
        if q0 == 1 and q1 == 0: suma_10 += cnt
        if q0 == 0 and q1 == 1: suma_01 += cnt
        if q0 == 1 and q1 == 1: suma_11 += cnt
    p00 = suma_00/shots; p01 = suma_01/shots
    p10 = suma_10/shots; p11 = suma_11/shots
    E_q0 = p10 + p11;   E_q1 = p01 + p11
    E_q0q1 = p11
    return E_q0q1 - E_q0 * E_q1

cov_sep = correlacion_bits(conteos_sep, SHOTS)
cov_ent = correlacion_bits(conteos_ent, SHOTS)
print(f"\nCovarianza estado separable:    {cov_sep:.4f}  (≈ 0)")
print(f"Covarianza estado entrelazado:  {cov_ent:.4f}  (≠ 0 → correlación cuántica)")
```

---

## 2.6 Desigualdades de Bell — Prueba Experimental

### El test CHSH — violación del límite clásico de 2

```python
# El parámetro CHSH mide correlaciones en 4 pares de ángulos.
# Clásicamente: |S_CHSH| ≤ 2
# Con entrelazamiento cuántico: S_CHSH = 2√2 ≈ 2.828

def medir_correlacion_chsh(theta_a: float, theta_b: float, shots: int = 4096) -> float:
    """
    Mide E(a, b) = P(mismo resultado) - P(diferente resultado)
    para dos detectores en ángulos theta_a y theta_b.
    Usa el estado |Φ+⟩.
    """
    qc = QuantumCircuit(2, 2)
    # Preparar |Φ+⟩
    qc.h(0)
    qc.cx(0, 1)
    # Rotar cada detector al ángulo de medición
    qc.ry(-2 * theta_a, 0)
    qc.ry(-2 * theta_b, 1)
    qc.measure([0, 1], [0, 1])

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=shots).result().get_counts()

    mismo = 0; diferente = 0
    for bits, cnt in conteos.items():
        q0 = int(bits[-1]); q1 = int(bits[-2]) if len(bits) >= 2 else 0
        if q0 == q1:
            mismo += cnt
        else:
            diferente += cnt

    total = mismo + diferente
    return (mismo - diferente) / total  # E(a,b) ∈ [-1, +1]

# Ángulos CHSH óptimos para maximizar la violación
a0 = 0;         a1 = np.pi/2       # detectores de Alice
b0 = np.pi/4;   b1 = -np.pi/4     # detectores de Bob

E_a0b0 = medir_correlacion_chsh(a0, b0)
E_a0b1 = medir_correlacion_chsh(a0, b1)
E_a1b0 = medir_correlacion_chsh(a1, b0)
E_a1b1 = medir_correlacion_chsh(a1, b1)

S_CHSH = abs(E_a0b0 - E_a0b1 + E_a1b0 + E_a1b1)
print(f"E(a0,b0) = {E_a0b0:.4f}")
print(f"E(a0,b1) = {E_a0b1:.4f}")
print(f"E(a1,b0) = {E_a1b0:.4f}")
print(f"E(a1,b1) = {E_a1b1:.4f}")
print(f"\n|S_CHSH| = {S_CHSH:.4f}")
print(f"Límite clásico:  |S| ≤ 2.0000")
print(f"Límite cuántico: |S| ≤ 2√2 = {2*np.sqrt(2):.4f}")
print(f"Violación detectada: {'SÍ ✓' if S_CHSH > 2 else 'NO'}")
```

---

## 2.7 Teletransportación Cuántica

```python
# Protocolo de teletransportación: Alice envía |ψ⟩ a Bob
# usando un par de Bell compartido + 2 bits clásicos

def teleportacion(alpha: complex, beta: complex) -> dict:
    """
    Teletransporta el estado |ψ⟩ = α|0⟩ + β|1⟩ de Alice a Bob.

    Retorna los conteos de medición del qubit de Bob.
    Requisito: |alpha|² + |beta|² = 1
    """
    assert abs(abs(alpha)**2 + abs(beta)**2 - 1) < 1e-6, "Estado no normalizado"

    qc = QuantumCircuit(3, 2)
    # Qubits: 0 = estado de Alice (|ψ⟩), 1 = qubit Alice del par Bell, 2 = qubit Bob del par Bell

    # Paso 1: Preparar el estado que Alice quiere teletransportar en q0
    # En Qiskit usamos initialize() para estados arbitrarios
    qc.initialize([alpha, beta], 0)

    # Paso 2: Preparar par de Bell entre q1 y q2
    qc.h(1)
    qc.cx(1, 2)

    # Paso 3: Alice aplica operaciones locales
    qc.cx(0, 1)   # CNOT entre el estado a enviar y su qubit del par
    qc.h(0)       # Hadamard en el estado original

    # Paso 4: Alice mide sus 2 qubits (q0 y q1) — colapsa el estado
    qc.measure(0, 0)   # resultado Alice qubit 0 → bit clásico 0
    qc.measure(1, 1)   # resultado Alice qubit 1 → bit clásico 1

    # Paso 5: Bob aplica correcciones condicionales en q2 según bits clásicos
    # Si bit_1 = 1 → Bob aplica X
    # Si bit_0 = 1 → Bob aplica Z
    with qc.if_else((qc.clbits[1], 1)):    # if bit1 == 1:
        qc.x(2)
    with qc.if_else((qc.clbits[0], 1)):    # if bit0 == 1:
        qc.z(2)

    # Paso 6: Bob verifica midiendo su qubit
    qc.save_statevector()

    return qc

# --- Ejemplo 1: Teletransportar |+⟩ = (|0⟩ + |1⟩)/√2 ---
alpha_T = 1/np.sqrt(2)
beta_T  = 1/np.sqrt(2)

qc_tele = teleportacion(alpha_T, beta_T)
print("Circuito de teletransportación:")
print(qc_tele.draw(fold=90))

# Ejecutar con simulador que soporta if_else
from qiskit_aer import AerSimulator
sim_tele = AerSimulator(method='statevector')
job_tele = sim_tele.run(transpile(qc_tele, sim_tele), shots=4096)
result_tele = job_tele.result()

# Verificar usando fidelidad del estado final
sv_final = np.array(result_tele.get_statevector())
# El estado de Bob (q2) debería ser |+⟩
sv_esperado_bob = np.array([alpha_T, beta_T])
print(f"\nEstado inicial a teletransportar: α={alpha_T:.3f}, β={beta_T:.3f}")
print("(El estado de Bob después del protocolo debería ser idéntico)")
```

---

## 2.8 Superdense Coding — 2 bits con 1 qubit

```python
# Protocolo inverso: Bob envía 2 bits clásicos a Alice con solo 1 qubit cuántico

def superdense_coding(bit0: int, bit1: int) -> dict:
    """
    Bob codifica (bit0, bit1) en un qubit compartido con Alice.
    Alice los decodifica midiendo el par de Bell.
    """
    assert bit0 in (0,1) and bit1 in (0,1), "Solo bits 0 o 1"

    qc = QuantumCircuit(2, 2)

    # Paso 1: Alice prepara el par de Bell (estado compartido previo)
    qc.h(0)
    qc.cx(0, 1)

    qc.barrier()

    # Paso 2: Bob codifica sus 2 bits SOLO en q1 (su qubit)
    # 00 → I (no hacer nada)
    # 01 → X
    # 10 → Z
    # 11 → iY = XZ
    if bit0 == 1:
        qc.x(1)
    if bit1 == 1:
        qc.z(1)

    qc.barrier()

    # Paso 3: Alice decodifica aplicando CNOT + H
    qc.cx(0, 1)
    qc.h(0)

    # Paso 4: Alice mide y recupera los bits originales de Bob
    qc.measure([0, 1], [0, 1])

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=1024).result().get_counts()
    return conteos, qc

# Probar todos los casos
print("Superdense Coding — Verificación")
print("─" * 40)
for b0, b1 in [(0,0), (0,1), (1,0), (1,1)]:
    conteos_sd, qc_sd = superdense_coding(b0, b1)
    # El único resultado debe ser '|b1 b0⟩' en la medición
    resultado = max(conteos_sd, key=conteos_sd.get)
    # Qiskit: mide q0 en bit 0 (derecha), q1 en bit 1
    bit_q0 = int(resultado[-1])   # bit clásico 0 (Alice mide q0)
    bit_q1 = int(resultado[-2]) if len(resultado) >= 2 else 0
    exito = (bit_q0 == b0) and (bit_q1 == b1)
    print(f"Bob envía ({b0},{b1}) → Alice recibe q0={bit_q0}, q1={bit_q1} {'✓' if exito else '✗'}")
    print(f"  Conteos: {conteos_sd}")

print(qc_sd.draw())
```

---

## 2.9 Estados GHZ y W — Entrelazamiento Multipartito

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Estado GHZ — (|000⟩ + |111⟩) / √2
# Máximo entrelazamiento tripartito
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
def estado_GHZ(n: int = 3) -> QuantumCircuit:
    qc = QuantumCircuit(n, name=f"GHZ-{n}")
    qc.h(0)
    for i in range(1, n):
        qc.cx(0, i)
    return qc

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Estado W — (|001⟩ + |010⟩ + |100⟩) / √3
# Entrelazamiento más "robusto" frente a pérdida de un qubit
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
def estado_W_3() -> QuantumCircuit:
    """Prepara el estado W de 3 qubits."""
    qc = QuantumCircuit(3, name="W-3")
    # Ry(2*arccos(√(2/3))) en q0 → amplitud √(1/3) en |1⟩, √(2/3) en |0⟩
    theta_0 = 2 * np.arccos(np.sqrt(2/3))
    qc.ry(theta_0, 0)
    # Controlado en q0=|1⟩: preparar (|01⟩ + |10⟩)/√2 en q1,q2
    qc.ch(0, 1)   # Hadamard controlado
    qc.cx(1, 2)   # CNOT q1→q2
    qc.x(1)       # flip para obtener el patrón correcto
    qc.cx(0, 1)
    return qc

# Verificar GHZ-3
qc_ghz = estado_GHZ(3)
sv_ghz = ejecutar_sv(qc_ghz)
print("Estado GHZ-3:")
print(f"  Statevector: {np.round(sv_ghz, 4)}")
print(f"  |000⟩: {abs(sv_ghz[0])**2:.3f}  |111⟩: {abs(sv_ghz[7])**2:.3f}")
print(f"  Solo hay amplitud en |000⟩ y |111⟩: {np.allclose(sv_ghz[[1,2,3,4,5,6]], 0)}")

# GHZ generalizado para n qubits
for n in [3, 4, 5, 8]:
    qc_n = estado_GHZ(n)
    sv_n = ejecutar_sv(qc_n)
    # En GHZ-n, solo los índices 0 y 2^n - 1 tienen amplitud 1/√2
    p_0   = abs(sv_n[0])**2
    p_all = abs(sv_n[-1])**2
    print(f"GHZ-{n}: P(|{'0'*n}⟩) = {p_0:.3f}, P(|{'1'*n}⟩) = {p_all:.3f}")

# Medición del estado GHZ — correlaciones perfectas entre los 3 qubits
qc_ghz_m = estado_GHZ(3)
qc_ghz_m.measure_all()
conteos_ghz = sim_qasm.run(transpile(qc_ghz_m, sim_qasm), shots=4096).result().get_counts()
print(f"\nConteos GHZ-3: {conteos_ghz}")
# Solo debe aparecer '000' y '111'
```

---

## 2.10 Entropía de Entrelazamiento — Cuantificando Correlaciones

```python
from qiskit.quantum_info import Statevector, partial_trace, entropy

def entropia_entrelazamiento(qc: QuantumCircuit, qubits_subsistema: list) -> float:
    """
    Calcula la entropía de von Neumann del subsistema dado.
    Valor 0  → estado separable (sin entrelazamiento).
    Valor ln(d) → máximo entrelazamiento (d = dimensión subsistema).
    """
    sv = Statevector(ejecutar_sv(qc))
    n_total = qc.num_qubits
    # Los qubits complementarios al subsistema
    qubits_traza = [i for i in range(n_total) if i not in qubits_subsistema]
    # Traza parcial sobre los qubits complementarios
    rho_sub = partial_trace(sv, qubits_traza)
    return entropy(rho_sub, base=2)  # entropía en bits

# Comparar estados
print("Entropía de entrelazamiento (qubit 0 vs resto):")
print("─" * 50)

# Estado separable |+⟩⊗|+⟩
qc_prod = QuantumCircuit(2)
qc_prod.h(0); qc_prod.h(1)
S_prod = entropia_entrelazamiento(qc_prod, [0])
print(f"|+⟩⊗|+⟩  (separable):   S = {S_prod:.4f} bits (debe ser 0)")

# Estado de Bell |Φ+⟩
qc_bell = estado_bell(0, 0)
S_bell = entropia_entrelazamiento(qc_bell, [0])
print(f"|Φ+⟩  (Bell 2 qubits):   S = {S_bell:.4f} bits (máx = 1)")

# Estado GHZ-3
qc_ghz3 = estado_GHZ(3)
S_ghz3_q0 = entropia_entrelazamiento(qc_ghz3, [0])
print(f"|GHZ⟩ (subsistema q0):  S = {S_ghz3_q0:.4f} bits (máx = 1)")

# Estado producto de 3 qubits sin entrelazar
qc_3sep = QuantumCircuit(3)
qc_3sep.h(0); qc_3sep.h(1); qc_3sep.h(2)
S_3sep = entropia_entrelazamiento(qc_3sep, [0])
print(f"|+⟩⊗|+⟩⊗|+⟩ (separ):  S = {S_3sep:.4f} bits (debe ser 0)")

# Visualizar entropía para diferentes ángulos de preparación
thetas = np.linspace(0, np.pi/2, 50)
S_vals = []
for t in thetas:
    qc_t = QuantumCircuit(2)
    qc_t.ry(2*t, 0)
    qc_t.cx(0, 1)
    S_vals.append(entropia_entrelazamiento(qc_t, [0]))

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(thetas * 180 / np.pi, S_vals, color='steelblue', linewidth=2)
ax.set_xlabel("Ángulo θ en Ry(2θ) (grados)")
ax.set_ylabel("Entropía de entrelazamiento (bits)")
ax.set_title("Entropía de entrelazamiento de Ry(2θ)·CNOT|00⟩")
ax.axhline(1.0, color='red', linestyle='--', linewidth=0.8, label="Máximo (1 bit)")
ax.axvline(45, color='green', linestyle='--', linewidth=0.8, label="θ=45° → Bell state")
ax.legend()
plt.tight_layout()
plt.savefig("outputs/entropia_entrelazamiento.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## Resumen de conceptos

| Concepto | Descripción | Puerta Qiskit |
|---|---|---|
| Superposición | α\|0⟩ + β\|1⟩, las amplitudes son complejas | `H`, `Ry(θ)`, `Rx(θ)` |
| Estado base | \|0⟩ o \|1⟩, resultado determinista | Ninguna (estado inicial) o `X` |
| Fase cuántica | Fase relativa entre amplitudes — observable | `S`, `T`, `Rz(θ)`, `P(λ)` |
| Colapso | Al medir, el estado elige aleatoriamente | `measure()` |
| Estado de Bell | Máximo entrelazamiento bipartito | `H` + `CX` |
| Estado GHZ | Entrelazamiento multipartito | `H` + n×`CX` |
| CHSH | Prueba experimental del entrelazamiento | Rotaciones + medición |
| Teletransportación | Transmisión del estado usando Bell + 2 bits | `H`, `CX`, correcciones condicionales |
| Superdense Coding | 2 bits clásicos con 1 qubit cuántico | Codificación en Bell pair |
| Entropía von Neumann | Cuantifica el entrelazamiento | `partial_trace` + `entropy` |

---

## checkpoint_m02.py

```python
"""
Checkpoint Módulo 2 — Qubits, Superposición y Entrelazamiento
Ejecutar con: python checkpoint_m02.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.quantum_info import Statevector, partial_trace, entropy
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

def shots(qc, n=4096):
    qc2 = qc.copy(); qc2.measure_all()
    return sim_qasm.run(transpile(qc2, sim_qasm), shots=n).result().get_counts()

print("=" * 55)
print("Checkpoint M02 — Qubits y Entrelazamiento")
print("=" * 55)

# ── Test 1: Estado |0⟩ por defecto ──────────────────────
qc0 = QuantumCircuit(1)
sv0 = sv(qc0)
assert np.allclose(sv0, [1+0j, 0+0j], atol=1e-6), "FALLO: |0⟩ incorrecto"
assert abs(sv0[0])**2 > 0.999, "FALLO: P(|0⟩) debe ser 1"
print("✓ Test 1: Estado |0⟩ correcto")

# ── Test 2: Puerta X → |1⟩ ──────────────────────────────
qc1 = QuantumCircuit(1); qc1.x(0)
sv1 = sv(qc1)
assert np.allclose(sv1, [0+0j, 1+0j], atol=1e-6), "FALLO: X|0⟩ incorrecto"
print("✓ Test 2: Puerta X → |1⟩ correcto")

# ── Test 3: Hadamard → superposición igual ───────────────
qcH = QuantumCircuit(1); qcH.h(0)
svH = sv(qcH)
assert abs(abs(svH[0])**2 - 0.5) < 0.01, "FALLO: P(0) debe ser 0.5 en |+⟩"
assert abs(abs(svH[1])**2 - 0.5) < 0.01, "FALLO: P(1) debe ser 0.5 en |+⟩"
assert abs(np.sum(np.abs(svH)**2) - 1.0) < 1e-6, "FALLO: Estado no normalizado"
print("✓ Test 3: Hadamard → superposición 50/50")

# ── Test 4: H·H = I ─────────────────────────────────────
qcHH = QuantumCircuit(1); qcHH.h(0); qcHH.h(0)
svHH = sv(qcHH)
assert np.allclose(svHH, [1+0j, 0+0j], atol=1e-6), "FALLO: H·H debe devolver |0⟩"
print("✓ Test 4: H·H = Identidad")

# ── Test 5: Estado de Bell |Φ+⟩ ─────────────────────────
qcB = QuantumCircuit(2); qcB.h(0); qcB.cx(0, 1)
svB = sv(qcB)
esperado_bell = np.array([1/np.sqrt(2), 0, 0, 1/np.sqrt(2)])
assert np.allclose(np.abs(svB), np.abs(esperado_bell), atol=1e-6), "FALLO: Bell state incorrecto"
print("✓ Test 5: Estado de Bell |Φ+⟩ correcto")

# ── Test 6: Correlaciones Bell — solo 00 y 11 ───────────
c_bell = shots(qcB)
total = sum(c_bell.values())
for bits, cnt in c_bell.items():
    assert bits in ('00', '11'), f"FALLO: resultado {bits} no esperado en Bell state"
p00 = c_bell.get('00', 0) / total
p11 = c_bell.get('11', 0) / total
assert 0.45 < p00 < 0.55, f"FALLO: P(00) = {p00:.3f}, esperado ~0.5"
assert 0.45 < p11 < 0.55, f"FALLO: P(11) = {p11:.3f}, esperado ~0.5"
print(f"✓ Test 6: Correlaciones Bell: P(00)={p00:.3f}, P(11)={p11:.3f}")

# ── Test 7: Estado GHZ-3 ─────────────────────────────────
qcG = QuantumCircuit(3); qcG.h(0); qcG.cx(0,1); qcG.cx(0,2)
svG = sv(qcG)
assert abs(abs(svG[0])**2 - 0.5) < 0.01, "FALLO: P(000) debe ser 0.5 en GHZ"
assert abs(abs(svG[7])**2 - 0.5) < 0.01, "FALLO: P(111) debe ser 0.5 en GHZ"
assert np.allclose(svG[[1,2,3,4,5,6]], 0, atol=1e-6), "FALLO: GHZ tiene amplitudes incorrectas"
print("✓ Test 7: Estado GHZ-3 correcto")

# ── Test 8: Entropía de entrelazamiento ─────────────────
sv_bell_obj = Statevector(svB)
rho_q0 = partial_trace(sv_bell_obj, [1])   # traza sobre q1
S_ent = entropy(rho_q0, base=2)
assert abs(S_ent - 1.0) < 0.01, f"FALLO: Entropía Bell = {S_ent:.4f}, esperado 1.0"
print(f"✓ Test 8: Entropía de entrelazamiento Bell = {S_ent:.4f} bits (máximo)")

# ── Test 9: Estado separable → entropía = 0 ─────────────
qcSep = QuantumCircuit(2); qcSep.h(0); qcSep.h(1)
svSep_obj = Statevector(sv(qcSep))
rho_sep = partial_trace(svSep_obj, [1])
S_sep = entropy(rho_sep, base=2)
assert abs(S_sep) < 0.01, f"FALLO: Estado separable debe tener S=0, got {S_sep:.4f}"
print(f"✓ Test 9: Entropía estado separable = {S_sep:.4f} bits (mínimo)")

# ── Test 10: Superdense Coding ───────────────────────────
def superdense_check(b0, b1):
    qc = QuantumCircuit(2, 2)
    qc.h(0); qc.cx(0, 1)
    if b0 == 1: qc.x(1)
    if b1 == 1: qc.z(1)
    qc.cx(0, 1); qc.h(0)
    qc.measure([0,1],[0,1])
    c = sim_qasm.run(transpile(qc, sim_qasm), shots=512).result().get_counts()
    resultado = max(c, key=c.get)
    return int(resultado[-1]) == b0 and int(resultado[-2] if len(resultado)>1 else 0) == b1

for b0, b1 in [(0,0),(0,1),(1,0),(1,1)]:
    assert superdense_check(b0, b1), f"FALLO: Superdense coding falló para ({b0},{b1})"
print("✓ Test 10: Superdense Coding (todos los casos correctos)")

print("\n" + "=" * 55)
print("✅ Todos los tests del M02 pasaron correctamente")
print("=" * 55)
```
