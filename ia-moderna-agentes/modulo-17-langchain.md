# Módulo 17 — LangChain — Cadenas y Agentes

**Objetivo:** Entender la arquitectura de LangChain, implementar chains (cadenas), prompts y agentes usando LangChain desde sus componentes fundamentales, con implementaciones compatibles que no requieren API keys.

**Herramientas:** `langchain`, `langchain-community`, `langchain-openai`, `numpy`

**Prerequisito:** M16 (Agentes ReAct)

---

## 1. Arquitectura de LangChain

```python
# langchain_arquitectura.py
import os

os.makedirs("outputs", exist_ok=True)

# LangChain organiza sus componentes en capas:
#
#  ┌─────────────────────────────────────────────┐
#  │          LCEL (LangChain Expression Language) │
#  │   chain = prompt | llm | output_parser       │
#  └─────────────────────────────────────────────┘
#           │
#  ┌────────────────────────────────────────────────┐
#  │  Runnable Protocol: invoke / batch / stream    │
#  └────────────────────────────────────────────────┘
#           │
#  ┌─────────────┬─────────────┬──────────────────┐
#  │  Prompts    │  LLMs/Chat  │  Output Parsers  │
#  │  Templates  │  Models     │  (str, JSON, ...)│
#  └─────────────┴─────────────┴──────────────────┘
#           │
#  ┌─────────────┬─────────────┬──────────────────┐
#  │  Retrievers │  VectorDB   │  Memory          │
#  │  Embeddings │  Stores     │  Buffers         │
#  └─────────────┴─────────────┴──────────────────┘
#           │
#  ┌─────────────────────────────────────────────┐
#  │  Agents + Tools: ReAct, OpenAI Functions    │
#  └─────────────────────────────────────────────┘

COMPONENTES = {
    "PromptTemplate": "Plantillas de prompts con variables",
    "ChatPromptTemplate": "Prompts para modelos de chat (system/human/ai)",
    "LLMChain": "Cadena básica: prompt → LLM → output",
    "SequentialChain": "Múltiples LLMChains en secuencia",
    "RunnablePassthrough": "Pasa datos sin transformar (LCEL)",
    "RunnableParallel": "Ejecuta múltiples runnables en paralelo",
    "ConversationBufferMemory": "Memoria de conversación en buffer",
    "VectorStoreRetriever": "Recuperador de documentos vectoriales",
    "AgentExecutor": "Ejecutor del bucle ReAct para agentes",
}

print("=== Componentes LangChain ===")
for k, v in COMPONENTES.items():
    print(f"  {k}: {v}")

with open("outputs/m17_intro.txt", "w") as f:
    f.write("M17: LangChain - Cadenas y Agentes\n")
print("\noutputs/m17_intro.txt guardado")
```

---

## 2. PromptTemplate y ChatPromptTemplate

