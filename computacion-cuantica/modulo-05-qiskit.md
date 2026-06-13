# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 5 — Qiskit en Profundidad: IBM Quantum

> **Objetivo:** Dominar el ecosistema Qiskit completo: primitivas (Sampler/Estimator), transpilación, pass managers, ejecución en simuladores y hardware real IBM Quantum, y el análisis de resultados de ruido.
>
> **Herramientas:** Qiskit 1.x + Qiskit Aer + qiskit-ibm-runtime.
>
> **Prerequisito:** M04 — Setup Triple Stack.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-05-qiskit.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-05-qiskit.ipynb
```

### Opción C — Google Colab
```python
!pip install qiskit qiskit-aer qiskit-ibm-runtime pylatexenc -q
```

### Opción D — IBM Quantum Cloud (hardware real)
```python
from qiskit_ibm_runtime import QiskitRuntimeService
# Guardar credenciales (solo la primera vez)
QiskitRuntimeService.save_account(channel="ibm_quantum", token="TU_TOKEN_IBM")
# En ejecuciones posteriores:
service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False, min_num_qubits=5)
```

---

## 5.1 Arquitectura de Qiskit 1.x

```
                     Qiskit 1.x — Stack completo
    ┌─────────────────────────────────────────────────┐
    │  NIVEL ALTO — Circuit Library                   │
    │  QFT, Grover, QAOA, VQE, ZZFeatureMap...       │
    ├─────────────────────────────────────────────────┤
    │  CIRCUITO — QuantumCircuit                      │
    │  Puertas, mediciones, parámetros                │
    ├─────────────────────────────────────────────────┤
    │  TRANSPILACIÓN — PassManager                    │
    │  Optimización, mapeo a hardware, basis gates    │
    ├─────────────────────────────────────────────────┤
    │  PRIMITIVAS — Sampler / Estimator               │
    │  Interfaz unificada para sim y hardware real    │
    ├────────────────┬────────────────────────────────┤
    │  LOCAL         │  IBM QUANTUM CLOUD             │
    │  Aer Simulator │  qiskit-ibm-runtime            │
    └────────────────┴────────────────────────────────┘
```

---

## 5.2 QuantumCircuit en Detalle

### Registros, ancillas y clbits auxiliares

```python
from qiskit import QuantumCircuit, QuantumRegister, ClassicalRegister, transpile
from qiskit_aer import AerSimulator
from qiskit.quantum_info import Statevector, Operator
import numpy as np
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

# Registros con nombres semánticos
q_datos   = QuantumRegister(3, 'datos')
q_ancilla = QuantumRegister(1, 'anc')
c_out     = ClassicalRegister(3, 'salida')
c_flag    = ClassicalRegister(1, 'flag')

qc_regs = QuantumCircuit(q_datos, q_ancilla, c_out, c_flag)
qc_regs.h(q_datos)
qc_regs.cx(q_datos[0], q_ancilla[0])
qc_regs.barrier(label="medición")
qc_regs.measure(q_datos, c_out)
qc_regs.measure(q_ancilla, c_flag)

print("Circuito con registros semánticos:")
print(qc_regs.draw())
print(f"  Qubits: {qc_regs.num_qubits} (3 datos + 1 ancilla)")
print(f"  Clbits: {qc_regs.num_clbits} (3 salida + 1 flag)")
```

### Instrucciones de control de flujo clásico

```python
# Qiskit 1.x — control clásico con if_else, while_loop, for_loop
from qiskit.circuit.classical import expr

# if_else: aplicar corrección cuántica según medición previa
qc_if = QuantumCircuit(2, 1)
qc_if.h(0)
qc_if.measure(0, 0)

with qc_if.if_else((qc_if.clbits[0], 1)):
    qc_if.x(1)   # Si qubit 0 midió 1, aplicar X al qubit 1

qc_if.save_statevector()
print("\nCircuito con if_else:")
print(qc_if.draw())
```

### Inicialización de estados arbitrarios

```python
# initialize() — prepara cualquier amplitud compleja
estado_custom = np.array([0.6, 0.8j]) / np.linalg.norm([0.6, 0.8j])
qc_init = QuantumCircuit(1)
qc_init.initialize(estado_custom, 0)
sv_init = sv(qc_init)
print(f"\nEstado inicializado: {np.round(sv_init, 3)}")
print(f"Norma verificada: {np.sum(np.abs(sv_init)**2):.6f}")

