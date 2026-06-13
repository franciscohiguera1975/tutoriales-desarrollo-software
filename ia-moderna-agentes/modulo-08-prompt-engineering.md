# Módulo 08 — Prompt Engineering: Zero-shot, Few-shot, CoT, ToT

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Dominar las técnicas de prompt engineering para extraer el máximo rendimiento de los LLMs: zero-shot, few-shot, Chain-of-Thought, Tree of Thoughts, ReAct y patrones avanzados.
**Herramientas:** Python, API de Anthropic/OpenAI (o llamadas simuladas)
**Prerequisito:** M06 (Arquitectura LLMs), M07 (Pre-entrenamiento)

---

## 1. Fundamentos: Anatomía de un Prompt

```python
# seccion_01_anatomia_prompt.py
"""
Un prompt bien construido tiene estructura clara.
Los LLMs de instrucciones (instruction-tuned) responden mejor
a prompts explícitos que definen rol, contexto, tarea y formato.
"""

# Componentes de un prompt efectivo
componentes = {
    "sistema": "Define el rol y comportamiento del modelo",
    "contexto": "Información de fondo necesaria para la tarea",
    "instrucción": "La tarea específica que debe realizar",
    "ejemplos": "Demostraciones del formato de entrada-salida (few-shot)",
    "entrada": "Los datos concretos sobre los que trabajar",
    "formato": "Cómo debe presentarse la respuesta",
    "restricciones": "Qué no debe hacer el modelo",
}

print("Anatomía de un prompt efectivo:")
for comp, desc in componentes.items():
    print(f"  [{comp.upper()}]: {desc}")

print()

# Ejemplo concreto de prompt estructurado
prompt_clasico = """[SISTEMA]
Eres un experto en análisis de sentimiento con 10 años de experiencia
en NLP. Clasificas textos de forma precisa y justificada.

[CONTEXTO]
Analizamos reseñas de productos de e-commerce en español para
detectar satisfacción del cliente.

[INSTRUCCIÓN]
Clasifica el sentimiento de la siguiente reseña.
Responde ÚNICAMENTE en el formato indicado.

[FORMATO DE RESPUESTA]
Sentimiento: [POSITIVO/NEGATIVO/NEUTRO]
Confianza: [ALTA/MEDIA/BAJA]
Razón: [máximo 15 palabras]

[RESEÑA]
{reseña}"""

print("Prompt estructurado:")
print(prompt_clasico.format(reseña="El producto llegó en perfectas condiciones y funciona mejor de lo esperado."))
```

---

## 2. Zero-Shot vs Few-Shot

