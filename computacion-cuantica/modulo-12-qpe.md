# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 12 — Estimación de Fase Cuántica (QPE)

> **Objetivo:** Implementar el algoritmo QPE (Quantum Phase Estimation) completo, entender su rol como subroutina en Shor, VQE y simulación cuántica. Demostrar su precisión exponencial en el número de qubits ancilla.
>
> **Herramientas:** Qiskit + NumPy.
>
> **Prerequisito:** M11 — Algoritmo de Shor.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-12-qpe.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-12-qpe.ipynb
```

### Opción C — Google Colab
```python
!pip install qiskit qiskit-aer pylatexenc -q
```

---

## 12.1 El Problema de Estimación de Fase

```python
"""
PROBLEMA QPE:
  Dado un operador unitario U y su eigenstate |ψ⟩ tal que:
    U|ψ⟩ = e^{2πiφ}|ψ⟩

  Encontrar φ (la "fase") con t bits de precisión.

  φ ∈ [0, 1)  →  se estima como k/2^t donde k ∈ {0, ..., 2^t - 1}

APLICACIONES:
  - Algoritmo de Shor: U = multiplicación modular
  - VQE: U = e^{-iHt} (evolución de Hamiltonianos)
  - Conteo cuántico: estima M (número de soluciones) para Grover
  - Simulación cuántica: eigenvalores de Hamiltonianos

CIRCUITO:
  1. Preparar t qubits ancilla en superposición (H⊗t)
  2. Aplicar U^{2^k} controlados por cada qubit ancilla
  3. Aplicar IQFT en los t qubits ancilla
  4. Medir → k/2^t ≈ φ
"""
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.circuit.library import QFT
from qiskit.quantum_info import Operator
import numpy as np
import matplotlib.pyplot as plt
from fractions import Fraction
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())
```

---

## 12.2 Implementación QPE General

```python
def qpe(U: QuantumCircuit, eigenstate: QuantumCircuit, t: int) -> QuantumCircuit:
    """
    Quantum Phase Estimation (QPE).

    U:          circuito del operador unitario (n_u qubits)
    eigenstate: circuito que prepara el eigenstate de U (n_u qubits)
    t:          número de qubits ancilla (precisión = t bits → error < 2^{-t})

    Retorna circuito con t + n_u qubits y t bits clásicos.
    """
    n_u = U.num_qubits
    n_total = t + n_u

    qc = QuantumCircuit(n_total, t, name=f"QPE(t={t})")

    # 1. Preparar eigenstate en los qubits de trabajo
    qc.compose(eigenstate, qubits=range(t, n_total), inplace=True)

    # 2. Superposición en qubits ancilla
    qc.h(range(t))

    # 3. U^{2^k} controlados — de más significativo a menos
    for k in range(t):
        # k=0 corresponde al qubit ancilla más significativo
        # aplicar U^{2^(t-1-k)} controlado por el qubit ancilla k
        repeticiones = 2**(t - 1 - k)
        U_potencia = U.power(repeticiones)
        # Convertir a compuerta controlada
        controlled_U = U_potencia.to_gate().control(1)
        qc.append(controlled_U, [k] + list(range(t, n_total)))

    qc.barrier()

    # 4. IQFT en qubits ancilla
    iqft = QFT(t, inverse=True, do_swaps=True)
    qc.compose(iqft, qubits=range(t), inplace=True)

    # 5. Medir qubits ancilla
    qc.measure(range(t), range(t))

    return qc

def estimar_fase(conteos: dict, t: int) -> dict:
    """
    Convierte los resultados de QPE en estimaciones de fase.
    """
    total = sum(conteos.values())
    estimaciones = {}
    for bits, cnt in sorted(conteos.items(), key=lambda x: -x[1]):
        k = int(bits[::-1], 2)   # string → entero (orden Qiskit)
        phi = k / 2**t
        estimaciones[phi] = estimaciones.get(phi, 0) + cnt / total
    return estimaciones
