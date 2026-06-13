# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 10 — Transformada de Fourier Cuántica (QFT)

> **Objetivo:** Implementar la QFT desde cero, entender su relación con la DFT clásica, y usarla como bloque fundamental para QPE y el algoritmo de Shor. La QFT es la versión cuántica de la FFT clásica con ventaja exponencial en ciertos problemas.
>
> **Herramientas:** Qiskit + NumPy.
>
> **Prerequisito:** M09 — Algoritmo de Grover.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-10-qft.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-10-qft.ipynb
```

### Opción C — Google Colab
```python
!pip install qiskit qiskit-aer pylatexenc -q
```

---

## 10.1 La Transformada de Fourier Discreta y su Versión Cuántica

```python
"""
DFT Clásica:
  f̂_k = (1/√N) Σ_{j=0}^{N-1} f_j · e^{2πi·jk/N}
  Complejidad: O(N log N) con FFT

QFT Cuántica:
  QFT|j⟩ = (1/√N) Σ_{k=0}^{N-1} e^{2πi·jk/N} |k⟩
  Complejidad: O(n²) = O(log² N)  ← ventaja exponencial sobre DFT

La QFT opera en los vectores de estado cuántico (amplitudes),
no en datos clásicos directamente.
"""

import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.circuit.library import QFT
from qiskit.quantum_info import Operator
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc: QuantumCircuit) -> np.ndarray:
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

# Verificar QFT vs DFT clásica
def dft_clasica(x: np.ndarray) -> np.ndarray:
    """DFT directa O(N²)."""
    N = len(x)
    return np.array([
        np.sum(x * np.exp(2j * np.pi * k * np.arange(N) / N)) / np.sqrt(N)
        for k in range(N)
    ])

# Preparar un estado cuántico con amplitudes específicas
# |ψ⟩ = Σ f_j |j⟩
n = 3; N = 2**n
f = np.array([1.0, 0.5, 0.25, 0.0, 0.5, 0.25, 0.0, 0.1])
f = f / np.linalg.norm(f)   # normalizar

# Inicializar estado cuántico con esas amplitudes
qc_init = QuantumCircuit(n)
qc_init.initialize(f.tolist(), range(n))

# Aplicar QFT
from qiskit.circuit.library import QFT
qft_n = QFT(n, do_swaps=True)
qc_qft = qc_init.compose(qft_n)
sv_qft = sv(qc_qft)

# Calcular DFT clásica para comparar
dft = dft_clasica(f)

print("QFT vs DFT clásica:")
print(f"{'Índice':>8} {'|DFT(f)|':>12} {'|QFT(f)|':>12} {'Diferencia':>12}")
print("─" * 50)
for k in range(N):
    print(f"  k={k}:   {abs(dft[k]):>10.4f}   {abs(sv_qft[k]):>10.4f}   "
          f"{abs(abs(dft[k]) - abs(sv_qft[k])):>10.6f}")

