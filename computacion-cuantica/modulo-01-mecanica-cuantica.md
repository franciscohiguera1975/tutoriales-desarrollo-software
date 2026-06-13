# Tutorial de Computación Cuántica
## Módulo 2 — Mecánica Cuántica para Programadores

> **Objetivo:** Construir la intuición matemática de la mecánica cuántica usando el lenguaje de un programador. Sin física de partículas, sin ecuaciones de Schrödinger completas — solo los formalismos que se usan directamente al programar circuitos.
>
> **Herramientas:** NumPy para simular manualmente los conceptos antes de usar los frameworks. Las visualizaciones de la esfera de Bloch se harán en el Módulo 3 con Qiskit/Cirq.
>
> **Checkpoint final:** Simulación manual en NumPy de un circuito de dos qubits con superposición, entrelazamiento y medición. Sin usar ningún framework cuántico todavía.

---

## 2.1 El Qubit — Unidad Básica de Información Cuántica

### Del bit clásico al qubit

Un **bit** clásico es 0 o 1. Un **qubit** es una superposición de ambos simultáneamente hasta que se mide.

Matemáticamente, un qubit es un vector en un espacio de Hilbert de 2 dimensiones sobre los números complejos:

```
|ψ⟩ = α|0⟩ + β|1⟩
```

Donde:
- `α` y `β` son **amplitudes de probabilidad** — números complejos
- `|α|² + |β|² = 1` — normalización (las probabilidades suman 1)
- `|0⟩` y `|1⟩` son los **estados base** (base computacional)

### Los estados base en notación matricial

```python
import numpy as np

# Estado base |0⟩ — equivalente al bit 0
ket_0 = np.array([1+0j, 0+0j])

# Estado base |1⟩ — equivalente al bit 1
ket_1 = np.array([0+0j, 1+0j])

# Verificar ortonormalidad
print(f"⟨0|0⟩ = {np.dot(ket_0.conj(), ket_0):.1f}")  # 1.0
print(f"⟨1|1⟩ = {np.dot(ket_1.conj(), ket_1):.1f}")  # 1.0
print(f"⟨0|1⟩ = {np.dot(ket_0.conj(), ket_1):.1f}")  # 0.0 — ortogonales
```

### Superposición — el estado más importante

```python
# Estado de superposición igual (50%-50%)
# |+⟩ = (|0⟩ + |1⟩) / √2
ket_plus = (ket_0 + ket_1) / np.sqrt(2)
print(f"\n|+⟩ = {ket_plus}")
print(f"Norma: {np.linalg.norm(ket_plus):.4f}")  # 1.0

# Probabilidad de medir 0 y 1
prob_0 = abs(ket_plus[0])**2
prob_1 = abs(ket_plus[1])**2
print(f"P(0) = |α|² = {prob_0:.4f}")  # 0.5
print(f"P(1) = |β|² = {prob_1:.4f}")  # 0.5

# Estado de superposición asimétrica
alpha = np.sqrt(0.7) + 0j
beta  = np.sqrt(0.3) + 0j
psi = alpha * ket_0 + beta * ket_1
print(f"\n|ψ⟩ = {alpha:.4f}|0⟩ + {beta:.4f}|1⟩")
print(f"P(0) = {abs(alpha)**2:.4f}")  # 0.7
print(f"P(1) = {abs(beta)**2:.4f}")   # 0.3
```

### Las amplitudes son números complejos

El poder de los qubits viene de que las amplitudes pueden ser complejas. Esto permite **interferencia**: las amplitudes pueden sumarse (interferencia constructiva) o cancelarse (interferencia destructiva).

```python
# Amplitud compleja — el ángulo de fase es información real
alpha_complejo = np.exp(1j * np.pi/4) / np.sqrt(2)  # módulo 1/√2, fase π/4
beta_complejo  = np.exp(1j * np.pi/3) / np.sqrt(2)  # módulo 1/√2, fase π/3

psi_complejo = alpha_complejo * ket_0 + beta_complejo * ket_1

print(f"α = {alpha_complejo:.4f}")
print(f"β = {beta_complejo:.4f}")
print(f"P(0) = {abs(alpha_complejo)**2:.4f}")  # 0.5 — la fase no afecta la probabilidad
print(f"P(1) = {abs(beta_complejo)**2:.4f}")   # 0.5

# PERO la fase sí afecta los resultados de puertas posteriores
# Esto es lo que hace poderosos a los algoritmos cuánticos
```

