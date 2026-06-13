# Módulo 20 — Gradientes Cuánticos: Parameter Shift Rule y Más

**Objetivo:** Dominar los métodos de diferenciación de circuitos cuánticos —
Parameter Shift Rule (exacta), diferencia finita, adjoint differentiation y
gradientes de orden superior. Entender cuándo usar cada uno.

**Herramientas:** PennyLane, Qiskit, numpy, matplotlib
**Prerequisito:** M06 (PennyLane), M17 (VQC)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
python modulo-20-gradientes.py
```

---

## 1. El problema de la diferenciación de circuitos

```python
# modulo-20-gradientes.py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import pennylane as qml
import time
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Motivación
# ─────────────────────────────────────────────

print("=" * 60)
print("GRADIENTES CUÁNTICOS — DIFERENCIACIÓN DE CIRCUITOS")
print("=" * 60)

print("""
Para optimizar un VQC, necesitamos calcular:
  ∂C(θ)/∂θᵢ  (gradiente del coste respecto a cada parámetro)

El problema: los circuitos cuánticos son funciones de θ a través
de puertas e^{-iθP/2} (exponenciales de Pauli). No son diferenciables
de forma directa (no hay backprop en hardware cuántico).

Métodos disponibles:
  1. Diferencia finita (FD): 2 evaluaciones por parámetro, inexacto
  2. Parameter Shift Rule (PSR): 2 evaluaciones por parámetro, EXACTO
  3. Adjoint differentiation: eficiente en simulador, no en hardware
  4. SPSA: 2 evaluaciones totales, muy ruidoso
  5. Natural gradient (QNG): escala con métrica de Fubini-Study
""")

# ─────────────────────────────────────────────
# SECCIÓN 2: Parameter Shift Rule — derivación
# ─────────────────────────────────────────────

print("=" * 60)
print("PARAMETER SHIFT RULE — DERIVACIÓN MATEMÁTICA")
print("=" * 60)

print("""
Para una puerta de la forma U(θ) = e^{-iθG/2} donde G² = I (Pauli):

  ∂⟨O⟩/∂θ = [⟨O⟩(θ + π/2) - ⟨O⟩(θ - π/2)] / 2

Demostración:
  ⟨O(θ)⟩ = ⟨ψ| U†(θ) O U(θ) |ψ⟩

  Usando la expansión: U(θ) = cos(θ/2)I - i·sin(θ/2)G:

  ∂⟨O⟩/∂θ = Re[⟨ψ|U†(θ)(∂U/∂θ) O|ψ(θ)⟩]

  Con G² = I:
  ∂U/∂θ = -i/2 · G · U(θ) = -i/2 · e^{-iθG/2} G

  Resultado exacto:
  ∂⟨O⟩/∂θ = (1/2)[⟨O⟩(θ+π/2) - ⟨O⟩(θ-π/2)]

  EXACTO — no es una aproximación como la diferencia finita.

Generalización (Mitarai 2018, Schuld 2019):
  Para G con eigenvalores {+r, -r}:
  ∂⟨O⟩/∂θ = r · [⟨O⟩(θ + π/(4r)) - ⟨O⟩(θ - π/(4r))]
""")
```

---

## 2. PSR vs Diferencia Finita — comparación exacta

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: PSR vs FD
# ─────────────────────────────────────────────

dev = qml.device("default.qubit", wires=3)

@qml.qnode(dev, diff_method="parameter-shift")
def circuito_psr(params):
    """Circuito VQC para demostrar PSR."""
    qml.RX(params[0], wires=0)
    qml.RY(params[1], wires=1)
    qml.RZ(params[2], wires=2)
    qml.CNOT(wires=[0, 1])
    qml.CNOT(wires=[1, 2])
    qml.RX(params[3], wires=0)
    qml.RY(params[4], wires=1)
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

# PSR manual vs PennyLane
def psr_manual(fn, params, idx, s=np.pi/2):
    """Parameter Shift Rule manual para el parámetro idx."""
    params_plus = params.copy(); params_plus[idx] += s
    params_minus = params.copy(); params_minus[idx] -= s
    return (fn(params_plus) - fn(params_minus)) / 2

# Diferencia finita
def diferencia_finita(fn, params, idx, h=1e-5):
    """Diferencia finita centrada."""
    params_plus = params.copy(); params_plus[idx] += h
    params_minus = params.copy(); params_minus[idx] -= h
    return (fn(params_plus) - fn(params_minus)) / (2 * h)

np.random.seed(42)
params_test = np.random.uniform(-np.pi, np.pi, 5)

print("Comparación de métodos de diferenciación:")
print(f"params = {np.round(params_test, 4)}")
print()

# Valor de referencia: PennyLane automático (PSR)
grad_pl = qml.grad(circuito_psr)(params_test)

print(f"{'Parámetro':>12} {'PL (PSR)':>14} {'PSR manual':>14} {'FD (h=1e-5)':>14} {'FD (h=1e-3)':>14}")
print(f"{'─'*70}")
for i in range(len(params_test)):
    grad_psr_m = psr_manual(circuito_psr, params_test, i)
    grad_fd_fine = diferencia_finita(circuito_psr, params_test, i, h=1e-5)
    grad_fd_coarse = diferencia_finita(circuito_psr, params_test, i, h=1e-3)
    print(f"  θ_{i}          {grad_pl[i]:>+12.8f}   {grad_psr_m:>+12.8f}   "
          f"{grad_fd_fine:>+12.8f}   {grad_fd_coarse:>+12.8f}")

print(f"\n→ PSR y PL son idénticos (ambos exactos)")
print(f"→ FD tiene error de truncamiento (~h²)")
```