```python
# langchain_prompts.py
import os
import re
from typing import Dict, List, Any, Optional

os.makedirs("outputs", exist_ok=True)

# --- Implementación mínima de PromptTemplate compatible con LangChain ---
class PromptTemplate:
    """
    Plantilla de prompt con variables entre llaves: {variable}
    Equivalente a langchain.prompts.PromptTemplate
    """
    def __init__(self, template: str, input_variables: List[str] = None):
        self.template = template
        # Auto-detectar variables si no se especifican
        if input_variables is None:
            self.input_variables = re.findall(r'\{(\w+)\}', template)
        else:
            self.input_variables = input_variables

    def format(self, **kwargs) -> str:
        """Rellena el template con los valores dados."""
        missing = set(self.input_variables) - set(kwargs.keys())
        if missing:
            raise ValueError(f"Variables faltantes: {missing}")
        return self.template.format(**kwargs)

    def __or__(self, other):
        """Operador | para LCEL: prompt | llm"""
        return Chain([self, other])

    def __repr__(self):
        return f"PromptTemplate(input_variables={self.input_variables})"

class ChatMessage:
    def __init__(self, role: str, content: str):
        self.role = role    # "system" | "human" | "ai"
        self.content = content

    def __repr__(self):
        return f"<{self.role}> {self.content[:50]}"

class ChatPromptTemplate:
    """
    Plantilla para modelos de chat con múltiples mensajes.
    Equivalente a langchain.prompts.ChatPromptTemplate
    """
    def __init__(self, messages: List[tuple]):
        """
        messages: lista de (role, template_str)
        roles: "system" | "human" | "ai"
        """
        self.messages = messages
        # Extraer variables de todos los templates
        all_vars = []
        for _, tmpl in messages:
            all_vars.extend(re.findall(r'\{(\w+)\}', tmpl))
        self.input_variables = list(set(all_vars))

    @classmethod
    def from_messages(cls, messages):
        return cls(messages)

    def format_messages(self, **kwargs) -> List[ChatMessage]:
        """Rellena todos los templates y retorna lista de ChatMessage."""
        return [
            ChatMessage(role=role, content=tmpl.format(**kwargs))
            for role, tmpl in self.messages
        ]

    def format(self, **kwargs) -> str:
        """Formato string para compatibilidad."""
        msgs = self.format_messages(**kwargs)
        return "\n".join([f"[{m.role}]: {m.content}" for m in msgs])

    def __or__(self, other):
        return Chain([self, other])

# --- Demos ---
print("=== PromptTemplate ===")

# Prompt básico
pt = PromptTemplate(
    template="Eres un experto en {tema}. Responde en {idioma}: {pregunta}",
    input_variables=["tema", "idioma", "pregunta"]
)
print(f"Variables: {pt.input_variables}")
prompt_rellenado = pt.format(
    tema="inteligencia artificial",
    idioma="español",
    pregunta="¿Qué es el aprendizaje por refuerzo?"
)
print(f"Prompt:\n{prompt_rellenado}\n")

# FewShot prompt manual
class FewShotPromptTemplate:
    """Prompt con ejemplos few-shot."""
    def __init__(self, prefix: str, suffix: str,
                 examples: List[Dict], example_template: str):
        self.prefix = prefix
        self.suffix = suffix
        self.examples = examples
        self.example_template = example_template
        self.input_variables = re.findall(r'\{(\w+)\}', suffix)

    def format(self, **kwargs) -> str:
        # Formatear cada ejemplo
        ejemplos_str = "\n".join([
            self.example_template.format(**ex)
            for ex in self.examples
        ])
        return f"{self.prefix}\n{ejemplos_str}\n{self.suffix.format(**kwargs)}"

few_shot = FewShotPromptTemplate(
    prefix="Clasifica el sentimiento del texto.\n",
    suffix="Texto: {texto}\nSentimiento:",
    examples=[
        {"texto": "Me encantó la película", "sentimiento": "Positivo"},
        {"texto": "Terrible experiencia", "sentimiento": "Negativo"},
        {"texto": "Estuvo bien", "sentimiento": "Neutral"},
    ],
    example_template="Texto: {texto}\nSentimiento: {sentimiento}"
)
print("=== FewShotPromptTemplate ===")
print(few_shot.format(texto="El servicio fue excelente"))

# ChatPromptTemplate
print("\n=== ChatPromptTemplate ===")
chat_pt = ChatPromptTemplate.from_messages([
    ("system", "Eres un asistente experto en {dominio}. Responde en {idioma}."),
    ("human", "{pregunta}"),
    ("ai", "Entendido, responderé sobre {dominio}:"),
])
print(f"Variables: {chat_pt.input_variables}")
msgs = chat_pt.format_messages(
    dominio="machine learning",
    idioma="español",
    pregunta="¿Qué es el overfitting?"
)
for msg in msgs:
    print(f"  {msg}")

import json
with open("outputs/m17_prompts_demo.json", "w", encoding="utf-8") as f:
    json.dump({
        "prompt_template": prompt_rellenado,
        "few_shot_preview": few_shot.format(texto="Gran servicio")[:100],
        "chat_messages": [str(m) for m in msgs]
    }, f, ensure_ascii=False, indent=2)
print("\noutputs/m17_prompts_demo.json guardado")
```

---

## 3. LCEL — LangChain Expression Language

