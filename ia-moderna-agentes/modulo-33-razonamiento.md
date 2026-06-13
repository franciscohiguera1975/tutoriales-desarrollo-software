# Módulo 33 — Modelos de Razonamiento: o1, DeepSeek-R1, Gemini

> **Objetivo:** Comprender la arquitectura del razonamiento extendido en LLMs: cadenas de pensamiento largas (CoT), búsqueda con MCTS, verificadores de proceso (PRM) y verificadores de resultados (ORM), con implementación de razonamiento iterativo.
>
> **Herramientas:** Python 3.11, math, json, random
>
> **Prerequisito:** M08 (Prompt Engineering), M06 (Arquitectura LLMs)

---

## 1. Cadenas de Pensamiento (Chain-of-Thought)

```python
# modulo-33-razonamiento.py — Parte 1: Chain-of-Thought

"""
Los modelos de razonamiento como o1 y DeepSeek-R1 generan tokens de
"pensamiento" internos antes de la respuesta final. Esto les permite:

1. Descomponer problemas en pasos intermedios
2. Verificar cada paso antes de continuar
3. Retroceder y corregir cuando detectan errores
4. Explorar múltiples caminos de solución

Arquitecturas:
- o1/o3: razonamiento interno oculto, respuesta final limpia
- DeepSeek-R1: razonamiento visible en <think>...</think>
- Gemini Flash Thinking: similar a DeepSeek-R1
"""

import json
import math
import random
import time
import hashlib
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Tuple
from enum import Enum


# ── Paso de razonamiento ──────────────────────────────────────────────────────

@dataclass
class PasoRazonamiento:
    """Un paso individual en la cadena de pensamiento."""
    numero: int
    descripcion: str
    contenido: str
    confianza: float = 1.0   # 0.0-1.0
    es_correcto: Optional[bool] = None
    tiempo_ms: int = 0

    def to_dict(self) -> Dict:
        return {
            "paso": self.numero,
            "descripcion": self.descripcion,
            "contenido": self.contenido[:200],
            "confianza": round(self.confianza, 3),
            "correcto": self.es_correcto,
        }


@dataclass
class CadenaRazonamiento:
    """Secuencia completa de pasos de razonamiento."""
    pregunta: str
    pasos: List[PasoRazonamiento] = field(default_factory=list)
    respuesta_final: str = ""
    tokens_pensamiento: int = 0
    tokens_respuesta: int = 0

    def agregar_paso(self, descripcion: str, contenido: str, confianza: float = 1.0) -> PasoRazonamiento:
        paso = PasoRazonamiento(
            numero=len(self.pasos) + 1,
            descripcion=descripcion,
            contenido=contenido,
            confianza=confianza,
        )
        self.pasos.append(paso)
        self.tokens_pensamiento += len(contenido.split())
        return paso

    def to_dict(self) -> Dict:
        return {
            "pregunta": self.pregunta,
            "pasos": [p.to_dict() for p in self.pasos],
            "respuesta_final": self.respuesta_final,
            "tokens_pensamiento": self.tokens_pensamiento,
            "tokens_respuesta": len(self.respuesta_final.split()),
            "confianza_promedio": sum(p.confianza for p in self.pasos) / max(1, len(self.pasos)),
        }


# ── Razonador matemático (simula CoT de o1) ───────────────────────────────────

class RazonadorMatematico:
    """
    Simula el razonamiento paso a paso para problemas matemáticos.
    Descompone el problema, verifica cada operación, detecta errores.
    """

    def resolver(self, problema: str) -> CadenaRazonamiento:
        cadena = CadenaRazonamiento(pregunta=problema)

        # Detectar tipo de problema
        prob_l = problema.lower()

        if any(k in prob_l for k in ["factorial", "!"]):
            return self._resolver_factorial(problema, cadena)
        if any(k in prob_l for k in ["primo", "prime", "es primo"]):
            return self._resolver_primo(problema, cadena)
        if "fibonacci" in prob_l:
            return self._resolver_fibonacci(problema, cadena)
        if any(op in problema for op in ["+", "-", "*", "/", "^"]):
            return self._resolver_aritmetica(problema, cadena)

        # Problema genérico: descomposición general
        cadena.agregar_paso("Comprensión", "Identifico el tipo de problema y datos dados.", 0.9)
        cadena.agregar_paso("Planificación", "Selecciono la estrategia de resolución más eficiente.", 0.85)
        cadena.agregar_paso("Ejecución", "Aplico la estrategia seleccionada paso a paso.", 0.80)
        cadena.agregar_paso("Verificación", "Compruebo la validez del resultado obtenido.", 0.90)
        cadena.respuesta_final = f"Problema resuelto mediante descomposición sistemática."
        return cadena

    def _resolver_factorial(self, problema: str, cadena: CadenaRazonamiento) -> CadenaRazonamiento:
        # Extraer n
        import re
        nums = re.findall(r'\d+', problema)
        n = int(nums[0]) if nums else 5

        cadena.agregar_paso("Identificación", f"El problema pide calcular {n}! (factorial de {n}).", 1.0)
        cadena.agregar_paso("Definición", "Recuerdo: n! = n × (n-1) × ... × 2 × 1; y 0! = 1.", 1.0)

        # Calcular paso a paso
        resultado = 1
        pasos_calc = []
        for i in range(1, n + 1):
            resultado *= i
            pasos_calc.append(f"{i}! = {resultado}")

        cadena.agregar_paso("Cálculo", " → ".join(pasos_calc[-min(5, len(pasos_calc)):]), 1.0)
        cadena.agregar_paso("Verificación", f"Verifico: {n} × {math.factorial(n)//n} = {math.factorial(n)}", 1.0)
        cadena.respuesta_final = f"{n}! = {math.factorial(n)}"
        return cadena

    def _resolver_primo(self, problema: str, cadena: CadenaRazonamiento) -> CadenaRazonamiento:
        import re
        nums = re.findall(r'\d+', problema)
        n = int(nums[0]) if nums else 17

        cadena.agregar_paso("Identificación", f"Debo determinar si {n} es un número primo.", 1.0)
        cadena.agregar_paso("Criterio", "Un número es primo si solo es divisible por 1 y sí mismo.", 1.0)

        if n < 2:
            cadena.agregar_paso("Caso base", f"{n} < 2, por definición no es primo.", 1.0)
            cadena.respuesta_final = f"{n} NO es primo."
            return cadena

        divisores = []
        for i in range(2, int(math.sqrt(n)) + 1):
            if n % i == 0:
                divisores.append(i)

        if divisores:
            cadena.agregar_paso("Prueba divisibilidad", f"Encontré divisores: {divisores}. {n} = {divisores[0]} × {n//divisores[0]}", 1.0)
            cadena.respuesta_final = f"{n} NO es primo. Divisores: {divisores}"
        else:
            cadena.agregar_paso("Prueba divisibilidad", f"Ningún número de 2 a sqrt({n})={int(math.sqrt(n))} divide a {n}.", 1.0)
            cadena.respuesta_final = f"{n} SÍ es primo."

        return cadena

    def _resolver_fibonacci(self, problema: str, cadena: CadenaRazonamiento) -> CadenaRazonamiento:
        import re
        nums = re.findall(r'\d+', problema)
        n = int(nums[0]) if nums else 10

        cadena.agregar_paso("Identificación", f"Calcular el término {n} de la sucesión Fibonacci.", 1.0)
        cadena.agregar_paso("Definición", "F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2) para n>1.", 1.0)

        a, b = 0, 1
        secuencia = [0, 1]
        for _ in range(n - 1):
            a, b = b, a + b
            secuencia.append(b)

        cadena.agregar_paso("Cálculo iterativo", f"Secuencia: {secuencia[:min(10, len(secuencia))]}", 1.0)
        cadena.agregar_paso("Verificación", f"F({n-1}) + F({n}) = {secuencia[-2]} + {secuencia[-1] if len(secuencia) > 1 else 'N/A'}", 0.95)
        cadena.respuesta_final = f"F({n}) = {secuencia[n] if n < len(secuencia) else b}"
        return cadena

    def _resolver_aritmetica(self, problema: str, cadena: CadenaRazonamiento) -> CadenaRazonamiento:
        cadena.agregar_paso("Identificación", "Problema aritmético con operaciones básicas.", 1.0)
        cadena.agregar_paso("Orden de operaciones", "Aplico PEMDAS: Paréntesis → Exponentes → × ÷ → + −", 1.0)

        try:
            expr_limpia = "".join(c for c in problema if c in "0123456789+-*/(). ")
            if expr_limpia.strip():
                resultado = eval(expr_limpia)
                cadena.agregar_paso("Cálculo", f"Evaluando '{expr_limpia.strip()}' = {resultado}", 1.0)
                cadena.agregar_paso("Verificación", f"El resultado {resultado} es consistente con las operaciones.", 1.0)
                cadena.respuesta_final = f"Resultado: {resultado}"
            else:
                cadena.respuesta_final = "No pude extraer la expresión matemática."
        except Exception as e:
            cadena.agregar_paso("Error detectado", f"Error en la expresión: {e}. Revisando...", 0.5)
            cadena.respuesta_final = "Error al evaluar la expresión."

        return cadena


# ── Demo 1 ────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Chain-of-Thought Matemático")
    print("=" * 60)

    razonador = RazonadorMatematico()
    problemas = [
        "Calcula el factorial de 7",
        "¿Es 97 un número primo?",
        "Calcula Fibonacci 12",
        "¿Cuánto es 15 * 4 + 7 - 3?",
    ]

    resultados = []
    for prob in problemas:
        cadena = razonador.resolver(prob)
        print(f"\nProblema: {prob}")
        print(f"Pasos: {len(cadena.pasos)}, Tokens de pensamiento: {cadena.tokens_pensamiento}")
        print(f"Respuesta: {cadena.respuesta_final}")
        resultados.append(cadena.to_dict())

    with open("outputs/m33_cot.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m33_cot.json")
```