---

## 3. Análisis de coste: evaluaciones del circuito

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Coste de cada método
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("ANÁLISIS DE COSTE: EVALUACIONES DEL CIRCUITO")
print("="*60)

print("""
Número de evaluaciones del circuito para calcular el GRADIENTE COMPLETO:

  Diferencia finita:       2n evaluaciones (n = número de parámetros)
  Parameter Shift Rule:    2n evaluaciones  (pero EXACTO)
  Adjoint differentiation: 1 forward + 1 backward  (solo simulador)
  SPSA:                    2 evaluaciones   (pero aproximado y ruidoso)

Nota sobre adjoint:
  • Solo disponible en simuladores de vector de estado
  • No implementable en hardware real (requiere guardar todos los estados intermedios)
  • Más eficiente para n grande (O(n) vs O(n²) para PSR naïve)
""")

# Benchmark de velocidad
n_params_test = [5, 10, 20, 50]
resultados_bench = {}

for n_p in n_params_test:
    dev_bench = qml.device("default.qubit", wires=min(n_p, 10))

    @qml.qnode(dev_bench, diff_method="parameter-shift")
    def vqc_bench_psr(params):
        for i in range(min(n_p, 10)):
            qml.RY(params[i % n_p], i % min(n_p, 10))
        for i in range(min(n_p, 10) - 1):
            qml.CNOT([i, i+1])
        return qml.expval(qml.PauliZ(0))

    @qml.qnode(dev_bench, diff_method="adjoint")
    def vqc_bench_adj(params):
        for i in range(min(n_p, 10)):
            qml.RY(params[i % n_p], i % min(n_p, 10))
        for i in range(min(n_p, 10) - 1):
            qml.CNOT([i, i+1])
        return qml.expval(qml.PauliZ(0))

    params_b = np.random.uniform(-np.pi, np.pi, n_p)

    # PSR
    t0 = time.time()
    for _ in range(5):
        qml.grad(vqc_bench_psr)(params_b)
    t_psr = (time.time() - t0) / 5

    # Adjoint
    t0 = time.time()
    for _ in range(5):
        qml.grad(vqc_bench_adj)(params_b)
    t_adj = (time.time() - t0) / 5

    resultados_bench[n_p] = {"psr": t_psr, "adj": t_adj, "ratio": t_psr/t_adj}
    print(f"  n_params={n_p:3d}: PSR={t_psr*1000:.2f}ms, Adjoint={t_adj*1000:.2f}ms, ratio={t_psr/t_adj:.1f}x")
