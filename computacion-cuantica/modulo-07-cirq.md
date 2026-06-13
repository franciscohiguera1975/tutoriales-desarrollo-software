# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 7 — Cirq: Google Quantum AI

> **Objetivo:** Dominar Cirq como framework de bajo nivel para computación cuántica: modelo de Moments, qubits físicos, simuladores avanzados, modelos de ruido, y la conexión con hardware Google (Sycamore/Willow). Entender por qué Cirq es diferente de Qiskit y PennyLane.
>
> **Herramientas:** Cirq 1.x + NumPy.
>
> **Prerequisito:** M06 — PennyLane.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-07-cirq.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-07-cirq.ipynb
```

### Opción C — Google Colab
```python
!pip install cirq -q
```

---

## 7.1 Filosofía de Cirq — Lo que lo hace diferente

```
Qiskit  → abstracción ALTA  — funciona con cualquier hardware, la transpilación lo resuelve
PennyLane → abstracción MUY ALTA — pensado para ML, diferenciación, hibridación
Cirq    → abstracción BAJA  — expone la física del hardware, Moments explícitos

En Cirq:
  - Los qubits tienen POSICIÓN FÍSICA (GridQubit, LineQubit)
  - Las operaciones se agrupan en MOMENTS (capas paralelas exactas)
  - El compilador NO reordena puertas automáticamente
  - Ideal para hardware Google: Sycamore (Weber), Willow
```

---

## 7.2 Tipos de Qubits en Cirq

```python
import cirq
import numpy as np
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

# ── LineQubit: índice lineal 0, 1, 2, ... ────────────────
q0, q1, q2 = cirq.LineQubit.range(3)
print(f"LineQubit: {q0}, {q1}, {q2}")

# ── GridQubit: posición 2D (fila, columna) — hardware Google ─
g00 = cirq.GridQubit(0, 0)   # (fila=0, col=0)
g01 = cirq.GridQubit(0, 1)
g10 = cirq.GridQubit(1, 0)
g11 = cirq.GridQubit(1, 1)
print(f"\nGridQubit: {g00}, {g01}, {g10}, {g11}")

# Los GridQubits reflejan la topología real del procesador
# Sycamore: grilla de 5×10 qubits, solo vecinos pueden interactuar

# ── NamedQubit: identificador semántico ───────────────────
alice = cirq.NamedQubit("Alice")
bob   = cirq.NamedQubit("Bob")
print(f"\nNamedQubit: {alice}, {bob}")

# ── Circuito con GridQubits ───────────────────────────────
# Representación de la topología 2×2 del procesador
grilla = cirq.GridQubit.rect(2, 2)   # 4 qubits en grilla 2×2
print(f"\nGridQubit.rect(2,2): {grilla}")

circuito_grilla = cirq.Circuit([
    cirq.H(grilla[0]),
    cirq.CNOT(grilla[0], grilla[1]),
    cirq.CNOT(grilla[1], grilla[3]),
    cirq.measure(*grilla, key='resultado'),
])
print("\nCircuito en grilla 2×2:")
print(circuito_grilla)
```

---

## 7.3 Moments — Paralelismo Explícito

```python
# En Cirq, el paralelismo es EXPLÍCITO mediante Moments
# Cada Moment contiene operaciones que pueden ejecutarse en paralelo

q = cirq.LineQubit.range(4)

# ── Construcción automática — Cirq detecta paralelismo ───
circ_auto = cirq.Circuit([
    cirq.H(q[0]), cirq.H(q[1]), cirq.H(q[2]), cirq.H(q[3]),  # paralelo
    cirq.CNOT(q[0], q[1]),                                     # Moment 2
    cirq.CNOT(q[2], q[3]),                                     # mismo Moment 2 (paralelo)
    cirq.CNOT(q[1], q[2]),                                     # Moment 3 — depende de q[1] y q[2]
])
print("Circuito con paralelismo automático:")
print(circ_auto)
print(f"\nNúmero de Moments: {len(circ_auto)}")
for i, momento in enumerate(circ_auto):
    ops = list(momento.operations)
    print(f"  Moment {i+1}: {[str(op) for op in ops]}")

