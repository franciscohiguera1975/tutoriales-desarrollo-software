# Módulo 28 — Quantum Reinforcement Learning

**Objetivo:** Implementar agentes de aprendizaje por refuerzo con políticas cuánticas (QRL). Comparar Q-Learning cuántico con el clásico, y explorar cómo un agente cuántico puede representar políticas en superposición.

**Herramientas:** PennyLane 0.39+, PyTorch 2.x, NumPy, Gymnasium (OpenAI Gym)
**Prerequisito:** M23 (QNN), M24 (Híbrido PyTorch)

---

## Ejecución

```bash
conda activate quantum
pip install gymnasium
mkdir -p outputs
python seccion_X.py
```

---

## 1. Fundamentos: RL y política cuántica

```python
# seccion_01_fundamentos.py
"""
Reinforcement Learning estándar:
  Estado s_t → Política π(a|s) → Acción a_t → Recompensa r_t → Estado s_{t+1}

Quantum RL (QRL):
  El agente tiene una política CUÁNTICA π_θ(a|s):
  - Estado clásico s → codificado en qubits
  - Red neuronal cuántica produce distribución de acciones
  - Parámetros θ optimizados con backpropagation (Policy Gradient)

Ventajas potenciales:
  1. Espacio de políticas exponencialmente grande (2^n estados cuánticos)
  2. Superposición: evaluar múltiples acciones en paralelo
  3. Entrelazamiento: correlaciones complejas entre dimensiones del estado

Limitaciones actuales (2024):
  - No hay ventaja demostrada sobre RL clásico en entornos estándar
  - El entorno sigue siendo clásico (lectura de estado = measurement = colapso)
  - El botton neck es la interfaz clásica-cuántica
"""

print("=== Quantum RL — Conceptos ===")

# Diagrama del ciclo RL cuántico
ciclo = """
   ┌─────────────────────────────────────────────────────┐
   │  Entorno (clásico: CartPole, FrozenLake, etc.)       │
   │                                                       │
   │  s_t ──→ [Encoding cuántico] ──→ |ψ(s_t)⟩           │
   │                                      │                │
   │                              [Circuito variacional]   │
   │                               U_θ(s_t)|0⟩            │
   │                                      │                │
   │                           [Medición] ⟨Z_i⟩           │
   │                                      │                │
   │               a_t = argmax_i ⟨Z_i⟩  │                │
   │                     ↓                                 │
   │              r_t, s_{t+1}                             │
   └─────────────────────────────────────────────────────┘

Optimización: REINFORCE / Policy Gradient
   ∇_θ J(θ) = E[∇_θ log π_θ(a|s) · G_t]
   donde G_t = Σ_{k=t}^{T} γ^{k-t} r_k (return)
"""
print(ciclo)

# Parámetros del entorno CartPole simplificado
print("Entorno CartPole (simplificado para QRL):")
print("  Estado:  (x, ẋ, θ, θ̇) — 4 dimensiones continuas")
print("  Acciones: {0: izquierda, 1: derecha}")
print("  Recompensa: +1 por cada step sin caer (máx 500)")
print("  Terminación: |θ| > 12° o |x| > 2.4")
```

---

## 2. Entorno FrozenLake discreto

Usamos FrozenLake (estado discreto) para QRL porque el encoding es directo.

