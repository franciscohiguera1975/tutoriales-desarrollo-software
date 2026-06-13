# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 3 — Puertas Cuánticas y Lectura de Circuitos

> **Objetivo:** Dominar el conjunto completo de puertas cuánticas de una y dos qubits, leer y depurar circuitos cuánticos, y construir circuitos reutilizables con Qiskit. Incluye la descomposición de puertas universales y la verificación matemática de operaciones unitarias.
>
> **Herramientas:** Qiskit + Qiskit Aer + NumPy.
>
> **Prerequisito:** M02 — Qubits, Superposición y Entrelazamiento.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-03-puertas-circuitos.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-03-puertas-circuitos.ipynb
```

### Opción C — Google Colab
```python
# Celda 1: instalación
!pip install qiskit qiskit-aer pylatexenc -q
```

---

## 3.1 Representación Matricial de las Puertas

### Por qué toda puerta cuántica es una matriz unitaria

```python
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.quantum_info import Operator
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

sim_sv = AerSimulator(method='statevector')

def sv(qc: QuantumCircuit) -> np.ndarray:
    """Obtiene el statevector de un circuito."""
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim_sv.run(transpile(qc2, sim_sv)).result().get_statevector())

def es_unitaria(M: np.ndarray, tol: float = 1e-9) -> bool:
    """Verifica U†·U = I."""
    producto = M.conj().T @ M
    return np.allclose(producto, np.eye(len(M)), atol=tol)

def matriz_de_puerta(nombre: str, qc: QuantumCircuit) -> np.ndarray:
    """Extrae la matriz unitaria de un circuito de una sola puerta."""
    op = Operator(qc)
    return op.data

# ── Las puertas de Pauli ─────────────────────────────────
# X (NOT cuántico), Y, Z
X = np.array([[0, 1], [1, 0]], dtype=complex)
Y = np.array([[0, -1j],[1j,  0]], dtype=complex)
Z = np.array([[1, 0], [0, -1]], dtype=complex)
I = np.eye(2, dtype=complex)

print("Verificación de unitariedad de las puertas de Pauli:")
for nombre, M in [("X", X), ("Y", Y), ("Z", Z)]:
    print(f"  {nombre}: {'✓ unitaria' if es_unitaria(M) else '✗ NO unitaria'}")
    print(f"    {nombre}² = I: {np.allclose(M @ M, I)}")  # todas son involutivas

# Obtener matrices directamente de Qiskit
for gate_name in ['x', 'y', 'z']:
    qc = QuantumCircuit(1)
    getattr(qc, gate_name)(0)
    M_qiskit = matriz_de_puerta(gate_name, qc)
    print(f"\n  {gate_name.upper()} (Qiskit):\n{np.round(M_qiskit, 2)}")
```

---

## 3.2 Puertas de Un Qubit — Catálogo Completo

### Puertas de Pauli y Hadamard

```python
def visualizar_puertas_1q():
    """Aplica cada puerta al estado |+⟩ y muestra el resultado."""
    fig, axes = plt.subplots(3, 4, figsize=(16, 10))
    fig.suptitle("Puertas de un qubit — Efecto sobre |+⟩", fontsize=13)

    # (nombre_legible, función_qiskit, args)
    puertas = [
        ("I (Identidad)",   lambda qc: None,                    {}),
        ("X (NOT)",         lambda qc: qc.x(0),                 {}),
        ("Y",               lambda qc: qc.y(0),                 {}),
        ("Z",               lambda qc: qc.z(0),                 {}),
        ("H (Hadamard)",    lambda qc: qc.h(0),                 {}),
        ("S (T²)",          lambda qc: qc.s(0),                 {}),
        ("S† (S†)",         lambda qc: qc.sdg(0),               {}),
        ("T (S√)",          lambda qc: qc.t(0),                 {}),
        ("T†",              lambda qc: qc.tdg(0),               {}),
        ("Rx(π/4)",         lambda qc: qc.rx(np.pi/4, 0),       {}),
        ("Ry(π/3)",         lambda qc: qc.ry(np.pi/3, 0),       {}),
        ("Rz(π/2)",         lambda qc: qc.rz(np.pi/2, 0),       {}),
    ]

    for ax, (nombre, aplicar, _) in zip(axes.flatten(), puertas):
        qc = QuantumCircuit(1)
        qc.h(0)   # partir de |+⟩
        if aplicar(qc) is not None or nombre != "I (Identidad)":
            aplicar(qc)

        sv_result = sv(qc)
        prob0 = abs(sv_result[0])**2
        prob1 = abs(sv_result[1])**2

        ax.bar(['|0⟩', '|1⟩'], [prob0, prob1], color=['steelblue', 'coral'])
        ax.set_ylim(0, 1.2)
        ax.set_title(nombre, fontsize=10)
        ax.set_ylabel("P", fontsize=8)
        ax.text(0, prob0 + 0.03, f"{prob0:.2f}", ha='center', fontsize=8)
        ax.text(1, prob1 + 0.03, f"{prob1:.2f}", ha='center', fontsize=8)

        # Mostrar las amplitudes
        ax.text(0.5, 0.9, f"α={sv_result[0]:.2f}\nβ={sv_result[1]:.2f}",
                transform=ax.transAxes, ha='center', fontsize=7,
                bbox=dict(boxstyle='round', facecolor='lightyellow', alpha=0.7))

    plt.tight_layout()
    plt.savefig("outputs/puertas_1qubit.png", dpi=150, bbox_inches='tight')
    plt.show()