# ── Construcción manual de Moments ────────────────────────
momento_1 = cirq.Moment([cirq.H(q[0]), cirq.H(q[1]), cirq.H(q[2]), cirq.H(q[3])])
momento_2 = cirq.Moment([cirq.CNOT(q[0], q[1]), cirq.CNOT(q[2], q[3])])
momento_3 = cirq.Moment([cirq.CNOT(q[1], q[2])])
momento_4 = cirq.Moment([cirq.measure(q[0], q[1], q[2], q[3], key='m')])

circ_manual = cirq.Circuit([momento_1, momento_2, momento_3, momento_4])
print("\nCircuito con Moments manuales:")
print(circ_manual)

# Verificar que ambos circuitos son equivalentes
sim = cirq.Simulator()
sv_auto   = sim.simulate(circ_auto).final_state_vector
sv_manual = sim.simulate(circ_manual.without_terminal_measurements()).final_state_vector
print(f"\n¿Equivalentes? {np.allclose(np.abs(sv_auto), np.abs(sv_manual), atol=1e-8)}")
```

---

## 7.4 Puertas en Cirq — Catálogo

```python
# ── Puertas de 1 qubit ────────────────────────────────────
q0 = cirq.LineQubit(0)

puertas_1q = {
    "X":          cirq.X(q0),
    "Y":          cirq.Y(q0),
    "Z":          cirq.Z(q0),
    "H":          cirq.H(q0),
    "S":          cirq.S(q0),
    "T":          cirq.T(q0),
    "S†":         cirq.S(q0)**-1,      # S† = S⁻¹
    "T†":         cirq.T(q0)**-1,
    "Rx(π/4)":    cirq.rx(np.pi/4)(q0),
    "Ry(π/3)":    cirq.ry(np.pi/3)(q0),
    "Rz(π/2)":    cirq.rz(np.pi/2)(q0),
    "PhasedX":    cirq.PhasedXPowGate(phase_exponent=0.5, exponent=0.5).on(q0),
}

print("Puertas de 1 qubit en Cirq:")
for nombre, op in list(puertas_1q.items())[:6]:
    print(f"  {nombre}: {op}")

# ── Puertas parametrizadas con exponentes ─────────────────
# En Cirq, T = Z^(1/4), S = Z^(1/2), X es X^1
# Esto permite rotaciones continuas:
T_cuadrado = cirq.Z(q0)**(0.25)   # mismo que T
T_cubo     = cirq.Z(q0)**(3/8)
print(f"\nZ^(1/4) = T: {T_cuadrado}")
print(f"Z^(3/8):     {T_cubo}")

# ── Puertas de 2 qubits ───────────────────────────────────
q0, q1 = cirq.LineQubit.range(2)

puertas_2q = {
    "CNOT":   cirq.CNOT(q0, q1),
    "CZ":     cirq.CZ(q0, q1),
    "SWAP":   cirq.SWAP(q0, q1),
    "iSWAP":  cirq.ISWAP(q0, q1),
    "√iSWAP": cirq.ISWAP(q0, q1)**0.5,    # √iSWAP — nativa en Google
    "XX":     cirq.XX(q0, q1),
    "YY":     cirq.YY(q0, q1),
    "ZZ":     cirq.ZZ(q0, q1),
}

print("\nPuertas de 2 qubits en Cirq:")
for nombre, op in puertas_2q.items():
    print(f"  {nombre}: {op}")

# ── La puerta Sycamore (√iSWAP·CZ) — nativa de Google ────
# Google usa principalmente √iSWAP + RZ como basis set
sycamore_gate = cirq.SycamoreGate()
print(f"\nSycamore gate: {sycamore_gate}")

# Verificar √iSWAP² = iSWAP
sqrt_iswap = cirq.ISWAP(q0, q1)**0.5
circ_sq    = cirq.Circuit([sqrt_iswap, sqrt_iswap])
U_sq = cirq.unitary(circ_sq)
U_isw = cirq.unitary(cirq.Circuit([cirq.ISWAP(q0, q1)]))
print(f"\n(√iSWAP)² == iSWAP: {np.allclose(np.abs(U_sq), np.abs(U_isw), atol=1e-8)}")
```

---

## 7.5 Simuladores de Cirq

```python
# ── Simulador de Estado Puro ──────────────────────────────
sim_puro = cirq.Simulator()

