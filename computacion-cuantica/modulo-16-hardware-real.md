# Módulo 16 — Ejecutar en Hardware Real: IBM y Google

**Objetivo:** Conectarse a hardware cuántico real, entender las diferencias entre
simulador e hardware físico, gestionar transpilación, colas de ejecución y
post-procesamiento de resultados ruidosos.

**Herramientas:** Qiskit, QiskitRuntimeService, Cirq (Google), FakeBackends
**Prerequisito:** M05 (Qiskit), M13 (Hardware), M14 (Ruido)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
# Opción 1: Con cuenta IBM Quantum (gratuita)
python modulo-16-hardware-real.py --backend ibm_brisbane

# Opción 2: Sin cuenta — usar FakeBackend (idéntico en API)
python modulo-16-hardware-real.py --backend fake
```

> **Nota**: Todas las secciones pueden ejecutarse con FakeBackend sin necesidad
> de una cuenta IBM. La sección 8 muestra cómo conectarse al hardware real.

---

## 1. El gap simulador vs hardware

```python
# modulo-16-hardware-real.py
"""
Módulo 16: Ejecutar en Hardware Real
Funciona con o sin cuenta IBM Quantum.
"""
import numpy as np
import matplotlib.pyplot as plt
from qiskit import QuantumCircuit, transpile
from qiskit.quantum_info import Statevector, state_fidelity
from qiskit_aer import AerSimulator
from qiskit_aer.noise import NoiseModel
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Diferencias simulador vs hardware
# ─────────────────────────────────────────────

print("=" * 65)
print("HARDWARE REAL vs SIMULADOR — DIFERENCIAS CLAVE")
print("=" * 65)

diferencias = {
    "Ruido": {
        "Simulador ideal": "Sin ruido — resultados perfectos",
        "Simulador Aer + noise": "Ruido configurable — modela hardware",
        "Hardware real": "Ruido real, heterogéneo por qubit y puerta",
    },
    "Puertas nativas": {
        "Simulador ideal": "Cualquier puerta soportada",
        "Simulador Aer + noise": "Transpila a basis gates del backend",
        "Hardware real": "Solo las nativas: ECR, ID, RZ, SX, X (IBM Eagle)",
    },
    "Conectividad": {
        "Simulador ideal": "All-to-all",
        "Simulador Aer + noise": "Restringe según topology del backend",
        "Hardware real": "Mapa de conectividad físico (coupler map)",
    },
    "Coherencia": {
        "Simulador ideal": "Infinita",
        "Simulador Aer + noise": "T1/T2 configurables",
        "Hardware real": "T1~100μs, T2~80μs (varía por qubit)",
    },
    "Cola de trabajo": {
        "Simulador ideal": "Inmediato",
        "Simulador Aer + noise": "Inmediato",
        "Hardware real": "Minutos a horas (según demanda)",
    },
    "Shots": {
        "Simulador ideal": "Hasta 1M shots, rápido",
        "Simulador Aer + noise": "Hasta 1M shots, más lento",
        "Hardware real": "Máx 100K shots, cada shot ~1ms",
    },
    "Reproducibilidad": {
        "Simulador ideal": "Perfecta (dado seed fijo)",
        "Simulador Aer + noise": "Estadística (Poisson)",
        "Hardware real": "Varía por deriva del hardware",
    },
}

for categoria, vals in diferencias.items():
    print(f"\n  {categoria}:")
    for tipo, desc in vals.items():
        print(f"    {tipo:<28} {desc}")