> **Nota clave:** La fase no cambia las probabilidades de medición directa, pero sí cambia el resultado de operaciones posteriores. Los algoritmos cuánticos explotan esto para manipular interferencias.

---

## 2.2 La Esfera de Bloch — Visualización del Qubit

Todo estado de un qubit puede representarse como un punto en la superficie de una esfera de radio 1: la **esfera de Bloch**.

La parametrización usa dos ángulos:
```
|ψ⟩ = cos(θ/2)|0⟩ + e^(iφ) sin(θ/2)|1⟩
```

- `θ` (theta): ángulo polar — 0 es el polo norte (|0⟩), π es el polo sur (|1⟩)
- `φ` (phi): ángulo azimutal — la fase relativa entre |0⟩ y |1⟩

```python
import numpy as np

def estado_desde_angulos(theta, phi):
    """
    Genera un estado qubit a partir de coordenadas de la esfera de Bloch.

    Args:
        theta: ángulo polar [0, π]
        phi:   ángulo azimutal [0, 2π]

    Returns:
        np.array de shape (2,) con amplitudes complejas
    """
    alpha = np.cos(theta / 2)
    beta  = np.exp(1j * phi) * np.sin(theta / 2)
    return np.array([alpha, beta])

def coordenadas_bloch(psi):
    """Convierte un estado qubit a coordenadas cartesianas de la esfera de Bloch."""
    alpha, beta = psi[0], psi[1]
    x = 2 * np.real(alpha.conj() * beta)
    y = 2 * np.imag(alpha.conj() * beta)
    z = abs(alpha)**2 - abs(beta)**2
    return x, y, z

# Estados notables
estados = {
    '|0⟩':  estado_desde_angulos(0, 0),
    '|1⟩':  estado_desde_angulos(np.pi, 0),
    '|+⟩':  estado_desde_angulos(np.pi/2, 0),
    '|-⟩':  estado_desde_angulos(np.pi/2, np.pi),
    '|i⟩':  estado_desde_angulos(np.pi/2, np.pi/2),
    '|-i⟩': estado_desde_angulos(np.pi/2, 3*np.pi/2),
}

print(f"{'Estado':<8} {'α':>20} {'β':>20} {'(x,y,z) Bloch':>30}")
print('-' * 80)
for nombre, psi in estados.items():
    x, y, z = coordenadas_bloch(psi)
    print(f"{nombre:<8} {str(psi[0].round(3)):>20} {str(psi[1].round(3)):>20} "
          f"({x:.2f}, {y:.2f}, {z:.2f})")
```

### Visualización 3D de la esfera de Bloch

```python
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

def plot_bloch(estados_dict, titulo='Esfera de Bloch'):
    fig = plt.figure(figsize=(8, 8))
    ax = fig.add_subplot(111, projection='3d')

    # Dibujar la esfera
    u = np.linspace(0, 2 * np.pi, 30)
    v = np.linspace(0, np.pi, 30)
    xs = np.outer(np.cos(u), np.sin(v))
    ys = np.outer(np.sin(u), np.sin(v))
    zs = np.outer(np.ones_like(u), np.cos(v))
    ax.plot_surface(xs, ys, zs, alpha=0.05, color='cyan')

    # Ejes
    for eje in ['x', 'y', 'z']:
        ax.quiver(0, 0, 0,
                  1 if eje=='x' else 0, 1 if eje=='y' else 0, 1 if eje=='z' else 0,
                  color='gray', alpha=0.4, linewidth=1)
        ax.quiver(0, 0, 0,
                  -1 if eje=='x' else 0, -1 if eje=='y' else 0, -1 if eje=='z' else 0,
                  color='gray', alpha=0.2, linewidth=1)

    # Etiquetas de polo
    ax.text(0, 0, 1.15, '|0⟩', ha='center', fontsize=12, fontweight='bold')
    ax.text(0, 0, -1.15, '|1⟩', ha='center', fontsize=12, fontweight='bold')
    ax.text(1.15, 0, 0, '|+⟩', ha='center', fontsize=10, color='blue')
    ax.text(-1.15, 0, 0, '|-⟩', ha='center', fontsize=10, color='blue')

    # Plotear estados
    colores = plt.cm.Set1(np.linspace(0, 1, len(estados_dict)))
    for (nombre, psi), color in zip(estados_dict.items(), colores):
        x, y, z = coordenadas_bloch(psi)
        ax.quiver(0, 0, 0, x, y, z, color=color, linewidth=2.5,
                  arrow_length_ratio=0.15)
        ax.scatter([x], [y], [z], color=color, s=60, zorder=5)
        ax.text(x*1.2, y*1.2, z*1.1, nombre, fontsize=9, color=color)

    ax.set_xlabel('X'); ax.set_ylabel('Y'); ax.set_zlabel('Z')
    ax.set_title(titulo)
    ax.set_xlim([-1.2, 1.2]); ax.set_ylim([-1.2, 1.2]); ax.set_zlim([-1.2, 1.2])
    plt.tight_layout()
    plt.savefig('esfera_bloch.png', dpi=150)
    plt.show()

plot_bloch(estados)
```