visualizar_puertas_1q()
```

### Matrices de las puertas más importantes

```python
def imprimir_matriz(nombre: str, qc: QuantumCircuit):
    M = Operator(qc).data
    print(f"\n── {nombre} ──")
    for fila in M:
        vals = "  ".join(f"{v.real:+.3f}{v.imag:+.3f}j" for v in fila)
        print(f"  [{vals}]")
    print(f"  Unitaria: {es_unitaria(M)}")

for nombre, gate_fn in [
    ("X",    lambda qc: qc.x(0)),
    ("Y",    lambda qc: qc.y(0)),
    ("Z",    lambda qc: qc.z(0)),
    ("H",    lambda qc: qc.h(0)),
    ("S",    lambda qc: qc.s(0)),
    ("T",    lambda qc: qc.t(0)),
    ("Rx(π/2)", lambda qc: qc.rx(np.pi/2, 0)),
    ("Ry(π/2)", lambda qc: qc.ry(np.pi/2, 0)),
]:
    qc = QuantumCircuit(1)
    gate_fn(qc)
    imprimir_matriz(nombre, qc)
```

### La puerta U — representación universal de 1 qubit

```python
# Toda puerta de 1 qubit = U(θ, φ, λ) · fase_global
# U(θ,φ,λ) = [[cos(θ/2),            -e^{iλ}·sin(θ/2)],
#              [e^{iφ}·sin(θ/2),      e^{i(φ+λ)}·cos(θ/2)]]

def puerta_U_manual(theta: float, phi: float, lam: float) -> np.ndarray:
    return np.array([
        [np.cos(theta/2),               -np.exp(1j*lam) * np.sin(theta/2)],
        [np.exp(1j*phi) * np.sin(theta/2), np.exp(1j*(phi+lam)) * np.cos(theta/2)]
    ])

# Verificar que X = U(π, 0, π) (hasta fase global)
U_X = puerta_U_manual(np.pi, 0, np.pi)
print(f"U(π, 0, π):\n{np.round(U_X, 3)}")
print(f"Coincide con X: {np.allclose(np.abs(U_X), np.abs(X))}")

# H = U(π/2, 0, π)
U_H = puerta_U_manual(np.pi/2, 0, np.pi)
H = np.array([[1, 1], [1, -1]], dtype=complex) / np.sqrt(2)
print(f"\nU(π/2, 0, π) == H: {np.allclose(np.abs(U_H), np.abs(H))}")

# Qiskit: qc.u(theta, phi, lam, qubit)
qc_U = QuantumCircuit(1)
qc_U.u(np.pi/3, np.pi/4, np.pi/6, 0)
M_U = Operator(qc_U).data
U_manual = puerta_U_manual(np.pi/3, np.pi/4, np.pi/6)
print(f"\nU(π/3,π/4,π/6) == Qiskit: {np.allclose(M_U, U_manual, atol=1e-6)}")
```

---

## 3.3 Puertas de Dos Qubits — CNOT, CZ, SWAP y más

### CNOT — la puerta de dos qubits más fundamental

```python
# CNOT (Controlled-NOT): si el qubit de control es |1⟩, aplica X al target
# Matriz 4×4:
# |00⟩ → |00⟩
# |01⟩ → |01⟩
# |10⟩ → |11⟩
# |11⟩ → |10⟩

qc_CNOT = QuantumCircuit(2)
qc_CNOT.cx(0, 1)   # control=q0, target=q1
M_CNOT = Operator(qc_CNOT).data
print("Matriz CNOT (Qiskit, base computacional):")
print(np.round(M_CNOT.real, 2))
# [[1 0 0 0]
#  [0 0 0 1]    ← Qiskit ordena: q1,q0
#  [0 0 1 0]
#  [0 1 0 0]]
# Nota: Qiskit indexa los estados como |q1 q0⟩