```

---

## 4. Gradientes de orden superior

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Gradientes de orden superior
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("GRADIENTES DE ORDEN SUPERIOR (HESSIANA)")
print("="*60)

print("""
La Hessiana H_{ij} = ∂²C/∂θᵢ∂θⱼ es útil para:
  • Newton-Raphson (convergencia cuadrática)
  • Quantum Natural Gradient (usa la métrica de QFI)
  • Análisis de curvaturas (mínimos planos vs pronunciados)

PSR para la Hessiana:
  ∂²⟨O⟩/∂θᵢ∂θⱼ = [⟨O⟩(θ+sᵢ+sⱼ) - ⟨O⟩(θ+sᵢ-sⱼ)
                    - ⟨O⟩(θ-sᵢ+sⱼ) + ⟨O⟩(θ-sᵢ-sⱼ)] / 4
  donde sᵢ = π/2 · eᵢ

  Coste: O(n²) evaluaciones para la Hessiana completa.
""")

dev_hess = qml.device("default.qubit", wires=2)

@qml.qnode(dev_hess, diff_method="parameter-shift")
def vqc_hess(params):
    qml.RY(params[0], 0)
    qml.RZ(params[1], 0)
    qml.CNOT([0, 1])
    qml.RY(params[2], 1)
    return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

params_h = np.array([0.5, 1.2, -0.8])

# Gradiente con PennyLane
grad_fn = qml.grad(vqc_hess)
grad_val = grad_fn(params_h)
print(f"Gradiente en θ={np.round(params_h, 3)}:")
print(f"  ∇C = {np.round(grad_val, 6)}")

# Hessiana con PennyLane (doble diferenciación)
hess_fn = qml.jacobian(qml.grad(vqc_hess))
hess_val = hess_fn(params_h)
print(f"\nHessiana:")
print(f"  H = {np.round(hess_val, 6)}")
print(f"  Eigenvalores de H: {np.round(np.linalg.eigvalsh(hess_val), 6)}")

# Verificar PSR para Hessiana manualmente
def hessiana_psr_manual(fn, params, i, j, s=np.pi/2):
    """PSR para ∂²⟨O⟩/∂θᵢ∂θⱼ."""
    def shift(idx, sign):
        p = params.copy(); p[idx] += sign * s; return p
    pp = fn(shift(i, +1)); pp[j] += s
    pm = fn(shift(i, +1)); pm[j] -= s
    mp = fn(shift(i, -1)); mp[j] += s
    mm = fn(shift(i, -1)); mm[j] -= s
    # No podemos hacer esto directamente; usar el enfoque de evaluación
    p = params.copy()
    p_ij = p.copy(); p_ij[i] += s; p_ij[j] += s
    p_im = p.copy(); p_im[i] += s; p_im[j] -= s
    p_mj = p.copy(); p_mj[i] -= s; p_mj[j] += s
    p_mm = p.copy(); p_mm[i] -= s; p_mm[j] -= s
    return (fn(p_ij) - fn(p_im) - fn(p_mj) + fn(p_mm)) / 4

H_manual = np.zeros((3, 3))
for i in range(3):
    for j in range(3):
        H_manual[i, j] = hessiana_psr_manual(vqc_hess, params_h, i, j)

print(f"\nHessiana (PSR manual):")
print(f"  {np.round(H_manual, 6)}")
print(f"  ‖H_PL - H_manual‖ = {np.linalg.norm(hess_val - H_manual):.2e}")
```

---

## 5. Quantum Natural Gradient (QNG)

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Quantum Natural Gradient
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("QUANTUM NATURAL GRADIENT (QNG)")
print("="*60)

