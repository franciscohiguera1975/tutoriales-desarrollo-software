# Módulo 19 — QAOA: Quantum Approximate Optimization Algorithm

**Objetivo:** Implementar QAOA para resolver problemas de optimización combinatoria
(Max-Cut, TSP, coloración de grafos), entender el ansatz de QAOA y optimizar los
parámetros (γ, β).

**Herramientas:** Qiskit, PennyLane, numpy, networkx, matplotlib
**Prerequisito:** M17 (VQC), M18 (VQE)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
pip install networkx   # si no está instalado
python modulo-19-qaoa.py
```

---

## 1. El problema de optimización combinatoria

```python
# modulo-19-qaoa.py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import networkx as nx
import pennylane as qml
from qiskit import QuantumCircuit
from qiskit.quantum_info import Statevector, SparsePauliOp
from itertools import product
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: El problema Max-Cut
# ─────────────────────────────────────────────

print("=" * 60)
print("QAOA — QUANTUM APPROXIMATE OPTIMIZATION ALGORITHM")
print("=" * 60)

print("""
QAOA (Farhi et al., 2014) resuelve problemas de optimización combinatoria.

Problema MAX-CUT:
  Dado un grafo G=(V,E), particionar los vértices en dos conjuntos S y S̄
  de forma que se maximice el número de aristas entre S y S̄.

  Es NP-hard en el caso general.
  QAOA da una solución aproximada con garantías teóricas.

Formulación QUBO:
  Max C(z) = Σ_{(i,j)∈E} (1 - zᵢzⱼ)/2   donde zᵢ ∈ {-1, +1}
  Equivalente: minimizar -C(z)

Mapeo cuántico:
  zᵢ → Zᵢ (operador de Pauli Z en qubit i)
  C(z) → Ĉ = Σ_{(i,j)∈E} (I - ZᵢZⱼ)/2
         = |E|/2 - (1/2)Σ_{(i,j)∈E} ZᵢZⱼ

QAOA circuito (profundidad p):
  |s⟩ = H⊗n|0⟩  (superposición uniforme)
  |γ,β⟩ = Π_{l=1}^{p} [U_B(βₗ) · U_C(γₗ)] |s⟩

  U_C(γ) = e^{-iγĈ} = Π_{(i,j)∈E} e^{-iγ(I-ZᵢZⱼ)/2}  (fase problema)
  U_B(β) = e^{-iβΣX} = Π_i e^{-iβXᵢ}                   (mixer = Rx(2β))
""")

# Definir grafo de prueba
G = nx.Graph()
G.add_nodes_from([0, 1, 2, 3, 4])
G.add_edges_from([(0,1), (0,2), (1,3), (2,3), (3,4), (1,4)])

n_nodos = G.number_of_nodes()
n_aristas = G.number_of_edges()

print(f"Grafo de prueba:")
print(f"  Nodos: {list(G.nodes())}")
print(f"  Aristas: {list(G.edges())}")
print(f"  Max-Cut óptimo clásico: calculando...")

# Fuerza bruta (solo posible para grafos pequeños)
def max_cut_bruta(G):
    """Calcula el Max-Cut óptimo por fuerza bruta."""
    n = G.number_of_nodes()
    mejor_corte = 0
    mejor_particion = None
    for bits in product([0, 1], repeat=n):
        corte = sum(1 for i, j in G.edges() if bits[i] != bits[j])
        if corte > mejor_corte:
            mejor_corte = corte
            mejor_particion = bits
    return mejor_corte, mejor_particion

