# Tutorial 2 — Computación Cuántica y Quantum ML
## Módulo 6 — PennyLane: Diferenciación Automática Cuántica

> **Objetivo:** Dominar PennyLane como framework de QML: QNodes, gradientes cuánticos, integración con PyTorch/NumPy, y construcción de circuitos variacionales diferenciables. Este módulo establece las bases para los módulos de QML (M21-M28).
>
> **Herramientas:** PennyLane 0.38+ + NumPy + PyTorch.
>
> **Prerequisito:** M05 — Qiskit en Profundidad.

---

## Cómo ejecutar este módulo

### Opción A — Local (.py)
```bash
conda activate quantum
python modulo-06-pennylane.py
```

### Opción B — Jupyter
```bash
conda activate quantum
jupyter lab modulo-06-pennylane.ipynb
```

### Opción C — Google Colab
```python
!pip install pennylane pennylane-qiskit torch -q
```

---

## 6.1 QNodes — La Unidad Fundamental de PennyLane

```python
import pennylane as qml
import numpy as np
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

# Un QNode une: dispositivo cuántico + función de circuito
# La función retorna una medición observable

dev = qml.device("default.qubit", wires=2)

@qml.qnode(dev)
def circuito_basico(theta, phi):
    """
    Circuito de dos qubits con rotaciones paramétricas.
    Retorna el valor esperado de Z⊗Z.
    """
    qml.RY(theta, wires=0)
    qml.RY(phi, wires=1)
    qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

# Llamar como función Python normal
resultado = circuito_basico(np.pi/4, np.pi/3)
print(f"⟨Z⊗Z⟩(θ=45°, φ=60°) = {resultado:.4f}")

# Barrer θ manteniendo φ fijo
thetas = np.linspace(0, 2*np.pi, 60)
valores = [circuito_basico(t, np.pi/4) for t in thetas]

plt.figure(figsize=(8, 3))
plt.plot(thetas * 180 / np.pi, valores, color='steelblue', linewidth=2)
plt.xlabel("θ (grados)"); plt.ylabel("⟨Z⊗Z⟩")
plt.title("Valor esperado vs ángulo de rotación")
plt.axhline(0, color='gray', linestyle='--', linewidth=0.5)
plt.tight_layout()
plt.savefig("outputs/qnode_sweep.png", dpi=150, bbox_inches='tight')
plt.show()
```

### Tipos de mediciones en PennyLane

```python
dev_meas = qml.device("default.qubit", wires=3)

# ── expval — valor esperado de un observable ─────────────
@qml.qnode(dev_meas)
def medir_expval():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0))

# ── var — varianza de un observable ──────────────────────
@qml.qnode(dev_meas)
def medir_var():
    qml.Hadamard(0)
    return qml.var(qml.PauliZ(0))   # Var(Z) = 1 para |+⟩

# ── probs — distribución de probabilidades ────────────────
@qml.qnode(dev_meas)
def medir_probs():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.probs(wires=[0, 1])  # [P(00), P(01), P(10), P(11)]

# ── state — statevector completo ─────────────────────────
@qml.qnode(dev_meas)
def medir_estado():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.state()

# ── sample — muestras individuales ───────────────────────
@qml.qnode(qml.device("default.qubit", wires=2, shots=100))
def medir_samples():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.sample(wires=[0, 1])

print("Tipos de mediciones en PennyLane:")
print(f"  expval(Z⊗I):    {medir_expval():.4f}  (Bell → ⟨Z⟩=0)")
print(f"  var(Z) en |+⟩:  {medir_var():.4f}  (debe ser 1)")
print(f"  probs:          {np.round(medir_probs(), 3)}")
print(f"  state:          {np.round(medir_estado(), 3)}")
muestras = medir_samples()
print(f"  samples (10):   {muestras[:10]}")
```

---

## 6.2 Gradientes Cuánticos — Parameter Shift Rule

### El corazón del QML: ∂⟨O⟩/∂θ