# Verificación de la tabla de verdad
print("\nTabla de verdad CNOT:")
for estado_nombre, init_fn in [
    ("|00⟩", lambda qc: None),
    ("|01⟩", lambda qc: qc.x(0)),
    ("|10⟩", lambda qc: qc.x(1)),
    ("|11⟩", lambda qc: [qc.x(0), qc.x(1)]),
]:
    qc = QuantumCircuit(2)
    resultado = init_fn(qc)
    qc.cx(0, 1)
    sv_r = sv(qc)
    estado_salida = max(range(4), key=lambda i: abs(sv_r[i])**2)
    bits = format(estado_salida, '02b')
    print(f"  CNOT {estado_nombre} → |{bits}⟩ (P={abs(sv_r[estado_salida])**2:.2f})")
```

### Las puertas de 2 qubits más comunes

```python
# ── CZ — Controlled-Z ───────────────────────────────────
# Aplica Z al target si el control es |1⟩
# Simétrica: CZ(q0,q1) = CZ(q1,q0)
qc_CZ = QuantumCircuit(2)
qc_CZ.cz(0, 1)
M_CZ = Operator(qc_CZ).data
print("Matriz CZ:\n", np.round(M_CZ.real, 2))

# ── SWAP — intercambia los estados de dos qubits ─────────
qc_SWAP = QuantumCircuit(2)
qc_SWAP.swap(0, 1)
M_SWAP = Operator(qc_SWAP).data
print("\nMatriz SWAP:\n", np.round(M_SWAP.real, 2))

# SWAP = 3 CNOTs
qc_SWAP_3cx = QuantumCircuit(2)
qc_SWAP_3cx.cx(0, 1)
qc_SWAP_3cx.cx(1, 0)
qc_SWAP_3cx.cx(0, 1)
M_SWAP_3cx = Operator(qc_SWAP_3cx).data
print(f"\nSWAP == 3×CX: {np.allclose(M_SWAP, M_SWAP_3cx)}")

# ── iSWAP — variante común en hardware superconductor ────
qc_iSWAP = QuantumCircuit(2)
qc_iSWAP.iswap(0, 1)
M_iSWAP = Operator(qc_iSWAP).data
print("\nMatriz iSWAP:\n", np.round(M_iSWAP, 2))

# ── ECR — puerta nativa en IBM Eagle ─────────────────────
# (Effective Cross-Resonance) = la puerta física de 2 qubits en IBM
qc_ECR = QuantumCircuit(2)
qc_ECR.ecr(0, 1)
M_ECR = Operator(qc_ECR).data
print("\nMatriz ECR:\n", np.round(M_ECR, 2))
print(f"ECR unitaria: {es_unitaria(M_ECR)}")
```

### Puertas controladas generalizadas

```python
# ── CH — Hadamard Controlado ─────────────────────────────
qc_CH = QuantumCircuit(2)
qc_CH.ch(0, 1)

# ── CRx, CRy, CRz — Rotaciones controladas ──────────────
qc_CRy = QuantumCircuit(2)
qc_CRy.cry(np.pi/4, 0, 1)

# ── CCX — Toffoli (CCNOT): 3 qubits ─────────────────────
# Aplica X al qubit objetivo si los DOS controles son |1⟩
# Puerta clásicamente universal: puede implementar AND, OR, NOT
qc_Toffoli = QuantumCircuit(3)
qc_Toffoli.ccx(0, 1, 2)   # controles: q0, q1; target: q2
M_Toffoli = Operator(qc_Toffoli).data
print("Toffoli (8×8):")
print(np.round(M_Toffoli.real[:8, :8], 1))

# Verificar: CCX|110⟩ → |111⟩
qc_t_test = QuantumCircuit(3)
qc_t_test.x(0); qc_t_test.x(1)   # preparar |110⟩ (q1=1, q0=1, q2=0)
qc_t_test.ccx(0, 1, 2)
sv_t = sv(qc_t_test)
estado = max(range(8), key=lambda i: abs(sv_t[i])**2)
print(f"\nToffoli |110⟩ → |{format(estado,'03b')}⟩ (debe ser 111)")

# ── CSWAP — Fredkin ──────────────────────────────────────
qc_Fredkin = QuantumCircuit(3)
qc_Fredkin.cswap(0, 1, 2)   # control: q0; swap q1 ↔ q2
```

---

## 3.4 Descomposición de Puertas — Universalidad

### El conjunto {H, T, CNOT} es universal

```python
# Cualquier puerta unitaria arbitraria puede aproximarse
# con H, T y CNOT con precisión arbitraria (Teorema de Solovay-Kitaev)

# Ejemplo: Ry(θ) = Rz(-π/2) · Rx(θ) · Rz(π/2)
# Y Rx/Rz se descomponen en H, S, T, CNOT

# Verificar la descomposición de S = T·T
qc_SS = QuantumCircuit(1)
qc_SS.t(0); qc_SS.t(0)   # T² = S
M_SS = Operator(qc_SS).data

qc_S = QuantumCircuit(1); qc_S.s(0)
M_S  = Operator(qc_S).data
print(f"T·T == S: {np.allclose(M_SS, M_S, atol=1e-9)}")