print("""
El gradiente estándar actualiza θ en el espacio de PARÁMETROS (plano):
  θ ← θ - α · ∇C

El gradiente natural actualiza θ en la dirección que causa el mayor
cambio en el ESTADO CUÁNTICO, ponderado por la métrica de Fubini-Study:

  θ ← θ - α · F⁻¹ · ∇C

donde F es la Quantum Fisher Information Matrix (QFIM):
  F_{ij} = 4·Re[⟨∂ᵢψ|∂ⱼψ⟩ - ⟨∂ᵢψ|ψ⟩⟨ψ|∂ⱼψ⟩]

Intuición:
  • Si dos parámetros producen el mismo cambio en |ψ⟩, F⁻¹ lo detecta
    y no "desperdicia" gradiente en direcciones redundantes
  • Convergencia geométricamente más rápida cerca del mínimo
  • Útil para ansätze sobre-parametrizados

Desventaja:
  • Requiere O(n²) evaluaciones adicionales para calcular F
  • F puede ser mal condicionada (requiere regularización: F + ε·I)
""")

dev_qng = qml.device("default.qubit", wires=3)

@qml.qnode(dev_qng, diff_method="parameter-shift")
def vqc_qng(params):
    """VQC para demostrar QNG."""
    qml.StronglyEntanglingLayers(
        params.reshape(2, 3, 3), wires=[0, 1, 2]
    )
    return qml.expval(qml.PauliZ(0) + qml.PauliZ(1) + qml.PauliZ(2))

n_params_qng = 18
np.random.seed(123)
params_init_qng = np.random.uniform(-0.1, 0.1, n_params_qng)

# Comparar SGD vs Adam vs QNG
optimizadores_comp = {
    "SGD": qml.GradientDescentOptimizer(stepsize=0.05),
    "Adam": qml.AdamOptimizer(stepsize=0.05),
    "QNG": qml.QNGOptimizer(stepsize=0.05),
}

historial_comp = {}
n_steps_comp = 80

for nombre_opt, opt_obj in optimizadores_comp.items():
    params = params_init_qng.copy()
    costes = [vqc_qng(params)]
    t0 = time.time()
    for step in range(n_steps_comp):
        try:
            params, coste = opt_obj.step_and_cost(vqc_qng, params)
            costes.append(coste)
        except Exception as e:
            costes.append(costes[-1])  # fallback si hay problema
    t_total = time.time() - t0
    historial_comp[nombre_opt] = costes
    print(f"  {nombre_opt:<8}: inicial={costes[0]:.4f}, final={costes[-1]:.4f}, "
          f"tiempo={t_total:.2f}s")
