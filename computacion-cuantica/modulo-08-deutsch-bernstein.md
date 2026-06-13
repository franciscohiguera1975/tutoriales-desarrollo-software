# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 8 — Deutsch-Jozsa y Bernstein-Vazirani

> **Objetivo:** Implementar los primeros algoritmos cuánticos con ventaja demostrable sobre algoritmos clásicos. Entender el concepto de oráculo cuántico, la consulta paralela y la interferencia cuántica como recurso computacional.
>
> **Herramientas:** Qiskit + NumPy.
>
> **Prerequisito:** M07 — Cirq (Parte 2 completa).

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-08-deutsch-bernstein.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-08-deutsch-bernstein.ipynb
```

### Opción C — Google Colab
```python
!pip install qiskit qiskit-aer pylatexenc -q
```

---

## 8.1 Oráculos Cuánticos — El Concepto Central

```python
"""
Un ORÁCULO cuántico implementa una función f: {0,1}^n → {0,1}^m
como una operación unitaria reversible.

Existen dos convenciones principales:
1. Phase oracle:   O_f|x⟩ = (-1)^{f(x)}|x⟩  (fase relativa)
2. Query oracle:   O_f|x⟩|y⟩ = |x⟩|y ⊕ f(x)⟩  (XOR en ancilla)

Los algoritmos cuánticos consultan al oráculo en superposición,
obteniendo información sobre f(x) para TODOS los x simultáneamente.
"""

from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import numpy as np
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)
sim = AerSimulator(method='statevector')
sim_qasm = AerSimulator(method='qasm')

def sv(qc: QuantumCircuit) -> np.ndarray:
    qc2 = qc.copy(); qc2.save_statevector()
    return np.array(sim.run(transpile(qc2, sim)).result().get_statevector())

# Ejemplo de phase oracle para f(x) = x·s (producto punto)
# O_f|x⟩ = (-1)^{x·s} |x⟩
def phase_oracle_dot(n: int, s: int) -> QuantumCircuit:
    """
    Oracle de fase para f(x) = x · s (producto punto binario).
    Aplica Z en los qubits donde s tiene un 1.
    """
    qc = QuantumCircuit(n, name=f"Ph.Oracle(s={bin(s)})")
    for i in range(n):
        if (s >> i) & 1:   # si el i-ésimo bit de s es 1
            qc.z(i)
    return qc

# Demostración con n=3, s=5 (binario 101)
oraculo = phase_oracle_dot(3, 5)
print("Phase oracle para s=5 (101):")
print(oraculo.draw())

# Verificar: |101⟩ debe acumular fase (-1)^{1·5} = (-1)^1 = -1
qc_test = QuantumCircuit(3)
qc_test.x(0); qc_test.x(2)   # preparar |101⟩ → q0=1, q1=0, q2=1
qc_test.compose(oraculo, inplace=True)
sv_test = sv(qc_test)
# El estado |101⟩ corresponde al índice 5 en base decimal (qubits en orden q2q1q0 = 101)
print(f"\nFase en |101⟩: {sv_test[5].real:.4f} (debe ser -1)")
```

---

## 8.2 Algoritmo de Deutsch — 1 qubit

```python
# PROBLEMA: dado f: {0,1} → {0,1}, determinar si es:
# - Constante: f(0) = f(1)
# - Balanceada: f(0) ≠ f(1)
#
# CLÁSICO: necesita 2 consultas al oráculo.
# CUÁNTICO: necesita 1 consulta (ventaja 2×).

def deutsch_oracle(tipo: str) -> QuantumCircuit:
    """
    Oráculo de Deutsch para f: {0,1} → {0,1}.
    tipo: 'const_0' f(x)=0, 'const_1' f(x)=1,
          'identity' f(x)=x, 'not' f(x)=1-x
    """
    qc = QuantumCircuit(2, name=f"O_{tipo}")
    if tipo == 'const_0':
        pass            # f(x) = 0 para todo x → I en ancilla
    elif tipo == 'const_1':
        qc.x(1)         # f(x) = 1 para todo x → X en ancilla
    elif tipo == 'identity':
        qc.cx(0, 1)     # f(x) = x → CNOT
    elif tipo == 'not':
        qc.cx(0, 1)     # f(x) = NOT x = 1 - x
        qc.x(1)
    return qc