```python
# seccion_02_entorno.py
"""
FrozenLake 4x4 (Gymnasium):
- 16 estados discretos (0..15) → codificados en 4 qubits (basis encoding)
- 4 acciones: {0:izq, 1:abajo, 2:der, 3:arriba}
- Recompensa: +1 al llegar a la meta, 0 en otros pasos
- Stochastic: puede deslizarse en dirección perpendicular

Mapa:
  SFFF    S=Start, F=Frozen, H=Hole, G=Goal
  FHFH
  FFFH
  HFFG
"""
import numpy as np

# Implementación manual de FrozenLake 4x4 (sin gymnasium para portabilidad)
MAPA = [
    "SFFF",
    "FHFH",
    "FFFH",
    "HFFG"
]
N_ESTADOS  = 16
N_ACCIONES = 4

def estado_a_pos(s):
    return s // 4, s % 4

def pos_a_estado(r, c):
    return r * 4 + c

def siguiente_estado(s, a, determinista=True):
    """
    s: estado actual (0-15)
    a: acción (0:izq, 1:abajo, 2:der, 3:arriba)
    determinista: False → ruido estocástico (1/3 de probabilidad de deslizarse)
    """
    r, c = estado_a_pos(s)
    tipo = MAPA[r][c]

    if tipo in "HG":  # Terminal
        return s, 0, True

    # Dirección con posible deslizamiento
    dirs = [(0,-1), (1,0), (0,1), (-1,0)]  # izq, abajo, der, arriba

    if not determinista:
        # Escoger aleatoriamente entre [a-1, a, a+1] mod 4
        a_real = np.random.choice([a, (a-1)%4, (a+1)%4], p=[1/3, 1/3, 1/3])
    else:
        a_real = a

    dr, dc = dirs[a_real]
    nr, nc = max(0, min(3, r+dr)), max(0, min(3, c+dc))
    s_next = pos_a_estado(nr, nc)
    tipo_next = MAPA[nr][nc]

    recompensa = 1.0 if tipo_next == "G" else 0.0
    terminado  = tipo_next in "HG"

    return s_next, recompensa, terminado

class FrozenLakeEnv:
    """Entorno FrozenLake 4x4 determinista simplificado."""
    def __init__(self, determinista=True):
        self.determinista = determinista
        self.estado = 0

    def reset(self):
        self.estado = 0
        return self.estado

    def step(self, accion):
        s_next, r, done = siguiente_estado(self.estado, accion, self.determinista)
        self.estado = s_next
        return s_next, r, done

    def render(self):
        grid = [list(row) for row in MAPA]
        r, c = estado_a_pos(self.estado)
        grid[r][c] = "A"
        for row in grid:
            print(" ".join(row))
        print()

# Verificar entorno
env = FrozenLakeEnv(determinista=True)
estado = env.reset()
print("=== FrozenLake 4x4 ===")
print(f"Estado inicial: {estado} (posición {estado_a_pos(estado)})")
env.render()

# Probar secuencia de acciones para llegar a la meta (ruta óptima)
# 0→1→2→6→10→14→15 (acciones: abajo, der, der, abajo, abajo, der)
ruta = [1, 2, 2, 1, 1, 2]
estado = env.reset()
total_r = 0
for a in ruta:
    s_next, r, done = env.step(a)
    print(f"  Acción {['izq','abajo','der','arriba'][a]}: s={s_next} r={r} done={done}")
    total_r += r
    if done:
        break
print(f"Recompensa total ruta óptima: {total_r}")
```

---

## 3. Política cuántica

```python
# seccion_03_politica_cuantica.py
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import torch
import torch.nn as nn

N_QUBITS   = 4   # log2(16 estados) = 4 qubits
N_ACCIONES = 4
dev = qml.device("default.qubit", wires=N_QUBITS)

def basis_encode(estado, n_qubits):
    """
    Codifica estado entero en base binaria.
    estado=5 → '0101' → X en qubits 0 y 2
    """
    bits = format(estado, f"0{n_qubits}b")
    for i, b in enumerate(bits):
        if b == "1":
            qml.PauliX(wires=i)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def politica_cuantica(estado, theta):
    """
    Política cuántica π_θ(a|s).
    estado: entero [0, 15]
    theta:  [n_capas, N_QUBITS, 3] parámetros variacionales
    Retorna: [⟨Z0⟩, ⟨Z1⟩, ⟨Z2⟩, ⟨Z3⟩] → logits de acciones
    """
    # Encoding del estado
    basis_encode(estado, N_QUBITS)

    # Ansatz variacional
    qml.StronglyEntanglingLayers(theta, wires=range(N_QUBITS))

    # Medición: 4 valores esperados → logits para 4 acciones
    return [qml.expval(qml.PauliZ(i)) for i in range(N_ACCIONES)]

class AgenteCuantico(nn.Module):
    """Agente QRL con política cuántica y cabeza softmax."""
    def __init__(self, n_capas=3):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape(n_capas, N_QUBITS)
        self.theta = nn.Parameter(torch.randn(shape) * 0.1)

    def forward(self, estado):
        """Retorna logits de las 4 acciones."""
        logits = torch.stack(politica_cuantica(estado, self.theta))
        return logits

    def accion(self, estado):
        """Selecciona acción vía softmax (estocástico)."""
        with torch.no_grad():
            logits = self.forward(estado)
            probs = torch.softmax(logits, dim=0)
        dist = torch.distributions.Categorical(probs)
        a = dist.sample()
        return a.item(), dist.log_prob(a)

    def accion_greedy(self, estado):
        """Selecciona la acción con mayor logit (determinista)."""
        with torch.no_grad():
            logits = self.forward(estado)
        return torch.argmax(logits).item()

# Verificar agente
agente = AgenteCuantico(n_capas=3)
print("=== Agente Cuántico QRL ===")
for s in [0, 5, 10, 15]:
    a, lp = agente.accion(s)
    nombres_acc = ["←", "↓", "→", "↑"]
    print(f"  Estado {s:2d}: acción={nombres_acc[a]} log_prob={lp.item():.4f}")
print(f"\nParámetros: {sum(p.numel() for p in agente.parameters())}")
```