q = cirq.LineQubit.range(3)
circ_ghz = cirq.Circuit([
    cirq.H(q[0]),
    cirq.CNOT(q[0], q[1]),
    cirq.CNOT(q[1], q[2]),
])

resultado = sim_puro.simulate(circ_ghz)
sv = resultado.final_state_vector
print("GHZ-3 statevector:")
print(f"  {np.round(sv, 3)}")
print(f"  P(|000⟩) = {abs(sv[0])**2:.3f}")
print(f"  P(|111⟩) = {abs(sv[7])**2:.3f}")

# ── Simulador de Clifford (solo puertas {H, CNOT, S}) ─────
# Exponencialmente más eficiente para circuitos Clifford puros
sim_cliff = cirq.CliffordSimulator()
circ_cliff = cirq.Circuit([
    cirq.H(q[0]), cirq.H(q[1]),
    cirq.CNOT(q[0], q[1]),
    cirq.S(q[0]),
    cirq.measure(*q[:2], key='m')
])
result_cliff = sim_cliff.simulate(circ_cliff)
print(f"\nClifford simulator: {result_cliff}")

# ── Simulador de Density Matrix (con ruido) ───────────────
sim_dm = cirq.DensityMatrixSimulator()
resultado_dm = sim_dm.simulate(circ_ghz)
rho = resultado_dm.final_density_matrix
print(f"\nDensity matrix GHZ-3 (elemento 0,0): {rho[0,0]:.4f}")
print(f"Traza de ρ: {np.trace(rho).real:.6f}")
print(f"Pureza Tr(ρ²): {np.trace(rho @ rho).real:.6f}")

# ── Muestreo — equivalente a "shots" ─────────────────────
circ_sample = cirq.Circuit([
    cirq.H(q[0]),
    cirq.CNOT(q[0], q[1]),
    cirq.CNOT(q[1], q[2]),
    cirq.measure(*q, key='medicion'),
])
resultado_shots = sim_puro.run(circ_sample, repetitions=2048)
histograma = resultado_shots.histogram(key='medicion')
print("\nMuestreo GHZ-3 (2048 shots):")
for bits, cnt in sorted(histograma.items()):
    nombre_bits = format(bits, f'0{len(q)}b')
    print(f"  |{nombre_bits}⟩: {cnt/2048:.3f}")
```

---

## 7.6 Modelos de Ruido en Cirq

```python
# Cirq tiene un sistema de ruido más físico que Qiskit
# Los errores se modelan como canales cuánticos

# ── Canal de despolarización ─────────────────────────────
def circuito_con_ruido_depolarizante(p: float):
    """Circuito Bell con ruido depolarizante de probabilidad p."""
    q0, q1 = cirq.LineQubit.range(2)
    circ = cirq.Circuit([
        cirq.H(q0), cirq.depolarize(p)(q0),
        cirq.CNOT(q0, q1), cirq.depolarize(p)(q0), cirq.depolarize(p)(q1),
        cirq.measure(q0, q1, key='m'),
    ])
    return circ

# Comparar para distintos niveles de ruido
probabilidades = [0.0, 0.01, 0.05, 0.1, 0.2]
resultados_ruido = {}
for p in probabilidades:
    circ_n = circuito_con_ruido_depolarizante(p)
    sim_n  = cirq.DensityMatrixSimulator()
    res_n  = sim_n.run(circ_n, repetitions=4096)
    hist_n = res_n.histogram(key='m')
    total  = sum(hist_n.values())
    # Error = probabilidad de estados inesperados (01 y 10)
    p_error = (hist_n.get(1, 0) + hist_n.get(2, 0)) / total
    resultados_ruido[p] = {
        'p_00': hist_n.get(0, 0) / total,
        'p_11': hist_n.get(3, 0) / total,
        'p_error': p_error,
    }
    print(f"  p={p:.2f}: P(00)={resultados_ruido[p]['p_00']:.3f}, "
          f"P(11)={resultados_ruido[p]['p_11']:.3f}, "
          f"error={p_error:.3f}")