```python
# En circuitos cuánticos no se puede aplicar la regla de la cadena directamente.
# La Parameter Shift Rule dice:
#   ∂⟨O⟩/∂θ = [⟨O⟩(θ+π/2) - ⟨O⟩(θ-π/2)] / 2
# Esto es EXACTO, no una aproximación de diferencia finita.

dev_grad = qml.device("default.qubit", wires=1)

@qml.qnode(dev_grad)
def circuito_1d(theta):
    qml.RY(theta, wires=0)
    return qml.expval(qml.PauliZ(0))

theta_0 = np.pi / 4

# Gradiente manual con Parameter Shift Rule
shift = np.pi / 2
grad_manual = (circuito_1d(theta_0 + shift) - circuito_1d(theta_0 - shift)) / 2
print(f"Gradiente manual (PSR): ∂⟨Z⟩/∂θ|_{{θ=π/4}} = {grad_manual:.6f}")

# Gradiente automático de PennyLane — usa PSR internamente
grad_auto = qml.grad(circuito_1d)(theta_0)
print(f"Gradiente automático:               = {grad_auto:.6f}")

# Diferencia finita (aproximado, para comparar)
eps = 1e-4
grad_fd = (circuito_1d(theta_0 + eps) - circuito_1d(theta_0 - eps)) / (2 * eps)
print(f"Diferencia finita (approx):         = {grad_fd:.6f}")

# Valor analítico: d/dθ cos(θ) = -sin(θ)
grad_analitico = -np.sin(theta_0)
print(f"Valor analítico -sin(π/4):          = {grad_analitico:.6f}")

# Verificar curva del gradiente
thetas = np.linspace(0, 2*np.pi, 80)
vals   = [circuito_1d(t) for t in thetas]
grads  = [qml.grad(circuito_1d)(t) for t in thetas]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
ax1.plot(np.degrees(thetas), vals, label="⟨Z⟩(θ) = cos(θ)", color='steelblue')
ax1.set_xlabel("θ (°)"); ax1.set_ylabel("⟨Z⟩")
ax1.legend(); ax1.grid(alpha=0.3)

ax2.plot(np.degrees(thetas), grads, label="d⟨Z⟩/dθ = -sin(θ)", color='coral')
ax2.set_xlabel("θ (°)"); ax2.set_ylabel("Gradiente")
ax2.legend(); ax2.grid(alpha=0.3)

plt.suptitle("Función y gradiente de Ry(θ)")
plt.tight_layout()
plt.savefig("outputs/gradiente_qnode.png", dpi=150, bbox_inches='tight')
plt.show()
```

### Gradientes de múltiples parámetros

```python
dev_multi = qml.device("default.qubit", wires=2)

@qml.qnode(dev_multi)
def circuito_multi(params):
    """params = [θ0, θ1, φ] — circuito con 3 parámetros."""
    qml.RY(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    qml.RZ(params[2], wires=0)
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

params_0 = np.array([np.pi/4, np.pi/3, np.pi/6])

# Gradiente completo ∇_params ⟨O⟩
grad_vector = qml.grad(circuito_multi)(params_0)
print(f"\nGradiente del circuito multi-param:")
print(f"  ∂/∂θ0 = {grad_vector[0]:.6f}")
print(f"  ∂/∂θ1 = {grad_vector[1]:.6f}")
print(f"  ∂/∂φ  = {grad_vector[2]:.6f}")

# Jacobiano (para múltiples observables)
@qml.qnode(dev_multi)
def circuito_multi_out(params):
    qml.RY(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0)), qml.expval(qml.PauliZ(1))

jac = qml.jacobian(circuito_multi_out)(params_0[:2])
print(f"\nJacobiano (2 obs × 2 params):\n{np.round(jac, 4)}")
```

---

## 6.3 Integración con PyTorch

```python
import torch

# PennyLane puede usar torch como interfaz de diferenciación
dev_torch = qml.device("default.qubit", wires=2)

@qml.qnode(dev_torch, interface="torch", diff_method="parameter-shift")
def qnode_torch(params):
    qml.RY(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

# Parámetros como tensores de PyTorch con gradientes
params_torch = torch.tensor([np.pi/4, np.pi/3], requires_grad=True, dtype=torch.float64)

# Forward pass
resultado_torch = qnode_torch(params_torch)
print(f"\nQNode con PyTorch: ⟨ZZ⟩ = {resultado_torch.item():.4f}")

# Backward pass — autograd de PyTorch
resultado_torch.backward()
print(f"  d⟨ZZ⟩/dθ0 = {params_torch.grad[0].item():.6f}")
print(f"  d⟨ZZ⟩/dθ1 = {params_torch.grad[1].item():.6f}")

# Optimización simple con Adam de PyTorch
params_opt = torch.tensor([0.5, 0.5], requires_grad=True, dtype=torch.float64)
optimizer_torch = torch.optim.Adam([params_opt], lr=0.1)

historial_torch = []
for paso in range(100):
    optimizer_torch.zero_grad()
    loss = qnode_torch(params_opt)   # minimizamos ⟨ZZ⟩
    loss.backward()
    optimizer_torch.step()
    historial_torch.append(loss.item())

print(f"\nOptimización PyTorch (100 pasos):")
print(f"  Valor inicial: {historial_torch[0]:.4f}")
print(f"  Valor final:   {historial_torch[-1]:.4f}")
print(f"  Params finales: θ0={params_opt[0].item():.3f}, θ1={params_opt[1].item():.3f}")
```