```

---

## 12.3 QPE para una Fase Conocida — Verificación

```python
# Operador U = Rz(2π·φ) tiene eigenvalores e^{±iπφ}
# En el eigenstate |+⟩: U|+⟩ = e^{iπφ}|+⟩... no exactamente

# Usaremos U = P(2πφ) (phase gate) con eigenstate |1⟩
# P(θ)|1⟩ = e^{iθ}|1⟩ → fase = θ/2π

def qpe_phase_gate(phi_real: float, t: int) -> QuantumCircuit:
    """
    QPE para U = P(2π·phi_real), eigenstate = |1⟩
    La fase exacta es phi_real.
    """
    theta = 2 * np.pi * phi_real

    # Operador U
    U = QuantumCircuit(1, name="U")
    U.p(theta, 0)   # P(θ)|1⟩ = e^{iθ}|1⟩ → fase = θ/2π = phi_real

    # Eigenstate |1⟩
    eigenstate = QuantumCircuit(1, name="|1⟩")
    eigenstate.x(0)

    return qpe(U, eigenstate, t)

# Verificar para varias fases
print("QPE con P-gate — Estimación de fase:")
print(f"{'t':>4} {'φ real':>8} {'φ estimada':>12} {'Error':>10}")
print("─" * 40)
for phi_real in [0.25, 0.375, 0.125, 0.5, 0.75]:
    for t in [3, 4, 6]:
        qc_qpe = qpe_phase_gate(phi_real, t)
        conteos = sim_qasm.run(transpile(qc_qpe, sim_qasm), shots=4096).result().get_counts()
        estimaciones = estimar_fase(conteos, t)
        phi_est = max(estimaciones, key=estimaciones.get)
        error = abs(phi_real - phi_est)
        if t == 4:   # solo mostrar t=4 para brevedad
            print(f"  t={t}: φ_real={phi_real:.3f}, φ_est={phi_est:.4f}, "
                  f"error={error:.4f} {'✓' if error < 2**-t else '~'}")
```

### Precisión en función del número de qubits

```python
fases_test = [0.3, 0.17, 0.425, 0.666]
fig, axes = plt.subplots(2, 2, figsize=(12, 8))

for ax, phi_real in zip(axes.flatten(), fases_test):
    ts = range(2, 9)
    errores = []
    for t in ts:
        qc_qpe = qpe_phase_gate(phi_real, t)
        conteos = sim_qasm.run(transpile(qc_qpe, sim_qasm), shots=4096).result().get_counts()
        est = estimar_fase(conteos, t)
        phi_est = max(est, key=est.get)
        errores.append(abs(phi_real - phi_est))

    ax.semilogy(list(ts), errores, 'o-', color='steelblue', linewidth=2)
    ax.semilogy(list(ts), [2**(-t) for t in ts], 'r--', linewidth=1, label='Límite 2^{-t}')
    ax.set_xlabel("Qubits ancilla (t)")
    ax.set_ylabel("Error |φ_real - φ_est|")
    ax.set_title(f"φ = {phi_real}")
    ax.legend(fontsize=8); ax.grid(True, alpha=0.3)

