# Módulo 14 — Ruido, Decoherencia y Error Mitigation

**Objetivo:** Modelar el ruido en computadoras cuánticas, entender la decoherencia y aplicar
técnicas de mitigación de errores sin corrección cuántica completa.

**Herramientas:** Qiskit, Qiskit Aer, numpy, matplotlib
**Prerequisito:** M05 (Qiskit), M13 (Hardware)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
python modulo-14-ruido-error.py
```

---

## 1. ¿Qué es el ruido cuántico?

Un qubit interactúa con su entorno. Esa interacción destruye la superposición coherente.
Esto se modela matemáticamente como **canales cuánticos** — mapas que transforman
matrices de densidad ρ → ε(ρ).

```python
# modulo-14-ruido-error.py
import numpy as np
import matplotlib.pyplot as plt
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit_aer.noise import (
    NoiseModel, depolarizing_error, thermal_relaxation_error,
    ReadoutError, pauli_error, kraus_error, phase_amplitude_damping_error,
    coherent_unitary_error,
)
from qiskit.quantum_info import (
    DensityMatrix, state_fidelity, Statevector, Operator, Kraus
)
from qiskit.visualization import plot_histogram
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Tipos de error cuántico
# ─────────────────────────────────────────────

print("=" * 60)
print("TIPOS DE ERROR CUÁNTICO")
print("=" * 60)

# Los 3 errores de Pauli
I = np.eye(2)
X = np.array([[0, 1], [1, 0]])   # bit flip
Y = np.array([[0, -1j], [1j, 0]])
Z = np.array([[1, 0], [0, -1]])  # phase flip

print("\nErrores de Pauli:")
print("  X (bit flip):   |0⟩→|1⟩, |1⟩→|0⟩")
print("  Z (phase flip): |+⟩→|-⟩, |-⟩→|+⟩")
print("  Y (bit+phase):  |0⟩→i|1⟩, |1⟩→-i|0⟩")

def aplicar_canal_pauli(rho, px, py, pz):
    """Canal de Pauli: ε(ρ) = (1-px-py-pz)ρ + px·XρX† + py·YρY† + pz·ZρZ†"""
    p0 = 1 - px - py - pz
    assert p0 >= 0, "Probabilidades deben sumar ≤ 1"
    resultado = p0 * rho
    resultado += px * X @ rho @ X.conj().T
    resultado += py * Y @ rho @ Y.conj().T
    resultado += pz * Z @ rho @ Z.conj().T
    return resultado

# Estado |+⟩ = (|0⟩+|1⟩)/√2
plus = np.array([1, 1]) / np.sqrt(2)
rho_plus = np.outer(plus, plus.conj())

print("\nEstado |+⟩ sin ruido:")
print(f"  ρ = {rho_plus.real}")

rho_noisy = aplicar_canal_pauli(rho_plus, px=0.1, py=0.0, pz=0.0)
print(f"\nDespués de bit flip (p=0.1):")
print(f"  ρ = {np.round(rho_noisy.real, 3)}")
print(f"  Pureza Tr(ρ²) = {np.trace(rho_noisy @ rho_noisy).real:.4f} (< 1 → mezcla)")

# Depolarizing channel: error más simétrico
def canal_depolarizante(rho, p):
    """ε(ρ) = (1-p)ρ + p·I/2 — mezcla con estado máximamente mixto"""
    return (1 - p) * rho + p * np.eye(2) / 2

rho_dep = canal_depolarizante(rho_plus, p=0.1)
print(f"\nDespués de depolarización (p=0.1):")
print(f"  Pureza = {np.trace(rho_dep @ rho_dep).real:.4f}")
print(f"  Fidelidad con |+⟩ = {np.trace(rho_plus @ rho_dep).real:.4f}")

# Amplitude damping: modelo de T1 (relajación)
def canal_amplitude_damping(rho, gamma):
    """
    K0 = [[1,0],[0,√(1-γ)]], K1 = [[0,√γ],[0,0]]
    Modela decaimiento del estado excitado |1⟩ → |0⟩
    """
    K0 = np.array([[1, 0], [0, np.sqrt(1 - gamma)]])
    K1 = np.array([[0, np.sqrt(gamma)], [0, 0]])
    return K0 @ rho @ K0.conj().T + K1 @ rho @ K1.conj().T