---

## 2. Process Reward Model (PRM) y Outcome Reward Model (ORM)

```python
# Parte 2: PRM y ORM — Verificadores de razonamiento

"""
Los modelos de razonamiento avanzados usan dos tipos de verificadores:

- ORM (Outcome Reward Model): evalúa solo la respuesta final
  Simple pero no detecta errores intermedios.

- PRM (Process Reward Model): evalúa cada paso del razonamiento
  Más robusto: puede guiar la búsqueda hacia soluciones correctas.
  Entrenado con feedback humano en cada paso (no solo en el resultado).
"""


class VerificadorORM:
    """
    Outcome Reward Model: verifica solo si la respuesta final es correcta.
    """

    def verificar(self, pregunta: str, respuesta: str, respuesta_correcta: str) -> Dict:
        """Verifica si la respuesta final coincide con la esperada."""
        # Normalización: extraer números de la respuesta
        import re
        nums_resp = re.findall(r'-?\d+\.?\d*', respuesta)
        nums_corr = re.findall(r'-?\d+\.?\d*', respuesta_correcta)

        # Si ambas tienen números, comparar el último (el resultado)
        if nums_resp and nums_corr:
            es_correcto = nums_resp[-1] == nums_corr[-1]
        else:
            es_correcto = respuesta.strip().lower() == respuesta_correcta.strip().lower()

        return {
            "tipo": "ORM",
            "es_correcto": es_correcto,
            "score": 1.0 if es_correcto else 0.0,
            "respuesta": respuesta,
            "esperada": respuesta_correcta,
        }


class VerificadorPRM:
    """
    Process Reward Model: verifica cada paso del razonamiento.
    Asigna score a cada paso y detecta el primero incorrecto.
    """

    def verificar_paso(self, paso: PasoRazonamiento, contexto: str = "") -> float:
        """
        Evalúa un paso individual.
        En producción: LLM entrenado como verificador de pasos.
        Aquí: heurísticas basadas en contenido del paso.
        """
        score = paso.confianza  # base: confianza declarada

        # Penalizar pasos muy cortos (posiblemente superficiales)
        if len(paso.contenido) < 20:
            score *= 0.7

        # Penalizar si hay palabras de incertidumbre sin justificación
        incertidumbre = ["quizás", "tal vez", "podría", "puede que", "creo que"]
        if any(w in paso.contenido.lower() for w in incertidumbre):
            score *= 0.85

        # Bonificar si el paso incluye cálculos explícitos
        import re
        if re.search(r'\d+\s*[+\-*/=]\s*\d+', paso.contenido):
            score = min(1.0, score * 1.1)

        return round(max(0.0, min(1.0, score)), 3)

    def verificar_cadena(self, cadena: CadenaRazonamiento) -> Dict:
        """Verifica todos los pasos de la cadena y retorna análisis completo."""
        resultados_pasos = []
        primer_error = None

        for paso in cadena.pasos:
            score = self.verificar_paso(paso)
            es_correcto = score >= 0.7

            resultado_paso = {
                "paso": paso.numero,
                "descripcion": paso.descripcion,
                "score_prm": score,
                "es_correcto": es_correcto,
            }
            resultados_pasos.append(resultado_paso)

            if not es_correcto and primer_error is None:
                primer_error = paso.numero

        score_global = sum(r["score_prm"] for r in resultados_pasos) / max(1, len(resultados_pasos))

        return {
            "tipo": "PRM",
            "pasos_verificados": len(resultados_pasos),
            "score_global": round(score_global, 3),
            "primer_error_en_paso": primer_error,
            "es_correcto": primer_error is None,
            "detalle_pasos": resultados_pasos,
        }


# ── Demo 2 ────────────────────────────────────────────────────────────────────

def demo_prm_orm():
    print("\n" + "=" * 60)
    print("DEMO 2 — PRM vs ORM")
    print("=" * 60)

    razonador = RazonadorMatematico()
    orm = VerificadorORM()
    prm = VerificadorPRM()

    casos = [
        ("Calcula el factorial de 5", "5! = 120"),
        ("¿Es 13 un número primo?", "13 SÍ es primo."),
        ("Calcula Fibonacci 8", "F(8) = 21"),
    ]

    resultados = []
    for pregunta, respuesta_correcta in casos:
        cadena = razonador.resolver(pregunta)
        resultado_orm = orm.verificar(pregunta, cadena.respuesta_final, respuesta_correcta)
        resultado_prm = prm.verificar_cadena(cadena)

        print(f"\nProblema: {pregunta}")
        print(f"  ORM: {'✓' if resultado_orm['es_correcto'] else '✗'} (score={resultado_orm['score']})")
        print(f"  PRM: {'✓' if resultado_prm['es_correcto'] else '✗'} (score={resultado_prm['score_global']}, {resultado_prm['pasos_verificados']} pasos)")

        resultados.append({
            "pregunta": pregunta,
            "orm": resultado_orm,
            "prm": resultado_prm,
        })

    with open("outputs/m33_prm_orm.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m33_prm_orm.json")
    return resultados


if __name__ == "__main__":
    demo_prm_orm()
```