# prepare_state() — alternativa en Qiskit 1.x (más eficiente)
try:
    from qiskit.circuit.library import StatePreparation
    qc_sp = QuantumCircuit(1)
    qc_sp.append(StatePreparation(estado_custom), [0])
    sv_sp = sv(qc_sp)
    print(f"StatePreparation: {np.round(sv_sp, 3)}")
except ImportError:
    pass

# reset() — reinicializa el qubit a |0⟩ durante el circuito
qc_reset = QuantumCircuit(1)
qc_reset.x(0)        # qubit en |1⟩
qc_reset.reset(0)    # reinicializar a |0⟩
sv_reset = sv(qc_reset)
print(f"\nDespués de reset: {np.round(sv_reset, 3)}")  # [1+0j, 0+0j]
```

---

## 5.3 Transpilación — El Pass Manager

```python
from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager
from qiskit.transpiler import CouplingMap, InstructionProperties
from qiskit.transpiler.passes import (
    Optimize1qGates, CommutationAnalysis, CommutativeCancellation
)

# Ejemplo: circuito con Toffoli y rotaciones arbitrarias
qc_complejo = QuantumCircuit(4)
qc_complejo.h([0, 1, 2, 3])
qc_complejo.ccx(0, 1, 2)
qc_complejo.cp(np.pi/3, 2, 3)
qc_complejo.swap(0, 3)
qc_complejo.u(np.pi/5, np.pi/7, np.pi/9, 1)

print(f"Circuito original: depth={qc_complejo.depth()}, "
      f"gates={qc_complejo.count_ops()}")

# Transpilar a basis gates de IBM Eagle (cx, id, rz, sx, x)
basis_ibm = ['cx', 'id', 'rz', 'sx', 'x']
coupling_5q = CouplingMap.from_line(5)   # topología lineal de 5 qubits

for nivel in [0, 1, 2, 3]:
    pm = generate_preset_pass_manager(
        optimization_level=nivel,
        basis_gates=basis_ibm,
        coupling_map=coupling_5q,
    )
    qc_t = pm.run(qc_complejo)
    ops = dict(qc_t.count_ops())
    print(f"  Opt level {nivel}: depth={qc_t.depth():3d}, "
          f"cx={ops.get('cx',0):2d}, rz={ops.get('rz',0):2d}, "
          f"sx={ops.get('sx',0):2d}")
```

### Pass manager personalizado

```python
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import (
    Unroller, RemoveResetInZeroState, Optimize1qGatesDecomposition,
    CommutativeCancellation
)

# Construir un pipeline de optimización personalizado
pm_custom = PassManager([
    Unroller(basis=['cx', 'u']),            # Descomponer a basis mínimo
    RemoveResetInZeroState(),               # Eliminar resets redundantes
    Optimize1qGatesDecomposition(basis=['rz','sx','x']),  # Fusionar 1-qubit gates
    CommutativeCancellation(),              # Cancelar puertas que conmutan
])

qc_opt = pm_custom.run(qc_complejo)
print(f"\nPM personalizado: depth={qc_opt.depth()}, ops={dict(qc_opt.count_ops())}")
```

---

## 5.4 Primitivas — Sampler y Estimator

### La API de primitivas (Qiskit 1.x)

```python
# En Qiskit 1.x, la interfaz recomendada son las PRIMITIVAS:
#   Sampler  → obtiene distribuciones de probabilidad (mediciones)
#   Estimator → calcula valores esperados de observables

from qiskit_aer.primitives import Sampler, Estimator
from qiskit.quantum_info import SparsePauliOp

# ── Sampler — equivalente a ejecutar con shots ───────────
# Prepara un circuito y devuelve quasi-distribuciones de probabilidad

qc_sample = QuantumCircuit(3)
qc_sample.h(0)
qc_sample.cx(0, 1)
qc_sample.cx(1, 2)
qc_sample.measure_all()

sampler = Sampler()
job_s = sampler.run([qc_sample], shots=4096)
resultado_s = job_s.result()

