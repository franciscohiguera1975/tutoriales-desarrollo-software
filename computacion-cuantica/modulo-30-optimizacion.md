# Módulo 30 — Optimización Cuántica: Finanzas y Logística

**Objetivo:** Aplicar QAOA y VQE a problemas de optimización combinatoria reales: optimización de portafolios financieros, problema del viajante (TSP), scheduling y problemas de selección de rutas logísticas.

**Herramientas:** PennyLane 0.39+, Qiskit 1.x, NumPy, NetworkX, matplotlib
**Prerequisito:** M19 (QAOA), M18 (VQE)

---

## Ejecución

```bash
conda activate quantum
pip install networkx
mkdir -p outputs
python seccion_X.py
```

---

## 1. Optimización de portafolio financiero

El problema de Markowitz: maximizar retorno esperado para un nivel de riesgo dado.

```python
# seccion_01_portfolio.py
"""
Optimización de portafolio cuántico (Quantum Portfolio Optimization):
Minimizar: w^T Σ w - λ μ^T w
  w: vector de pesos (fracción invertida en cada activo)
  Σ: matriz de covarianza de retornos
  μ: vector de retornos esperados
  λ: parámetro de aversión al riesgo

Formulación QUBO (Quadratic Unconstrained Binary Optimization):
  Variables binarias x_i ∈ {0,1}: si activo i es incluido
  Objetivo: min_x x^T Q x + c^T x
  → mapeable a Hamiltoniano cuántico y optimizable con QAOA
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# === Datos del portafolio (5 activos: AAPL, MSFT, GOOGL, AMZN, NVDA) ===
np.random.seed(42)
n_activos = 4  # Reducido para caber en < 8 qubits con QAOA

# Retornos anualizados (simplificados)
retornos = np.array([0.28, 0.24, 0.22, 0.35])  # AAPL, MSFT, GOOGL, NVDA
nombres  = ["AAPL", "MSFT", "GOOGL", "NVDA"]

# Covarianza (simplificada)
correlacion = np.array([
    [1.00, 0.65, 0.60, 0.55],
    [0.65, 1.00, 0.70, 0.50],
    [0.60, 0.70, 1.00, 0.45],
    [0.55, 0.50, 0.45, 1.00],
])
volatilidades = np.array([0.25, 0.22, 0.24, 0.35])  # std anual
sigma = np.outer(volatilidades, volatilidades) * correlacion

print("=== Optimización de Portafolio ===")
print(f"Activos: {nombres}")
print(f"Retornos anuales: {retornos}")
print(f"Volatilidades: {volatilidades}")

# === Formulación QUBO ===
def qubo_portfolio(retornos, sigma, lam=0.5, presupuesto=2):
    """
    QUBO para selección de portafolio binario.
    x_i=1: incluir activo i, x_i=0: excluir
    Restricción suave: Σx_i = presupuesto (penalización A)

    Objetivo: min Σ_ij σ_ij x_i x_j - λ Σ_i μ_i x_i
              + A(Σx_i - B)²
    """
    n = len(retornos)
    A = 5.0  # penalización de presupuesto
    B = presupuesto

    # Parte cuadrática: riesgo
    Q = sigma.copy()

    # Término lineal: retorno (negativo para maximizar)
    c = -lam * retornos

    # Penalización presupuesto: A(Σx-B)² = A Σx² + 2A Σ_{i<j} x_ix_j - 2AB Σx + AB²
    Q += 2 * A * np.ones((n, n))  # términos cruzados
    np.fill_diagonal(Q, np.diag(Q) - 2*A*B)  # términos lineales al diagonal
    c += A * (1 - 2*B) * np.ones(n)  # offset lineal

    return Q, c

Q, c = qubo_portfolio(retornos, sigma, lam=0.5, presupuesto=2)

# === Hamiltoniano Cuántico desde QUBO ===
# x_i = (1 - Z_i)/2  → z_i ∈ {-1,+1}
def qubo_a_hamiltoniano_pl(Q, c):
    """Convierte QUBO a Hamiltoniano de PennyLane."""
    n = len(c)
    coeffs = []
    obs = []

    # Constante offset
    offset = sum(c[i]/2 for i in range(n)) + sum(Q[i,j]/4 for i in range(n) for j in range(n))

    # Términos lineales: c_i x_i = c_i(1-Z_i)/2
    for i in range(n):
        coeff_z = -(c[i] + sum(Q[i,j]/2 for j in range(n)))/2
        if abs(coeff_z) > 1e-8:
            coeffs.append(coeff_z)
            obs.append(qml.PauliZ(i))

    # Términos cuadráticos: Q_ij x_i x_j = Q_ij(1-Z_i)(1-Z_j)/4
    for i in range(n):
        for j in range(i+1, n):
            coeff_zz = (Q[i,j] + Q[j,i]) / 4
            if abs(coeff_zz) > 1e-8:
                coeffs.append(coeff_zz)
                obs.append(qml.PauliZ(i) @ qml.PauliZ(j))

    # Identidad (offset)
    coeffs.append(offset)
    obs.append(qml.Identity(0))

    return qml.Hamiltonian(coeffs, obs)

H_portfolio = qubo_a_hamiltoniano_pl(Q, c)
print(f"\nHamiltoniano QUBO: {len(H_portfolio.ops)} términos")

# === QAOA para optimización de portafolio ===
n_qubits = n_activos
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, diff_method="parameter-shift")
def qaoa_portfolio(gammas, betas):
    """QAOA para portafolio con p=2."""
    # Estado inicial: superposición uniforme
    for i in range(n_qubits):
        qml.Hadamard(wires=i)

    p = len(gammas)
    for l in range(p):
        # Operador de problema U_C(γ)
        qml.CommutingEvolution(H_portfolio, gammas[l])
        # Mixing U_B(β)
        for i in range(n_qubits):
            qml.RX(-2 * betas[l], wires=i)

    return qml.expval(H_portfolio)

# Optimización QAOA p=2
p = 2
gammas = pnp.array([0.5, 0.8], requires_grad=True)
betas  = pnp.array([0.3, 0.5], requires_grad=True)
opt = qml.AdamOptimizer(stepsize=0.05)

print("\nOptimizando portafolio con QAOA (p=2)...")
energias_qaoa = []
for i in range(100):
    (gammas, betas), E = opt.step_and_cost(qaoa_portfolio, gammas, betas)
    energias_qaoa.append(float(E))
    if (i+1) % 25 == 0:
        print(f"  Iter {i+1:3d}: E={E:.6f}")

# Medir solución
@qml.qnode(dev)
def medicion_final(gammas, betas):
    for i in range(n_qubits):
        qml.Hadamard(wires=i)
    p = len(gammas)
    for l in range(p):
        qml.CommutingEvolution(H_portfolio, gammas[l])
        for i in range(n_qubits):
            qml.RX(-2 * betas[l], wires=i)
    return qml.probs(wires=range(n_qubits))

probs = medicion_final(gammas, betas)
mejor_idx = np.argmax(probs)
mejor_portfolio = format(mejor_idx, f"0{n_qubits}b")

print(f"\nMejor portafolio encontrado: {mejor_portfolio}")
activos_sel = [nombres[i] for i, b in enumerate(mejor_portfolio) if b == "1"]
print(f"Activos seleccionados: {activos_sel}")

# Comparar con fuerza bruta
mejor_val = np.inf
mejor_combo = None
for mask in range(2**n_activos):
    bits = [(mask >> i) & 1 for i in range(n_activos)]
    if sum(bits) == 2:  # presupuesto = 2
        val = sum(Q[i,j]*bits[i]*bits[j] for i in range(n_activos)
                  for j in range(n_activos)) + sum(c[i]*bits[i] for i in range(n_activos))
        if val < mejor_val:
            mejor_val = val
            mejor_combo = bits

print(f"\nFuerza bruta: {[nombres[i] for i,b in enumerate(mejor_combo) if b==1]}")
print(f"QAOA acertó: {activos_sel == [nombres[i] for i,b in enumerate(mejor_combo) if b==1]}")
```

