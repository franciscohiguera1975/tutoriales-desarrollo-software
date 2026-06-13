# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 9 — Algoritmo de Grover: Búsqueda Cuadrática

> **Objetivo:** Implementar el algoritmo de Grover desde cero, entender la amplificación de amplitud como mecanismo de búsqueda, y aplicarlo a problemas reales (SAT, búsqueda en base de datos, k-clique).
>
> **Herramientas:** Qiskit + NumPy.
>
> **Prerequisito:** M08 — Deutsch-Jozsa y Bernstein-Vazirani.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-09-grover.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-09-grover.ipynb
```

### Opción C — Google Colab
```python
!pip install qiskit qiskit-aer pylatexenc -q
```

---

## 9.1 La Idea de Grover — Amplificación de Amplitud

```python
"""
PROBLEMA: dada una base de datos de N=2^n elementos no ordenada,
          encontrar el elemento(s) que satisface f(x) = 1.

CLÁSICO: O(N) consultas esperadas (busca en promedio N/2 elementos).
CUÁNTICO: O(√N) consultas — ventaja CUADRÁTICA.

MECANISMO: Amplificación de Amplitud
  1. Preparar superposición uniforme: todos los estados tienen amplitud 1/√N
  2. Oráculo de fase: invierte el signo de los estados solución
  3. Difusor de Grover: "refleja" en torno al promedio de amplitudes
  4. Repetir ~π√N/4 veces → la amplitud del/los estados solución crece
"""

from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.quantum_info import Statevector
import numpy as np
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc: QuantumCircuit) -> np.ndarray:
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

# Visualización del efecto de amplificación paso a paso
def visualizar_amplificacion(n: int = 3, objetivo: int = 5):
    """
    Muestra cómo las amplitudes evolucionan con cada iteración de Grover.
    """
    N = 2**n
    amplitudes = np.ones(N) / np.sqrt(N)   # superposición uniforme

    n_iters = int(np.pi / 4 * np.sqrt(N))

    fig, axes = plt.subplots(1, n_iters + 1, figsize=(4 * (n_iters + 1), 3))
    colores = ['coral' if i == objetivo else 'steelblue' for i in range(N)]

    ax = axes[0]
    ax.bar(range(N), amplitudes, color=colores)
    ax.set_title("Inicial\n(superposición)")
    ax.set_ylim(-0.6, 0.9)
    ax.axhline(0, color='black', linewidth=0.5)
    ax.set_ylabel("Amplitud")

    for it in range(n_iters):
        # Oráculo: invertir signo del objetivo
        amplitudes[objetivo] *= -1

        # Difusor: reflexión en torno al promedio
        promedio = np.mean(amplitudes)
        amplitudes = 2 * promedio - amplitudes
        amplitudes[objetivo] = 2 * promedio - amplitudes[objetivo]

        # Revertir la inversión del oráculo para el difusor
        # (el difusor hace: 2|s⟩⟨s| - I donde s es la superposición uniforme)
        # Corrección: reiniciar correctamente
        amplitudes_orig = amplitudes.copy()
        amplitudes[objetivo] *= -1
        promedio = np.mean(amplitudes)
        amplitudes = 2 * promedio - amplitudes

        ax = axes[it + 1]
        ax.bar(range(N), amplitudes, color=colores)
        ax.set_title(f"Iter {it+1}")
        ax.set_ylim(-0.6, 0.9)
        ax.axhline(0, color='black', linewidth=0.5)
        ax.axhline(float(np.mean(amplitudes)), color='green', linestyle='--',
                   linewidth=1, alpha=0.6, label="media")

    plt.suptitle(f"Amplificación de Amplitud (n={n}, objetivo={objetivo})", fontsize=11)
    plt.tight_layout()
    plt.savefig("outputs/grover_amplificacion.png", dpi=150, bbox_inches='tight')
    plt.show()