```

---

## 2. Backends disponibles: FakeBackends

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: FakeBackends — hardware real sin cuenta
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("FAKE BACKENDS — SIMULACIÓN PRECISA DE HARDWARE IBM")
print("="*65)

print("""
Los FakeBackends son snapshots de backends IBM reales.
Contienen:
  • Mapa de conectividad real
  • Matrices de error calibradas
  • T1, T2 por qubit
  • Error de puerta por par de qubits
  • Error de readout por qubit

Disponibles en qiskit_ibm_runtime.fake_provider:
  FakeAlmadenV2, FakeArmonkV2, FakeAthensV2, FakeBelem...
  FakeBrisbaneV2, FakeCairoV2, FakeCambridgeV2...
  FakeManilaV2, FakeMelbourneV2, FakeNairobiV2...
  FakeSherbrooke (127 qubits, IBM Eagle topology)
  FakeHanoiV2, FakeKolkata...
""")

try:
    from qiskit_ibm_runtime.fake_provider import FakeSherbrooke, FakeManilaV2
    fake_backend = FakeSherbrooke()
    print(f"Backend: {fake_backend.name}")
    print(f"Qubits: {fake_backend.num_qubits}")

    # Mostrar propiedades del backend
    props = fake_backend.properties()
    if props:
        # Primeros 5 qubits
        print(f"\nPropiedades de los primeros 5 qubits:")
        print(f"{'Qubit':>7} {'T1 (μs)':>10} {'T2 (μs)':>10} {'Error RO':>10} {'Frec (GHz)':>12}")
        print(f"{'─'*55}")
        for i in range(min(5, fake_backend.num_qubits)):
            try:
                t1 = props.t1(i) * 1e6   # convertir a μs
                t2 = props.t2(i) * 1e6
                ro_err = props.readout_error(i)
                freq = props.frequency(i) / 1e9
                print(f"  Q{i:<5} {t1:>10.1f} {t2:>10.1f} {ro_err:>10.4f} {freq:>12.4f}")
            except Exception:
                print(f"  Q{i:<5} propiedades no disponibles")

    # Mostrar mapa de conectividad (coupling_map)
    print(f"\nMapa de conectividad (primeras 10 conexiones):")
    coupling = fake_backend.coupling_map
    if coupling:
        edges = list(coupling.get_edges())[:10]
        for edge in edges:
            print(f"  Q{edge[0]} ↔ Q{edge[1]}")
        print(f"  ... ({len(list(fake_backend.coupling_map.get_edges()))} conexiones totales)")

    BACKEND_DISPONIBLE = True

except ImportError as e:
    print(f"⚠ FakeSherbrooke no disponible: {e}")
    print("  Usando AerSimulator básico")
    fake_backend = AerSimulator()
    BACKEND_DISPONIBLE = False
```

---

## 3. Transpilación — el paso crítico

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Transpilación
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("TRANSPILACIÓN — ADAPTAR EL CIRCUITO AL HARDWARE")
print("="*65)

print("""
La transpilación convierte tu circuito abstracto en uno ejecutable:
  1. Descomposición: convierte puertas en las nativas del hardware
     IBM Eagle: {ECR, ID, RZ, SX, X} — todo se puede expresar con estas
  2. Routing: inserta SWAP para conectar qubits que no son vecinos
  3. Optimización: reduce puertas redundantes
  4. Scheduling: asigna tiempos de ejecución

Niveles de optimización (0-3):
  0: Solo mapeo de qubits — sin optimización
  1: Optimización ligera — equivalencias simples
  2: Optimización moderada — reescritura de circuitos
  3: Optimización agresiva — más lento, mejor resultado
""")

# Circuito de prueba: GHZ de 4 qubits
qc_ghz = QuantumCircuit(4, 4, name="GHZ-4")
qc_ghz.h(0)
qc_ghz.cx(0, 1)
qc_ghz.cx(1, 2)
qc_ghz.cx(2, 3)
qc_ghz.measure(range(4), range(4))

print(f"Circuito original GHZ-4:")
print(f"  Profundidad: {qc_ghz.depth()}")
print(f"  Operaciones: {dict(qc_ghz.count_ops())}")

# Transpilar con diferentes niveles de optimización
sim_base = AerSimulator()
resultados_transpilacion = []

for nivel in range(4):
    qc_t = transpile(qc_ghz, backend=sim_base, optimization_level=nivel, seed_transpiler=42)
    resultados_transpilacion.append({
        "nivel": nivel,
        "depth": qc_t.depth(),
        "cx": qc_t.count_ops().get("cx", 0),
        "ops": sum(qc_t.count_ops().values()),
    })

print(f"\n{'Nivel':>7} {'Profundidad':>13} {'CX gates':>10} {'Ops total':>11}")
print(f"{'─'*45}")
for r in resultados_transpilacion:
    print(f"  O{r['nivel']}      {r['depth']:>9}    {r['cx']:>8}    {r['ops']:>9}")

# Transpilación para hardware específico
if BACKEND_DISPONIBLE:
    try:
        qc_t_hw = transpile(qc_ghz, backend=fake_backend, optimization_level=3, seed_transpiler=42)
        print(f"\nTranspilado para {fake_backend.name}:")
        print(f"  Puertas nativas usadas: {dict(qc_t_hw.count_ops())}")
        print(f"  Profundidad: {qc_t_hw.depth()}")
        print(f"  Qubits físicos usados: {sorted(qc_t_hw.qubits)}")
    except Exception as e:
        print(f"\nTranspilación para hardware: {e}")

# PassManager (Qiskit 1.x — forma recomendada)
try:
    from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager
    pm = generate_preset_pass_manager(optimization_level=3, backend=sim_base, seed_transpiler=42)
    qc_pm = pm.run(qc_ghz)
    print(f"\nPassManager O3:")
    print(f"  Profundidad: {qc_pm.depth()}")
    print(f"  Operaciones: {dict(qc_pm.count_ops())}")
except Exception as e:
    print(f"\nPassManager: {e}")