# Descomposición de SWAP usando 3 CNOT
qc_swap_manual = QuantumCircuit(2)
qc_swap_manual.cx(0, 1); qc_swap_manual.cx(1, 0); qc_swap_manual.cx(0, 1)
M_manual = Operator(qc_swap_manual).data
M_swap   = Operator(QuantumCircuit(2).copy()).data   # vacío
qc_swap_ref = QuantumCircuit(2); qc_swap_ref.swap(0, 1)
M_swap_ref = Operator(qc_swap_ref).data
print(f"3×CNOT == SWAP: {np.allclose(M_manual, M_swap_ref, atol=1e-9)}")

# Descomposición de CZ mediante CNOT + H
# CZ = (I ⊗ H) · CNOT · (I ⊗ H)
qc_CZ_manual = QuantumCircuit(2)
qc_CZ_manual.h(1)          # H en qubit target
qc_CZ_manual.cx(0, 1)      # CNOT
qc_CZ_manual.h(1)          # H en qubit target
M_CZ_manual = Operator(qc_CZ_manual).data

qc_CZ_ref = QuantumCircuit(2); qc_CZ_ref.cz(0, 1)
M_CZ_ref = Operator(qc_CZ_ref).data
print(f"H·CNOT·H == CZ: {np.allclose(M_CZ_manual, M_CZ_ref, atol=1e-9)}")
```

### Transpilación — cómo Qiskit descompone puertas

```python
from qiskit.compiler import transpile as qk_transpile
from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager

# Un circuito con Toffoli se puede transpilar al basis set de IBM
qc_demo = QuantumCircuit(3)
qc_demo.ccx(0, 1, 2)   # Toffoli de alto nivel

# Basis set de IBM (Eagle r3): cx, id, rz, sx, x
qc_transpilado = qk_transpile(
    qc_demo,
    basis_gates=['cx', 'id', 'rz', 'sx', 'x'],
    optimization_level=1
)
print("Toffoli transpilado al basis set IBM:")
print(qc_transpilado.draw())
print(f"\nPuertas en el circuito original: {dict(qc_demo.count_ops())}")
print(f"Puertas después de transpilar:   {dict(qc_transpilado.count_ops())}")
print(f"Profundidad original:  {qc_demo.depth()}")
print(f"Profundidad transpilada: {qc_transpilado.depth()}")
```

---

## 3.5 Construcción de Circuitos — Herramientas de Qiskit

### Dibujar circuitos

```python
# ── Distintos formatos de visualización ─────────────────
qc_visual = QuantumCircuit(3, 2, name="Demo")
qc_visual.h(0)
qc_visual.cx(0, 1)
qc_visual.cx(1, 2)
qc_visual.barrier()
qc_visual.measure([0, 1], [0, 1])
qc_visual.rz(np.pi/4, 2)

# Texto (siempre disponible, sin dependencias extras)
print("── Formato texto ──")
print(qc_visual.draw('text'))

# Matplotlib (requiere pylatexenc para el formato 'mpl')
try:
    fig = qc_visual.draw('mpl', style={'backgroundcolor': '#EEEEEE'})
    fig.savefig("outputs/circuito_demo.png", dpi=150, bbox_inches='tight')
    print("\n── Formato matplotlib guardado en outputs/circuito_demo.png ──")
except Exception as e:
    print(f"Matplotlib draw no disponible: {e}")
```

### Registros, mediciones y bits clásicos

```python
from qiskit import QuantumRegister, ClassicalRegister

# Registros con nombres descriptivos
q_alice = QuantumRegister(2, 'alice')
q_bob   = QuantumRegister(1, 'bob')
c_alice = ClassicalRegister(2, 'c_alice')
c_bob   = ClassicalRegister(1, 'c_bob')

qc_regs = QuantumCircuit(q_alice, q_bob, c_alice, c_bob)
qc_regs.h(q_alice[0])
qc_regs.cx(q_alice[0], q_alice[1])
qc_regs.cx(q_alice[1], q_bob[0])
qc_regs.measure(q_alice, c_alice)
qc_regs.measure(q_bob,   c_bob)
print(qc_regs.draw())
print(f"\nQubits: {qc_regs.num_qubits}, Bits clásicos: {qc_regs.num_clbits}")
```

### Circuitos paramétricos

```python
from qiskit.circuit import Parameter, ParameterVector

# Un parámetro simbólico — útil para VQC y optimización
theta = Parameter('θ')
phi   = Parameter('φ')

qc_param = QuantumCircuit(2)
qc_param.ry(theta, 0)
qc_param.rz(phi, 1)
qc_param.cx(0, 1)
qc_param.ry(theta / 2, 1)   # reutilizar el mismo parámetro
print("Circuito paramétrico:")
print(qc_param.draw())
print(f"Parámetros: {qc_param.parameters}")