---

## 2.3 Puertas Cuánticas — Operaciones sobre Qubits

Una **puerta cuántica** es una matriz unitaria que transforma el vector de estado. Las puertas son reversibles — siempre se puede deshacer una operación cuántica (a diferencia de la medición).

### Puertas de un qubit

```python
import numpy as np

# ─── Puertas de Pauli ───────────────────────────────────────────
# Pauli-X: inversión de bit (NOT cuántico)
# |0⟩ → |1⟩, |1⟩ → |0⟩
X = np.array([[0, 1],
              [1, 0]], dtype=complex)

# Pauli-Y: inversión con fase imaginaria
Y = np.array([[0, -1j],
              [1j,  0]], dtype=complex)

# Pauli-Z: inversión de fase
# |0⟩ → |0⟩, |1⟩ → -|1⟩
Z = np.array([[1,  0],
              [0, -1]], dtype=complex)

# ─── Puerta Hadamard ────────────────────────────────────────────
# La puerta más importante: crea superposición desde estados base
# |0⟩ → |+⟩ = (|0⟩ + |1⟩)/√2
# |1⟩ → |-⟩ = (|0⟩ - |1⟩)/√2
H = np.array([[1,  1],
              [1, -1]], dtype=complex) / np.sqrt(2)

# ─── Puertas de rotación ────────────────────────────────────────
def Rx(theta):
    """Rotación alrededor del eje X por ángulo theta."""
    c, s = np.cos(theta/2), np.sin(theta/2)
    return np.array([[c,    -1j*s],
                     [-1j*s, c   ]], dtype=complex)

def Ry(theta):
    """Rotación alrededor del eje Y por ángulo theta."""
    c, s = np.cos(theta/2), np.sin(theta/2)
    return np.array([[c, -s],
                     [s,  c]], dtype=complex)

def Rz(theta):
    """Rotación alrededor del eje Z por ángulo theta."""
    return np.array([[np.exp(-1j*theta/2), 0                ],
                     [0,                   np.exp(1j*theta/2)]], dtype=complex)

# ─── Puerta de fase ─────────────────────────────────────────────
def P(phi):
    """Puerta de fase — agrega fase e^(iφ) al estado |1⟩."""
    return np.array([[1, 0           ],
                     [0, np.exp(1j*phi)]], dtype=complex)

# Casos especiales de puerta de fase:
S = P(np.pi/2)    # √Z
T = P(np.pi/4)    # ⁴√Z — importante en algoritmo de Shor
```

### Verificar unitariedad de todas las puertas

```python
def verificar_unitaria(nombre, U):
    """Una puerta es unitaria si U @ U† = I"""
    producto = U @ U.conj().T
    es_unitaria = np.allclose(producto, np.eye(U.shape[0]))
    print(f"  {nombre}: {'✓ Unitaria' if es_unitaria else '✗ NO unitaria'}")

print("Verificación de puertas:")
verificar_unitaria('X (Pauli-X)', X)
verificar_unitaria('Y (Pauli-Y)', Y)
verificar_unitaria('Z (Pauli-Z)', Z)
verificar_unitaria('H (Hadamard)', H)
verificar_unitaria('Rx(π/3)', Rx(np.pi/3))
verificar_unitaria('Ry(π/4)', Ry(np.pi/4))
verificar_unitaria('Rz(2π/5)', Rz(2*np.pi/5))
verificar_unitaria('S', S)
verificar_unitaria('T', T)
```

### Aplicar puertas a estados

