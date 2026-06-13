# Módulo 16 — Fundamentos de Agentes — ReAct y Tool Use

**Objetivo:** Implementar agentes basados en el paradigma ReAct (Reason + Act): bucle de razonamiento, ejecución de herramientas y observación; construir un framework de agente con tool use desde cero.

**Herramientas:** `numpy`, `anthropic`, `openai`, `json`, `re`

**Prerequisito:** M08 (Prompt Engineering), M13 (RAG Básico)

---

## 1. ¿Qué es un agente?

```python
# que_es_agente.py
import os

os.makedirs("outputs", exist_ok=True)

# Un agente es un sistema que:
# 1. Percibe el entorno (observaciones)
# 2. Razona sobre el estado actual
# 3. Toma acciones (usa herramientas)
# 4. Recibe retroalimentación (observaciones de herramientas)
# 5. Itera hasta completar el objetivo

# Diferencia con LLM simple:
# - LLM: una llamada → respuesta directa
# - Agente: múltiples pasos con herramientas, memoria y planificación

COMPONENTES_AGENTE = {
    "cerebro": "LLM (GPT-4, Claude, Llama) — razona y decide",
    "herramientas": "Funciones que el agente puede invocar (búsqueda, código, APIs)",
    "memoria": "Historial de conversación + memoria a largo plazo",
    "planificador": "Estrategia para descomponer tareas complejas",
    "ejecutor": "Bucle que ejecuta herramientas y recoge observaciones",
}

print("=== Componentes de un Agente ===")
for componente, desc in COMPONENTES_AGENTE.items():
    print(f"  {componente}: {desc}")

# Paradigmas de agentes
paradigmas = {
    "ReAct": "Thought → Action → Observation (iterativo)",
    "Plan-and-Execute": "Plan completo → ejecutar pasos secuencialmente",
    "MRKL": "Modular Reasoning, Knowledge, Language — router a módulos especializados",
    "Reflexion": "ReAct + autoevaluación y reflexión sobre errores previos",
    "Tree of Thoughts": "Árbol de razonamiento + evaluación de caminos",
}

print("\n=== Paradigmas de Agentes ===")
for paradigma, desc in paradigmas.items():
    print(f"  {paradigma}: {desc}")

with open("outputs/m16_intro.txt", "w") as f:
    f.write("M16: Agentes ReAct\n")
print("\noutputs/m16_intro.txt guardado")
```

---

## 2. ReAct — Reason + Act