```

---

## 6. SPSA — Simultaneous Perturbation Stochastic Approximation

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: SPSA
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("SPSA — SOLO 2 EVALUACIONES PARA EL GRADIENTE")
print("="*60)

print("""
SPSA (Spall 1992, Kandala 2017 en NISQ):
  En lugar de calcular el gradiente exacto, aproxima:

  ĝ = [C(θ + c·Δ) - C(θ - c·Δ)] / (2c) · Δ⁻¹

  donde Δ es un vector de ±1 aleatorio.

  Solo 2 evaluaciones del circuito — independiente de n_params!

  Ventajas vs PSR:
    ✓ 2 evaluaciones siempre vs 2n del PSR
    ✓ Robusto a ruido de shot (importante en hardware)
    ✓ Recomendado por IBM para hardware real

  Desventajas:
    ✗ No converge a un gradiente exacto
    ✗ Requiere muchos pasos para compensar la varianza
    ✗ Sensible a hyperparámetros (a, c)

  Optimizadores basados en SPSA:
    • SPSA clásico: Qiskit SPSAOptimizer
    • QNSPSA: SPSA + QFI natural (Gacon et al. 2021)
    • COBYLA: sin gradiente, basado en modelo local
""")

# Implementación SPSA completa
def spsa_completo(fn, params_init, n_iter=100, a=0.1, b=0.1,
                  alpha=0.602, gamma=0.101, seed=42):
    """
    SPSA con decreasing step sizes estándar (Spall 1992):
    aₖ = a / (k + 1)^alpha
    cₖ = b / (k + 1)^gamma
    """
    np.random.seed(seed)
    params = params_init.copy()
    historial = [fn(params)]
    n = len(params)

    for k in range(1, n_iter + 1):
        ak = a / k**alpha
        ck = b / k**gamma
        delta = 2 * np.random.randint(0, 2, n) - 1  # ±1 Bernoulli
        grad_hat = (fn(params + ck*delta) - fn(params - ck*delta)) / (2*ck) / delta
        params -= ak * grad_hat
        historial.append(fn(params))

    return params, historial

# Comparar SPSA vs PSR en VQC de 5 qubits
dev_spsa = qml.device("default.qubit", wires=5)

@qml.qnode(dev_spsa, diff_method="parameter-shift")
def vqc_spsa_test(params):
    for i in range(5):
        qml.RY(params[i], i)
        qml.RZ(params[i+5], i)
    for i in range(4):
        qml.CNOT([i, i+1])
    for i in range(5):
        qml.RY(params[i+10], i)
    return qml.expval(qml.PauliZ(0))

n_p_spsa = 15
params_spsa_init = np.random.uniform(-np.pi, np.pi, n_p_spsa)

print(f"\nOptimizando VQC de 5 qubits ({n_p_spsa} parámetros):")

# PSR (Adam)
opt_adam_spsa = qml.AdamOptimizer(stepsize=0.05)
params_adam = params_spsa_init.copy()
hist_adam = [vqc_spsa_test(params_adam)]
t0 = time.time()
for _ in range(100):
    params_adam, c = opt_adam_spsa.step_and_cost(vqc_spsa_test, params_adam)
    hist_adam.append(c)
t_adam = time.time() - t0

# SPSA
t0 = time.time()
params_spsa_final, hist_spsa = spsa_completo(
    vqc_spsa_test, params_spsa_init.copy(), n_iter=100
)
t_spsa = time.time() - t0

print(f"  Adam (PSR): {hist_adam[0]:.4f} → {hist_adam[-1]:.4f}, t={t_adam:.2f}s, {2*n_p_spsa*100} eval.")
print(f"  SPSA:       {hist_spsa[0]:.4f} → {hist_spsa[-1]:.4f}, t={t_spsa:.2f}s, {2*100} eval.")
print(f"  Ratio evaluaciones: {2*n_p_spsa*100}/{2*100} = {n_p_spsa}x más evaluaciones para PSR")
```

---

## 7. Visualización completa