```python
# lcel_chains.py
import os
import json
from typing import Any, Dict, List, Optional, Callable, Union

os.makedirs("outputs", exist_ok=True)

# --- Protocolo Runnable ---
class Runnable:
    """Protocolo base de LangChain: cualquier objeto que tenga invoke()"""
    def invoke(self, input: Any) -> Any:
        raise NotImplementedError

    def __or__(self, other: 'Runnable') -> 'RunnableSequence':
        """Operador | para componer runnables: a | b | c"""
        return RunnableSequence([self, other])

    def batch(self, inputs: List[Any]) -> List[Any]:
        """Procesa múltiples inputs."""
        return [self.invoke(inp) for inp in inputs]

class RunnableSequence(Runnable):
    """
    Secuencia de Runnables conectados con |.
    La salida de cada uno es la entrada del siguiente.
    Equivalente a chain = a | b | c en LCEL.
    """
    def __init__(self, steps: List[Runnable]):
        self.steps = steps

    def invoke(self, input: Any) -> Any:
        resultado = input
        for step in self.steps:
            resultado = step.invoke(resultado)
        return resultado

    def __or__(self, other: Runnable) -> 'RunnableSequence':
        return RunnableSequence(self.steps + [other])

class RunnablePassthrough(Runnable):
    """Pasa el input sin transformar. Útil para preservar la query original."""
    def invoke(self, input: Any) -> Any:
        return input

class RunnableParallel(Runnable):
    """
    Ejecuta múltiples runnables en paralelo con el mismo input.
    Retorna dict con los resultados de cada rama.
    """
    def __init__(self, mapping: Dict[str, Runnable]):
        self.mapping = mapping

    def invoke(self, input: Any) -> Dict:
        return {key: runnable.invoke(input)
                for key, runnable in self.mapping.items()}

class RunnableLambda(Runnable):
    """Envuelve una función Python como Runnable."""
    def __init__(self, fn: Callable):
        self.fn = fn

    def invoke(self, input: Any) -> Any:
        return self.fn(input)

# --- LLM simulado ---
class LLMSimulado(Runnable):
    """LLM de prueba que opera sobre el prompt."""
    def invoke(self, prompt: str) -> str:
        if "sentimiento" in prompt.lower() or "clasifica" in prompt.lower():
            if any(w in prompt.lower() for w in ["bien", "excelente", "fantástico", "genial"]):
                return "Positivo"
            elif any(w in prompt.lower() for w in ["mal", "terrible", "pésimo"]):
                return "Negativo"
            return "Neutral"
        elif "resume" in prompt.lower() or "resumen" in prompt.lower():
            palabras = prompt.split()
            return f"[Resumen simulado de {len(palabras)} palabras]"
        return f"[LLM] Respuesta para: {prompt[:50]}..."

# --- Output Parsers ---
class StrOutputParser(Runnable):
    """Parser que retorna el string directamente."""
    def invoke(self, text: str) -> str:
        return text.strip()

class JsonOutputParser(Runnable):
    """Parser que extrae JSON del output del LLM."""
    def invoke(self, text: str) -> Dict:
        import re
        match = re.search(r'\{.*?\}', text, re.DOTALL)
        if match:
            try:
                return json.loads(match.group())
            except json.JSONDecodeError:
                pass
        return {"raw": text}

class CommaSeparatedListOutputParser(Runnable):
    """Parser que convierte 'a, b, c' en ['a', 'b', 'c']."""
    def invoke(self, text: str) -> List[str]:
        return [item.strip() for item in text.split(",") if item.strip()]

# --- PromptTemplate como Runnable ---
class PromptRunnable(Runnable):
    def __init__(self, template: str):
        import re
        self.template = template
        self.vars = re.findall(r'\{(\w+)\}', template)

    def invoke(self, input: Union[str, Dict]) -> str:
        if isinstance(input, str):
            return self.template.format(**{self.vars[0]: input}) if self.vars else input
        return self.template.format(**input)

# --- Demo LCEL ---
print("=== LCEL Demo ===\n")

llm = LLMSimulado()
str_parser = StrOutputParser()
list_parser = CommaSeparatedListOutputParser()

# Chain 1: clasificación de sentimiento
prompt_sentimiento = PromptRunnable(
    "Clasifica el sentimiento de este texto (Positivo/Negativo/Neutral): {texto}"
)
chain_sentimiento = prompt_sentimiento | llm | str_parser

textos = ["El servicio fue excelente", "Terrible experiencia", "Estuvo bien"]
print("Chain de clasificación de sentimiento:")
for texto in textos:
    resultado = chain_sentimiento.invoke({"texto": texto})
    print(f"  '{texto}' → {resultado}")

# Chain 2: resumen
prompt_resumen = PromptRunnable("Resume en una oración: {texto}")
chain_resumen = prompt_resumen | llm | str_parser

resultado_resumen = chain_resumen.invoke({"texto": "Python es un lenguaje de programación versátil."})
print(f"\nChain de resumen:\n  → {resultado_resumen}")

# Chain 3: parallel (genera pregunta + contexto simultáneamente)
chain_parallel = RunnableParallel({
    "texto_original": RunnablePassthrough(),
    "sentimiento": RunnableLambda(lambda x: chain_sentimiento.invoke({"texto": x})),
    "longitud": RunnableLambda(lambda x: len(x.split())),
})

resultado_parallel = chain_parallel.invoke("El producto es fantástico")
print(f"\nChain paralelo:")
for k, v in resultado_parallel.items():
    print(f"  {k}: {v}")

# Chain 4: secuencial (análisis en 2 pasos)
def enriquecer(input_dict: Dict) -> Dict:
    """Añade metadata al resultado."""
    return {**input_dict, "procesado": True, "n_palabras": len(input_dict.get("texto_original", "").split())}

chain_completo = chain_parallel | RunnableLambda(enriquecer)
resultado_completo = chain_completo.invoke("Gran experiencia con el equipo")
print(f"\nChain completo:")
print(json.dumps(resultado_completo, ensure_ascii=False, indent=2))

with open("outputs/m17_lcel_demo.json", "w", encoding="utf-8") as f:
    json.dump(resultado_completo, f, ensure_ascii=False, indent=2)
print("\noutputs/m17_lcel_demo.json guardado")
```