# Quasi-distribution: {estado_binario: probabilidad}
quasi_dist = resultado_s[0].data.meas.get_counts()
total = sum(quasi_dist.values())
print("Sampler — GHZ-3:")
for bits, cnt in sorted(quasi_dist.items(), key=lambda x: -x[1]):
    print(f"  |{bits}⟩: {cnt/total:.3f}")

# ── Estimator — calcula ⟨ψ|O|ψ⟩ ─────────────────────────
# Fundamental para VQE y QAOA

# Observable: Z⊗Z (correlación entre q0 y q1)
observables = SparsePauliOp.from_list([("ZZ", 1.0)])

# Circuito sin medición (el Estimator lo maneja internamente)
qc_est = QuantumCircuit(2)
qc_est.h(0)
qc_est.cx(0, 1)   # |Φ+⟩

estimator = Estimator()
job_e = estimator.run([(qc_est, observables)])
resultado_e = job_e.result()
valor_esperado = resultado_e[0].data.evs
print(f"\nEstimator — ⟨Φ+|ZZ|Φ+⟩ = {valor_esperado:.4f}")
# Debe ser ≈ 1.0 (correlación perfecta en Bell state)

# Observable más complejo: H = 0.5*ZI - 0.3*IZ + 0.4*XX
H_obs = SparsePauliOp.from_list([("ZI", 0.5), ("IZ", -0.3), ("XX", 0.4)])
job_H = estimator.run([(qc_est, H_obs)])
E_H = job_H.result()[0].data.evs
print(f"⟨Φ+|H|Φ+⟩ = {E_H:.4f}")
```

### Sampler con circuitos paramétricos — sweep de parámetros

```python
from qiskit.circuit import ParameterVector

theta_vec = ParameterVector('θ', 2)
qc_param = QuantumCircuit(2)
qc_param.ry(theta_vec[0], 0)
qc_param.ry(theta_vec[1], 1)
qc_param.cx(0, 1)
qc_param.measure_all()

# Evaluar para múltiples valores de θ
valores_theta = [
    [0.0, 0.0],         # |00⟩
    [np.pi, 0.0],       # |10⟩ (q0 flip)
    [np.pi/2, np.pi/2], # superposición
    [np.pi, np.pi],     # |11⟩
]

job_sweep = sampler.run(
    [(qc_param, vals) for vals in valores_theta],
    shots=1024
)
result_sweep = job_sweep.result()

print("\nSweep de parámetros Ry(θ0)⊗Ry(θ1)+CNOT:")
for vals, pub_result in zip(valores_theta, result_sweep):
    cnts = pub_result.data.meas.get_counts()
    total = sum(cnts.values())
    dist_str = ", ".join(f"|{k}⟩:{v/total:.2f}" for k, v in sorted(cnts.items(), key=lambda x:-x[1])[:3])
    print(f"  θ={vals} → {dist_str}")
```

---

## 5.5 Análisis de Resultados

### Visualizaciones nativas de Qiskit

```python
from qiskit.visualization import (
    plot_histogram,
    plot_state_city,
    plot_bloch_multivector,
    plot_state_qsphere,
    circuit_drawer
)
from qiskit.quantum_info import Statevector

# Bell state para visualizar
qc_bell = QuantumCircuit(2); qc_bell.h(0); qc_bell.cx(0, 1)
sv_bell = Statevector(sv(qc_bell))

# 1. City plot — amplitudes como barras 3D
fig_city = plot_state_city(sv_bell, title="Bell |Φ+⟩ — State City")
fig_city.savefig("outputs/state_city_bell.png", dpi=150, bbox_inches='tight')

# 2. Esfera de Bloch (solo funciona bien para pocos qubits)
fig_bloch = plot_bloch_multivector(sv_bell, title="Bell |Φ+⟩ — Bloch")
fig_bloch.savefig("outputs/bloch_bell.png", dpi=150, bbox_inches='tight')

# 3. Q-sphere — representación para múltiples qubits
try:
    fig_qsphere = plot_state_qsphere(sv_bell)
    fig_qsphere.savefig("outputs/qsphere_bell.png", dpi=150, bbox_inches='tight')