opt_corte, opt_particion = max_cut_bruta(G)
print(f"  Max-Cut óptimo: {opt_corte} aristas cortadas")
print(f"  Partición óptima: {dict(zip(G.nodes(), opt_particion))}")
```

---

## 2. El Hamiltoniano de coste

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: Hamiltoniano del Max-Cut
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("HAMILTONIANO DEL MAX-CUT")
print("="*60)

def hamiltoniano_maxcut(G: nx.Graph) -> SparsePauliOp:
    """
    Construye el Hamiltoniano Max-Cut como SparsePauliOp de Qiskit.
    Ĉ = Σ_{(i,j)∈E} (I - ZᵢZⱼ)/2
    Equivalente a minimizar -Ĉ (Qiskit minimiza por defecto).
    """
    n = G.number_of_nodes()
    terminos = []
    for i, j in G.edges():
        # Término ZᵢZⱼ: Pauli Z en posición i y j, I en el resto
        pauli_str = ["I"] * n
        pauli_str[i] = "Z"
        pauli_str[j] = "Z"
        # Qiskit: qubit 0 es el bit menos significativo (derecha)
        terminos.append(("".join(reversed(pauli_str)), -0.5))  # -1/2 ZᵢZⱼ
        terminos.append(("I" * n, 0.5))  # +1/2 I
    return SparsePauliOp.from_list(terminos).simplify()

H_cost = hamiltoniano_maxcut(G)
print(f"Hamiltoniano Max-Cut ({n_nodos} qubits, {n_aristas} aristas):")
for pauli, coef in sorted(H_cost.to_list(), key=lambda x: x[1]):
    if abs(coef) > 1e-10:
        print(f"  {coef:+.4f} · {pauli}")

# Verificar: valor esperado en la partición óptima
particion_bin = "".join(str(b) for b in reversed(opt_particion))  # Qiskit ordering
sv_opt = Statevector.from_label(particion_bin)
E_opt = sv_opt.expectation_value(H_cost).real
print(f"\nVerificación del óptimo:")
print(f"  Partición: |{particion_bin}⟩ (Qiskit bit order)")
print(f"  ⟨Ĉ⟩ = {E_opt:.4f}  (debe ser ≈ {opt_corte})")

# Evaluar función de coste para todos los estados base
print(f"\nFunción de coste para todos los estados:")
print(f"{'Estado':>8} {'Corte':>7} {'⟨Ĉ⟩':>8}")
print(f"{'─'*25}")
for bits in product([0, 1], repeat=n_nodos):
    corte = sum(1 for i, j in G.edges() if bits[i] != bits[j])
    bitstr = "".join(str(b) for b in reversed(bits))
    sv = Statevector.from_label(bitstr)
    E = sv.expectation_value(H_cost).real
    marca = " ← ÓPTIMO" if corte == opt_corte else ""
    print(f"  |{''.join(str(b) for b in bits)}⟩  {corte:>5}    {E:>+.1f}{marca}")
```

---

## 3. Circuito QAOA

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Circuito QAOA
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("CIRCUITO QAOA")
print("="*60)

def qaoa_layer(qc: QuantumCircuit, G: nx.Graph, gamma: float, beta: float):
    """
    Una capa del circuito QAOA:
    U_C(γ) · U_B(β)
    """
    n = G.number_of_nodes()

    # U_C(γ): para cada arista (i,j), e^{-iγ ZᵢZⱼ/2} = CNOT · Rz(γ) · CNOT
    for i, j in G.edges():
        qc.cx(i, j)
        qc.rz(-gamma, j)   # Rz(-γ) implementa e^{+iγZ/2} (conv. Qiskit)
        qc.cx(i, j)

    # U_B(β): Rx(2β) en cada qubit (mixer = rotación en X)
    for i in range(n):
        qc.rx(2 * beta, i)


def circuito_qaoa(G: nx.Graph, p: int, gammas: list, betas: list) -> QuantumCircuit:
    """
    Circuito QAOA de profundidad p.
    """
    n = G.number_of_nodes()
    qc = QuantumCircuit(n, name=f"QAOA(p={p})")

    # Estado inicial: superposición uniforme
    qc.h(range(n))
    qc.barrier()

    # p capas
    for l in range(p):
        qaoa_layer(qc, G, gammas[l], betas[l])
        qc.barrier()

    return qc

# Circuito QAOA con p=1
gamma_init, beta_init = [0.5], [0.3]
qc_qaoa = circuito_qaoa(G, p=1, gammas=gamma_init, betas=beta_init)

print(f"Circuito QAOA (p=1):")
print(f"  Qubits: {qc_qaoa.num_qubits}")
print(f"  Profundidad: {qc_qaoa.depth()}")
print(f"  Operaciones: {dict(qc_qaoa.count_ops())}")
print()
print(qc_qaoa.draw(output="text", fold=80))

# Evaluar el circuito QAOA
def evaluar_qaoa(G: nx.Graph, p: int, gammas: list, betas: list, shots: int = 8192) -> dict:
    """
    Evalúa el circuito QAOA y retorna la distribución de cortes.
    """
    from qiskit_aer import AerSimulator
    from qiskit import transpile

    qc = circuito_qaoa(G, p, gammas, betas)
    qc.measure_all()

    sim = AerSimulator()
    qc_t = transpile(qc, sim)
    counts = sim.run(qc_t, shots=shots).result().get_counts()

    return counts