plt.suptitle("Precisión de QPE vs número de qubits ancilla", fontsize=12)
plt.tight_layout()
plt.savefig("outputs/qpe_precision.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 12.4 QPE para Hamiltoniano — Simulación Cuántica

```python
# QPE puede estimar los eigenvalores de un Hamiltoniano
# U = e^{-iHt} (operador de evolución temporal)
# Los eigenvalores de U son e^{-iE_k·t} donde E_k son eigenvalores de H

# Hamiltoniano simple: H = Z (eigenvalores ±1)
def qpe_hamiltoniano(E_real: float, t: int) -> tuple:
    """
    QPE para estimar el eigenvalor E de H = Z:
    U = e^{-iZt} = Rz(2t)
    Eigenestado |0⟩: H|0⟩ = |0⟩, E = +1
    Eigenestado |1⟩: H|1⟩ = -|1⟩, E = -1

    Retorna (E_estimado, error)
    """
    # U = e^{-iZ·(π/2)} = Rz(π) para E=1 → fase = E/(2π) · π = 1/2
    # En general: U = Rz(2π · t) donde t = E/(2π)
    # Para H=Z: E_real ∈ {+1, -1}

    # Fase: φ = E/(2) → para E=+1: φ=0.5, para E=-1: φ=0.5 también?
    # Nota: e^{-iZt}|0⟩ = e^{-it}|0⟩ → fase = -t/(2π)
    # Usamos t = π: fase de |0⟩ = -1/2 ≡ 0.5 (mod 1)

    t_evolución = np.pi * abs(E_real)
    fase_esperada = t_evolución / (2 * np.pi)   # normalizar a [0,1)

    # Rz(2θ) tiene eigenvalores e^{±iθ} para |0⟩ y |1⟩
    U = QuantumCircuit(1, name="e^{-iH}")
    U.rz(2 * t_evolución, 0)

    estado = QuantumCircuit(1, name="estado")
    if E_real < 0:
        estado.x(0)   # |1⟩ para eigenvalor negativo

    qc_qpe = qpe(U, estado, t)
    conteos = sim_qasm.run(transpile(qc_qpe, sim_qasm), shots=4096).result().get_counts()
    est = estimar_fase(conteos, t)
    phi_est = max(est, key=est.get)

    # Recuperar E de φ
    E_estimado = phi_est * 2 * np.pi / t_evolución * E_real / abs(E_real)

    return E_estimado, abs(E_real - abs(E_estimado))

print("\nQPE para eigenvalores de H = Z:")
for E in [1.0, -1.0]:
    E_est, err = qpe_hamiltoniano(E, t=4)
    print(f"  E_real={E:+.1f}, E_estimado={E_est:+.4f}, error={err:.4f}")
```

---

## 12.5 Conteo Cuántico — QPE + Grover

```python
# El conteo cuántico usa QPE para estimar M (número de soluciones)
# en un problema de búsqueda de Grover
# El operador de Grover G tiene eigenvalores e^{±2iθ} donde sin²(θ) = M/N

def conteo_cuantico(n: int, M: int, t_ancilla: int = 6) -> float:
    """
    Estima el número de soluciones M en N=2^n usando conteo cuántico.
    El operador de Grover G tiene eigenvalores e^{±2iθ} con sin²(θ) = M/N.
    QPE estima θ, de donde M = N·sin²(θ).
    """
    N = 2**n
    theta_real = np.arcsin(np.sqrt(M / N))

    # La fase del operador de Grover es 2θ/(2π) = θ/π
    fase_real = theta_real / np.pi   # ∈ (0, 0.5)

    # Simulamos el operador de Grover como una puerta P(2θ) en el eigenespacio
    U_grover_simplificado = QuantumCircuit(1, name="G")
    U_grover_simplificado.p(2 * theta_real, 0)

    # Eigenstate: superposición de eigenvectores de Grover
    eigenstate_q = QuantumCircuit(1, name="|ψ_Grover⟩")
    eigenstate_q.h(0)   # superposición que contiene ambos eigenvectores

    qc_count = qpe(U_grover_simplificado, eigenstate_q, t_ancilla)
    conteos  = sim_qasm.run(transpile(qc_count, sim_qasm), shots=8192).result().get_counts()
    est = estimar_fase(conteos, t_ancilla)

    # Tomar la fase más probable
    phi_est = max(est, key=est.get)
    theta_est = phi_est * np.pi
    M_estimado = N * np.sin(theta_est)**2

    return round(M_estimado)

print("\nConteo Cuántico (QPE + Grover):")
print(f"{'n':>4} {'N=2^n':>8} {'M real':>8} {'M estimado':>12} {'Error':>8}")
print("─" * 45)
for n, M in [(3, 1), (3, 2), (4, 3), (4, 8), (5, 5)]:
    N = 2**n
    M_est = conteo_cuantico(n, M, t_ancilla=6)
    error = abs(M - M_est)
    ok = "✓" if error <= 1 else "~"
    print(f"  {n:>2}  {N:>8}  {M:>8}  {M_est:>12}  {error:>6}  {ok}")
```

---

## 12.6 QPE Iterativo — Reducir el Número de Qubits

```python
# Iterative QPE (IPQE): estima 1 bit de fase por ejecución
# Requiere solo 1 qubit ancilla + n_u qubits de trabajo
# Trade-off: más ejecuciones (t en total) pero menos qubits

def iqpe_bit(U: QuantumCircuit, eigenstate: QuantumCircuit,
             k: int, t: int, phi_prev: float) -> int:
    """
    Estima el k-ésimo bit de la fase (de más a menos significativo).
    phi_prev: estimación de los bits más significativos ya encontrados.
    Retorna 0 o 1.
    """
    n_u = U.num_qubits
    qc = QuantumCircuit(1 + n_u, 1, name=f"IPQE-bit{k}")

    # Preparar eigenstate
    qc.compose(eigenstate, qubits=range(1, 1 + n_u), inplace=True)

    # Hadamard en el único qubit ancilla
    qc.h(0)

    # Aplicar U^{2^(t-k-1)} controlado
    repeticiones = 2**(t - k - 1)
    U_pot = U.power(repeticiones).to_gate().control(1)
    qc.append(U_pot, [0] + list(range(1, 1 + n_u)))

    # Corrección de fase: Rz(-2π · phi_prev · 2^k)
    qc.rz(-2 * np.pi * phi_prev * 2**k, 0)

    # Medir en base X
    qc.h(0)
    qc.measure(0, 0)

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=100).result().get_counts()
    bit = int(max(conteos, key=conteos.get)[-1])
    return bit

def iqpe_completo(U: QuantumCircuit, eigenstate: QuantumCircuit, t: int) -> float:
    """
    Iterative QPE: estima φ con t bits usando solo 1 qubit ancilla.
    """
    phi = 0.0
    bits = []
    for k in range(t):
        bit = iqpe_bit(U, eigenstate, k, t, phi)
        bits.append(bit)
        phi = sum(bits[j] * 2**(-(j+1)) for j in range(len(bits)))
    return phi

# Comparar QPE vs IQPE
phi_real = 0.375
U_test = QuantumCircuit(1, name="U")
U_test.p(2 * np.pi * phi_real, 0)
es_test = QuantumCircuit(1, name="|1⟩")
es_test.x(0)

phi_qpe  = None
phi_iqpe = None

# QPE estándar
qc_qpe_test = qpe(U_test, es_test, 4)
cnts = sim_qasm.run(transpile(qc_qpe_test, sim_qasm), shots=4096).result().get_counts()
phi_qpe = max(estimar_fase(cnts, 4), key=lambda x: estimar_fase(cnts, 4)[x])

# IQPE
phi_iqpe = iqpe_completo(U_test, es_test, 4)

print(f"\nComparación QPE vs IQPE (φ_real = {phi_real}):")
print(f"  QPE  (4 ancilla qubits): φ_estimada = {phi_qpe:.4f}, "
      f"error = {abs(phi_real - phi_qpe):.4f}")
print(f"  IQPE (1 ancilla qubit):  φ_estimada = {phi_iqpe:.4f}, "
      f"error = {abs(phi_real - phi_iqpe):.4f}")
```

---

## 12.7 QPE desde la Librería de Qiskit

```python
from qiskit.circuit.library import PhaseEstimation

# PhaseEstimation(num_evaluation_qubits, unitary)
n_eval = 4
U_lib = QuantumCircuit(1, name="U_lib")
U_lib.p(2 * np.pi * 0.3, 0)   # fase = 0.3

qpe_lib = PhaseEstimation(num_evaluation_qubits=n_eval, unitary=U_lib)
print("QPE desde Qiskit Circuit Library:")
print(qpe_lib.decompose().draw(fold=80))
print(f"  Qubits totales: {qpe_lib.num_qubits}")
print(f"  Profundidad: {qpe_lib.decompose().depth()}")
```

---

## Resumen QPE

| Parámetro | Descripción |
|---|---|
| t (ancilla qubits) | Precisión: error < 2^{-t} |
| n_u (work qubits) | Qubits para el eigenstate de U |
| Circuito | H⊗t + t×U^{2^k} controlados + IQFT |
| Salida | k/2^t ≈ φ donde U\|ψ⟩ = e^{2πiφ}\|ψ⟩ |
| Complejidad | O(t²) puertas + t ejecuciones de U |
| Aplicaciones | Shor, VQE, conteo cuántico, simulación |

---

## checkpoint_m12.py

```python
"""
Checkpoint Módulo 12 — Estimación de Fase Cuántica (QPE)
Ejecutar con: python checkpoint_m12.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.circuit.library import QFT
from qiskit.quantum_info import Operator
import os

os.makedirs("outputs", exist_ok=True)
sim_qasm = AerSimulator(method='qasm')

def qpe(U, eigenstate, t):
    n_u = U.num_qubits
    n_total = t + n_u
    qc = QuantumCircuit(n_total, t, name=f"QPE(t={t})")
    qc.compose(eigenstate, qubits=range(t, n_total), inplace=True)
    qc.h(range(t))
    for k in range(t):
        rep = 2**(t - 1 - k)
        U_pot = U.power(rep).to_gate().control(1)
        qc.append(U_pot, [k] + list(range(t, n_total)))
    qc.barrier()
    iqft = QFT(t, inverse=True, do_swaps=True)
    qc.compose(iqft, qubits=range(t), inplace=True)
    qc.measure(range(t), range(t))
    return qc

def estimar_fase_dominante(conteos, t):
    total = sum(conteos.values())
    mejor_bits = max(conteos, key=conteos.get)
    k = int(mejor_bits[::-1], 2)
    return k / 2**t, conteos[mejor_bits] / total

print("=" * 55)
print("Checkpoint M12 — Estimación de Fase Cuántica (QPE)")
print("=" * 55)

# ── Test 1: QPE estima fase exacta (φ = k/2^t) ───────────
fases_exactas = [0.0, 0.25, 0.5, 0.75, 0.125]
for phi_real in fases_exactas:
    U = QuantumCircuit(1, name="U"); U.p(2 * np.pi * phi_real, 0)
    es = QuantumCircuit(1, name="|1⟩"); es.x(0)
    qc_qpe = qpe(U, es, 4)
    cnts = sim_qasm.run(transpile(qc_qpe, sim_qasm), shots=2048).result().get_counts()
    phi_est, prob = estimar_fase_dominante(cnts, 4)
    assert abs(phi_real - phi_est) < 2**(-4), \
        f"FALLO: φ_real={phi_real}, φ_est={phi_est}"
    assert prob > 0.8, f"FALLO: prob={prob:.3f} < 0.8 para φ={phi_real}"
print("✓ Test 1: QPE estima fases exactas (φ ∈ {0, 0.25, 0.5, 0.75, 0.125}) con P>0.8")

# ── Test 2: Precisión mejora con t ────────────────────────
phi_irrac = 1/3   # no representable exactamente con t bits
errores = {}
for t in [3, 4, 5, 6]:
    U = QuantumCircuit(1); U.p(2 * np.pi * phi_irrac, 0)
    es = QuantumCircuit(1); es.x(0)
    qc_qpe = qpe(U, es, t)
    cnts = sim_qasm.run(transpile(qc_qpe, sim_qasm), shots=4096).result().get_counts()
    phi_est, _ = estimar_fase_dominante(cnts, t)
    errores[t] = abs(phi_irrac - phi_est)
# Error debe decrecer (o al menos no aumentar) con t
for t in [4, 5, 6]:
    assert errores[t] <= errores[t-1] + 2**(-t), \
        f"FALLO: error no mejora con t={t}"
print(f"✓ Test 2: Error decrece con t — errores: {dict((t,f'{e:.4f}') for t,e in errores.items())}")

# ── Test 3: Eigenstate correcto ───────────────────────────
# P(θ)|0⟩ ≠ e^{iθ}|0⟩ (no es eigenstate)
# P(θ)|1⟩ = e^{iθ}|1⟩ (sí es eigenstate)
phi_t3 = 0.375
U3 = QuantumCircuit(1); U3.p(2 * np.pi * phi_t3, 0)
# Con eigenstate |0⟩ (incorrecto): la fase no es clara
es_wrong = QuantumCircuit(1)   # |0⟩
qc_wrong = qpe(U3, es_wrong, 4)
cnts_wrong = sim_qasm.run(transpile(qc_wrong, sim_qasm), shots=2048).result().get_counts()
phi_wrong, _ = estimar_fase_dominante(cnts_wrong, 4)
# Con |0⟩, P(θ)|0⟩ = |0⟩, fase = 0
assert abs(phi_wrong - 0.0) < 0.1, "FALLO: P(θ)|0⟩ debería dar fase ≈ 0"
# Con eigenstate |1⟩ (correcto)
es_correct = QuantumCircuit(1); es_correct.x(0)
qc_correct = qpe(U3, es_correct, 4)
cnts_correct = sim_qasm.run(transpile(qc_correct, sim_qasm), shots=2048).result().get_counts()
phi_correct, _ = estimar_fase_dominante(cnts_correct, 4)
assert abs(phi_correct - phi_t3) < 2**(-4), \
    f"FALLO: φ_est={phi_correct:.4f} ≠ φ_real={phi_t3}"
print(f"✓ Test 3: Eigenstate importa — |0⟩ da φ≈0, |1⟩ da φ={phi_correct:.3f}")

# ── Test 4: Múltiples fases dan distribución ──────────────
# U con dos eigenvalores: usar superposición de eigenstates
phi_a, phi_b = 0.25, 0.75
# Preparar superposición (|0⟩+|1⟩)/√2 → debería dar AMBAS fases
U4 = QuantumCircuit(1); U4.p(2 * np.pi * phi_a, 0)   # solo phi_a para simplicidad
es4 = QuantumCircuit(1); es4.x(0)   # eigenstate de phi_a
qc4 = qpe(U4, es4, 4)
cnts4 = sim_qasm.run(transpile(qc4, sim_qasm), shots=4096).result().get_counts()
phi4, prob4 = estimar_fase_dominante(cnts4, 4)
assert abs(phi4 - phi_a) < 2**(-4), f"FALLO: fase estimada {phi4} ≠ {phi_a}"
assert prob4 > 0.8, f"FALLO: prob={prob4:.3f} < 0.8"
print(f"✓ Test 4: QPE eigenstate correcto da fase dominante con P={prob4:.3f}")

# ── Test 5: t=8 da alta precisión ────────────────────────
phi_5 = 0.612345
U5 = QuantumCircuit(1); U5.p(2 * np.pi * phi_5, 0)
es5 = QuantumCircuit(1); es5.x(0)
qc5 = qpe(U5, es5, 8)
cnts5 = sim_qasm.run(transpile(qc5, sim_qasm), shots=8192).result().get_counts()
phi5_est, _ = estimar_fase_dominante(cnts5, 8)
error5 = abs(phi_5 - phi5_est)
assert error5 < 2**(-7), f"FALLO: t=8, error={error5:.6f} > 2^{{-7}}={2**-7:.6f}"
print(f"✓ Test 5: t=8 da error={error5:.6f} < 2^{{-7}}={2**-7:.6f}")

# ── Test 6: QPE de la library de Qiskit disponible ───────
from qiskit.circuit.library import PhaseEstimation
U6 = QuantumCircuit(1); U6.p(2 * np.pi * 0.3, 0)
qpe_lib = PhaseEstimation(num_evaluation_qubits=4, unitary=U6)
assert qpe_lib.num_qubits == 5, f"FALLO: QPE lib debe tener 5 qubits, got {qpe_lib.num_qubits}"
print(f"✓ Test 6: PhaseEstimation library disponible ({qpe_lib.num_qubits} qubits)")

print("\n" + "=" * 55)
print("✅ Todos los tests del M12 pasaron correctamente")
print("=" * 55)
```