---

## 4. Entrenamiento REINFORCE

```python
# seccion_04_reinforce.py
"""
Algoritmo REINFORCE (Monte Carlo Policy Gradient):
1. Ejecutar episodio completo con política actual
2. Calcular returns G_t = Σ_{k=t}^{T} γ^{k-t} r_k
3. Actualizar: θ ← θ + α Σ_t G_t ∇_θ log π_θ(a_t|s_t)

Loss = -Σ_t G_t · log π_θ(a_t|s_t)  [negativo para minimizar]
"""
import numpy as np
import torch
import torch.nn as nn
from seccion_02_entorno import FrozenLakeEnv
from seccion_03_politica_cuantica import AgenteCuantico

def ejecutar_episodio(agente, env, max_pasos=50):
    """Ejecuta un episodio completo y retorna trayectoria."""
    estado = env.reset()
    estados, acciones, log_probs, recompensas = [], [], [], []

    for _ in range(max_pasos):
        a, lp = agente.accion(estado)
        s_next, r, done = env.step(a)

        estados.append(estado)
        acciones.append(a)
        log_probs.append(lp)
        recompensas.append(r)

        estado = s_next
        if done:
            break

    return estados, acciones, log_probs, recompensas

def calcular_returns(recompensas, gamma=0.99):
    """G_t = Σ_{k=t}^{T} γ^{k-t} r_k."""
    T = len(recompensas)
    returns = []
    G = 0
    for r in reversed(recompensas):
        G = r + gamma * G
        returns.insert(0, G)
    # Normalizar (reduce varianza)
    returns = torch.FloatTensor(returns)
    if len(returns) > 1:
        returns = (returns - returns.mean()) / (returns.std() + 1e-8)
    return returns

def entrenar_reinforce(agente, env, n_episodios=300, gamma=0.99, lr=0.01, verbose=True):
    """Entrena el agente QRL con REINFORCE."""
    optimizer = torch.optim.Adam(agente.parameters(), lr=lr)
    historial = {"recompensa": [], "longitud": [], "loss": [], "tasa_exito": []}

    ventana = 20  # para media móvil
    for ep in range(1, n_episodios + 1):
        # 1) Ejecutar episodio
        _, _, log_probs, recompensas = ejecutar_episodio(agente, env)

        # 2) Calcular returns
        returns = calcular_returns(recompensas, gamma)

        # 3) Calcular pérdida policy gradient
        loss = 0
        for lp, G in zip(log_probs, returns):
            loss -= lp * G  # negativo para descenso de gradiente

        # 4) Actualizar
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        # Métricas
        R_total = sum(recompensas)
        historial["recompensa"].append(R_total)
        historial["longitud"].append(len(recompensas))
        historial["loss"].append(loss.item())

        # Tasa de éxito en ventana
        if ep >= ventana:
            tasa = sum(1 for r in historial["recompensa"][-ventana:] if r > 0) / ventana
            historial["tasa_exito"].append(tasa)
        else:
            historial["tasa_exito"].append(0)

        if verbose and ep % 50 == 0:
            media_r = np.mean(historial["recompensa"][-ventana:])
            print(f"Ep {ep:4d}/{n_episodios} | R_media={media_r:.3f} | "
                  f"Tasa={historial['tasa_exito'][-1]:.2f} | "
                  f"Pasos={len(recompensas)}")

    return historial

# Entrenamiento
env = FrozenLakeEnv(determinista=True)
agente = AgenteCuantico(n_capas=3)

print("=== Entrenando Agente QRL (REINFORCE) ===")
hist_qrl = entrenar_reinforce(agente, env, n_episodios=300, gamma=0.99,
                               lr=0.01, verbose=True)

# Evaluación greedy
n_eval = 100
exitos = 0
for _ in range(n_eval):
    s = env.reset()
    for _ in range(50):
        a = agente.accion_greedy(s)
        s, r, done = env.step(a)
        if done:
            if r > 0:
                exitos += 1
            break
print(f"\nEvaluación greedy ({n_eval} episodios): {exitos/n_eval:.2%} de éxito")
```