---

## 3. Monte Carlo Tree Search (MCTS) para Razonamiento

```python
# Parte 3: MCTS aplicado a búsqueda de razonamiento

"""
o1 y DeepSeek-R1 usan búsqueda tipo MCTS para explorar múltiples
caminos de razonamiento y seleccionar el más prometedor.

MCTS en razonamiento:
- Nodo: estado del razonamiento (pasos dados hasta ahora)
- Acción: siguiente paso a generar
- Recompensa: score del PRM para ese paso
- UCB: equilibra exploración vs explotación
"""


@dataclass
class NodoMCTS:
    """Nodo del árbol MCTS: representa un estado parcial de razonamiento."""
    id: str = field(default_factory=lambda: __import__("uuid").uuid4().hex[:6])
    contenido: str = ""        # pensamiento en este nodo
    padre: Optional["NodoMCTS"] = None
    hijos: List["NodoMCTS"] = field(default_factory=list)
    visitas: int = 0
    valor_acumulado: float = 0.0
    score_prm: float = 0.0    # evaluación del PRM para este nodo

    @property
    def valor_medio(self) -> float:
        return self.valor_acumulado / max(1, self.visitas)

    def ucb(self, c: float = 1.41) -> float:
        """Upper Confidence Bound: equilibra explotación y exploración."""
        if self.visitas == 0:
            return float("inf")
        padre_visitas = self.padre.visitas if self.padre else 1
        return self.valor_medio + c * math.sqrt(math.log(padre_visitas) / self.visitas)

    def es_hoja(self) -> bool:
        return len(self.hijos) == 0

    def profundidad(self) -> int:
        d = 0
        nodo = self
        while nodo.padre:
            d += 1
            nodo = nodo.padre
        return d


class MCTSRazonador:
    """
    Aplica MCTS para explorar múltiples caminos de razonamiento
    y seleccionar el más prometedor (mayor score PRM).
    """

    def __init__(self, max_iter: int = 20, max_profundidad: int = 5, c: float = 1.41):
        self.max_iter = max_iter
        self.max_profundidad = max_profundidad
        self.c = c
        self._razonador = RazonadorMatematico()
        self._prm = VerificadorPRM()
        self._nodos_creados = 0

    def _generar_pensamiento(self, contexto: str, nivel: int) -> str:
        """Genera un pensamiento para el siguiente paso (simulado)."""
        pensamientos = {
            0: ["Identifico el tipo de problema.", "Analizo los datos del enunciado.", "Comprendo qué se pide calcular."],
            1: ["Selecciono la estrategia más eficiente.", "Recuerdo las fórmulas relevantes.", "Divido el problema en subproblemas."],
            2: ["Aplico el método seleccionado.", "Calculo paso a paso.", "Verifico la consistencia de las operaciones."],
            3: ["Compruebo el resultado obtenido.", "Valido con un caso especial.", "Confirmo que la respuesta es coherente."],
            4: ["La solución es correcta y completa.", "Presento el resultado final.", "TERMINATE"],
        }
        opciones = pensamientos.get(nivel, ["Continúo el análisis."])
        seed = int(hashlib.md5(contexto.encode()).hexdigest(), 16) % len(opciones)
        return opciones[seed]

    def _evaluar_nodo(self, nodo: NodoMCTS) -> float:
        """Evalúa la calidad de un nodo con el PRM (simulado)."""
        paso = PasoRazonamiento(
            numero=nodo.profundidad() + 1,
            descripcion="Paso MCTS",
            contenido=nodo.contenido,
            confianza=0.8,
        )
        return self._prm.verificar_paso(paso)

    def buscar(self, pregunta: str) -> Dict:
        """
        Ejecuta MCTS para encontrar la mejor cadena de razonamiento.
        Retorna el camino de mayor valor.
        """
        raiz = NodoMCTS(contenido=f"Pregunta: {pregunta}")
        self._nodos_creados = 1

        for iteracion in range(self.max_iter):
            # 1. SELECCIÓN: recorrer árbol con UCB
            nodo = raiz
            while not nodo.es_hoja() and nodo.profundidad() < self.max_profundidad:
                nodo = max(nodo.hijos, key=lambda n: n.ucb(self.c))

            # 2. EXPANSIÓN: si el nodo no está completamente expandido
            if nodo.profundidad() < self.max_profundidad:
                nuevos_pensamientos = [
                    self._generar_pensamiento(nodo.contenido, nodo.profundidad() + 1)
                    for _ in range(min(3, self.max_profundidad - nodo.profundidad()))
                ]
                for pensamiento in set(nuevos_pensamientos):  # evitar duplicados
                    hijo = NodoMCTS(contenido=pensamiento, padre=nodo)
                    hijo.score_prm = self._evaluar_nodo(hijo)
                    nodo.hijos.append(hijo)
                    self._nodos_creados += 1

                if nodo.hijos:
                    nodo = nodo.hijos[0]

            # 3. SIMULACIÓN: evaluar el nodo seleccionado
            recompensa = nodo.score_prm

            # 4. RETROPROPAGACIÓN
            nodo_actual = nodo
            while nodo_actual is not None:
                nodo_actual.visitas += 1
                nodo_actual.valor_acumulado += recompensa
                nodo_actual = nodo_actual.padre

        # Extraer mejor camino
        mejor_camino = self._extraer_mejor_camino(raiz)
        return {
            "pregunta": pregunta,
            "iteraciones": self.max_iter,
            "nodos_explorados": self._nodos_creados,
            "mejor_camino": [n.contenido for n in mejor_camino],
            "score_mejor": mejor_camino[-1].valor_medio if mejor_camino else 0,
        }

    def _extraer_mejor_camino(self, raiz: NodoMCTS) -> List[NodoMCTS]:
        """Extrae el camino de mayor valor promedio."""
        camino = [raiz]
        nodo = raiz
        while nodo.hijos:
            nodo = max(nodo.hijos, key=lambda n: n.valor_medio)
            camino.append(nodo)
        return camino


def demo_mcts():
    print("\n" + "=" * 60)
    print("DEMO 3 — MCTS para Razonamiento")
    print("=" * 60)

    mcts = MCTSRazonador(max_iter=15, max_profundidad=4)
    problemas = [
        "Calcula el factorial de 6",
        "¿Es 29 un número primo?",
    ]

    resultados = []
    for prob in problemas:
        res = mcts.buscar(prob)
        print(f"\nProblema: {prob}")
        print(f"  Nodos explorados: {res['nodos_explorados']}")
        print(f"  Mejor camino ({len(res['mejor_camino'])} pasos):")
        for i, paso in enumerate(res['mejor_camino'], 1):
            print(f"    {i}. {paso[:60]}")
        print(f"  Score: {res['score_mejor']:.3f}")
        resultados.append(res)

    with open("outputs/m33_mcts.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m33_mcts.json")
    print("\n[M33 completado]")
    return resultados


if __name__ == "__main__":
    demo_mcts()
```