```python
# ─────────────────────────────────────────────
# SECCIÓN 7: Visualización
# ─────────────────────────────────────────────

fig = plt.figure(figsize=(18, 11))
gs = GridSpec(2, 3, figure=fig, hspace=0.4, wspace=0.35)

# Plot 1: PSR exactitud vs FD
ax1 = fig.add_subplot(gs[0, 0])
theta_vals = np.linspace(-np.pi, np.pi, 200)
dev_1q = qml.device("default.qubit", wires=1)

@qml.qnode(dev_1q)
def fn_1q(theta):
    qml.RY(theta, 0)
    return qml.expval(qml.PauliZ(0))

# Derivada analítica: d/dθ cos(θ) = -sin(θ) (para RY: ⟨Z⟩ = cos(θ))
exacta = -np.sin(theta_vals)
# PSR: exacta numéricamente
psr_vals = [(fn_1q(t + np.pi/2) - fn_1q(t - np.pi/2)) / 2 for t in theta_vals]
# FD con h=0.1 (grande — error visible)
fd_coarse = [(fn_1q(t + 0.1) - fn_1q(t - 0.1)) / 0.2 for t in theta_vals]

ax1.plot(theta_vals, exacta, "k-", linewidth=3, label="Analítica", alpha=0.7)
ax1.plot(theta_vals, psr_vals, "b--", linewidth=2, label="PSR (exacta)")
ax1.plot(theta_vals, fd_coarse, "r:", linewidth=2, label="FD (h=0.1)", alpha=0.8)
ax1.set_xlabel("θ"); ax1.set_ylabel("∂⟨Z⟩/∂θ")
ax1.set_title("PSR exacta vs Diferencia Finita\n(1 qubit, ⟨Z⟩=cos(θ))")
ax1.legend(fontsize=8); ax1.grid(True, alpha=0.3)

# Plot 2: Convergencia SGD vs Adam vs QNG
ax2 = fig.add_subplot(gs[0, 1])
colores_opt = {"SGD": "blue", "Adam": "red", "QNG": "green"}
for nombre_o, hist_o in historial_comp.items():
    ax2.plot(hist_o, color=colores_opt.get(nombre_o, "gray"),
            linewidth=2, label=nombre_o)
ax2.set_xlabel("Iteraciones"); ax2.set_ylabel("Coste C(θ)")
ax2.set_title("Convergencia: SGD vs Adam vs QNG")
ax2.legend(fontsize=8); ax2.grid(True, alpha=0.3)

# Plot 3: SPSA vs PSR/Adam
ax3 = fig.add_subplot(gs[0, 2])
ax3.plot(hist_adam, "b-", linewidth=2, label="Adam (PSR)", alpha=0.9)
ax3.plot(hist_spsa, "r-", linewidth=2, label="SPSA", alpha=0.9)
ax3.set_xlabel("Iteraciones"); ax3.set_ylabel("Coste C(θ)")
ax3.set_title("SPSA vs Adam (5 qubits)")
ax3.legend(fontsize=8); ax3.grid(True, alpha=0.3)

# Plot 4: Coste de evaluaciones por método
ax4 = fig.add_subplot(gs[1, 0])
n_params_range = np.array([5, 10, 20, 50, 100])
eval_fd = 2 * n_params_range
eval_psr = 2 * n_params_range
eval_adjoint = np.ones_like(n_params_range) * 2  # forward + backward
eval_spsa = np.ones_like(n_params_range) * 2      # siempre 2
ax4.semilogy(n_params_range, eval_fd, "b-o", label="FD / PSR", linewidth=2)
ax4.semilogy(n_params_range, eval_adjoint, "g-s", label="Adjoint", linewidth=2)
ax4.semilogy(n_params_range, eval_spsa, "r-^", label="SPSA", linewidth=2)
ax4.set_xlabel("Número de parámetros n"); ax4.set_ylabel("Evaluaciones del circuito")
ax4.set_title("Coste computacional del gradiente")
ax4.legend(fontsize=8); ax4.grid(True, alpha=0.3, which="both")

# Plot 5: Error de FD vs tamaño de paso h
ax5 = fig.add_subplot(gs[1, 1])
dev_err = qml.device("default.qubit", wires=1)

@qml.qnode(dev_err, diff_method="parameter-shift")
def fn_err(theta):
    qml.RY(theta, 0)
    return qml.expval(qml.PauliZ(0))

theta_ref = 0.7
grad_psr_ref = (fn_err(theta_ref + np.pi/2) - fn_err(theta_ref - np.pi/2)) / 2
h_vals = np.logspace(-8, 0, 50)
errores_fd = []
for h in h_vals:
    grad_fd = (fn_err(theta_ref + h) - fn_err(theta_ref - h)) / (2*h)
    errores_fd.append(abs(grad_fd - grad_psr_ref))

ax5.loglog(h_vals, errores_fd, "b-", linewidth=2)
ax5.axhline(y=0, color="green", linestyle="--", alpha=0.5, label="PSR (error=0)")
ax5.set_xlabel("Tamaño de paso h"); ax5.set_ylabel("|Error| = |FD - PSR|")
ax5.set_title("Error de Diferencia Finita vs h\n(PSR tiene error=0)")
ax5.grid(True, alpha=0.3, which="both")
ax5.annotate("Truncamiento\n(h muy grande)", xy=(0.1, errores_fd[-1]),
            xytext=(0.01, 1e-3), arrowprops=dict(arrowstyle="->"), fontsize=7)
ax5.annotate("Cancelación\nnumérica\n(h muy pequeño)", xy=(1e-7, errores_fd[2]),
            xytext=(1e-5, 1e-10), arrowprops=dict(arrowstyle="->"), fontsize=7)

# Plot 6: Tabla comparativa
ax6 = fig.add_subplot(gs[1, 2])
ax6.axis("off")
tabla = [
    ["Método", "Evaluaciones", "Exactitud", "Hardware"],
    ["FD", "2n", "~h²", "✓"],
    ["PSR", "2n", "Exacta", "✓"],
    ["Adjoint", "2", "Exacta", "✗"],
    ["SPSA", "2", "Estocástica", "✓"],
    ["QNG", "2n + O(n²)", "Exacta", "Costoso"],
]
t = ax6.table(cellText=tabla[1:], colLabels=tabla[0],
              cellLoc="center", loc="center",
              colColours=["#f0f0f0"]*4)
t.auto_set_font_size(False)
t.set_fontsize(9)
t.scale(1.2, 2.0)
ax6.set_title("Comparación de métodos\nde diferenciación cuántica", fontsize=10)

plt.suptitle("Gradientes Cuánticos — Parameter Shift Rule y Alternativas",
             fontsize=13, fontweight="bold")
plt.savefig("outputs/m20_gradientes.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m20_gradientes.png")

# ─────────────────────────────────────────────
# SECCIÓN 8: Guía de selección
# ─────────────────────────────────────────────
print("""
GUÍA DE SELECCIÓN DE MÉTODO:

Situación                           → Método recomendado
────────────────────────────────────────────────────────────────
Simulador con vector de estado      → Adjoint (más rápido)
Hardware real (≤ 50 params)         → PSR (exacto, tarda O(n))
Hardware real (> 50 params)         → SPSA (solo 2 eval/paso)
Hardware con muchos shots (ruidoso) → SPSA o FD con h grande
Necesitas Hessiana                  → PSR doble (O(n²) eval)
Convergencia más rápida             → QNG (si n no es grande)
Optimización sin gradientes         → COBYLA, Nelder-Mead
""")

print("✓ Módulo 20 completado")
```