```python
# react_basico.py
import json
import re
import os
from typing import List, Dict, Callable, Optional, Any

os.makedirs("outputs", exist_ok=True)

# --- Herramientas (tools) ---
def buscar_wikipedia(query: str) -> str:
    """Simula búsqueda en Wikipedia."""
    base_conocimiento = {
        "transformer": "El Transformer es una arquitectura de deep learning que usa mecanismo de atención. Fue introducido en 2017 por Vaswani et al. en 'Attention is All You Need'.",
        "bert": "BERT (Bidirectional Encoder Representations from Transformers) es un modelo de lenguaje preentrenado por Google en 2018. Usa Masked Language Modeling.",
        "gpt": "GPT (Generative Pre-trained Transformer) es una serie de modelos de OpenAI. GPT-3 tiene 175B parámetros. GPT-4 es multimodal.",
        "rlhf": "RLHF (Reinforcement Learning from Human Feedback) es una técnica de fine-tuning que usa preferencias humanas para alinear modelos. Usado en ChatGPT.",
        "lora": "LoRA (Low-Rank Adaptation) añade matrices de bajo rango a capas de transformers para fine-tuning eficiente. Reduce parámetros entrenables en 99%.",
        "rag": "RAG (Retrieval-Augmented Generation) combina recuperación de documentos con generación de texto para reducir alucinaciones.",
    }
    query_lower = query.lower()
    for clave, texto in base_conocimiento.items():
        if clave in query_lower:
            return texto
    return f"No encontré información sobre '{query}' en Wikipedia."

def calculadora(expresion: str) -> str:
    """Evalúa expresiones matemáticas simples."""
    try:
        # Solo permitir operaciones básicas
        if re.match(r'^[\d\s\+\-\*\/\.\(\)]+$', expresion):
            resultado = eval(expresion)
            return f"Resultado: {resultado}"
        else:
            return f"Error: expresión no permitida '{expresion}'"
    except Exception as e:
        return f"Error: {str(e)}"

def buscar_codigo(lenguaje: str, tema: str) -> str:
    """Retorna ejemplos de código."""
    ejemplos = {
        ("python", "list comprehension"): "[x**2 for x in range(10)]",
        ("python", "class"): "class Foo:\n    def __init__(self):\n        pass",
        ("python", "decorator"): "@functools.wraps(fn)\ndef wrapper(*args, **kwargs):\n    return fn(*args, **kwargs)",
    }
    key = (lenguaje.lower(), tema.lower())
    return ejemplos.get(key, f"# Ejemplo de {tema} en {lenguaje}\n# [código aquí]")

def get_fecha_actual() -> str:
    """Retorna la fecha actual."""
    import datetime
    return datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# --- Registro de herramientas ---
HERRAMIENTAS: Dict[str, Dict] = {
    "buscar_wikipedia": {
        "fn": buscar_wikipedia,
        "descripcion": "Busca información en Wikipedia sobre un tema",
        "parametros": {"query": "str — término a buscar"},
    },
    "calculadora": {
        "fn": calculadora,
        "descripcion": "Evalúa expresiones matemáticas (suma, resta, multiplicación, división)",
        "parametros": {"expresion": "str — expresión matemática, ej: '2 + 3 * 4'"},
    },
    "buscar_codigo": {
        "fn": buscar_codigo,
        "descripcion": "Busca ejemplos de código en un lenguaje de programación",
        "parametros": {
            "lenguaje": "str — lenguaje de programación",
            "tema": "str — concepto o patrón a buscar"
        },
    },
    "get_fecha_actual": {
        "fn": get_fecha_actual,
        "descripcion": "Obtiene la fecha y hora actuales",
        "parametros": {},
    },
}

# --- Agente ReAct ---
class AgenteReAct:
    """
    Agente que implementa el paradigma ReAct:
    Thought (razonamiento) → Action (acción) → Observation (observación)

    Formato de respuesta del LLM:
    Thought: [razonamiento sobre qué hacer]
    Action: [nombre_herramienta(parametros)]
    Observation: [resultado de la herramienta — añadido por el ejecutor]
    ... (repite hasta)
    Thought: [tengo suficiente información]
    Final Answer: [respuesta final]
    """
    def __init__(self, herramientas: Dict, llm_fn: Callable, max_pasos: int = 10):
        self.herramientas = herramientas
        self.llm_fn = llm_fn
        self.max_pasos = max_pasos
        self.historial: List[Dict] = []

    def _construir_system_prompt(self) -> str:
        """Prompt del sistema que describe las herramientas disponibles."""
        tools_desc = "\n".join([
            f"- {nombre}: {info['descripcion']}\n  Parámetros: {info['parametros']}"
            for nombre, info in self.herramientas.items()
        ])

        return f"""Eres un agente asistente que responde preguntas usando herramientas.
Sigue SIEMPRE este formato:

Thought: [tu razonamiento sobre qué hacer]
Action: nombre_herramienta(param1="valor1", param2="valor2")
Observation: [resultado de la herramienta — será llenado automáticamente]

Repite Thought/Action/Observation hasta tener suficiente información. Luego:
Final Answer: [respuesta final completa]

Herramientas disponibles:
{tools_desc}

IMPORTANTE: Solo usa herramientas listadas. El formato Action debe ser exacto."""

    def _parsear_action(self, texto: str) -> Optional[Dict]:
        """
        Parsea la acción del LLM.
        Formato: nombre_herramienta(param1="val1", param2="val2")
        Retorna: {"nombre": str, "parametros": dict} o None si no hay action.
        """
        # Buscar línea Action:
        match = re.search(r'Action:\s*(\w+)\((.*?)\)', texto, re.DOTALL)
        if not match:
            return None

        nombre = match.group(1).strip()
        params_str = match.group(2).strip()

        # Parsear parámetros key="value" o key=value
        parametros = {}
        if params_str:
            # Intenta como JSON primero
            try:
                # Convertir a formato JSON
                json_str = "{" + re.sub(r'(\w+)=', r'"\1":', params_str) + "}"
                json_str = json_str.replace("'", '"')
                parametros = json.loads(json_str)
            except Exception:
                # Fallback: un solo parámetro posicional
                parametros = {"query": params_str.strip('"').strip("'")}

        return {"nombre": nombre, "parametros": parametros}

    def _ejecutar_herramienta(self, action: Dict) -> str:
        """Ejecuta la herramienta y retorna la observación."""
        nombre = action["nombre"]
        parametros = action["parametros"]

        if nombre not in self.herramientas:
            return f"Error: herramienta '{nombre}' no encontrada. Disponibles: {list(self.herramientas.keys())}"

        try:
            fn = self.herramientas[nombre]["fn"]
            resultado = fn(**parametros)
            return str(resultado)
        except Exception as e:
            return f"Error al ejecutar {nombre}: {str(e)}"

    def _hay_respuesta_final(self, texto: str) -> Optional[str]:
        """Extrae la respuesta final si existe."""
        match = re.search(r'Final Answer:\s*(.+?)(?:\n\n|\Z)', texto, re.DOTALL)
        if match:
            return match.group(1).strip()
        return None

    def run(self, tarea: str) -> Dict:
        """
        Ejecuta el bucle ReAct para completar la tarea.
        Retorna: {"respuesta": str, "pasos": list, "n_pasos": int}
        """
        print(f"\n[Agente] Tarea: {tarea}\n" + "="*50)

        # Contexto acumulado del agente
        contexto = f"Tarea: {tarea}\n\n"
        pasos = []

        for paso_num in range(self.max_pasos):
            # Llamar al LLM con el contexto actual
            prompt_completo = self._construir_system_prompt() + f"\n\n{contexto}"
            respuesta_llm = self.llm_fn(prompt_completo)

            print(f"\n[Paso {paso_num + 1}]\nLLM: {respuesta_llm}")

            # Verificar si hay respuesta final
            respuesta_final = self._hay_respuesta_final(respuesta_llm)
            if respuesta_final:
                pasos.append({"tipo": "final", "contenido": respuesta_final})
                print(f"\n[Final] {respuesta_final}")
                return {
                    "respuesta": respuesta_final,
                    "pasos": pasos,
                    "n_pasos": paso_num + 1
                }

            # Parsear y ejecutar acción
            action = self._parsear_action(respuesta_llm)
            if not action:
                # El LLM no generó acción válida, intentar continuar
                contexto += f"{respuesta_llm}\nObservation: No se pudo ejecutar ninguna acción.\n"
                continue

            # Ejecutar herramienta
            observacion = self._ejecutar_herramienta(action)
            print(f"[Tool] {action['nombre']}({action['parametros']}) → {observacion[:100]}")

            # Actualizar contexto
            contexto += f"{respuesta_llm}\nObservation: {observacion}\n"
            pasos.append({
                "tipo": "action",
                "herramienta": action["nombre"],
                "parametros": action["parametros"],
                "observacion": observacion[:100]
            })

        return {
            "respuesta": "Máximo de pasos alcanzado sin respuesta final.",
            "pasos": pasos,
            "n_pasos": self.max_pasos
        }

# --- LLM Simulado para pruebas ---
class LLMSimuladoReAct:
    """
    LLM simulado que genera respuestas ReAct predefinidas para demo.
    En producción: reemplazar con llamada a Claude/GPT-4.
    """
    def __init__(self):
        self._paso = 0
        self._scripts: Dict[str, List[str]] = {
            "transformer": [
                'Thought: Necesito buscar información sobre el Transformer.\nAction: buscar_wikipedia(query="transformer")',
                'Thought: Tengo la información necesaria.\nFinal Answer: El Transformer es una arquitectura de deep learning introducida en 2017 que usa mecanismo de atención para procesar secuencias en paralelo.',
            ],
            "calculo": [
                'Thought: Debo calcular esta expresión.\nAction: calculadora(expresion="125 * 8 + 300")',
                'Thought: Tengo el resultado.\nFinal Answer: El resultado es 1300 (125 × 8 = 1000, más 300).',
            ],
            "fecha": [
                'Thought: Debo obtener la fecha actual.\nAction: get_fecha_actual()',
                'Thought: Tengo la fecha actual.\nFinal Answer: La fecha actual es la que retornó la herramienta.',
            ],
        }

    def __call__(self, prompt: str) -> str:
        # Detectar qué script usar según el prompt
        prompt_lower = prompt.lower()
        if "transformer" in prompt_lower or "bert" in prompt_lower:
            script = self._scripts["transformer"]
        elif "calcul" in prompt_lower or "suma" in prompt_lower or "multiplic" in prompt_lower:
            script = self._scripts["calculo"]
        elif "fecha" in prompt_lower or "hora" in prompt_lower:
            script = self._scripts["fecha"]
        else:
            return 'Thought: No tengo suficiente información.\nFinal Answer: No puedo responder esta pregunta con las herramientas disponibles.'

        respuesta = script[min(self._paso, len(script) - 1)]
        self._paso += 1
        return respuesta

# --- Demo ---
llm = LLMSimuladoReAct()
agente = AgenteReAct(HERRAMIENTAS, llm, max_pasos=5)

tareas = [
    "¿Qué es el Transformer y cuándo fue creado?",
    "¿Cuánto es 125 multiplicado por 8 más 300?",
]

import json
resultados_react = []
for tarea in tareas:
    llm._paso = 0  # Reset para cada tarea
    resultado = agente.run(tarea)
    resultados_react.append({
        "tarea": tarea,
        "n_pasos": resultado["n_pasos"],
        "respuesta": resultado["respuesta"][:100]
    })

with open("outputs/m16_react_resultados.json", "w", encoding="utf-8") as f:
    json.dump(resultados_react, f, ensure_ascii=False, indent=2)
print("\noutputs/m16_react_resultados.json guardado")
```