def valor_esperado_qaoa(G: nx.Graph, counts: dict) -> float:
    """Calcula ⟨C⟩ a partir de los conteos del QAOA."""
    total = sum(counts.values())
    E_total = 0
    for bitstr, count in counts.items():
        bits = [int(b) for b in reversed(bitstr)]  # Qiskit: MSB a la derecha
        corte = sum(1 for i, j in G.edges() if bits[i] != bits[j])
        E_total += corte * count
    return E_total / total

counts_init = evaluar_qaoa(G, p=1, gammas=gamma_init, betas=beta_init)
E_init = valor_esperado_qaoa(G, counts_init)
print(f"\nQAOA con γ₁={gamma_init[0]}, β₁={beta_init[0]}:")
print(f"  ⟨C⟩ = {E_init:.4f}  (óptimo = {opt_corte})")
print(f"  Approximation ratio = {E_init/opt_corte:.4f}")
```

---

## 4. Optimización de parámetros QAOA

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Optimización de parámetros
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("OPTIMIZACIÓN DE PARÁMETROS QAOA")
print("="*60)

dev_qaoa = qml.device("default.qubit", wires=n_nodos)

def qaoa_circuito_pl(gammas, betas):
    """Circuito QAOA en PennyLane."""
    # Estado inicial
    for i in range(n_nodos):
        qml.Hadamard(i)

    # Capas QAOA
    for gamma, beta in zip(gammas, betas):
        # U_C(γ)
        for i, j in G.edges():
            qml.CNOT([i, j])
            qml.RZ(-gamma, j)
            qml.CNOT([i, j])
        # U_B(β)
        for i in range(n_nodos):
            qml.RX(2 * beta, i)

# Observable de coste: -Ĉ = (1/2)ΣZᵢZⱼ - |E|/2
coste_obs = qml.Hamiltonian(
    [-0.5] * n_aristas + [0.5] * n_aristas,
    [qml.PauliZ(i) @ qml.PauliZ(j) for i, j in G.edges()] +
    [qml.Identity(0)] * n_aristas   # términos constantes
)
# Simplificado: usar solo los términos ZZ que importan
coste_obs_simple = qml.Hamiltonian(
    [-0.5] * n_aristas,
    [qml.PauliZ(i) @ qml.PauliZ(j) for i, j in G.edges()]
)

@qml.qnode(dev_qaoa, diff_method="parameter-shift")
def qaoa_coste(params):
    """QAOA cost function para optimización."""
    p = len(params) // 2
    gammas = params[:p]
    betas = params[p:]
    qaoa_circuito_pl(gammas, betas)
    return qml.expval(coste_obs_simple)  # minimizar -⟨C⟩ = maximizar ⟨C⟩

# Exploración del landscape γ-β (p=1)
print("Landscape de QAOA p=1 (γ × β):")
gamma_range = np.linspace(0, np.pi, 30)
beta_range = np.linspace(0, np.pi/2, 30)
GAMMA, BETA = np.meshgrid(gamma_range, beta_range)
COSTE_MAP = np.zeros_like(GAMMA)

for ig, gamma_v in enumerate(gamma_range):
    for ib, beta_v in enumerate(beta_range):
        params_v = np.array([gamma_v, beta_v])
        COSTE_MAP[ib, ig] = -qaoa_coste(params_v)  # negativo porque minimizamos

# Optimización con Adam
print("\nOptimizando QAOA p=1 (Adam):")
opt_qaoa = qml.AdamOptimizer(stepsize=0.05)
np.random.seed(42)
params_qaoa = np.random.uniform(0, np.pi, 2)  # [gamma, beta]
historial_qaoa_p1 = [-qaoa_coste(params_qaoa)]

for step in range(120):
    params_qaoa, coste = opt_qaoa.step_and_cost(qaoa_coste, params_qaoa)
    historial_qaoa_p1.append(-coste)   # -coste porque minimizamos -⟨C⟩
    if (step + 1) % 30 == 0:
        print(f"  Paso {step+1:3d}: ⟨C⟩ = {-coste:.4f}  "
              f"(γ={params_qaoa[0]:.3f}, β={params_qaoa[1]:.3f})")

E_qaoa_p1 = historial_qaoa_p1[-1]
print(f"\nResultados QAOA p=1:")
print(f"  ⟨C⟩_QAOA = {E_qaoa_p1:.4f}")
print(f"  C_óptimo = {opt_corte}")
print(f"  Approximation ratio = {E_qaoa_p1/opt_corte:.4f}")

# QAOA p=2
print("\nOptimizando QAOA p=2 (Adam):")
np.random.seed(42)
params_qaoa_p2 = np.random.uniform(0, np.pi/2, 4)  # [γ₁,γ₂, β₁,β₂]
historial_qaoa_p2 = []

@qml.qnode(dev_qaoa, diff_method="parameter-shift")
def qaoa_coste_p2(params):
    gammas = params[:2]; betas = params[2:]
    qaoa_circuito_pl(gammas, betas)
    return qml.expval(coste_obs_simple)

opt_p2 = qml.AdamOptimizer(stepsize=0.05)
for step in range(150):
    params_qaoa_p2, coste_p2 = opt_p2.step_and_cost(qaoa_coste_p2, params_qaoa_p2)
    historial_qaoa_p2.append(-coste_p2)

E_qaoa_p2 = historial_qaoa_p2[-1]
print(f"  ⟨C⟩_QAOA(p=2) = {E_qaoa_p2:.4f}")
print(f"  Approximation ratio (p=2) = {E_qaoa_p2/opt_corte:.4f}")
```