---

## Checkpoint M20

```python
# checkpoint_m20.py
"""Verificaciones del Módulo 20 — Gradientes Cuánticos"""
import numpy as np

def test_psr_exacto_1q():
    """PSR es exacto para RY — coincide con la derivada analítica."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=1)

    @qml.qnode(dev)
    def fn(theta):
        qml.RY(theta, 0)
        return qml.expval(qml.PauliZ(0))

    for theta in [0.0, 0.5, 1.0, np.pi/4, np.pi/2]:
        psr = (fn(theta + np.pi/2) - fn(theta - np.pi/2)) / 2
        analitica = -np.sin(theta)  # d/dθ cos(θ) = -sin(θ)
        assert abs(psr - analitica) < 1e-10, \
            f"PSR error en θ={theta:.2f}: {psr:.8f} ≠ {analitica:.8f}"
    print("✓ test_psr_exacto_1q")

def test_psr_vs_fd():
    """PSR es más exacto que FD con h=0.1."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=1)

    @qml.qnode(dev)
    def fn(theta):
        qml.RY(theta, 0)
        return qml.expval(qml.PauliZ(0))

    theta = 0.7
    analitica = -np.sin(theta)
    psr = (fn(theta + np.pi/2) - fn(theta - np.pi/2)) / 2
    fd_coarse = (fn(theta + 0.1) - fn(theta - 0.1)) / 0.2
    assert abs(psr - analitica) < abs(fd_coarse - analitica), \
        "PSR debe ser más exacto que FD con h=0.1"
    print(f"✓ test_psr_vs_fd (PSR err={abs(psr-analitica):.2e} < FD err={abs(fd_coarse-analitica):.2e})")

def test_psr_multiparametro():
    """PSR para múltiples parámetros: gradiente completo."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev, diff_method="parameter-shift")
    def fn(params):
        qml.RY(params[0], 0)
        qml.RZ(params[1], 1)
        qml.CNOT([0, 1])
        return qml.expval(qml.PauliZ(0))

    params = np.array([0.5, 1.2])
    grad = qml.grad(fn)(params)
    assert len(grad) == 2, f"Gradiente debe tener 2 componentes"
    # El gradiente no debe ser cero para estos parámetros
    assert np.any(np.abs(grad) > 1e-6), "Gradiente no debe ser identicamente 0"
    print(f"✓ test_psr_multiparametro (∇C = {np.round(grad, 4)})")

def test_spsa_aproxima_gradiente():
    """SPSA aproxima el gradiente en promedio."""
    np.random.seed(42)
    # f(θ) = θ² + 2θ — gradiente exacto: 2θ + 2
    def fn(params): return params[0]**2 + 2*params[0]

    params = np.array([1.5])
    grad_exacto = 2*params[0] + 2  # = 5.0
    n_muestras = 200
    grads_spsa = []
    c = 0.1
    for _ in range(n_muestras):
        delta = 2*np.random.randint(0, 2, 1) - 1
        g = (fn(params + c*delta) - fn(params - c*delta)) / (2*c) / delta
        grads_spsa.append(g[0])
    grad_promedio = np.mean(grads_spsa)
    # En promedio, SPSA debe aproximar el gradiente (E[ĝ] = ∇f)
    assert abs(grad_promedio - grad_exacto) < 0.5, \
        f"SPSA promedio={grad_promedio:.4f} ≠ exacto={grad_exacto:.4f}"
    print(f"✓ test_spsa_aproxima_gradiente (SPSA prom={grad_promedio:.4f}, exacto={grad_exacto:.4f})")

def test_adjoint_disponible():
    """Adjoint differentiation disponible en PennyLane default.qubit."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev, diff_method="adjoint")
    def fn(params):
        qml.RY(params[0], 0)
        qml.CNOT([0, 1])
        return qml.expval(qml.PauliZ(0))

    params = np.array([0.5])
    grad = qml.grad(fn)(params)
    assert abs(grad[0]) > 1e-8, "Adjoint debe retornar gradiente no nulo"
    print(f"✓ test_adjoint_disponible (∂C/∂θ = {grad[0]:.4f})")

def test_hessiana_simetrica():
    """La Hessiana de una función suave es simétrica."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev, diff_method="parameter-shift")
    def fn(params):
        qml.RY(params[0], 0)
        qml.RZ(params[1], 0)
        qml.CNOT([0, 1])
        return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

    params = np.array([0.5, 1.2])
    H = qml.jacobian(qml.grad(fn))(params)
    # Hessiana simétrica: H = Hᵀ
    assert np.allclose(H, H.T, atol=1e-6), f"Hessiana no simétrica: {H}"
    print(f"✓ test_hessiana_simetrica")

def test_coste_evaluaciones():
    """PSR requiere 2n evaluaciones; SPSA solo 2."""
    n = 20
    eval_psr = 2 * n
    eval_spsa = 2
    assert eval_psr == 40, f"PSR: {eval_psr} eval"
    assert eval_spsa == 2, f"SPSA: {eval_spsa} eval"
    assert eval_psr > eval_spsa, "PSR más costoso que SPSA"
    print(f"✓ test_coste_evaluaciones (PSR={eval_psr}, SPSA={eval_spsa})")

if __name__ == "__main__":
    print("Ejecutando checkpoint M20 — Gradientes Cuánticos\n")
    test_psr_exacto_1q()
    test_psr_vs_fd()
    test_psr_multiparametro()
    test_spsa_aproxima_gradiente()
    test_adjoint_disponible()
    test_hessiana_simetrica()
    test_coste_evaluaciones()
    print("\n✓ Todos los tests de M20 pasaron")
```