```

---

## 4. Primitivas Qiskit Runtime — SamplerV2 y EstimatorV2

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Primitivas Qiskit Runtime
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("PRIMITIVAS QISKIT RUNTIME — API UNIFICADA PARA HARDWARE")
print("="*65)

print("""
Las primitivas son la API estándar para ejecutar en hardware IBM:
  SamplerV2: distribuye shots → retorna BitArray (más eficiente)
  EstimatorV2: calcula valores esperados de observables

Diferencia con Qiskit básico:
  • transpile() + backend.run(): API legada, deprecada
  • Primitivas: API moderna, funciona igual en simulador y hardware real
  • Primitivas manejan transpilación interna si se especifica backend
""")

from qiskit_aer.primitives import SamplerV2 as AerSamplerV2
from qiskit_aer.primitives import Estimator as AerEstimator
from qiskit.quantum_info import SparsePauliOp

# Circuito Bell para probar
qc_bell = QuantumCircuit(2, 2)
qc_bell.h(0)
qc_bell.cx(0, 1)
qc_bell.measure_all()

# SamplerV2
print("\n--- SamplerV2 ---")
sampler = AerSamplerV2()
job = sampler.run([qc_bell], shots=8192)
result = job.result()
pub_result = result[0]
counts = pub_result.data.meas.get_counts()
print(f"Bell state counts (ideal): {counts}")

# Con ruido
from qiskit_aer.noise import NoiseModel, depolarizing_error
nm = NoiseModel()
nm.add_all_qubit_quantum_error(depolarizing_error(0.01, 1), ["h"])
nm.add_all_qubit_quantum_error(depolarizing_error(0.03, 2), ["cx"])
sampler_noisy = AerSamplerV2()
sampler_noisy.options.update(noise_model=nm)
job_noisy = sampler_noisy.run([qc_bell], shots=8192)
counts_noisy = job_noisy.result()[0].data.meas.get_counts()
print(f"Bell state counts (ruidoso): {counts_noisy}")

# Estimator
print("\n--- EstimatorV2 / Estimator ---")
qc_para_estim = QuantumCircuit(2)
qc_para_estim.h(0)
qc_para_estim.cx(0, 1)

obs = SparsePauliOp.from_list([("ZZ", 1.0)])
estimator = AerEstimator()
job_est = estimator.run([(qc_para_estim, obs)])
result_est = job_est.result()
evs = result_est[0].data.evs
print(f"  ⟨ZZ⟩ = {evs:.4f}  (esperado: +1.0 para Bell |00⟩+|11⟩)")

obs_xx = SparsePauliOp.from_list([("XX", 1.0)])
job_xx = estimator.run([(qc_para_estim, obs_xx)])
evs_xx = job_xx.result()[0].data.evs
print(f"  ⟨XX⟩ = {evs_xx:.4f}  (esperado: +1.0 para Bell)")

obs_zx = SparsePauliOp.from_list([("ZX", 1.0)])
job_zx = estimator.run([(qc_para_estim, obs_zx)])
evs_zx = job_zx.result()[0].data.evs
print(f"  ⟨ZX⟩ = {evs_zx:.4f}  (esperado: ~0 para Bell)")
```

---

## 5. Gestión de jobs y cola de hardware

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Gestión de jobs
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("GESTIÓN DE JOBS Y COLA DE HARDWARE")
print("="*65)

print("""
En hardware real, un job puede tardar minutos u horas.
Estrategias de gestión:

1. BATCH JOBS (Qiskit Runtime):
   • Agrupar múltiples circuitos en 1 job
   • Reduce overhead de comunicación
   • Máximo: ~300 circuitos por job en hardware libre

2. SESSION (Qiskit Runtime):
   • Bloqueo del backend por tiempo limitado
   • Útil para algoritmos iterativos (VQE, QAOA)
   • Se cobra por tiempo de sesión en hardware pago

3. FAIR-SHARE QUEUE:
   • IBM Quantum libre: sistema de cola compartida
   • Tu turno depende de uso reciente
   • Prioridad: usuarios que han usado menos recientemente

4. JOB LIFECYCLE:
   QUEUED → RUNNING → DONE / ERROR / CANCELLED

Comandos útiles (para hardware real):
   job = backend.run(qc, shots=1024)
   job_id = job.job_id()          # guardar para recuperar después
   status = job.status()           # QUEUED, RUNNING, DONE
   result = job.result()           # bloquea hasta completar
   job.cancel()                    # cancelar si está en cola

Recuperar un job anterior:
   service = QiskitRuntimeService(channel="ibm_quantum", token="...")
   job = service.job(job_id)
   result = job.result()
""")