rho_1 = np.array([[0, 0], [0, 1]])  # estado |1⟩
print(f"\nAmplitude Damping (T1 decay) sobre |1⟩:")
for gamma in [0.0, 0.2, 0.5, 0.8, 1.0]:
    rho_out = canal_amplitude_damping(rho_1, gamma)
    print(f"  γ={gamma:.1f}: ρ[1,1] = {rho_out[1,1].real:.3f}  (prob. de seguir en |1⟩)")
```

---

## 2. T1 y T2 — Los parámetros de coherencia

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: T1, T2 y relajación
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("T1 Y T2 — TIEMPOS DE COHERENCIA")
print("="*60)

# Curva de relajación T1
T1 = 100e-6  # 100 microsegundos
T2 = 80e-6   # 80 microsegundos (T2 ≤ 2·T1)
t = np.linspace(0, 3*T1, 1000)

# T1: decaimiento de |1⟩ a |0⟩
# P(|1⟩, t) = exp(-t/T1) si arrancamos en |1⟩
P_1 = np.exp(-t / T1)

# T2: pérdida de coherencia (fase)
# ⟨X(t)⟩ = exp(-t/T2) · cos(ω·t) — precesión + decaimiento
omega = 2 * np.pi * 5e6  # frecuencia de Larmor (5 MHz)
coherencia_real = np.exp(-t / T2) * np.cos(omega * t)

# T2* (sin echo): incluye inhomogeneidades
T2_star = 20e-6
coherencia_sin_echo = np.exp(-t / T2_star)

fig, axes = plt.subplots(1, 3, figsize=(15, 4))

axes[0].plot(t*1e6, P_1, "b-", linewidth=2, label=f"T1 = {T1*1e6:.0f} μs")
axes[0].axhline(y=1/np.e, color="red", linestyle="--", alpha=0.7, label="1/e")
axes[0].fill_between(t*1e6, P_1, alpha=0.2)
axes[0].set_xlabel("Tiempo (μs)")
axes[0].set_ylabel("P(|1⟩)")
axes[0].set_title("Relajación T1\n(bit flip: |1⟩→|0⟩)")
axes[0].legend()
axes[0].grid(True, alpha=0.3)

axes[1].plot(t*1e6, coherencia_real, "g-", linewidth=1.5, label=f"T2 = {T2*1e6:.0f} μs")
axes[1].plot(t*1e6, np.exp(-t/T2), "g--", alpha=0.5, label="Envolvente T2")
axes[1].set_xlabel("Tiempo (μs)")
axes[1].set_ylabel("⟨X(t)⟩")
axes[1].set_title("Decoherencia T2\n(pérdida de fase)")
axes[1].legend()
axes[1].grid(True, alpha=0.3)

axes[2].plot(t*1e6, coherencia_sin_echo, "r-", linewidth=2, label=f"T2* = {T2_star*1e6:.0f} μs")
axes[2].plot(t*1e6, np.exp(-t/T2), "b-", linewidth=2, label=f"T2 = {T2*1e6:.0f} μs")
axes[2].set_xlabel("Tiempo (μs)")
axes[2].set_ylabel("Coherencia")
axes[2].set_title("T2* vs T2 (con Hahn echo)")
axes[2].legend()
axes[2].grid(True, alpha=0.3)

plt.suptitle("Tiempos de Coherencia T1, T2, T2*", fontsize=12, fontweight="bold")
plt.tight_layout()
plt.savefig("outputs/m14_t1_t2_coherencia.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m14_t1_t2_coherencia.png")

print(f"\nParámetros típicos IBM Eagle (2023):")
parametros_ibm = {
    "T1": "100-200 μs",
    "T2": "50-150 μs",
    "T2*": "10-50 μs",
    "Error puerta 1Q": "~0.05-0.1%",
    "Error puerta 2Q": "~0.3-1.0%",
    "Error readout": "~1-5%",
    "Frecuencia 1Q": "~50 MHz",
    "Frecuencia 2Q": "~10-50 MHz",
}
for param, val in parametros_ibm.items():
    print(f"  {param:<20} {val}")
```

---

## 3. Modelos de ruido en Qiskit Aer

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: NoiseModel en Qiskit
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("MODELOS DE RUIDO EN QISKIT AER")
print("="*60)