---

## 3. Tool Use con APIs reales (Claude / OpenAI)

```python
# tool_use_api.py
import json
import os

os.makedirs("outputs", exist_ok=True)

# --- Definición de herramientas en formato OpenAI/Anthropic ---
TOOLS_OPENAI = [
    {
        "type": "function",
        "function": {
            "name": "buscar_wikipedia",
            "description": "Busca información en Wikipedia sobre un tema específico",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "El término o tema a buscar"
                    }
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculadora",
            "description": "Evalúa expresiones matemáticas",
            "parameters": {
                "type": "object",
                "properties": {
                    "expresion": {
                        "type": "string",
                        "description": "Expresión matemática a evaluar, ej: '2 + 3 * 4'"
                    }
                },
                "required": ["expresion"]
            }
        }
    }
]

TOOLS_ANTHROPIC = [
    {
        "name": "buscar_wikipedia",
        "description": "Busca información en Wikipedia sobre un tema específico",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "El término o tema a buscar"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "calculadora",
        "description": "Evalúa expresiones matemáticas simples",
        "input_schema": {
            "type": "object",
            "properties": {
                "expresion": {
                    "type": "string",
                    "description": "Expresión matemática a evaluar"
                }
            },
            "required": ["expresion"]
        }
    }
]

print("=== Tool Definitions ===")
print("\nOpenAI format (primeras 100 chars):")
print(json.dumps(TOOLS_OPENAI[0], indent=2)[:200])

print("\nAnthropic format (primeras 100 chars):")
print(json.dumps(TOOLS_ANTHROPIC[0], indent=2)[:200])

# --- Código de uso con Anthropic (requiere ANTHROPIC_API_KEY) ---
CODIGO_ANTHROPIC = '''
import anthropic
import json

client = anthropic.Anthropic()  # usa ANTHROPIC_API_KEY del entorno

# Implementaciones de herramientas
def buscar_wikipedia(query: str) -> str:
    return f"Resultado de Wikipedia para: {query}"

def calculadora(expresion: str) -> str:
    try:
        return str(eval(expresion))
    except:
        return "Error en la expresión"

TOOL_FUNCTIONS = {
    "buscar_wikipedia": buscar_wikipedia,
    "calculadora": calculadora,
}

def agente_con_tools(pregunta: str, max_iter: int = 5) -> str:
    messages = [{"role": "user", "content": pregunta}]

    for _ in range(max_iter):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=TOOLS_ANTHROPIC,
            messages=messages
        )

        # Verificar si el modelo quiere usar una herramienta
        if response.stop_reason == "tool_use":
            tool_use_blocks = [b for b in response.content
                               if b.type == "tool_use"]
            tool_results = []

            for tool_block in tool_use_blocks:
                fn = TOOL_FUNCTIONS.get(tool_block.name)
                if fn:
                    resultado = fn(**tool_block.input)
                else:
                    resultado = f"Herramienta '{tool_block.name}' no encontrada"

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_block.id,
                    "content": resultado
                })

            # Añadir respuesta del modelo y resultados al historial
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})

        elif response.stop_reason == "end_turn":
            # El modelo terminó, extraer respuesta final
            texto = next((b.text for b in response.content
                          if hasattr(b, "text")), "")
            return texto

    return "Máximo de iteraciones alcanzado"
'''

print("\n=== Código de uso con Anthropic API ===")
print(CODIGO_ANTHROPIC[:600] + "...")

# Guardar código de referencia
with open("outputs/m16_tool_use_anthropic.py", "w") as f:
    f.write(CODIGO_ANTHROPIC)
print("\noutputs/m16_tool_use_anthropic.py guardado")
```