# Vincular valores concretos
qc_concreto = qc_param.assign_parameters({theta: np.pi/3, phi: np.pi/4})
sv_concreto = sv(qc_concreto)
print(f"\nCircuito con θ=60°, φ=45°:")
print(f"  Statevector: {np.round(sv_concreto, 3)}")

# ParameterVector — útil para VQC con muchos parámetros
params = ParameterVector('θ', 4)
qc_vqc = QuantumCircuit(2)
qc_vqc.ry(params[0], 0); qc_vqc.ry(params[1], 1)
qc_vqc.cx(0, 1)
qc_vqc.ry(params[2], 0); qc_vqc.ry(params[3], 1)
print(f"\nVQC con ParameterVector de 4 parámetros: {qc_vqc.parameters}")
```

### Composición y subcircuitos

```python
# Módulos reutilizables de circuito

def bloque_entrelazamiento(n: int) -> QuantumCircuit:
    """Aplica CNOTs en cadena: 0→1, 1→2, ..., n-2→n-1."""
    qc = QuantumCircuit(n, name="Ent")
    for i in range(n - 1):
        qc.cx(i, i + 1)
    return qc

def capa_rotaciones(n: int, params: ParameterVector) -> QuantumCircuit:
    """Capa de Ry sobre todos los qubits con parámetros independientes."""
    assert len(params) >= n, "Faltan parámetros"
    qc = QuantumCircuit(n, name="Rot")
    for i in range(n):
        qc.ry(params[i], i)
    return qc

# Construir VQC modular
n_qubits = 3
ps = ParameterVector('p', 2 * n_qubits)

qc_modular = QuantumCircuit(n_qubits)
qc_modular.compose(capa_rotaciones(n_qubits, ps[:n_qubits]),     inplace=True)
qc_modular.compose(bloque_entrelazamiento(n_qubits),             inplace=True)
qc_modular.compose(capa_rotaciones(n_qubits, ps[n_qubits:]),     inplace=True)

print("VQC modular (2 capas + entrelazamiento):")
print(qc_modular.draw())
print(f"Parámetros totales: {len(qc_modular.parameters)}")

# Convertir a instrucción reutilizable
from qiskit.circuit.library import BlueprintCircuit
bloque_inst = qc_modular.to_instruction(label="VQC-layer")
```

### Inversa de un circuito

```python
# qc.inverse() devuelve U† (el circuito adjunto)
qc_original = QuantumCircuit(2)
qc_original.h(0)
qc_original.cx(0, 1)
qc_original.rz(np.pi/4, 1)

qc_inverso = qc_original.inverse()
print("Circuito original:")
print(qc_original.draw())
print("\nCircuito inverso (U†):")
print(qc_inverso.draw())

# Verificar que U · U† = I
qc_UUd = QuantumCircuit(2)
qc_UUd.compose(qc_original, inplace=True)
qc_UUd.compose(qc_inverso, inplace=True)
sv_UUd = sv(qc_UUd)
print(f"\nU · U†|00⟩ = {np.round(sv_UUd, 3)}")
print(f"¿Es |00⟩? {np.allclose(sv_UUd, [1,0,0,0], atol=1e-6)}")
```

---

## 3.6 Medición y Colapso

### Midiendo en distintas bases

```python
# La medición estándar es en la base Z (base computacional)
# Para medir en otras bases, aplicamos una rotación ANTES de medir

def medir_base(qc_preparacion: QuantumCircuit, base: str, shots: int = 4096) -> dict:
    """
    Mide un circuito de 1 qubit en la base especificada.
    base: 'Z' (computacional), 'X', 'Y'
    """
    sim = AerSimulator(method='qasm')
    qc = qc_preparacion.copy()
    if base == 'X':
        qc.h(0)          # Rotar base X → Z antes de medir
    elif base == 'Y':
        qc.sdg(0); qc.h(0)   # Rotar base Y → Z
    qc.measure_all()
    return sim.run(transpile(qc, sim), shots=shots).result().get_counts()

# Estado |+⟩ = (|0⟩+|1⟩)/√2
qc_plus = QuantumCircuit(1); qc_plus.h(0)

c_Z = medir_base(qc_plus, 'Z')  # aleatorio 50/50
c_X = medir_base(qc_plus, 'X')  # siempre |+⟩ en base X → siempre 0
c_Y = medir_base(qc_plus, 'Y')  # aleatorio en base Y

total = 4096
print("Estado |+⟩ medido en distintas bases:")
print(f"  Base Z: {c_Z}  →  {c_Z.get('0',0)/total:.2f} / {c_Z.get('1',0)/total:.2f}")
print(f"  Base X: {c_X}  →  P(0) = {c_X.get('0',0)/total:.2f}  (debe ser ≈ 1)")
print(f"  Base Y: {c_Y}  →  {c_Y.get('0',0)/total:.2f} / {c_Y.get('1',0)/total:.2f}")
```

### Medición parcial y colapso del estado global

```python
# En un sistema entrelazado, medir UN qubit colapsa AMBOS