```python
ket_0 = np.array([1+0j, 0+0j])
ket_1 = np.array([0+0j, 1+0j])

def aplicar_puerta(U, psi):
    return U @ psi

def mostrar_estado(nombre, psi):
    p0 = abs(psi[0])**2
    p1 = abs(psi[1])**2
    print(f"  {nombre}: α={psi[0].round(4)}, β={psi[1].round(4)} "
          f"| P(0)={p0:.3f}, P(1)={p1:.3f}")

print("Efectos de la puerta Hadamard:")
mostrar_estado("H|0⟩", aplicar_puerta(H, ket_0))  # superposición 50-50
mostrar_estado("H|1⟩", aplicar_puerta(H, ket_1))  # superposición 50-50 con fase
mostrar_estado("H(H|0⟩)", aplicar_puerta(H, aplicar_puerta(H, ket_0)))  # vuelve a |0⟩

print("\nEfectos de rotaciones:")
mostrar_estado("Rx(π)|0⟩", aplicar_puerta(Rx(np.pi), ket_0))    # ≈ -i|1⟩
mostrar_estado("Ry(π/2)|0⟩", aplicar_puerta(Ry(np.pi/2), ket_0))  # (|0⟩+|1⟩)/√2
mostrar_estado("Rz(π)|0⟩", aplicar_puerta(Rz(np.pi), ket_0))    # fase global
```

---

## 2.4 Sistemas de Múltiples Qubits — Producto Tensorial

Un sistema de **n qubits** se representa como el **producto tensorial** de los estados individuales. El espacio de estados crece exponencialmente: 2 qubits → vector de 4 componentes, 3 qubits → 8, n qubits → 2ⁿ.

```python
import numpy as np

ket_0 = np.array([1+0j, 0+0j])
ket_1 = np.array([0+0j, 1+0j])

# Sistema de 2 qubits: estado |00⟩ = |0⟩ ⊗ |0⟩
ket_00 = np.kron(ket_0, ket_0)
ket_01 = np.kron(ket_0, ket_1)
ket_10 = np.kron(ket_1, ket_0)
ket_11 = np.kron(ket_1, ket_1)

print("Estados base de 2 qubits:")
print(f"|00⟩ = {ket_00}")
print(f"|01⟩ = {ket_01}")
print(f"|10⟩ = {ket_10}")
print(f"|11⟩ = {ket_11}")

# Estado general de 2 qubits
# |ψ⟩ = α₀₀|00⟩ + α₀₁|01⟩ + α₁₀|10⟩ + α₁₁|11⟩
amplitudes = np.array([0.5, 0.5, 0.5, 0.5])  # superposición igual de todos
print(f"\nSuperposición uniforme: {amplitudes}")
print(f"Norma: {np.linalg.norm(amplitudes):.4f}")

# Extender puertas a sistemas de 2 qubits
H_tensor_I = np.kron(H, np.eye(2))  # Hadamard en qubit 0, identidad en qubit 1
I_tensor_H = np.kron(np.eye(2), H)  # Identidad en qubit 0, Hadamard en qubit 1

estado_inicial = ket_00
tras_H_en_0 = H_tensor_I @ estado_inicial
print(f"\nHadamard en qubit 0, partiendo de |00⟩:")
print(f"  {tras_H_en_0}")
print(f"  P(00)={abs(tras_H_en_0[0])**2:.3f}, P(01)={abs(tras_H_en_0[1])**2:.3f}, "
      f"P(10)={abs(tras_H_en_0[2])**2:.3f}, P(11)={abs(tras_H_en_0[3])**2:.3f}")
```

### Puerta CNOT — Entrelazamiento

La puerta **CNOT** (Controlled-NOT) es la más importante de 2 qubits. Invierte el qubit objetivo si y solo si el qubit de control es |1⟩. Es la puerta que crea **entrelazamiento**.

```python
# Puerta CNOT en base |00⟩, |01⟩, |10⟩, |11⟩
# Control: qubit 0, Target: qubit 1
# |00⟩ → |00⟩
# |01⟩ → |01⟩
# |10⟩ → |11⟩  ← aquí invierte
# |11⟩ → |10⟩  ← aquí invierte
CNOT = np.array([[1, 0, 0, 0],
                 [0, 1, 0, 0],
                 [0, 0, 0, 1],
                 [0, 0, 1, 0]], dtype=complex)

# Verificar unitaria
producto = CNOT @ CNOT.conj().T
print("CNOT es unitaria:", np.allclose(producto, np.eye(4)))

# Crear un estado de Bell — el ejemplo de entrelazamiento más famoso
# Paso 1: H en qubit 0 → |+0⟩ = (|00⟩ + |10⟩)/√2
paso1 = H_tensor_I @ ket_00
print(f"\nTras H en qubit 0: {paso1.round(4)}")

# Paso 2: CNOT → estado de Bell Φ⁺
estado_bell = CNOT @ paso1
print(f"Estado de Bell |Φ⁺⟩: {estado_bell.round(4)}")
print(f"P(00) = {abs(estado_bell[0])**2:.3f}")
print(f"P(01) = {abs(estado_bell[1])**2:.3f}")
print(f"P(10) = {abs(estado_bell[2])**2:.3f}")
print(f"P(11) = {abs(estado_bell[3])**2:.3f}")
```