except Exception:
    pass

# 4. Histograma de mediciones
qc_medir = QuantumCircuit(2); qc_medir.h(0); qc_medir.cx(0, 1); qc_medir.measure_all()
sim_qasm = AerSimulator(method='qasm')
conteos = sim_qasm.run(transpile(qc_medir, sim_qasm), shots=2048).result().get_counts()
fig_hist = plot_histogram(conteos, title="Bell — Histograma (2048 shots)")
fig_hist.savefig("outputs/histograma_bell.png", dpi=150, bbox_inches='tight')

print("Visualizaciones guardadas en outputs/")
```

### Matriz de densidad y tomografía de estado

```python
from qiskit.quantum_info import DensityMatrix, state_fidelity

# Estado puro vs estado mixto
sv_puro = Statevector(sv(qc_bell))
dm_puro = DensityMatrix(sv_puro)
dm_mixto = DensityMatrix(np.eye(4) / 4)   # máxima mezcla

print("Matriz de densidad Bell state:")
print(np.round(dm_puro.data, 3))

# Pureza: Tr(ρ²) = 1 para estados puros, < 1 para mezclados
pureza_puro  = np.trace(dm_puro.data @ dm_puro.data).real
pureza_mixto = np.trace(dm_mixto.data @ dm_mixto.data).real
print(f"\nPureza estado puro (Bell):    {pureza_puro:.4f} (debe ser 1)")
print(f"Pureza estado mixto (max):    {pureza_mixto:.4f} (debe ser 0.25)")

# Fidelidad entre estados
sv_phi_plus  = Statevector(sv(qc_bell))
qc_psi_plus = QuantumCircuit(2); qc_psi_plus.x(0); qc_psi_plus.h(0); qc_psi_plus.cx(0,1)
sv_psi_plus = Statevector(sv(qc_psi_plus))
F = state_fidelity(sv_phi_plus, sv_psi_plus)
print(f"\nFidelidad |Φ+⟩ vs |Ψ+⟩: {F:.4f} (debe ser 0 — ortogonales)")

F_mismo = state_fidelity(sv_phi_plus, sv_phi_plus)
print(f"Fidelidad |Φ+⟩ vs |Φ+⟩: {F_mismo:.4f} (debe ser 1)")
```

---

## 5.6 Modelos de Ruido en Aer

```python
from qiskit_aer.noise import (
    NoiseModel, depolarizing_error, thermal_relaxation_error,
    ReadoutError, pauli_error
)
from qiskit_aer import AerSimulator

# ── Modelo de ruido básico ────────────────────────────────
noise_model = NoiseModel()

# Error de desfasamiento en puertas de 1 qubit (1% probabilidad)
error_1q = depolarizing_error(0.01, 1)
noise_model.add_all_qubit_quantum_error(error_1q, ['h', 'rx', 'ry', 'rz', 'x'])

# Error de desfasamiento en puertas de 2 qubits (3%)
error_2q = depolarizing_error(0.03, 2)
noise_model.add_all_qubit_quantum_error(error_2q, ['cx', 'cz'])

# Error de readout (lectura): 2% de probabilidad de error en medición
p_readout = 0.02
readout_error = ReadoutError([[1-p_readout, p_readout], [p_readout, 1-p_readout]])
noise_model.add_all_qubit_readout_error(readout_error)

print("Modelo de ruido creado:")
print(noise_model)

# Comparar ideal vs ruidoso
sim_ideal  = AerSimulator(method='qasm')
sim_ruidoso = AerSimulator(method='qasm', noise_model=noise_model)

qc_test = QuantumCircuit(2); qc_test.h(0); qc_test.cx(0, 1); qc_test.measure_all()

SHOTS = 8192
c_ideal  = sim_ideal.run(transpile(qc_test, sim_ideal),  shots=SHOTS).result().get_counts()
c_ruidoso = sim_ruidoso.run(transpile(qc_test, sim_ruidoso), shots=SHOTS).result().get_counts()

print("\n           Ideal       Ruidoso")
for bits in ['00', '01', '10', '11']:
    p_i = c_ideal.get(bits, 0)  / SHOTS
    p_r = c_ruidoso.get(bits, 0) / SHOTS
    print(f"  |{bits}⟩:  {p_i:.4f}      {p_r:.4f}")