visualizar_amplificacion(n=3, objetivo=5)
```

---

## 9.2 Implementación del Oráculo de Grover

```python
def grover_oracle(n: int, objetivo: int) -> QuantumCircuit:
    """
    Oráculo de fase para el estado objetivo.
    O_f|objetivo⟩ = -|objetivo⟩
    O_f|x⟩ = |x⟩ para x ≠ objetivo

    Implementación: multi-controlled-Z en los qubits activos en 'objetivo'.
    """
    qc = QuantumCircuit(n, name=f"Oracle({objetivo})")
    # Convertir el objetivo a string de bits (invertir para Qiskit)
    bits_objetivo = format(objetivo, f'0{n}b')[::-1]

    # Aplicar X en qubits donde el bit del objetivo es 0
    # (para que el estado objetivo sea |11...1⟩ antes del multi-CZ)
    for i, bit in enumerate(bits_objetivo):
        if bit == '0':
            qc.x(i)

    # Multi-controlled-Z (MCZ): aplica fase -1 a |11...1⟩
    if n == 1:
        qc.z(0)
    elif n == 2:
        qc.cz(0, 1)
    else:
        # Usar H + MCX + H para implementar MCZ
        qc.h(n - 1)
        qc.mcx(list(range(n - 1)), n - 1)   # multi-controlled-X
        qc.h(n - 1)

    # Revertir las X aplicadas antes
    for i, bit in enumerate(bits_objetivo):
        if bit == '0':
            qc.x(i)

    return qc

def grover_difusor(n: int) -> QuantumCircuit:
    """
    Difusor de Grover: D = H⊗n · (2|0⟩⟨0| - I) · H⊗n
    Equivalente a reflexión en torno al estado de superposición uniforme.
    """
    qc = QuantumCircuit(n, name="Difusor")
    qc.h(range(n))
    # (2|0⟩⟨0| - I): marca el estado |0...0⟩ con fase -1
    qc.x(range(n))
    qc.h(n - 1)
    qc.mcx(list(range(n - 1)), n - 1)
    qc.h(n - 1)
    qc.x(range(n))
    qc.h(range(n))
    return qc

def algoritmo_grover(n: int, objetivo: int, n_iter: int = None) -> dict:
    """
    Algoritmo de Grover completo.
    n: número de qubits (busca en 2^n elementos)
    objetivo: índice del elemento a encontrar
    n_iter: número de iteraciones (por defecto óptimo = π√N/4)
    """
    N = 2**n
    if n_iter is None:
        n_iter = max(1, int(np.pi / 4 * np.sqrt(N)))

    oraculo = grover_oracle(n, objetivo)
    difusor  = grover_difusor(n)

    qc = QuantumCircuit(n, n, name="Grover")
    # 1. Superposición uniforme
    qc.h(range(n))
    qc.barrier()

    # 2. n_iter iteraciones (Oráculo + Difusor)
    for _ in range(n_iter):
        qc.compose(oraculo,  inplace=True)
        qc.compose(difusor,  inplace=True)
        qc.barrier()

    # 3. Medición
    qc.measure(range(n), range(n))

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=4096).result().get_counts()
    # Convertir a probabilidades
    total = sum(conteos.values())
    probs = {int(k[::-1], 2): v / total for k, v in conteos.items()}

    return {
        'conteos':    conteos,
        'probs':      probs,
        'mejor':      max(probs, key=probs.get),
        'prob_mejor': probs.get(objetivo, 0),
        'n_iter':     n_iter,
        'circuito':   qc,
    }
```

---

## 9.3 Demostración con Distintos Tamaños

```python
print("Algoritmo de Grover — Búsqueda en 2^n elementos:")
print("─" * 60)
print(f"{'n':>4} {'N=2^n':>8} {'n_iter':>7} {'P(objetivo)':>12} {'✓?':>4}")
print("─" * 60)

for n in [2, 3, 4, 5, 6]:
    N = 2**n
    objetivo = N // 3   # elegir un objetivo "no trivial"
    resultado = algoritmo_grover(n, objetivo)
    ok = "✓" if resultado['mejor'] == objetivo else "✗"
    print(f"  {n:>2}  {N:>8}  {resultado['n_iter']:>7}  "
          f"{resultado['prob_mejor']:>11.4f}  {ok:>3}")

# Visualizar para n=4
res_4 = algoritmo_grover(4, objetivo=9)
probs_array = np.zeros(16)
for idx, p in res_4['probs'].items():
    probs_array[idx] = p