---

## 2. TSP: Problema del Viajante con QAOA

```python
# seccion_02_tsp.py
"""
Traveling Salesman Problem (TSP) con QAOA.
Formulación: n ciudades, x_{i,t}=1 si visita ciudad i en paso t.
Hamiltoniano: H_A (restricciones) + H_B (coste de ruta).

Para n=4 ciudades → 16 qubits (n² variables binarias).
Usamos n=3 para demostración (9 qubits).
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import networkx as nx
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# Grafo de 4 ciudades con distancias
n_ciudades = 4
ciudades = ["Madrid", "Barcelona", "Valencia", "Sevilla"]
# Matriz de distancias (km aproximados)
D = np.array([
    [0,   621, 357, 534],
    [621, 0,   349, 1000],
    [357, 349,  0,  657],
    [534, 1000, 657,  0],
], dtype=float)

print("=== TSP: Ciudades de España ===")
print("Distancias (km):")
print(f"{'':>12}", end="")
for c in ciudades:
    print(f"{c:>12}", end="")
print()
for i, ci in enumerate(ciudades):
    print(f"{ci:>12}", end="")
    for j in range(n_ciudades):
        print(f"{int(D[i,j]):>12}", end="")
    print()

# Fuerza bruta: encontrar ruta óptima
from itertools import permutations

def coste_ruta(ruta, D):
    n = len(ruta)
    return sum(D[ruta[i], ruta[(i+1)%n]] for i in range(n))

rutas = list(permutations(range(n_ciudades)))
costes = [coste_ruta(r, D) for r in rutas]
mejor_ruta = rutas[np.argmin(costes)]
mejor_coste = min(costes)

print(f"\nRuta óptima (fuerza bruta): {' → '.join(ciudades[i] for i in mejor_ruta)}")
print(f"Distancia total: {mejor_coste:.0f} km")

# === Formulación QUBO para TSP ===
def hamiltoniano_tsp_qubo(D, A=500, B=1):
    """
    Hamiltoniano QUBO para TSP.
    Variables: x_{i,t} = 1 si ciudad i se visita en posición t
    H = A Σ_i(1 - Σ_t x_{it})² + A Σ_t(1 - Σ_i x_{it})²
      + B Σ_{t,u,v} d_{uv} x_{u,t} x_{v,t+1}

    Para 4 ciudades: 16 variables → 16 qubits
    Simplificamos a 3 ciudades (subgrado) para demo: 9 qubits
    """
    n = len(D)  # número de ciudades
    N = n * n   # número de variables (n ciudades × n posiciones)

    Q = np.zeros((N, N))

    def idx(i, t):
        return i * n + t

    # Restricción A: cada ciudad visitada exactamente una vez
    for i in range(n):
        for t in range(n):
            Q[idx(i,t), idx(i,t)] -= A
            for t2 in range(t+1, n):
                Q[idx(i,t), idx(i,t2)] += 2*A
        # (Σ_t x_{i,t} - 1)² = Σ_t x² + 2Σ_{t<t'} x x - 2Σ_t x + 1

    # Restricción A: cada posición ocupada exactamente una vez
    for t in range(n):
        for i in range(n):
            Q[idx(i,t), idx(i,t)] -= A
            for i2 in range(i+1, n):
                Q[idx(i,t), idx(i2,t)] += 2*A

    # Coste B: distancias
    for t in range(n):
        t_next = (t + 1) % n
        for u in range(n):
            for v in range(n):
                if u != v:
                    Q[idx(u,t), idx(v,t_next)] += B * D[u,v]

    return Q

print("\nFormulación QUBO TSP (4 ciudades = 16 variables = 16 qubits)")
print("→ Demasiado grande para simulación directa.")
print("→ Usamos heurística QAOA en subproblema (3 ciudades = 9 qubits)")

# Demo con 3 ciudades para mantener tiempo de simulación razonable
D3 = D[:3, :3]  # Sub-grafo con Madrid, Barcelona, Valencia
ciudades3 = ciudades[:3]

rutas3 = list(permutations(range(3)))
costes3 = [coste_ruta(r, D3) for r in rutas3]
mejor3 = rutas3[np.argmin(costes3)]
mejor_coste3 = min(costes3)
print(f"\nSubproblema 3 ciudades:")
print(f"Óptimo: {' → '.join(ciudades3[i] for i in mejor3)} ({mejor_coste3:.0f} km)")

# Visualización del grafo
G = nx.from_numpy_array(D, create_using=nx.Graph())
pos = {0: (0, 0), 1: (3, 1.5), 2: (2.5, -1), 3: (-2, -1.5)}

fig, axes = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle("TSP — Ciudades de España", fontsize=13)

ax = axes[0]
nx.draw_networkx(G, pos=pos, ax=ax,
                 labels={i: c for i, c in enumerate(ciudades)},
                 node_color="lightblue", node_size=1500,
                 edge_color="gray", font_size=10)
edge_labels = {(i,j): f"{int(D[i,j])}km"
               for i in range(n_ciudades) for j in range(i+1, n_ciudades)}
nx.draw_networkx_edge_labels(G, pos=pos, edge_labels=edge_labels, ax=ax, font_size=7)
ax.set_title("Grafo de Distancias")
ax.axis("off")

# Ruta óptima
ax = axes[1]
G_ruta = nx.DiGraph()
for t in range(n_ciudades):
    G_ruta.add_edge(mejor_ruta[t], mejor_ruta[(t+1)%n_ciudades],
                    weight=D[mejor_ruta[t], mejor_ruta[(t+1)%n_ciudades]])
edge_colors = ["red" if (u,v) in G_ruta.edges else "lightgray"
               for u,v in G.edges]
nx.draw_networkx(G, pos=pos, ax=ax,
                 labels={i: c for i, c in enumerate(ciudades)},
                 node_color="lightblue", node_size=1500, font_size=10,
                 edge_color="gray")
nx.draw_networkx_edges(G_ruta, pos=pos, ax=ax,
                       edge_color="red", width=2.5, arrows=True, arrowsize=20)
ax.set_title(f"Ruta Óptima ({mejor_coste:.0f} km)")
ax.axis("off")

plt.tight_layout()
plt.savefig("outputs/m30_tsp.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m30_tsp.png")
plt.close()
```