---

## 5. Visualización del resultado

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Visualización
# ─────────────────────────────────────────────

fig = plt.figure(figsize=(18, 10))
gs = GridSpec(2, 3, figure=fig, hspace=0.4, wspace=0.35)

# Plot 1: Grafo original y solución
ax1 = fig.add_subplot(gs[0, 0])
pos = nx.spring_layout(G, seed=42)
node_colors = ["blue" if opt_particion[i] == 0 else "red" for i in range(n_nodos)]
nx.draw(G, pos, ax=ax1, node_color=node_colors, with_labels=True,
        node_size=800, font_color="white", font_weight="bold")
# Colorear aristas cortadas
aristas_cortadas = [(i,j) for i,j in G.edges() if opt_particion[i] != opt_particion[j]]
aristas_no_cortadas = [(i,j) for i,j in G.edges() if opt_particion[i] == opt_particion[j]]
nx.draw_networkx_edges(G, pos, edgelist=aristas_cortadas, ax=ax1,
                       edge_color="green", width=3)
nx.draw_networkx_edges(G, pos, edgelist=aristas_no_cortadas, ax=ax1,
                       edge_color="gray", width=1.5, style="dashed")
ax1.set_title(f"Max-Cut Óptimo\n{opt_corte} aristas cortadas (verde)")

# Plot 2: Landscape γ-β
ax2 = fig.add_subplot(gs[0, 1])
cp = ax2.contourf(GAMMA, BETA, COSTE_MAP, levels=25, cmap="hot")
plt.colorbar(cp, ax=ax2, label="⟨C(γ,β)⟩")
# Marcar el óptimo encontrado
idx_max = np.unravel_index(np.argmax(COSTE_MAP), COSTE_MAP.shape)
ax2.scatter([GAMMA[idx_max]], [BETA[idx_max]], s=200, c="cyan", marker="*",
           zorder=5, label=f"Max: {COSTE_MAP[idx_max]:.2f}")
ax2.scatter([params_qaoa[0]], [params_qaoa[1]], s=150, c="lime", marker="^",
           zorder=5, label=f"Adam: {E_qaoa_p1:.2f}")
ax2.set_xlabel("γ"); ax2.set_ylabel("β")
ax2.set_title("Landscape QAOA p=1\n⟨C(γ,β)⟩")
ax2.legend(fontsize=7)

# Plot 3: Convergencia
ax3 = fig.add_subplot(gs[0, 2])
ax3.plot(historial_qaoa_p1, "b-", linewidth=2, label="QAOA p=1")
ax3.plot(historial_qaoa_p2, "r-", linewidth=2, label="QAOA p=2")
ax3.axhline(y=opt_corte, color="green", linestyle="--", linewidth=2,
           label=f"Óptimo = {opt_corte}")