from qiskit_aer import AerSimulator

sim_sv2 = AerSimulator(method='statevector')

# Preparar Bell |Φ+⟩
qc_bell = QuantumCircuit(2, 1)
qc_bell.h(0)
qc_bell.cx(0, 1)
# Medir solo el qubit 0
qc_bell.measure(0, 0)
qc_bell.save_statevector()

job = sim_sv2.run(transpile(qc_bell, sim_sv2), shots=4)
result = job.result()
# Con el simulador de statevector, el estado colapsa al medir
sv_post = np.array(result.get_statevector())
conteos_parcial = result.get_counts()

print("Estado Bell |Φ+⟩ después de medir qubit 0:")
print(f"  Statevector post-medición: {np.round(sv_post, 3)}")
print(f"  Conteos: {conteos_parcial}")
# El statevector del qubit 1 colapsa al mismo valor que q0
```

---

## 3.7 Análisis de Circuitos — Profundidad y Recursos

```python
def analizar_circuito(qc: QuantumCircuit, nombre: str = ""):
    """Imprime métricas de un circuito."""
    print(f"\n── Análisis: {nombre or qc.name} ──")
    print(f"  Qubits:     {qc.num_qubits}")
    print(f"  Clbits:     {qc.num_clbits}")
    print(f"  Profundidad (depth):  {qc.depth()}")
    print(f"  Ancho (width):        {qc.width()}")
    print(f"  Total de puertas:     {qc.size()}")
    print(f"  Conteo de operaciones: {dict(qc.count_ops())}")
    parametros = qc.parameters
    if parametros:
        print(f"  Parámetros libres: {[str(p) for p in parametros]}")

# Comparar distintos circuitos
qc_ghz5 = QuantumCircuit(5)
qc_ghz5.h(0)
for i in range(4): qc_ghz5.cx(i, i+1)
analizar_circuito(qc_ghz5, "GHZ-5")

qc_qft3 = QuantumCircuit(3)
# QFT de 3 qubits manual
qc_qft3.h(0)
qc_qft3.cp(np.pi/2, 0, 1)
qc_qft3.cp(np.pi/4, 0, 2)
qc_qft3.h(1)
qc_qft3.cp(np.pi/2, 1, 2)
qc_qft3.h(2)
qc_qft3.swap(0, 2)
analizar_circuito(qc_qft3, "QFT-3")

# Impacto de la transpilación en la profundidad
qc_toffoli = QuantumCircuit(3); qc_toffoli.ccx(0, 1, 2)
analizar_circuito(qc_toffoli, "Toffoli (abstracto)")
qc_tofo_t = qk_transpile(qc_toffoli, basis_gates=['cx','rz','sx','x'], optimization_level=3)
analizar_circuito(qc_tofo_t, "Toffoli (transpilado)")
```

---

## 3.8 Fidelidad de Proceso — Verificar Implementaciones

```python
from qiskit.quantum_info import process_fidelity, Operator

def verificar_equivalencia(qc1: QuantumCircuit, qc2: QuantumCircuit, nombre: str = ""):
    """
    Calcula la fidelidad de proceso entre dos circuitos.
    Fidelidad = 1.0 → circuitos idénticos (hasta fase global).
    """
    op1 = Operator(qc1)
    op2 = Operator(qc2)
    f   = process_fidelity(op1, op2)
    print(f"Fidelidad {nombre}: {f:.8f} {'✓' if f > 0.9999 else '✗'}")
    return f

# Equivalencias bien conocidas
qc_CZ_a = QuantumCircuit(2); qc_CZ_a.cz(0, 1)
qc_CZ_b = QuantumCircuit(2); qc_CZ_b.h(1); qc_CZ_b.cx(0, 1); qc_CZ_b.h(1)
verificar_equivalencia(qc_CZ_a, qc_CZ_b, "CZ == H·CNOT·H")

qc_SWAP_a = QuantumCircuit(2); qc_SWAP_a.swap(0, 1)
qc_SWAP_b = QuantumCircuit(2)
qc_SWAP_b.cx(0,1); qc_SWAP_b.cx(1,0); qc_SWAP_b.cx(0,1)
verificar_equivalencia(qc_SWAP_a, qc_SWAP_b, "SWAP == 3×CNOT")

qc_HH = QuantumCircuit(1); qc_HH.h(0); qc_HH.h(0)
qc_Id = QuantumCircuit(1)   # identidad (circuito vacío)
verificar_equivalencia(qc_HH, qc_Id, "H·H == I")