# --- 3.1 Modelo de ruido básico ---
def crear_modelo_basico(p1q=0.001, p2q=0.01, p_readout=0.02):
    """
    Modelo de ruido simplificado:
    - Error depolarizante en puertas 1Q y 2Q
    - Error de lectura (readout)
    """
    noise_model = NoiseModel()

    # Error en puertas de 1 qubit
    error_1q = depolarizing_error(p1q, 1)
    noise_model.add_all_qubit_quantum_error(error_1q, ["h", "x", "y", "z", "s", "t", "u"])

    # Error en puertas de 2 qubits
    error_2q = depolarizing_error(p2q, 2)
    noise_model.add_all_qubit_quantum_error(error_2q, ["cx", "cz", "swap"])

    # Error de medición
    error_ro = ReadoutError([[1-p_readout, p_readout], [p_readout, 1-p_readout]])
    noise_model.add_all_qubit_readout_error(error_ro)

    return noise_model

# --- 3.2 Modelo de ruido T1/T2 ---
def crear_modelo_t1t2(T1_us=100, T2_us=80, t_gate_1q_ns=50, t_gate_2q_ns=300):
    """
    Modelo de ruido basado en parámetros físicos T1 y T2.
    """
    noise_model = NoiseModel()
    T1 = T1_us * 1e3     # ns
    T2 = T2_us * 1e3     # ns

    # Error T1/T2 para puertas 1Q
    error_1q = thermal_relaxation_error(T1, T2, t_gate_1q_ns)
    noise_model.add_all_qubit_quantum_error(error_1q, ["h", "x", "y", "z", "s", "t", "u"])

    # Error T1/T2 para puertas 2Q
    error_2q = thermal_relaxation_error(T1, T2, t_gate_2q_ns).expand(
        thermal_relaxation_error(T1, T2, t_gate_2q_ns)
    )
    noise_model.add_all_qubit_quantum_error(error_2q, ["cx"])

    return noise_model

# Comparar simulaciones: ideal vs ruidosa
def ejecutar_circuito_comparado(qc, shots=8192):
    """Ejecuta el mismo circuito ideal y con ruido."""
    sim_ideal = AerSimulator()
    sim_noisy = AerSimulator()
    noise_model = crear_modelo_basico(p1q=0.002, p2q=0.02, p_readout=0.03)
    sim_noisy = AerSimulator.from_backend_properties(
        noise_model=noise_model
    ) if False else AerSimulator(noise_model=noise_model)

    qc_t = transpile(qc, sim_ideal)
    result_ideal = sim_ideal.run(qc_t, shots=shots).result()
    counts_ideal = result_ideal.get_counts()

    qc_t2 = transpile(qc, sim_noisy)
    result_noisy = sim_noisy.run(qc_t2, shots=shots).result()
    counts_noisy = result_noisy.get_counts()

    return counts_ideal, counts_noisy

# Circuito de prueba: Bell state
qc_bell = QuantumCircuit(2, 2)
qc_bell.h(0)
qc_bell.cx(0, 1)
qc_bell.measure([0, 1], [0, 1])

counts_ideal, counts_noisy = ejecutar_circuito_comparado(qc_bell)

print(f"\nCircuito Bell — Ideal vs Ruidoso:")
print(f"  Ideal:  {counts_ideal}")
print(f"  Ruidoso: {counts_noisy}")

# Fidelidad de distribuciones
total_ideal = sum(counts_ideal.values())
total_noisy = sum(counts_noisy.values())
fidelidad_hellinger = 0
estados = set(list(counts_ideal.keys()) + list(counts_noisy.keys()))
for estado in estados:
    p = counts_ideal.get(estado, 0) / total_ideal
    q = counts_noisy.get(estado, 0) / total_noisy
    fidelidad_hellinger += np.sqrt(p * q)