---

## 6.4 Métodos de Diferenciación Comparados

```python
# PennyLane soporta múltiples métodos de diferenciación:
# - "parameter-shift": exacto, 2 evals por parámetro
# - "backprop":        rápido, solo en simuladores clásicos
# - "finite-diff":     aproximado, 1 eval por parámetro
# - "adjoint":        eficiente en memoria para muchos qubits

n_qubits = 4
n_params  = 2 * n_qubits   # 2 capas de rotaciones

dev_bench = qml.device("default.qubit", wires=n_qubits)
params_bench = np.random.uniform(0, 2*np.pi, n_params)

def crear_qnode(metodo: str):
    @qml.qnode(dev_bench, diff_method=metodo)
    def circ(params):
        for i in range(n_qubits):
            qml.RY(params[i], wires=i)
        for i in range(n_qubits - 1):
            qml.CNOT(wires=[i, i+1])
        for i in range(n_qubits):
            qml.RY(params[n_qubits + i], wires=i)
        return qml.expval(qml.PauliZ(0))
    return circ

import time

metodos = ["parameter-shift", "backprop", "finite-diff"]
print(f"Benchmark gradientes ({n_qubits} qubits, {n_params} params):")
for metodo in metodos:
    try:
        circ = crear_qnode(metodo)
        # Compilar (primer llamado tiene overhead)
        _ = qml.grad(circ)(params_bench)
        # Medir
        t0 = time.perf_counter()
        for _ in range(20):
            grad = qml.grad(circ)(params_bench)
        t1 = time.perf_counter()
        print(f"  {metodo:20s}: {(t1-t0)/20*1000:.2f} ms/eval")
    except Exception as e:
        print(f"  {metodo:20s}: no disponible ({e})")
```

---

## 6.5 Templates — Circuitos Prefabricados de PennyLane

```python
# PennyLane incluye templates: bloques de circuito reutilizables

dev_tmpl = qml.device("default.qubit", wires=4)

# ── StronglyEntanglingLayers ─────────────────────────────
# Capas densamente entrelazadas — ansatz para QML
@qml.qnode(dev_tmpl)
def strongly_entangling(weights):
    qml.StronglyEntanglingLayers(weights, wires=range(4))
    return qml.state()

shape_se = qml.StronglyEntanglingLayers.shape(n_layers=2, n_wires=4)
weights_se = np.random.uniform(0, 2*np.pi, shape_se)
sv_se = strongly_entangling(weights_se)
print(f"StronglyEntanglingLayers shape: {shape_se}")
print(f"  Dimensión del statevector: {len(sv_se)}")

# ── BasicEntanglerLayers ──────────────────────────────────
@qml.qnode(dev_tmpl)
def basic_entangler(weights):
    qml.BasicEntanglerLayers(weights, wires=range(4))
    return qml.state()

shape_be = qml.BasicEntanglerLayers.shape(n_layers=3, n_wires=4)
weights_be = np.random.uniform(0, 2*np.pi, shape_be)
sv_be = basic_entangler(weights_be)
print(f"\nBasicEntanglerLayers shape: {shape_be}")

# ── AngleEmbedding — codificar datos clásicos en ángulos ─
@qml.qnode(dev_tmpl)
def angle_embedding(x):
    qml.AngleEmbedding(x, wires=range(4), rotation='Y')
    return qml.state()

x_data = np.array([0.3, 0.7, 1.2, 0.9])
sv_emb = angle_embedding(x_data)
print(f"\nAngleEmbedding con {x_data}: sv[0:4]={np.round(np.abs(sv_emb[:4])**2, 3)}")

# ── AmplitudeEmbedding — codificar en amplitudes ─────────
@qml.qnode(dev_tmpl)
def amplitude_embedding(x):
    qml.AmplitudeEmbedding(x, wires=range(4), normalize=True)
    return qml.state()

x_amp = np.array([1.0, 2.0, 0.5, 1.5, 0.3, 0.8, 1.1, 0.4,
                   0.9, 0.2, 1.3, 0.6, 0.7, 0.1, 1.4, 0.5])
sv_amp = amplitude_embedding(x_amp)
print(f"\nAmplitudeEmbedding: norma = {np.sum(np.abs(sv_amp)**2):.6f}")

# ── QAOAAnsatz ────────────────────────────────────────────
# (se verá en detalle en M19)
print("\nTemplates disponibles en qml.templates:")
for nombre in dir(qml):
    obj = getattr(qml, nombre)
    if hasattr(obj, '__module__') and 'templates' in str(getattr(obj, '__module__', '')):
        try:
            if callable(obj) and hasattr(obj, 'shape'):
                print(f"  qml.{nombre}")
        except: pass
```