qc_SS = QuantumCircuit(1); qc_SS.s(0); qc_SS.s(0)
qc_Z  = QuantumCircuit(1); qc_Z.z(0)
verificar_equivalencia(qc_SS, qc_Z, "S·S == Z")
```

---

## 3.9 Librería de Circuitos — Qiskit Circuit Library

```python
from qiskit.circuit.library import (
    QFT, GroverOperator, PhaseEstimation,
    TwoLocal, EfficientSU2, RealAmplitudes,
    ZZFeatureMap, ZFeatureMap,
    QuantumVolume
)

# QFT — Transformada de Fourier Cuántica
qft4 = QFT(4, approximation_degree=0, do_swaps=True)
print("QFT de 4 qubits:")
print(qft4.draw(fold=80))
analizar_circuito(qft4, "QFT-4")

# TwoLocal — ansatz variacional genérico
tl = TwoLocal(3, ['ry','rz'], 'cx', reps=2)
print("\nTwoLocal (ry+rz, cx, 2 reps):")
print(tl.draw(fold=80))
print(f"Parámetros: {len(tl.parameters)}")

# EfficientSU2 — ansatz con puertas SU(2) eficientes
eff = EfficientSU2(3, reps=1)
analizar_circuito(eff, "EfficientSU2")

# RealAmplitudes — solo rotaciones Ry + CNOT (amplitudes reales)
ra = RealAmplitudes(3, reps=2)
analizar_circuito(ra, "RealAmplitudes")

