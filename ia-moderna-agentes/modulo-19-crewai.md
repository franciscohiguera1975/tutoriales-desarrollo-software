# Módulo 19 — CrewAI — Equipos de Agentes

**Objetivo:** Implementar sistemas multi-agente colaborativos con el paradigma CrewAI: agentes con roles especializados, tareas encadenadas, procesos secuenciales y jerárquicos, y flujos de trabajo con herramientas compartidas.

**Herramientas:** `crewai`, `numpy`, `json`

**Prerequisito:** M16 (Agentes ReAct)

---

## 1. Arquitectura CrewAI

```python
# crewai_arquitectura.py
import os

os.makedirs("outputs", exist_ok=True)

# CrewAI organiza el trabajo en:
#
#  Crew
#  ├── Agents (agentes con rol, objetivo, backstory)
#  │   ├── Agent 1: Investigador
#  │   ├── Agent 2: Escritor
#  │   └── Agent 3: Revisor
#  ├── Tasks (tareas con descripción, agente asignado, output esperado)
#  │   ├── Task 1: Investigar tema → Agent 1
#  │   ├── Task 2: Redactar artículo → Agent 2
#  │   └── Task 3: Revisar y corregir → Agent 3
#  └── Process
#      ├── Sequential: Task1 → Task2 → Task3
#      └── Hierarchical: Manager asigna tareas dinámicamente

COMPONENTES = {
    "Agent": "Entidad con rol, objetivo, backstory y herramientas",
    "Task": "Unidad de trabajo con descripción, agente y output esperado",
    "Crew": "Equipo de agentes que colaboran en un objetivo común",
    "Tool": "Función que un agente puede invocar para obtener información",
    "Process.sequential": "Las tareas se ejecutan en orden, output → input",
    "Process.hierarchical": "Manager LLM asigna y coordina tareas dinamicamente",
}

print("=== Componentes CrewAI ===")
for k, v in COMPONENTES.items():
    print(f"  {k}: {v}")

FLUJOS = {
    "Investigacion": "Investigar → Escribir → Revisar → Publicar",
    "Analisis_datos": "Recopilar → Limpiar → Analizar → Visualizar → Reportar",
    "Desarrollo_SW": "Planificar → Desarrollar → Testear → Documentar → Desplegar",
    "Atencion_cliente": "Clasificar → Resolver → Escalar → Responder → Seguir",
}

print("\n=== Flujos de trabajo típicos ===")
for k, v in FLUJOS.items():
    print(f"  {k}: {v}")

with open("outputs/m19_intro.txt", "w") as f:
    f.write("M19: CrewAI - Equipos de Agentes\n")
print("\noutputs/m19_intro.txt guardado")
```

---

## 2. Agent y Task — implementación desde cero