---

## 6.6 Optimización Variacional con PennyLane

### Minimizar un hamiltoniano simple

```python
# Problema: encontrar el estado que minimiza ⟨H⟩
# H = 0.5·Z₀ - 0.3·Z₁ + 0.4·(X₀⊗X₁) + 0.2·(Y₀⊗Y₁)

dev_opt = qml.device("default.qubit", wires=2)

H_simple = (
    0.5 * qml.PauliZ(0)
    - 0.3 * qml.PauliZ(1)
    + 0.4 * (qml.PauliX(0) @ qml.PauliX(1))
    + 0.2 * (qml.PauliY(0) @ qml.PauliY(1))
)

@qml.qnode(dev_opt)
def ansatz_2q(params):
    """Ansatz de 2 capas para 2 qubits."""
    qml.RY(params[0], wires=0); qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    qml.RY(params[2], wires=0); qml.RY(params[3], wires=1)
    qml.CNOT(wires=[1, 0])
    return qml.expval(H_simple)

# Comparar optimizadores de PennyLane
def optimizar(optimizer, n_pasos=150, seed=42):
    np.random.seed(seed)
    params = np.random.uniform(0, np.pi, 4)
    historia = []
    for _ in range(n_pasos):
        params, energia = optimizer.step_and_cost(ansatz_2q, params)
        historia.append(energia)
    return historia

# Gradiente descendiente básico
opt_gd   = qml.GradientDescentOptimizer(stepsize=0.1)
# Adam — adaptativo
opt_adam = qml.AdamOptimizer(stepsize=0.05)
# Riemannian Gradient Descent — respeta geometría cuántica
opt_qng  = qml.QNGOptimizer(stepsize=0.1)

print("Optimización de ⟨H⟩ (150 pasos):")
historiales = {}
for nombre, opt in [("GD", opt_gd), ("Adam", opt_adam), ("QNG", opt_qng)]:
    try:
        hist = optimizar(opt)
        historiales[nombre] = hist
        print(f"  {nombre:6s}: inicio={hist[0]:.4f}, final={hist[-1]:.4f}")
    except Exception as e:
        print(f"  {nombre:6s}: error — {e}")

# Visualizar convergencia
fig, ax = plt.subplots(figsize=(8, 4))
for nombre, hist in historiales.items():
    ax.plot(hist, label=nombre)
ax.set_xlabel("Paso de optimización")
ax.set_ylabel("⟨H⟩")
ax.set_title("Convergencia de optimizadores cuánticos")
ax.legend()
ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig("outputs/convergencia_optimizadores.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 6.7 Dibujar y Depurar Circuitos en PennyLane

```python
dev_draw = qml.device("default.qubit", wires=3)

@qml.qnode(dev_draw)
def circuito_dibujable(params):
    qml.Hadamard(0)
    qml.StronglyEntanglingLayers(
        params.reshape(qml.StronglyEntanglingLayers.shape(2, 3)),
        wires=range(3)
    )
    return qml.expval(qml.PauliZ(0))

params_d = np.random.uniform(0, 2*np.pi, 2*3*3)

# draw() — representación en texto
print("── Circuito (texto) ──")
print(qml.draw(circuito_dibujable)(params_d))