fig, ax = plt.subplots(figsize=(10, 4))
colores = ['coral' if i == 9 else 'steelblue' for i in range(16)]
ax.bar(range(16), probs_array, color=colores)
ax.set_xlabel("Estado |x⟩"); ax.set_ylabel("Probabilidad")
ax.set_title(f"Grover n=4: búsqueda del estado |9⟩ ({res_4['n_iter']} iteraciones)")
ax.axhline(1/16, color='gray', linestyle='--', linewidth=1, label="Uniforme (sin Grover)")
ax.legend()
plt.tight_layout()
plt.savefig("outputs/grover_resultado.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 9.4 Número Óptimo de Iteraciones

```python
# Las iteraciones de Grover hacen oscilar la probabilidad como un seno²
# Máximo en k = π√N/4, luego decrece (¡no hay que pasarse!)

def probabilidad_grover_analitica(N: int, k: int) -> float:
    """P(objetivo después de k iteraciones) = sin²((2k+1)·arcsin(1/√N))."""
    theta = np.arcsin(1 / np.sqrt(N))
    return np.sin((2 * k + 1) * theta)**2

fig, axes = plt.subplots(1, 3, figsize=(15, 4))
for ax, n in zip(axes, [4, 6, 8]):
    N = 2**n
    k_max = int(np.pi * np.sqrt(N) / 2)   # 2× el óptimo para ver oscilación
    ks = range(k_max + 1)
    probs_analiticas = [probabilidad_grover_analitica(N, k) for k in ks]

    # Simular experimentalmente para algunos valores de k
    objetivo = N // 4
    probs_exp = []
    for k in range(0, min(k_max + 1, 25)):
        r = algoritmo_grover(n, objetivo, n_iter=k)
        probs_exp.append(r['prob_mejor'])

    ax.plot(list(ks), probs_analiticas, 'b-', label='Analítico', linewidth=2)
    ax.plot(range(min(k_max+1, 25)), probs_exp, 'ro', label='Simulado', markersize=4)
    ax.axvline(np.pi * np.sqrt(N) / 4, color='green', linestyle='--',
               linewidth=1, alpha=0.7, label=f'k_opt≈{np.pi*np.sqrt(N)/4:.1f}')
    ax.set_xlabel("Iteraciones k"); ax.set_ylabel("P(objetivo)")
    ax.set_title(f"n={n}, N={N}")
    ax.legend(fontsize=8); ax.grid(alpha=0.3)
    ax.set_ylim(0, 1.05)

plt.suptitle("Oscilación de probabilidad en Grover", fontsize=12)
plt.tight_layout()
plt.savefig("outputs/grover_oscilacion.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 9.5 Múltiples Soluciones

```python
# Si hay M soluciones de N posibles, el número óptimo de iteraciones es:
# k_opt ≈ (π/4) · √(N/M)
# La probabilidad de encontrar UNA de las M soluciones es ≈ 1

def grover_oracle_multiple(n: int, objetivos: list) -> QuantumCircuit:
    """Oráculo para múltiples estados objetivo."""
    qc = QuantumCircuit(n, name=f"Oracle(M={len(objetivos)})")
    for obj in objetivos:
        bits = format(obj, f'0{n}b')[::-1]
        # Aplicar X en los qubits con bit=0
        qubits_flip = [i for i, b in enumerate(bits) if b == '0']
        qc.x(qubits_flip)
        # MCZ
        if n == 1:
            qc.z(0)
        elif n == 2:
            qc.cz(0, 1)
        else:
            qc.h(n-1); qc.mcx(list(range(n-1)), n-1); qc.h(n-1)
        # Revertir X
        qc.x(qubits_flip)
    return qc

n = 4; N = 2**n
objetivos = [3, 7, 11]   # M=3 soluciones de 16
M = len(objetivos)

k_opt_multiple = max(1, int(np.pi / 4 * np.sqrt(N / M)))
oraculo_m = grover_oracle_multiple(n, objetivos)
difusor_m  = grover_difusor(n)

qc_multi = QuantumCircuit(n, n)
qc_multi.h(range(n))
for _ in range(k_opt_multiple):
    qc_multi.compose(oraculo_m, inplace=True)
    qc_multi.compose(difusor_m, inplace=True)
qc_multi.measure(range(n), range(n))

conteos_multi = sim_qasm.run(transpile(qc_multi, sim_qasm), shots=4096).result().get_counts()
total_m = sum(conteos_multi.values())

print(f"\nGrover con M={M} soluciones (N={N}, k_opt={k_opt_multiple}):")
prob_total_solucion = sum(conteos_multi.get(format(o, f'0{n}b')[::-1], 0) / total_m
                          for o in objetivos)
print(f"  P(cualquier solución) = {prob_total_solucion:.4f}")
print("  Distribución:")
for obj in objetivos:
    bits_key = format(obj, f'0{n}b')[::-1]
    p = conteos_multi.get(bits_key, 0) / total_m
    print(f"    Solución |{obj}⟩ = {obj:04b}: P = {p:.4f}")
```

---

## 9.6 Grover para SAT — 3-SAT

```python
# Grover puede resolver 3-SAT cuadráticamente más rápido que búsqueda clásica
# 3-SAT: encontrar asignación de variables que satisfaga todas las cláusulas

def oracle_3sat(n_vars: int, clausulas: list) -> QuantumCircuit:
    """
    Oráculo para 3-SAT.
    clausulas: lista de [(var1, neg1), (var2, neg2), (var3, neg3)]
    neg=True significa que la variable aparece negada en la cláusula.
    """
    n_clauses = len(clausulas)
    # Necesitamos qubits ancilla para las cláusulas
    n_total = n_vars + n_clauses + 1   # vars + ancillas_cláusula + ancilla_final
    qc = QuantumCircuit(n_total, name="3-SAT Oracle")

    # Paso 1: evaluar cada cláusula en ancillas de cláusula
    for c_idx, clausula in enumerate(clausulas):
        q_anc = n_vars + c_idx
        # Una cláusula (a ∨ b ∨ c) se satisface si NOT(¬a ∧ ¬b ∧ ¬c)
        # Implementar como: flip ancilla si TODOS los literales son 0
        qubits_neg = []
        for (var, negada) in clausula:
            if negada:
                qc.x(var)       # temporalmente invertir (literal negado)
                qubits_neg.append(var)
        # ancilla = 1 si la cláusula NO se satisface (todos los literales falsos)
        qc.ccx(clausula[0][0], clausula[1][0], q_anc)  # simplificado a 3-qubit
        for var in qubits_neg:
            qc.x(var)   # revertir

    # Paso 2: marcar si TODAS las cláusulas son verdaderas
    q_final = n_vars + n_clauses
    qc.mcx(list(range(n_vars, n_vars + n_clauses)), q_final)

    # Paso 3 (phase kickback): ancilla_final en |−⟩ da la fase -1
    # (se inicializa externamente)

    # Paso 4: revertir evaluación de cláusulas (limpieza)
    for c_idx, clausula in enumerate(clausulas):
        q_anc = n_vars + c_idx
        qubits_neg = []
        for (var, negada) in clausula:
            if negada:
                qc.x(var)
                qubits_neg.append(var)
        qc.ccx(clausula[0][0], clausula[1][0], q_anc)
        for var in qubits_neg:
            qc.x(var)

    return qc

# Ejemplo simple: 2-SAT con 3 variables
# f = (x0 ∨ x1) ∧ (¬x1 ∨ x2)
# Soluciones: necesitamos verificar manualmente para el ejemplo

def buscar_sat_grover_simple(n: int, objetivo: int) -> dict:
    """Búsqueda simple con oráculo de fase básico (verifica una asignación)."""
    return algoritmo_grover(n, objetivo)

# Para un ejemplo educativo, usar el oráculo básico de Grover
# El oráculo SAT completo con ancillas se implementa en módulos avanzados
print("Grover SAT — Búsqueda de asignación satisfactoria:")
print("(Para el ejemplo, usamos un oráculo simplificado que marca un único estado)")
n_vars = 4
solucion = 0b1010   # la "asignación" que satisface la fórmula
resultado_sat = algoritmo_grover(n_vars, solucion)
print(f"  Variables: {n_vars}, Solución: {solucion:0{n_vars}b}")
print(f"  Encontrada: {resultado_sat['mejor']:0{n_vars}b}")
print(f"  Probabilidad: {resultado_sat['prob_mejor']:.4f}")
print(f"  Iteraciones: {resultado_sat['n_iter']} (vs {2**n_vars} en búsqueda clásica)")
```

---

## 9.7 Grover desde la Librería de Qiskit

```python
from qiskit.circuit.library import GroverOperator, PhaseOracle

# ── PhaseOracle: oráculo a partir de expresión booleana ───
# Requiere el paquete tweedledum
try:
    from qiskit.circuit.library import PhaseOracle

    # Expresión booleana: (a AND b) OR (NOT a AND c)
    expresion = "(a & b) | (~a & c)"
    oraculo_bool = PhaseOracle(expresion)
    print(f"PhaseOracle para '{expresion}':")
    print(oraculo_bool.draw())

    # Combinar con GroverOperator
    grover_op = GroverOperator(oraculo_bool)
    print(f"\nGroverOperator: {grover_op.num_qubits} qubits")
except Exception as e:
    print(f"PhaseOracle no disponible: {e}")
    print("Instalar: pip install qiskit[optimization]")

# ── GroverOperator con oráculo manual ─────────────────────
from qiskit.circuit.library import GroverOperator

n_demo = 3
objetivo_demo = 5
oraculo_demo = grover_oracle(n_demo, objetivo_demo)

# Añadir reflexión en 0 para usarlo con GroverOperator
grover_op_manual = GroverOperator(oraculo_demo)
print(f"\nGroverOperator manual: depth={grover_op_manual.decompose().depth()}")

qc_lib = QuantumCircuit(n_demo, n_demo)
qc_lib.h(range(n_demo))
for _ in range(int(np.pi/4 * np.sqrt(2**n_demo))):
    qc_lib.compose(grover_op_manual, inplace=True)
qc_lib.measure(range(n_demo), range(n_demo))

conteos_lib = sim_qasm.run(transpile(qc_lib, sim_qasm), shots=2048).result().get_counts()
total_l = sum(conteos_lib.values())
mejor_l = max(conteos_lib, key=conteos_lib.get)
mejor_idx_l = int(mejor_l[::-1], 2)
print(f"GroverOperator (Qiskit library): encontrado={mejor_idx_l} "
      f"(objetivo={objetivo_demo}), P={conteos_lib[mejor_l]/total_l:.4f}")
```

---

## 9.8 Análisis de Complejidad y Limitaciones

```python
"""
Resumen del algoritmo de Grover:

✓ VENTAJA:
  - Cuadrática: O(√N) consultas vs O(N) clásico
  - Aplicable a cualquier problema de búsqueda no estructurada
  - Componente del algoritmo de Shor y otros algoritmos avanzados

✗ LIMITACIONES:
  - Solo ventaja cuadrática (no exponencial como Shor)
  - Requiere conocer M (número de soluciones) para k_opt
  - No aplica cuando el problema tiene estructura adicional
  - En la práctica, el costo del circuito oráculo puede dominar

⚡ APLICACIONES REALES:
  - Búsqueda en bases de datos cuánticas
  - Criptografía: rompe AES-128 en O(2^64) en vez de O(2^128)
  - Combinado con QAOA para optimización
  - Componente de amplificación en algoritmos más complejos
"""

# Tabla de ventaja cuadrática
print("Ventaja cuadrática de Grover:")
print(f"{'n':>4} {'N=2^n':>10} {'Clásico O(N)':>14} {'Grover O(√N)':>14} {'Speedup':>10}")
print("─" * 60)
for n in [10, 20, 30, 40, 50, 60, 128, 256]:
    N = 2**n
    clasico = N // 2   # promedio
    grover  = int(np.pi / 4 * np.sqrt(N))
    speedup = clasico / grover
    if n <= 30:
        print(f"  {n:>2}  {N:>10.2e}  {clasico:>14.2e}  {grover:>14.2e}  {speedup:>9.0f}×")
    else:
        print(f"  {n:>2}  {N:>10.2e}  {'~2^'+str(n-1):>14}  {'~2^'+str(n//2):>14}  {'~2^'+str(n//2-1):>9}×")
```

---

## checkpoint_m09.py

```python
"""
Checkpoint Módulo 9 — Algoritmo de Grover
Ejecutar con: python checkpoint_m09.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import os

os.makedirs("outputs", exist_ok=True)
sim_qasm = AerSimulator(method='qasm')
sim_sv   = AerSimulator(method='statevector')

def sv(qc):
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim_sv.run(transpile(qc2, sim_sv)).result().get_statevector())

def grover_oracle(n, objetivo):
    qc = QuantumCircuit(n, name=f"Oracle({objetivo})")
    bits = format(objetivo, f'0{n}b')[::-1]
    for i, b in enumerate(bits):
        if b == '0': qc.x(i)
    if n == 1: qc.z(0)
    elif n == 2: qc.cz(0, 1)
    else:
        qc.h(n-1); qc.mcx(list(range(n-1)), n-1); qc.h(n-1)
    for i, b in enumerate(bits):
        if b == '0': qc.x(i)
    return qc

def grover_difusor(n):
    qc = QuantumCircuit(n, name="Difusor")
    qc.h(range(n)); qc.x(range(n))
    qc.h(n-1); qc.mcx(list(range(n-1)), n-1); qc.h(n-1)
    qc.x(range(n)); qc.h(range(n))
    return qc

def grover(n, objetivo, k=None):
    N = 2**n
    if k is None: k = max(1, int(np.pi/4 * np.sqrt(N)))
    qc = QuantumCircuit(n, n)
    qc.h(range(n))
    for _ in range(k):
        qc.compose(grover_oracle(n, objetivo), inplace=True)
        qc.compose(grover_difusor(n), inplace=True)
    qc.measure(range(n), range(n))
    cnts = sim_qasm.run(transpile(qc, sim_qasm), shots=4096).result().get_counts()
    total = sum(cnts.values())
    return max(cnts, key=cnts.get), max(cnts.values()) / total

print("=" * 55)
print("Checkpoint M09 — Grover")
print("=" * 55)

# ── Test 1: Grover n=2 encuentra objetivo ────────────────
for obj in range(4):
    mejor, prob = grover(2, obj)
    encontrado = int(mejor[::-1], 2)
    assert encontrado == obj, f"FALLO: Grover n=2, obj={obj}, encontrado={encontrado}"
print("✓ Test 1: Grover n=2 — los 4 estados encontrados")

# ── Test 2: Grover n=3 ────────────────────────────────────
for obj in [0, 3, 5, 7]:
    mejor, prob = grover(3, obj)
    encontrado = int(mejor[::-1], 2)
    assert encontrado == obj, f"FALLO: Grover n=3, obj={obj}, encontrado={encontrado}"
    assert prob > 0.8, f"FALLO: P(obj={obj}) = {prob:.3f} < 0.8"
print("✓ Test 2: Grover n=3 con P > 0.8")

# ── Test 3: n=4, n=5 ─────────────────────────────────────
for n in [4, 5]:
    obj = 2**n - 3
    mejor, prob = grover(n, obj)
    encontrado = int(mejor[::-1], 2)
    assert encontrado == obj, f"FALLO: n={n}, obj={obj}"
    assert prob > 0.7, f"FALLO: n={n} P={prob:.3f} < 0.7"
print("✓ Test 3: Grover n=4, n=5 correctos")

# ── Test 4: Oráculo invierte signo del objetivo ───────────
n_t = 3; obj_t = 4
qc_ora = QuantumCircuit(n_t)
qc_ora.h(range(n_t))   # superposición uniforme
qc_ora.compose(grover_oracle(n_t, obj_t), inplace=True)
sv_tras_oracle = sv(qc_ora)
# El estado objetivo debe tener amplitud negativa
idx_obj = obj_t   # Qiskit indexa |q_{n-1}...q_0⟩, verificar
amp_obj = sv_tras_oracle[obj_t]
assert amp_obj.real < 0, f"FALLO: amplitud objetivo debe ser negativa, got {amp_obj}"
print(f"✓ Test 4: Oráculo invierte amplitud del estado |{obj_t}⟩")

# ── Test 5: Probabilidad analítica vs simulada ────────────
def prob_analitica(N, k):
    theta = np.arcsin(1/np.sqrt(N))
    return np.sin((2*k+1)*theta)**2

for n in [3, 4]:
    N = 2**n
    k_opt = max(1, int(np.pi/4 * np.sqrt(N)))
    prob_an = prob_analitica(N, k_opt)
    _, prob_sim = grover(n, N//3, k_opt)
    assert abs(prob_an - prob_sim) < 0.1, \
        f"FALLO: n={n}, P_analítica={prob_an:.3f}, P_simulada={prob_sim:.3f}"
    print(f"✓ Test 5a: n={n} P_analítica={prob_an:.3f} ≈ P_simulada={prob_sim:.3f}")

# ── Test 6: k_opt correcta ────────────────────────────────
for n in [2, 3, 4, 6]:
    N = 2**n
    k_esperado = max(1, int(np.pi/4 * np.sqrt(N)))
    prob_k_opt = prob_analitica(N, k_esperado)
    # En k_opt, la probabilidad debe ser cercana al máximo (>0.5)
    assert prob_k_opt > 0.5, \
        f"FALLO: k_opt={k_esperado} da P={prob_k_opt:.3f} < 0.5 para n={n}"
print("✓ Test 6: k_opt da probabilidad > 0.5 para n=2,3,4,6")

print("\n" + "=" * 55)
print("✅ Todos los tests del M09 pasaron correctamente")
print("=" * 55)
```