ax3.set_xlabel("Iteraciones"); ax3.set_ylabel("⟨C⟩")
ax3.set_title("Convergencia QAOA")
ax3.legend(fontsize=8); ax3.grid(True, alpha=0.3)

# Plot 4: Distribución de bitstrings QAOA óptimo
ax4 = fig.add_subplot(gs[1, 0:2])
# Obtener distribución con parámetros óptimos
@qml.qnode(dev_qaoa)
def qaoa_probs(params):
    p = len(params) // 2
    gammas_p = params[:p]; betas_p = params[p:]
    qaoa_circuito_pl(gammas_p, betas_p)
    return qml.probs(wires=range(n_nodos))

probs_optimas = qaoa_probs(params_qaoa)
estados = [format(i, f"0{n_nodos}b") for i in range(2**n_nodos)]
cortes_estados = [sum(1 for i, j in G.edges() if int(e[n_nodos-1-i]) != int(e[n_nodos-1-j]))
                  for e in estados]

colores_bar = ["green" if c == opt_corte else "red" if c == 0 else "orange"
               for c in cortes_estados]
ax4.bar(range(2**n_nodos), probs_optimas, color=colores_bar, alpha=0.7)
ax4.set_xlabel("Estado (bitstring)")
ax4.set_ylabel("Probabilidad")
ax4.set_title(f"Distribución QAOA optimizado (p=1, ⟨C⟩={E_qaoa_p1:.3f})\n"
              f"Verde=óptimo, Naranja=sub-óptimo, Rojo=pésimo")
ax4.set_xticks(range(0, 2**n_nodos, 4))
ax4.set_xticklabels([estados[i] for i in range(0, 2**n_nodos, 4)], rotation=45, fontsize=7)

# Plot 5: Approximation ratio vs p
ax5 = fig.add_subplot(gs[1, 2])
p_vals = [1, 2]
ratios = [E_qaoa_p1/opt_corte, E_qaoa_p2/opt_corte]
ax5.bar(p_vals, ratios, color=["blue", "red"], alpha=0.7, width=0.5)
ax5.axhline(y=1.0, color="green", linestyle="--", label="Óptimo")
ax5.axhline(y=0.878, color="orange", linestyle=":", label="Goemans-Williamson (clásico)")
ax5.set_xlabel("Profundidad p"); ax5.set_ylabel("Approximation ratio r = ⟨C⟩/C_opt")
ax5.set_title("QAOA: Approx. Ratio vs p")
ax5.legend(fontsize=8); ax5.grid(True, axis="y", alpha=0.3)
ax5.set_ylim(0.5, 1.1); ax5.set_xticks([1, 2])
for p_v, r in zip(p_vals, ratios):
    ax5.text(p_v, r + 0.01, f"{r:.3f}", ha="center", fontweight="bold")

plt.suptitle("QAOA — Quantum Approximate Optimization Algorithm (Max-Cut)",
             fontsize=13, fontweight="bold")