```python
# crewai_core.py
import os
import json
import time
from typing import List, Dict, Any, Callable, Optional
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

@dataclass
class Tool:
    """Herramienta disponible para los agentes."""
    name: str
    func: Callable
    description: str

    def run(self, input: str) -> str:
        try:
            return str(self.func(input))
        except Exception as e:
            return f"Error en {self.name}: {e}"

@dataclass
class Agent:
    """
    Agente con identidad, objetivo y capacidades.

    Parámetros clave:
    - role: título del agente (ej. "Investigador Senior")
    - goal: objetivo principal del agente
    - backstory: contexto y experiencia del agente (mejora la coherencia del LLM)
    - tools: lista de herramientas disponibles
    - llm: función LLM a usar
    - verbose: si mostrar razonamiento interno
    - max_iter: máximo de pasos ReAct por tarea
    """
    role: str
    goal: str
    backstory: str
    tools: List[Tool] = field(default_factory=list)
    llm: Callable = None
    verbose: bool = False
    max_iter: int = 5
    memory: bool = False
    _context: List[Dict] = field(default_factory=list, init=False)

    def __post_init__(self):
        if self.llm is None:
            self.llm = self._default_llm

    def _default_llm(self, prompt: str) -> str:
        return f"[{self.role}] Procesando: {prompt[:60]}..."

    def _build_system_prompt(self) -> str:
        tools_str = "\n".join([
            f"- {t.name}: {t.description}" for t in self.tools
        ])
        return f"""Eres {self.role}.
Objetivo: {self.goal}
Contexto: {self.backstory}

Herramientas disponibles:
{tools_str if tools_str else "Ninguna"}

Cuando uses una herramienta:
Action: nombre_herramienta
Action Input: el input para la herramienta

Cuando tengas la respuesta final:
Final Answer: [respuesta completa]"""

    def execute_task(self, task: 'Task',
                     context: str = "") -> str:
        """Ejecuta una tarea usando el bucle ReAct."""
        import re

        prompt_base = self._build_system_prompt()
        prompt = f"{prompt_base}\n\nTarea: {task.description}"
        if context:
            prompt += f"\n\nContexto previo:\n{context}"
        if task.expected_output:
            prompt += f"\n\nOutput esperado: {task.expected_output}"

        scratchpad = ""
        tools_dict = {t.name: t for t in self.tools}

        for step in range(self.max_iter):
            respuesta = self.llm(prompt + "\n" + scratchpad)

            if self.verbose:
                print(f"  [{self.role}] Paso {step+1}: {respuesta[:80]}")

            # Verificar Final Answer
            match_final = re.search(r'Final Answer:\s*(.+?)(?:\n\n|\Z)',
                                    respuesta, re.DOTALL)
            if match_final:
                return match_final.group(1).strip()

            # Ejecutar tool si existe
            match_action = re.search(r'Action:\s*(\w+)', respuesta)
            match_input = re.search(r'Action Input:\s*(.+?)(?:\n|$)', respuesta)

            if match_action:
                tool_name = match_action.group(1).strip()
                tool_input = match_input.group(1).strip() if match_input else ""

                if tool_name in tools_dict:
                    obs = tools_dict[tool_name].run(tool_input)
                else:
                    obs = f"Herramienta '{tool_name}' no disponible"

                scratchpad += f"{respuesta}\nObservation: {obs}\n"
            else:
                # Sin acción reconocida, tomar la respuesta como final
                return respuesta.strip()

        return f"[{self.role}] Tarea completada con información disponible."

@dataclass
class Task:
    """
    Tarea asignada a un agente con descripción y output esperado.
    """
    description: str
    agent: Agent
    expected_output: str = ""
    context: List['Task'] = field(default_factory=list)  # tareas previas como contexto
    output_file: str = ""
    async_execution: bool = False
    _output: str = field(default="", init=False)

    def execute(self, extra_context: str = "") -> str:
        """Ejecuta la tarea con el agente asignado."""
        # Recopilar contexto de tareas previas
        ctx_parts = []
        for prev_task in self.context:
            if prev_task._output:
                ctx_parts.append(f"[{prev_task.agent.role}]: {prev_task._output}")

        if extra_context:
            ctx_parts.append(extra_context)

        contexto_combinado = "\n\n".join(ctx_parts)
        self._output = self.agent.execute_task(self, contexto_combinado)

        if self.output_file:
            os.makedirs(os.path.dirname(self.output_file) or ".", exist_ok=True)
            with open(self.output_file, "w") as f:
                f.write(self._output)

        return self._output

class Process:
    SEQUENTIAL = "sequential"
    HIERARCHICAL = "hierarchical"

class Crew:
    """
    Equipo de agentes que colaboran para completar un conjunto de tareas.

    Proceso SEQUENTIAL:
    - Las tareas se ejecutan en orden
    - El output de cada tarea está disponible como contexto para las siguientes

    Proceso HIERARCHICAL:
    - Un agente Manager (LLM especial) coordina y asigna tareas dinámicamente
    - Más flexible pero más costoso en llamadas al LLM
    """
    def __init__(self, agents: List[Agent], tasks: List[Task],
                 process: str = Process.SEQUENTIAL,
                 manager_llm: Callable = None,
                 verbose: bool = False):
        self.agents = agents
        self.tasks = tasks
        self.process = process
        self.manager_llm = manager_llm
        self.verbose = verbose
        self._results: List[Dict] = []
        self._start_time: float = 0

    def kickoff(self, inputs: Dict = None) -> str:
        """Inicia la ejecución del crew."""
        self._start_time = time.time()
        inputs = inputs or {}

        if self.verbose:
            print(f"\n=== Crew iniciado ({self.process}) ===")
            print(f"  Agentes: {[a.role for a in self.agents]}")
            print(f"  Tareas: {len(self.tasks)}")

        if self.process == Process.SEQUENTIAL:
            return self._ejecutar_secuencial(inputs)
        else:
            return self._ejecutar_jerarquico(inputs)

    def _ejecutar_secuencial(self, inputs: Dict) -> str:
        """Ejecuta tareas en secuencia."""
        outputs = []
        for i, task in enumerate(self.tasks):
            if self.verbose:
                print(f"\n[Tarea {i+1}/{len(self.tasks)}] {task.agent.role}: {task.description[:50]}")

            # Reemplazar variables del input en la descripción
            desc_con_inputs = task.description
            for k, v in inputs.items():
                desc_con_inputs = desc_con_inputs.replace(f"{{{k}}}", str(v))
            task.description = desc_con_inputs

            output = task.execute()
            outputs.append({
                "tarea": i + 1,
                "agente": task.agent.role,
                "output": output
            })
            self._results.append(outputs[-1])

            if self.verbose:
                print(f"  Output: {output[:80]}...")

        elapsed = time.time() - self._start_time
        resultado_final = outputs[-1]["output"] if outputs else ""
        print(f"\nCrew completado en {elapsed:.2f}s")
        return resultado_final

    def _ejecutar_jerarquico(self, inputs: Dict) -> str:
        """Manager asigna tareas dinámicamente."""
        manager_llm = self.manager_llm or (lambda p: "Asignar tarea al primer agente disponible")
        outputs = []

        for task in self.tasks:
            # Manager decide qué agente asigna la tarea
            prompt_manager = f"""Como manager, asigna esta tarea al agente más adecuado.
Agentes disponibles: {[a.role for a in self.agents]}
Tarea: {task.description}
Responde solo con el rol del agente asignado:"""
            agente_asignado_rol = manager_llm(prompt_manager)

            # Buscar agente por rol (o usar el de la tarea)
            agente = task.agent
            for a in self.agents:
                if a.role.lower() in agente_asignado_rol.lower():
                    agente = a
                    break

            if self.verbose:
                print(f"\n[Manager] Asignando a {agente.role}: {task.description[:50]}")

            task.agent = agente
            output = task.execute()
            outputs.append({"tarea": task.description[:30], "agente": agente.role, "output": output})

        return outputs[-1]["output"] if outputs else ""

    def usage_metrics(self) -> Dict:
        return {
            "n_tasks": len(self._results),
            "agents_used": list(set(r["agente"] for r in self._results)),
            "elapsed_s": time.time() - self._start_time
        }

print("Crew framework definido.")
print("Componentes: Agent, Task, Crew, Process, Tool")

with open("outputs/m19_framework.py", "w") as f:
    f.write("# CrewAI framework\n# Ver modulo-19-crewai.md para implementacion completa\n")
print("outputs/m19_framework.py guardado")
```