# Graficar tasa de error vs p_depo
fig, ax = plt.subplots(figsize=(7, 4))
ax.plot(probabilidades, [resultados_ruido[p]['p_error'] for p in probabilidades],
        'o-', color='coral', linewidth=2, markersize=6)
ax.set_xlabel("Probabilidad de despolarización (p)")
ax.set_ylabel("Tasa de error en Bell state")
ax.set_title("Degradación del Bell state con ruido depolarizante")
ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig("outputs/ruido_cirq_depolarizante.png", dpi=150, bbox_inches='tight')
plt.show()
```

### Modelos de ruido más complejos

```python
# ── AmplitudeDamping — relajación de energía (T1) ────────
class RuidoT1T2(cirq.NoiseModel):
    """Modelo de ruido T1/T2 para hardware superconductor."""
    def __init__(self, p_depol_1q: float = 0.003, p_depol_2q: float = 0.01,
                 p_readout: float = 0.015):
        self.p1 = p_depol_1q
        self.p2 = p_depol_2q
        self.p_ro = p_readout

    def noisy_operation(self, op: cirq.Operation) -> cirq.OP_TREE:
        gate = op.gate
        if isinstance(gate, cirq.MeasurementGate):
            # Error en readout
            yield op
            for q in op.qubits:
                yield cirq.BitFlipChannel(self.p_ro)(q)
        elif len(op.qubits) == 1:
            yield op
            yield cirq.depolarize(self.p1)(op.qubits[0])
        elif len(op.qubits) == 2:
            yield op
            for q in op.qubits:
                yield cirq.depolarize(self.p2)(q)
        else:
            yield op

# Aplicar modelo de ruido al simulador
modelo_ruido = RuidoT1T2(p_depol_1q=0.005, p_depol_2q=0.02, p_readout=0.01)
sim_con_ruido = cirq.DensityMatrixSimulator(noise=modelo_ruido)

q0, q1 = cirq.LineQubit.range(2)
circ_bell_puro = cirq.Circuit([
    cirq.H(q0), cirq.CNOT(q0, q1),
    cirq.measure(q0, q1, key='m'),
])
res_con_ruido = sim_con_ruido.run(circ_bell_puro, repetitions=4096)
hist_ruido = res_con_ruido.histogram(key='m')
total_r = sum(hist_ruido.values())

print("\nBell state con modelo de ruido T1/T2:")
for bits in range(4):
    nombre = format(bits, '02b')
    print(f"  |{nombre}⟩: {hist_ruido.get(bits,0)/total_r:.3f}")

# ── Canales estándar de Cirq ──────────────────────────────
q = cirq.LineQubit(0)
print("\nCanales de ruido disponibles en Cirq:")
canales = {
    "BitFlipChannel(0.1)":          cirq.BitFlipChannel(0.1)(q),
    "PhaseFlipChannel(0.1)":        cirq.PhaseFlipChannel(0.1)(q),
    "DepolarizingChannel(0.1)":     cirq.depolarize(0.1)(q),
    "AmplitudeDampingChannel(0.1)": cirq.amplitude_damp(0.1)(q),
    "PhaseDampingChannel(0.1)":     cirq.phase_damp(0.1)(q),
}
for nombre, op in canales.items():
    print(f"  {nombre}")
```

---

## 7.7 Compilación y Optimización para Hardware Google

```python
# El compilador de Cirq optimiza circuitos para backends específicos
# Basis set de Google (Sycamore): √iSWAP + Rz + PhasedXPowGate

from cirq.transformers import (
    optimize_for_target_gateset,
    merge_single_qubit_gates_to_phased_x_and_z,
    two_qubit_to_sqrt_iswap,
)

# Circuito de ejemplo con puertas abstractas
q0, q1, q2 = cirq.LineQubit.range(3)
circ_abstracto = cirq.Circuit([
    cirq.H(q0),
    cirq.CNOT(q0, q1),
    cirq.CNOT(q1, q2),
    cirq.T(q0),
    cirq.S(q1),
    cirq.ry(np.pi/5)(q2),
    cirq.CZ(q0, q2),
])
print("Circuito abstracto:")
print(circ_abstracto)

