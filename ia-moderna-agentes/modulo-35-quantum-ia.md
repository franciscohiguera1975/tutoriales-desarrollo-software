# Módulo 35 — Convergencia IA + Computación Cuántica

> **Objetivo:** Comprender los principios de la computación cuántica (qubits, superposición, entrelazamiento, puertas cuánticas) y cómo se intersectan con la IA: algoritmos de optimización cuántica (QAOA, VQE), QML y limitaciones actuales del NISQ era.
>
> **Herramientas:** Python 3.11, math, cmath, json, random
>
> **Prerequisito:** M06 (Arquitectura LLMs), M32 (Optimización)

---

## 1. Fundamentos: Qubits y Puertas Cuánticas

```python
# modulo-35-quantum-ia.py — Parte 1: Simulador cuántico básico

"""
La computación cuántica clásica vs cuántica:

┌─────────────────────────────────────────────────────────────────┐
│ BIT clásico: 0 ó 1                                             │
│ QUBIT: α|0⟩ + β|1⟩  donde |α|² + |β|² = 1                    │
│                                                                  │
│ Propiedades cuánticas clave:                                     │
│  - Superposición: un qubit puede ser 0 Y 1 simultáneamente      │
│  - Entrelazamiento: dos qubits correlacionados instantáneamente  │
│  - Interferencia: amplitudes se suman/cancelan                   │
│  - Medición: colapsa la superposición a 0 ó 1                   │
│                                                                  │
│ Para IA, la promesa cuántica es:                                 │
│  - Optimización cuadrática más rápida (QUBO → QAOA)             │
│  - Álgebra lineal cuántica (HHL algorithm)                       │
│  - Kernels cuánticos para QML                                    │
│  - Muestreo cuántico para modelos generativos                    │
└─────────────────────────────────────────────────────────────────┘
"""

import math
import cmath
import json
import random
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple, Union


# ── Qubit ─────────────────────────────────────────────────────────────────────

@dataclass
class Qubit:
    """
    Estado de un qubit: |ψ⟩ = α|0⟩ + β|1⟩
    donde α, β son números complejos con |α|² + |β|² = 1
    """
    alpha: complex = complex(1, 0)  # amplitud del estado |0⟩
    beta: complex  = complex(0, 0)  # amplitud del estado |1⟩

    def __post_init__(self):
        self._normalizar()

    def _normalizar(self):
        """Asegura que |α|² + |β|² = 1"""
        norm = math.sqrt(abs(self.alpha)**2 + abs(self.beta)**2)
        if norm > 0:
            self.alpha /= norm
            self.beta  /= norm

    @property
    def prob_0(self) -> float:
        """Probabilidad de medir |0⟩"""
        return abs(self.alpha)**2

    @property
    def prob_1(self) -> float:
        """Probabilidad de medir |1⟩"""
        return abs(self.beta)**2

    def medir(self) -> int:
        """Colapsa el estado cuántico y retorna 0 ó 1."""
        if random.random() < self.prob_0:
            self.alpha = complex(1, 0)
            self.beta  = complex(0, 0)
            return 0
        else:
            self.alpha = complex(0, 0)
            self.beta  = complex(1, 0)
            return 1

    def to_dict(self) -> Dict:
        return {
            "alpha": [self.alpha.real, self.alpha.imag],
            "beta":  [self.beta.real,  self.beta.imag],
            "prob_0": round(self.prob_0, 4),
            "prob_1": round(self.prob_1, 4),
        }

    def __repr__(self):
        return (f"Qubit({self.alpha.real:.3f}|0⟩ + "
                f"{self.beta.real:.3f}|1⟩) "
                f"[P0={self.prob_0:.3f}, P1={self.prob_1:.3f}]")


# ── Puertas cuánticas (matrices 2x2) ─────────────────────────────────────────

class PuertaCuantica:
    """Representa una puerta cuántica como matriz unitaria 2x2."""

    def __init__(self, nombre: str, matriz: List[List[complex]]):
        self.nombre = nombre
        self.matriz = matriz  # [[a, b], [c, d]]

    def aplicar(self, qubit: Qubit) -> Qubit:
        """Aplica la puerta al qubit: |ψ'⟩ = U|ψ⟩"""
        a, b = self.matriz[0]
        c, d = self.matriz[1]
        nuevo_alpha = a * qubit.alpha + b * qubit.beta
        nuevo_beta  = c * qubit.alpha + d * qubit.beta
        return Qubit(nuevo_alpha, nuevo_beta)

    @staticmethod
    def hadamard() -> "PuertaCuantica":
        """Puerta H: crea superposición uniforme desde |0⟩."""
        c = 1 / math.sqrt(2)
        return PuertaCuantica("H", [[complex(c), complex(c)], [complex(c), complex(-c)]])

    @staticmethod
    def pauli_x() -> "PuertaCuantica":
        """Puerta X (NOT cuántico): intercambia |0⟩ y |1⟩."""
        return PuertaCuantica("X", [[complex(0), complex(1)], [complex(1), complex(0)]])

    @staticmethod
    def pauli_z() -> "PuertaCuantica":
        """Puerta Z: cambia la fase de |1⟩."""
        return PuertaCuantica("Z", [[complex(1), complex(0)], [complex(0), complex(-1)]])

    @staticmethod
    def phase(theta: float) -> "PuertaCuantica":
        """Puerta de fase: R(θ) = [[1, 0], [0, e^(iθ)]]"""
        return PuertaCuantica("R", [[complex(1), complex(0)], [complex(0), cmath.exp(complex(0, theta))]])

    @staticmethod
    def rx(theta: float) -> "PuertaCuantica":
        """Rotación alrededor del eje X: Rx(θ)"""
        c = math.cos(theta / 2)
        s = math.sin(theta / 2)
        return PuertaCuantica("Rx", [[complex(c), complex(0, -s)], [complex(0, -s), complex(c)]])


# ── Registro cuántico (múltiples qubits) ──────────────────────────────────────

class RegistroCuantico:
    """
    Registro de N qubits con estado como vector de amplitudes de 2^N entradas.
    Para N=2: |00⟩, |01⟩, |10⟩, |11⟩
    """

    def __init__(self, n_qubits: int):
        self.n = n_qubits
        dim = 2 ** n_qubits
        # Estado inicial: todos en |0...0⟩
        self._state: List[complex] = [complex(0)] * dim
        self._state[0] = complex(1)  # |00...0⟩ con amplitud 1

    def aplicar_hadamard(self, qubit_idx: int):
        """Aplica H al qubit en la posición qubit_idx."""
        dim = 2 ** self.n
        nuevo_state = [complex(0)] * dim
        c = 1 / math.sqrt(2)

        for i in range(dim):
            bit = (i >> qubit_idx) & 1
            j = i ^ (1 << qubit_idx)  # flip bit qubit_idx

            if bit == 0:
                nuevo_state[i]  += c * self._state[i]
                nuevo_state[j]  += c * self._state[i]
            else:
                nuevo_state[j]  += c * self._state[i]
                nuevo_state[i]  -= c * self._state[i]

        self._state = nuevo_state

    def aplicar_cnot(self, control: int, target: int):
        """Puerta CNOT: flip del target si el control es |1⟩."""
        dim = 2 ** self.n
        nuevo_state = [complex(0)] * dim

        for i in range(dim):
            if (i >> control) & 1:  # si el qubit de control es 1
                j = i ^ (1 << target)  # flip del target
                nuevo_state[j] += self._state[i]
            else:
                nuevo_state[i] += self._state[i]

        self._state = nuevo_state

    def medir(self) -> List[int]:
        """Mide todos los qubits. Colapsa el estado."""
        probs = [abs(a)**2 for a in self._state]
        # Normalizar por si hay pequeños errores numéricos
        total = sum(probs)
        probs = [p / total for p in probs]

        # Seleccionar estado según distribución de probabilidad
        r = random.random()
        acumulado = 0
        estado_medido = 0
        for i, p in enumerate(probs):
            acumulado += p
            if r <= acumulado:
                estado_medido = i
                break

        # Colapsar al estado medido
        self._state = [complex(0)] * (2 ** self.n)
        self._state[estado_medido] = complex(1)

        # Extraer bits individuales
        return [(estado_medido >> i) & 1 for i in range(self.n)]

    def probabilidades(self) -> List[Tuple[str, float]]:
        """Retorna las probabilidades de cada estado base."""
        resultado = []
        for i, amp in enumerate(self._state):
            prob = abs(amp) ** 2
            if prob > 1e-10:
                estado_str = format(i, "0" + str(self.n) + "b")
                resultado.append((f"|{estado_str}⟩", round(prob, 4)))
        return resultado


# ── Demo 1 ────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Simulador Cuántico Básico")
    print("=" * 60)

    # Qubit simple y puertas
    q = Qubit()  # |0⟩
    print(f"\nInicial: {q}")

    H = PuertaCuantica.hadamard()
    q_super = H.aplicar(q)
    print(f"Tras H:  {q_super}")

    X = PuertaCuantica.pauli_x()
    q_flipped = X.aplicar(Qubit())
    print(f"Tras X:  {q_flipped}")

    # Registro de 2 qubits: estado de Bell (entrelazamiento)
    print("\n--- Estado de Bell (entrelazamiento) ---")
    reg = RegistroCuantico(2)
    reg.aplicar_hadamard(0)   # H en qubit 0
    reg.aplicar_cnot(0, 1)    # CNOT: control=0, target=1
    probs = reg.probabilidades()
    print(f"Probabilidades: {probs}")
    medicion = reg.medir()
    print(f"Medición: {medicion} (ambos qubits siempre iguales por entrelazamiento)")

    # Experimento: medir qubit en superposición N veces
    print("\n--- Estadística de mediciones (N=100) ---")
    conteo = {"0": 0, "1": 0}
    for _ in range(100):
        q_test = Qubit()
        q_test = H.aplicar(q_test)  # superposición 50/50
        resultado = q_test.medir()
        conteo[str(resultado)] += 1
    print(f"0: {conteo['0']}%, 1: {conteo['1']}% (esperado ~50% cada uno)")

    with open("outputs/m35_qubits.json", "w", encoding="utf-8") as f:
        json.dump({
            "qubit_inicial": Qubit().to_dict(),
            "qubit_superposicion": q_super.to_dict(),
            "estado_bell": probs,
            "estadistica_mediciones": conteo,
        }, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m35_qubits.json")
```