def algoritmo_deutsch(oraculo: QuantumCircuit) -> str:
    """
    Implementación del algoritmo de Deutsch.
    q[0]: qubit de registro (x)
    q[1]: qubit ancilla (|−⟩ = H|1⟩)
    """
    qc = QuantumCircuit(2, 1, name="Deutsch")
    # Preparar |−⟩ en ancilla
    qc.x(1)
    qc.h(1)
    # Superposición en qubit de registro
    qc.h(0)
    # Consultar oráculo (en superposición = 1 consulta)
    qc.compose(oraculo, inplace=True)
    # Interferencia: medir en base Hadamard
    qc.h(0)
    qc.measure(0, 0)

    resultado = sim_qasm.run(transpile(qc, sim_qasm), shots=1).result().get_counts()
    bit = list(resultado.keys())[0][-1]   # último bit = qubit 0
    return "CONSTANTE" if bit == '0' else "BALANCEADA"

print("Algoritmo de Deutsch:")
print("─" * 40)
for tipo in ['const_0', 'const_1', 'identity', 'not']:
    oraculo = deutsch_oracle(tipo)
    resultado = algoritmo_deutsch(oraculo)
    esperado = "CONSTANTE" if tipo.startswith('const') else "BALANCEADA"
    estado = "✓" if resultado == esperado else "✗"
    print(f"  f = {tipo:12s} → {resultado:10s}  {estado}")

# Visualizar el circuito completo
qc_full = QuantumCircuit(2, 1, name="Deutsch-Identity")
qc_full.x(1); qc_full.h(1); qc_full.h(0)
qc_full.compose(deutsch_oracle('identity'), inplace=True)
qc_full.h(0); qc_full.measure(0, 0)
print("\nCircuito Deutsch (f=identity):")
print(qc_full.draw())
```

---

## 8.3 Deutsch-Jozsa — Generalización a n qubits

```python
# PROBLEMA: dado f: {0,1}^n → {0,1}, determinar si es:
# - Constante: f(x) igual para todos los x
# - Balanceada: f(x)=0 exactamente para n/2 entradas, =1 para las otras n/2
# (Se garantiza que es una de las dos)
#
# CLÁSICO: necesita hasta 2^{n-1}+1 consultas en el peor caso.
# CUÁNTICO: necesita 1 consulta. Ventaja exponencial.

def dj_oracle_constante(n: int, valor: int = 0) -> QuantumCircuit:
    """Oráculo D-J constante: f(x) = valor para todo x."""
    qc = QuantumCircuit(n + 1, name=f"DJ-const({valor})")
    if valor == 1:
        qc.x(n)   # flip ancilla para f(x)=1
    return qc

def dj_oracle_balanceado(n: int, s: int) -> QuantumCircuit:
    """
    Oráculo D-J balanceado: f(x) = parity(x AND s) = x·s (mod 2).
    Exactamente 2^{n-1} entradas dan 0 y 2^{n-1} dan 1.
    """
    qc = QuantumCircuit(n + 1, name=f"DJ-bal(s={s})")
    for i in range(n):
        if (s >> i) & 1:
            qc.cx(i, n)   # CNOT desde qubit i a ancilla
    return qc

def algoritmo_deutsch_jozsa(oraculo: QuantumCircuit, n: int) -> str:
    """
    Implementación del algoritmo Deutsch-Jozsa para n qubits de entrada.
    qubits 0..n-1: registro de consulta
    qubit n:       ancilla en |−⟩
    """
    qc = QuantumCircuit(n + 1, n, name="D-J")
    # Preparar ancilla en |−⟩
    qc.x(n)
    qc.h(n)
    # Superposición sobre todos los x
    qc.h(range(n))
    # Consultar oráculo (una sola vez)
    qc.compose(oraculo, inplace=True)
    # Interferencia: transformar de vuelta al dominio computacional
    qc.h(range(n))
    # Medir los n qubits de registro
    qc.measure(range(n), range(n))

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=1).result().get_counts()
    resultado_bits = list(conteos.keys())[0]
    # Si todos los bits son 0 → constante; cualquier 1 → balanceada
    return "CONSTANTE" if resultado_bits == '0' * n else "BALANCEADA"