# Descomponer CNOT en términos de √iSWAP (basis Google)
try:
    circ_google = cirq.optimize_for_target_gateset(
        circ_abstracto,
        gateset=cirq.SqrtIswapTargetGateset(),
    )
    print("\nCircuito optimizado para Google (√iSWAP basis):")
    print(circ_google)
    print(f"\nMoments: {len(circ_abstracto)} → {len(circ_google)}")
except Exception as e:
    print(f"Optimización: {e}")
    # Fallback: transformación manual
    circ_google = two_qubit_to_sqrt_iswap.TwoQubitToSqrtIswap().optimize_circuit(circ_abstracto)
    print(f"\nCircuito transformado a √iSWAP:")
    print(circ_google)
```

---

## 7.8 Conexión con Google Quantum AI

```python
# ── Procesador virtual (sin cuenta real) ──────────────────
# Cirq incluye especificaciones de hardware virtual para pruebas

def explorar_procesador_virtual():
    """Muestra las características del procesador Sycamore simulado."""
    # Dispositivo de grilla 3×3 para experimentar
    device = cirq.GridQubit.rect(3, 3)
    print("Dispositivo virtual 3×3:")
    for qubit in device:
        vecinos = [q for q in device if cirq.GridQubit.is_adjacent(qubit, q)]
        print(f"  {qubit}: vecinos = {vecinos}")

explorar_procesador_virtual()

# ── Ejecutar en procesador simulado con restricciones físicas ──
def circuito_respetando_topologia(grilla: list) -> cirq.Circuit:
    """
    Solo aplica puertas entre qubits adyacentes en la grilla.
    """
    q = {(i, j): cirq.GridQubit(i, j) for i in range(2) for j in range(2)}

    circ = cirq.Circuit([
        cirq.H(q[(0,0)]),
        cirq.CNOT(q[(0,0)], q[(0,1)]),   # (0,0) y (0,1) son adyacentes ✓
        cirq.CNOT(q[(0,1)], q[(1,1)]),   # (0,1) y (1,1) son adyacentes ✓
        # cirq.CNOT(q[(0,0)], q[(1,1)]), # NO adyacentes — requeriría SWAP
        cirq.measure(*q.values(), key='m'),
    ])
    return circ

circ_topo = circuito_respetando_topologia(None)
print("\nCircuito respetando topología 2×2:")
print(circ_topo)

# Simular
sim_topo = cirq.Simulator()
res_topo = sim_topo.run(circ_topo, repetitions=1024)
hist_topo = res_topo.histogram(key='m')
total_t = sum(hist_topo.values())
print("Resultados:")
for bits, cnt in sorted(hist_topo.items(), key=lambda x: -x[1])[:5]:
    print(f"  {format(bits, '04b')}: {cnt/total_t:.3f}")

# ── Plantilla para hardware real Google (con acceso) ──────
def plantilla_google_cloud(project_id: str = None, processor_id: str = None):
    """
    Plantilla para ejecutar en Google Quantum Computing Service.
    Requiere proyecto de Google Cloud con acceso a quantum.
    """
    if project_id is None:
        print("Para hardware real Google:")
        print("  1. Solicitar acceso: https://quantumai.google/quantum-computing-service")
        print("  2. Crear proyecto Google Cloud")
        print("  3. Instalar: pip install cirq-google")
        print("  4. Configurar:")
        print("     import cirq_google")
        print("     engine = cirq_google.get_engine(project_id='mi-proyecto')")
        print("     processor = engine.get_processor('rainbow')")
        print("     job = processor.run(circuit, repetitions=1000)")
        return

plantilla_google_cloud()
```

---

## 7.9 Visualización y Análisis de Circuitos en Cirq

```python
# ── Visualización ASCII ───────────────────────────────────
q = cirq.LineQubit.range(4)
circ_visual = cirq.Circuit([
    cirq.H.on_each(*q),
    cirq.CNOT(q[0], q[1]),
    cirq.CNOT(q[2], q[3]),
    [cirq.rz(np.pi/4)(qi) for qi in q],
    cirq.CNOT(q[1], q[2]),
    cirq.measure(*q, key='m'),
])
print("Circuito Cirq (ASCII):")
print(circ_visual)