# draw_mpl() — figura matplotlib
print("\n── Circuito (matplotlib) ──")
fig_circ, ax_circ = qml.draw_mpl(circuito_dibujable)(params_d)
fig_circ.savefig("outputs/pennylane_circuito.png", dpi=150, bbox_inches='tight')
plt.close(fig_circ)

# Inspeccionar las operaciones del circuito
print("\n── Operaciones del circuito ──")
tape = qml.tape.QuantumScript.from_queue(
    qml.queuing.AnnotatedQueue()
)
# Usar specs() para obtener metadatos
specs = qml.specs(circuito_dibujable)(params_d)
print(f"  Profundidad: {specs['resources'].depth}")
print(f"  Puertas:     {dict(specs['resources'].gate_types)}")
print(f"  Parámetros:  {specs['num_trainable_params']}")
```

---

## 6.8 Circuito Híbrido Clásico-Cuántico

```python
import torch
import torch.nn as nn

# Modelo híbrido: capa clásica → circuito cuántico → capa clásica
# Patrón fundamental para QML

n_qubits = 4
dev_hibrido = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev_hibrido, interface="torch", diff_method="parameter-shift")
def capa_cuantica(inputs, weights):
    """
    Inputs:  n_qubits valores clásicos (embeddings)
    Weights: parámetros variacionales entrenables
    """
    # Codificar inputs clásicos como ángulos
    qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation='Y')
    # Capa variacional
    qml.StronglyEntanglingLayers(
        weights.reshape(qml.StronglyEntanglingLayers.shape(1, n_qubits)),
        wires=range(n_qubits)
    )
    # Mediciones de cada qubit
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class ModeloHibrido(nn.Module):
    def __init__(self, n_features_in: int, n_clases: int):
        super().__init__()
        # Capa clásica de entrada: reducir a n_qubits features
        self.pre_net  = nn.Linear(n_features_in, n_qubits)
        self.activacion = nn.Tanh()   # mapear a [-1, 1] ≈ ángulos útiles

        # Parámetros cuánticos entrenables
        shape_w = qml.StronglyEntanglingLayers.shape(1, n_qubits)
        self.q_weights = nn.Parameter(
            torch.tensor(np.random.uniform(0, np.pi, shape_w), dtype=torch.float32)
        )
        # Capa clásica de salida
        self.post_net = nn.Linear(n_qubits, n_clases)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # 1. Preprocesamiento clásico
        x = self.activacion(self.pre_net(x)) * np.pi  # escalar a ángulos
        # 2. Capa cuántica (batch)
        salidas_q = torch.stack([
            torch.stack(capa_cuantica(x[i], self.q_weights))
            for i in range(x.shape[0])
        ])
        # 3. Postprocesamiento clásico
        return self.post_net(salidas_q)

# Prueba rápida con datos sintéticos
modelo = ModeloHibrido(n_features_in=8, n_clases=3)
X_dummy = torch.randn(5, 8, dtype=torch.float32)  # 5 muestras, 8 features
logits  = modelo(X_dummy)
print(f"\nModelo híbrido:")
print(f"  Input shape:  {X_dummy.shape}")
print(f"  Output shape: {logits.shape}")
print(f"  Parámetros totales:")
for nombre, param in modelo.named_parameters():
    print(f"    {nombre}: {param.shape} ({param.numel()} params)")
total_params = sum(p.numel() for p in modelo.parameters())
print(f"  TOTAL: {total_params} parámetros entrenables")
```

---

## 6.9 Cuantum Natural Gradient (QNG)

```python
# QNG adapta el gradiente a la métrica de Fubini-Study del espacio de Hilbert
# Es el análogo cuántico del Natural Gradient clásico
# En la práctica, converge mucho más rápido que GD estándar

dev_qng = qml.device("default.qubit", wires=2)