---

## 2.5 Entrelazamiento — La Propiedad sin Análogo Clásico

El **entrelazamiento cuántico** ocurre cuando el estado de múltiples qubits no puede expresarse como producto de estados individuales. Es la base del poder computacional cuántico.

```python
# ¿Está el estado de Bell entrelazado?
# Un estado NO entrelazado se puede escribir como |ψ_A⟩ ⊗ |ψ_B⟩
# Verificamos si el estado de Bell puede factorizarse

def esta_entrelazado(estado_2q, tol=1e-10):
    """
    Verifica entrelazamiento mediante el rango de la matriz densidad reducida.
    Un estado puro está entrelazado si su rango > 1.
    """
    # Reshapear como matriz 2x2 (Schmidt decomposition)
    M = estado_2q.reshape(2, 2)
    # Descomposición de valores singulares
    s = np.linalg.svd(M, compute_uv=False)
    # Estado separable → solo 1 valor singular no nulo
    rango = np.sum(s > tol)
    return rango > 1, s

print("Estado de Bell:")
entrelazado, svd_vals = esta_entrelazado(estado_bell)
print(f"  SVD valores: {svd_vals.round(4)}")
print(f"  ¿Entrelazado? {entrelazado}")  # True

print("\nEstado separable (|+⟩ ⊗ |0⟩):")
estado_sep = H_tensor_I @ ket_00
entrelazado, svd_vals = esta_entrelazado(estado_sep)
print(f"  SVD valores: {svd_vals.round(4)}")
print(f"  ¿Entrelazado? {entrelazado}")  # False

# Los 4 estados de Bell
def estado_bell_phi_plus():   return CNOT @ (H_tensor_I @ ket_00)
def estado_bell_phi_minus():  return CNOT @ (H_tensor_I @ ket_01)
def estado_bell_psi_plus():   return CNOT @ (H_tensor_I @ ket_10)
def estado_bell_psi_minus():  return CNOT @ (H_tensor_I @ ket_11)

print("\nLos 4 estados de Bell (base de Bell):")
for nombre, estado in [('|Φ⁺⟩', estado_bell_phi_plus()),
                        ('|Φ⁻⟩', estado_bell_phi_minus()),
                        ('|Ψ⁺⟩', estado_bell_psi_plus()),
                        ('|Ψ⁻⟩', estado_bell_psi_minus())]:
    ent, _ = esta_entrelazado(estado)
    print(f"  {nombre}: {estado.round(3)} | Entrelazado: {ent}")
```

> **¿Por qué importa en cuántica computacional?**
> El entrelazamiento permite que n qubits representen 2ⁿ estados simultáneamente. Un procesador de 300 qubits entrelazados representaría más estados que átomos hay en el universo observable. Los algoritmos cuánticos aprovechan esto para procesar información de forma masivamente paralela.

---

## 2.6 Medición — El Colapso de la Superposición

La **medición** es la operación que extrae información clásica de un estado cuántico. Al medir, el estado colapsa a uno de los estados base con probabilidad |amplitud|².