```python
# seccion_02_zero_few_shot.py
"""
Zero-shot: el modelo resuelve la tarea sin ejemplos.
Few-shot:  el modelo aprende el formato/tarea de K ejemplos en el contexto.

Few-shot es especialmente útil cuando:
  - El formato de salida es no estándar
  - La tarea tiene definición ambigua
  - Necesitamos consistencia de estilo
  - K ejemplos = contexto de aprendizaje en tiempo de inferencia
"""

# Simulador de LLM para demos (respuestas hardcodeadas para el ejemplo)
class LLMSimulador:
    """
    Simula llamadas a un LLM para demostrar las técnicas de prompting.
    En producción, reemplazar con llamadas reales a la API.
    """
    def __init__(self, modelo="claude-3-5-sonnet"):
        self.modelo = modelo
        self.llamadas = 0

    def completar(self, prompt, max_tokens=200, temperatura=0.0):
        """Simulación: en producción usar anthropic.Anthropic().messages.create(...)"""
        self.llamadas += 1
        # Respuestas simuladas según palabras clave
        if "sentimiento" in prompt.lower() and "excelente" in prompt.lower():
            return "Sentimiento: POSITIVO\nConfianza: ALTA\nRazón: Lenguaje muy positivo y entusiasta"
        elif "extrae" in prompt.lower() and "json" in prompt.lower():
            return '{"nombre": "Juan García", "edad": 34, "ciudad": "Madrid"}'
        elif "traduce" in prompt.lower():
            return "The artificial intelligence is transforming the world rapidly."
        elif "paso a paso" in prompt.lower() or "chain of thought" in prompt.lower():
            return "Paso 1: Identificar lo conocido...\nPaso 2: Aplicar la fórmula...\nPaso 3: Calcular...\nRespuesta: 42"
        return "[Respuesta simulada del LLM]"

llm = LLMSimulador()

# Zero-shot
prompt_zero_shot = """Clasifica el sentimiento de esta reseña:
"El producto es excelente, lo recomiendo totalmente"

Responde con: POSITIVO, NEGATIVO o NEUTRO"""

print("=== ZERO-SHOT ===")
print(f"Prompt:\n{prompt_zero_shot}")
print(f"\nRespuesta: {llm.completar(prompt_zero_shot)}\n")

# Few-shot (3 ejemplos)
prompt_few_shot = """Clasifica el sentimiento de reseñas de productos.

Ejemplo 1:
Reseña: "Me encantó, llegó rápido y funciona perfecto"
Sentimiento: POSITIVO

Ejemplo 2:
Reseña: "Roto desde el primer día, una pérdida de dinero"
Sentimiento: NEGATIVO

Ejemplo 3:
Reseña: "Es lo que pedí, ni más ni menos"
Sentimiento: NEUTRO

Ahora clasifica:
Reseña: "El producto es excelente, lo recomiendo totalmente"
Sentimiento:"""

print("=== FEW-SHOT (3 ejemplos) ===")
print(f"Prompt:\n{prompt_few_shot}")
print(f"\nRespuesta: {llm.completar(prompt_few_shot)}\n")

# Cuándo usar cada uno
print("Comparativa Zero-shot vs Few-shot:")
print(f"{'Aspecto':<25} {'Zero-shot':>15} {'Few-shot':>15}")
print("-" * 57)
aspectos = [
    ("Tokens usados",       "Mínimos",     "Más (K×ejemplos)"),
    ("Consistencia",        "Variable",    "Alta"),
    ("Formato no estándar", "Arriesgado",  "Confiable"),
    ("Tareas comunes",      "Excelente",   "Bueno"),
    ("Tareas específicas",  "Regular",     "Mejor"),
    ("Velocidad",           "Más rápido",  "Más lento"),
]
for asp, zero, few in aspectos:
    print(f"{asp:<25} {zero:>15} {few:>15}")
```

---

## 3. Chain-of-Thought (CoT)

```python
# seccion_03_chain_of_thought.py
"""
Chain-of-Thought (Wei et al. 2022):
  Añadir "Piensa paso a paso" o mostrar razonamiento en los ejemplos
  mejora drásticamente el rendimiento en tareas de razonamiento.

Variantes:
  - Zero-shot CoT: "Let's think step by step" / "Piensa paso a paso"
  - Few-shot CoT: ejemplos con razonamiento explícito
  - Self-consistency CoT: generar múltiples razonamientos → votar
"""

# Zero-shot CoT
def construir_cot_zero_shot(problema):
    return f"""{problema}

Piensa paso a paso antes de dar la respuesta final.
Al final, escribe "Respuesta final: X" """

# Few-shot CoT
def construir_cot_few_shot(problema):
    return f"""Resuelve los siguientes problemas de matemáticas paso a paso.

Problema: María tiene 12 manzanas. Da 3 a su hermana y compra el doble de las que le quedan.
¿Cuántas tiene ahora?
Razonamiento:
- Empieza con 12 manzanas
- Da 3: 12 - 3 = 9 manzanas
- Compra el doble de las que tiene: 9 × 2 = 18 manzanas nuevas
- Total: 9 + 18 = 27 manzanas
Respuesta final: 27

Problema: Un tren va a 80 km/h. En 2.5 horas, ¿cuántos kilómetros recorre?
Razonamiento:
- Velocidad = 80 km/h
- Tiempo = 2.5 h
- Distancia = velocidad × tiempo = 80 × 2.5 = 200 km
Respuesta final: 200

Problema: {problema}
Razonamiento:"""

# Self-consistency CoT
def self_consistency_cot(problema, llm, n_muestras=5, temperatura=0.7):
    """
    Genera N razonamientos con temperatura alta, luego vota la respuesta más frecuente.
    Más robusto que un solo razonamiento.
    """
    respuestas = []
    prompt = construir_cot_few_shot(problema)

    print(f"Generando {n_muestras} razonamientos para self-consistency...")
    for i in range(n_muestras):
        r = llm.completar(prompt + " paso a paso", max_tokens=200, temperatura=temperatura)
        # Extraer respuesta final (simulado)
        respuestas.append(42 + (i % 2))  # simulación de varianza

    from collections import Counter
    votos = Counter(respuestas)
    ganadora = votos.most_common(1)[0][0]
    confianza = votos[ganadora] / n_muestras

    print(f"Votos: {dict(votos)}")
    print(f"Respuesta por mayoría: {ganadora} (confianza: {confianza:.0%})")
    return ganadora

# Demos
llm = LLMSimulador()
problema = "Juan tiene 15 euros. Gasta 4 en un café y recibe el doble de lo que le queda como regalo. ¿Cuánto tiene?"

print("=== Zero-Shot CoT ===")
print(construir_cot_zero_shot(problema))
print()
print("=== Few-Shot CoT ===")
print(construir_cot_few_shot(problema))
print()
print("=== Self-Consistency ===")
result = self_consistency_cot(problema, llm)
```