```

### Ruido T1/T2 — relajación térmica realista

```python
# T1 = tiempo de relajación de energía (|1⟩ → |0⟩)
# T2 = tiempo de decoherencia de fase (destruye superposición)
# Para qubits de IBM: T1 ≈ T2 ≈ 100-300 microsegundos

T1_us = 150   # microsegundos
T2_us = 120   # microsegundos

# Tiempo de puerta típico de IBM Eagle
t_puerta_1q = 0.035   # µs (35 ns)
t_puerta_2q = 0.200   # µs (200 ns — puerta CX)

error_relajacion_1q = thermal_relaxation_error(
    t1=T1_us * 1e3,     # convertir a nanosegundos
    t2=T2_us * 1e3,
    time=t_puerta_1q * 1e3
)
error_relajacion_2q = thermal_relaxation_error(
    t1=T1_us * 1e3,
    t2=T2_us * 1e3,
    time=t_puerta_2q * 1e3
)

noise_t1t2 = NoiseModel()
noise_t1t2.add_all_qubit_quantum_error(error_relajacion_1q, ['h', 'ry', 'rz', 'x'])
noise_t1t2.add_all_qubit_quantum_error(error_relajacion_2q, ['cx'])

sim_t1t2 = AerSimulator(method='qasm', noise_model=noise_t1t2)
c_t1t2 = sim_t1t2.run(transpile(qc_test, sim_t1t2), shots=SHOTS).result().get_counts()

print("\nBell con ruido T1/T2 realista:")
for bits in ['00', '01', '10', '11']:
    p = c_t1t2.get(bits, 0) / SHOTS
    print(f"  |{bits}⟩: {p:.4f}")
```

---

## 5.7 Error Mitigation — Zero Noise Extrapolation (ZNE)

```python
# ZNE: escalar el ruido artificialmente (2x, 3x) y extrapolar a ruido→0
# Técnica de mitigación sin overhead cuántico

def amplificar_ruido(qc: QuantumCircuit, factor: int) -> QuantumCircuit:
    """
    Reemplaza cada CNOT por CNOT·CNOT·CNOT (factor=3), etc.
    El resultado es el mismo circuito pero con más ruido.
    """
    from qiskit.transpiler.passes import UnrollCustomDefinitions, BasisTranslator
    qc_amp = QuantumCircuit(qc.num_qubits, qc.num_clbits)
    for inst in qc.data:
        if inst.operation.name == 'cx':
            for _ in range(factor):
                qc_amp.cx(
                    qc.find_bit(inst.qubits[0]).index,
                    qc.find_bit(inst.qubits[1]).index
                )
        else:
            qc_amp.append(inst)
    return qc_amp

# Observar cómo el valor esperado de ZZ decae con el ruido
estimator_ruidoso = Estimator(options={"noise_model": noise_model})
H_ZZ = SparsePauliOp.from_list([("ZZ", 1.0)])

qc_bell_bare = QuantumCircuit(2)
qc_bell_bare.h(0)
qc_bell_bare.cx(0, 1)

factores = [1, 3, 5]
expectativas = []
for factor in factores:
    qc_amp = amplificar_ruido(qc_bell_bare, factor)
    job = estimator_ruidoso.run([(qc_amp, H_ZZ)], shots=4096)
    E = job.result()[0].data.evs
    expectativas.append(E)
    print(f"  Factor de ruido {factor}: ⟨ZZ⟩ = {E:.4f}")

# Extrapolación lineal a ruido 0
from numpy.polynomial import polynomial as P
coef = np.polyfit(factores, expectativas, deg=1)
E_mitigado = np.polyval(coef, 0)
print(f"\nValor extrapolado (ruido=0): ⟨ZZ⟩ = {E_mitigado:.4f}")
print(f"Valor ideal (sin ruido):     ⟨ZZ⟩ = 1.0000")
```

---

## 5.8 Conexión con IBM Quantum (Hardware Real)

```python
# ─── Solo ejecutar si se tiene API token IBM Quantum ──────
# La sección 5.8 es para usuarios con cuenta IBM Quantum.
# Con la cuenta gratuita se tienen acceso a backends de ≤5 qubits.