---

## 5. Agente clásico de referencia

```python
# seccion_05_agente_clasico.py
"""Q-Learning tabular como referencia."""
import numpy as np
from seccion_02_entorno import FrozenLakeEnv

class QLearningTabular:
    """Q-Learning clásico con tabla Q[s, a]."""
    def __init__(self, n_estados=16, n_acciones=4,
                 lr=0.1, gamma=0.99, epsilon=1.0, epsilon_decay=0.995):
        self.Q = np.zeros((n_estados, n_acciones))
        self.lr      = lr
        self.gamma   = gamma
        self.epsilon = epsilon
        self.epsilon_min   = 0.01
        self.epsilon_decay = epsilon_decay

    def accion(self, estado):
        if np.random.rand() < self.epsilon:
            return np.random.randint(4)
        return np.argmax(self.Q[estado])

    def actualizar(self, s, a, r, s_next, done):
        td_target = r + (0 if done else self.gamma * np.max(self.Q[s_next]))
        td_error  = td_target - self.Q[s, a]
        self.Q[s, a] += self.lr * td_error
        if done:
            self.epsilon = max(self.epsilon_min, self.epsilon * self.epsilon_decay)

def entrenar_qlearning(n_episodios=300, max_pasos=50):
    env   = FrozenLakeEnv(determinista=True)
    agente = QLearningTabular()
    historial = {"recompensa": [], "tasa_exito": []}
    ventana = 20

    for ep in range(1, n_episodios + 1):
        s = env.reset()
        R_total = 0

        for _ in range(max_pasos):
            a = agente.accion(s)
            s_next, r, done = env.step(a)
            agente.actualizar(s, a, r, s_next, done)
            R_total += r
            s = s_next
            if done:
                break

        historial["recompensa"].append(R_total)
        tasa = (sum(1 for r in historial["recompensa"][-ventana:] if r > 0)
                / min(ep, ventana))
        historial["tasa_exito"].append(tasa)

        if ep % 50 == 0:
            print(f"Ep {ep:4d} | Tasa={tasa:.2f} | ε={agente.epsilon:.3f}")

    return historial, agente

print("=== Entrenando Q-Learning Tabular ===")
hist_ql, agente_ql = entrenar_qlearning(n_episodios=300)

# Evaluación
env = FrozenLakeEnv(determinista=True)
exitos = 0
for _ in range(100):
    s = env.reset()
    for _ in range(50):
        a = int(np.argmax(agente_ql.Q[s]))
        s, r, done = env.step(a)
        if done:
            if r > 0: exitos += 1
            break
print(f"\nQ-Learning greedy: {exitos}% de éxito")
```

---

## 6. Visualización QRL completa