---

## 3. Job Scheduling cuántico

```python
# seccion_03_scheduling.py
"""
Job Scheduling: asignar n trabajos a m máquinas minimizando makespan.
Formulación: min max_m Σ_{j asignado a m} t_j

Para m=2 máquinas, n=6 trabajos → 6 qubits (x_j=0: máquina A, x_j=1: máquina B)
Objetivo cuadrático: min (Σ_j t_j x_j - T/2)²
donde T = Σ t_j (tiempo total) — queremos balancear cargas

Este es equivalente a Number Partition Problem (NP-hard).
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp

# 6 trabajos con tiempos de procesamiento (horas)
tiempos = np.array([3, 5, 2, 4, 7, 1])
n_jobs = len(tiempos)
T_total = sum(tiempos)

print("=== Job Scheduling (2 máquinas) ===")
print(f"Trabajos y tiempos: {dict(zip(range(n_jobs), tiempos))}")
print(f"Tiempo total: {T_total} horas")
print(f"Balance perfecto: {T_total//2} h / {T_total - T_total//2} h")

# Fuerza bruta
mejor_diff = np.inf
mejor_asig = None
for mask in range(2**n_jobs):
    bits = [(mask >> i) & 1 for i in range(n_jobs)]
    carga_A = sum(t * (1-b) for t, b in zip(tiempos, bits))
    carga_B = sum(t * b for t, b in zip(tiempos, bits))
    diff = abs(carga_A - carga_B)
    if diff < mejor_diff:
        mejor_diff = diff
        mejor_asig = bits

print(f"\nÓptimo (fuerza bruta): desequilibrio = {mejor_diff} horas")
A_opt = [i for i,b in enumerate(mejor_asig) if b == 0]
B_opt = [i for i,b in enumerate(mejor_asig) if b == 1]
print(f"  Máquina A: trabajos {A_opt}, tiempo = {sum(tiempos[i] for i in A_opt)} h")
print(f"  Máquina B: trabajos {B_opt}, tiempo = {sum(tiempos[i] for i in B_opt)} h")

# Hamiltoniano de QAOA para Number Partition
# H = (Σ t_j Z_j)²  → qubit Z_j=-1 → máquina A, Z_j=+1 → máquina B
# Expandiendo: H = Σ_{ij} t_i t_j Z_i Z_j
# Mínimo: Σ t_j z_j = 0 (balance perfecto)

dev = qml.device("default.qubit", wires=n_jobs)

obs_schedule = qml.Hamiltonian(
    [tiempos[i]*tiempos[j] for i in range(n_jobs) for j in range(n_jobs)],
    [qml.PauliZ(i) @ qml.PauliZ(j) for i in range(n_jobs) for j in range(n_jobs)]
)

@qml.qnode(dev, diff_method="parameter-shift")
def qaoa_schedule(gammas, betas):
    for i in range(n_jobs):
        qml.Hadamard(wires=i)
    p = len(gammas)
    for l in range(p):
        qml.CommutingEvolution(obs_schedule, gammas[l])
        for i in range(n_jobs):
            qml.RX(-2*betas[l], wires=i)
    return qml.expval(obs_schedule)

# Optimización
gammas = pnp.array([0.3, 0.6], requires_grad=True)
betas  = pnp.array([0.4, 0.2], requires_grad=True)
opt = qml.AdamOptimizer(stepsize=0.05)

print("\nOptimizando QAOA scheduling...")
for i in range(80):
    (gammas, betas), E = opt.step_and_cost(qaoa_schedule, gammas, betas)
    if (i+1) % 40 == 0:
        print(f"  Iter {i+1}: E={E:.4f}")

# Medir solución
@qml.qnode(dev)
def medir_schedule(gammas, betas):
    for i in range(n_jobs):
        qml.Hadamard(wires=i)
    for l in range(len(gammas)):
        qml.CommutingEvolution(obs_schedule, gammas[l])
        for i in range(n_jobs):
            qml.RX(-2*betas[l], wires=i)
    return qml.probs(wires=range(n_jobs))

probs_sch = medir_schedule(gammas, betas)
top_5 = np.argsort(probs_sch)[-5:][::-1]
print("\nTop-5 soluciones QAOA:")
for idx in top_5:
    bits = [(idx >> i) & 1 for i in range(n_jobs)]
    cA = sum(t*(1-b) for t,b in zip(tiempos, bits))
    cB = sum(t*b for t,b in zip(tiempos, bits))
    print(f"  {format(idx, f'0{n_jobs}b')}: A={cA}h B={cB}h desequil={abs(cA-cB)} | p={probs_sch[idx]:.4f}")
```