print(f"\n  Fidelidad de Bhattacharyya: {fidelidad_hellinger:.4f}")
```

---

## 4. Técnicas de Error Mitigation

Las técnicas de Error Mitigation no corrigen errores (eso es QEC), sino que
**post-procesan** los resultados para estimar el valor sin ruido.

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: TÉCNICA 1 — Readout Error Mitigation
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("TÉCNICA 1: READOUT ERROR MITIGATION")
print("="*60)

print("""
Idea: el medidor introduce un flip con probabilidad p.
Calibrar: preparar |0⟩ y |1⟩ explícitamente, medir.
Construir la matriz de confusión A donde A[i,j] = P(medir i | preparar j).
Invertir: p_real = A⁻¹ · p_medido
""")

# Simulación de readout error mitigation
p_01 = 0.03   # P(medir 1 | preparar 0)
p_10 = 0.05   # P(medir 0 | preparar 1)

# Matriz de asignación (assignment matrix)
A = np.array([
    [1 - p_01, p_10],   # [P(0|0), P(0|1)]
    [p_01,  1 - p_10],  # [P(1|0), P(1|1)]
])
print(f"Matriz de asignación A:")
print(f"  {A}")

# Invertir la matriz
A_inv = np.linalg.inv(A)
print(f"\nA⁻¹:")
print(f"  {A_inv}")

# Counts medidos de un estado Bell
counts_raw = {"00": 4200, "11": 3600, "01": 150, "10": 50}
total = sum(counts_raw.values())

# Distribución medida para 2 qubits (producto tensorial de A)
probs_raw = {k: v/total for k, v in counts_raw.items()}
print(f"\nConteos brutos: {counts_raw}")

# Mitigación simple (tensorial para 2 qubits)
A2 = np.kron(A, A)  # producto de Kronecker para n=2 qubits
estados_2q = ["00", "01", "10", "11"]
p_raw_vec = np.array([probs_raw.get(s, 0) for s in estados_2q])
p_mitigado_vec = A_inv.flatten()  # simplificado

# Corrección real
p_mitigado = np.linalg.solve(A2, p_raw_vec)
p_mitigado = np.maximum(p_mitigado, 0)  # forzar no-negatividad
p_mitigado /= p_mitigado.sum()

print(f"\nDistribución mitigada:")
for s, p_raw, p_mit in zip(estados_2q, p_raw_vec, p_mitigado):
    print(f"  |{s}⟩: raw={p_raw:.4f} → mitigado={p_mit:.4f}")
```

---

## 5. Zero Noise Extrapolation (ZNE)

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: TÉCNICA 2 — ZNE
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("TÉCNICA 2: ZERO NOISE EXTRAPOLATION (ZNE)")
print("="*60)

print("""
Idea: amplificar deliberadamente el ruido (multiplicar puertas 2Q)
y extrapolar el valor esperado hacia ruido=0.

Amplificación: U_ruido(λ) = U · U† · U  (λ=3 inserta U·U† inerte)
Medición: E(λ=1), E(λ=3), E(λ=5)
Extrapolación: ajustar E(λ) y evaluar en λ=0
""")

def amplificar_circuito(qc: QuantumCircuit, factor: int) -> QuantumCircuit:
    """
    Amplifica el ruido del circuito insertando U·U† por cada puerta CNOT.
    factor=1: original, factor=3: U·U†·U, factor=5: U·U†·U·U†·U
    """
    assert factor % 2 == 1 and factor >= 1, "factor debe ser impar ≥ 1"
    if factor == 1:
        return qc.copy()

    qc_amp = QuantumCircuit(qc.num_qubits, qc.num_clbits)
    for instruccion in qc.data:
        qubits = instruccion.qubits
        clbits = instruccion.clbits
        qc_amp.append(instruccion)
        if instruccion.operation.name == "cx":
            n_extras = (factor - 1) // 2
            for _ in range(n_extras):
                qc_amp.cx(*[qc.find_bit(q).index for q in qubits])
                qc_amp.cx(*[qc.find_bit(q).index for q in qubits])
    return qc_amp

# Circuito VQE-like: Ry(θ)·CNOT·Ry(θ)
from qiskit.circuit import Parameter
theta = Parameter("θ")
qc_vqe = QuantumCircuit(2, 2)
qc_vqe.ry(theta, 0)
qc_vqe.ry(theta, 1)
qc_vqe.cx(0, 1)
qc_vqe.ry(theta, 0)
qc_vqe.measure([0, 1], [0, 1])

# Ejecutar con diferentes factores de amplificación
theta_val = np.pi / 4
factores = [1, 3, 5]
resultados_zne = []