```python
# seccion_06_visualizacion.py
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_04_reinforce import hist_qrl, agente
from seccion_05_agente_clasico import hist_ql, agente_ql
from seccion_02_entorno import FrozenLakeEnv, estado_a_pos, N_ESTADOS, N_ACCIONES, MAPA
import torch

fig, axes = plt.subplots(2, 3, figsize=(15, 10))
fig.suptitle("Quantum RL vs Q-Learning Clásico — FrozenLake 4x4", fontsize=14)

ventana = 20

def media_movil(arr, w):
    return [np.mean(arr[max(0,i-w+1):i+1]) for i in range(len(arr))]

eps = range(1, 301)

# Panel 1: Tasa de éxito
ax = axes[0, 0]
ax.plot(eps, hist_qrl["tasa_exito"],  "b-", label="QRL (REINFORCE)", linewidth=2)
ax.plot(eps, hist_ql["tasa_exito"],   "r-", label="Q-Learning",      linewidth=2)
ax.set_xlabel("Episodio"); ax.set_ylabel("Tasa de éxito (ventana 20)")
ax.set_title("Tasa de Éxito")
ax.legend(); ax.grid(alpha=0.3); ax.set_ylim([0, 1.05])

# Panel 2: Curva de loss QRL
ax = axes[0, 1]
ax.plot(eps, media_movil(hist_qrl["loss"], 20), "b-", linewidth=2)
ax.set_xlabel("Episodio"); ax.set_ylabel("Policy Loss")
ax.set_title("Pérdida REINFORCE (QRL)"); ax.grid(alpha=0.3)

# Panel 3: Longitud de episodio
ax = axes[0, 2]
ax.plot(eps, media_movil(hist_qrl["longitud"], 20), "b-", label="QRL", linewidth=2)
ax.set_xlabel("Episodio"); ax.set_ylabel("Pasos promedio")
ax.set_title("Longitud de Episodio"); ax.legend(); ax.grid(alpha=0.3)

# Panel 4: Política aprendida QRL (mapa de flechas)
ax = axes[1, 0]
nombres_acc = ["←", "↓", "→", "↑"]
grid = np.zeros((4, 4))
for s in range(N_ESTADOS):
    r, c = estado_a_pos(s)
    tipo = MAPA[r][c]
    a = agente.accion_greedy(s)
    grid[r, c] = a

im = ax.imshow(grid, cmap="viridis", vmin=0, vmax=3)
plt.colorbar(im, ax=ax, ticks=[0,1,2,3], label="Acción")
for s in range(N_ESTADOS):
    r, c = estado_a_pos(s)
    tipo = MAPA[r][c]
    if tipo == "H":
        ax.text(c, r, "✗", ha="center", va="center", fontsize=14, color="red")
    elif tipo == "G":
        ax.text(c, r, "★", ha="center", va="center", fontsize=14, color="gold")
    elif tipo == "S":
        ax.text(c, r, nombres_acc[agente.accion_greedy(s)], ha="center", va="center",
                fontsize=12, color="white", fontweight="bold")
    else:
        ax.text(c, r, nombres_acc[agente.accion_greedy(s)], ha="center", va="center",
                fontsize=12, color="white")
ax.set_title("Política QRL (flechas)"); ax.set_xticks([]); ax.set_yticks([])

# Panel 5: Distribución de acciones por estado (QRL)
ax = axes[1, 1]
estados_ejemplo = [0, 1, 2, 4, 5, 6, 8, 9, 10, 13, 14, 15]
probs_matrix = np.zeros((len(estados_ejemplo), N_ACCIONES))
for i, s in enumerate(estados_ejemplo):
    with torch.no_grad():
        logits = agente.forward(s)
        probs = torch.softmax(logits, dim=0).numpy()
    probs_matrix[i] = probs

im2 = ax.imshow(probs_matrix.T, cmap="Blues", vmin=0, vmax=1)
plt.colorbar(im2, ax=ax, label="Probabilidad")
ax.set_yticks(range(N_ACCIONES))
ax.set_yticklabels(["←", "↓", "→", "↑"])
ax.set_xticks(range(len(estados_ejemplo)))
ax.set_xticklabels([str(s) for s in estados_ejemplo], fontsize=8)
ax.set_xlabel("Estado"); ax.set_ylabel("Acción")
ax.set_title("Distribución de Política QRL")

# Panel 6: Comparación final (bar chart)
ax = axes[1, 2]
def eval_agente(agente_fn, n_eval=100):
    env = FrozenLakeEnv(determinista=True)
    exitos = 0
    for _ in range(n_eval):
        s = env.reset()
        for _ in range(50):
            a = agente_fn(s)
            s, r, done = env.step(a)
            if done:
                if r > 0: exitos += 1
                break
    return exitos / n_eval

acc_qrl = eval_agente(lambda s: agente.accion_greedy(s))
acc_ql  = eval_agente(lambda s: int(np.argmax(agente_ql.Q[s])))

colores = ["steelblue", "coral"]
bars = ax.bar(["QRL\n(cuántico)", "Q-Learning\n(clásico)"],
              [acc_qrl, acc_ql], color=colores, edgecolor="black")
for bar, acc in zip(bars, [acc_qrl, acc_ql]):
    ax.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,
            f"{acc:.2f}", ha="center", va="bottom", fontsize=12, fontweight="bold")
ax.set_ylabel("Tasa de Éxito")
ax.set_title("Comparación Final (100 episodios)")
ax.set_ylim([0, 1.15]); ax.grid(axis="y", alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m28_qrl.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m28_qrl.png")
plt.close()
```