---

## 4. ConversationChain con Memoria

```python
# langchain_memoria.py
import os
import json
from typing import List, Dict, Any

os.makedirs("outputs", exist_ok=True)

# --- Memory Classes ---
class ConversationBufferMemory:
    """
    Almacena el historial completo de la conversación.
    Problema: crece sin límite → puede exceder el contexto del LLM.
    """
    def __init__(self, human_prefix="Human", ai_prefix="AI"):
        self.human_prefix = human_prefix
        self.ai_prefix = ai_prefix
        self.messages: List[Dict] = []

    def save_context(self, human_input: str, ai_output: str):
        self.messages.append({"role": "human", "content": human_input})
        self.messages.append({"role": "ai", "content": ai_output})

    def load_memory_variables(self) -> Dict:
        historial = "\n".join([
            f"{self.human_prefix}: {m['content']}" if m['role'] == 'human'
            else f"{self.ai_prefix}: {m['content']}"
            for m in self.messages
        ])
        return {"history": historial}

    def clear(self):
        self.messages = []

class ConversationBufferWindowMemory(ConversationBufferMemory):
    """
    Mantiene solo las últimas K interacciones (K pares human/AI).
    """
    def __init__(self, k: int = 5, **kwargs):
        super().__init__(**kwargs)
        self.k = k

    def load_memory_variables(self) -> Dict:
        # Mantener solo los últimos 2k mensajes (k pares)
        msgs_recientes = self.messages[-self.k * 2:]
        historial = "\n".join([
            f"{self.human_prefix}: {m['content']}" if m['role'] == 'human'
            else f"{self.ai_prefix}: {m['content']}"
            for m in msgs_recientes
        ])
        return {"history": historial}

class ConversationSummaryMemory:
    """
    Mantiene un resumen acumulativo en lugar del historial completo.
    El LLM actualiza el resumen con cada nueva interacción.
    Ventaja: contexto de longitud constante.
    """
    def __init__(self, llm_fn, human_prefix="Human", ai_prefix="AI"):
        self.llm_fn = llm_fn
        self.summary = ""
        self.human_prefix = human_prefix
        self.ai_prefix = ai_prefix

    def _actualizar_resumen(self, nuevo_intercambio: str):
        prompt = f"""Tenemos este resumen de la conversación hasta ahora:
{self.summary if self.summary else '(sin historial previo)'}

Nuevo intercambio:
{nuevo_intercambio}

Actualiza el resumen incluyendo el nuevo intercambio (máximo 3 oraciones):"""
        return self.llm_fn(prompt)

    def save_context(self, human_input: str, ai_output: str):
        intercambio = f"{self.human_prefix}: {human_input}\n{self.ai_prefix}: {ai_output}"
        self.summary = self._actualizar_resumen(intercambio)

    def load_memory_variables(self) -> Dict:
        return {"history": f"Resumen: {self.summary}"}

# --- ConversationChain ---
class ConversationChain:
    """
    Cadena conversacional con memoria integrada.
    Mantiene el historial de conversación y lo inyecta en cada prompt.
    """
    def __init__(self, llm, memory=None, prompt_template: str = None):
        self.llm = llm
        self.memory = memory or ConversationBufferMemory()
        self.prompt_template = prompt_template or """Eres un asistente útil.

Historial de conversación:
{history}
Humano: {human_input}
"""

    def predict(self, human_input: str) -> str:
        mem_vars = self.memory.load_memory_variables()
        history = mem_vars.get("history", "")
        prompt = self.prompt_template.format(history=history, human_input=human_input)
        ai_output = self.llm(prompt)
        self.memory.save_context(human_input, ai_output)
        return ai_output

# --- LLM simulado ---
class LLMSim:
    def __call__(self, prompt: str) -> str:
        if 'Python' in prompt or 'python' in prompt:
            return 'Python es un lenguaje de programacion de alto nivel, versatil y con gran ecosistema para IA.'
        elif 'IA' in prompt or 'inteligencia artificial' in prompt.lower():
            return 'La inteligencia artificial es la simulacion de procesos cognitivos humanos por sistemas computacionales.'
        elif 'aprendizaje' in prompt.lower() or 'machine learning' in prompt.lower():
            return 'El machine learning permite a los sistemas aprender de datos sin ser explicitamente programados.'
        elif 'Resumen' in prompt or 'resumen' in prompt:
            return 'Resumen actualizado con la nueva informacion proporcionada.'
        return f'[LLM] Procesado: {prompt[-50:]}'

# --- Demo ---
import os, json
os.makedirs('outputs', exist_ok=True)
llm = LLMSim()

# Buffer memory
print('=== ConversationChain con Buffer Memory ===')
mem_buffer = ConversationBufferMemory()
conv = ConversationChain(llm, mem_buffer)

conversacion = [
    'Hola, me gustaria aprender sobre Python',
    'Que aplicaciones tiene en inteligencia artificial?',
    'Como funciona el machine learning?',
]

for msg in conversacion:
    respuesta = conv.predict(msg)
    print(f'  Human: {msg[:50]}')
    print(f'  AI: {respuesta[:80]}')
    print()

hist = mem_buffer.load_memory_variables()['history']
print(f'Historial: {len(hist)} chars, {len(mem_buffer.messages)} mensajes')

# Window memory (k=2)
print('=== Window Memory (k=2) ===')
mem_window = ConversationBufferWindowMemory(k=2)
conv_w = ConversationChain(llm, mem_window)
for msg in conversacion:
    conv_w.predict(msg)
hist_w = mem_window.load_memory_variables()['history']
print(f'Historial ventana: {len(hist_w)} chars (vs buffer: {len(hist)} chars)')

# Summary memory
print('=== Summary Memory ===')
mem_summary = ConversationSummaryMemory(llm)
conv_s = ConversationChain(llm, mem_summary)
for msg in conversacion:
    conv_s.predict(msg)
print(f'Resumen: {mem_summary.summary[:100]}')

resultados = {
    'buffer_chars': len(hist),
    'window_chars': len(hist_w),
    'summary_chars': len(mem_summary.summary)
}
with open('outputs/m17_memoria_demo.json', 'w') as f:
    json.dump(resultados, f, indent=2)
print('outputs/m17_memoria_demo.json guardado')


---

## 5. Agente con LangChain Tools

```python
# langchain_agent.py
import os
import re
import json
from typing import List, Dict, Any, Callable, Optional