```python
import numpy as np

def medir_qubit(psi, n_shots=1000):
    """
    Simula la medición de un qubit n veces (shots).

    Args:
        psi: vector de estado del qubit
        n_shots: número de mediciones

    Returns:
        dict con conteos de 0 y 1
    """
    prob_0 = abs(psi[0])**2
    prob_1 = abs(psi[1])**2

    # Muestrear según las probabilidades
    resultados = np.random.choice([0, 1], size=n_shots, p=[prob_0, prob_1])
    conteos = {'0': np.sum(resultados == 0), '1': np.sum(resultados == 1)}

    return conteos, prob_0, prob_1

def medir_2qubits(estado, n_shots=1000):
    """Simula la medición de un sistema de 2 qubits."""
    probs = np.abs(estado)**2
    estados_base = ['00', '01', '10', '11']
    resultados = np.random.choice(estados_base, size=n_shots, p=probs)
    conteos = {s: np.sum(resultados == s) for s in estados_base}
    return conteos

# Medir estado de superposición |+⟩
ket_plus = (np.array([1+0j, 0+0j]) + np.array([0+0j, 1+0j])) / np.sqrt(2)
conteos, p0, p1 = medir_qubit(ket_plus, n_shots=1000)
print(f"Midiendo |+⟩ (1000 shots):")
print(f"  Teoría:    P(0)={p0:.3f}, P(1)={p1:.3f}")
print(f"  Resultado: P(0)≈{conteos['0']/1000:.3f}, P(1)≈{conteos['1']/1000:.3f}")

# Medir estado de Bell — entrelazamiento en acción
print(f"\nMidiendo |Φ⁺⟩ (1000 shots):")
conteos_bell = medir_2qubits(estado_bell, n_shots=1000)
print(f"  Conteos: {conteos_bell}")
print(f"  Obs: Solo aparecen 00 y 11 — los qubits siempre colapsan juntos")
print(f"  Esto es entrelazamiento: medir uno determina el otro instantáneamente")
```

### La medición destruye la superposición

```python
def simular_colapso(psi):
    """
    Simula el colapso de un estado cuántico al medir.
    Después de medir, el estado ya no es superposición.
    """
    prob_0 = abs(psi[0])**2
    # Simular resultado de medición
    resultado = np.random.choice([0, 1], p=[prob_0, 1-prob_0])

    # Estado tras la medición (colapsado)
    if resultado == 0:
        estado_post_medicion = np.array([1+0j, 0+0j])
    else:
        estado_post_medicion = np.array([0+0j, 1+0j])

    return resultado, estado_post_medicion

ket_plus = (np.array([1+0j, 0+0j]) + np.array([0+0j, 1+0j])) / np.sqrt(2)

print("Simulando 5 mediciones de |+⟩:")
for i in range(5):
    resultado, post = simular_colapso(ket_plus)
    print(f"  Medición {i+1}: resultado={resultado} | estado post-medición={post}")
    print(f"             Si mido de nuevo: siempre obtengo {resultado}")
```

---

## 2.7 Circuito Cuántico Manual — Simulador desde Cero

Implementaremos un simulador básico que ejecuta circuitos de hasta 3 qubits. Esto es lo que hacen Qiskit, Cirq y PennyLane internamente.