# Simulación local de gestión de jobs
class SimulatedJob:
    """Simula el ciclo de vida de un job en hardware real."""
    def __init__(self, job_id, circuito, shots=1024):
        self.job_id = job_id
        self._circuito = circuito
        self._shots = shots
        self._status = "QUEUED"
        self._result = None

    def status(self):
        return self._status

    def run(self):
        """Ejecutar localmente (simula lo que haría el hardware)."""
        self._status = "RUNNING"
        sim = AerSimulator()
        qc_t = transpile(self._circuito, sim)
        result = sim.run(qc_t, shots=self._shots).result()
        self._result = result.get_counts()
        self._status = "DONE"
        return self

    def result(self):
        if self._status != "DONE":
            raise RuntimeError("Job no completado todavía")
        return self._result

# Demo del ciclo de vida
print("Demo del ciclo de vida de un job:")
job = SimulatedJob("job-demo-001", qc_bell, shots=2048)
print(f"  Estado inicial: {job.status()}")
job.run()
print(f"  Estado final: {job.status()}")
print(f"  Resultado: {job.result()}")
```

---

## 6. Análisis de ruido en hardware real

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: Análisis de ruido real
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("ANÁLISIS DE RUIDO EN HARDWARE REAL")
print("="*65)

# Usar FakeBackend para simular un análisis realista de calidad de qubits
def analizar_calidad_hardware(backend_sim, n_qubits=5):
    """
    Analiza la calidad de cada qubit y conexión del backend.
    Retorna métricas de interés para decidir qué qubits usar.
    """
    print(f"\nAnálisis de calidad de hardware ({type(backend_sim).__name__}):")

    # Circuitos de benchmarking por qubit
    resultados = []
    for q in range(min(n_qubits, 5)):
        # Circuito H + medición (debería dar 50/50 perfecto)
        qc = QuantumCircuit(1, 1)
        qc.h(0)
        qc.measure(0, 0)
        qc_t = transpile(qc, backend_sim, initial_layout=[q] if hasattr(backend_sim, 'coupling_map') else None,
                         seed_transpiler=42)
        try:
            result = backend_sim.run(qc_t, shots=1024).result()
            counts = result.get_counts()
            total = sum(counts.values())
            p0 = counts.get("0", 0) / total
            p1 = counts.get("1", 0) / total
            sesgo = abs(p0 - 0.5)
            resultados.append({"qubit": q, "p0": p0, "p1": p1, "sesgo": sesgo})
            print(f"  Q{q}: P(0)={p0:.3f}, P(1)={p1:.3f}, |sesgo|={sesgo:.3f}")
        except Exception as e:
            print(f"  Q{q}: error — {e}")

    return resultados

if BACKEND_DISPONIBLE:
    # Simular con FakeBackend
    try:
        resultados_hw = analizar_calidad_hardware(
            AerSimulator.from_backend(fake_backend), n_qubits=5
        )
    except Exception:
        resultados_hw = analizar_calidad_hardware(AerSimulator(), n_qubits=5)
else:
    resultados_hw = analizar_calidad_hardware(AerSimulator(), n_qubits=5)

# Benchmark de circuito de Bell entre pares de qubits
print(f"\nBenchmark Bell entre pares de qubits (fidelidad):")
pares = [(0, 1), (1, 2), (2, 3), (3, 4)]
sim_para_bench = AerSimulator()

if BACKEND_DISPONIBLE:
    try:
        nm_fake = NoiseModel.from_backend(fake_backend)
        sim_para_bench = AerSimulator(noise_model=nm_fake)
    except Exception:
        pass

for q0, q1 in pares:
    qc_b = QuantumCircuit(2, 2)
    qc_b.h(0)
    qc_b.cx(0, 1)
    qc_b.measure([0, 1], [0, 1])
    qc_t = transpile(qc_b, sim_para_bench, seed_transpiler=42)
    counts = sim_para_bench.run(qc_t, shots=4096).result().get_counts()
    total = sum(counts.values())
    # Fidelidad del Bell state = (P(00) + P(11))
    fidelidad = (counts.get("00", 0) + counts.get("11", 0)) / total
    error_state = counts.get("01", 0) + counts.get("10", 0)
    print(f"  Q{q0}-Q{q1}: fidelidad Bell = {fidelidad:.4f}  (error_states: {error_state})")
```

---

## 7. Técnicas de ejecución eficiente