---

## 4. Tree of Thoughts (ToT)

```python
# seccion_04_tree_of_thoughts.py
"""
Tree of Thoughts (Yao et al. 2023):
  - Extiende CoT de cadena lineal → árbol de razonamientos
  - Explora múltiples caminos, evalúa cada uno, retrocede si es un camino muerto
  - Algoritmos: BFS, DFS, beam search

Ideal para:
  - Juegos (24 game, crossword)
  - Planificación con restricciones
  - Problemas combinatorios
  - Debugging complejo

Componentes:
  1. Generator: propone pensamientos/pasos candidatos
  2. Evaluator: evalúa si un estado es prometedor (sure/maybe/impossible)
  3. Search: BFS o DFS para explorar el árbol
"""
from typing import List, Tuple
import random

class NodoPensamiento:
    def __init__(self, estado: str, padre=None, profundidad=0):
        self.estado     = estado
        self.padre      = padre
        self.profundidad = profundidad
        self.valor      = 0.0   # evaluación del evaluador
        self.hijos      = []

    def __repr__(self):
        return f"Nodo('{self.estado[:30]}...', val={self.valor:.2f})"

class TreeOfThoughts:
    """
    ToT simplificado para un problema de planificación.
    Ejemplo: generar un plan de 3 pasos para resolver un problema.
    """
    def __init__(self, problema: str, llm, max_ramas=3, max_profundidad=3):
        self.problema      = problema
        self.llm           = llm
        self.max_ramas     = max_ramas
        self.max_prof      = max_profundidad

    def generar_pensamientos(self, estado_actual: str) -> List[str]:
        """
        Pide al LLM k pensamientos/pasos candidatos para el estado actual.
        """
        prompt = f"""Problema: {self.problema}

Estado actual del razonamiento:
{estado_actual}

Propón {self.max_ramas} pensamientos/pasos diferentes que podrían seguir
(uno por línea, comenzando con 'Opción X:'):"""

        # Simulación de respuestas
        opciones = [
            f"Opción 1: Analizar los datos disponibles y establecer prioridades",
            f"Opción 2: Buscar patrones en el problema original",
            f"Opción 3: Descomponer en subproblemas más pequeños",
        ]
        return opciones

    def evaluar_pensamiento(self, estado: str) -> float:
        """
        Evalúa si el estado es prometedor.
        Retorna: 1.0 (seguro), 0.5 (quizás), 0.0 (imposible)
        """
        prompt = f"""Evalúa si este estado de razonamiento lleva a una buena solución
para el problema: {self.problema}

Estado: {estado}

Responde con: sure (1.0), maybe (0.5) o impossible (0.0)"""

        # Simulación
        return random.choice([0.5, 0.5, 1.0, 0.5])

    def bfs(self) -> str:
        """BFS sobre el árbol de pensamientos."""
        raiz  = NodoPensamiento("Inicio del razonamiento", profundidad=0)
        cola  = [raiz]
        mejor = raiz
        mejor_val = 0.0

        while cola:
            nodo = cola.pop(0)
            if nodo.profundidad >= self.max_prof:
                continue

            pensamientos = self.generar_pensamientos(nodo.estado)
            for p in pensamientos:
                nuevo_estado = nodo.estado + "\n" + p
                hijo = NodoPensamiento(nuevo_estado, padre=nodo,
                                        profundidad=nodo.profundidad+1)
                hijo.valor = self.evaluar_pensamiento(nuevo_estado)
                nodo.hijos.append(hijo)

                if hijo.valor > mejor_val:
                    mejor = hijo; mejor_val = hijo.valor

                if hijo.valor > 0.3:  # solo explorar ramas prometedoras
                    cola.append(hijo)

        return mejor.estado

# Demo ToT
class LLMSimulador:
    def completar(self, prompt, **kwargs):
        return "[respuesta simulada]"

problema = "Diseña una estrategia de 3 pasos para implementar RAG en producción"
tot = TreeOfThoughts(problema, LLMSimulador(), max_ramas=3, max_profundidad=2)
mejor_plan = tot.bfs()
print("=== Tree of Thoughts (BFS) ===")
print(f"Problema: {problema}")
print(f"\nMejor plan encontrado:\n{mejor_plan}")
```