```python
import numpy as np
from functools import reduce

class SimuladorCuantico:
    """
    Simulador cuántico minimal para n qubits.
    Usa representación vectorial (statevector).
    """

    def __init__(self, n_qubits):
        self.n = n_qubits
        self.dim = 2 ** n_qubits
        # Estado inicial: todos los qubits en |0⟩
        self.estado = np.zeros(self.dim, dtype=complex)
        self.estado[0] = 1.0  # |000...0⟩

    def _puerta_en_posicion(self, U, qubit):
        """Expande una puerta de 1 qubit al espacio completo de n qubits."""
        ops = []
        for i in range(self.n):
            if i == qubit:
                ops.append(U)
            else:
                ops.append(np.eye(2, dtype=complex))
        # Producto tensorial de todas las operaciones
        return reduce(np.kron, ops)

    def h(self, qubit):
        """Aplica Hadamard al qubit indicado."""
        H = np.array([[1, 1], [1, -1]], dtype=complex) / np.sqrt(2)
        U = self._puerta_en_posicion(H, qubit)
        self.estado = U @ self.estado
        return self

    def x(self, qubit):
        """Aplica Pauli-X (NOT) al qubit indicado."""
        X = np.array([[0, 1], [1, 0]], dtype=complex)
        U = self._puerta_en_posicion(X, qubit)
        self.estado = U @ self.estado
        return self

    def z(self, qubit):
        """Aplica Pauli-Z al qubit indicado."""
        Z = np.array([[1, 0], [0, -1]], dtype=complex)
        U = self._puerta_en_posicion(Z, qubit)
        self.estado = U @ self.estado
        return self

    def ry(self, qubit, theta):
        """Aplica rotación Ry(theta) al qubit indicado."""
        c, s = np.cos(theta/2), np.sin(theta/2)
        Ry_mat = np.array([[c, -s], [s, c]], dtype=complex)
        U = self._puerta_en_posicion(Ry_mat, qubit)
        self.estado = U @ self.estado
        return self

    def rz(self, qubit, phi):
        """Aplica rotación Rz(phi) al qubit indicado."""
        Rz_mat = np.array([[np.exp(-1j*phi/2), 0],
                           [0, np.exp(1j*phi/2)]], dtype=complex)
        U = self._puerta_en_posicion(Rz_mat, qubit)
        self.estado = U @ self.estado
        return self

    def cnot(self, control, target):
        """Aplica CNOT con qubit de control y target indicados."""
        U = np.eye(self.dim, dtype=complex)
        for i in range(self.dim):
            bits = format(i, f'0{self.n}b')
            if bits[control] == '1':
                # Invertir el bit target
                bits_list = list(bits)
                bits_list[target] = '0' if bits_list[target] == '1' else '1'
                j = int(''.join(bits_list), 2)
                U[i, i] = 0
                U[j, i] = 1
                U[i, j] = 0
                U[j, j] = 0
        self.estado = U @ self.estado
        return self

    def medir(self, n_shots=1024):
        """Simula mediciones y retorna conteos."""
        probs = np.abs(self.estado)**2
        estados_base = [format(i, f'0{self.n}b') for i in range(self.dim)]
        resultados = np.random.choice(estados_base, size=n_shots, p=probs)
        conteos = {s: int(np.sum(resultados == s)) for s in estados_base
                   if np.sum(resultados == s) > 0}
        return dict(sorted(conteos.items()))

    def probabilidades(self):
        """Retorna las probabilidades teóricas."""
        probs = np.abs(self.estado)**2
        estados_base = [format(i, f'0{self.n}b') for i in range(self.dim)]
        return {s: round(float(p), 6) for s, p in zip(estados_base, probs) if p > 1e-10}

    def vector_estado(self):
        return self.estado.copy()

    def reset(self):
        self.estado = np.zeros(self.dim, dtype=complex)
        self.estado[0] = 1.0
        return self


# ─── Ejemplo 1: Estado de Bell ────────────────────────────────────
print("=" * 50)
print("Circuito: Estado de Bell")
print("=" * 50)
sim = SimuladorCuantico(2)
sim.h(0).cnot(0, 1)

print(f"Vector de estado: {sim.vector_estado().round(4)}")
print(f"Probabilidades:   {sim.probabilidades()}")
print(f"Mediciones (1024 shots): {sim.medir()}")

# ─── Ejemplo 2: Superposición de 3 qubits ─────────────────────────
print("\n" + "=" * 50)
print("Circuito: Superposición de 3 qubits (H⊗H⊗H)")
print("=" * 50)
sim3 = SimuladorCuantico(3)
sim3.h(0).h(1).h(2)

print(f"Probabilidades: {sim3.probabilidades()}")
print(f"Mediciones (1024 shots): {sim3.medir()}")
print("Obs: 8 estados con probabilidad ~1/8 cada uno")

# ─── Ejemplo 3: Circuito variacional simple ───────────────────────
print("\n" + "=" * 50)
print("Circuito Variacional: Ry(θ)|0⟩")
print("=" * 50)
for theta in [0, np.pi/4, np.pi/2, 3*np.pi/4, np.pi]:
    sim_v = SimuladorCuantico(1)
    sim_v.ry(0, theta)
    probs = sim_v.probabilidades()
    p0 = probs.get('0', 0)
    p1 = probs.get('1', 0)
    print(f"  θ = {theta/np.pi:.2f}π → P(0)={p0:.4f}, P(1)={p1:.4f}")
```

---

## 2.8 Valor Esperado — La Salida de un Circuito Cuántico

En QML, la salida de un circuito cuántico no es un bit sino el **valor esperado** de un observable (típicamente la puerta Pauli-Z). Esto es lo que se usa como predicción.