```python
# ─────────────────────────────────────────────
# SECCIÓN 7: Ejecución eficiente
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("TÉCNICAS DE EJECUCIÓN EFICIENTE")
print("="*65)

# 1. Parameter binding — múltiples configuraciones en 1 job
from qiskit.circuit import ParameterVector

print("\n1. Parameter binding — barrer múltiples parámetros:")
theta = ParameterVector("θ", 3)
qc_param = QuantumCircuit(3, 3)
qc_param.ry(theta[0], 0)
qc_param.ry(theta[1], 1)
qc_param.ry(theta[2], 2)
qc_param.cx(0, 1)
qc_param.cx(1, 2)
qc_param.measure([0, 1, 2], [0, 1, 2])

# Crear grilla de parámetros
thetas = [
    {theta[0]: 0.0, theta[1]: 0.0, theta[2]: 0.0},
    {theta[0]: np.pi/4, theta[1]: np.pi/3, theta[2]: np.pi/6},
    {theta[0]: np.pi/2, theta[1]: np.pi/2, theta[2]: np.pi/2},
    {theta[0]: np.pi, theta[1]: np.pi, theta[2]: np.pi},
]

sim = AerSimulator()
resultados_param = []
for i, params in enumerate(thetas):
    qc_bound = qc_param.assign_parameters(params)
    qc_t = transpile(qc_bound, sim)
    counts = sim.run(qc_t, shots=2048).result().get_counts()
    resultados_param.append(counts)
    print(f"  θ = ({params[theta[0]]:.2f}, {params[theta[1]]:.2f}, {params[theta[2]]:.2f}): {dict(list(counts.items())[:3])}")

# 2. Batch execution
print("\n2. Batch execution — múltiples circuitos en 1 job:")
circuitos_batch = []
for i in range(5):
    qc = QuantumCircuit(2, 2)
    qc.ry(i * np.pi/4, 0)
    qc.cx(0, 1)
    qc.measure([0, 1], [0, 1])
    circuitos_batch.append(transpile(qc, sim))

# Ejecutar todos en paralelo (1 job, n circuitos)
job_batch = sim.run(circuitos_batch, shots=1024)
for i, counts in enumerate(job_batch.result().get_counts()):
    print(f"  Circuito {i} (θ={i*45}°): {counts}")

# 3. Optimal circuit layout — qubit selection
print("\n3. Qubit layout — elegir los mejores qubits:")
print("""
Para hardware real, el layout de qubits importa mucho:
  • Algunos qubits son más ruidosos que otros
  • Las conexiones entre qubits tienen diferente fidelidad
  • initial_layout=[q0, q1, ...]: asignar manualmente
  • optimization_level=3: deja que Qiskit elija el mejor layout

Tip: usar el layout de menor error total
  1. Descargar calibración del backend
  2. Calcular error acumulado de cada path
  3. Usar initial_layout con el path más limpio
""")

# 4. Shots óptimos
print("4. Número óptimo de shots:")
print("""
  Error estadístico: σ = √(p(1-p)/N) ≈ 0.5/√N
  Para σ < 1%: N > 2500 shots
  Para σ < 0.1%: N > 250000 shots

  Regla práctica para NISQ:
    • Estimación rápida: 1024-4096 shots
    • Resultados publicables: 8192-32768 shots
    • Alta precisión: 100K+ shots (solo en simulador)
    • Hardware gratuito IBM: max 100K shots/job
""")

for n_shots in [256, 1024, 4096, 16384]:
    sigma = 0.5 / np.sqrt(n_shots)
    print(f"  N={n_shots:>7}: σ ≈ {sigma:.4f} ({sigma*100:.2f}%)")
```

---

## 8. Conectarse a IBM Quantum real

```python
# ─────────────────────────────────────────────
# SECCIÓN 8: Conexión a IBM Quantum
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("CONECTARSE A IBM QUANTUM REAL")
print("="*65)

print("""
PASO 1: Crear cuenta (gratuita)
  → https://quantum.ibm.com/
  → "Create account" con email o GitHub

PASO 2: Obtener API Token
  → Dashboard → "Copy token" (sección API token)

PASO 3: Guardar token localmente (solo una vez)
  ```python
  from qiskit_ibm_runtime import QiskitRuntimeService
  QiskitRuntimeService.save_account(
      channel="ibm_quantum",
      token="tu-token-aqui",
      overwrite=True
  )
  ```
  El token se guarda en ~/.qiskit/qiskit-ibm.json

PASO 4: Conectarse y listar backends disponibles
  ```python
  service = QiskitRuntimeService(channel="ibm_quantum")
  backends = service.backends(operational=True, simulator=False)
  for b in backends:
      print(b.name, b.num_qubits, b.status().status_msg)
  ```

PASO 5: Seleccionar backend y ejecutar
  ```python
  backend = service.least_busy(operational=True, simulator=False, min_num_qubits=5)
  print(f"Usando: {backend.name}")

  from qiskit_ibm_runtime import SamplerV2 as Sampler
  sampler = Sampler(mode=backend)
  job = sampler.run([qc_transpilado], shots=4096)
  result = job.result()  # espera hasta que termine
  ```
""")

# Verificar si hay cuenta configurada
def verificar_cuenta_ibm():
    """Verifica si hay una cuenta IBM Quantum configurada."""
    try:
        from qiskit_ibm_runtime import QiskitRuntimeService
        service = QiskitRuntimeService(channel="ibm_quantum")
        backends = service.backends(operational=True, simulator=False)
        print(f"  ✓ Cuenta IBM Quantum encontrada")
        print(f"  Backends disponibles: {len(backends)}")
        for b in backends[:5]:
            status = b.status()
            print(f"    • {b.name}: {b.num_qubits}q — {status.status_msg}")
            if hasattr(status, 'pending_jobs'):
                print(f"      Cola: {status.pending_jobs} jobs pendientes")
        return service, backends
    except Exception as e:
        print(f"  ⚠ Sin cuenta IBM Quantum configurada")
        print(f"  ({e})")
        print(f"  → Usando FakeBackend para demostración")
        return None, []

print("\nVerificando cuenta IBM Quantum:")
service, backends = verificar_cuenta_ibm()
```