---

## 2. QAOA: Quantum Approximate Optimization Algorithm

```python
# Parte 2: QAOA para problemas de optimización combinatoria

"""
QAOA resuelve problemas QUBO (Quadratic Unconstrained Binary Optimization)
que son la forma natural de muchos problemas de ML:
- Selección de features
- Clustering
- Scheduling
- MaxCut (particionamiento de grafos)

QAOA alterna entre:
1. Capa de problema (cost): e^{-iγ·C}
2. Capa de mixing (driver): e^{-iβ·B}

Los parámetros γ, β se optimizan clásicamente (loop variacional).
"""

import itertools


def energia_maxcut(bits: List[int], aristas: List[Tuple[int, int]]) -> float:
    """
    Calcula la energía (número de aristas cortadas) para un estado MaxCut.
    MaxCut: particionar nodos en dos grupos maximizando aristas entre grupos.
    bits[i] = 0 o 1 indica el grupo del nodo i.
    """
    return sum(1 for u, v in aristas if bits[u] != bits[v])


class CircuitoQAOA:
    """
    Simulación clásica de QAOA para MaxCut.
    Para n qubits, simula el estado como vector de 2^n amplitudes.
    """

    def __init__(self, n_nodos: int, aristas: List[Tuple[int, int]], p_capas: int = 1):
        self.n = n_nodos
        self.aristas = aristas
        self.p = p_capas
        self.dim = 2 ** n_nodos

    def estado_inicial(self) -> List[complex]:
        """Estado inicial |+⟩^n: superposición uniforme de todos los estados."""
        amp = complex(1 / math.sqrt(self.dim))
        return [amp] * self.dim

    def aplicar_capa_problema(self, state: List[complex], gamma: float) -> List[complex]:
        """
        Capa de costo: e^{-iγ·C} donde C es el hamiltoniano del problema.
        Para cada arista (u,v), aplica fase si los qubits son distintos.
        """
        nuevo = list(state)
        for i in range(self.dim):
            bits = [(i >> q) & 1 for q in range(self.n)]
            for u, v in self.aristas:
                if bits[u] != bits[v]:
                    nuevo[i] *= cmath.exp(complex(0, -gamma))
        return nuevo

    def aplicar_capa_mixing(self, state: List[complex], beta: float) -> List[complex]:
        """
        Capa de mixing: e^{-iβ·B} donde B = Σ X_i (suma de Pauli X).
        Aplica rotación Rx(2β) a cada qubit.
        """
        cos_b = math.cos(beta)
        sin_b = math.sin(beta)
        nuevo = list(state)

        for q in range(self.n):
            temp = [complex(0)] * self.dim
            for i in range(self.dim):
                j = i ^ (1 << q)  # flip qubit q
                temp[i] += cos_b * nuevo[i]
                temp[i] -= complex(0, sin_b) * nuevo[j]
            nuevo = temp

        return nuevo

    def ejecutar(self, params: List[float]) -> Tuple[List[complex], float]:
        """
        Ejecuta el circuito QAOA con parámetros [γ1, β1, γ2, β2, ...].
        Retorna (estado_final, energia_esperada).
        """
        state = self.estado_inicial()
        gammas = params[0::2][:self.p]
        betas  = params[1::2][:self.p]

        for k in range(self.p):
            state = self.aplicar_capa_problema(state, gammas[k])
            state = self.aplicar_capa_mixing(state, betas[k])

        # Calcular energía esperada ⟨ψ|C|ψ⟩
        energia = sum(
            abs(state[i])**2 * energia_maxcut([(i >> q) & 1 for q in range(self.n)], self.aristas)
            for i in range(self.dim)
        )
        return state, energia

    def optimizar(self, n_iter: int = 20) -> Dict:
        """
        Optimiza los parámetros QAOA con búsqueda de cuadrícula (simplificado).
        En producción: gradient-free optimizer (COBYLA, SPSA, etc.)
        """
        mejor_energia = -float("inf")
        mejores_params = None
        mejor_estado = None
        historial = []

        # Grid search sobre γ y β (simplificado para 1 capa)
        for gamma in [i * math.pi / 8 for i in range(1, 5)]:
            for beta in [i * math.pi / 8 for i in range(1, 5)]:
                params = [gamma, beta]
                estado, energia = self.ejecutar(params)
                historial.append({"gamma": round(gamma, 3), "beta": round(beta, 3), "energia": round(energia, 4)})

                if energia > mejor_energia:
                    mejor_energia = energia
                    mejores_params = params
                    mejor_estado = estado

        # Extraer la solución más probable del estado final
        probs = [(abs(a)**2, i) for i, a in enumerate(mejor_estado)]
        probs.sort(reverse=True)
        estado_mas_probable = probs[0][1]
        bits_solucion = [(estado_mas_probable >> q) & 1 for q in range(self.n)]

        return {
            "mejor_energia": round(mejor_energia, 4),
            "mejores_params": {"gamma": round(mejores_params[0], 3), "beta": round(mejores_params[1], 3)},
            "solucion_bits": bits_solucion,
            "aristas_cortadas": int(energia_maxcut(bits_solucion, self.aristas)),
            "max_posible": len(self.aristas),
            "historial_top5": sorted(historial, key=lambda x: -x["energia"])[:5],
        }


def demo_qaoa():
    print("\n" + "=" * 60)
    print("DEMO 2 — QAOA para MaxCut")
    print("=" * 60)

    # Problema MaxCut: grafo con 4 nodos
    # 0-1-2-3-0 (ciclo) + diagonal 1-3
    aristas = [(0,1), (1,2), (2,3), (3,0), (1,3)]
    n_nodos = 4

    print(f"Grafo: {n_nodos} nodos, {len(aristas)} aristas: {aristas}")
    print("Buscando partición óptima (MaxCut)...")

    qaoa = CircuitoQAOA(n_nodos, aristas, p_capas=1)
    resultado = qaoa.optimizar(n_iter=16)

    print(f"\nSolución QAOA:")
    print(f"  Bits: {resultado['solucion_bits']}")
    print(f"  Aristas cortadas: {resultado['aristas_cortadas']}/{resultado['max_posible']}")
    print(f"  Energía: {resultado['mejor_energia']}")
    print(f"  Params: γ={resultado['mejores_params']['gamma']}, β={resultado['mejores_params']['beta']}")

    # Solución exacta (fuerza bruta para comparar)
    mejor_exacto = max(
        range(2**n_nodos),
        key=lambda i: energia_maxcut([(i>>q)&1 for q in range(n_nodos)], aristas)
    )
    bits_exacto = [(mejor_exacto>>q)&1 for q in range(n_nodos)]
    print(f"\nSolución exacta (brute force): {bits_exacto}, aristas={int(energia_maxcut(bits_exacto, aristas))}")

    with open("outputs/m35_qaoa.json", "w", encoding="utf-8") as f:
        json.dump(resultado, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m35_qaoa.json")
    return resultado


if __name__ == "__main__":
    demo_qaoa()
    print("\n[M35 completado]")
```