os.makedirs("outputs", exist_ok=True)

# --- Tool definition ---
class Tool:
    def __init__(self, name: str, func: Callable, description: str):
        self.name = name
        self.func = func
        self.description = description

    def run(self, input: str) -> str:
        try:
            return str(self.func(input))
        except Exception as e:
            return f"Error: {e}"

def buscar(q: str) -> str:
    base = {
        "transformer": "Arquitectura DL con atencion multi-cabeza (2017).",
        "bert": "Modelo Google preentrenado con MLM bidireccional.",
        "langchain": "Framework para construir aplicaciones con LLMs.",
        "rag": "RAG combina recuperacion vectorial con generacion LLM.",
    }
    for k, v in base.items():
        if k in q.lower():
            return v
    return f"Sin resultados para '{q}'"

def calcular(expr: str) -> str:
    try:
        expr_clean = re.sub(r"[^0-9\+\-\*\/\.\(\)\s]", "", expr)
        return str(eval(expr_clean)) if expr_clean.strip() else "Expresion invalida"
    except Exception as e:
        return f"Error: {e}"

herramientas = [
    Tool("buscar", buscar, "Busca informacion sobre un tema"),
    Tool("calcular", calcular, "Calcula expresiones matematicas"),
]

class AgenteLangChain:
    """
    Agente estilo LangChain con ZERO_SHOT_REACT_DESCRIPTION.
    El LLM decide que herramienta usar basado en su descripcion.
    """
    def __init__(self, llm, tools: List[Tool], max_iter: int = 5):
        self.llm = llm
        self.tools = {t.name: t for t in tools}
        self.max_iter = max_iter

    def _tools_description(self) -> str:
        return "