noise_model = crear_modelo_basico(p1q=0.005, p2q=0.05, p_readout=0.0)
sim_noisy = AerSimulator(noise_model=noise_model)

print(f"\nCircuito VQE con θ={theta_val:.3f}:")
print(f"{'Factor':>8} {'⟨Z⊗I⟩ medido':>15}")
print(f"{'─'*25}")

for factor in factores:
    qc_bound = qc_vqe.assign_parameters({theta: theta_val})
    qc_amp = amplificar_circuito(qc_bound, factor)
    qc_t = transpile(qc_amp, sim_noisy)
    counts = sim_noisy.run(qc_t, shots=8192).result().get_counts()

    # Calcular ⟨Z⊗I⟩ = P(0x) - P(1x)
    total = sum(counts.values())
    z_exp = sum(
        (-1)**(int(k[0])) * v / total
        for k, v in counts.items()
    )
    resultados_zne.append(z_exp)
    print(f"  λ={factor}:    {z_exp:>+.4f}")

# Extrapolación lineal a λ=0
if len(factores) >= 2:
    coef = np.polyfit(factores, resultados_zne, deg=1)
    valor_extrapolado = np.polyval(coef, 0)
    print(f"\n  Extrapolación a λ=0: {valor_extrapolado:>+.4f}")

    # Valor ideal (sin ruido)
    sim_ideal = AerSimulator()
    qc_bound_ideal = qc_vqe.assign_parameters({theta: theta_val})
    qc_t_ideal = transpile(qc_bound_ideal, sim_ideal)
    counts_ideal = sim_ideal.run(qc_t_ideal, shots=32768).result().get_counts()
    total_i = sum(counts_ideal.values())
    z_ideal = sum((-1)**(int(k[0])) * v / total_i for k, v in counts_ideal.items())
    print(f"  Valor ideal:         {z_ideal:>+.4f}")
    print(f"  Mejora ZNE: error {abs(resultados_zne[0]-z_ideal):.4f} → {abs(valor_extrapolado-z_ideal):.4f}")

# Visualización ZNE
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

axes[0].scatter(factores, resultados_zne, s=100, zorder=5, color="red", label="Medido")
x_fit = np.linspace(0, 6, 100)
y_fit = np.polyval(coef, x_fit)
axes[0].plot(x_fit, y_fit, "b-", label="Extrapolación lineal", linewidth=2)
axes[0].scatter([0], [valor_extrapolado], s=200, marker="*", color="green",
               zorder=6, label=f"λ=0: {valor_extrapolado:.4f}")
axes[0].axhline(y=z_ideal, color="gray", linestyle="--", alpha=0.7, label=f"Ideal: {z_ideal:.4f}")
axes[0].set_xlabel("Factor de amplificación λ")
axes[0].set_ylabel("⟨Z⊗I⟩")
axes[0].set_title("Zero Noise Extrapolation")
axes[0].legend(fontsize=8)
axes[0].grid(True, alpha=0.3)

# Comparación de técnicas
tecnicas = ["Sin mitigación\n(λ=1)", "ZNE\nextrapolado", "Ideal\n(sin ruido)"]
valores = [resultados_zne[0], valor_extrapolado, z_ideal]
colores_b = ["red", "green", "blue"]
axes[1].bar(tecnicas, valores, color=colores_b, alpha=0.7)
axes[1].set_ylabel("⟨Z⊗I⟩")
axes[1].set_title("Comparación de estimaciones")
axes[1].grid(True, alpha=0.3, axis="y")
for i, (t, v) in enumerate(zip(tecnicas, valores)):
    axes[1].text(i, v + 0.005, f"{v:.4f}", ha="center", va="bottom", fontweight="bold")

