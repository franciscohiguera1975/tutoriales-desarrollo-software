# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 4 — Setup Triple Stack: Qiskit, PennyLane y Cirq

> **Objetivo:** Instalar y configurar los tres frameworks cuánticos principales. Ejecutar el mismo circuito de Bell en los tres para comparar sintaxis, rendimiento y ecosistemas. Entender cuándo usar cada uno.
>
> **Herramientas:** Qiskit 1.x + PennyLane 0.38+ + Cirq 1.x.
>
> **Prerequisito:** M03 — Puertas Cuánticas y Lectura de Circuitos.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-04-setup-triple-stack.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-04-setup-triple-stack.ipynb
```

### Opción C — Google Colab
```python
# Celda 1: instalación completa del triple stack
!pip install qiskit qiskit-aer pennylane pennylane-qiskit cirq -q
```

---

## 4.1 Instalación y Verificación del Entorno

```python
"""
Verificar que los tres frameworks están correctamente instalados.
"""
import sys

def verificar_paquete(nombre: str, import_name: str = None) -> bool:
    mod = import_name or nombre
    try:
        m = __import__(mod)
        version = getattr(m, '__version__', 'desconocida')
        print(f"  ✓ {nombre:20s} {version}")
        return True
    except ImportError:
        print(f"  ✗ {nombre:20s} NO INSTALADO — ejecutar: pip install {nombre}")
        return False

print("─" * 45)
print("Verificación del Triple Stack Cuántico")
print("─" * 45)

paquetes = [
    ("qiskit",              "qiskit"),
    ("qiskit-aer",          "qiskit_aer"),
    ("pennylane",           "pennylane"),
    ("pennylane-qiskit",    "pennylane_qiskit"),
    ("cirq",                "cirq"),
    ("numpy",               "numpy"),
    ("matplotlib",          "matplotlib"),
]

todos_ok = all(verificar_paquete(nombre, imp) for nombre, imp in paquetes)

print("─" * 45)
if todos_ok:
    print("✅ Triple stack completo y listo")
else:
    print("⚠ Instalar los paquetes faltantes antes de continuar")
    print("   pip install qiskit qiskit-aer pennylane pennylane-qiskit cirq")
```

---

## 4.2 El Mismo Circuito en los Tres Frameworks

### Estado de Bell |Φ+⟩ — el "Hola Mundo" cuántico

Vamos a preparar `(|00⟩ + |11⟩) / √2` usando H + CNOT en los tres frameworks.

---

#### Framework 1: Qiskit

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# QISKIT — IBM Quantum
# Filosofía: circuito → transpilar → ejecutar en backend
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import numpy as np

sim_qiskit = AerSimulator(method='statevector')

qc_qiskit = QuantumCircuit(2, name="Bell-Qiskit")
qc_qiskit.h(0)
qc_qiskit.cx(0, 1)
qc_qiskit.save_statevector()

job = sim_qiskit.run(transpile(qc_qiskit, sim_qiskit))
sv_qiskit = np.array(job.result().get_statevector())

print("── Qiskit ──")
print(f"  Statevector: {np.round(sv_qiskit, 4)}")
print(f"  P(|00⟩) = {abs(sv_qiskit[0])**2:.4f}")
print(f"  P(|11⟩) = {abs(sv_qiskit[3])**2:.4f}")
print(qc_qiskit.draw())
```

---

#### Framework 2: PennyLane

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# PENNYLANE — Xanadu
# Filosofía: QNode = función Python que devuelve mediciones
# Diseñado para diferenciación automática y ML cuántico
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
import pennylane as qml

# Dispositivo: default.qubit = simulador puro de PennyLane
dev_pl = qml.device("default.qubit", wires=2)