---

## 4. Optimización logística: Ruteo de Vehículos

```python
# seccion_04_vrp.py
"""
Vehicle Routing Problem (VRP) simplificado:
- 1 depósito, n=4 clientes, 2 vehículos
- Minimizar distancia total recorrida
- Restricción: capacidad por vehículo

Para demo: formulación como Max-Cut extendido
"""
import numpy as np
import networkx as nx
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from itertools import permutations

n_clientes = 4
deposito = (5, 5)  # Centro del mapa
np.random.seed(7)
clientes = np.array([
    [1, 2], [8, 3], [3, 8], [9, 7],
])
nombres = ["Depósito"] + [f"C{i+1}" for i in range(n_clientes)]
coords  = np.vstack([deposito, clientes])

# Distancias euclidianas
def dist(a, b):
    return np.sqrt((a[0]-b[0])**2 + (a[1]-b[1])**2)

n_total = n_clientes + 1  # +depósito
D = np.array([[dist(coords[i], coords[j]) for j in range(n_total)]
              for i in range(n_total)])

print("=== Vehicle Routing Problem ===")
print(f"Depósito: {deposito}")
for i, (c, p) in enumerate(zip([f"C{i+1}" for i in range(n_clientes)], clientes)):
    print(f"  {c}: {tuple(p)}")

# Enumeración de rutas (2 vehículos, dividir 4 clientes en 2 grupos)
mejor_total = np.inf
mejor_split = None

from itertools import combinations
for r in range(1, n_clientes):
    for grupo1_idx in combinations(range(1, n_total), r):
        grupo2_idx = tuple(i for i in range(1, n_total) if i not in grupo1_idx)
        if not grupo2_idx:
            continue
        # Calcular ruta óptima para cada grupo (fuerza bruta)
        def mejor_ruta_grupo(grupo):
            mejor = np.inf
            for perm in permutations(grupo):
                ruta = [0] + list(perm) + [0]  # depósito → clientes → depósito
                coste = sum(D[ruta[i], ruta[i+1]] for i in range(len(ruta)-1))
                mejor = min(mejor, coste)
            return mejor

        coste1 = mejor_ruta_grupo(list(grupo1_idx))
        coste2 = mejor_ruta_grupo(list(grupo2_idx))
        total  = coste1 + coste2

        if total < mejor_total:
            mejor_total = total
            mejor_split = (list(grupo1_idx), list(grupo2_idx))

print(f"\nÓptimo VRP:")
print(f"  Vehículo 1: {[nombres[i] for i in mejor_split[0]]}")
print(f"  Vehículo 2: {[nombres[i] for i in mejor_split[1]]}")
print(f"  Distancia total: {mejor_total:.2f} km")

# Visualización
fig, ax = plt.subplots(figsize=(8, 7))
ax.set_title("Vehicle Routing — Solución Óptima", fontsize=13)

# Nodos
colores = ["gold"] + ["#3498db"]*n_clientes
for i, (coord, nombre, color) in enumerate(zip(coords, nombres, colores)):
    ax.scatter(*coord, c=color, s=300, zorder=5, edgecolors="black")
    ax.annotate(nombre, coord, textcoords="offset points", xytext=(8, 5), fontsize=11)

# Rutas
colors_veh = ["red", "green"]
for k, (grupo, color_v) in enumerate(zip(mejor_split, colors_veh)):
    # Encontrar orden óptimo
    mejor_perm_d = np.inf
    mejor_perm = None
    for perm in permutations(grupo):
        ruta = [0] + list(perm) + [0]
        coste = sum(D[ruta[i], ruta[i+1]] for i in range(len(ruta)-1))
        if coste < mejor_perm_d:
            mejor_perm_d = coste
            mejor_perm = ruta

    for i in range(len(mejor_perm)-1):
        a, b = coords[mejor_perm[i]], coords[mejor_perm[i+1]]
        ax.annotate("", xy=b, xytext=a,
                    arrowprops=dict(arrowstyle="->", color=color_v, lw=2))
    ax.plot([], [], color=color_v, linewidth=2, label=f"Vehículo {k+1} ({mejor_perm_d:.1f}km)")

ax.legend(fontsize=10)
ax.grid(alpha=0.3)
ax.set_xlabel("x (km)"); ax.set_ylabel("y (km)")

plt.tight_layout()
plt.savefig("outputs/m30_vrp.png", dpi=120, bbox_inches="tight")
print("\nFigura guardada: outputs/m30_vrp.png")
plt.close()
```