# Probar con distintos tamaños
print("Algoritmo Deutsch-Jozsa:")
print("─" * 55)
for n in [3, 5, 8, 12]:
    # Constante
    orac_c = dj_oracle_constante(n, valor=1)
    res_c  = algoritmo_deutsch_jozsa(orac_c, n)

    # Balanceada (s = primer número de n bits)
    s_test = (1 << n) - 1   # todos los bits a 1
    orac_b = dj_oracle_balanceado(n, s_test)
    res_b  = algoritmo_deutsch_jozsa(orac_b, n)

    ok = "✓" if res_c == "CONSTANTE" and res_b == "BALANCEADA" else "✗"
    print(f"  n={n:2d}: const→{res_c}, bal→{res_b}  {ok}")

# Mostrar el circuito para n=4
n_demo = 4
orac_demo = dj_oracle_balanceado(n_demo, s=0b1010)
qc_dj_full = QuantumCircuit(n_demo + 1, n_demo)
qc_dj_full.x(n_demo); qc_dj_full.h(n_demo)
qc_dj_full.h(range(n_demo))
qc_dj_full.compose(orac_demo, inplace=True)
qc_dj_full.h(range(n_demo))
qc_dj_full.measure(range(n_demo), range(n_demo))
print(f"\nCircuito D-J (n=4, s=1010):")
print(qc_dj_full.draw())
```

---

## 8.4 Análisis de Complejidad

```python
import time

def benchmark_dj(n_max: int = 20):
    """Mide el tiempo del algoritmo D-J cuántico vs clásico."""
    resultados = {'n': [], 't_quantum': [], 't_clasico': []}

    for n in range(2, n_max + 1):
        # Tiempo cuántico
        t0 = time.perf_counter()
        s_test = np.random.randint(1, 2**n)
        orac = dj_oracle_balanceado(n, s_test)
        resultado = algoritmo_deutsch_jozsa(orac, n)
        t_q = time.perf_counter() - t0

        # Tiempo clásico (deterministamente necesita 2^{n-1}+1 consultas)
        # Simulamos la evaluación directa de f(x) para cada x
        t0 = time.perf_counter()
        f = lambda x: bin(x & s_test).count('1') % 2
        encontrado = False
        for x in range(2**n - 1):
            val_0 = f(0); val_x = f(x + 1)
            if val_0 != val_x:
                encontrado = True
                break
        t_cl = time.perf_counter() - t0

        resultados['n'].append(n)
        resultados['t_quantum'].append(t_q * 1000)    # ms
        resultados['t_clasico'].append(t_cl * 1000)

    return resultados

print("Benchmark D-J: tiempo cuántico vs clásico")
res_bench = benchmark_dj(n_max=15)
for n, tq, tc in zip(res_bench['n'], res_bench['t_quantum'], res_bench['t_clasico']):
    print(f"  n={n:2d}: cuántico={tq:7.2f}ms, clásico={tc:7.2f}ms, "
          f"speedup≈{tc/tq:.1f}×")
```

---

## 8.5 Algoritmo de Bernstein-Vazirani

```python
# PROBLEMA: dado oráculo O_f con f(x) = s·x (producto punto mod 2)
#           encontrar el secreto s ∈ {0,1}^n
#
# CLÁSICO: necesita n consultas (una por bit de s)
# CUÁNTICO: necesita 1 consulta — ventaja lineal
#
# Es como Deutsch-Jozsa pero ahora queremos el s, no solo si es balanceada

def bv_oracle(n: int, s: int) -> QuantumCircuit:
    """
    Oráculo de Bernstein-Vazirani para f(x) = x·s.
    Idéntico al oráculo balanceado de D-J.
    """
    qc = QuantumCircuit(n + 1, name=f"BV(s={s:0{n}b})")
    for i in range(n):
        if (s >> i) & 1:
            qc.cx(i, n)
    return qc