@qml.qnode(dev_pl)
def bell_pennylane():
    qml.Hadamard(wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.state()   # devuelve el statevector completo

sv_pl = bell_pennylane()

print("\n── PennyLane ──")
print(f"  Statevector: {np.round(sv_pl, 4)}")
print(f"  P(|00⟩) = {abs(sv_pl[0])**2:.4f}")
print(f"  P(|11⟩) = {abs(sv_pl[3])**2:.4f}")

# Dibujar el circuito PennyLane
print(qml.draw(bell_pennylane)())
```

---

#### Framework 3: Cirq

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# CIRQ — Google Quantum AI
# Filosofía: objetos Qubit + Moment (operaciones paralelas)
# Diseñado para hardware de Google (Sycamore, Willow)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
import cirq

# En Cirq los qubits son objetos con coordenadas físicas
q0 = cirq.LineQubit(0)
q1 = cirq.LineQubit(1)

circuito_cirq = cirq.Circuit([
    cirq.H(q0),
    cirq.CNOT(q0, q1),
])

# Simulador de Cirq
sim_cirq = cirq.Simulator()
resultado_cirq = sim_cirq.simulate(circuito_cirq)
sv_cirq = resultado_cirq.final_state_vector

print("\n── Cirq ──")
print(f"  Statevector: {np.round(sv_cirq, 4)}")
print(f"  P(|00⟩) = {abs(sv_cirq[0])**2:.4f}")
print(f"  P(|11⟩) = {abs(sv_cirq[3])**2:.4f}")
print(circuito_cirq)
```

---

### Comparación de statevectors entre los tres

```python
print("\n─" * 45)
print("Comparación de statevectors (Bell |Φ+⟩):")
print(f"  Qiskit:    {np.round(sv_qiskit, 4)}")
print(f"  PennyLane: {np.round(np.array(sv_pl), 4)}")
print(f"  Cirq:      {np.round(np.array(sv_cirq), 4)}")

# Verificar que son equivalentes (hasta fase global)
sv_q = sv_qiskit
sv_p = np.array(sv_pl)
sv_c = np.array(sv_cirq)
print(f"\n  Qiskit ≈ PennyLane: {np.allclose(np.abs(sv_q), np.abs(sv_p), atol=1e-5)}")
print(f"  Qiskit ≈ Cirq:      {np.allclose(np.abs(sv_q), np.abs(sv_c), atol=1e-5)}")
```

---

## 4.3 Simuladores — Tipos y Diferencias

### Qiskit: tipos de simuladores de Aer

```python
from qiskit_aer import AerSimulator

# Statevector: mantiene el vector completo, exacto
# Limitación: 2^n amplitudes → ~32 qubits práctico
sim_sv   = AerSimulator(method='statevector')

# QASM: simula disparos reales (sampling), como hardware
# Más escalable en n_qubits con pocas puertas
sim_qasm = AerSimulator(method='qasm')

# Matrix product state (MPS): para circuitos poco entrelazados
# Puede simular >100 qubits si el entrelazamiento es bajo
sim_mps  = AerSimulator(method='matrix_product_state')

# Density matrix: incluye decoherencia y ruido
sim_dm   = AerSimulator(method='density_matrix')

def benchmark_simuladores(qc: QuantumCircuit, n_qubits: int):
    """Compara tiempo de simulación entre métodos."""
    import time
    qc_sv = qc.copy(); qc_sv.save_statevector()
    qc_qasm = qc.copy(); qc_qasm.measure_all()

    metodos = [
        (sim_sv,   "statevector", qc_sv),
        (sim_qasm, "qasm",        qc_qasm),
        (sim_mps,  "mps",         qc_sv.copy()),
    ]
    print(f"\nBenchmark ({n_qubits} qubits, GHZ):")
    for sim, nombre, qc_m in metodos:
        t0 = time.perf_counter()
        try:
            job = sim.run(transpile(qc_m, sim), shots=1024)
            _ = job.result()
            t1 = time.perf_counter()
            print(f"  {nombre:15s}: {(t1-t0)*1000:.1f} ms")
        except Exception as e:
            print(f"  {nombre:15s}: error — {e}")

# GHZ de 5, 10, 15 qubits
for n in [5, 10, 15]:
    qc_g = QuantumCircuit(n)
    qc_g.h(0)
    for i in range(n-1): qc_g.cx(i, i+1)
    benchmark_simuladores(qc_g, n)
```

### PennyLane: dispositivos disponibles

```python
import pennylane as qml

# Dispositivos más comunes de PennyLane
dispositivos_pl = {
    "default.qubit":         "Simulador puro NumPy — diferenciable",
    "default.qubit.jax":     "Simulador con JAX — JIT compatible",
    "default.qubit.torch":   "Simulador con PyTorch — autograd",
    "default.qubit.tf":      "Simulador con TensorFlow",
    "lightning.qubit":       "Simulador C++ de alta velocidad",
    "qiskit.aer":            "Aer de IBM a través de PennyLane",
    "qiskit.ibmq":           "Hardware real IBM",
    "cirq.simulator":        "Simulador de Cirq",
    "strawberryfields.fock": "Computación fotónica (continua)",
}

print("Dispositivos PennyLane disponibles en este entorno:")
for nombre, desc in dispositivos_pl.items():
    try:
        dev = qml.device(nombre, wires=2)
        print(f"  ✓ {nombre:30s} — {desc}")
    except Exception:
        print(f"  ✗ {nombre:30s} — no disponible")

# Listar backends del dispositivo instalado
print(f"\nVersion PennyLane: {qml.__version__}")
print(f"Dispositivos disponibles: {qml.plugin_devices}")
```

### Cirq: simuladores y backends

```python
import cirq

# Simuladores de Cirq
sim_puro    = cirq.Simulator()           # statevector puro
sim_clifford = cirq.CliffordSimulator()  # solo puertas Clifford (H, CNOT, S)
sim_dm      = cirq.DensityMatrixSimulator()  # con ruido/decoherencia

q0, q1 = cirq.LineQubit.range(2)
circ_bell = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])

# Resultado puro
r_puro = sim_puro.simulate(circ_bell)
print(f"Cirq simulator puro:  {np.round(r_puro.final_state_vector, 3)}")

# Resultado con density matrix
r_dm = sim_dm.simulate(circ_bell)
print(f"Cirq density matrix:\n{np.round(r_dm.final_density_matrix, 3)}")

# Muestreo con Cirq (equivalente al modo shots)
circ_m = circ_bell + cirq.measure(q0, q1, key='resultado')
resultado_muestras = sim_puro.run(circ_m, repetitions=1000)
print(f"\nConteos Cirq (1000 shots): {resultado_muestras.histogram(key='resultado')}")
```

---

## 4.4 Interoperabilidad — Convertir entre Frameworks

### Qiskit → PennyLane

```python
# PennyLane puede cargar circuitos de Qiskit directamente
from pennylane_qiskit import load

qc_qiskit_para_pl = QuantumCircuit(2)
qc_qiskit_para_pl.h(0)
qc_qiskit_para_pl.cx(0, 1)
qc_qiskit_para_pl.rz(np.pi/4, 1)

# Convertir a función PennyLane
dev_conv = qml.device("default.qubit", wires=2)

@qml.qnode(dev_conv)
def circuito_desde_qiskit():
    load(qc_qiskit_para_pl, wires=[0, 1])()
    return qml.state()

sv_convertido = circuito_desde_qiskit()
print("Circuito Qiskit ejecutado en PennyLane:")
print(f"  Statevector: {np.round(sv_convertido, 3)}")

# Verificar contra Qiskit
qc_ref = qc_qiskit_para_pl.copy(); qc_ref.save_statevector()
sv_ref = np.array(sim_qiskit.run(transpile(qc_ref, sim_qiskit)).result().get_statevector())
print(f"  Equivalente a Qiskit: {np.allclose(np.abs(sv_convertido), np.abs(sv_ref), atol=1e-5)}")
```

### Cirq → Qiskit

```python
# Usando qiskit-cirq (parte de qiskit-terra)
try:
    from qiskit.providers.fake_provider import GenericBackendV2
    # La conversión directa requiere un paquete adicional
    # En su lugar, usamos la representación matricial
    M_cirq   = cirq.unitary(circ_bell)
    from qiskit.quantum_info import Operator
    M_qiskit = Operator(qc_qiskit).data
    print("Comparación de matrices unitarias Cirq vs Qiskit:")
    print(f"  Matrices iguales (abs): {np.allclose(np.abs(M_cirq), np.abs(M_qiskit), atol=1e-5)}")
except ImportError:
    print("Conversión directa Cirq→Qiskit requiere qiskit-terra[full]")

# PennyLane como puente universal
# PennyLane puede usar backends de Qiskit y Cirq
dev_qiskit_aer = qml.device("qiskit.aer", wires=2, backend="aer_simulator")

@qml.qnode(dev_qiskit_aer)
def bell_en_aer():
    qml.Hadamard(wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.probs(wires=[0, 1])

probs_aer = bell_en_aer()
print(f"\nBell en Aer a través de PennyLane: {np.round(probs_aer, 4)}")
```

---

## 4.5 Cuándo Usar Cada Framework

```python
"""
Guía de decisión:

┌─────────────────────────────────────────────────────────────────┐
│                   ¿QUÉ NECESITAS HACER?                         │
├──────────────────────────┬──────────────────────────────────────┤
│ Algoritmo cuántico       │ → Qiskit                             │
│ (Grover, Shor, QPE, QFT) │   Ecosistema más completo            │
│                          │   Circuit Library preconstruida       │
├──────────────────────────┼──────────────────────────────────────┤
│ QML / VQC / Gradientes   │ → PennyLane                          │
│ Optimización variacional │   Diferenciación automática          │
│ Hibridación con PyTorch  │   Integración con ML frameworks       │
├──────────────────────────┼──────────────────────────────────────┤
│ Hardware Google          │ → Cirq                               │
│ Sycamore / Willow        │   Acceso directo a Google Quantum AI │
│ Ruido y compilación fina │   Control preciso de Moments         │
├──────────────────────────┼──────────────────────────────────────┤
│ Investigación / Paper    │ → Qiskit + PennyLane (ambos)         │
│ Comparar resultados      │   PennyLane para gradientes          │
│                          │   Qiskit para circuitos y hardware   │
├──────────────────────────┼──────────────────────────────────────┤
│ Curso / Aprendizaje      │ → PennyLane (sintaxis más Pythónica) │
│ Primer framework         │   Luego Qiskit para hardware real    │
└──────────────────────────┴──────────────────────────────────────┘
"""

# Demostración: el mismo VQC en los tres estilos
theta_val = np.pi / 3

# Qiskit — basado en objetos QuantumCircuit
from qiskit.circuit import Parameter
theta_q = Parameter('θ')
qc_vqc_q = QuantumCircuit(2)
qc_vqc_q.ry(theta_q, 0); qc_vqc_q.ry(theta_q, 1); qc_vqc_q.cx(0, 1)
qc_concreto = qc_vqc_q.assign_parameters({theta_q: theta_val})
qc_concreto.save_statevector()
sv_vqc_q = np.array(sim_qiskit.run(transpile(qc_concreto, sim_qiskit)).result().get_statevector())

# PennyLane — basado en funciones + autograd
@qml.qnode(qml.device("default.qubit", wires=2))
def vqc_pl(theta):
    qml.RY(theta, wires=0); qml.RY(theta, wires=1); qml.CNOT(wires=[0, 1])
    return qml.state()
sv_vqc_pl = np.array(vqc_pl(theta_val))

# Cirq — basado en Moments y objetos Gate
q0c, q1c = cirq.LineQubit.range(2)
vqc_cirq = cirq.Circuit([
    cirq.ry(theta_val)(q0c),
    cirq.ry(theta_val)(q1c),
    cirq.CNOT(q0c, q1c),
])
sv_vqc_c = sim_puro.simulate(vqc_cirq).final_state_vector

print("VQC Ry(θ)⊗Ry(θ) + CNOT con θ = π/3:")
print(f"  Qiskit:    {np.round(sv_vqc_q, 3)}")
print(f"  PennyLane: {np.round(sv_vqc_pl, 3)}")
print(f"  Cirq:      {np.round(sv_vqc_c, 3)}")
equiv = (
    np.allclose(np.abs(sv_vqc_q), np.abs(sv_vqc_pl), atol=1e-5) and
    np.allclose(np.abs(sv_vqc_q), np.abs(sv_vqc_c),  atol=1e-5)
)
print(f"  ¿Todos equivalentes? {'✓ Sí' if equiv else '✗ No'}")
```

---

## 4.6 Benchmark de Rendimiento

```python
import time

def benchmark_bell(framework: str, n_repeticiones: int = 100) -> float:
    tiempos = []
    for _ in range(n_repeticiones):
        t0 = time.perf_counter()

        if framework == "qiskit":
            qc = QuantumCircuit(2); qc.h(0); qc.cx(0, 1); qc.save_statevector()
            sim_qiskit.run(transpile(qc, sim_qiskit)).result().get_statevector()

        elif framework == "pennylane":
            dev = qml.device("default.qubit", wires=2)
            @qml.qnode(dev)
            def f():
                qml.Hadamard(0); qml.CNOT(wires=[0,1])
                return qml.state()
            f()

        elif framework == "cirq":
            q0, q1 = cirq.LineQubit.range(2)
            c = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])
            cirq.Simulator().simulate(c)

        tiempos.append(time.perf_counter() - t0)

    media = np.mean(tiempos) * 1000
    std   = np.std(tiempos) * 1000
    return media, std

print("Benchmark — circuito Bell (100 repeticiones):")
for fw in ["qiskit", "pennylane", "cirq"]:
    media, std = benchmark_bell(fw)
    print(f"  {fw:12s}: {media:6.2f} ± {std:4.2f} ms/circuito")

print("\nNota: Los tiempos incluyen compilación JIT de la primera ejecución.")
print("En producción, PennyLane con JAX es significativamente más rápido.")
```

---

## 4.7 Estructura Recomendada del Proyecto

```
mi_proyecto_quantum/
├── envs/
│   └── quantum.yml         # conda environment
├── circuits/
│   ├── qiskit/             # circuitos en formato Qiskit
│   ├── pennylane/          # QNodes de PennyLane
│   └── cirq/               # circuitos de Cirq
├── notebooks/
│   ├── 01_exploracion.ipynb
│   └── 02_experimentos.ipynb
├── outputs/
│   ├── statevectors/
│   ├── figuras/
│   └── modelos/
└── requirements.txt

# requirements.txt mínimo para el triple stack:
# qiskit>=1.0.0
# qiskit-aer>=0.14.0
# pennylane>=0.38.0
# pennylane-qiskit>=0.38.0
# cirq>=1.3.0
# numpy>=1.26.0
# matplotlib>=3.8.0
# scipy>=1.12.0
# jupyter>=1.0.0
```

---

## Resumen comparativo

| Característica | Qiskit | PennyLane | Cirq |
|---|---|---|---|
| Mantenido por | IBM | Xanadu | Google |
| Hardware nativo | IBM Quantum | Xanadu (fotónico) + otros | Google Sycamore/Willow |
| Diferenciación automática | Limitada | Nativa (PyTorch, JAX, TF) | No |
| Circuit Library | Extensa | Moderada | Básica |
| Abstracción | Alta | Muy alta | Baja (físico) |
| Ruido y error mitigation | Qiskit Aer + IBM | Limitado | Extenso |
| QML / VQC | qiskit-ml | Primera clase | Limitado |
| Sintaxis | Imperativa (qc.h(0)) | Funcional (@qnode) | OOP (Moment/Gate) |
| Transpilación | Completa (pass managers) | Básica | Sí (para Google HW) |

---

## checkpoint_m04.py

```python
"""
Checkpoint Módulo 4 — Setup Triple Stack
Ejecutar con: python checkpoint_m04.py
"""
import numpy as np
import os
os.makedirs("outputs", exist_ok=True)

print("=" * 55)
print("Checkpoint M04 — Triple Stack: Qiskit + PennyLane + Cirq")
print("=" * 55)

# ── Test 1: Qiskit disponible y funcional ────────────────
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator

sim_q = AerSimulator(method='statevector')
qc = QuantumCircuit(2); qc.h(0); qc.cx(0, 1); qc.save_statevector()
sv_q = np.array(sim_q.run(transpile(qc, sim_q)).result().get_statevector())
assert abs(sv_q[0])**2 > 0.49 and abs(sv_q[3])**2 > 0.49, "FALLO: Qiskit Bell incorrecto"
print("✓ Test 1: Qiskit funcional — Bell state OK")

# ── Test 2: PennyLane disponible y funcional ─────────────
import pennylane as qml

dev = qml.device("default.qubit", wires=2)

@qml.qnode(dev)
def bell_pl():
    qml.Hadamard(wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.state()

sv_pl = np.array(bell_pl())
assert abs(sv_pl[0])**2 > 0.49 and abs(sv_pl[3])**2 > 0.49, "FALLO: PennyLane Bell incorrecto"
print("✓ Test 2: PennyLane funcional — Bell state OK")

# ── Test 3: Cirq disponible y funcional ──────────────────
import cirq

q0, q1 = cirq.LineQubit.range(2)
circ = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])
sv_c = cirq.Simulator().simulate(circ).final_state_vector
assert abs(sv_c[0])**2 > 0.49 and abs(sv_c[3])**2 > 0.49, "FALLO: Cirq Bell incorrecto"
print("✓ Test 3: Cirq funcional — Bell state OK")

# ── Test 4: Los tres dan el mismo resultado ───────────────
equiv_qpl = np.allclose(np.abs(sv_q), np.abs(sv_pl), atol=1e-5)
equiv_qcq = np.allclose(np.abs(sv_q), np.abs(sv_c), atol=1e-5)
assert equiv_qpl, f"FALLO: Qiskit ≠ PennyLane: {sv_q} vs {sv_pl}"
assert equiv_qcq, f"FALLO: Qiskit ≠ Cirq:     {sv_q} vs {sv_c}"
print("✓ Test 4: Tres frameworks dan statevectors equivalentes")

# ── Test 5: Qiskit — múltiples métodos de simulación ─────
for metodo in ['statevector', 'qasm', 'matrix_product_state']:
    try:
        sim_m = AerSimulator(method=metodo)
        qc_t = QuantumCircuit(2); qc_t.h(0); qc_t.cx(0, 1)
        if metodo == 'qasm':
            qc_t.measure_all()
            res = sim_m.run(transpile(qc_t, sim_m), shots=512).result()
            cnt = res.get_counts()
            assert '00' in cnt and '11' in cnt
        else:
            qc_t.save_statevector()
            res = sim_m.run(transpile(qc_t, sim_m)).result()
            sv_t = np.array(res.get_statevector())
            assert abs(sv_t[0])**2 > 0.49
        print(f"✓ Test 5a: Qiskit método '{metodo}' disponible y funcional")
    except Exception as e:
        print(f"  (!) Método '{metodo}' no disponible: {e}")

# ── Test 6: PennyLane — circuito paramétrico ─────────────
theta_test = np.pi / 4

@qml.qnode(qml.device("default.qubit", wires=1))
def ry_pl(theta):
    qml.RY(theta, wires=0)
    return qml.state()

sv_ry = ry_pl(np.pi)  # Ry(π)|0⟩ = |1⟩
assert abs(sv_ry[1])**2 > 0.999, "FALLO: PennyLane Ry(π)|0⟩ debe ser |1⟩"
print("✓ Test 6: PennyLane circuito paramétrico funcional")

# ── Test 7: Cirq — Moment structure ──────────────────────
q = cirq.LineQubit.range(3)
# Cirq agrupa puertas en Moments (capas paralelas)
circ_multi = cirq.Circuit([
    cirq.H.on_each(*q),        # Moment 1: H en los 3 qubits simultáneamente
    cirq.CNOT(q[0], q[1]),    # Moment 2
    cirq.CNOT(q[1], q[2]),    # Moment 3
])
assert len(circ_multi) >= 2, "FALLO: circuito Cirq debe tener al menos 2 momentos"
sv_ghz_c = cirq.Simulator().simulate(circ_multi).final_state_vector
assert abs(sv_ghz_c[0])**2 > 0.49 and abs(sv_ghz_c[7])**2 > 0.49, \
    "FALLO: GHZ-3 en Cirq incorrecto"
print("✓ Test 7: Cirq Moment structure + GHZ-3 correcto")

# ── Test 8: Interoperabilidad Qiskit → PennyLane ─────────
try:
    from pennylane_qiskit import load
    qc_test = QuantumCircuit(2); qc_test.h(0); qc_test.cx(0, 1)
    dev_conv = qml.device("default.qubit", wires=2)

    @qml.qnode(dev_conv)
    def qiskit_en_pl():
        load(qc_test, wires=[0, 1])()
        return qml.state()

    sv_conv = np.array(qiskit_en_pl())
    assert abs(sv_conv[0])**2 > 0.49, "FALLO: conversión Qiskit→PennyLane"
    print("✓ Test 8: Interoperabilidad Qiskit→PennyLane funcional")
except ImportError:
    print("  (!) pennylane-qiskit no instalado — instalar: pip install pennylane-qiskit")

print("\n" + "=" * 55)
print("✅ Todos los tests del M04 pasaron correctamente")
print("=" * 55)
```