".join([
            f"- {t.name}: {t.description}" for t in self.tools.values()
        ])

    def _build_prompt(self, question: str, scratchpad: str) -> str:
        return f"""Responde la pregunta usando estas herramientas:
{self._tools_description()}

Formato:
Thought: razona que debes hacer
Action: nombre_herramienta
Action Input: el input para la herramienta
Observation: [resultado de la herramienta]
... (repite si necesario)
Thought: ya tengo la respuesta
Final Answer: respuesta completa

Pregunta: {question}
{scratchpad}"""

    def _parse(self, text: str):
        final = re.search(r"Final Answer:\s*(.+?)(?:
|$)", text, re.DOTALL)
        if final:
            return {"type": "final", "answer": final.group(1).strip()}
        action = re.search(r"Action:\s*(\w+)", text)
        action_input = re.search(r"Action Input:\s*(.+?)(?:
|$)", text)
        if action:
            return {
                "type": "action",
                "name": action.group(1).strip(),
                "input": action_input.group(1).strip() if action_input else ""
            }
        return {"type": "unknown"}

    def run(self, question: str) -> str:
        scratchpad = ""
        for _ in range(self.max_iter):
            prompt = self._build_prompt(question, scratchpad)
            resp = self.llm(prompt)
            scratchpad += resp + "
"
            parsed = self._parse(resp)
            if parsed["type"] == "final":
                return parsed["answer"]
            elif parsed["type"] == "action":
                tool_name = parsed["name"]
                if tool_name in self.tools:
                    obs = self.tools[tool_name].run(parsed["input"])
                else:
                    obs = f"Herramienta '{tool_name}' no encontrada"
                scratchpad += f"Observation: {obs}
"
            else:
                break
        return "No se pudo completar la tarea."

# LLM simulado para el agente
class LLMAgenteSim:
    def __init__(self):
        self._step = 0
        self._scripts = {
            "transformer": [
                "Thought: Buscare info sobre Transformer.
Action: buscar
Action Input: transformer",
                "Thought: Ya tengo la respuesta.
Final Answer: El Transformer es una arquitectura DL con atencion multi-cabeza, introducida en 2017.",
            ],
            "calculo": [
                "Thought: Debo calcular.
Action: calcular
Action Input: 15 * 7 + 25",
                "Thought: Tengo el resultado.
Final Answer: El resultado es 130.",
            ],
        }

    def __call__(self, prompt: str) -> str:
        if "transformer" in prompt.lower() or "bert" in prompt.lower():
            script = self._scripts["transformer"]
        else:
            script = self._scripts["calculo"]
        resp = script[min(self._step, len(script)-1)]
        self._step += 1
        return resp