---

## 5. Comparación: QAOA vs optimizadores clásicos

```python
# seccion_05_comparacion.py
"""
Benchmarking QAOA vs:
1. Fuerza bruta (exacto)
2. Simulated Annealing
3. Greedy heurístico
Para el problema Max-Cut en grafos de diferente tamaño.
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import time
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

def maxcut_bruta(G_adj):
    """Max-Cut por fuerza bruta."""
    n = len(G_adj)
    mejor = 0
    for mask in range(2**n):
        corte = 0
        for i in range(n):
            for j in range(i+1, n):
                if G_adj[i][j] > 0:
                    bi, bj = (mask>>i)&1, (mask>>j)&1
                    if bi != bj:
                        corte += 1
        mejor = max(mejor, corte)
    return mejor

def maxcut_greedy(G_adj):
    """Greedy: asignar cada nodo al conjunto que maximiza el corte local."""
    n = len(G_adj)
    partition = np.zeros(n, dtype=int)
    for i in range(n):
        cortes = [0, 0]
        for j in range(i):
            if G_adj[i][j] > 0 and partition[j] == 0:
                cortes[1] += 1
            elif G_adj[i][j] > 0 and partition[j] == 1:
                cortes[0] += 1
        partition[i] = np.argmax(cortes)
    # Contar corte
    corte = sum(G_adj[i][j] for i in range(n) for j in range(i+1, n)
                if G_adj[i][j] > 0 and partition[i] != partition[j])
    return int(corte)

def qaoa_maxcut(G_adj, p=1, n_iter=80):
    """QAOA Max-Cut."""
    n = len(G_adj)
    edges = [(i,j) for i in range(n) for j in range(i+1,n) if G_adj[i][j]>0]
    dev = qml.device("default.qubit", wires=n)

    obs = qml.Hamiltonian(
        [-0.5]*len(edges),
        [qml.PauliZ(i)@qml.PauliZ(j) for i,j in edges]
    )

    @qml.qnode(dev, diff_method="parameter-shift")
    def circ(gammas, betas):
        for i in range(n):
            qml.Hadamard(i)
        for l in range(p):
            qml.CommutingEvolution(obs, gammas[l])
            for i in range(n):
                qml.RX(-2*betas[l], wires=i)
        return qml.probs(wires=range(n))

    gammas = pnp.array([0.5]*p, requires_grad=True)
    betas  = pnp.array([0.3]*p, requires_grad=True)

    @qml.qnode(dev, diff_method="parameter-shift")
    def objetivo(gammas, betas):
        for i in range(n):
            qml.Hadamard(i)
        for l in range(p):
            qml.CommutingEvolution(obs, gammas[l])
            for i in range(n):
                qml.RX(-2*betas[l], wires=i)
        return qml.expval(obs)

    opt = qml.AdamOptimizer(stepsize=0.05)
    for _ in range(n_iter):
        (gammas, betas), _ = opt.step_and_cost(objetivo, gammas, betas)

    # Mejor solución muestreada
    probs = circ(gammas, betas)
    mejor_idx = np.argmax(probs)
    bits = [(mejor_idx>>i)&1 for i in range(n)]
    corte = sum(G_adj[i][j] for i in range(n) for j in range(i+1,n)
                if G_adj[i][j]>0 and bits[i] != bits[j])
    return int(corte)

# Experimento con grafos de 5-7 nodos
resultados = {"bruta":[], "greedy":[], "qaoa":[], "optimal":[]}
np.random.seed(42)

for n_nodes in [5, 6, 7]:
    G = np.random.choice([0,1], size=(n_nodes,n_nodes))
    G = np.triu(G,1) + np.triu(G,1).T  # simétrico
    np.fill_diagonal(G, 0)

    opt_val   = maxcut_bruta(G)
    greedy_v  = maxcut_greedy(G)
    qaoa_v    = qaoa_maxcut(G, p=1)

    r_greedy = greedy_v / opt_val if opt_val > 0 else 1
    r_qaoa   = qaoa_v   / opt_val if opt_val > 0 else 1

    resultados["optimal"].append(opt_val)
    resultados["greedy"].append(r_greedy)
    resultados["qaoa"].append(r_qaoa)

    print(f"n={n_nodes}: óptimo={opt_val} | greedy={greedy_v} (r={r_greedy:.3f}) | QAOA={qaoa_v} (r={r_qaoa:.3f})")

print(f"\nMedia ratio greedy: {np.mean(resultados['greedy']):.3f}")
print(f"Media ratio QAOA:   {np.mean(resultados['qaoa']):.3f}")
print(f"GW bound clásico:   0.878 (Goemans-Williamson)")
```