@qml.qnode(dev_qng)
def vqe_simple(params):
    qml.RY(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    qml.RY(params[2], wires=0)
    qml.RY(params[3], wires=1)
    return qml.expval(
        0.5 * qml.PauliZ(0) - 0.3 * qml.PauliZ(1) + 0.4 * (qml.PauliX(0) @ qml.PauliX(1))
    )

# Comparación: GD vs QNG en mismas condiciones
np.random.seed(0)
p0 = np.random.uniform(0, np.pi, 4)

def run_opt(optimizer, n_steps=100):
    params = p0.copy()
    hist = []
    for _ in range(n_steps):
        params, val = optimizer.step_and_cost(vqe_simple, params)
        hist.append(val)
    return hist, params

hist_gd, p_gd   = run_opt(qml.GradientDescentOptimizer(0.2))
hist_adam, p_adam = run_opt(qml.AdamOptimizer(0.05))
try:
    hist_qng, p_qng  = run_opt(qml.QNGOptimizer(0.1))
    qng_disponible = True
except Exception:
    hist_qng = []
    qng_disponible = False

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(hist_gd,   label="GD (lr=0.2)",   linewidth=2)
ax.plot(hist_adam, label="Adam (lr=0.05)", linewidth=2)
if qng_disponible:
    ax.plot(hist_qng, label="QNG (lr=0.1)", linewidth=2, linestyle='--')
ax.set_xlabel("Paso"); ax.set_ylabel("⟨H⟩")
ax.set_title("Convergencia: GD vs Adam vs QNG")
ax.legend(); ax.grid(alpha=0.3)
plt.tight_layout()
plt.savefig("outputs/qng_convergencia.png", dpi=150, bbox_inches='tight')
plt.show()

print(f"\nOptimización VQE:")
print(f"  GD final:   {hist_gd[-1]:.4f}")
print(f"  Adam final: {hist_adam[-1]:.4f}")
if qng_disponible:
    print(f"  QNG final:  {hist_qng[-1]:.4f}")
```

---

## Resumen — PennyLane vs Qiskit

| Aspecto | PennyLane | Qiskit |
|---|---|---|
| Paradigma central | QNode (función diferenciable) | QuantumCircuit (objeto) |
| Gradientes | Parameter Shift, backprop, adjoint | Limitado (Estimator) |
| ML integración | PyTorch, JAX, TF nativo | qiskit-machine-learning |
| Mediciones | expval, var, probs, state, sample | Counts, StatevectorResult |
| Optimizadores | GD, Adam, QNG, SPSA, Rotosolve | No incluye |
| Templates | StronglyEntangling, BasicEntangler, Embeddings | Circuit Library |
| Hardware | IBM (via plugin), simuladores, fotónico | IBM Quantum nativo |
| Ruido | Básico (canales) | Completo (NoiseModel) |
| Mejor para | QML, VQC, investigación de gradientes | Algoritmos, hardware IBM |

---

## checkpoint_m06.py

```python
"""
Checkpoint Módulo 6 — PennyLane: Diferenciación Automática Cuántica
Ejecutar con: python checkpoint_m06.py
"""
import numpy as np
import pennylane as qml
import os

os.makedirs("outputs", exist_ok=True)

print("=" * 55)
print("Checkpoint M06 — PennyLane")
print("=" * 55)

dev = qml.device("default.qubit", wires=2)

# ── Test 1: QNode básico ─────────────────────────────────
@qml.qnode(dev)
def bell_state():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.state()

sv = np.array(bell_state())
assert abs(sv[0])**2 > 0.49 and abs(sv[3])**2 > 0.49, "FALLO: Bell state"
print("✓ Test 1: QNode básico — Bell state")

# ── Test 2: expval de Bell ────────────────────────────────
@qml.qnode(dev)
def expval_ZZ():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

E_ZZ = expval_ZZ()
assert abs(E_ZZ - 1.0) < 0.01, f"FALLO: ⟨ZZ⟩={E_ZZ:.4f} debe ser 1"
print(f"✓ Test 2: ⟨ZZ⟩ Bell state = {E_ZZ:.4f} ≈ 1.0")

# ── Test 3: Gradiente PSR exacto ─────────────────────────
dev1 = qml.device("default.qubit", wires=1)

@qml.qnode(dev1)
def ry_expval(theta):
    qml.RY(theta, wires=0)
    return qml.expval(qml.PauliZ(0))

theta_test = np.pi / 4
grad_pl = qml.grad(ry_expval)(theta_test)
grad_analitico = -np.sin(theta_test)
assert abs(grad_pl - grad_analitico) < 1e-6, \
    f"FALLO: gradiente PSR={grad_pl:.6f}, analítico={grad_analitico:.6f}"
print(f"✓ Test 3: PSR gradiente = {grad_pl:.6f} ≈ analítico {grad_analitico:.6f}")

# ── Test 4: Gradiente multi-parámetro ────────────────────
@qml.qnode(dev)
def circuito_2params(params):
    qml.RY(params[0], wires=0); qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    return qml.expval(qml.PauliZ(0))

params_t = np.array([np.pi/3, np.pi/5])
grad_v = qml.grad(circuito_2params)(params_t)
assert len(grad_v) == 2, "FALLO: debe retornar 2 gradientes"
print(f"✓ Test 4: Gradiente multi-param: {np.round(grad_v, 4)}")

# ── Test 5: probs suman 1 ────────────────────────────────
@qml.qnode(dev)
def probs_ghz():
    qml.Hadamard(0); qml.CNOT(wires=[0, 1])
    return qml.probs(wires=[0, 1])

prbs = probs_ghz()
assert abs(sum(prbs) - 1.0) < 1e-6, "FALLO: probs no suman 1"
assert abs(prbs[0] - 0.5) < 0.01 and abs(prbs[3] - 0.5) < 0.01, \
    f"FALLO: probs Bell = {prbs}"
print(f"✓ Test 5: probs Bell = {np.round(prbs, 3)}")

# ── Test 6: StronglyEntanglingLayers ─────────────────────
dev4 = qml.device("default.qubit", wires=4)
shape = qml.StronglyEntanglingLayers.shape(n_layers=2, n_wires=4)

@qml.qnode(dev4)
def strongly_entangling(w):
    qml.StronglyEntanglingLayers(w, wires=range(4))
    return qml.state()

weights = np.random.uniform(0, 2*np.pi, shape)
sv4 = strongly_entangling(weights)
norma = np.sum(np.abs(sv4)**2)
assert abs(norma - 1.0) < 1e-6, f"FALLO: norma={norma}"
print(f"✓ Test 6: StronglyEntanglingLayers shape={shape}, norma={norma:.6f}")

# ── Test 7: Optimización — Adam reduce ⟨H⟩ ──────────────
H_test = qml.PauliZ(0) @ qml.PauliZ(1) - 0.5 * qml.PauliX(0)

@qml.qnode(dev)
def ansatz(params):
    qml.RY(params[0], wires=0); qml.RY(params[1], wires=1)
    qml.CNOT(wires=[0, 1])
    return qml.expval(H_test)

np.random.seed(42)
params = np.random.uniform(0, np.pi, 2)
opt = qml.AdamOptimizer(stepsize=0.1)
E_inicial = ansatz(params)
for _ in range(50):
    params, E = opt.step_and_cost(ansatz, params)
E_final = ansatz(params)
assert E_final < E_inicial, \
    f"FALLO: Adam no reduce ⟨H⟩: inicial={E_inicial:.4f}, final={E_final:.4f}"
print(f"✓ Test 7: Adam optimiza ⟨H⟩: {E_inicial:.4f} → {E_final:.4f}")

# ── Test 8: AngleEmbedding ────────────────────────────────
dev4_2 = qml.device("default.qubit", wires=4)

@qml.qnode(dev4_2)
def embed(x):
    qml.AngleEmbedding(x, wires=range(4), rotation='Y')
    return qml.state()

x = np.array([0.1, 0.2, 0.3, 0.4])
sv_emb = embed(x)
norma_e = np.sum(np.abs(sv_emb)**2)
assert abs(norma_e - 1.0) < 1e-6, f"FALLO: AngleEmbedding norma={norma_e}"
print("✓ Test 8: AngleEmbedding normalizado")

# ── Test 9: PyTorch interface ─────────────────────────────
try:
    import torch
    dev_t = qml.device("default.qubit", wires=2)

    @qml.qnode(dev_t, interface="torch")
    def qnode_t(params):
        qml.RY(params[0], wires=0); qml.RY(params[1], wires=1)
        qml.CNOT(wires=[0, 1])
        return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

    p = torch.tensor([0.5, 0.7], requires_grad=True, dtype=torch.float64)
    out = qnode_t(p)
    out.backward()
    assert p.grad is not None, "FALLO: no se calculó el gradiente PyTorch"
    print(f"✓ Test 9: PyTorch interface, gradiente={p.grad.tolist()}")
except ImportError:
    print("  (!) PyTorch no disponible")

print("\n" + "=" * 55)
print("✅ Todos los tests del M06 pasaron correctamente")
print("=" * 55)
```