---

## 5. Patrones Avanzados de Prompting

```python
# seccion_05_patrones_avanzados.py
"""
Técnicas avanzadas de prompt engineering.
"""

# 1. Role Prompting + Persona
def prompt_persona(tarea, experto="desarrollador senior de Python con 15 años de experiencia"):
    return f"""Eres {experto}.
Tu objetivo es: {tarea}

Responde con la profundidad y precisión que tu experiencia permite.
Menciona trade-offs y mejores prácticas cuando sea relevante."""

# 2. Structured Output Prompting
def prompt_json_estructurado(texto, schema):
    return f"""Extrae la información del siguiente texto y devuelve EXCLUSIVAMENTE un JSON
válido que siga este schema:

Schema:
{schema}

Texto:
{texto}

JSON (sin markdown, solo el objeto JSON):"""

ejemplo_json = prompt_json_estructurado(
    texto="Juan García, 34 años, trabaja en Madrid como ingeniero de software.",
    schema='{"nombre": str, "edad": int, "ciudad": str, "profesion": str}'
)
print("=== Structured Output ===")
print(ejemplo_json[:200])

# 3. Chain Prompting (encadenar prompts)
def pipeline_analisis(documento, llm):
    """Divide una tarea compleja en pasos encadenados."""

    # Paso 1: Extraer información clave
    p1 = f"Extrae los 5 puntos clave de este documento en bullet points:\n\n{documento}"
    puntos_clave = llm.completar(p1)

    # Paso 2: Clasificar por categoría
    p2 = f"Clasifica cada punto en: TÉCNICO, NEGOCIO o RIESGO:\n\n{puntos_clave}"
    clasificados = llm.completar(p2)

    # Paso 3: Priorizar
    p3 = f"Ordena los puntos por prioridad (1=alta, 3=baja):\n\n{clasificados}"
    priorizado = llm.completar(p3)

    return {"puntos": puntos_clave, "clasificados": clasificados, "priorizado": priorizado}

# 4. Least-to-Most Prompting (de más fácil a más difícil)
def least_to_most(problema_complejo):
    return f"""Para resolver el siguiente problema complejo, primero vamos a identificar
y resolver los subproblemas más simples.

Problema: {problema_complejo}

Paso 1: ¿Cuáles son los subproblemas más simples que necesitamos resolver primero?
Paso 2: Resuelve cada subproblema por orden de complejidad.
Paso 3: Combina las soluciones para resolver el problema completo."""

# 5. Metacognitive Prompting (pedir al modelo que evalúe su propia respuesta)
def metacognitive_prompt(pregunta):
    return f"""Responde la siguiente pregunta y luego evalúa tu respuesta.

Pregunta: {pregunta}

Respuesta:
[Tu respuesta aquí]

Autoevaluación:
- Confianza (1-10): ?
- Posibles errores o limitaciones: ?
- Qué información adicional mejoraría la respuesta: ?"""

# 6. Negative Prompting (decir qué NO hacer)
def negative_prompt(tarea, prohibiciones):
    return f"""Tarea: {tarea}

PROHIBIDO:
{chr(10).join(f'- {p}' for p in prohibiciones)}

Ahora resuelve la tarea respetando todas las restricciones anteriores."""

prohibiciones = [
    "No uses jerga técnica sin explicarla",
    "No asumas conocimiento previo del lector",
    "No excedas 200 palabras",
    "No des una lista de más de 5 puntos",
]
print("\n=== Negative Prompting ===")
print(negative_prompt("Explica qué es un Transformer", prohibiciones))
```