---

## 7. CartPole con política cuántica (extensión)

```python
# seccion_07_cartpole.py
"""
CartPole con estado continuo (4 dimensiones):
- Ángulo encoding: cada feature → Ry(x_i) en un qubit
- 2 acciones: {push-left=0, push-right=1}
- Requiere gymnasium instalado
"""
import numpy as np
import torch
import torch.nn as nn
import pennylane as qml

N_QUBITS_CP = 4
N_ACCIONES_CP = 2
dev_cp = qml.device("default.qubit", wires=N_QUBITS_CP)

@qml.qnode(dev_cp, interface="torch", diff_method="parameter-shift")
def politica_cartpole(estado_norm, theta):
    """
    estado_norm: 4 features normalizadas a [-π, π]
    theta: [n_capas, 4, 3] parámetros
    """
    qml.AngleEmbedding(estado_norm, wires=range(N_QUBITS_CP), rotation="Y")
    qml.StronglyEntanglingLayers(theta, wires=range(N_QUBITS_CP))
    return [qml.expval(qml.PauliZ(0)), qml.expval(qml.PauliZ(1))]

class AgenteCartPole(nn.Module):
    def __init__(self, n_capas=2):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape(n_capas, N_QUBITS_CP)
        self.theta = nn.Parameter(torch.randn(shape) * 0.1)
        # Normalización de estado
        self.obs_bounds = torch.FloatTensor([4.8, 3.5, 0.42, 3.5])

    def normalizar_estado(self, obs):
        """Normaliza observación a [-π, π]."""
        obs_t = torch.FloatTensor(obs)
        return (obs_t / self.obs_bounds) * np.pi

    def forward(self, obs):
        x = self.normalizar_estado(obs)
        logits = torch.stack(politica_cartpole(x, self.theta))
        return logits

    def accion(self, obs):
        logits = self.forward(obs)
        probs = torch.softmax(logits, dim=0)
        dist = torch.distributions.Categorical(probs)
        a = dist.sample()
        return a.item(), dist.log_prob(a)

def entrenar_cartpole(n_episodios=100, gamma=0.99, lr=0.005, render=False):
    """Entrenamiento REINFORCE en CartPole."""
    try:
        import gymnasium as gym
        env = gym.make("CartPole-v1")
    except ImportError:
        print("Gymnasium no disponible. Instalalo con: pip install gymnasium")
        return None, None

    agente = AgenteCartPole(n_capas=2)
    opt = torch.optim.Adam(agente.parameters(), lr=lr)
    historial = {"reward": []}

    for ep in range(1, n_episodios + 1):
        obs, _ = env.reset()
        log_probs, rewards = [], []

        for _ in range(500):
            a, lp = agente.accion(obs)
            obs, r, terminated, truncated, _ = env.step(a)
            log_probs.append(lp)
            rewards.append(r)
            if terminated or truncated:
                break

        # Returns
        G = 0
        returns = []
        for r in reversed(rewards):
            G = r + gamma * G
            returns.insert(0, G)
        returns = torch.FloatTensor(returns)
        if len(returns) > 1:
            returns = (returns - returns.mean()) / (returns.std() + 1e-8)

        # Loss y update
        loss = -sum(lp * G for lp, G in zip(log_probs, returns))
        opt.zero_grad(); loss.backward(); opt.step()

        historial["reward"].append(sum(rewards))
        if ep % 20 == 0:
            media = np.mean(historial["reward"][-20:])
            print(f"Ep {ep:3d} | Reward media: {media:.1f}")

    env.close()
    return agente, historial

print("=== CartPole Cuántico ===")
agente_cp, hist_cp = entrenar_cartpole(n_episodios=100, lr=0.005)
if hist_cp:
    print(f"Reward final (últimos 20 eps): {np.mean(hist_cp['reward'][-20:]):.1f}")
    print("Nota: CartPole-v1 se considera resuelto con reward media >= 475")
```

---

## Checkpoint