print(f"\n¿QFT ≈ DFT? {np.allclose(np.abs(sv_qft), np.abs(dft), atol=1e-5)}")
```

---

## 10.2 Implementación Manual de la QFT

```python
def qft_manual(n: int, do_swaps: bool = True) -> QuantumCircuit:
    """
    Implementación manual de la QFT de n qubits.

    La QFT se construye con:
    - n puertas Hadamard
    - n(n-1)/2 puertas de fase controlada CR_k
    - n//2 puertas SWAP (para reordenar los bits)

    CR_k = [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,e^{2πi/2^k}]]
    """
    qc = QuantumCircuit(n, name=f"QFT-{n}")

    def qft_rotaciones(qc, n):
        if n == 0:
            return qc
        # Hadamard en el qubit más significativo (qubit 0 por convención)
        qc.h(n - 1)
        # Puertas de fase controladas CR_k
        for k in range(1, n):
            # e^{2πi/2^{k+1}} = e^{iπ/2^k}
            qc.cp(np.pi / 2**k, n - k - 1, n - 1)
        # Aplicar QFT recursivamente en los qubits restantes
        qft_rotaciones(qc, n - 1)
        return qc

    qft_rotaciones(qc, n)

    # SWAP para reordenar los qubits (QFT produce bits en orden inverso)
    if do_swaps:
        for i in range(n // 2):
            qc.swap(i, n - i - 1)

    return qc

# Verificar que la implementación manual coincide con la de Qiskit
for n_test in [2, 3, 4, 5]:
    qft_lib   = QFT(n_test, do_swaps=True)
    qft_manual_circ = qft_manual(n_test, do_swaps=True)

    M_lib    = Operator(qft_lib).data
    M_manual = Operator(qft_manual_circ).data

    fidelidad = abs(np.trace(M_lib.conj().T @ M_manual))**2 / (2**n_test)**2
    print(f"QFT-{n_test}: fidelidad manual vs Qiskit = {fidelidad:.8f}")
```

### Visualización de la estructura de la QFT

```python
for n_disp in [3, 4]:
    qft_disp = qft_manual(n_disp, do_swaps=True)
    print(f"\nQFT-{n_disp} (implementación manual):")
    print(qft_disp.draw())
    print(f"  Profundidad: {qft_disp.depth()}")
    print(f"  Puertas: {dict(qft_disp.count_ops())}")
```

---

## 10.3 La QFT en Acción — Detección de Periodicity

```python
# La QFT convierte una función periódica en su espectro de frecuencias
# Si f(x) tiene período r, su QFT tiene picos en los múltiplos de N/r

def estado_periodico(n: int, periodo: int, fase: float = 0.0) -> np.ndarray:
    """
    Crea un estado cuántico con amplitudes periódicas.
    |ψ⟩ ∝ Σ_{k=0}^{N/r-1} |k·r⟩
    """
    N = 2**n
    amplitudes = np.zeros(N, dtype=complex)
    for k in range(0, N, periodo):
        amplitudes[k] = np.exp(1j * fase * k)
    return amplitudes / np.linalg.norm(amplitudes)

n = 4; N = 2**n

# Probar con distintos períodos
fig, axes = plt.subplots(2, 3, figsize=(15, 8))

for ax_pair, periodo in zip(zip(axes[0], axes[1]), [2, 4, 8]):
    ax_time, ax_freq = ax_pair
    # Estado periódico
    amps = estado_periodico(n, periodo)

    # Aplicar QFT
    qc_p = QuantumCircuit(n)
    qc_p.initialize(amps.tolist(), range(n))
    qc_p.compose(qft_manual(n), inplace=True)
    sv_qft_p = sv(qc_p)

    # Graficar amplitudes antes y después de QFT
    ax_time.bar(range(N), np.abs(amps)**2, color='steelblue')
    ax_time.set_title(f"Estado periódico (r={periodo})")
    ax_time.set_xlabel("|x⟩"); ax_time.set_ylabel("P(|x⟩)")

    ax_freq.bar(range(N), np.abs(sv_qft_p)**2, color='coral')
    ax_freq.set_title(f"QFT(r={periodo}): picos en múltiplos de {N//periodo}")
    ax_freq.set_xlabel("|k⟩"); ax_freq.set_ylabel("P(|k⟩)")

    # Verificar picos en los lugares esperados
    picos_esperados = list(range(0, N, N // periodo))
    for p in picos_esperados:
        ax_freq.axvline(p, color='green', linestyle='--', alpha=0.5)

plt.suptitle("QFT detecta la periodicidad: antes y después", fontsize=12)
plt.tight_layout()
plt.savefig("outputs/qft_periodicidad.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 10.4 QFT−1 (IQFT) — Transformada Inversa

```python
def iqft_manual(n: int, do_swaps: bool = True) -> QuantumCircuit:
    """
    Transformada de Fourier Cuántica Inversa.
    IQFT = QFT† (adjunta de la QFT).
    """
    return qft_manual(n, do_swaps).inverse()

# QFT · IQFT = Identidad
n_test = 4
qft_c  = qft_manual(n_test)
iqft_c = iqft_manual(n_test)

# Verificar que QFT · IQFT = I
qc_round = QuantumCircuit(n_test)
qc_round.compose(qft_c,  inplace=True)
qc_round.compose(iqft_c, inplace=True)

M_round = Operator(qc_round).data
print(f"QFT · IQFT = I: {np.allclose(M_round, np.eye(2**n_test), atol=1e-8)}")

# Uso práctico: preparar estado de frecuencia y recuperar el tiempo
# |k=3⟩ debería dar el período r=N/gcd(k,N) al aplicar IQFT
n_iqft = 4; N = 2**n_iqft
k_freq = 4   # frecuencia

qc_iqft_demo = QuantumCircuit(n_iqft)
# Preparar estado |k_freq⟩
bits_k = format(k_freq, f'0{n_iqft}b')
for i, bit in enumerate(bits_k[::-1]):
    if bit == '1':
        qc_iqft_demo.x(i)

qc_iqft_demo.compose(iqft_manual(n_iqft), inplace=True)
sv_iqft = sv(qc_iqft_demo)

print(f"\nIQFT|{k_freq}⟩ en espacio de tiempo:")
for j, amp in enumerate(sv_iqft):
    if abs(amp)**2 > 0.01:
        print(f"  |{j}⟩: {amp:.4f}  P={abs(amp)**2:.4f}")
print(f"Período esperado: r = N/k = {N}/{k_freq} = {N//k_freq}")
```

---

## 10.5 Comparación QFT vs FFT Clásica

```python
import time

def benchmark_qft_fft(n_max: int = 12):
    """Compara QFT cuántica (simulada) vs FFT clásica."""
    resultados = {'n': [], 't_qft': [], 't_fft': [], 'gates_qft': []}

    for n in range(2, n_max + 1):
        N = 2**n
        # Señal aleatoria
        x = np.random.randn(N) + 1j * np.random.randn(N)
        x /= np.linalg.norm(x)

        # FFT clásica
        t0 = time.perf_counter()
        for _ in range(10):
            _ = np.fft.fft(x) / np.sqrt(N)
        t_fft = (time.perf_counter() - t0) / 10 * 1000

        # QFT cuántica (simulación)
        qft_circ = qft_manual(n)
        t0 = time.perf_counter()
        qc_bench = QuantumCircuit(n)
        qc_bench.initialize(x.tolist(), range(n))
        qc_bench.compose(qft_circ, inplace=True)
        _ = sv(qc_bench)
        t_qft = (time.perf_counter() - t0) * 1000

        n_gates = qft_circ.size()
        resultados['n'].append(n)
        resultados['t_qft'].append(t_qft)
        resultados['t_fft'].append(t_fft)
        resultados['gates_qft'].append(n_gates)

    return resultados

r = benchmark_qft_fft(n_max=10)
print("Benchmark QFT vs FFT:")
print(f"{'n':>4} {'N=2^n':>8} {'FFT (ms)':>10} {'QFT-sim (ms)':>14} {'Puertas QFT':>13}")
print("─" * 55)
for i in range(len(r['n'])):
    print(f"  {r['n'][i]:>2}  {2**r['n'][i]:>8}  {r['t_fft'][i]:>9.3f}  "
          f"{r['t_qft'][i]:>13.3f}  {r['gates_qft'][i]:>12}")

print("\nNota: la QFT simulada es más lenta (incluye overhead de simulación).")
print("En hardware cuántico real, la QFT necesita solo O(n²) pasos.")
```

---

## 10.6 QFT para Estimación de Fases — Preview de QPE

```python
# La QFT es el corazón del algoritmo QPE (Quantum Phase Estimation)
# QPE: dado U|ψ⟩ = e^{2πiφ}|ψ⟩, estimar φ con t bits de precisión

def qft_single_qubit(phi: float) -> float:
    """
    Demostración 1D del principio de QPE:
    Si el estado de entrada es |k/N⟩ en el dominio de frecuencia,
    la IQFT devuelve |k⟩.
    """
    # Preparar estado |φ·N⟩ en superposición (fase codificada en amplitudes)
    n = 4; N = 2**n

    # phi_estimar = 0.375 = 3/8 → queremos recuperar k=6 (= 0.375 · 16 = 6)
    phi_real = phi
    k_real = phi_real * N

    amplitudes = np.zeros(N, dtype=complex)
    amplitudes[int(round(k_real)) % N] = 1.0   # estado base puro en frecuencia

    # Aplicar IQFT para ir al dominio de tiempo (registros de fase)
    qc_phi = QuantumCircuit(n)
    qc_phi.initialize(amplitudes.tolist(), range(n))
    qc_phi.compose(iqft_manual(n), inplace=True)
    sv_phi = sv(qc_phi)

    # La IQFT debería dar un estado periódico con período N/k_real
    # Al medir, obtenemos k_real → phi = k/N
    mejor_idx = np.argmax(np.abs(sv_phi)**2)
    phi_estimada = mejor_idx / N
    return phi_estimada

# Estimar varias fases
fases_test = [0.25, 0.375, 0.5, 0.125, 0.75]
print("Preview QPE — estimación de fases con QFT/IQFT:")
for phi in fases_test:
    phi_est = qft_single_qubit(phi)
    print(f"  φ real = {phi:.4f}, φ estimada = {phi_est:.4f}  "
          f"{'✓' if abs(phi - phi_est) < 0.01 else '~'}")
```

---

## 10.7 QFT Aproximada — Para Hardware Real

```python
# En hardware real, las puertas de fase muy pequeñas (CR_k para k grande)
# tienen poco efecto y pueden eliminarse sin pérdida significativa

def qft_aproximada(n: int, grado_aprox: int, do_swaps: bool = True) -> QuantumCircuit:
    """
    QFT aproximada: elimina las puertas de fase con k > grado_aprox.
    grado_aprox=n → QFT exacta
    grado_aprox=2 → solo Hadamard + CR_1 por qubit (muy reducida)
    """
    qc = QuantumCircuit(n, name=f"QFT-approx-{grado_aprox}")

    def qft_rotaciones_aprox(qc, qubit, n_total):
        qc.h(qubit)
        for k in range(1, min(grado_aprox, n_total - qubit)):
            target = qubit + k
            if target < n_total:
                qc.cp(np.pi / 2**k, target, qubit)

    for q in range(n - 1, -1, -1):
        qft_rotaciones_aprox(qc, q, n)

    if do_swaps:
        for i in range(n // 2):
            qc.swap(i, n - i - 1)

    return qc

# Comparar fidelidad de QFT exacta vs aproximada para n=8
n_aprox = 8
qft_exacta  = QFT(n_aprox, do_swaps=True)
M_exacta = Operator(qft_exacta).data

print("Fidelidad QFT exacta vs aproximada (n=8):")
for grado in range(1, n_aprox + 1):
    qft_ap = qft_aproximada(n_aprox, grado)
    M_ap   = Operator(qft_ap).data
    fid    = abs(np.trace(M_exacta.conj().T @ M_ap))**2 / (2**n_aprox)**2
    gates  = qft_ap.count_ops()
    n_cp   = gates.get('cp', 0)
    print(f"  grado={grado}: fidelidad={fid:.6f}, puertas CP={n_cp}")

# Las fases más pequeñas que ~π/1024 no son físicamente distinguibles
# En hardware real con ~ 1% error de puerta, grado 3-4 es suficiente
```

---

## 10.8 Circuito QFT desde la Librería de Qiskit

```python
from qiskit.circuit.library import QFT

# Opciones de QFT en Qiskit
qft_exacta_lib  = QFT(5, approximation_degree=0, do_swaps=True)
qft_aprox_lib   = QFT(5, approximation_degree=2, do_swaps=True)
qft_sin_swap    = QFT(5, approximation_degree=0, do_swaps=False)
qft_inversa_lib = QFT(5, inverse=True)

print("QFT-5 exacta:")
print(qft_exacta_lib.decompose().draw(fold=80))

print(f"\nComparación de variantes QFT-5:")
print(f"{'Variante':30} {'Depth':>8} {'Gates':>8}")
print("─" * 50)
for nombre, qft_var in [
    ("Exacta (con SWAP)",      qft_exacta_lib),
    ("Aprox. grado=2",         qft_aprox_lib),
    ("Sin SWAP",               qft_sin_swap),
    ("Inversa",                qft_inversa_lib),
]:
    qft_dec = qft_var.decompose()
    print(f"  {nombre:28}: depth={qft_dec.depth():>4}, gates={qft_dec.size():>5}")
```

---

## checkpoint_m10.py

```python
"""
Checkpoint Módulo 10 — Transformada de Fourier Cuántica (QFT)
Ejecutar con: python checkpoint_m10.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.circuit.library import QFT
from qiskit.quantum_info import Operator
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

def qft_manual(n, do_swaps=True):
    qc = QuantumCircuit(n, name=f"QFT-{n}")
    def rot(qc, n):
        if n == 0: return
        qc.h(n - 1)
        for k in range(1, n):
            qc.cp(np.pi / 2**k, n - k - 1, n - 1)
        rot(qc, n - 1)
    rot(qc, n)
    if do_swaps:
        for i in range(n // 2):
            qc.swap(i, n - i - 1)
    return qc

print("=" * 55)
print("Checkpoint M10 — QFT")
print("=" * 55)

# ── Test 1: QFT-2 coincide con la librería Qiskit ────────
for n in [2, 3, 4]:
    qft_lib = QFT(n, do_swaps=True)
    qft_man = qft_manual(n, do_swaps=True)
    M_lib = Operator(qft_lib).data
    M_man = Operator(qft_man).data
    fid = abs(np.trace(M_lib.conj().T @ M_man))**2 / (2**n)**2
    assert fid > 0.999, f"FALLO: QFT-{n} fidelidad={fid:.6f}"
    print(f"✓ Test 1a: QFT-{n} manual ≈ librería (fidelidad={fid:.8f})")

# ── Test 2: QFT es unitaria ───────────────────────────────
for n in [2, 3, 4, 5]:
    qft_u = QFT(n)
    M = Operator(qft_u).data
    assert np.allclose(M.conj().T @ M, np.eye(2**n), atol=1e-8), \
        f"FALLO: QFT-{n} no es unitaria"
print("✓ Test 2: QFT es unitaria para n=2,3,4,5")

# ── Test 3: QFT · QFT† = I ───────────────────────────────
n = 4
qft_c = qft_manual(n)
iqft_c = qft_c.inverse()
qc_r = QuantumCircuit(n)
qc_r.compose(qft_c,  inplace=True)
qc_r.compose(iqft_c, inplace=True)
M_r = Operator(qc_r).data
assert np.allclose(M_r, np.eye(2**n), atol=1e-8), "FALLO: QFT·QFT† ≠ I"
print(f"✓ Test 3: QFT · IQFT = I (n={n})")

# ── Test 4: QFT detecta periodicidad ─────────────────────
n_p = 4; N_p = 2**n_p
for periodo in [2, 4, 8]:
    amps = np.zeros(N_p, dtype=complex)
    for k in range(0, N_p, periodo):
        amps[k] = 1.0
    amps /= np.linalg.norm(amps)

    qc_p = QuantumCircuit(n_p)
    qc_p.initialize(amps.tolist(), range(n_p))
    qc_p.compose(qft_manual(n_p), inplace=True)
    sv_qft = sv(qc_p)

    # Los picos deben estar en múltiplos de N/periodo
    picos = [k for k in range(N_p) if abs(sv_qft[k])**2 > 0.01]
    picos_esperados = list(range(0, N_p, N_p // periodo))
    assert set(picos) == set(picos_esperados), \
        f"FALLO: picos período={periodo}: {picos} vs esperados {picos_esperados}"
    print(f"✓ Test 4a: QFT período={periodo} → picos en {picos_esperados}")

# ── Test 5: QFT coincide con DFT clásica (magnitudes) ────
def dft(x):
    N = len(x)
    return np.array([np.sum(x * np.exp(2j*np.pi*k*np.arange(N)/N)) / np.sqrt(N)
                     for k in range(N)])

n_dft = 3
f = np.random.randn(2**n_dft) + 1j * np.random.randn(2**n_dft)
f /= np.linalg.norm(f)
dft_result = dft(f)

qc_dft = QuantumCircuit(n_dft)
qc_dft.initialize(f.tolist(), range(n_dft))
qc_dft.compose(qft_manual(n_dft), inplace=True)
sv_dft = sv(qc_dft)

assert np.allclose(np.abs(sv_dft), np.abs(dft_result), atol=1e-5), \
    f"FALLO: |QFT| ≠ |DFT|"
print(f"✓ Test 5: |QFT(f)| = |DFT(f)| para estado aleatorio n=3")

# ── Test 6: QFT-8 tiene el número correcto de puertas ────
qft8 = qft_manual(8)
n_h  = qft8.count_ops().get('h', 0)
n_cp = qft8.count_ops().get('cp', 0)
assert n_h == 8, f"FALLO: QFT-8 debe tener 8 puertas H, got {n_h}"
assert n_cp == 8*7//2, f"FALLO: QFT-8 debe tener {8*7//2} CP, got {n_cp}"
print(f"✓ Test 6: QFT-8 tiene {n_h} H y {n_cp} CP (correcto)")

# ── Test 7: QFT|0⟩ = superposición uniforme ──────────────
n_u = 4
qc_u = QuantumCircuit(n_u)
qc_u.compose(qft_manual(n_u), inplace=True)
sv_u = sv(qc_u)
probs_u = np.abs(sv_u)**2
assert np.allclose(probs_u, np.ones(2**n_u) / 2**n_u, atol=1e-6), \
    f"FALLO: QFT|0⟩ debe ser superposición uniforme"
print(f"✓ Test 7: QFT|0⟩ = superposición uniforme (P = {probs_u[0]:.4f} para cada estado)")

print("\n" + "=" * 55)
print("✅ Todos los tests del M10 pasaron correctamente")
print("=" * 55)
```