---

## 6. ReAct — Reasoning + Acting

```python
# seccion_06_react.py
"""
ReAct (Yao et al. 2023): combina razonamiento + acciones.

Ciclo ReAct:
  Pensamiento → Acción → Observación → Pensamiento → Acción → ...

Diferencia con CoT:
  CoT: solo razona internamente (sin acceso al mundo)
  ReAct: puede consultar herramientas externas (search, calculadora, API)
         → reduce alucinaciones al verificar con hechos reales

Formato:
  Thought: [razonamiento interno]
  Action: [herramienta]([argumento])
  Observation: [resultado de la herramienta]
  ... (repetir hasta llegar a la respuesta)
  Final Answer: [respuesta]
"""

# Herramientas disponibles para el agente
class Herramientas:
    @staticmethod
    def buscar(query: str) -> str:
        """Simula una búsqueda web."""
        base = {
            "LLaMA 3 parámetros":  "LLaMA-3 tiene versiones de 8B y 70B parámetros.",
            "GPT-4 fecha":         "GPT-4 fue lanzado el 14 de marzo de 2023.",
            "Python versión":      "La versión actual de Python es 3.12 (2024).",
        }
        for k, v in base.items():
            if any(w in query.lower() for w in k.lower().split()):
                return v
        return f"No se encontró información sobre: {query}"

    @staticmethod
    def calcular(expresion: str) -> str:
        """Evalúa expresión matemática simple."""
        try:
            # Evitar eval peligroso; solo operaciones básicas
            import re
            if re.match(r'^[\d\s\+\-\*\/\.\(\)]+$', expresion):
                return str(eval(expresion))
            return "Error: expresión no válida"
        except:
            return "Error al calcular"

    @staticmethod
    def listar_herramientas():
        return ["buscar(query)", "calcular(expresion)"]

# Agente ReAct simulado
def react_agente(pregunta: str, herramientas: Herramientas, max_pasos=5):
    """
    Simula un agente ReAct que alterna entre razonamiento y acción.
    En producción, el 'pensamiento' viene del LLM.
    """
    print(f"Pregunta: {pregunta}\n")
    historial = f"Pregunta: {pregunta}\n"

    # Simulación hardcodeada para demo
    pasos = [
        ("Pensamiento", "Necesito saber cuántos parámetros tiene LLaMA-3 para calcular el tamaño en GB con FP16."),
        ("Acción",      "buscar(LLaMA 3 parámetros)"),
        ("Observación", herramientas.buscar("LLaMA 3 parámetros")),
        ("Pensamiento", "LLaMA-3 70B tiene 70 billones de parámetros. En FP16: 70B × 2 bytes = 140 GB."),
        ("Acción",      "calcular(70 * 2)"),
        ("Observación", herramientas.calcular("70 * 2")),
        ("Pensamiento", "140 GB. Pero con quantización int4 sería 70B × 0.5 bytes = 35 GB."),
        ("Respuesta Final", "LLaMA-3 70B requiere ~140 GB en FP16 o ~35 GB con quantización int4."),
    ]

    for tipo, contenido in pasos:
        print(f"{tipo}: {contenido}")
        historial += f"\n{tipo}: {contenido}"
        if tipo == "Respuesta Final":
            break

    return historial

herramientas = Herramientas()
react_agente(
    "¿Cuántos GB de VRAM necesito para correr LLaMA-3 70B?",
    herramientas
)
```

---

## 7. Evaluación y Optimización de Prompts