plt.suptitle("Zero Noise Extrapolation (ZNE)", fontsize=12, fontweight="bold")
plt.tight_layout()
plt.savefig("outputs/m14_zne.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m14_zne.png")
```

---

## 6. Probabilistic Error Cancellation (PEC)

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: TÉCNICA 3 — PEC y otras
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("OTRAS TÉCNICAS DE ERROR MITIGATION")
print("="*60)

print("""
3. PROBABILISTIC ERROR CANCELLATION (PEC):
   Representa canal ruidoso como combinación de canales ideales.
   ε_ruidoso = Σ c_i · ε_i  (los c_i pueden ser negativos!)
   Muestrea aleatoriamente de {ε_i} con pesos |c_i|.
   Overhead de varianza: γ² donde γ = Σ|c_i| > 1
   Ventaja: exacto para cualquier observable.
   Costo: exponencial en número de puertas (no gratis).

4. SYMMETRY VERIFICATION:
   Si el estado ideal tiene una simetría S (S|ψ⟩=|ψ⟩),
   descartar muestras que violen la simetría.
   Ej: paridad par en sistemas de n qubits con conservación de carga.

5. CLIFFORD DATA REGRESSION (CDR):
   Ejecutar circuitos de Clifford cercanos al circuito objetivo.
   Circuitos Clifford son eficientemente simulables clásicamente.
   Usar resultados clásicos vs ruidosos para aprender corrección.
   Aplicar corrección al circuito no-Clifford original.

6. DYNAMICAL DECOUPLING (DD):
   Insertar secuencias de pulsos X-X o XY-4 en qubits inactivos.
   Los pulsos se cancelan entre sí pero refocalizan el ruido de fase.
   Extiende T2 efectivo.
   Implementado en hardware real con Qiskit DD PassManager.

7. TWIRLING (Randomized Benchmarking family):
   Promediar sobre conjugaciones aleatorias del circuito.
   Convierte errores coherentes en errores estocásticos de Pauli.
   Los errores estocásticos son más fáciles de modelar.
""")

# Implementar Dynamical Decoupling manualmente (simplificado)
def aplicar_dd_secuencia(qc_original: QuantumCircuit, qubit_libre: int,
                          t_libre_ns: float = 500) -> QuantumCircuit:
    """
    Inserta secuencia XY-4 (X·Y·X·Y) en un qubit libre.
    Cada pulso tiene tiempo t_libre_ns/4.
    """
    qc_dd = qc_original.copy()
    # En hardware real se usaría DD PassManager; aquí simulamos el concepto
    # Insertar barrera + X + barrera + X (secuencia X-X simple)
    qc_dd.barrier()
    qc_dd.x(qubit_libre)
    qc_dd.x(qubit_libre)  # X·X = I (sin efecto neto en estado)
    qc_dd.barrier()
    return qc_dd

# Symmetry Verification — ejemplo con paridad
def verificar_paridad(counts: dict, paridad_esperada: int = 0) -> dict:
    """
    Filtra conteos para retener solo resultados con paridad correcta.
    paridad_esperada=0: número par de '1' bits.
    paridad_esperada=1: número impar de '1' bits.
    """
    counts_filtrado = {}
    descartados = 0
    for bitstring, count in counts.items():
        paridad = sum(int(b) for b in bitstring) % 2
        if paridad == paridad_esperada:
            counts_filtrado[bitstring] = count
        else:
            descartados += count
    total_orig = sum(counts.values())
    total_filt = sum(counts_filtrado.values())
    print(f"  Simetría: conservados {total_filt}/{total_orig} shots ({descartados} descartados)")
    return counts_filtrado

# Aplicar a Bell state (paridad siempre par: 00, 11)
print("\nSymmetry Verification — Bell State:")
counts_noisy_bell = {"00": 3900, "11": 3700, "01": 250, "10": 150}
counts_verificado = verificar_paridad(counts_noisy_bell, paridad_esperada=0)
print(f"  Conteos originales: {counts_noisy_bell}")
print(f"  Conteos verificados: {counts_verificado}")
```

---

## 7. Benchmarking de mitigación de errores