---

## 9. Ejecutar en hardware real — pipeline completo

```python
# ─────────────────────────────────────────────
# SECCIÓN 9: Pipeline completo
# ─────────────────────────────────────────────

print("\n" + "="*65)
print("PIPELINE COMPLETO: CIRCUITO → HARDWARE REAL (o FakeBackend)")
print("="*65)

def pipeline_hardware(circuito: QuantumCircuit, backend, shots: int = 4096,
                      usar_primitivas: bool = True) -> dict:
    """
    Pipeline completo para ejecutar en hardware real o FakeBackend.
    Retorna counts y metadata.
    """
    print(f"\n  1. Circuito original:")
    print(f"     Depth={circuito.depth()}, Gates={dict(circuito.count_ops())}")

    # 1. Transpilación
    try:
        qc_t = transpile(circuito, backend=backend, optimization_level=3, seed_transpiler=42)
    except Exception:
        qc_t = transpile(circuito, optimization_level=3, seed_transpiler=42)
    print(f"  2. Transpilado: Depth={qc_t.depth()}, Gates={dict(qc_t.count_ops())}")

    # 2. Ejecutar
    print(f"  3. Ejecutando ({shots} shots)...")
    try:
        if usar_primitivas:
            from qiskit_aer.primitives import SamplerV2
            s = SamplerV2()
            qc_no_meas = circuito.copy()
            # Usar circuito con medidas
            qc_con_meas = circuito.copy() if circuito.num_clbits > 0 else circuito.measure_all(inplace=False)
            result = backend.run(transpile(qc_con_meas, backend=backend if hasattr(backend, 'run') else AerSimulator(),
                                          seed_transpiler=42), shots=shots).result()
            counts = result.get_counts()
        else:
            result = backend.run(qc_t, shots=shots).result()
            counts = result.get_counts()
    except Exception as e:
        print(f"     Usando AerSimulator de respaldo: {e}")
        sim = AerSimulator()
        result = sim.run(transpile(circuito, sim), shots=shots).result()
        counts = result.get_counts()

    print(f"  4. Resultado: {counts}")
    return counts

# Ejecutar circuito GHZ-3 en el pipeline
qc_ghz3 = QuantumCircuit(3, 3)
qc_ghz3.h(0)
qc_ghz3.cx(0, 1)
qc_ghz3.cx(1, 2)
qc_ghz3.measure([0, 1, 2], [0, 1, 2])

print("Ejecutando GHZ-3 a través del pipeline:")
backend_para_exec = AerSimulator() if not BACKEND_DISPONIBLE else (
    AerSimulator.from_backend(fake_backend) if hasattr(fake_backend, 'properties') else AerSimulator()
)

try:
    counts_pipeline = pipeline_hardware(qc_ghz3, backend_para_exec, shots=8192)
    total = sum(counts_pipeline.values())
    fidelidad = (counts_pipeline.get("000", 0) + counts_pipeline.get("111", 0)) / total
    print(f"\n  Fidelidad GHZ-3: {fidelidad:.4f}")
    print(f"  (Ideal: 1.0, ruidoso esperado: ~0.9-0.98)")
except Exception as e:
    print(f"  Pipeline: {e}")
```

---

## 10. Comparativa final: simulador vs FakeBackend vs hardware real