# Demo
llm_agente = LLMAgenteSim()
agente = AgenteLangChain(llm_agente, herramientas)

preguntas = [
    "Que es el Transformer?",
    "Cuanto es 15 por 7 mas 25?",
]

resultados = []
for p in preguntas:
    llm_agente._step = 0
    r = agente.run(p)
    print(f"Q: {p}")
    print(f"A: {r}")
    print()
    resultados.append({"pregunta": p, "respuesta": r})

with open("outputs/m17_agent_resultados.json", "w", encoding="utf-8") as f:
    json.dump(resultados, f, ensure_ascii=False, indent=2)
print("outputs/m17_agent_resultados.json guardado")
```

---

## 6. Checkpoint — Pruebas unitarias

```python
# checkpoint_m17.py
"""
Checkpoint M17: LangChain — Cadenas y Agentes
Ejecutar con: python checkpoint_m17.py
"""
import re
import json
import os
from typing import Any, Dict, List, Optional, Callable

os.makedirs("outputs", exist_ok=True)

# Implementations
class PromptTemplate:
    def __init__(self, template: str):
        self.template = template
        self.input_variables = re.findall(r"\{(\w+)\}", template)

    def format(self, **kwargs) -> str:
        return self.template.format(**kwargs)

class Runnable:
    def invoke(self, input: Any) -> Any:
        raise NotImplementedError
    def __or__(self, other): return RunnableSequence([self, other])

class RunnableSequence(Runnable):
    def __init__(self, steps):
        self.steps = steps
    def invoke(self, input):
        result = input
        for s in self.steps: result = s.invoke(result)
        return result
    def __or__(self, other): return RunnableSequence(self.steps + [other])

class RunnableLambda(Runnable):
    def __init__(self, fn): self.fn = fn
    def invoke(self, inp): return self.fn(inp)

class RunnableParallel(Runnable):
    def __init__(self, mapping): self.mapping = mapping
    def invoke(self, inp): return {k: v.invoke(inp) for k, v in self.mapping.items()}