---

## Checkpoint

```python
# checkpoint_m35.py — 6 tests para Quantum + IA

import math, cmath, random
from typing import Dict, List, Tuple
from dataclasses import dataclass


@dataclass
class Qubit:
    alpha: complex = complex(1, 0); beta: complex = complex(0, 0)
    def __post_init__(self):
        norm = math.sqrt(abs(self.alpha)**2 + abs(self.beta)**2)
        if norm > 0: self.alpha /= norm; self.beta /= norm
    @property
    def prob_0(self): return abs(self.alpha)**2
    @property
    def prob_1(self): return abs(self.beta)**2
    def medir(self):
        if random.random() < self.prob_0: self.alpha,self.beta = complex(1),complex(0); return 0
        else: self.alpha,self.beta = complex(0),complex(1); return 1

class PuertaCuantica:
    def __init__(self, nombre, matriz): self.nombre = nombre; self.matriz = matriz
    def aplicar(self, qubit):
        a,b = self.matriz[0]; c,d = self.matriz[1]
        return Qubit(a*qubit.alpha + b*qubit.beta, c*qubit.alpha + d*qubit.beta)
    @staticmethod
    def hadamard():
        c = 1/math.sqrt(2)
        return PuertaCuantica("H", [[complex(c),complex(c)],[complex(c),complex(-c)]])
    @staticmethod
    def pauli_x():
        return PuertaCuantica("X", [[complex(0),complex(1)],[complex(1),complex(0)]])

def energia_maxcut(bits, aristas):
    return sum(1 for u,v in aristas if bits[u] != bits[v])


def test_01_qubit_estado_inicial():
    """Qubit inicial es |0⟩ con prob_0=1, prob_1=0."""
    q = Qubit()
    assert abs(q.prob_0 - 1.0) < 1e-9
    assert abs(q.prob_1 - 0.0) < 1e-9
    print("✓ test_01 — Qubit estado inicial OK")

def test_02_hadamard_superposicion():
    """Puerta H crea superposición 50/50."""
    H = PuertaCuantica.hadamard()
    q = H.aplicar(Qubit())
    assert abs(q.prob_0 - 0.5) < 1e-9
    assert abs(q.prob_1 - 0.5) < 1e-9
    print("✓ test_02 — Hadamard superposición OK")

def test_03_pauli_x_flip():
    """Puerta X invierte |0⟩ a |1⟩."""
    X = PuertaCuantica.pauli_x()
    q = X.aplicar(Qubit())  # |0⟩ → |1⟩
    assert abs(q.prob_0 - 0.0) < 1e-9
    assert abs(q.prob_1 - 1.0) < 1e-9
    print("✓ test_03 — Pauli-X flip OK")

def test_04_normalizacion_qubit():
    """Qubit siempre normalizado: |α|² + |β|² = 1."""
    q = Qubit(complex(3, 0), complex(4, 0))  # no normalizado
    assert abs(q.prob_0 + q.prob_1 - 1.0) < 1e-9
    # Verificar: 3/5 y 4/5
    assert abs(q.prob_0 - 0.36) < 1e-9  # (3/5)² = 9/25
    assert abs(q.prob_1 - 0.64) < 1e-9  # (4/5)² = 16/25
    print("✓ test_04 — Normalización qubit OK")

def test_05_energia_maxcut():
    """energia_maxcut cuenta correctamente aristas cortadas."""
    aristas = [(0,1), (1,2), (2,3), (3,0)]
    # Partición alternada: [0,1,0,1] corta las 4 aristas
    bits_opt = [0, 1, 0, 1]
    assert energia_maxcut(bits_opt, aristas) == 4
    # Todos iguales: 0 aristas cortadas
    bits_nul = [0, 0, 0, 0]
    assert energia_maxcut(bits_nul, aristas) == 0
    print("✓ test_05 — Energía MaxCut OK")

def test_06_medicion_colapsa_estado():
    """La medición colapsa la superposición a 0 ó 1."""
    H = PuertaCuantica.hadamard()
    q = H.aplicar(Qubit())
    assert abs(q.prob_0 - 0.5) < 1e-9  # antes de medir: 50/50
    resultado = q.medir()
    assert resultado in [0, 1]
    # Tras medir, el estado colapsa
    assert abs(q.prob_0 + q.prob_1 - 1.0) < 1e-9
    if resultado == 0:
        assert abs(q.prob_0 - 1.0) < 1e-9
    else:
        assert abs(q.prob_1 - 1.0) < 1e-9
    print("✓ test_06 — Colapso por medición OK")


if __name__ == "__main__":
    tests = [
        test_01_qubit_estado_inicial,
        test_02_hadamard_superposicion,
        test_03_pauli_x_flip,
        test_04_normalizacion_qubit,
        test_05_energia_maxcut,
        test_06_medicion_colapsa_estado,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M35 completado")
```