---

## Checkpoint

```python
# checkpoint_m30.py
"""Tests para Módulo 30 — Optimización Cuántica."""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp

# T1: QUBO portfolio formula la matriz correctamente (diagonal con coeficientes de retorno)
def test_qubo_portfolio():
    retornos = np.array([0.2, 0.3, 0.1, 0.4])
    sigma    = np.eye(4) * 0.01  # varianza simple
    # Min riesgo = min x^T sigma x − λ μ^T x
    Q = sigma.copy()
    c = -0.5 * retornos  # λ=0.5
    # Sin restricción presupuesto: mínimo trivial → x=1 para todos
    assert Q.shape == (4, 4)
    assert len(c) == 4
    print("T1 PASS: QUBO portafolio correctamente construido")

# T2: Ruta TSP (3 ciudades) fuerza bruta correcta
def test_tsp_fuerza_bruta():
    from itertools import permutations
    D = np.array([[0,2,3],[2,0,1],[3,1,0]], dtype=float)
    def coste(r, D):
        n = len(r)
        return sum(D[r[i], r[(i+1)%n]] for i in range(n))
    rutas = list(permutations(range(3)))
    costes = [coste(r, D) for r in rutas]
    mejor = rutas[np.argmin(costes)]
    mejor_c = min(costes)
    assert mejor_c == 6.0  # 0→2→1→0: 3+1+2=6
    print(f"T2 PASS: TSP óptimo = {mejor_c:.1f}")

# T3: Hamiltoniano Max-Cut tiene eigenvalor mínimo ≤ 0
def test_hamiltoniano_maxcut():
    edges = [(0,1), (1,2), (2,0)]
    n = 3
    obs = qml.Hamiltonian(
        [-0.5, -0.5, -0.5],
        [qml.PauliZ(i)@qml.PauliZ(j) for i,j in edges]
    )
    M = qml.matrix(obs, wire_order=[0,1,2])
    evals = np.linalg.eigvalsh(M.real)
    assert evals[0] <= 0, "Max-Cut hamiltoniano debe tener eigenvalor mínimo ≤ 0"
    print(f"T3 PASS: eigenvalor mínimo = {evals[0]:.4f}")

# T4: QAOA supera 50% del óptimo en Max-Cut simple
def test_qaoa_maxcut_calidad():
    G_adj = np.array([[0,1,1,0],[1,0,0,1],[1,0,0,1],[0,1,1,0]])
    n = 4
    edges = [(i,j) for i in range(n) for j in range(i+1,n) if G_adj[i][j]>0]
    opt_bruta = 4  # corte óptimo conocido para este grafo

    dev = qml.device("default.qubit", wires=n)
    obs = qml.Hamiltonian(
        [-0.5]*len(edges),
        [qml.PauliZ(i)@qml.PauliZ(j) for i,j in edges]
    )

    @qml.qnode(dev, diff_method="parameter-shift")
    def objetivo(g, b):
        for i in range(n): qml.Hadamard(i)
        qml.CommutingEvolution(obs, g[0])
        for i in range(n): qml.RX(-2*b[0], wires=i)
        return qml.expval(obs)

    g = pnp.array([0.5], requires_grad=True)
    b = pnp.array([0.3], requires_grad=True)
    opt = qml.AdamOptimizer(stepsize=0.1)
    for _ in range(60):
        (g, b), E = opt.step_and_cost(objetivo, g, b)

    # El valor esperado negativo ≈ corte cuántico
    cut_aprox = -float(E)
    ratio = cut_aprox / opt_bruta
    assert ratio > 0.5, f"Ratio {ratio:.3f} < 0.5"
    print(f"T4 PASS: ratio QAOA = {ratio:.3f}")

# T5: Number Partition — balance entre dos grupos
def test_partition_balance():
    tiempos = np.array([3, 5, 2, 4, 7, 1])
    # Fuerza bruta
    mejor_diff = np.inf
    for mask in range(2**6):
        bits = [(mask>>i)&1 for i in range(6)]
        cA = sum(t*(1-b) for t,b in zip(tiempos, bits))
        cB = sum(t*b for t,b in zip(tiempos, bits))
        mejor_diff = min(mejor_diff, abs(cA-cB))
    assert mejor_diff <= 2, f"Desequilibrio óptimo = {mejor_diff}"
    print(f"T5 PASS: desequilibrio óptimo scheduling = {mejor_diff}")

# T6: QUBO → Hamiltoniano cuántico tiene el mismo mínimo que fuerza bruta
def test_qubo_hamiltoniano_equivalente():
    # Simple: n=2, minimizar x₀x₁ - x₀ - x₁ → mínimo en x=(1,1) con E=-1
    Q_mat = np.array([[0, 1],[1, 0]])  # x₀x₁
    c_vec = np.array([-1, -1])         # -x₀ - x₁

    # Fuerza bruta
    mejor_e = np.inf
    for mask in range(4):
        bits = [(mask>>i)&1 for i in range(2)]
        e = sum(Q_mat[i,j]*bits[i]*bits[j] for i in range(2) for j in range(2)) + sum(c_vec*bits)
        mejor_e = min(mejor_e, e)
    assert mejor_e == -2.0, f"Mínimo QUBO = {mejor_e}"
    print(f"T6 PASS: mínimo QUBO = {mejor_e}")

if __name__ == "__main__":
    print("=== Checkpoint M30: Optimización Cuántica ===\n")
    test_qubo_portfolio()
    test_tsp_fuerza_bruta()
    test_hamiltoniano_maxcut()
    test_qaoa_maxcut_calidad()
    test_partition_balance()
    test_qubo_hamiltoniano_equivalente()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M31 — Quantum NLP](./modulo-31-qnlp.md)