# ── Información del circuito ─────────────────────────────
print(f"\nEstadísticas del circuito:")
print(f"  Qubits: {sorted(circ_visual.all_qubits())}")
print(f"  Profundidad (Moments): {len(circ_visual)}")
print(f"  Operaciones totales: {sum(1 for _ in circ_visual.all_operations())}")

# Contar por tipo de puerta
from collections import Counter
tipos = Counter(
    type(op.gate).__name__
    for op in circ_visual.all_operations()
    if op.gate is not None
)
print(f"  Tipos de puerta: {dict(tipos)}")

# ── Exportar como SVG/PNG (requiere svgwrite) ────────────
try:
    svg_text = circ_visual._repr_svg_()
    with open("outputs/circuito_cirq.svg", "w") as f:
        f.write(svg_text)
    print("\nCircuito exportado como SVG")
except Exception:
    pass   # svgwrite no siempre disponible

# ── Matriz unitaria del circuito ─────────────────────────
circ_sin_medicion = circ_visual[:-1]   # quitar medición
U = cirq.unitary(circ_sin_medicion)
print(f"\nMatriz unitaria: forma {U.shape}, unitaria: {np.allclose(U.conj().T @ U, np.eye(16), atol=1e-8)}")
```

---

## 7.10 Ejemplo Completo — Protocolo BB84 (Preview de M32)

```python
# Protocolo de criptografía cuántica BB84 simplificado
# Alice envía qubits en bases aleatorias, Bob mide en bases aleatorias
# Solo los bits donde coinciden las bases forman la clave

import random

def bb84_simulado(n_bits: int = 20, p_eave: float = 0.0) -> dict:
    """
    Simula el protocolo BB84 con n_bits.
    p_eave: probabilidad de intercepción por Eve (introduce errores).
    """
    random.seed(42)
    np.random.seed(42)
    sim = cirq.Simulator()

    bits_alice   = [random.randint(0, 1) for _ in range(n_bits)]
    bases_alice  = [random.randint(0, 1) for _ in range(n_bits)]  # 0=Z, 1=X
    bases_bob    = [random.randint(0, 1) for _ in range(n_bits)]
    bits_bob     = []

    for bit_a, base_a, base_b in zip(bits_alice, bases_alice, bases_bob):
        q = cirq.LineQubit(0)
        ops = []

        # Alice prepara el qubit
        if bit_a == 1:
            ops.append(cirq.X(q))    # |1⟩ en base Z, o |−⟩ en base X
        if base_a == 1:
            ops.append(cirq.H(q))    # cambiar a base X

        # Eve intercepta (si hay escucha)
        if p_eave > 0 and random.random() < p_eave:
            base_eve = random.randint(0, 1)
            if base_eve == 1:
                ops.append(cirq.H(q))
            ops.append(cirq.measure(q, key='eve'))
            # Eve reenvía (re-preparación) — ignoramos este paso simplificado

        # Bob mide
        if base_b == 1:
            ops.append(cirq.H(q))    # rotar a base X antes de medir Z
        ops.append(cirq.measure(q, key='bob'))

        circ_bb84 = cirq.Circuit(ops)
        res = sim.run(circ_bb84, repetitions=1)
        bits_bob.append(int(res.measurements['bob'][0][0]))

    # Reconciliación: Bob y Alice comparan bases públicamente
    clave_alice = []
    clave_bob   = []
    indices_coincidentes = []

    for i, (ba, bb) in enumerate(zip(bases_alice, bases_bob)):
        if ba == bb:
            clave_alice.append(bits_alice[i])
            clave_bob.append(bits_bob[i])
            indices_coincidentes.append(i)

    tasa_coincidencia = len(indices_coincidentes) / n_bits
    errores = sum(a != b for a, b in zip(clave_alice, clave_bob))
    qber = errores / len(clave_alice) if clave_alice else 0   # Quantum Bit Error Rate

    return {
        'bits_alice':        bits_alice,
        'bits_bob':          bits_bob,
        'bases_alice':       bases_alice,
        'bases_bob':         bases_bob,
        'clave_alice':       clave_alice,
        'clave_bob':         clave_bob,
        'n_coincidentes':    len(indices_coincidentes),
        'tasa_coincidencia': tasa_coincidencia,
        'QBER':              qber,
    }