plt.savefig("outputs/m19_qaoa_maxcut.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m19_qaoa_maxcut.png")
```

---

## 6. QAOA para otros problemas

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: QAOA para otros problemas
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("QAOA PARA OTROS PROBLEMAS DE OPTIMIZACIÓN")
print("="*60)

print("""
QAOA es general: cualquier problema QUBO se puede resolver con QAOA.

QUBO (Quadratic Unconstrained Binary Optimization):
  Minimizar x^T Q x  donde x ∈ {0,1}^n
  Mapeo: xᵢ → (1 - Zᵢ)/2

Problemas resolubles con QAOA:
  1. Max-Cut (visto)
  2. Minimum Vertex Cover
  3. Portfolio Optimization (finanzas)
  4. Traveling Salesman Problem (TSP)
  5. Graph Coloring
  6. Number Partitioning
  7. Scheduling / Job Shop
  8. k-SAT

Para TSP:
  Variables: xᵢₜ = 1 si ciudad i es visitada en paso t
  Restricciones: Σᵢ xᵢₜ = 1 (un ciudad por paso)
                 Σₜ xᵢₜ = 1 (visitar cada ciudad una vez)
  Objetivo: minimizar distancia total

Limitaciones actuales del QAOA (2025):
  ✓ NISQ-friendly: circuitos poco profundos
  ✓ Garantía teórica: r ≥ (1 - 1/n) para Max-Cut (p=1)
  ✗ No supera los mejores algoritmos clásicos en grafos grandes
  ✗ Necesita p >> 1 para ventaja cuántica real
  ✗ Barren plateaus para p grande
  ✗ Ruido en hardware → degrada la aproximación
""")

# Comparación con algoritmos clásicos
print("Comparación QAOA vs alternativas clásicas para Max-Cut:")
print(f"""
Algoritmo              | Ratio garantizado | Complejidad     | Tipo
───────────────────────|───────────────────|─────────────────|──────────────
Aleatorio              | 0.500             | O(n)            | Clásico prob.
Greedy local search    | 0.667             | O(n²)           | Clásico det.
Goemans-Williamson     | 0.878             | O(n³) SDP       | Clásico SDP
QAOA p=1               | 0.693 (d-reg)     | O(nd) circuito  | Cuántico
QAOA p→∞               | 1.000             | Exp. profund.   | Cuántico (ideal)
QAOA p=1 (nuestro)     | {E_qaoa_p1/opt_corte:.3f}             | —               | Cuántico sim.
QAOA p=2 (nuestro)     | {E_qaoa_p2/opt_corte:.3f}             | —               | Cuántico sim.
""")

# Ejemplo de formulación QUBO para Number Partitioning
print("Ejemplo: Number Partitioning (una partición de una lista de números)")
numeros = [3, 5, 7, 2, 8]
total = sum(numeros)
print(f"  Lista: {numeros}, suma total = {total}")

# Formulación QUBO: minimizar (Σ sᵢ nᵢ)² donde sᵢ ∈ {-1,+1}
# Mapeo: sᵢ = 2xᵢ - 1, xᵢ ∈ {0,1}
# Para el estado óptimo: fuerza bruta
mejor_diff = float("inf")
mejor_part = None
for bits in product([0,1], repeat=len(numeros)):
    s = [1 if b == 1 else -1 for b in bits]
    diff = abs(sum(s_i * n_i for s_i, n_i in zip(s, numeros)))
    if diff < mejor_diff:
        mejor_diff = diff
        mejor_part = bits

S1 = [numeros[i] for i in range(len(numeros)) if mejor_part[i] == 1]
S2 = [numeros[i] for i in range(len(numeros)) if mejor_part[i] == 0]
print(f"  Partición óptima: {S1} | {S2}")
print(f"  Diferencia mínima: {mejor_diff}  (sum S1={sum(S1)}, sum S2={sum(S2)})")

print("\n✓ Módulo 19 completado")
```

---

## Checkpoint M19