```python
# Observable Pauli-Z: eigenvalor +1 para |0⟩, -1 para |1⟩
# ⟨Z⟩ = P(0) - P(1)
# Rango: [-1, +1]

def valor_esperado_Z(psi):
    """Calcula ⟨ψ|Z|ψ⟩ = |α|² - |β|² = P(0) - P(1)"""
    return abs(psi[0])**2 - abs(psi[1])**2

def valor_esperado_Z_sim(sim, qubit=0, n_shots=10000):
    """Estima ⟨Z⟩ mediante muestreo (como en hardware real)."""
    conteos = sim.medir(n_shots)
    plus1  = sum(v for k, v in conteos.items() if k[qubit] == '0')
    minus1 = sum(v for k, v in conteos.items() if k[qubit] == '1')
    return (plus1 - minus1) / n_shots

print("Valor esperado de Z para distintos estados:")
for theta in [0, np.pi/4, np.pi/2, 3*np.pi/4, np.pi]:
    sim_v = SimuladorCuantico(1)
    sim_v.ry(0, theta)
    psi = sim_v.vector_estado()
    vev_exacto = valor_esperado_Z(psi)
    vev_shot   = valor_esperado_Z_sim(sim_v.reset().ry(0, theta))
    print(f"  Ry({theta/np.pi:.2f}π)|0⟩: ⟨Z⟩_exacto={vev_exacto:.4f}, "
          f"⟨Z⟩_shots={vev_shot:.4f}")

print("\n⟨Z⟩ ∈ [-1, +1] — esto se usa directamente como predicción en QML")
print("Para clasificación binaria: signo(⟨Z⟩) → clase 0 o clase 1")
```

---

## ✅ Checkpoint Módulo 2

### Simulador manual de circuitos

Ejecuta el circuito completo de verificación:

```python
# checkpoint_m2.py
import numpy as np

# ─── Parte 1: Estado de Bell ──────────────────────────────────────
sim = SimuladorCuantico(2)
sim.h(0).cnot(0, 1)
probs = sim.probabilidades()
assert abs(probs.get('00', 0) - 0.5) < 0.01, "Error: P(00) debe ser 0.5"
assert abs(probs.get('11', 0) - 0.5) < 0.01, "Error: P(11) debe ser 0.5"
assert '01' not in probs and '10' not in probs, "Error: No debe haber P(01) o P(10)"
print("✓ Estado de Bell verificado")

# ─── Parte 2: Unitariedad de puertas ────────────────────────────
H = np.array([[1, 1], [1, -1]]) / np.sqrt(2)
assert np.allclose(H @ H.conj().T, np.eye(2)), "H no es unitaria"
print("✓ Unitariedad de Hadamard verificada")

# ─── Parte 3: Born rule ──────────────────────────────────────────
theta = np.pi / 3
sim_v = SimuladorCuantico(1)
sim_v.ry(0, theta)
psi = sim_v.vector_estado()
assert abs(abs(psi[0])**2 + abs(psi[1])**2 - 1.0) < 1e-10, "Norma != 1"
print("✓ Born rule (norma unitaria) verificada")

# ─── Parte 4: Valor esperado ────────────────────────────────────
vev = abs(psi[0])**2 - abs(psi[1])**2
assert -1.0 <= vev <= 1.0, "⟨Z⟩ fuera de rango [-1, 1]"
print(f"✓ Valor esperado ⟨Z⟩ = {vev:.4f} ∈ [-1, 1]")

print("\n✓ Checkpoint Módulo 2 completado")
```

### Conceptos dominados

| Concepto | Verificación |
|---|---|
| Qubit como vector en C² | Amplitudes complejas, norma 1 |
| Esfera de Bloch | 6 estados notables mapeados |
| Puertas unitarias | X, Y, Z, H, Rx, Ry, Rz, S, T |
| Producto tensorial | Sistemas de 2 y 3 qubits |
| CNOT y entrelazamiento | Estado de Bell creado y verificado |
| Medición y Born rule | Simulación con n_shots |
| Valor esperado ⟨Z⟩ | Salida de circuitos en QML |
| Simulador en NumPy | Circuitos de hasta 3 qubits |

### Archivos generados

| Archivo | Descripción |
|---|---|
| `esfera_bloch.png` | Visualización 3D de estados notables |
| `checkpoint_m2.py` | Script de verificación |

---

## Resumen

| Concepto | Estado |
|---|---|
| Qubits: amplitudes complejas, Born rule | ✅ |
| Esfera de Bloch: θ y φ | ✅ |
| Puertas de 1 qubit: X, Y, Z, H, Rx, Ry, Rz, S, T | ✅ |
| Sistemas de múltiples qubits: producto tensorial | ✅ |
| Entrelazamiento: estados de Bell, SVD test | ✅ |
| Medición: colapso, Born rule, n_shots | ✅ |
| Simulador manual en NumPy | ✅ |
| Valor esperado ⟨Z⟩ como salida de circuito | ✅ |

**Siguiente módulo →** M3: Setup del triple stack (Qiskit + PennyLane + Cirq) — mismo circuito, tres implementaciones