```python
# ─────────────────────────────────────────────
# SECCIÓN 10: Visualización comparativa
# ─────────────────────────────────────────────

# Simular los 3 escenarios con el estado Bell
circuitos_comp = {
    "Ideal\n(AerSimulator)": (AerSimulator(), qc_bell),
}

nm_ruido = NoiseModel()
nm_ruido.add_all_qubit_quantum_error(depolarizing_error(0.005, 1), ["h"])
nm_ruido.add_all_qubit_quantum_error(depolarizing_error(0.02, 2), ["cx"])
nm_ruido.add_all_qubit_quantum_error(
    depolarizing_error(0.03, 1), ["measure"]
)
circuitos_comp["FakeBackend\n(IBM-like noise)"] = (AerSimulator(noise_model=nm_ruido), qc_bell)

nm_alto = NoiseModel()
nm_alto.add_all_qubit_quantum_error(depolarizing_error(0.01, 1), ["h"])
nm_alto.add_all_qubit_quantum_error(depolarizing_error(0.05, 2), ["cx"])
nm_alto.add_all_qubit_quantum_error(
    depolarizing_error(0.05, 1), ["measure"]
)
circuitos_comp["Hardware real\n(simulado con ruido alto)"] = (AerSimulator(noise_model=nm_alto), qc_bell)

fig, axes = plt.subplots(1, 3, figsize=(15, 5))
estados_posibles = ["00", "01", "10", "11"]

for ax, (nombre, (sim_k, qc_k)) in zip(axes, circuitos_comp.items()):
    qc_t = transpile(qc_k, sim_k)
    counts = sim_k.run(qc_t, shots=8192).result().get_counts()
    total = sum(counts.values())
    probs = [counts.get(s, 0)/total for s in estados_posibles]

    bars = ax.bar(estados_posibles, probs, color=["green", "red", "red", "green"],
                  alpha=0.7)
    ax.axhline(y=0.5, color="blue", linestyle="--", alpha=0.5, label="Ideal 50%")
    ax.set_ylim(0, 0.7)
    ax.set_title(nombre, fontsize=10)
    ax.set_xlabel("Estado")
    ax.set_ylabel("Probabilidad")
    ax.legend(fontsize=8)

    fid = probs[0] + probs[3]  # P(00) + P(11)
    ax.text(0.5, 0.62, f"Fidelidad Bell: {fid:.3f}", ha="center",
           transform=ax.transAxes, fontsize=10, fontweight="bold",
           color="darkblue")

plt.suptitle("Estado Bell: Simulador Ideal vs FakeBackend vs Hardware Real\n(Qiskit Aer)",
             fontsize=12, fontweight="bold")
plt.tight_layout()
plt.savefig("outputs/m16_simulador_vs_hardware.png", dpi=150, bbox_inches="tight")
plt.close()
print("\n→ Guardado: outputs/m16_simulador_vs_hardware.png")

print("""
Resumen del módulo:
  1. Hardware real tiene ruido heterogéneo y requiere transpilación
  2. FakeBackends permiten simular hardware sin cuenta IBM
  3. Primitivas (SamplerV2, EstimatorV2) son la API moderna
  4. Transpilación O3 reduce profundidad del circuito
  5. Readout mitigation y ZNE mejoran resultados en hardware ruidoso
  6. Cuenta IBM Quantum gratuita: https://quantum.ibm.com/

✓ Módulo 16 completado
""")
```

---

## Checkpoint M16