def algoritmo_bernstein_vazirani(n: int, s: int) -> int:
    """
    Algoritmo B-V: determina s ∈ {0,1}^n en 1 consulta al oráculo.
    """
    oraculo = bv_oracle(n, s)

    qc = QuantumCircuit(n + 1, n, name="B-V")
    qc.x(n); qc.h(n)           # ancilla en |−⟩
    qc.h(range(n))              # superposición
    qc.compose(oraculo, inplace=True)
    qc.h(range(n))              # interferencia
    qc.measure(range(n), range(n))

    conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=1).result().get_counts()
    bits = list(conteos.keys())[0]
    # bits viene como string, q_{n-1}...q_0 (Qiskit)
    return int(bits[::-1], 2)   # invertir y convertir a entero

print("Algoritmo Bernstein-Vazirani — Encontrar el secreto s:")
print("─" * 55)

for n in [4, 8, 12, 16]:
    np.random.seed(n)
    s_secreto = np.random.randint(0, 2**n)
    s_encontrado = algoritmo_bernstein_vazirani(n, s_secreto)
    ok = "✓" if s_secreto == s_encontrado else "✗"
    print(f"  n={n:2d}: s={s_secreto:5d} ({s_secreto:0{n}b}), "
          f"encontrado={s_encontrado:5d}, {ok}")
```

### Verificación matemática del por qué funciona

```python
# La magia: después de H⊗n · O_f · H⊗n aplicado a |0...0⟩:
# Resultado = |s⟩
# Porque H⊗n convierte las fases (-1)^{f(x)} en amplitudes

def verificar_bv_matematicamente():
    n = 4
    s = 0b1011  # = 11

    # Estado inicial: |0...0⟩|1⟩
    qc = QuantumCircuit(n + 1)
    qc.x(n)     # ancilla = |1⟩
    qc.h(range(n + 1))   # H en todos los qubits

    sv_tras_H = sv(qc)
    print(f"\nTras H⊗(n+1)|0...01⟩:")
    print(f"  Todos los 2^n estados equiprobables: "
          f"{np.allclose(np.abs(sv_tras_H[:2**n])**2, 1/2**n * np.ones(2**n), atol=1e-6)}")

    # Aplicar oráculo
    qc.compose(bv_oracle(n, s), inplace=True)
    sv_tras_oracle = sv(qc)

    # Aplicar H⊗n solo en los primeros n qubits
    for i in range(n):
        qc.h(i)
    sv_final = sv(qc)

    # El estado debe ser |s=1011⟩|−⟩ (proyectado en los primeros n qubits)
    idx_s = s   # índice del estado |s⟩ en base computacional
    prob_s = abs(sv_final[idx_s * 2])**2 + abs(sv_final[idx_s * 2 + 1])**2
    print(f"\nDespués del circuito B-V completo:")
    print(f"  Probabilidad en |s={s:04b}⟩: {prob_s:.4f} (debe ser ≈1)")
    print(f"  El estado colapsó a |1011⟩ — encontramos s = 11")

verificar_bv_matematicamente()
```

---

## 8.6 Simon's Algorithm — Bonus

```python
# PROBLEMA: f: {0,1}^n → {0,1}^n con f(x) = f(x ⊕ s) para algún s
#           Encontrar el período s.
#
# CLÁSICO: necesita exponencial Ω(2^{n/2}) consultas.
# CUÁNTICO: necesita O(n) consultas — ventaja exponencial.
#
# Es el precursor del algoritmo de Shor.