```python
# seccion_07_evaluacion_prompts.py
"""
Cómo evaluar y mejorar prompts sistemáticamente.
"""
import random

def evaluar_prompt(prompt_fn, casos_prueba, llm, metrica="exacta"):
    """
    Evalúa un prompt en múltiples casos de prueba.

    prompt_fn: función que genera el prompt dado el input
    casos_prueba: lista de (input, output_esperado)
    metrica: "exacta", "contiene", "semantica"
    """
    correctas = 0
    resultados = []

    for input_text, esperado in casos_prueba:
        prompt   = prompt_fn(input_text)
        respuesta = llm.completar(prompt)

        if metrica == "exacta":
            correcto = respuesta.strip().lower() == esperado.strip().lower()
        elif metrica == "contiene":
            correcto = esperado.lower() in respuesta.lower()
        else:
            # Métrica semántica: ver M29 (RAGAS)
            correcto = True  # placeholder

        correctas += correcto
        resultados.append({"input": input_text, "esperado": esperado,
                           "respuesta": respuesta, "correcto": correcto})

    accuracy = correctas / len(casos_prueba)
    return accuracy, resultados

# Casos de prueba para clasificación de sentimiento
casos = [
    ("Excelente producto, muy recomendado",       "POSITIVO"),
    ("Llegó roto y sin instrucciones",             "NEGATIVO"),
    ("Es lo que esperaba, correcto",               "NEUTRO"),
    ("Increíble calidad, superó mis expectativas", "POSITIVO"),
    ("Malo, no funciona",                          "NEGATIVO"),
]

# Simular LLM con algo de ruido
class LLMConRuido:
    def completar(self, prompt, **kwargs):
        # 80% de acierto simulado
        textos_positivos = ["excelente", "increíble", "recomendado", "superó"]
        textos_negativos = ["roto", "malo", "no funciona"]
        p = prompt.lower()
        if any(t in p for t in textos_positivos): return "POSITIVO"
        if any(t in p for t in textos_negativos): return "NEGATIVO"
        return random.choice(["NEUTRO", "POSITIVO", "NEGATIVO"])

llm_con_ruido = LLMConRuido()

# Comparar prompt base vs mejorado
def prompt_base(texto):
    return f"Clasifica el sentimiento: {texto}\nResponde: POSITIVO, NEGATIVO o NEUTRO"

def prompt_mejorado(texto):
    return f"""Eres un experto en análisis de sentimiento.
Clasifica el sentimiento de la siguiente reseña de producto.

Directrices:
- POSITIVO: elogia el producto, expresa satisfacción
- NEGATIVO: critica el producto, expresa insatisfacción
- NEUTRO: descripción objetiva sin emoción clara

Reseña: {texto}

Responde únicamente con una de estas palabras: POSITIVO, NEGATIVO o NEUTRO"""

acc_base, _ = evaluar_prompt(prompt_base, casos, llm_con_ruido, "exacta")
acc_mejor, _ = evaluar_prompt(prompt_mejorado, casos, llm_con_ruido, "exacta")

print("Evaluación de prompts:")
print(f"  Prompt base:     {acc_base:.0%} accuracy")
print(f"  Prompt mejorado: {acc_mejor:.0%} accuracy")

# Técnicas de mejora sistemática
print("""
Proceso de optimización de prompts:
  1. Línea base: medir accuracy con prompt mínimo
  2. Análisis de errores: ¿en qué casos falla?
  3. Hipótesis: ¿qué cambio podría ayudar?
  4. A/B test: evaluar en conjunto de validación
  5. Iterar: repetir con el mejor prompt

Herramientas automáticas:
  - DSPy: optimización automática de prompts con gradiente
  - TextGrad: gradientes textuales para optimizar prompts
  - PromptBreeder: evolución automática de prompts
""")
```

---

## Checkpoint M08