---

## 3. Crew de Investigación y Escritura

```python
# crew_investigacion.py
import os
import json
import re
from typing import List, Callable
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

# Herramientas
def buscar_informacion(query: str) -> str:
    base = {
        "transformer": "El Transformer (2017) revolucionó el NLP con atención multi-cabeza. Permite procesamiento paralelo de secuencias.",
        "bert": "BERT (2018) usa encoders bidireccionales y MLM. Preentrenado por Google, 340M parámetros.",
        "gpt": "GPT usa decoders causales. GPT-3 (175B) y GPT-4 son los más capaces de OpenAI.",
        "llama": "LLaMA de Meta AI (2023). LLaMA 3 escala hasta 70B. Open-source, ideal para fine-tuning.",
        "rag": "RAG combina búsqueda vectorial con generación LLM. Reduce alucinaciones al anclar en documentos.",
    }
    q_lower = query.lower()
    for k, v in base.items():
        if k in q_lower:
            return v
    return f"Información sobre '{query}': tema de IA con desarrollos recientes."

def buscar_estadisticas(tema: str) -> str:
    stats = {
        "transformer": "Adoptado en 90% de modelos NLP (2024). Más de 5000 papers en 2023.",
        "llm": "GPT-4 procesa 32K tokens. Claude 3 alcanza 200K tokens de contexto.",
        "rag": "RAG reduce alucinaciones un 30-50%. Mejora precision en 20-30%.",
    }
    for k, v in stats.items():
        if k in tema.lower():
            return v
    return f"Estadísticas de {tema}: alta adopción en la industria."

def revisar_gramatica(texto: str) -> str:
    # Verificaciones básicas simuladas
    errores = []
    if len(texto.split(".")) < 2:
        errores.append("Texto muy corto: añadir más oraciones")
    if texto[0].islower():
        errores.append("El texto debe empezar con mayúscula")
    if not texto.endswith("."):
        errores.append("El texto debe terminar con punto")

    if errores:
        return f"Correcciones necesarias: {'; '.join(errores)}"
    return "Texto aprobado: buena gramática y estructura."

# LLMs especializados por rol
def llm_investigador(prompt: str) -> str:
    if "transformer" in prompt.lower():
        return "Final Answer: El Transformer fue introducido en 2017. Usa mecanismo de atención multi-cabeza que procesa secuencias en paralelo. Es la base de BERT, GPT, LLaMA y todos los modelos modernos."
    elif "bert" in prompt.lower():
        return "Final Answer: BERT (2018) de Google usa encoders bidireccionales preentrenados con MLM. Tiene 340M parámetros y marcó un antes y después en NLP."
    return f"Final Answer: Investigación completa sobre: {prompt[:50]}."

def llm_escritor(prompt: str) -> str:
    # Extraer el contenido del contexto previo
    if "Contexto previo" in prompt:
        ctx = prompt.split("Contexto previo:")[-1][:200]
        return f"Final Answer: Artículo: {ctx[:100].strip()}. Este tema es fundamental en la IA moderna y tiene amplias aplicaciones en procesamiento de lenguaje natural."
    return f"Final Answer: Artículo redactado sobre el tema asignado con información técnica detallada."

def llm_revisor(prompt: str) -> str:
    if "Final Answer" not in prompt:
        return "Final Answer: ✓ Artículo revisado y aprobado. Estructura clara, información precisa, sin errores gramaticales detectados."
    return "Final Answer: ✓ Revisión completada. Texto de alta calidad listo para publicación."

# Definir agentes con el framework
@dataclass
class Tool:
    name: str
    func: Callable
    description: str
    def run(self, input: str) -> str:
        try: return str(self.func(input))
        except Exception as e: return f"Error: {e}"

@dataclass
class Agent:
    role: str
    goal: str
    backstory: str
    tools: List[Tool] = field(default_factory=list)
    llm: Callable = None
    verbose: bool = True
    max_iter: int = 3
    _context: List = field(default_factory=list, init=False)

    def __post_init__(self):
        if self.llm is None:
            self.llm = lambda p: f"[{self.role}] {p[-40:]}"

    def execute_task(self, task, context="") -> str:
        tools_dict = {t.name: t for t in self.tools}
        prompt = f"Rol: {self.role}\nObjetivo: {self.goal}\nTarea: {task.description}"
        if context: prompt += f"\nContexto previo:\n{context}"

        for _ in range(self.max_iter):
            resp = self.llm(prompt)
            match_final = re.search(r'Final Answer:\s*(.+?)(?:\n\n|\Z)', resp, re.DOTALL)
            if match_final:
                return match_final.group(1).strip()
            match_act = re.search(r'Action:\s*(\w+)', resp)
            match_inp = re.search(r'Action Input:\s*(.+?)(?:\n|$)', resp)
            if match_act:
                tname = match_act.group(1).strip()
                tinput = match_inp.group(1).strip() if match_inp else ""
                obs = tools_dict[tname].run(tinput) if tname in tools_dict else f"Tool '{tname}' not found"
                prompt += f"\nObservation: {obs}"
            else:
                return resp.strip()
        return f"[{self.role}] Tarea completada."

@dataclass
class Task:
    description: str
    agent: Agent
    expected_output: str = ""
    context: List = field(default_factory=list)
    _output: str = field(default="", init=False)

    def execute(self, extra_context="") -> str:
        ctx_parts = [f"[{t.agent.role}]: {t._output}" for t in self.context if t._output]
        if extra_context: ctx_parts.append(extra_context)
        self._output = self.agent.execute_task(self, "\n\n".join(ctx_parts))
        return self._output

# --- Crew de investigación ---
investigador = Agent(
    role="Investigador Senior de IA",
    goal="Investigar y recopilar información técnica precisa sobre temas de IA",
    backstory="Experto en NLP con 10 años de experiencia en investigación académica.",
    tools=[
        Tool("buscar_informacion", buscar_informacion, "Busca información técnica"),
        Tool("buscar_estadisticas", buscar_estadisticas, "Busca estadísticas y datos"),
    ],
    llm=llm_investigador,
    verbose=True
)

escritor = Agent(
    role="Escritor Técnico",
    goal="Redactar artículos claros y precisos basados en la investigación",
    backstory="Periodista técnico especializado en IA con habilidad para simplificar conceptos.",
    llm=llm_escritor,
    verbose=True
)

revisor = Agent(
    role="Editor y Revisor",
    goal="Revisar y mejorar la calidad del contenido generado",
    backstory="Editor senior con ojo para la precisión técnica y claridad narrativa.",
    tools=[
        Tool("revisar_gramatica", revisar_gramatica, "Revisa gramática y estructura"),
    ],
    llm=llm_revisor,
    verbose=True
)

# Tareas
tarea_investigar = Task(
    description="Investiga y recopila información sobre el Transformer: historia, funcionamiento y relevancia actual.",
    agent=investigador,
    expected_output="Informe detallado con datos técnicos, estadísticas y aplicaciones."
)

tarea_escribir = Task(
    description="Escribe un artículo de 3 párrafos sobre el Transformer basándote en la investigación.",
    agent=escritor,
    expected_output="Artículo bien estructurado, claro y técnicamente preciso.",
    context=[tarea_investigar]  # Usa el output de investigación como contexto
)

tarea_revisar = Task(
    description="Revisa el artículo, corrígelo si es necesario y da el visto bueno final.",
    agent=revisor,
    expected_output="Artículo revisado y aprobado, o lista de correcciones necesarias.",
    context=[tarea_escribir]
)

# Ejecutar secuencialmente
print("=== Crew de Investigación ===\n")
outputs = {}
for i, tarea in enumerate([tarea_investigar, tarea_escribir, tarea_revisar], 1):
    print(f"[Tarea {i}] {tarea.agent.role}: {tarea.description[:60]}...")
    output = tarea.execute()
    outputs[f"tarea_{i}"] = {"agente": tarea.agent.role, "output": output[:150]}
    print(f"  Output: {output[:100]}...\n")

with open("outputs/m19_crew_investigacion.json", "w", encoding="utf-8") as f:
    json.dump(outputs, f, ensure_ascii=False, indent=2)
print("outputs/m19_crew_investigacion.json guardado")
```