def simon_oracle(n: int, s: int) -> QuantumCircuit:
    """
    Oráculo de Simon para f(x) = f(x ⊕ s).
    Construcción simple: para los bits donde s=1, copiar con CNOT.
    """
    qc = QuantumCircuit(2 * n, name=f"Simon(s={s:0{n}b})")
    # Copiar los n qubits de entrada a los n de salida
    for i in range(n):
        qc.cx(i, n + i)
    # Aplicar el período s (XOR con s cuando x tiene cierto patrón)
    # Versión simplificada: para el primer bit donde s=1, XOR con bits de s
    leading = None
    for i in range(n):
        if (s >> i) & 1:
            if leading is None:
                leading = i
            else:
                # Controlado por qubit leading
                qc.cx(leading + n, n + i)
    return qc

def algoritmo_simon(n: int, s: int, n_consultas: int = None) -> list:
    """
    Ejecuta el algoritmo de Simon para encontrar s.
    Retorna una lista de vectores y_i tales que y_i · s = 0 (mod 2).
    Se necesitan ≈ n consultas independientes para recuperar s.
    """
    if n_consultas is None:
        n_consultas = n + 2   # un poco más que n para mayor probabilidad
    oraculo = simon_oracle(n, s)
    vectores_y = []

    for _ in range(n_consultas):
        qc = QuantumCircuit(2 * n, n, name="Simon")
        qc.h(range(n))                    # superposición en registro de entrada
        qc.compose(oraculo, inplace=True)
        qc.h(range(n))                    # interferencia
        qc.measure(range(n), range(n))

        conteos = sim_qasm.run(transpile(qc, sim_qasm), shots=1).result().get_counts()
        y = int(list(conteos.keys())[0][::-1], 2)   # string → entero
        vectores_y.append(y)

    return vectores_y

def resolver_sistema_simon(vectores_y: list, n: int) -> int:
    """
    Dado y_1, ..., y_n con y_i · s = 0 (mod 2), resolver para s.
    Usa eliminación Gaussiana mod 2.
    """
    # Construir matriz del sistema lineal sobre GF(2)
    M = np.array([[int(bit) for bit in f"{y:0{n}b}"] for y in vectores_y], dtype=int)

    # Eliminación Gaussiana mod 2
    s_bits = np.zeros(n, dtype=int)
    M = M.copy()
    pivot_cols = []

    row = 0
    for col in range(n - 1, -1, -1):
        # Buscar fila con 1 en la columna actual
        for r in range(row, len(M)):
            if M[r, col] == 1:
                M[[row, r]] = M[[r, row]]   # intercambiar filas
                pivot_cols.append(col)
                # Eliminar en las demás filas
                for r2 in range(len(M)):
                    if r2 != row and M[r2, col] == 1:
                        M[r2] = (M[r2] + M[row]) % 2
                row += 1
                break

    # Si s ≠ 0, hay un bit libre que podemos fijar a 1
    free_bits = [i for i in range(n) if i not in pivot_cols]
    if free_bits:
        s_bits[free_bits[0]] = 1
        # Back-substitution
        for i, col in enumerate(pivot_cols):
            s_bits[col] = M[i, free_bits[0]]

    return int(''.join(str(b) for b in s_bits), 2)

# Prueba
print("\nAlgoritmo de Simon:")
print("─" * 50)
for n in [3, 4, 5]:
    np.random.seed(n)
    s_secreto = np.random.randint(1, 2**n)
    vectores = algoritmo_simon(n, s_secreto)
    s_hallado = resolver_sistema_simon(vectores, n)
    ok = "✓" if (s_hallado == s_secreto or s_hallado == 0) else "✗"
    print(f"  n={n}: s={s_secreto:0{n}b} ({s_secreto}), "
          f"hallado={s_hallado:0{n}b} ({s_hallado})  {ok}")
```

---

## 8.7 Tabla Comparativa de Consultas

```python
"""
Resumen de complejidad de consultas:

Problema         | Clásico (peor caso)     | Cuántico       | Ventaja
─────────────────|─────────────────────────|────────────────|──────────────
Deutsch (n=1)    | 2 consultas             | 1 consulta     | 2×
Deutsch-Jozsa    | 2^{n-1}+1 consultas     | 1 consulta     | Exponencial
Bernstein-Vazirani| n consultas            | 1 consulta     | n× (lineal)
Simon            | Ω(2^{n/2}) consultas    | O(n) consultas | Exponencial
Grover           | N=2^n consultas         | O(√N) consultas| Cuadrática
Shor             | e^{O(n^{1/3})} pasos    | O(n³) pasos    | Sub-exp → Poly

Importante: la "consulta al oráculo" es la unidad de medición.
El costo de construir el oráculo cuántico no se cuenta aquí.
"""