---

## 4. Agente con memoria y planificación

```python
# agente_con_memoria.py
import json
import os
from typing import List, Dict, Any, Optional
from collections import deque

os.makedirs("outputs", exist_ok=True)

class MemoriaAgente:
    """
    Sistema de memoria para agentes:
    - Memoria de trabajo: contexto inmediato de la conversación
    - Memoria episódica: historial de interacciones pasadas
    - Memoria semántica: hechos permanentes aprendidos
    """
    def __init__(self, max_contexto: int = 10):
        self.contexto: deque = deque(maxlen=max_contexto)  # memoria de trabajo
        self.episodios: List[Dict] = []                     # memoria episódica
        self.hechos: Dict[str, str] = {}                   # memoria semántica

    def agregar_mensaje(self, rol: str, contenido: str):
        """Añade un mensaje al contexto de trabajo."""
        self.contexto.append({"rol": rol, "contenido": contenido})

    def guardar_episodio(self, tarea: str, resultado: str):
        """Guarda un episodio completado."""
        self.episodios.append({
            "tarea": tarea,
            "resultado": resultado,
            "n_episodio": len(self.episodios) + 1
        })

    def aprender_hecho(self, clave: str, valor: str):
        """Almacena un hecho permanente."""
        self.hechos[clave] = valor

    def contexto_como_texto(self) -> str:
        return "\n".join([f"{m['rol']}: {m['contenido']}"
                          for m in self.contexto])

    def buscar_episodios_similares(self, tarea: str) -> List[Dict]:
        """Recupera episodios relevantes por overlap de palabras."""
        palabras_tarea = set(tarea.lower().split())
        similares = []
        for ep in self.episodios:
            palabras_ep = set(ep["tarea"].lower().split())
            overlap = len(palabras_tarea & palabras_ep)
            if overlap > 0:
                similares.append({**ep, "overlap": overlap})
        return sorted(similares, key=lambda x: x["overlap"], reverse=True)[:3]

class PlanificadorAgente:
    """
    Descompone tareas complejas en sub-tareas ejecutables.
    Estrategia: descomposición jerárquica con prioridades.
    """
    def __init__(self, llm_fn):
        self.llm_fn = llm_fn

    def planificar(self, tarea: str) -> List[Dict]:
        """
        Genera un plan de sub-tareas para la tarea principal.
        En producción: usar LLM para generar el plan.
        """
        # Simulación: descomponer por palabras clave
        subtareas = []

        if "busca" in tarea.lower() or "encuentra" in tarea.lower():
            subtareas.append({"id": 1, "accion": "buscar", "descripcion": f"Buscar información sobre: {tarea}", "prioridad": 1})

        if "calcula" in tarea.lower() or "cuánto" in tarea.lower():
            subtareas.append({"id": 2, "accion": "calcular", "descripcion": f"Realizar cálculos para: {tarea}", "prioridad": 1})

        if "resume" in tarea.lower() or "explica" in tarea.lower():
            subtareas.append({"id": 3, "accion": "sintetizar", "descripcion": "Sintetizar información recuperada", "prioridad": 2})

        if not subtareas:
            subtareas.append({"id": 1, "accion": "resolver", "descripcion": tarea, "prioridad": 1})

        # Añadir siempre tarea de respuesta final
        subtareas.append({"id": len(subtareas)+1, "accion": "responder", "descripcion": "Formular respuesta final", "prioridad": 3})

        return subtareas

class AgenteConMemoria:
    """
    Agente con memoria episódica, semántica y planificación.
    """
    def __init__(self, herramientas, llm_fn, max_pasos=8):
        self.herramientas = herramientas
        self.llm_fn = llm_fn
        self.max_pasos = max_pasos
        self.memoria = MemoriaAgente(max_contexto=20)
        self.planificador = PlanificadorAgente(llm_fn)

    def run(self, tarea: str) -> Dict:
        # Buscar episodios similares en memoria
        episodios_similares = self.memoria.buscar_episodios_similares(tarea)

        # Generar plan
        plan = self.planificador.planificar(tarea)
        print(f"\n[Plan] {len(plan)} sub-tareas:")
        for st in plan:
            print(f"  [{st['prioridad']}] {st['accion']}: {st['descripcion'][:50]}")

        # Añadir contexto de episodios previos si existen
        contexto_previo = ""
        if episodios_similares:
            contexto_previo = f"Episodios previos relevantes:\n"
            for ep in episodios_similares[:2]:
                contexto_previo += f"  - Tarea: {ep['tarea'][:50]} → Resultado: {ep['resultado'][:50]}\n"

        # Ejecutar con ReAct simplificado
        observaciones = []
        for subtarea in plan:
            if subtarea["accion"] == "responder":
                continue
            obs = f"Completé: {subtarea['descripcion']}"
            observaciones.append(obs)

        # Generar respuesta final
        respuesta = f"[Agente] Tarea completada en {len(plan)} pasos."
        if observaciones:
            respuesta += f" Observaciones: {'; '.join(observaciones[:2])}"

        # Guardar en memoria
        self.memoria.guardar_episodio(tarea, respuesta)
        self.memoria.agregar_mensaje("user", tarea)
        self.memoria.agregar_mensaje("assistant", respuesta)

        return {
            "tarea": tarea,
            "plan": plan,
            "episodios_usados": len(episodios_similares),
            "respuesta": respuesta
        }

# --- Demo ---
import re

def buscar_wikipedia(query: str) -> str:
    base = {
        "transformer": "Arquitectura de DL con atención multi-cabeza, 2017.",
        "bert": "Modelo bidireccional de Google, preentrenado con MLM.",
        "rag": "Recuperación + generación para reducir alucinaciones.",
    }
    return base.get(query.lower(), f"No encontré '{query}'")

def calculadora(expresion: str) -> str:
    try:
        if re.match(r'^[\d\s\+\-\*\/\.\(\)]+$', expresion):
            return str(eval(expresion))
        return "Error: expresión no permitida"
    except Exception as e:
        return f"Error: {e}"

herramientas = {
    "buscar_wikipedia": {"fn": buscar_wikipedia, "descripcion": "Busca en Wikipedia"},
    "calculadora": {"fn": calculadora, "descripcion": "Calcula expresiones"},
}

def llm_sim(prompt): return 'Thought: Resolviendo.\nFinal Answer: Tarea completada.'

agente = AgenteConMemoria(herramientas, llm_sim)

tareas = [
    "Busca información sobre el Transformer",
    "Explica qué es BERT",
    "Busca sobre transformers (segunda vez)",
]

resultados = []
for tarea in tareas:
    r = agente.run(tarea)
    resultados.append({"tarea": tarea, "n_subtareas": len(r["plan"]),
                       "episodios_usados": r["episodios_usados"]})
    print(f"\n'{tarea}': {r['episodios_usados']} episodios previos relevantes")

print(f"\nMemoria episódica: {len(agente.memoria.episodios)} episodios guardados")

with open("outputs/m16_agente_memoria.json", "w", encoding="utf-8") as f:
    json.dump(resultados, f, ensure_ascii=False, indent=2)
print("outputs/m16_agente_memoria.json guardado")
```