```python
# checkpoint_m28.py
"""Tests para Módulo 28 — Quantum RL."""
import numpy as np
import torch
import torch.nn as nn
import pennylane as qml

N_QUBITS = 4
N_ACCIONES = 4
dev = qml.device("default.qubit", wires=N_QUBITS)

def basis_encode_test(estado, n_q):
    bits = format(estado, f"0{n_q}b")
    for i, b in enumerate(bits):
        if b == "1":
            qml.PauliX(wires=i)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def politica_test(estado, theta):
    basis_encode_test(estado, N_QUBITS)
    qml.StronglyEntanglingLayers(theta, wires=range(N_QUBITS))
    return [qml.expval(qml.PauliZ(i)) for i in range(N_ACCIONES)]

class AgenteTest(nn.Module):
    def __init__(self):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape(2, N_QUBITS)
        self.theta = nn.Parameter(torch.randn(shape) * 0.1)

    def forward(self, estado):
        return torch.stack(politica_test(estado, self.theta))

    def accion(self, estado):
        logits = self.forward(estado)
        probs = torch.softmax(logits, dim=0)
        dist = torch.distributions.Categorical(probs)
        a = dist.sample()
        return a.item(), dist.log_prob(a)

# T1: Política produce 4 logits para estado dado
def test_logits_shape():
    agente = AgenteTest()
    logits = agente.forward(5)
    assert logits.shape == (4,), f"Shape logits: {logits.shape}"
    print("T1 PASS: política produce 4 logits")

# T2: Softmax de logits suma a 1 (distribución de probabilidades)
def test_distribucion_valida():
    agente = AgenteTest()
    logits = agente.forward(7)
    probs = torch.softmax(logits, dim=0)
    assert abs(probs.sum().item() - 1.0) < 1e-5
    assert all(p > 0 for p in probs)
    print("T2 PASS: distribución de política suma a 1")

# T3: Gradient de policy loss llega a theta
def test_policy_gradient():
    agente = AgenteTest()
    _, lp = agente.accion(3)
    loss = -lp * 1.0  # G_t = 1 ficticio
    loss.backward()
    assert agente.theta.grad is not None
    assert agente.theta.grad.norm().item() > 0
    print("T3 PASS: gradiente REINFORCE llega a parámetros cuánticos")

# T4: Basis encoding es determinista
def test_basis_encode_determinista():
    dev2 = qml.device("default.qubit", wires=N_QUBITS)

    @qml.qnode(dev2)
    def estado_s():
        basis_encode_test(5, N_QUBITS)  # 5 = 0101
        return qml.probs(wires=range(N_QUBITS))

    p1 = estado_s()
    p2 = estado_s()
    assert np.allclose(p1, p2)
    # P(|0101⟩) debe ser ~1
    assert p1[5] > 0.99, f"P(|5⟩) = {p1[5]:.4f} (debe ser ~1)"
    print(f"T4 PASS: P(|0101⟩=5) = {p1[5]:.4f}")

# T5: Entorno FrozenLake alcanza meta con ruta correcta
def test_frozen_lake_ruta_optima():
    from seccion_02_entorno import FrozenLakeEnv
    env = FrozenLakeEnv(determinista=True)
    s = env.reset()
    # Ruta: abajo, der, der, abajo, abajo, der
    ruta = [1, 2, 2, 1, 1, 2]
    r_total = 0
    for a in ruta:
        s, r, done = env.step(a)
        r_total += r
        if done: break
    assert r_total == 1.0, f"Recompensa ruta óptima: {r_total}"
    print("T5 PASS: ruta óptima FrozenLake obtiene recompensa 1.0")

# T6: Returns G_t bien calculados
def test_returns():
    rewards = [0, 0, 0, 1.0]
    gamma = 0.99
    G = 0
    returns = []
    for r in reversed(rewards):
        G = r + gamma * G
        returns.insert(0, G)
    # G_0 = γ³ = 0.99³ ≈ 0.9703
    assert abs(returns[0] - 0.99**3) < 1e-4, f"G_0={returns[0]:.6f}"
    assert abs(returns[-1] - 1.0) < 1e-6
    print(f"T6 PASS: returns correctos G_0={returns[0]:.4f} ≈ γ³={0.99**3:.4f}")

if __name__ == "__main__":
    print("=== Checkpoint M28: Quantum RL ===\n")
    test_logits_shape()
    test_distribucion_valida()
    test_policy_gradient()
    test_basis_encode_determinista()
    test_frozen_lake_ruta_optima()
    test_returns()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M29 — Química Cuántica y Farmacología](./modulo-29-quimica.md)