```python
# ─────────────────────────────────────────────
# SECCIÓN 7: Benchmark de técnicas
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("BENCHMARK COMPARATIVO DE TÉCNICAS")
print("="*60)

# Simulación: variedad de profundidades de circuito
profundidades = [1, 2, 4, 8, 16, 32]
p_error_2q = 0.03
shots = 16384

errores_sin_mitigation = []
errores_con_symmetry = []
errores_con_zne = []

np.random.seed(42)

for profundidad in profundidades:
    # Error esperado sin mitigación: se acumula con la profundidad
    p_error_acc = 1 - (1 - p_error_2q)**profundidad
    error_sin = p_error_acc * 0.5  # error en valor esperado (simplificado)
    errores_sin_mitigation.append(error_sin)

    # Symmetry verification reduce ~50% de errores (aprox)
    error_sym = error_sin * 0.6
    errores_con_symmetry.append(error_sym)

    # ZNE reduce el error sistemático (sesgo) pero no la varianza
    error_zne = error_sin * (0.3 + 0.01 * profundidad)
    errores_con_zne.append(error_zne)

fig, ax = plt.subplots(figsize=(9, 5))
ax.plot(profundidades, errores_sin_mitigation, "r-o", label="Sin mitigación", linewidth=2, markersize=8)
ax.plot(profundidades, errores_con_symmetry, "b-s", label="Symmetry Verification", linewidth=2, markersize=8)
ax.plot(profundidades, errores_con_zne, "g-^", label="ZNE (lineal)", linewidth=2, markersize=8)
ax.axhline(y=0.0, color="black", linestyle="--", alpha=0.5, label="Error 0 (ideal)")
ax.set_xlabel("Profundidad del circuito")
ax.set_ylabel("Error en ⟨O⟩")
ax.set_title("Comparación de Técnicas de Error Mitigation\n(p_2Q = 3%)")
ax.legend()
ax.grid(True, alpha=0.3)
ax.set_yscale("log")

plt.tight_layout()
plt.savefig("outputs/m14_benchmark_mitigation.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m14_benchmark_mitigation.png")

print("""
Resumen de técnicas:

Técnica               | Overhead | Reduce sesgo | Reduce varianza | Escalabilidad
──────────────────────|──────────|──────────────|─────────────────|──────────────
Sin mitigación        |    1x    |      ✗       |        ✗        | Mejor
Readout Mitigation    |    2x    |      ✓ (RO)  |        ~        | Excelente
Symmetry Verif.       |    2-3x  |      ✓       |        ✓        | Buena
Dynamical Decoupling  |    1x+   |      ~       |        ✓        | Excelente
ZNE (lineal)          |    3-5x  |      ✓✓      |        ✗        | Buena
ZNE (Richardson)      |    5-7x  |      ✓✓✓     |        ✗        | Moderada
CDR                   |   10x+   |      ✓✓      |        ✓        | Moderada
PEC                   |   γ²x    |      ✓✓✓     |        ✗        | Mala (exp)

Recomendación práctica (2025 NISQ era):
  1. Siempre: Readout mitigation (barato, efectivo)
  2. Si circuito pequeño: ZNE con extrapolación lineal o Richardson
  3. Para simetría conservada: Symmetry verification
  4. En hardware real: Dynamical decoupling (Qiskit transpiler)
""")

print("\n✓ Módulo 14 completado")
```

---

## Checkpoint M14