# ZZFeatureMap — mapa de características para QSVM
zz = ZZFeatureMap(3, reps=2)
analizar_circuito(zz, "ZZFeatureMap-3")
```

---

## Resumen de puertas cuánticas

| Puerta | Qubits | Qiskit | Acción |
|---|---|---|---|
| X (NOT) | 1 | `qc.x(q)` | \|0⟩↔\|1⟩ |
| Y | 1 | `qc.y(q)` | \|0⟩→i\|1⟩, \|1⟩→-i\|0⟩ |
| Z | 1 | `qc.z(q)` | \|1⟩→-\|1⟩ (fase) |
| H | 1 | `qc.h(q)` | \|0⟩→\|+⟩, \|1⟩→\|−⟩ |
| S | 1 | `qc.s(q)` | \|1⟩→i\|1⟩ (fase π/2) |
| T | 1 | `qc.t(q)` | \|1⟩→e^{iπ/4}\|1⟩ (fase π/4) |
| Rx(θ) | 1 | `qc.rx(θ,q)` | Rotación eje X de ángulo θ |
| Ry(θ) | 1 | `qc.ry(θ,q)` | Rotación eje Y — solo amplitudes reales |
| Rz(θ) | 1 | `qc.rz(θ,q)` | Rotación eje Z (solo fase) |
| U(θ,φ,λ) | 1 | `qc.u(θ,φ,λ,q)` | Puerta universal de 1 qubit |
| CNOT | 2 | `qc.cx(c,t)` | X en target si control=\|1⟩ |
| CZ | 2 | `qc.cz(c,t)` | Z en target si control=\|1⟩ |
| SWAP | 2 | `qc.swap(a,b)` | Intercambia a↔b |
| Toffoli | 3 | `qc.ccx(c0,c1,t)` | X en target si c0=c1=\|1⟩ |
| Fredkin | 3 | `qc.cswap(c,a,b)` | SWAP(a,b) si control=\|1⟩ |

---

## checkpoint_m03.py

```python
"""
Checkpoint Módulo 3 — Puertas Cuánticas y Lectura de Circuitos
Ejecutar con: python checkpoint_m03.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.quantum_info import Operator, process_fidelity
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

def es_unitaria(M, tol=1e-9):
    return np.allclose(M.conj().T @ M, np.eye(len(M)), atol=tol)

print("=" * 55)
print("Checkpoint M03 — Puertas y Circuitos")
print("=" * 55)

# ── Test 1: Matrices de Pauli ────────────────────────────
for gate_name in ['x', 'y', 'z']:
    qc = QuantumCircuit(1); getattr(qc, gate_name)(0)
    M = Operator(qc).data
    assert es_unitaria(M), f"FALLO: puerta {gate_name} no es unitaria"
    assert np.allclose(M @ M, np.eye(2), atol=1e-9), f"FALLO: {gate_name}² ≠ I"
print("✓ Test 1: Puertas de Pauli son unitarias e involutivas")

# ── Test 2: H = H† (Hadamard es hermitiana y unitaria) ───
qcH = QuantumCircuit(1); qcH.h(0)
MH = Operator(qcH).data
assert np.allclose(MH, MH.conj().T, atol=1e-9), "FALLO: H no es hermitiana"
assert es_unitaria(MH), "FALLO: H no es unitaria"
print("✓ Test 2: Hadamard es hermitiana y unitaria")

# ── Test 3: S² = Z ───────────────────────────────────────
qcSS = QuantumCircuit(1); qcSS.s(0); qcSS.s(0)
qcZ  = QuantumCircuit(1); qcZ.z(0)
f_SS_Z = process_fidelity(Operator(qcSS), Operator(qcZ))
assert f_SS_Z > 0.9999, f"FALLO: S²≠Z, fidelidad={f_SS_Z:.6f}"
print("✓ Test 3: S·S = Z (verificado con fidelidad de proceso)")

# ── Test 4: T⁴ = Z ───────────────────────────────────────
qcT4 = QuantumCircuit(1)
for _ in range(4): qcT4.t(0)
f_T4_Z = process_fidelity(Operator(qcT4), Operator(qcZ))
assert f_T4_Z > 0.9999, f"FALLO: T⁴≠Z, fidelidad={f_T4_Z:.6f}"
print("✓ Test 4: T⁴ = Z")

# ── Test 5: SWAP = 3×CNOT ────────────────────────────────
qcSwap = QuantumCircuit(2); qcSwap.swap(0, 1)
qc3cx  = QuantumCircuit(2); qc3cx.cx(0,1); qc3cx.cx(1,0); qc3cx.cx(0,1)
f_swap = process_fidelity(Operator(qcSwap), Operator(qc3cx))
assert f_swap > 0.9999, f"FALLO: SWAP≠3×CNOT, fidelidad={f_swap:.6f}"
print("✓ Test 5: SWAP = 3×CNOT")

# ── Test 6: CZ = H·CNOT·H ────────────────────────────────
qcCZ   = QuantumCircuit(2); qcCZ.cz(0, 1)
qcCZ_d = QuantumCircuit(2); qcCZ_d.h(1); qcCZ_d.cx(0,1); qcCZ_d.h(1)
f_cz = process_fidelity(Operator(qcCZ), Operator(qcCZ_d))
assert f_cz > 0.9999, f"FALLO: CZ≠H·CNOT·H, fidelidad={f_cz:.6f}"
print("✓ Test 6: CZ = H·CNOT·H")

# ── Test 7: Tabla de verdad de Toffoli ───────────────────
toffoli_truth = {
    (0,0,0): (0,0,0),
    (0,0,1): (0,0,1),
    (0,1,0): (0,1,0),
    (0,1,1): (0,1,1),
    (1,0,0): (1,0,0),
    (1,0,1): (1,0,1),
    (1,1,0): (1,1,1),   # ← aplica X al target cuando c0=c1=1
    (1,1,1): (1,1,0),
}
for (c0, c1, t), (ec0, ec1, et) in toffoli_truth.items():
    qc = QuantumCircuit(3)
    if c0: qc.x(0)
    if c1: qc.x(1)
    if t:  qc.x(2)
    qc.ccx(0, 1, 2)
    sv_r = sv(qc)
    # Índice del estado esperado: en Qiskit q2 es bit más significativo
    idx_esp = (ec0) | (ec1 << 1) | (et << 2)   # q0 = bit 0, q1 = bit 1, q2 = bit 2
    assert abs(sv_r[idx_esp])**2 > 0.999, \
        f"FALLO: Toffoli({c0},{c1},{t}) → esperado {(ec0,ec1,et)}"
print("✓ Test 7: Tabla de verdad Toffoli completa (8 casos)")

# ── Test 8: Circuito paramétrico y binding ───────────────
from qiskit.circuit import Parameter
theta = Parameter('θ')
qcP = QuantumCircuit(1); qcP.ry(theta, 0)
qcP_b = qcP.assign_parameters({theta: np.pi})
sv_Ry_pi = sv(qcP_b)
# Ry(π)|0⟩ = |1⟩
assert abs(sv_Ry_pi[1])**2 > 0.999, f"FALLO: Ry(π)|0⟩ debe ser |1⟩"
print("✓ Test 8: Circuito paramétrico → Ry(π)|0⟩ = |1⟩")

# ── Test 9: Circuit inverse ──────────────────────────────
qcOrig = QuantumCircuit(2); qcOrig.h(0); qcOrig.cx(0,1); qcOrig.rz(np.pi/4, 1)
qcFull = QuantumCircuit(2)
qcFull.compose(qcOrig, inplace=True)
qcFull.compose(qcOrig.inverse(), inplace=True)
sv_id = sv(qcFull)
assert np.allclose(sv_id, [1,0,0,0], atol=1e-6), "FALLO: U·U† ≠ |00⟩"
print("✓ Test 9: U · U† = I (circuito inverso)")

# ── Test 10: Análisis de circuito ────────────────────────
from qiskit.circuit.library import QFT
qft4 = QFT(4)
assert qft4.num_qubits == 4, "FALLO: QFT-4 debe tener 4 qubits"
assert qft4.depth() > 0, "FALLO: QFT-4 debe tener profundidad > 0"
print(f"✓ Test 10: QFT-4 correcta (depth={qft4.depth()}, "
      f"gates={qft4.size()})")

print("\n" + "=" * 55)
print("✅ Todos los tests del M03 pasaron correctamente")
print("=" * 55)
```