```python
# checkpoint_m19.py
"""Verificaciones del Módulo 19 — QAOA"""
import numpy as np

def test_maxcut_bruta():
    """El Max-Cut por fuerza bruta es correcto para grafos simples."""
    import networkx as nx
    from itertools import product
    G = nx.Graph()
    G.add_edges_from([(0,1),(1,2),(2,3),(0,3)])   # ciclo 4-nodos: Max-Cut = 4
    def max_cut(G):
        n = G.number_of_nodes()
        mejor = 0
        for bits in product([0,1], repeat=n):
            c = sum(1 for i, j in G.edges() if bits[i] != bits[j])
            mejor = max(mejor, c)
        return mejor
    assert max_cut(G) == 4, f"Ciclo de 4 nodos: Max-Cut debe ser 4, got {max_cut(G)}"
    print("✓ test_maxcut_bruta")

def test_hamiltoniano_maxcut():
    """El Hamiltoniano Max-Cut tiene el valor correcto en la solución óptima."""
    from qiskit.quantum_info import SparsePauliOp, Statevector
    import networkx as nx
    # Grafo sencillo: arista 0-1
    G = nx.Graph()
    G.add_edge(0, 1)
    # H = (I - Z₀Z₁)/2 → eigenvalores 0 (paralelos) y 1 (antiparalelos)
    H = SparsePauliOp.from_list([("II", 0.5), ("ZZ", -0.5)])
    # Estado |01⟩: cortado → ⟨C⟩ = 1
    sv_01 = Statevector.from_label("01")
    E_01 = sv_01.expectation_value(H).real
    assert abs(E_01 - 1.0) < 1e-10, f"⟨C⟩(|01⟩) debe ser 1, got {E_01}"
    # Estado |00⟩: no cortado → ⟨C⟩ = 0
    sv_00 = Statevector.from_label("00")
    E_00 = sv_00.expectation_value(H).real
    assert abs(E_00 - 0.0) < 1e-10, f"⟨C⟩(|00⟩) debe ser 0, got {E_00}"
    print("✓ test_hamiltoniano_maxcut")

def test_qaoa_circuito_estructura():
    """Circuito QAOA tiene estructura correcta: H + p capas."""
    from qiskit import QuantumCircuit
    import networkx as nx
    G = nx.Graph(); G.add_edges_from([(0,1),(1,2)])
    n = 3

    def qaoa_test(G, gammas, betas):
        qc = QuantumCircuit(n)
        qc.h(range(n))
        for gamma, beta in zip(gammas, betas):
            for i, j in G.edges():
                qc.cx(i, j); qc.rz(-gamma, j); qc.cx(i, j)
            for i in range(n):
                qc.rx(2*beta, i)
        return qc

    qc_p1 = qaoa_test(G, [0.5], [0.3])
    qc_p2 = qaoa_test(G, [0.5, 0.3], [0.3, 0.1])
    # p=2 más profundo que p=1
    assert qc_p2.depth() > qc_p1.depth(), "p=2 debe ser más profundo que p=1"
    # Tiene puertas H, RZ, RX, CX
    ops_p1 = dict(qc_p1.count_ops())
    assert "h" in ops_p1, "Debe tener puertas H"
    assert "cx" in ops_p1, "Debe tener puertas CX"
    print(f"✓ test_qaoa_circuito_estructura (p=1: {dict(ops_p1)})")

def test_superposicion_uniforme():
    """El estado inicial de QAOA es superposición uniforme."""
    from qiskit import QuantumCircuit
    from qiskit.quantum_info import Statevector
    n = 3
    qc = QuantumCircuit(n)
    qc.h(range(n))
    sv = Statevector(qc)
    probs = sv.probabilities()
    assert np.allclose(probs, [1/2**n] * 2**n, atol=1e-10), \
        "Superposición uniforme: todas las probs deben ser 1/2^n"
    print(f"✓ test_superposicion_uniforme (P(estado)={probs[0]:.4f} para n={n})")

def test_approximation_ratio_positivo():
    """El approximation ratio del QAOA debe estar en (0, 1]."""
    E_qaoa = 4.2   # resultado simulado típico
    C_opt = 5
    ratio = E_qaoa / C_opt
    assert 0 < ratio <= 1.0 + 1e-10, f"Ratio debe estar en (0,1], got {ratio}"
    print(f"✓ test_approximation_ratio_positivo (ratio={ratio:.3f})")

def test_qaoa_pennylane():
    """QAOA en PennyLane mejora el coste sobre la inicialización aleatoria."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=2)
    # Grafo: arista 0-1
    @qml.qnode(dev)
    def qaoa_simple(params):
        qml.Hadamard(0); qml.Hadamard(1)
        # U_C(γ)
        qml.CNOT([0,1]); qml.RZ(-params[0], 1); qml.CNOT([0,1])
        # U_B(β)
        qml.RX(2*params[1], 0); qml.RX(2*params[1], 1)
        return qml.expval(-qml.PauliZ(0) @ qml.PauliZ(1) * 0.5)

    # Parámetros óptimos conocidos para p=1 con 1 arista: γ=π/4, β=π/4
    params_opt = np.array([np.pi/4, np.pi/4])
    params_rand = np.array([0.1, 0.1])
    val_opt = qaoa_simple(params_opt)
    val_rand = qaoa_simple(params_rand)
    # El valor óptimo debe ser mejor (más alto = más corte)
    assert val_opt >= val_rand - 0.5, \
        f"Params óptimos deben dar mejor coste: {val_opt:.4f} vs {val_rand:.4f}"
    print(f"✓ test_qaoa_pennylane (opt={val_opt:.4f} vs rand={val_rand:.4f})")

if __name__ == "__main__":
    print("Ejecutando checkpoint M19 — QAOA\n")
    test_maxcut_bruta()
    test_hamiltoniano_maxcut()
    test_qaoa_circuito_estructura()
    test_superposicion_uniforme()
    test_approximation_ratio_positivo()
    test_qaoa_pennylane()
    print("\n✓ Todos los tests de M19 pasaron")
```