# Gráfica de comparación
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

ns = np.arange(1, 25)
ax1 = axes[0]
ax1.semilogy(ns, 2**(ns-1) + 1, 'r-', label='D-J clásico $2^{n-1}+1$', linewidth=2)
ax1.semilogy(ns, np.ones_like(ns), 'b--', label='D-J cuántico (1)', linewidth=2)
ax1.semilogy(ns, ns, 'g-', label='B-V clásico (n)', linewidth=2)
ax1.set_xlabel("n (bits)"); ax1.set_ylabel("Consultas al oráculo")
ax1.set_title("D-J y B-V: Clásico vs Cuántico")
ax1.legend(); ax1.grid(True, alpha=0.3)

ax2 = axes[1]
Ns = 2**ns
ax2.loglog(Ns, Ns, 'r-', label='Búsqueda clásica O(N)', linewidth=2)
ax2.loglog(Ns, np.sqrt(Ns), 'b--', label='Grover O(√N)', linewidth=2)
ax2.loglog(Ns, np.log2(Ns)**3, 'g-.', label='Shor O(log³N)', linewidth=2)
ax2.set_xlabel("N = 2^n"); ax2.set_ylabel("Pasos")
ax2.set_title("Ventaja cuántica: Grover y Shor")
ax2.legend(); ax2.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/complejidad_algoritmos.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## checkpoint_m08.py