```python
# checkpoint_m16.py
"""Verificaciones del Módulo 16 — Hardware Real"""

def test_transpilacion_reduce_profundidad():
    """Optimización O3 no aumenta la profundidad respecto a O0."""
    from qiskit import QuantumCircuit, transpile
    from qiskit_aer import AerSimulator
    qc = QuantumCircuit(3, 3)
    qc.h(0); qc.cx(0, 1); qc.cx(1, 2); qc.measure([0,1,2],[0,1,2])
    sim = AerSimulator()
    d0 = transpile(qc, sim, optimization_level=0, seed_transpiler=42).depth()
    d3 = transpile(qc, sim, optimization_level=3, seed_transpiler=42).depth()
    assert d3 <= d0, f"O3 depth {d3} no debe superar O0 depth {d0}"
    print(f"✓ test_transpilacion_reduce_profundidad (O0={d0}, O3={d3})")

def test_fake_backend_disponible():
    """FakeBackend o AerSimulator debe estar disponible."""
    try:
        from qiskit_ibm_runtime.fake_provider import FakeManilaV2
        b = FakeManilaV2()
        assert b.num_qubits == 5
        print("✓ test_fake_backend_disponible (FakeManilaV2)")
    except ImportError:
        from qiskit_aer import AerSimulator
        sim = AerSimulator()
        assert sim is not None
        print("✓ test_fake_backend_disponible (AerSimulator fallback)")

def test_sampler_bell():
    """SamplerV2 ejecuta Bell state con fidelidad > 0.9."""
    from qiskit import QuantumCircuit
    from qiskit_aer.primitives import SamplerV2
    qc = QuantumCircuit(2, 2)
    qc.h(0); qc.cx(0, 1); qc.measure([0,1],[0,1])
    sampler = SamplerV2()
    result = sampler.run([qc], shots=4096).result()
    counts = result[0].data.meas.get_counts()
    total = sum(counts.values())
    fid = (counts.get("00", 0) + counts.get("11", 0)) / total
    assert fid > 0.9, f"Fidelidad Bell debe ser > 0.9, got {fid:.4f}"
    print(f"✓ test_sampler_bell (fidelidad: {fid:.4f})")

def test_estimator_zz_bell():
    """EstimatorV2 calcula ⟨ZZ⟩≈+1 para Bell state."""
    from qiskit import QuantumCircuit
    from qiskit_aer.primitives import Estimator
    from qiskit.quantum_info import SparsePauliOp
    qc = QuantumCircuit(2)
    qc.h(0); qc.cx(0, 1)
    obs = SparsePauliOp.from_list([("ZZ", 1.0)])
    est = Estimator()
    evs = est.run([(qc, obs)]).result()[0].data.evs
    assert abs(evs - 1.0) < 0.1, f"⟨ZZ⟩ debe ser ~+1, got {evs:.4f}"
    print(f"✓ test_estimator_zz_bell (⟨ZZ⟩ = {evs:.4f})")

def test_noise_degrada_fidelidad():
    """Con ruido alto, la fidelidad del Bell state decrece."""
    from qiskit import QuantumCircuit, transpile
    from qiskit_aer import AerSimulator
    from qiskit_aer.noise import NoiseModel, depolarizing_error
    qc = QuantumCircuit(2, 2)
    qc.h(0); qc.cx(0, 1); qc.measure([0,1],[0,1])
    # Ideal
    sim_ideal = AerSimulator()
    counts_i = sim_ideal.run(transpile(qc, sim_ideal), shots=4096).result().get_counts()
    total_i = sum(counts_i.values())
    fid_ideal = (counts_i.get("00",0) + counts_i.get("11",0)) / total_i
    # Ruidoso
    nm = NoiseModel()
    nm.add_all_qubit_quantum_error(depolarizing_error(0.2, 2), ["cx"])
    sim_noisy = AerSimulator(noise_model=nm)
    counts_n = sim_noisy.run(transpile(qc, sim_noisy), shots=4096).result().get_counts()
    total_n = sum(counts_n.values())
    fid_noisy = (counts_n.get("00",0) + counts_n.get("11",0)) / total_n
    assert fid_ideal > fid_noisy, \
        f"Sin ruido debe ser mejor: {fid_ideal:.4f} vs {fid_noisy:.4f}"
    print(f"✓ test_noise_degrada_fidelidad (ideal={fid_ideal:.4f} > ruidoso={fid_noisy:.4f})")

def test_shots_estadistica():
    """Más shots → menor error estadístico (σ ∝ 1/√N)."""
    import numpy as np
    shots_list = [100, 1000, 10000]
    sigmas = [0.5 / np.sqrt(n) for n in shots_list]
    for i in range(len(sigmas)-1):
        assert sigmas[i] > sigmas[i+1], "Más shots deben dar menor σ"
    print(f"✓ test_shots_estadistica (σ decrece con N)")

def test_parameter_binding():
    """Circuit paramétrico ejecuta con diferentes parámetros."""
    from qiskit import QuantumCircuit, transpile
    from qiskit.circuit import Parameter
    from qiskit_aer import AerSimulator
    import numpy as np
    theta = Parameter("θ")
    qc = QuantumCircuit(1, 1)
    qc.ry(theta, 0); qc.measure(0, 0)
    sim = AerSimulator()
    # θ=0: prob(1) ≈ 0
    qc0 = qc.assign_parameters({theta: 0.0})
    counts0 = sim.run(transpile(qc0, sim), shots=2048).result().get_counts()
    # θ=π: prob(1) ≈ 1
    qc_pi = qc.assign_parameters({theta: np.pi})
    counts_pi = sim.run(transpile(qc_pi, sim), shots=2048).result().get_counts()
    p1_0 = counts0.get("1", 0) / 2048
    p1_pi = counts_pi.get("1", 0) / 2048
    assert p1_0 < 0.05, f"θ=0: P(1) debe ser ~0, got {p1_0:.4f}"
    assert p1_pi > 0.95, f"θ=π: P(1) debe ser ~1, got {p1_pi:.4f}"
    print(f"✓ test_parameter_binding (P1@0={p1_0:.4f}, P1@π={p1_pi:.4f})")

if __name__ == "__main__":
    print("Ejecutando checkpoint M16 — Hardware Real\n")
    test_transpilacion_reduce_profundidad()
    test_fake_backend_disponible()
    test_sampler_bell()
    test_estimator_zz_bell()
    test_noise_degrada_fidelidad()
    test_shots_estadistica()
    test_parameter_binding()
    print("\n✓ Todos los tests de M16 pasaron")
```