---

## 4. Proceso Jerárquico con Manager

```python
# crew_jerarquico.py
import os
import json
import re
from typing import List, Callable, Dict
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

class ManagerLLM:
    """
    LLM que actúa como Manager en un proceso jerárquico.
    Decide qué agente es más adecuado para cada tarea.
    """
    def __init__(self, agentes_roles: List[str]):
        self.agentes_roles = agentes_roles

    def asignar(self, tarea: str) -> str:
        """Asigna la tarea al agente más adecuado basado en palabras clave."""
        tarea_lower = tarea.lower()

        if any(w in tarea_lower for w in ["investiga", "busca", "recopila", "analiza"]):
            return "Investigador"
        elif any(w in tarea_lower for w in ["escribe", "redacta", "genera", "crea texto"]):
            return "Escritor"
        elif any(w in tarea_lower for w in ["revisa", "corrige", "verifica", "evalúa"]):
            return "Revisor"
        elif any(w in tarea_lower for w in ["código", "programa", "implementa"]):
            return "Desarrollador"
        else:
            return self.agentes_roles[0]  # Default al primer agente

class CrewJerarquico:
    """
    Crew con proceso jerárquico:
    El Manager LLM asigna tareas dinámicamente a los agentes disponibles.
    """
    def __init__(self, agentes: Dict[str, Callable], manager: ManagerLLM):
        self.agentes = agentes
        self.manager = manager
        self.historial = []

    def ejecutar_tarea(self, descripcion: str,
                       contexto_previo: str = "") -> str:
        """El manager asigna y el agente ejecuta."""
        # Manager decide
        rol_asignado = self.manager.asignar(descripcion)
        agente_fn = self.agentes.get(rol_asignado,
                                      list(self.agentes.values())[0])

        print(f"  [Manager] → {rol_asignado}: {descripcion[:50]}")

        # Agente ejecuta
        prompt = f"Tarea: {descripcion}"
        if contexto_previo:
            prompt += f"\nContexto: {contexto_previo[:200]}"
        resultado = agente_fn(prompt)

        self.historial.append({
            "tarea": descripcion[:50],
            "agente": rol_asignado,
            "resultado": resultado[:100]
        })
        return resultado

    def kickoff(self, tareas: List[str]) -> str:
        print("=== Crew Jerárquico ===\n")
        contexto = ""
        ultimo_resultado = ""

        for tarea in tareas:
            ultimo_resultado = self.ejecutar_tarea(tarea, contexto)
            contexto = ultimo_resultado  # Output → contexto para siguiente tarea

        return ultimo_resultado

# Demo
def agente_investigador(p):
    return f"[Investigador] Investigación sobre: {p[:50]}. Hallazgos: BERT revolucionó NLP en 2018."

def agente_escritor(p):
    return f"[Escritor] Artículo: Las arquitecturas transformer han transformado el campo del NLP desde 2017."

def agente_revisor(p):
    return f"[Revisor] ✓ Contenido aprobado: {p[:60]}..."

def agente_dev(p):
    return f"[Dev] Código implementado para: {p[:40]}"

agentes = {
    "Investigador": agente_investigador,
    "Escritor": agente_escritor,
    "Revisor": agente_revisor,
    "Desarrollador": agente_dev,
}

manager = ManagerLLM(list(agentes.keys()))
crew = CrewJerarquico(agentes, manager)

tareas = [
    "Investiga las diferencias entre BERT y GPT",
    "Escribe un resumen de los hallazgos de la investigación",
    "Revisa y valida el resumen generado",
]

resultado = crew.kickoff(tareas)
print(f"\nResultado final: {resultado[:100]}")

with open("outputs/m19_crew_jerarquico.json", "w", encoding="utf-8") as f:
    json.dump(crew.historial, f, ensure_ascii=False, indent=2)
print("outputs/m19_crew_jerarquico.json guardado")
```