# Sin escucha
res_sin_eve = bb84_simulado(n_bits=100, p_eave=0.0)
print("BB84 sin Eve:")
print(f"  Bits enviados: {res_sin_eve['n_coincidentes']} bases coincidentes de 100")
print(f"  Tasa de coincidencia: {res_sin_eve['tasa_coincidencia']:.2f} (esperado ≈0.5)")
print(f"  QBER: {res_sin_eve['QBER']:.4f} (esperado 0 — sin errores)")

# Con escucha (Eve intercepta 50% de los qubits)
res_con_eve = bb84_simulado(n_bits=100, p_eave=0.5)
print("\nBB84 con Eve (50% intercepción):")
print(f"  QBER: {res_con_eve['QBER']:.4f} (esperado ≈0.25 — Eve introduce errores)")
print("  Nota: QBER > 0.11 indica presencia de eavesdropper")
```

---

## Resumen — Cuándo usar Cirq

| Situación | ¿Usar Cirq? |
|---|---|
| Hardware Google (Sycamore, Willow) | ✓ Primera opción |
| Control preciso de Moments/paralelismo | ✓ Diseñado para esto |
| Modelos de ruido físicos detallados | ✓ Canal API expresivo |
| Criptografía cuántica (BB84, QKD) | ✓ Control de bajo nivel |
| QML con gradientes automáticos | ✗ Usar PennyLane |
| Algoritmos de alto nivel (Grover, Shor) | ~ Posible, Qiskit más fácil |
| Hardware IBM | ✗ Usar Qiskit |
| Primer framework para aprender | ~ Curva de aprendizaje alta |

---

## checkpoint_m07.py

```python
"""
Checkpoint Módulo 7 — Cirq: Google Quantum AI
Ejecutar con: python checkpoint_m07.py
"""
import cirq
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)
sim = cirq.Simulator()

print("=" * 55)
print("Checkpoint M07 — Cirq")
print("=" * 55)

# ── Test 1: LineQubit y GridQubit ────────────────────────
q_line = cirq.LineQubit.range(3)
q_grid = cirq.GridQubit.rect(2, 2)
assert len(q_line) == 3, "FALLO: LineQubit.range(3)"
assert len(q_grid) == 4, "FALLO: GridQubit.rect(2,2)"
print(f"✓ Test 1: LineQubit {q_line}, GridQubit {q_grid}")

# ── Test 2: Circuito Bell básico ─────────────────────────
q0, q1 = cirq.LineQubit.range(2)
circ_bell = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])
sv_bell = sim.simulate(circ_bell).final_state_vector
assert abs(sv_bell[0])**2 > 0.49 and abs(sv_bell[3])**2 > 0.49, \
    "FALLO: Bell state Cirq"
print(f"✓ Test 2: Bell state: P(00)={abs(sv_bell[0])**2:.3f}, P(11)={abs(sv_bell[3])**2:.3f}")

# ── Test 3: Moments ───────────────────────────────────────
q = cirq.LineQubit.range(4)
circ_parallel = cirq.Circuit([
    cirq.H(q[0]), cirq.H(q[1]), cirq.H(q[2]), cirq.H(q[3]),
])
assert len(circ_parallel) == 1, \
    f"FALLO: 4 H paralelos deben ser 1 Moment, got {len(circ_parallel)}"
print(f"✓ Test 3: 4 H paralelos = 1 Moment")

# ── Test 4: Simulador de sampling ────────────────────────
circ_m = cirq.Circuit([
    cirq.H(q0), cirq.CNOT(q0, q1),
    cirq.measure(q0, q1, key='m'),
])
result = sim.run(circ_m, repetitions=2048)
hist = result.histogram(key='m')
total = sum(hist.values())
p00 = hist.get(0, 0) / total
p11 = hist.get(3, 0) / total
assert 0.45 < p00 < 0.55, f"FALLO: P(00) = {p00:.3f}"
assert 0.45 < p11 < 0.55, f"FALLO: P(11) = {p11:.3f}"
print(f"✓ Test 4: Sampling Bell: P(00)={p00:.3f}, P(11)={p11:.3f}")