---

## Resumen

| Concepto | Clásico | Cuántico |
|---|---|---|
| **Bit de información** | 0 ó 1 | α\|0⟩ + β\|1⟩ (superposición) |
| **Estado de N bits** | 2^N posibles, 1 a la vez | 2^N amplitudes simultáneas |
| **Operación** | Lógica booleana (AND, OR) | Puertas unitarias (H, CNOT, Rx) |
| **Lectura** | Sin efecto | Colapsa el estado |
| **Correlación** | Independiente | Entrelazamiento cuántico |
| **Ventaja** | Todos los problemas actuales | Optimización, simulación, cripto |

```python
# Mapa de aplicaciones IA + Cuántica:
APLICACIONES = {
    "QAOA": "Optimización combinatoria (feature selection, scheduling)",
    "VQE": "Simulación química (diseño de fármacos y materiales)",
    "HHL": "Álgebra lineal cuántica (regresión, PageRank)",
    "QML_kernels": "Clasificación en espacio de Hilbert de alta dimensión",
    "Quantum_sampling": "Modelos generativos, muestreo de distribuciones",
}

# Limitaciones actuales (era NISQ, 2024-2025):
LIMITACIONES = {
    "decoherencia": "Los qubits pierden su estado cuántico en microsegundos",
    "error_rate": "Tasa de error ~0.1-1% por puerta (clásico: 1e-15)",
    "conectividad": "No todos los qubits están conectados entre sí",
    "escala": "~1000 qubits actuales vs ~10^6 necesarios para ventaja real",
    "temperatura": "Requiere ~15 milikelvin (más frío que el espacio exterior)",
}
```