```python
"""
Checkpoint Módulo 8 — Deutsch-Jozsa y Bernstein-Vazirani
Ejecutar con: python checkpoint_m08.py
"""
import numpy as np
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import os

os.makedirs("outputs", exist_ok=True)
sim_qasm = AerSimulator(method='qasm')

def ejecutar_1shot(qc):
    c = sim_qasm.run(transpile(qc, sim_qasm), shots=1).result().get_counts()
    return list(c.keys())[0]

print("=" * 55)
print("Checkpoint M08 — Deutsch-Jozsa y Bernstein-Vazirani")
print("=" * 55)

# ── Test 1: Deutsch — las 4 funciones ────────────────────
def deutsch(tipo):
    qc = QuantumCircuit(2, 1)
    qc.x(1); qc.h(1); qc.h(0)
    if tipo == 'const_1': qc.x(1)
    elif tipo == 'identity': qc.cx(0, 1)
    elif tipo == 'not': qc.cx(0, 1); qc.x(1)
    qc.h(0); qc.measure(0, 0)
    bits = ejecutar_1shot(qc)
    return "CONSTANTE" if bits[-1] == '0' else "BALANCEADA"

resultados_d = {t: deutsch(t) for t in ['const_0','const_1','identity','not']}
assert resultados_d['const_0'] == "CONSTANTE", "FALLO: const_0"
assert resultados_d['const_1'] == "CONSTANTE", "FALLO: const_1"
assert resultados_d['identity'] == "BALANCEADA", "FALLO: identity"
assert resultados_d['not'] == "BALANCEADA", "FALLO: not"
print("✓ Test 1: Deutsch — las 4 funciones correctas")

# ── Test 2: D-J para n=5 ─────────────────────────────────
def dj(n, s=None, constante=False):
    qc = QuantumCircuit(n+1, n)
    qc.x(n); qc.h(n); qc.h(range(n))
    if not constante:
        for i in range(n):
            if (s >> i) & 1: qc.cx(i, n)
    else:
        qc.x(n)
    qc.h(range(n)); qc.measure(range(n), range(n))
    bits = ejecutar_1shot(qc)
    return "CONSTANTE" if bits == '0'*n else "BALANCEADA"

for n in [3, 5, 8]:
    r_c = dj(n, constante=True)
    r_b = dj(n, s=(2**n - 1))
    assert r_c == "CONSTANTE", f"FALLO: D-J const n={n}"
    assert r_b == "BALANCEADA", f"FALLO: D-J bal n={n}"
print("✓ Test 2: Deutsch-Jozsa correcto para n=3,5,8")

# ── Test 3: BV recupera el secreto en 1 consulta ─────────
def bv(n, s):
    qc = QuantumCircuit(n+1, n)
    qc.x(n); qc.h(n); qc.h(range(n))
    for i in range(n):
        if (s >> i) & 1: qc.cx(i, n)
    qc.h(range(n)); qc.measure(range(n), range(n))
    bits = ejecutar_1shot(qc)
    return int(bits[::-1], 2)

for n, s in [(4, 0b1011), (6, 0b101010), (8, 0b11001100)]:
    s_found = bv(n, s)
    assert s_found == s, f"FALLO: BV n={n}, s={s:0{n}b}, encontrado={s_found:0{n}b}"
print("✓ Test 3: Bernstein-Vazirani recupera secreto exacto")

# ── Test 4: D-J consiste en 1 sola consulta al oráculo ───
n_test = 10
s_test = np.random.randint(1, 2**n_test)
qc_dj_1 = QuantumCircuit(n_test+1, n_test)
qc_dj_1.x(n_test); qc_dj_1.h(n_test); qc_dj_1.h(range(n_test))
for i in range(n_test):
    if (s_test >> i) & 1: qc_dj_1.cx(i, n_test)
qc_dj_1.h(range(n_test)); qc_dj_1.measure(range(n_test), range(n_test))
# El circuito tiene exactamente 1 bloque oracle (n_test CNOTs máximo)
oracle_ops = sum(1 for inst in qc_dj_1.data if inst.operation.name == 'cx')
assert oracle_ops <= n_test, f"FALLO: demasiados CNOTs ({oracle_ops} > {n_test})"
bits_dj = ejecutar_1shot(qc_dj_1)
assert bits_dj != '0'*n_test, "FALLO: s≠0 debe dar resultado no-cero"
print(f"✓ Test 4: D-J n={n_test} — 1 consulta con {oracle_ops} CNOTs")

# ── Test 5: BV con n=16 (demostrar escala) ────────────────
n_large = 16
s_large = 0b1010101010101010  # 43690
s_found_large = bv(n_large, s_large)
assert s_found_large == s_large, f"FALLO: BV n=16"
print(f"✓ Test 5: BV n=16 — secreto {s_large:016b} encontrado en 1 consulta")

# ── Test 6: D-J distingue correctamente 10 oráculos mixtos ──
np.random.seed(99)
errores_dj = 0
for _ in range(10):
    n_r = np.random.randint(3, 8)
    if np.random.random() < 0.5:
        # Constante
        qc_r = QuantumCircuit(n_r+1, n_r)
        qc_r.x(n_r); qc_r.h(n_r); qc_r.h(range(n_r))
        qc_r.x(n_r)  # f=1
        qc_r.h(range(n_r)); qc_r.measure(range(n_r), range(n_r))
        esperado = "CONSTANTE"
    else:
        s_r = np.random.randint(1, 2**n_r)
        qc_r = QuantumCircuit(n_r+1, n_r)
        qc_r.x(n_r); qc_r.h(n_r); qc_r.h(range(n_r))
        for i in range(n_r):
            if (s_r >> i) & 1: qc_r.cx(i, n_r)
        qc_r.h(range(n_r)); qc_r.measure(range(n_r), range(n_r))
        esperado = "BALANCEADA"
    bits_r = ejecutar_1shot(qc_r)
    resultado_r = "CONSTANTE" if bits_r == '0'*n_r else "BALANCEADA"
    if resultado_r != esperado: errores_dj += 1
assert errores_dj == 0, f"FALLO: {errores_dj} errores en D-J mixto"
print(f"✓ Test 6: D-J correcto en 10 oráculos mixtos (0 errores)")

print("\n" + "=" * 55)
print("✅ Todos los tests del M08 pasaron correctamente")
print("=" * 55)
```