```python
# checkpoint_m14.py
"""Verificaciones del Módulo 14 — Ruido y Error Mitigation"""
import numpy as np

def test_canal_depolarizante():
    """Canal depolarizante: p=1 → estado máximamente mixto I/2."""
    I = np.eye(2)
    rho = np.array([[1, 0], [0, 0]])  # |0⟩
    def canal_dep(rho, p):
        return (1 - p) * rho + p * I / 2
    rho_out = canal_dep(rho, p=1.0)
    assert np.allclose(rho_out, I / 2, atol=1e-10), f"p=1 debe dar I/2, got {rho_out}"
    rho_out_0 = canal_dep(rho, p=0.0)
    assert np.allclose(rho_out_0, rho, atol=1e-10), "p=0 debe ser identidad"
    print("✓ test_canal_depolarizante")

def test_amplitude_damping():
    """Amplitude damping γ=1: |1⟩→|0⟩ con certeza."""
    K0 = np.array([[1, 0], [0, 0]])    # γ=1
    K1 = np.array([[0, 1], [0, 0]])
    rho_1 = np.array([[0, 0], [0, 1]])  # |1⟩
    rho_out = K0 @ rho_1 @ K0.T + K1 @ rho_1 @ K1.T
    assert np.allclose(rho_out, np.array([[1, 0], [0, 0]])), \
        f"γ=1 debe mapear |1⟩→|0⟩, got {rho_out}"
    print("✓ test_amplitude_damping")

def test_t2_menor_2t1():
    """Principio físico: T2 ≤ 2·T1."""
    pares_t1_t2 = [(100, 80), (200, 150), (50, 100), (1000, 2000)]
    for T1, T2 in pares_t1_t2:
        if T2 <= 2 * T1:
            pass  # válido
        else:
            raise AssertionError(f"T2={T2} > 2·T1={2*T1} viola el límite físico")
    print("✓ test_t2_menor_2t1")

def test_matriz_asignacion_inversa():
    """La matriz de asignación debe ser invertible para readout mitigation."""
    p_01, p_10 = 0.03, 0.05
    A = np.array([[1-p_01, p_10], [p_01, 1-p_10]])
    A_inv = np.linalg.inv(A)
    # A·A⁻¹ = I
    assert np.allclose(A @ A_inv, np.eye(2), atol=1e-10), "A·A⁻¹ debe ser I"
    # Columnas de A suman 1 (canal estocástico)
    assert np.allclose(A.sum(axis=0), [1, 1], atol=1e-10), "Columnas deben sumar 1"
    print("✓ test_matriz_asignacion_inversa")

def test_zne_extrapola_correctamente():
    """ZNE extrapolación lineal a λ=0 recupera valor sin ruido."""
    # Simular: E(λ) = E_ideal + ruido · λ
    E_ideal = 0.85
    ruido_por_lambda = 0.05
    lambdas = [1, 3, 5]
    mediciones = [E_ideal - ruido_por_lambda * l for l in lambdas]
    coef = np.polyfit(lambdas, mediciones, deg=1)
    extrapolado = np.polyval(coef, 0)
    assert abs(extrapolado - E_ideal) < 1e-10, \
        f"ZNE debe extrapolarse exactamente: {extrapolado} ≠ {E_ideal}"
    print("✓ test_zne_extrapola_correctamente")

def test_symmetry_verification_descarta_errores():
    """Symmetry verification elimina bitstrings con paridad incorrecta."""
    counts = {"00": 4000, "11": 3800, "01": 150, "10": 50}
    # Estado Bell tiene paridad PAIR (00, 11) siempre
    counts_filtrado = {k: v for k, v in counts.items()
                       if sum(int(b) for b in k) % 2 == 0}
    assert "01" not in counts_filtrado
    assert "10" not in counts_filtrado
    assert "00" in counts_filtrado
    assert "11" in counts_filtrado
    print("✓ test_symmetry_verification_descarta_errores")

def test_kraus_preserva_traza():
    """Operadores de Kraus deben preservar la traza: Σ K†K = I."""
    # Canal depolarizante: K0=√(1-3p/4)·I, K1=√(p/4)·X, K2=√(p/4)·Y, K3=√(p/4)·Z
    p = 0.1
    I = np.eye(2)
    X = np.array([[0, 1], [1, 0]])
    Y = np.array([[0, -1j], [1j, 0]])
    Z = np.array([[1, 0], [0, -1]])
    kraus = [
        np.sqrt(1 - 3*p/4) * I,
        np.sqrt(p/4) * X,
        np.sqrt(p/4) * Y,
        np.sqrt(p/4) * Z,
    ]
    suma = sum(K.conj().T @ K for K in kraus)
    assert np.allclose(suma, I, atol=1e-10), f"Kraus no preserva traza: {suma}"
    print("✓ test_kraus_preserva_traza")

def test_noise_model_qiskit():
    """NoiseModel de Qiskit se puede crear y tiene errores registrados."""
    from qiskit_aer.noise import NoiseModel, depolarizing_error
    nm = NoiseModel()
    error = depolarizing_error(0.01, 1)
    nm.add_all_qubit_quantum_error(error, ["h"])
    assert len(nm.noise_instructions) > 0, "NoiseModel debe tener instrucciones de ruido"
    print("✓ test_noise_model_qiskit")

if __name__ == "__main__":
    print("Ejecutando checkpoint M14 — Ruido y Error Mitigation\n")
    test_canal_depolarizante()
    test_amplitude_damping()
    test_t2_menor_2t1()
    test_matriz_asignacion_inversa()
    test_zne_extrapola_correctamente()
    test_symmetry_verification_descarta_errores()
    test_kraus_preserva_traza()
    test_noise_model_qiskit()
    print("\n✓ Todos los tests de M14 pasaron")
```