---

## 5. CrewAI con API real (referencia)

```python
# crewai_api_referencia.py
import os

os.makedirs("outputs", exist_ok=True)

CODIGO_CREWAI = '''
# Instalación: pip install crewai crewai-tools

from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, WebsiteSearchTool

# Herramientas
search_tool = SerperDevTool()  # Requiere SERPER_API_KEY
web_tool = WebsiteSearchTool()

# Agentes
investigador = Agent(
    role="Investigador Senior",
    goal="Investigar y encontrar información actualizada sobre {tema}",
    backstory="""Eres un investigador experto con acceso a las últimas
    publicaciones y noticias sobre IA.""",
    tools=[search_tool, web_tool],
    verbose=True,
    llm="gpt-4o"  # o "claude-3-5-sonnet-20241022"
)

escritor = Agent(
    role="Escritor Técnico",
    goal="Crear contenido técnico claro y preciso",
    backstory="Especialista en comunicación técnica con background en IA.",
    verbose=True,
    llm="gpt-4o"
)

# Tareas
tarea_investigar = Task(
    description="""Investiga los últimos avances en {tema}.
    Encuentra: tendencias principales, actores clave, aplicaciones reales.
    Usa las herramientas disponibles para buscar información actualizada.""",
    expected_output="Informe de 3 párrafos con hallazgos clave y fuentes.",
    agent=investigador,
    output_file="investigacion.md"
)

tarea_escribir = Task(
    description="""Basándote en la investigación, escribe un artículo técnico
    sobre {tema} para una audiencia de ingenieros de ML.""",
    expected_output="Artículo de 500 palabras con introducción, desarrollo y conclusión.",
    agent=escritor,
    context=[tarea_investigar],  # Usa output de investigación
    output_file="articulo_final.md"
)

# Crew
crew = Crew(
    agents=[investigador, escritor],
    tasks=[tarea_investigar, tarea_escribir],
    process=Process.sequential,
    verbose=True
)

# Kickoff con inputs
resultado = crew.kickoff(inputs={"tema": "RAG avanzado con LlamaIndex"})
print(resultado)

# Métricas de uso
print(crew.usage_metrics)
'''

with open("outputs/m19_crewai_api.py", "w") as f:
    f.write(CODIGO_CREWAI)
print("=== Código CrewAI con API real ===")
print(CODIGO_CREWAI[:400] + "...")
print("\noutputs/m19_crewai_api.py guardado")
```