```python
# checkpoint_m08.py
"""
Tests de verificación para Módulo 08 — Prompt Engineering.
Ejecutar con: python checkpoint_m08.py
"""

def test_1_zero_shot_tiene_tarea():
    """Un prompt zero-shot contiene la tarea explícita."""
    tarea = "Clasifica el sentimiento"
    prompt = f"{tarea}: 'Me encantó el producto'"
    assert tarea.lower() in prompt.lower()
    print("✓ Test 1: Prompt zero-shot contiene la tarea")

def test_2_few_shot_tiene_ejemplos():
    """Un prompt few-shot tiene al menos K ejemplos."""
    ejemplos = [
        ("Excelente producto", "POSITIVO"),
        ("Muy malo", "NEGATIVO"),
        ("Está bien", "NEUTRO"),
    ]
    prompt_lines = []
    for inp, out in ejemplos:
        prompt_lines.append(f"Reseña: {inp}")
        prompt_lines.append(f"Sentimiento: {out}")
    prompt = "\n".join(prompt_lines) + "\nReseña: [nueva reseña]\nSentimiento:"
    assert prompt.count("Sentimiento:") == len(ejemplos) + 1
    print(f"✓ Test 2: Few-shot prompt tiene {len(ejemplos)} ejemplos")

def test_3_cot_incluye_razonamiento():
    """Prompt CoT incluye la instrucción de razonar paso a paso."""
    def cot_prompt(problema):
        return f"{problema}\n\nPiensa paso a paso antes de dar la respuesta final."
    p = cot_prompt("¿Cuánto es 15 × 8?")
    assert "paso a paso" in p.lower() or "step by step" in p.lower()
    print("✓ Test 3: CoT prompt incluye instrucción de razonamiento")

def test_4_self_consistency_vota():
    """Self-consistency elige la respuesta más frecuente."""
    from collections import Counter
    respuestas = [42, 42, 43, 42, 43]
    votos = Counter(respuestas)
    ganadora = votos.most_common(1)[0][0]
    assert ganadora == 42
    print(f"✓ Test 4: Self-consistency elige por mayoría: {ganadora}")

def test_5_react_alterna_pensamiento_accion():
    """Formato ReAct alterna Thought/Action/Observation."""
    trace = [
        "Thought: necesito buscar información",
        "Action: buscar(query)",
        "Observation: resultado de la búsqueda",
        "Thought: ahora puedo responder",
        "Final Answer: la respuesta",
    ]
    tipos = [t.split(":")[0] for t in trace]
    assert tipos[0] == "Thought"
    assert tipos[1] == "Action"
    assert tipos[2] == "Observation"
    assert tipos[-1] == "Final Answer"
    print(f"✓ Test 5: ReAct sigue el formato T→A→O→...→FA")

def test_6_prompt_negativo_incluye_restricciones():
    """Negative prompting incluye lista de restricciones."""
    restricciones = ["no uses jerga", "máximo 100 palabras"]
    prompt = "Tarea: explica IA\n\nPROHIBIDO:\n"
    for r in restricciones:
        prompt += f"- {r}\n"
    for r in restricciones:
        assert r in prompt
    print(f"✓ Test 6: Negative prompt incluye {len(restricciones)} restricciones")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M08 — Prompt Engineering")
    print("=" * 50)
    test_1_zero_shot_tiene_tarea()
    test_2_few_shot_tiene_ejemplos()
    test_3_cot_incluye_razonamiento()
    test_4_self_consistency_vota()
    test_5_react_alterna_pensamiento_accion()
    test_6_prompt_negativo_incluye_restricciones()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M08 completado")
```

---

## Resumen del Módulo

| Técnica | Cuándo usar | Ventaja | Costo |
|---|---|---|---|
| **Zero-shot** | Tareas comunes, LLMs fuertes | Mínimos tokens | Variable |
| **Few-shot** | Formato no estándar, consistencia | Alta consistencia | K×ejemplo tokens |
| **CoT** | Razonamiento, math, lógica | Mucho mejor en reasoning | +50-100 tokens |
| **Self-consistency** | Alta precisión requerida | Más robusto | N×costo base |
| **ToT** | Problemas con exploración | Soluciones creativas | Alto (árbol de llamadas) |
| **ReAct** | Necesita datos externos | Reduce alucinaciones | Múltiples llamadas |
| **Chain prompting** | Tareas complejas | Modularidad | Múltiples llamadas |

**Próximo módulo:** M09 — Fine-tuning Completo: SFT y RLHF