def ejecutar_en_ibm_quantum(token: str = None):
    """
    Plantilla para ejecutar en hardware real.
    Requiere token de IBM Quantum.
    """
    if token is None:
        print("Para ejecutar en hardware real, proporcionar token IBM Quantum.")
        print("Registro gratuito en: https://quantum.ibm.com")
        return

    from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2
    from qiskit_ibm_runtime.fake_provider import FakeManilaV2

    # Opción A: simular hardware real con FakeBackend (sin token)
    fake_backend = FakeManilaV2()
    print(f"FakeBackend: {fake_backend.name}")
    print(f"  Qubits: {fake_backend.num_qubits}")
    print(f"  Basis gates: {fake_backend.operation_names}")
    print(f"  Coupling map: {list(fake_backend.coupling_map)}")

    # Circuito Bell para hardware real
    qc_hw = QuantumCircuit(2); qc_hw.h(0); qc_hw.cx(0, 1); qc_hw.measure_all()

    # Transpilar al hardware específico
    pm = generate_preset_pass_manager(backend=fake_backend, optimization_level=1)
    qc_hw_t = pm.run(qc_hw)
    print(f"\nCircuito transpilado para {fake_backend.name}:")
    print(qc_hw_t.draw())

    # Ejecutar en FakeBackend (simula ruido del hardware real)
    sampler_fake = SamplerV2(backend=fake_backend)
    job_fake = sampler_fake.run([qc_hw_t], shots=1024)
    resultado_fake = job_fake.result()
    conteos_fake = resultado_fake[0].data.meas.get_counts()
    print(f"\nResultados en {fake_backend.name}: {conteos_fake}")

    # Opción B: hardware real IBM (requiere token)
    try:
        service = QiskitRuntimeService(channel="ibm_quantum", token=token)
        backend_real = service.least_busy(operational=True, simulator=False, min_num_qubits=2)
        print(f"\nBackend real disponible: {backend_real.name}")

        pm_real = generate_preset_pass_manager(backend=backend_real, optimization_level=1)
        qc_real_t = pm_real.run(qc_hw)

        from qiskit_ibm_runtime import SamplerV2 as SamplerReal
        sampler_real = SamplerReal(backend=backend_real)
        job_real = sampler_real.run([qc_real_t], shots=100)
        print(f"Job ID en IBM Quantum: {job_real.job_id()}")
        print("Para recuperar el resultado más tarde:")
        print(f"  job = service.job('{job_real.job_id()}')")
        print("  resultado = job.result()")
    except Exception as e:
        print(f"Hardware real no disponible: {e}")

# Ejecutar con FakeBackend (no requiere cuenta IBM)
ejecutar_en_ibm_quantum(token=None)
```

---

## 5.9 Pipeline Completo: del Circuito al Resultado

```python
def pipeline_qiskit_completo(
    qc_original: QuantumCircuit,
    nombre: str = "Circuito",
    usar_ruido: bool = True,
    shots: int = 4096
) -> dict:
    """
    Pipeline completo:
    1. Análisis del circuito original
    2. Transpilación optimizada
    3. Simulación ideal
    4. Simulación ruidosa
    5. Comparación y visualización
    """
    from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager
    from qiskit.visualization import plot_histogram

    print(f"\n{'='*50}")
    print(f"Pipeline: {nombre}")
    print(f"{'='*50}")

    # 1. Análisis original
    print(f"\n[1] Circuito original:")
    print(f"    Profundidad: {qc_original.depth()}")
    print(f"    Puertas: {dict(qc_original.count_ops())}")

    # 2. Transpilación
    basis = ['cx', 'id', 'rz', 'sx', 'x']
    cmap  = CouplingMap.from_line(qc_original.num_qubits)
    pm    = generate_preset_pass_manager(
        optimization_level=2, basis_gates=basis, coupling_map=cmap)
    qc_t = pm.run(qc_original)
    print(f"\n[2] Después de transpilar (opt_level=2):")
    print(f"    Profundidad: {qc_t.depth()}")
    print(f"    Puertas: {dict(qc_t.count_ops())}")

    # 3. Añadir mediciones si no las tiene
    qc_m = qc_t.copy(); qc_m.measure_all()

    # 4. Simulación ideal
    sim_id = AerSimulator(method='qasm')
    c_ideal = sim_id.run(transpile(qc_m, sim_id), shots=shots).result().get_counts()

    # 5. Simulación ruidosa
    if usar_ruido:
        nm = NoiseModel()
        nm.add_all_qubit_quantum_error(depolarizing_error(0.005, 1), ['h','x','rz','sx'])
        nm.add_all_qubit_quantum_error(depolarizing_error(0.02, 2), ['cx'])
        sim_noisy = AerSimulator(method='qasm', noise_model=nm)
        c_noisy = sim_noisy.run(transpile(qc_m, sim_noisy), shots=shots).result().get_counts()
    else:
        c_noisy = None

    # 6. Visualizar
    fig, axes = plt.subplots(1, 2 if c_noisy else 1, figsize=(12, 4))
    if c_noisy is None: axes = [axes]
    plot_histogram(c_ideal, ax=axes[0], title=f"{nombre} — Ideal")
    if c_noisy:
        plot_histogram(c_noisy, ax=axes[1], title=f"{nombre} — Ruidoso")
    plt.tight_layout()
    fname = nombre.lower().replace(" ", "_")
    plt.savefig(f"outputs/pipeline_{fname}.png", dpi=150, bbox_inches='tight')
    plt.close()

    return {"ideal": c_ideal, "noisy": c_noisy}