class ConversationBufferMemory:
    def __init__(self):
        self.messages = []
    def save_context(self, human, ai):
        self.messages.append({"role": "human", "content": human})
        self.messages.append({"role": "ai", "content": ai})
    def load_memory_variables(self):
        return {"history": "
".join(f"{m['role']}: {m['content']}" for m in self.messages)}

class ConversationBufferWindowMemory(ConversationBufferMemory):
    def __init__(self, k=3):
        super().__init__(); self.k = k
    def load_memory_variables(self):
        msgs = self.messages[-self.k*2:]
        return {"history": "
".join(f"{m['role']}: {m['content']}" for m in msgs)}

# Tests
def test_prompt_template_format():
    """T1: PromptTemplate rellena variables correctamente"""
    pt = PromptTemplate("Hola {nombre}, eres un experto en {tema}.")
    resultado = pt.format(nombre="Ana", tema="NLP")
    assert "Ana" in resultado, "Debe contener el nombre"
    assert "NLP" in resultado, "Debe contener el tema"
    assert "{nombre}" not in resultado, "No debe quedar la variable sin rellenar"
    print(f"T1 PASS: '{resultado}'")

def test_prompt_template_detecta_variables():
    """T2: PromptTemplate detecta automáticamente las variables"""
    pt = PromptTemplate("Query: {query}. Idioma: {idioma}.")
    assert "query" in pt.input_variables, "Debe detectar 'query'"
    assert "idioma" in pt.input_variables, "Debe detectar 'idioma'"
    assert len(pt.input_variables) == 2, f"Debe haber 2 variables, got {len(pt.input_variables)}"
    print(f"T2 PASS: variables={pt.input_variables}")

def test_lcel_pipeline():
    """T3: operador | compone runnables en secuencia"""
    double = RunnableLambda(lambda x: x * 2)
    add_10 = RunnableLambda(lambda x: x + 10)
    to_str = RunnableLambda(lambda x: f"Resultado: {x}")

    chain = double | add_10 | to_str
    resultado = chain.invoke(5)  # 5 * 2 = 10, + 10 = 20, "Resultado: 20"
    assert resultado == "Resultado: 20", f"Esperado 'Resultado: 20', got '{resultado}'"
    print(f"T3 PASS: chain(5) = '{resultado}'")

def test_runnable_parallel():
    """T4: RunnableParallel ejecuta múltiples ramas con el mismo input"""
    chain_parallel = RunnableParallel({
        "doble": RunnableLambda(lambda x: x * 2),
        "triple": RunnableLambda(lambda x: x * 3),
        "string": RunnableLambda(lambda x: str(x)),
    })
    resultado = chain_parallel.invoke(7)
    assert resultado["doble"] == 14, f"doble={resultado['doble']}, esperado 14"
    assert resultado["triple"] == 21, f"triple={resultado['triple']}, esperado 21"
    assert resultado["string"] == "7", f"string={resultado['string']}, esperado '7'"
    print(f"T4 PASS: parallel(7) = {resultado}")

def test_buffer_memory_guarda_historial():
    """T5: ConversationBufferMemory acumula mensajes"""
    mem = ConversationBufferMemory()
    assert len(mem.messages) == 0, "Debe iniciar vacío"

    mem.save_context("Hola", "Hola, ¿cómo estás?")
    mem.save_context("Bien, gracias", "Me alegra oírlo.")

    assert len(mem.messages) == 4, f"Deben ser 4 mensajes, got {len(mem.messages)}"
    historial = mem.load_memory_variables()["history"]
    assert "Hola" in historial, "El historial debe contener 'Hola'"
    assert "alegra" in historial, "El historial debe contener la respuesta AI"
    print(f"T5 PASS: {len(mem.messages)} mensajes en historial ({len(historial)} chars)")

def test_window_memory_limita_historial():
    """T6: ConversationBufferWindowMemory mantiene solo últimas k interacciones"""
    mem = ConversationBufferWindowMemory(k=2)
    intercambios = [
        ("Mensaje 1", "Respuesta 1"),
        ("Mensaje 2", "Respuesta 2"),
        ("Mensaje 3", "Respuesta 3"),
        ("Mensaje 4", "Respuesta 4"),  # Solo los últimos 2 deben aparecer
    ]
    for h, a in intercambios:
        mem.save_context(h, a)

    historial = mem.load_memory_variables()["history"]
    assert "Mensaje 1" not in historial, "Mensaje 1 debe haber sido descartado"
    assert "Mensaje 2" not in historial, "Mensaje 2 debe haber sido descartado"
    assert "Mensaje 3" in historial, "Mensaje 3 debe estar en la ventana k=2"
    assert "Mensaje 4" in historial, "Mensaje 4 debe estar en la ventana k=2"
    print(f"T6 PASS: ventana k=2 contiene mensajes 3 y 4 ({len(historial)} chars)")

if __name__ == "__main__":
    tests = [
        test_prompt_template_format,
        test_prompt_template_detecta_variables,
        test_lcel_pipeline,
        test_runnable_parallel,
        test_buffer_memory_guarda_historial,
        test_window_memory_limita_historial,
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
        print("M17 COMPLETADO")
```

---

## Resumen

| Componente | Descripción | Equivalente LCEL |
|---|---|---|
| `PromptTemplate` | Plantilla con variables | Primera etapa del pipe |
| `ChatPromptTemplate` | Mensajes system/human/ai | `from_messages([...])` |
| `LLMChain` | Prompt + LLM | `prompt | llm` |
| `SequentialChain` | Cadenas en serie | `chain1 | chain2` |
| `RunnableParallel` | Ramas en paralelo | `{k: runnable, ...}` |
| `ConversationChain` | Con memoria de historial | `prompt | llm + memory` |
| `AgentExecutor` | Bucle ReAct con tools | `agent | tools_loop` |

**LCEL (LangChain Expression Language):**
```python
# Composición de cadenas con operador |
chain = (
    {"contexto": retriever, "pregunta": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
respuesta = chain.invoke("¿Qué es el Transformer?")
```