---

## 6. Checkpoint — Pruebas unitarias

```python
# checkpoint_m19.py
"""
Checkpoint M19: CrewAI — Equipos de Agentes
Ejecutar con: python checkpoint_m19.py
"""
import os
import json
import re
from typing import List, Callable, Dict
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

@dataclass
class Tool:
    name: str
    func: Callable
    description: str
    def run(self, input: str) -> str:
        try: return str(self.func(input))
        except Exception as e: return f"Error: {e}"

@dataclass
class Agent:
    role: str
    goal: str
    backstory: str
    tools: List[Tool] = field(default_factory=list)
    llm: Callable = None
    verbose: bool = False
    max_iter: int = 3
    def __post_init__(self):
        if self.llm is None: self.llm = lambda p: f"Final Answer: [{self.role}] completado."
    def execute_task(self, task, context="") -> str:
        prompt = f"Tarea: {task.description}"
        if context: prompt += f"\nContexto: {context}"
        for _ in range(self.max_iter):
            resp = self.llm(prompt)
            match = re.search(r'Final Answer:\s*(.+?)(?:\n\n|\Z)', resp, re.DOTALL)
            if match: return match.group(1).strip()
            match_act = re.search(r'Action:\s*(\w+)', resp)
            match_inp = re.search(r'Action Input:\s*(.+?)(?:\n|$)', resp)
            tools_dict = {t.name: t for t in self.tools}
            if match_act:
                tn = match_act.group(1).strip()
                ti = match_inp.group(1).strip() if match_inp else ""
                obs = tools_dict[tn].run(ti) if tn in tools_dict else f"Tool '{tn}' not found"
                prompt += f"\nObservation: {obs}"
            else: return resp.strip()
        return f"[{self.role}] Completado."

@dataclass
class Task:
    description: str
    agent: Agent
    expected_output: str = ""
    context: List = field(default_factory=list)
    _output: str = field(default="", init=False)
    def execute(self, extra_context="") -> str:
        ctx_parts = [f"[{t.agent.role}]: {t._output}" for t in self.context if t._output]
        if extra_context: ctx_parts.append(extra_context)
        self._output = self.agent.execute_task(self, "\n\n".join(ctx_parts))
        return self._output

# Tests
def test_agent_execute_task():
    """T1: el agente ejecuta una tarea y retorna respuesta"""
    agente = Agent(
        role="TestAgent",
        goal="Completar tareas de prueba",
        backstory="Agente de prueba",
        llm=lambda p: "Final Answer: Tarea ejecutada correctamente."
    )
    tarea = Task(description="Prueba básica", agent=agente, expected_output="Confirmación")
    output = tarea.execute()
    assert output == "Tarea ejecutada correctamente.", f"Output incorrecto: {output}"
    print(f"T1 PASS: agente ejecuta tarea: '{output}'")

def test_task_context_pass():
    """T2: el output de tarea previa se pasa como contexto a la siguiente"""
    contexto_capturado = []

    def llm_con_contexto(prompt):
        if "Contexto" in prompt:
            contexto_capturado.append(prompt)
        return "Final Answer: Tarea con contexto completada."

    agente1 = Agent("A1", "Generar", "A1", llm=lambda p: "Final Answer: Output de A1.")
    agente2 = Agent("A2", "Procesar", "A2", llm=llm_con_contexto)

    tarea1 = Task("Primera tarea", agente1)
    tarea2 = Task("Segunda tarea (con contexto)", agente2, context=[tarea1])

    tarea1.execute()
    assert tarea1._output == "Output de A1.", f"Output tarea1: {tarea1._output}"
    tarea2.execute()
    assert len(contexto_capturado) > 0, "El contexto de tarea1 debe pasarse a tarea2"
    assert "Output de A1" in contexto_capturado[0], "El contexto debe contener output de A1"
    print(f"T2 PASS: contexto de tarea1 pasado correctamente a tarea2")

def test_herramienta_en_agente():
    """T3: el agente puede invocar herramientas"""
    resultados_tool = []

    def mi_tool(query):
        resultados_tool.append(query)
        return "Resultado de la herramienta"

    tool = Tool("mi_herramienta", mi_tool, "Herramienta de prueba")

    pasos = [
        "Action: mi_herramienta\nAction Input: prueba",
        "Final Answer: Herramienta ejecutada."
    ]
    paso_actual = [0]

    def llm_steps(p):
        resp = pasos[min(paso_actual[0], len(pasos)-1)]
        paso_actual[0] += 1
        return resp

    agente = Agent("ToolAgent", "Usar herramientas", "Con tools", tools=[tool], llm=llm_steps)
    tarea = Task("Usa la herramienta", agente)
    output = tarea.execute()
    assert len(resultados_tool) > 0, "La herramienta debe haber sido invocada"
    assert resultados_tool[0] == "prueba", f"Input de tool incorrecto: {resultados_tool[0]}"
    print(f"T3 PASS: herramienta invocada con input='{resultados_tool[0]}'")

def test_proceso_secuencial():
    """T4: proceso secuencial ejecuta tareas en orden"""
    orden_ejecucion = []

    def make_llm(id_):
        def llm(p):
            orden_ejecucion.append(id_)
            return f"Final Answer: Output {id_}"
        return llm

    agentes = [Agent(f"A{i}", f"Goal {i}", f"Back {i}", llm=make_llm(i)) for i in range(3)]
    tareas = [Task(f"Tarea {i}", agentes[i]) for i in range(3)]

    for tarea in tareas:
        tarea.execute()

    assert orden_ejecucion == [0, 1, 2], f"Orden incorrecto: {orden_ejecucion}"
    print(f"T4 PASS: tareas ejecutadas en orden {orden_ejecucion}")

def test_manager_asigna_agente_correcto():
    """T5: el manager asigna la tarea al agente más adecuado"""
    roles = ["Investigador", "Escritor", "Revisor", "Desarrollador"]

    def manager_asignar(tarea):
        t = tarea.lower()
        if any(w in t for w in ["investiga", "busca"]): return "Investigador"
        if any(w in t for w in ["escribe", "redacta"]): return "Escritor"
        if any(w in t for w in ["revisa", "corrige"]): return "Revisor"
        if any(w in t for w in ["código", "implementa"]): return "Desarrollador"
        return roles[0]

    casos = [
        ("Investiga los avances en NLP", "Investigador"),
        ("Escribe un artículo sobre IA", "Escritor"),
        ("Revisa el borrador del informe", "Revisor"),
        ("Implementa código en Python", "Desarrollador"),
    ]
    for tarea, rol_esperado in casos:
        asignado = manager_asignar(tarea)
        assert asignado == rol_esperado, f"Tarea '{tarea}': esperado '{rol_esperado}', got '{asignado}'"
    print(f"T5 PASS: manager asigna correctamente en {len(casos)} casos")

def test_crew_output_es_ultima_tarea():
    """T6: el output del crew es el output de la última tarea"""
    agente = Agent("Final", "Completar", "Último agente",
                   llm=lambda p: "Final Answer: Este es el resultado final del crew.")
    tareas = [
        Task("Tarea 1", agente),
        Task("Tarea 2", agente),
        Task("Tarea 3 (final)", agente),
    ]

    for t in tareas:
        t.execute()

    output_crew = tareas[-1]._output
    assert output_crew == "Este es el resultado final del crew.", \
        f"Output incorrecto: {output_crew}"
    assert tareas[0]._output != "", "Tarea 1 debe tener output"
    assert tareas[1]._output != "", "Tarea 2 debe tener output"
    print(f"T6 PASS: output del crew = '{output_crew[:50]}'")

if __name__ == "__main__":
    tests = [
        test_agent_execute_task,
        test_task_context_pass,
        test_herramienta_en_agente,
        test_proceso_secuencial,
        test_manager_asigna_agente_correcto,
        test_crew_output_es_ultima_tarea,
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
        print("M19 COMPLETADO")
```

---

## Resumen

| Componente | Descripción | Analogía |
|---|---|---|
| `Agent` | Entidad con rol + LLM + tools | Empleado especializado |
| `Task` | Unidad de trabajo asignada | Ticket de trabajo |
| `Crew` | Equipo de agentes colaborativos | Equipo de proyecto |
| `Process.sequential` | Tareas en cadena, A→B→C | Pipeline lineal |
| `Process.hierarchical` | Manager asigna dinámicamente | Organización con jefe |
| `Tool` | Función invocable por agente | Habilidad del empleado |

**Flujo CrewAI típico:**
```python
crew = Crew(
    agents=[investigador, escritor, revisor],
    tasks=[investigar, escribir, revisar],
    process=Process.sequential,
    verbose=True
)
resultado = crew.kickoff(inputs={"tema": "LLMs en producción"})
```

**Cuándo usar CrewAI:**
- Tareas que requieren múltiples especialistas
- Flujos de trabajo con etapas bien definidas
- Sistemas donde cada agente tiene herramientas específicas
- Procesamiento de documentos complejos (investigar → escribir → revisar)