---

## Checkpoint

```python
# checkpoint_m33.py — 6 tests para Modelos de Razonamiento

import math, json, re
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional


@dataclass
class PasoRazonamiento:
    numero: int; descripcion: str; contenido: str
    confianza: float = 1.0; es_correcto: Optional[bool] = None; tiempo_ms: int = 0
    def to_dict(self): return {"paso":self.numero,"descripcion":self.descripcion,"confianza":self.confianza}

@dataclass
class CadenaRazonamiento:
    pregunta: str; pasos: List[PasoRazonamiento] = field(default_factory=list)
    respuesta_final: str = ""; tokens_pensamiento: int = 0
    def agregar_paso(self, desc, cont, conf=1.0):
        p = PasoRazonamiento(len(self.pasos)+1, desc, cont, conf)
        self.pasos.append(p); self.tokens_pensamiento += len(cont.split()); return p
    def to_dict(self): return {"pregunta":self.pregunta,"pasos":[p.to_dict() for p in self.pasos],"respuesta":self.respuesta_final}

class VerificadorORM:
    def verificar(self, pregunta, respuesta, respuesta_correcta):
        nums_r = re.findall(r'-?\d+\.?\d*', respuesta)
        nums_c = re.findall(r'-?\d+\.?\d*', respuesta_correcta)
        es_correcto = (nums_r[-1] == nums_c[-1]) if nums_r and nums_c else respuesta.lower() == respuesta_correcta.lower()
        return {"tipo":"ORM","es_correcto":es_correcto,"score":1.0 if es_correcto else 0.0}

class VerificadorPRM:
    def verificar_paso(self, paso):
        score = paso.confianza
        if len(paso.contenido) < 20: score *= 0.7
        if re.search(r'\d+\s*[+\-*/=]\s*\d+', paso.contenido): score = min(1.0, score * 1.1)
        return round(max(0.0, min(1.0, score)), 3)
    def verificar_cadena(self, cadena):
        scores = [self.verificar_paso(p) for p in cadena.pasos]
        return {"score_global":sum(scores)/max(1,len(scores)),"es_correcto":all(s>=0.7 for s in scores),"pasos":len(scores)}


def test_01_cadena_razonamiento_agregar_pasos():
    """CadenaRazonamiento acumula pasos y tokens."""
    cadena = CadenaRazonamiento("2+2")
    cadena.agregar_paso("Identificación", "Es una suma simple de dos números enteros.", 1.0)
    cadena.agregar_paso("Cálculo", "2 + 2 = 4", 1.0)
    assert len(cadena.pasos) == 2
    assert cadena.pasos[0].numero == 1
    assert cadena.pasos[1].numero == 2
    assert cadena.tokens_pensamiento > 0
    print("✓ test_01 — CadenaRazonamiento OK")

def test_02_orm_respuesta_correcta():
    """ORM reconoce respuesta correcta con número coincidente."""
    orm = VerificadorORM()
    resultado = orm.verificar("factorial 5", "5! = 120", "120")
    assert resultado["es_correcto"] == True
    assert resultado["score"] == 1.0
    print("✓ test_02 — ORM correcta OK")

def test_03_orm_respuesta_incorrecta():
    """ORM detecta respuesta incorrecta."""
    orm = VerificadorORM()
    resultado = orm.verificar("factorial 5", "5! = 60", "120")
    assert resultado["es_correcto"] == False
    assert resultado["score"] == 0.0
    print("✓ test_03 — ORM incorrecta OK")

def test_04_prm_paso_con_calculo_bonificado():
    """PRM bonifica pasos con expresiones matemáticas explícitas."""
    prm = VerificadorPRM()
    paso_con_calc = PasoRazonamiento(1, "Cálculo", "3 * 4 = 12, luego 12 + 5 = 17", confianza=0.9)
    paso_sin_calc = PasoRazonamiento(2, "General", "El resultado es correcto.", confianza=0.9)
    score_calc = prm.verificar_paso(paso_con_calc)
    score_sin = prm.verificar_paso(paso_sin_calc)
    assert score_calc >= score_sin, "Paso con cálculo debe tener score >= que sin cálculo"
    print("✓ test_04 — PRM bonificación cálculo OK")

def test_05_prm_paso_corto_penalizado():
    """PRM penaliza pasos demasiado cortos."""
    prm = VerificadorPRM()
    paso_corto = PasoRazonamiento(1, "Corto", "OK.", confianza=1.0)
    paso_largo = PasoRazonamiento(2, "Largo", "Analizo el problema en detalle y verifico cada operación.", confianza=1.0)
    score_corto = prm.verificar_paso(paso_corto)
    score_largo = prm.verificar_paso(paso_largo)
    assert score_corto < score_largo, "Paso corto debe ser penalizado"
    print("✓ test_05 — PRM penalización corto OK")

def test_06_factorial_matematico():
    """Verificación directa de factoriales."""
    def factorial(n): return 1 if n == 0 else n * factorial(n-1)
    casos = [(0,1),(1,1),(5,120),(7,5040),(10,3628800)]
    for n, esperado in casos:
        assert factorial(n) == esperado, f"factorial({n}) debe ser {esperado}"
    print("✓ test_06 — Factoriales correctos OK")


if __name__ == "__main__":
    tests = [
        test_01_cadena_razonamiento_agregar_pasos,
        test_02_orm_respuesta_correcta,
        test_03_orm_respuesta_incorrecta,
        test_04_prm_paso_con_calculo_bonificado,
        test_05_prm_paso_corto_penalizado,
        test_06_factorial_matematico,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M33 completado")
```