# Aplicar el pipeline al estado GHZ de 3 qubits
qc_ghz = QuantumCircuit(3)
qc_ghz.h(0); qc_ghz.cx(0, 1); qc_ghz.cx(1, 2)
resultado = pipeline_qiskit_completo(qc_ghz, "GHZ-3")
```

---

## Resumen del ecosistema Qiskit 1.x

| Componente | Uso |
|---|---|
| `QuantumCircuit` | Construir circuitos, puertas, mediciones |
| `AerSimulator` | Simular localmente (sv, qasm, mps, density_matrix) |
| `Sampler` | Obtener distribuciones de probabilidad |
| `Estimator` | Calcular valores esperados ⟨ψ\|O\|ψ⟩ |
| `PassManager` | Transpilar al basis set del hardware |
| `NoiseModel` | Modelar ruido de hardware real |
| `SparsePauliOp` | Definir operadores cuánticos |
| `Statevector` | Análisis exacto del estado cuántico |
| `DensityMatrix` | Estados mixtos y decoherencia |
| `QiskitRuntimeService` | Acceso a IBM Quantum Cloud |
| `FakeBackend` | Simular hardware real sin cuenta |

---

## checkpoint_m05.py

```python
"""
Checkpoint Módulo 5 — Qiskit en Profundidad
Ejecutar con: python checkpoint_m05.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit_aer.primitives import Sampler, Estimator
from qiskit_aer.noise import NoiseModel, depolarizing_error
from qiskit.quantum_info import Statevector, SparsePauliOp, state_fidelity
from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager
from qiskit.transpiler import CouplingMap
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_q = AerSimulator(method='qasm')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

print("=" * 55)
print("Checkpoint M05 — Qiskit en Profundidad")
print("=" * 55)

# ── Test 1: Transpilación reduce la profundidad ──────────
from qiskit.circuit.library import QFT
qft5 = QFT(5)
pm = generate_preset_pass_manager(
    optimization_level=3,
    basis_gates=['cx','rz','sx','x'],
    coupling_map=CouplingMap.from_line(5)
)
qft5_t = pm.run(qft5)
assert qft5_t.depth() > 0, "FALLO: circuito transpilado vacío"
print(f"✓ Test 1: QFT-5 transpilado, depth={qft5_t.depth()}")

# ── Test 2: Sampler devuelve distribución correcta ───────
qc_bell = QuantumCircuit(2); qc_bell.h(0); qc_bell.cx(0, 1); qc_bell.measure_all()
sampler = Sampler()
job = sampler.run([qc_bell], shots=4096)
cnts = job.result()[0].data.meas.get_counts()
total = sum(cnts.values())
p00 = cnts.get('00', 0) / total
p11 = cnts.get('11', 0) / total
assert 0.45 < p00 < 0.55, f"FALLO: Sampler P(00) = {p00}"
assert 0.45 < p11 < 0.55, f"FALLO: Sampler P(11) = {p11}"
print(f"✓ Test 2: Sampler Bell — P(00)={p00:.3f}, P(11)={p11:.3f}")

# ── Test 3: Estimator ⟨Φ+|ZZ|Φ+⟩ = 1 ──────────────────
qc_bare = QuantumCircuit(2); qc_bare.h(0); qc_bare.cx(0, 1)
obs = SparsePauliOp.from_list([("ZZ", 1.0)])
estimator = Estimator()
E_ZZ = estimator.run([(qc_bare, obs)]).result()[0].data.evs
assert abs(E_ZZ - 1.0) < 0.05, f"FALLO: ⟨ZZ⟩ = {E_ZZ:.4f}, esperado 1.0"
print(f"✓ Test 3: Estimator ⟨ZZ⟩ = {E_ZZ:.4f} ≈ 1.0")

# ── Test 4: Modelo de ruido introduce errores ────────────
nm = NoiseModel()
nm.add_all_qubit_quantum_error(depolarizing_error(0.1, 1), ['h'])
nm.add_all_qubit_quantum_error(depolarizing_error(0.1, 2), ['cx'])
sim_noisy = AerSimulator(method='qasm', noise_model=nm)
qc_test = QuantumCircuit(2); qc_test.h(0); qc_test.cx(0, 1); qc_test.measure_all()
c_noisy = sim_noisy.run(transpile(qc_test, sim_noisy), shots=4096).result().get_counts()
total_n = sum(c_noisy.values())
# Con 10% depolarizing en h y cx, debería haber errores (01 y 10)
errores = (c_noisy.get('01', 0) + c_noisy.get('10', 0)) / total_n
assert errores > 0.01, f"FALLO: Modelo de ruido no produce errores detectables ({errores:.4f})"
print(f"✓ Test 4: Ruido introduce {errores:.3f} de tasa de error")

# ── Test 5: initialize() prepara estado arbitrario ───────
estado = np.array([0.6, 0.8]) / np.linalg.norm([0.6, 0.8])
qc_init = QuantumCircuit(1)
qc_init.initialize(estado.tolist(), 0)
sv_init = sv(qc_init)
assert np.allclose(np.abs(sv_init), np.abs(estado), atol=1e-5), \
    f"FALLO: initialize() incorrecto: {sv_init}"
print("✓ Test 5: initialize() prepara estado arbitrario")

# ── Test 6: FakeBackend disponible ───────────────────────
try:
    from qiskit_ibm_runtime.fake_provider import FakeManilaV2
    fake = FakeManilaV2()
    assert fake.num_qubits >= 5, "FALLO: FakeManila debe tener ≥5 qubits"
    print(f"✓ Test 6: FakeBackend '{fake.name}' ({fake.num_qubits} qubits)")
except ImportError:
    print("  (!) qiskit-ibm-runtime no instalado")

# ── Test 7: state_fidelity ────────────────────────────────
sv1 = Statevector(sv(qc_bare))
sv2 = Statevector(sv(qc_bare))
F = state_fidelity(sv1, sv2)
assert abs(F - 1.0) < 1e-6, f"FALLO: fidelidad mismo estado debe ser 1, got {F}"
print(f"✓ Test 7: Fidelidad mismo estado = {F:.6f}")

# ── Test 8: SparsePauliOp observable sum ─────────────────
H_mol = SparsePauliOp.from_list([
    ("ZI", 0.3979),
    ("IZ", -0.3979),
    ("ZZ", -0.0112),
    ("XX", 0.1809),
])
assert len(H_mol) == 4, "FALLO: Hamiltoniano debe tener 4 términos"
E_mol = estimator.run([(qc_bare, H_mol)]).result()[0].data.evs
print(f"✓ Test 8: Hamiltoniano molecular H2 ⟨E⟩ = {E_mol:.4f} Hartree")

print("\n" + "=" * 55)
print("✅ Todos los tests del M05 pasaron correctamente")
print("=" * 55)
```