# ── Test 5: Density Matrix simulator ─────────────────────
sim_dm = cirq.DensityMatrixSimulator()
res_dm = sim_dm.simulate(circ_bell)
rho = res_dm.final_density_matrix
pureza = np.trace(rho @ rho).real
assert abs(pureza - 1.0) < 0.01, f"FALLO: pureza Bell = {pureza:.4f}"
print(f"✓ Test 5: DensityMatrix — pureza Bell = {pureza:.4f} ≈ 1")

# ── Test 6: Ruido depolarizante ───────────────────────────
circ_noisy = cirq.Circuit([
    cirq.H(q0), cirq.depolarize(0.15)(q0),
    cirq.CNOT(q0, q1), cirq.depolarize(0.15)(q0), cirq.depolarize(0.15)(q1),
    cirq.measure(q0, q1, key='m'),
])
res_noisy = sim_dm.run(circ_noisy, repetitions=4096)
hist_noisy = res_noisy.histogram(key='m')
total_n = sum(hist_noisy.values())
errores = (hist_noisy.get(1,0) + hist_noisy.get(2,0)) / total_n
assert errores > 0.05, f"FALLO: ruido p=0.15 debe introducir errores, got {errores:.4f}"
print(f"✓ Test 6: Ruido depolarizante 15% — tasa de error = {errores:.3f}")

# ── Test 7: Matriz unitaria ───────────────────────────────
q0, q1 = cirq.LineQubit.range(2)
circ_U = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])
U = cirq.unitary(circ_U)
assert np.allclose(U.conj().T @ U, np.eye(4), atol=1e-8), "FALLO: U no es unitaria"
print(f"✓ Test 7: Matriz unitaria Bell (4×4), unitaria: ✓")

# ── Test 8: GHZ-3 ────────────────────────────────────────
q = cirq.LineQubit.range(3)
circ_ghz = cirq.Circuit([cirq.H(q[0]), cirq.CNOT(q[0],q[1]), cirq.CNOT(q[1],q[2])])
sv_ghz = sim.simulate(circ_ghz).final_state_vector
assert abs(sv_ghz[0])**2 > 0.49 and abs(sv_ghz[7])**2 > 0.49, "FALLO: GHZ-3"
print(f"✓ Test 8: GHZ-3: P(000)={abs(sv_ghz[0])**2:.3f}, P(111)={abs(sv_ghz[7])**2:.3f}")

# ── Test 9: Clifford simulator ────────────────────────────
sim_cliff = cirq.CliffordSimulator()
q0, q1 = cirq.LineQubit.range(2)
circ_cliff = cirq.Circuit([cirq.H(q0), cirq.CNOT(q0, q1)])
# Clifford no puede dar statevector directamente — verificar estadísticas
circ_cliff_m = circ_cliff + cirq.measure(q0, q1, key='m')
res_cliff = sim_cliff.run(circ_cliff_m, repetitions=1024)
hist_c = res_cliff.histogram(key='m')
p00_c = hist_c.get(0,0)/1024; p11_c = hist_c.get(3,0)/1024
assert 0.45 < p00_c < 0.55, f"FALLO: Clifford sim P(00)={p00_c:.3f}"
print(f"✓ Test 9: CliffordSimulator Bell: P(00)={p00_c:.3f}")

# ── Test 10: GridQubit adjacency ─────────────────────────
g00 = cirq.GridQubit(0, 0)
g01 = cirq.GridQubit(0, 1)
g10 = cirq.GridQubit(1, 0)
g11 = cirq.GridQubit(1, 1)
assert cirq.GridQubit.is_adjacent(g00, g01), "FALLO: (0,0) y (0,1) deben ser adyacentes"
assert cirq.GridQubit.is_adjacent(g00, g10), "FALLO: (0,0) y (1,0) deben ser adyacentes"
assert not cirq.GridQubit.is_adjacent(g00, g11), "FALLO: (0,0) y (1,1) NO deben ser adyacentes"
print("✓ Test 10: GridQubit adjacency correcto")

print("\n" + "=" * 55)
print("✅ Todos los tests del M07 pasaron correctamente")
print("=" * 55)
```