---

## Resumen

| Técnica | Descripción | Ventaja | Costo |
|---|---|---|---|
| **CoT estándar** | Generación lineal de pasos | Simple, efectivo en problemas medianos | Sin backtracking |
| **CoT con verificación** | PRM evalúa cada paso | Detecta errores intermedios | Requiere PRM entrenado |
| **MCTS** | Exploración de múltiples caminos | Encuentra soluciones óptimas | Mucho más tokens |
| **ORM** | Verifica solo resultado final | Rápido, simple de entrenar | No detecta errores intermedios |
| **PRM** | Verifica cada paso | Alta precisión, guía la búsqueda | Costoso de entrenar |

```python
# Comparativa de modelos de razonamiento (2024-2025):
MODELOS_RAZONAMIENTO = {
    "o1": {"tokens_thinking": "ocultos", "metodo": "MCTS+PRM", "benchmarks": {"AIME": "74%", "GPQA": "78%"}},
    "o3": {"tokens_thinking": "ocultos", "metodo": "MCTS+PRM_mejorado", "benchmarks": {"AIME": "96%", "GPQA": "88%"}},
    "DeepSeek-R1": {"tokens_thinking": "visibles <think>", "metodo": "GRPO+PRM", "benchmarks": {"AIME": "72%", "GPQA": "71%"}},
    "Gemini_Flash_Thinking": {"tokens_thinking": "visibles", "metodo": "CoT_extendido", "benchmarks": {"AIME": "68%"}},
}
```