---

## 5. Checkpoint — Pruebas unitarias

```python
# checkpoint_m16.py
"""
Checkpoint M16: Fundamentos de Agentes — ReAct y Tool Use
Ejecutar con: python checkpoint_m16.py
"""
import json
import re
import os
from typing import Dict, List, Optional

os.makedirs("outputs", exist_ok=True)

# ── Implementaciones locales ──────────────────────────────────────────────────
def parsear_action(texto: str) -> Optional[Dict]:
    match = re.search(r'Action:\s*(\w+)\((.*?)\)', texto, re.DOTALL)
    if not match:
        return None
    nombre = match.group(1).strip()
    params_str = match.group(2).strip()
    parametros = {}
    if params_str:
        try:
            json_str = "{" + re.sub(r'(\w+)=', r'"\1":', params_str) + "}"
            json_str = json_str.replace("'", '"')
            parametros = json.loads(json_str)
        except Exception:
            parametros = {"query": params_str.strip('"').strip("'")}
    return {"nombre": nombre, "parametros": parametros}

def hay_respuesta_final(texto: str) -> Optional[str]:
    match = re.search(r'Final Answer:\s*(.+?)(?:\n\n|\Z)', texto, re.DOTALL)
    return match.group(1).strip() if match else None

def calculadora_tool(expresion: str) -> str:
    try:
        if re.match(r'^[\d\s\+\-\*\/\.\(\)]+$', expresion):
            return str(eval(expresion))
        return "Error"
    except Exception as e:
        return f"Error: {e}"

# ── Tests ─────────────────────────────────────────────────────────────────────
def test_parsear_action_formato_correcto():
    """T1: el parser extrae correctamente nombre y parámetros de la acción"""
    texto = 'Thought: Necesito buscar.\nAction: buscar_wikipedia(query="transformer")'
    action = parsear_action(texto)
    assert action is not None, "Debe parsear la acción"
    assert action["nombre"] == "buscar_wikipedia", f"Nombre incorrecto: {action['nombre']}"
    assert "query" in action["parametros"], "Debe tener parámetro query"
    assert action["parametros"]["query"] == "transformer", \
        f"Query incorrecta: {action['parametros']['query']}"
    print(f"T1 PASS: action={action}")

def test_parsear_action_sin_action():
    """T2: sin Action: retorna None"""
    texto = "Thought: Tengo suficiente información.\nFinal Answer: La respuesta es 42."
    action = parsear_action(texto)
    assert action is None, f"Sin Action debe retornar None, got {action}"
    print("T2 PASS: sin Action retorna None")

def test_detectar_respuesta_final():
    """T3: detecta correctamente Final Answer"""
    texto = "Thought: Ya sé la respuesta.\nFinal Answer: El Transformer usa atención."
    respuesta = hay_respuesta_final(texto)
    assert respuesta is not None, "Debe detectar Final Answer"
    assert "Transformer" in respuesta, "La respuesta debe contener 'Transformer'"
    print(f"T3 PASS: respuesta_final={respuesta[:50]}")

def test_calculadora_tool_correcto():
    """T4: la herramienta calculadora evalúa expresiones correctamente"""
    casos = [
        ("2 + 3", "5"),
        ("10 * 4 + 2", "42"),
        ("100 / 4", "25.0"),
    ]
    for expr, esperado in casos:
        resultado = calculadora_tool(expr)
        assert resultado == esperado, \
            f"calculadora('{expr}') = '{resultado}', esperado '{esperado}'"
    print(f"T4 PASS: calculadora evalúa {len(casos)} expresiones correctamente")

def test_herramienta_invalida_retorna_error():
    """T5: llamar herramienta inexistente retorna mensaje de error"""
    herramientas = {"buscar": {"fn": lambda q: "resultado", "descripcion": "busca"}}
    nombre = "herramienta_inexistente"
    if nombre not in herramientas:
        resultado = f"Error: herramienta '{nombre}' no encontrada"
    else:
        resultado = herramientas[nombre]["fn"]()
    assert "Error" in resultado or "no encontrada" in resultado, \
        f"Debe retornar error, got: {resultado}"
    print(f"T5 PASS: herramienta inválida genera error: '{resultado[:60]}'")

def test_react_loop_termina_con_final_answer():
    """T6: el bucle ReAct termina cuando el LLM produce Final Answer"""
    pasos = [
        'Thought: Buscaré información.\nAction: buscar_wikipedia(query="rag")',
        'Thought: Tengo suficiente información.\nFinal Answer: RAG combina recuperación y generación.',
    ]
    paso_actual = [0]

    def llm_sim(prompt):
        idx = min(paso_actual[0], len(pasos) - 1)
        paso_actual[0] += 1
        return pasos[idx]

    def buscar_wiki(query):
        return "RAG es Retrieval-Augmented Generation."

    herramientas = {"buscar_wikipedia": {"fn": buscar_wiki, "descripcion": "busca"}}

    # Simular bucle ReAct
    contexto = "Tarea: ¿Qué es RAG?\n\n"
    max_pasos = 5
    respuesta_final = None

    for _ in range(max_pasos):
        resp_llm = llm_sim(contexto)
        respuesta_final = hay_respuesta_final(resp_llm)
        if respuesta_final:
            break
        action = parsear_action(resp_llm)
        if action and action["nombre"] in herramientas:
            obs = herramientas[action["nombre"]]["fn"](**action["parametros"])
            contexto += f"{resp_llm}\nObservation: {obs}\n"

    assert respuesta_final is not None, "El bucle debe terminar con respuesta final"
    assert "RAG" in respuesta_final, f"Respuesta debe mencionar RAG: {respuesta_final}"
    print(f"T6 PASS: bucle termina con respuesta: '{respuesta_final[:60]}'")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_parsear_action_formato_correcto,
        test_parsear_action_sin_action,
        test_detectar_respuesta_final,
        test_calculadora_tool_correcto,
        test_herramienta_invalida_retorna_error,
        test_react_loop_termina_con_final_answer,
    ]

    passed = 0
    for test in tests:
        try:
            test()
            passed += 1
        except AssertionError as e:
            print(f"FAIL: {test.__name__}: {e}")
        except Exception as e:
            print(f"ERROR: {test.__name__}: {e}")

    print(f"\n{'='*50}")
    print(f"Resultado: {passed}/{len(tests)} tests pasaron")
    if passed == len(tests):
        print("M16 COMPLETADO")
```

---

## Resumen

| Componente | Descripción | Implementación |
|---|---|---|
| **ReAct loop** | Thought → Action → Observation | Bucle con parser de acción |
| **Tool use** | Funciones invocables por el agente | Dict de funciones con schema |
| **Parser** | Extrae Action del texto del LLM | Regex sobre formato `nombre(params)` |
| **Memoria** | Historial + episodios + hechos | deque + lista + dict |
| **Planificador** | Descompone tarea en sub-tareas | LLM + heurísticas |

**Formato ReAct:**
```
Thought: [razonamiento sobre qué hacer]
Action: herramienta(param="valor")
Observation: [resultado de la herramienta]
Thought: [nueva reflexión con la observación]
Action: otra_herramienta(param="valor2")
...
Final Answer: [respuesta final]
```

**Tool use con Anthropic API:**
- El modelo retorna `stop_reason="tool_use"` cuando quiere usar una herramienta
- El `content` incluye bloques `ToolUseBlock` con `name`, `id`, e `input`
- Se ejecuta la herramienta y se retorna `ToolResultBlock` al modelo
- El ciclo continúa hasta `stop_reason="end_turn"`
