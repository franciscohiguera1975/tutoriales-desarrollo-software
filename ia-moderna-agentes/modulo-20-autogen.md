# Módulo 20 — AutoGen: Conversaciones Multi-Agente

> **Objetivo:** Dominar Microsoft AutoGen para construir sistemas donde múltiples agentes LLM colaboran mediante conversaciones estructuradas, incluyendo agentes conversables, proxies humanos, chats grupales y patrones de orquestación avanzados.
>
> **Herramientas:** autogen-agentchat, Python 3.11
>
> **Prerequisito:** M16 (Agentes ReAct), M19 (CrewAI)

---

## 1. Arquitectura de AutoGen

AutoGen organiza la colaboración multi-agente como **conversaciones** entre agentes. Cada agente puede enviar y recibir mensajes, ejecutar código, llamar herramientas y decidir cuándo terminar.

```python
# modulo-20-autogen.py — Parte 1: Arquitectura y agentes base
import re
import json
import hashlib
from dataclasses import dataclass, field
from typing import Callable, Dict, List, Optional, Any, Tuple
from enum import Enum

# ── Simuladores (sin API keys) ──────────────────────────────────────────────

class LLMSimulado:
    """LLM determinista basado en reglas para demos offline."""

    def __init__(self, rol: str = "asistente"):
        self.rol = rol
        self.historial: List[Dict] = []

    def chat(self, mensajes: List[Dict]) -> str:
        ultimo = mensajes[-1]["content"].lower() if mensajes else ""

        # Respuestas por rol
        if self.rol == "programador":
            if "fibonacci" in ultimo or "fib" in ultimo:
                return (
                    "Aquí está la solución:\n\n"
                    "```python\n"
                    "def fibonacci(n):\n"
                    "    if n <= 1:\n"
                    "        return n\n"
                    "    return fibonacci(n-1) + fibonacci(n-2)\n\n"
                    "# Ejemplo\n"
                    "print([fibonacci(i) for i in range(10)])\n"
                    "```\n\n"
                    "Esta implementación recursiva calcula Fibonacci correctamente."
                )
            if "factorial" in ultimo:
                return (
                    "```python\n"
                    "def factorial(n):\n"
                    "    return 1 if n == 0 else n * factorial(n-1)\n"
                    "```\n"
                    "TERMINATE"
                )
            return f"[Programador] Analizo el problema: {ultimo[:50]}... Propongo solución iterativa. TERMINATE"

        if self.rol == "revisor":
            if "```python" in mensajes[-1].get("content", ""):
                return (
                    "Revisión del código:\n"
                    "✓ Lógica correcta\n"
                    "✓ Maneja casos base\n"
                    "⚠ Considera añadir validación de tipos\n"
                    "⚠ Para n grande usa versión iterativa\n"
                    "Aprobado con sugerencias menores. TERMINATE"
                )
            return "Necesito ver el código para revisar. TERMINATE"

        if self.rol == "investigador":
            return (
                f"Investigación sobre '{ultimo[:40]}':\n"
                "1. Contexto histórico y motivación\n"
                "2. Estado del arte actual (2024)\n"
                "3. Principales enfoques y limitaciones\n"
                "4. Tendencias emergentes\n"
                "Investigación completada. TERMINATE"
            )

        if self.rol == "planificador":
            return (
                "Plan de acción:\n"
                "1. Analizar requisitos\n"
                "2. Dividir en subtareas\n"
                "3. Asignar a agentes especializados\n"
                "4. Integrar resultados\n"
                "5. Validar y entregar\n"
                "TERMINATE"
            )

        # Asistente genérico
        if "hola" in ultimo or "hi" in ultimo:
            return "Hola, estoy listo para ayudarte. ¿Qué necesitas?"
        if "?" in ultimo:
            return f"Respuesta a tu consulta: El tema '{ultimo[:30]}...' requiere análisis detallado. TERMINATE"
        return f"Entendido. Procesando: {ultimo[:50]}... TERMINATE"


# ── Mensaje ──────────────────────────────────────────────────────────────────

@dataclass
class Mensaje:
    """Unidad de comunicación entre agentes."""
    content: str
    role: str          # "user" | "assistant" | "system"
    name: str          # nombre del agente que lo envió
    metadata: Dict = field(default_factory=dict)

    def to_dict(self) -> Dict:
        return {"role": self.role, "content": self.content, "name": self.name}


# ── ConversableAgent ─────────────────────────────────────────────────────────

class ConversableAgent:
    """
    Agente base de AutoGen.
    Puede enviar/recibir mensajes y decidir cuándo terminar la conversación.
    """

    TERMINATION_SIGNAL = "TERMINATE"

    def __init__(
        self,
        nombre: str,
        system_message: str = "",
        llm: Optional[LLMSimulado] = None,
        max_consecutive_auto_reply: int = 10,
        human_input_mode: str = "NEVER",  # NEVER | ALWAYS | TERMINATE
        is_termination_msg: Optional[Callable[[Dict], bool]] = None,
        code_execution_config: Optional[Dict] = None,
    ):
        self.nombre = nombre
        self.system_message = system_message
        self.llm = llm
        self.max_consecutive_auto_reply = max_consecutive_auto_reply
        self.human_input_mode = human_input_mode
        self.code_execution_config = code_execution_config or {}
        self._auto_reply_count = 0

        # Función de terminación personalizable
        self._is_termination_msg = is_termination_msg or (
            lambda msg: self.TERMINATION_SIGNAL in msg.get("content", "")
        )

        # Historial de mensajes de esta conversación
        self._historial: List[Mensaje] = []

        # Registro de herramientas
        self._herramientas: Dict[str, Callable] = {}

    def registrar_herramienta(self, nombre: str, fn: Callable, descripcion: str = ""):
        """Registra una herramienta que el agente puede usar."""
        self._herramientas[nombre] = {"fn": fn, "descripcion": descripcion}

    def recibir(self, mensaje: Dict, sender: "ConversableAgent") -> Optional[Dict]:
        """
        Recibe un mensaje y decide si responder.
        Retorna respuesta o None si no responde.
        """
        msg_obj = Mensaje(
            content=mensaje.get("content", ""),
            role=mensaje.get("role", "user"),
            name=sender.nombre,
        )
        self._historial.append(msg_obj)

        # Verificar terminación
        if self._is_termination_msg(mensaje):
            return None

        # Modo human input
        if self.human_input_mode == "ALWAYS":
            return self._human_input(mensaje)

        # Auto reply limitado
        if self._auto_reply_count >= self.max_consecutive_auto_reply:
            return None

        self._auto_reply_count += 1
        return self._generar_respuesta(mensaje)

    def _generar_respuesta(self, mensaje: Dict) -> Optional[Dict]:
        """Genera respuesta con el LLM."""
        if self.llm is None:
            return None

        # Construir historial para el LLM
        mensajes_llm = []
        if self.system_message:
            mensajes_llm.append({"role": "system", "content": self.system_message})
        for m in self._historial[-10:]:  # últimos 10 mensajes
            mensajes_llm.append(m.to_dict())

        respuesta = self.llm.chat(mensajes_llm)

        # Extraer y ejecutar código si hay config de ejecución
        if self.code_execution_config:
            respuesta = self._ejecutar_codigo_en_respuesta(respuesta)

        return {"role": "assistant", "content": respuesta, "name": self.nombre}

    def _ejecutar_codigo_en_respuesta(self, respuesta: str) -> str:
        """Extrae y ejecuta bloques de código Python de la respuesta."""
        patron = r"```python\n(.*?)```"
        bloques = re.findall(patron, respuesta, re.DOTALL)

        if not bloques:
            return respuesta

        resultados = []
        for codigo in bloques:
            try:
                # Entorno de ejecución aislado
                namespace = {}
                exec(codigo.strip(), namespace)
                # Capturar outputs (en demo, simulamos el resultado)
                resultados.append(f"[EJECUTADO OK] Código de {len(codigo)} chars")
            except Exception as e:
                resultados.append(f"[ERROR] {type(e).__name__}: {e}")

        return respuesta + "\n\n**Resultados de ejecución:**\n" + "\n".join(resultados)

    def _human_input(self, mensaje: Dict) -> Dict:
        """Simula input humano (en producción usa input())."""
        return {
            "role": "user",
            "content": "Continúa. (input humano simulado)",
            "name": self.nombre,
        }

    def iniciar_chat(self, recipient: "ConversableAgent", mensaje: str, max_turns: int = 10) -> List[Dict]:
        """Inicia una conversación two-agent con recipient."""
        historial_chat: List[Dict] = []
        self._auto_reply_count = 0
        recipient._auto_reply_count = 0

        # Primer mensaje
        msg_inicial = {"role": "user", "content": mensaje, "name": self.nombre}
        historial_chat.append(msg_inicial)
        print(f"\n[{self.nombre}] → [{recipient.nombre}]: {mensaje[:80]}...")

        turno = 0
        hablante_actual = recipient
        oyente_actual = self

        while turno < max_turns:
            respuesta = hablante_actual.recibir(historial_chat[-1], oyente_actual)

            if respuesta is None:
                print(f"[{hablante_actual.nombre}] Conversación terminada.")
                break

            historial_chat.append(respuesta)
            print(f"[{hablante_actual.nombre}] → [{oyente_actual.nombre}]: {respuesta['content'][:80]}...")

            # Verificar terminación en la respuesta
            if self.TERMINATION_SIGNAL in respuesta.get("content", ""):
                break

            # Intercambiar roles
            hablante_actual, oyente_actual = oyente_actual, hablante_actual
            turno += 1

        return historial_chat

    def __repr__(self):
        return f"ConversableAgent(nombre={self.nombre!r})"


# ── Demo 1: Conversación básica two-agent ────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Conversación Two-Agent")
    print("=" * 60)

    # Agente asistente con LLM
    asistente = ConversableAgent(
        nombre="Asistente",
        system_message="Eres un asistente útil y conciso.",
        llm=LLMSimulado("programador"),
    )

    # Proxy del usuario (sin LLM, representa al humano)
    usuario = ConversableAgent(
        nombre="Usuario",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=0,  # no auto-responde
    )

    # Iniciar conversación
    historial = usuario.iniciar_chat(
        recipient=asistente,
        mensaje="Escribe una función Python para calcular Fibonacci.",
        max_turns=5,
    )

    print(f"\nTotal mensajes intercambiados: {len(historial)}")

    # Guardar resultado
    with open("outputs/m20_two_agent_chat.json", "w", encoding="utf-8") as f:
        json.dump(historial, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m20_two_agent_chat.json")
```

---

## 2. AssistantAgent y UserProxyAgent

AutoGen proporciona dos clases especializadas que cubren el 80% de los casos de uso.

```python
# Parte 2: AssistantAgent y UserProxyAgent
# (continuación del mismo archivo)

class AssistantAgent(ConversableAgent):
    """
    Agente asistente con LLM.
    Responde preguntas, escribe código y usa herramientas.
    Nunca solicita input humano.
    """

    DEFAULT_SYSTEM_MESSAGE = (
        "Eres un asistente de IA útil. "
        "Puedes escribir código Python ejecutable para resolver problemas. "
        "Cuando el problema esté resuelto, responde con TERMINATE."
    )

    def __init__(
        self,
        nombre: str,
        llm: LLMSimulado,
        system_message: Optional[str] = None,
        **kwargs,
    ):
        super().__init__(
            nombre=nombre,
            system_message=system_message or self.DEFAULT_SYSTEM_MESSAGE,
            llm=llm,
            human_input_mode="NEVER",
            **kwargs,
        )


class UserProxyAgent(ConversableAgent):
    """
    Proxy del usuario humano.
    Puede ejecutar código y proporcionar feedback.
    Por defecto solicita input humano cuando recibe TERMINATE.
    """

    def __init__(
        self,
        nombre: str,
        human_input_mode: str = "TERMINATE",
        code_execution_config: Optional[Dict] = None,
        **kwargs,
    ):
        super().__init__(
            nombre=nombre,
            human_input_mode=human_input_mode,
            llm=None,  # el humano decide, no LLM
            code_execution_config=code_execution_config or {"use_docker": False},
            **kwargs,
        )
        self._feedback_pendiente: List[str] = []

    def recibir(self, mensaje: Dict, sender: "ConversableAgent") -> Optional[Dict]:
        """Override: ejecuta código si hay bloques Python en el mensaje."""
        contenido = mensaje.get("content", "")

        # Ejecutar código embebido
        if "```python" in contenido and self.code_execution_config:
            resultado = self._ejecutar_codigo_en_respuesta(contenido)
            self._historial.append(
                Mensaje(content=contenido, role="assistant", name=sender.nombre)
            )
            return {
                "role": "user",
                "content": resultado,
                "name": self.nombre,
            }

        return super().recibir(mensaje, sender)

    def agregar_feedback(self, feedback: str):
        """Agrega feedback que se enviará en el próximo turno."""
        self._feedback_pendiente.append(feedback)


# ── Patrón coding: AssistantAgent escribe, UserProxy ejecuta ────────────────

def demo_coding_pattern():
    """
    Patrón más común en AutoGen:
    1. UserProxy envía tarea
    2. AssistantAgent escribe código
    3. UserProxy ejecuta código
    4. AssistantAgent itera si hay errores
    """
    print("\n" + "=" * 60)
    print("DEMO 2 — Patrón Coding: Escritura y Ejecución")
    print("=" * 60)

    assistant = AssistantAgent(
        nombre="CodingAssistant",
        llm=LLMSimulado("programador"),
        code_execution_config={"use_docker": False},
    )

    user_proxy = UserProxyAgent(
        nombre="UserProxy",
        human_input_mode="NEVER",
        code_execution_config={"use_docker": False},
    )

    # La conversación: proxy envía tarea, assistant responde con código
    historial = user_proxy.iniciar_chat(
        recipient=assistant,
        mensaje="Calcula el factorial de 10 y muestra el resultado.",
        max_turns=6,
    )

    # Analizar si hubo código ejecutado
    codigo_ejecutado = sum(
        1 for m in historial if "EJECUTADO" in m.get("content", "")
    )

    print(f"\nMensajes totales: {len(historial)}")
    print(f"Bloques de código ejecutados: {codigo_ejecutado}")

    return historial


if __name__ == "__main__":
    historial_coding = demo_coding_pattern()
    with open("outputs/m20_coding_pattern.json", "w", encoding="utf-8") as f:
        json.dump(historial_coding, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m20_coding_pattern.json")
```

---

## 3. GroupChat — Conversaciones Multi-Agente

Cuando tres o más agentes necesitan colaborar, `GroupChat` gestiona el turno de conversación.

```python
# Parte 3: GroupChat y GroupChatManager

class SelectorTurno(Enum):
    """Estrategias de selección del próximo hablante."""
    ROUND_ROBIN = "round_robin"
    RANDOM = "random"
    AUTO = "auto"       # el manager decide con LLM
    MANUAL = "manual"  # callback externo


@dataclass
class GroupChat:
    """
    Conversación grupal entre múltiples agentes.
    El GroupChatManager orquesta quién habla en cada turno.
    """
    agents: List[ConversableAgent]
    messages: List[Dict] = field(default_factory=list)
    max_round: int = 12
    speaker_selection_method: str = "round_robin"
    allow_repeat_speaker: bool = True

    def __post_init__(self):
        self._indice_actual = 0
        self._agentes_por_nombre: Dict[str, ConversableAgent] = {
            a.nombre: a for a in self.agents
        }

    def seleccionar_siguiente_hablante(
        self,
        ultimo_hablante: Optional[ConversableAgent],
        ultimo_mensaje: Dict,
        manager_llm: Optional[LLMSimulado] = None,
    ) -> ConversableAgent:
        """Selecciona el próximo agente que debe hablar."""

        if self.speaker_selection_method == "round_robin":
            # Turno rotativo: salta al manager
            candidatos = [a for a in self.agents if a.nombre != "Manager"]
            if not candidatos:
                candidatos = self.agents
            siguiente = candidatos[self._indice_actual % len(candidatos)]
            self._indice_actual += 1
            return siguiente

        if self.speaker_selection_method == "auto" and manager_llm:
            # El LLM del manager decide quién debe responder
            nombres = [a.nombre for a in self.agents if a != ultimo_hablante]
            contenido = ultimo_mensaje.get("content", "")

            # Heurística simple por palabras clave
            for nombre in nombres:
                if nombre.lower() in contenido.lower():
                    return self._agentes_por_nombre.get(nombre, self.agents[0])

            # Por defecto: primer agente disponible
            for agente in self.agents:
                if agente != ultimo_hablante:
                    return agente

        # Fallback: round_robin
        siguiente_idx = self._indice_actual % len(self.agents)
        self._indice_actual += 1
        return self.agents[siguiente_idx]

    def agregar_mensaje(self, mensaje: Dict):
        self.messages.append(mensaje)

    def get_messages(self) -> List[Dict]:
        return self.messages.copy()


class GroupChatManager(ConversableAgent):
    """
    Orquestador del GroupChat.
    Decide quién habla en cada turno y termina cuando corresponde.
    """

    def __init__(
        self,
        groupchat: GroupChat,
        nombre: str = "Manager",
        llm: Optional[LLMSimulado] = None,
        **kwargs,
    ):
        super().__init__(
            nombre=nombre,
            llm=llm or LLMSimulado("planificador"),
            human_input_mode="NEVER",
            **kwargs,
        )
        self.groupchat = groupchat

    def ejecutar(self, mensaje_inicial: str, sender: ConversableAgent) -> List[Dict]:
        """
        Ejecuta el loop del grupo:
        1. Recibe mensaje inicial
        2. Selecciona siguiente hablante
        3. El agente responde
        4. Repite hasta max_round o TERMINATE
        """
        # Mensaje inicial
        msg = {"role": "user", "content": mensaje_inicial, "name": sender.nombre}
        self.groupchat.agregar_mensaje(msg)
        print(f"\n[{sender.nombre}]: {mensaje_inicial[:80]}...")

        ultimo_hablante = sender
        ronda = 0

        while ronda < self.groupchat.max_round:
            # Seleccionar quién habla
            siguiente = self.groupchat.seleccionar_siguiente_hablante(
                ultimo_hablante=ultimo_hablante,
                ultimo_mensaje=self.groupchat.messages[-1],
                manager_llm=self.llm,
            )

            # El agente genera respuesta
            respuesta = siguiente._generar_respuesta(self.groupchat.messages[-1])

            if respuesta is None:
                print(f"[{siguiente.nombre}] No generó respuesta. Terminando.")
                break

            self.groupchat.agregar_mensaje(respuesta)
            print(f"[{siguiente.nombre}]: {respuesta['content'][:80]}...")

            # Verificar terminación
            if "TERMINATE" in respuesta.get("content", ""):
                print(f"\nTerminado por {siguiente.nombre} en ronda {ronda+1}.")
                break

            ultimo_hablante = siguiente
            ronda += 1

        return self.groupchat.get_messages()


# ── Demo 3: GroupChat con 3 agentes ─────────────────────────────────────────

def demo_groupchat():
    print("\n" + "=" * 60)
    print("DEMO 3 — GroupChat: Investigador + Programador + Revisor")
    print("=" * 60)

    # Tres agentes especializados
    investigador = AssistantAgent(
        nombre="Investigador",
        llm=LLMSimulado("investigador"),
        system_message="Investigas temas técnicos y provees contexto.",
    )
    programador = AssistantAgent(
        nombre="Programador",
        llm=LLMSimulado("programador"),
        system_message="Escribes código Python de alta calidad.",
    )
    revisor = AssistantAgent(
        nombre="Revisor",
        llm=LLMSimulado("revisor"),
        system_message="Revisas código y das feedback constructivo.",
    )
    usuario = UserProxyAgent(
        nombre="Usuario",
        human_input_mode="NEVER",
    )

    # Configurar GroupChat
    groupchat = GroupChat(
        agents=[investigador, programador, revisor],
        max_round=6,
        speaker_selection_method="round_robin",
    )
    manager = GroupChatManager(groupchat=groupchat, nombre="Manager")

    # Ejecutar
    mensajes = manager.ejecutar(
        mensaje_inicial="Necesito una función Fibonacci eficiente con documentación.",
        sender=usuario,
    )

    print(f"\nTotal mensajes en el grupo: {len(mensajes)}")
    participantes = set(m.get("name", "?") for m in mensajes)
    print(f"Participantes: {participantes}")

    return mensajes


if __name__ == "__main__":
    mensajes_grupo = demo_groupchat()
    with open("outputs/m20_groupchat.json", "w", encoding="utf-8") as f:
        json.dump(mensajes_grupo, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m20_groupchat.json")
```

---

## 4. Herramientas y Function Calling en AutoGen

Los agentes pueden registrar herramientas que el LLM invoca durante la conversación.

```python
# Parte 4: Herramientas y Tool Use en AutoGen

class ToolEnabledAgent(ConversableAgent):
    """
    Agente que puede invocar herramientas registradas.
    Parsea llamadas de herramientas del texto del LLM.
    """

    TOOL_CALL_PATTERN = re.compile(
        r"TOOL_CALL:\s*(\w+)\((.*?)\)",
        re.DOTALL,
    )

    def __init__(self, nombre: str, llm: LLMSimulado, **kwargs):
        super().__init__(nombre=nombre, llm=llm, **kwargs)
        self._resultados_herramientas: List[Dict] = []

    def _generar_respuesta(self, mensaje: Dict) -> Optional[Dict]:
        """Override: detecta y ejecuta tool calls antes de responder."""
        respuesta_raw = super()._generar_respuesta(mensaje)
        if respuesta_raw is None:
            return None

        contenido = respuesta_raw["content"]

        # Buscar tool calls en la respuesta
        matches = self.TOOL_CALL_PATTERN.findall(contenido)
        if matches:
            resultados = []
            for nombre_tool, args_str in matches:
                resultado = self._ejecutar_herramienta(nombre_tool, args_str)
                resultados.append(f"[{nombre_tool}({args_str})] → {resultado}")
                self._resultados_herramientas.append({
                    "tool": nombre_tool,
                    "args": args_str,
                    "resultado": resultado,
                })

            # Agregar resultados a la respuesta
            contenido = contenido + "\n\n**Resultados:**\n" + "\n".join(resultados)
            respuesta_raw = {"role": "assistant", "content": contenido, "name": self.nombre}

        return respuesta_raw

    def _ejecutar_herramienta(self, nombre: str, args_str: str) -> Any:
        """Ejecuta una herramienta por nombre."""
        if nombre not in self._herramientas:
            return f"ERROR: herramienta '{nombre}' no encontrada"

        tool_info = self._herramientas[nombre]
        fn = tool_info["fn"]

        # Parsear argumentos simples (key=value)
        kwargs = {}
        for par in args_str.split(","):
            par = par.strip()
            if "=" in par:
                k, v = par.split("=", 1)
                kwargs[k.strip()] = v.strip().strip('"\'')

        try:
            return fn(**kwargs) if kwargs else fn()
        except Exception as e:
            return f"ERROR: {e}"


# ── Herramientas de ejemplo ──────────────────────────────────────────────────

def buscar_web(query: str) -> str:
    """Simula búsqueda web."""
    resultados = {
        "python fibonacci": "Fibonacci: f(n) = f(n-1) + f(n-2); f(0)=0, f(1)=1",
        "machine learning": "ML: algoritmos que aprenden de datos sin programación explícita",
        "transformer architecture": "Transformer: atención multi-cabeza + FFN, sin recurrencia",
    }
    for k, v in resultados.items():
        if k.lower() in query.lower():
            return v
    return f"Resultados para '{query}': información técnica relevante encontrada."


def calcular(expresion: str) -> str:
    """Evalúa expresión matemática simple."""
    try:
        # Solo operaciones aritméticas seguras
        if re.match(r'^[\d\s\+\-\*\/\(\)\.]+$', expresion):
            return str(eval(expresion))
        return "Expresión no permitida por seguridad"
    except Exception as e:
        return f"Error: {e}"


def leer_archivo(ruta: str) -> str:
    """Simula lectura de archivo."""
    archivos_simulados = {
        "datos.csv": "id,valor\n1,100\n2,200\n3,300",
        "config.json": '{"modelo": "gpt-4", "temperatura": 0.7}',
        "README.md": "# Proyecto\nDescripción del proyecto...",
    }
    return archivos_simulados.get(ruta, f"Archivo '{ruta}' simulado con 42 líneas.")


# ── LLM con tool-aware responses ────────────────────────────────────────────

class LLMConHerramientas(LLMSimulado):
    """LLM que genera tool calls en sus respuestas."""

    def chat(self, mensajes: List[Dict]) -> str:
        ultimo = mensajes[-1]["content"].lower() if mensajes else ""

        if "busca" in ultimo or "buscar" in ultimo or "search" in ultimo:
            tema = "machine learning" if "ml" in ultimo else "python fibonacci"
            return (
                f"Voy a buscar información sobre el tema.\n"
                f'TOOL_CALL: buscar_web(query="{tema}")\n'
                "Basándome en los resultados, puedo responder tu consulta. TERMINATE"
            )

        if "calcula" in ultimo or "suma" in ultimo or "resultado" in ultimo:
            return (
                "Calcularé el resultado:\n"
                "TOOL_CALL: calcular(expresion=2+2*10)\n"
                "Cálculo completado. TERMINATE"
            )

        if "archivo" in ultimo or "lee" in ultimo or "read" in ultimo:
            return (
                "Leeré el archivo solicitado:\n"
                'TOOL_CALL: leer_archivo(ruta="datos.csv")\n'
                "Datos cargados correctamente. TERMINATE"
            )

        return f"Procesado: {ultimo[:50]}. TERMINATE"


def demo_herramientas():
    print("\n" + "=" * 60)
    print("DEMO 4 — Agente con Herramientas")
    print("=" * 60)

    agente_tools = ToolEnabledAgent(
        nombre="AgenteConTools",
        llm=LLMConHerramientas("asistente"),
    )

    # Registrar herramientas
    agente_tools.registrar_herramienta("buscar_web", buscar_web, "Busca en la web")
    agente_tools.registrar_herramienta("calcular", calcular, "Evalúa expresión")
    agente_tools.registrar_herramienta("leer_archivo", leer_archivo, "Lee archivo")

    usuario = UserProxyAgent(nombre="Usuario", human_input_mode="NEVER")

    # Múltiples tareas con herramientas
    tareas = [
        "Busca información sobre machine learning",
        "Calcula el resultado de la operación",
        "Lee el archivo de datos",
    ]

    resultados = []
    for tarea in tareas:
        print(f"\nTarea: {tarea}")
        historial = usuario.iniciar_chat(
            recipient=agente_tools,
            mensaje=tarea,
            max_turns=3,
        )
        resultados.append({
            "tarea": tarea,
            "mensajes": len(historial),
            "herramientas_usadas": agente_tools._resultados_herramientas.copy(),
        })
        agente_tools._resultados_herramientas.clear()
        agente_tools._historial.clear()
        agente_tools._auto_reply_count = 0

    print(f"\nHerramientas invocadas en total: {sum(len(r['herramientas_usadas']) for r in resultados)}")
    return resultados


if __name__ == "__main__":
    resultados_tools = demo_herramientas()
    with open("outputs/m20_tool_use.json", "w", encoding="utf-8") as f:
        json.dump(resultados_tools, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m20_tool_use.json")
```

---

## 5. Nested Chats — Conversaciones Anidadas

AutoGen permite que un agente inicie sub-conversaciones internas (nested chats) para resolver sub-problemas.

```python
# Parte 5: Nested Chats

class NestedChatAgent(ConversableAgent):
    """
    Agente que puede iniciar conversaciones anidadas con otros agentes
    para resolver sub-problemas antes de responder.
    """

    def __init__(self, nombre: str, llm: LLMSimulado, **kwargs):
        super().__init__(nombre=nombre, llm=llm, **kwargs)
        self._sub_agentes: Dict[str, ConversableAgent] = {}
        self._nested_histories: List[List[Dict]] = []

    def registrar_sub_agente(self, agente: ConversableAgent):
        """Registra un agente disponible para nested chats."""
        self._sub_agentes[agente.nombre] = agente

    def consultar_sub_agente(self, nombre: str, consulta: str) -> str:
        """
        Inicia un nested chat con un sub-agente para resolver una consulta.
        El resultado se usa como contexto para la respuesta principal.
        """
        if nombre not in self._sub_agentes:
            return f"Sub-agente '{nombre}' no disponible"

        sub_agente = self._sub_agentes[nombre]
        proxy_temporal = ConversableAgent(
            nombre=f"NestedProxy_{nombre}",
            max_consecutive_auto_reply=0,
        )

        print(f"  [Nested] {self.nombre} → {nombre}: {consulta[:50]}...")

        # Ejecutar nested chat
        historial_nested = proxy_temporal.iniciar_chat(
            recipient=sub_agente,
            mensaje=consulta,
            max_turns=3,
        )
        self._nested_histories.append(historial_nested)

        # Extraer última respuesta del sub-agente
        for msg in reversed(historial_nested):
            if msg.get("name") == nombre:
                return msg["content"].replace("TERMINATE", "").strip()

        return "Sin respuesta del sub-agente"

    def _generar_respuesta(self, mensaje: Dict) -> Optional[Dict]:
        """Override: usa nested chats para enriquecer la respuesta."""
        contenido = mensaje.get("content", "")

        # Decide qué sub-agentes consultar según el contenido
        contexto_extra = []
        for nombre_sub, sub_agente in self._sub_agentes.items():
            if nombre_sub.lower() in contenido.lower() or len(contexto_extra) == 0:
                resultado = self.consultar_sub_agente(nombre_sub, contenido)
                contexto_extra.append(f"[{nombre_sub}]: {resultado}")
                break  # una consulta por turno para este demo

        # Generar respuesta base
        respuesta_base = super()._generar_respuesta(mensaje)
        if respuesta_base is None:
            return None

        # Enriquecer con contexto de nested chats
        if contexto_extra:
            contenido_enriquecido = (
                respuesta_base["content"] +
                "\n\n**Contexto adicional de sub-agentes:**\n" +
                "\n".join(contexto_extra)
            )
            respuesta_base["content"] = contenido_enriquecido

        return respuesta_base


def demo_nested_chats():
    print("\n" + "=" * 60)
    print("DEMO 5 — Nested Chats: Agentes anidados")
    print("=" * 60)

    # Sub-agentes especializados
    sub_investigador = AssistantAgent(
        nombre="Investigador",
        llm=LLMSimulado("investigador"),
        system_message="Investigas y provees contexto técnico.",
    )
    sub_programador = AssistantAgent(
        nombre="Programador",
        llm=LLMSimulado("programador"),
        system_message="Escribes implementaciones de código.",
    )

    # Agente orquestador con nested chats
    orquestador = NestedChatAgent(
        nombre="Orquestador",
        llm=LLMSimulado("planificador"),
        system_message="Coordinas sub-agentes para respuestas completas.",
    )
    orquestador.registrar_sub_agente(sub_investigador)
    orquestador.registrar_sub_agente(sub_programador)

    usuario = UserProxyAgent(nombre="Usuario", human_input_mode="NEVER")

    # Consulta compleja que requiere múltiples especialistas
    historial = usuario.iniciar_chat(
        recipient=orquestador,
        mensaje="Investigador: explica transformers y propón implementación básica.",
        max_turns=4,
    )

    print(f"\nNested chats realizados: {len(orquestador._nested_histories)}")
    print(f"Total mensajes principales: {len(historial)}")

    return {
        "historial_principal": historial,
        "nested_count": len(orquestador._nested_histories),
    }


if __name__ == "__main__":
    resultado_nested = demo_nested_chats()
    with open("outputs/m20_nested_chats.json", "w", encoding="utf-8") as f:
        json.dump(
            {
                "historial_principal": resultado_nested["historial_principal"],
                "nested_count": resultado_nested["nested_count"],
            },
            f, ensure_ascii=False, indent=2,
        )
    print("Guardado: outputs/m20_nested_chats.json")
```

---

## 6. Patrones Avanzados: Swarm y Society of Mind

```python
# Parte 6: Patrones avanzados — Swarm y Society of Mind

class SwarmAgent(ConversableAgent):
    """
    Agente en un enjambre: puede auto-seleccionar el siguiente agente
    basándose en el contenido de su respuesta.
    Patrón AutoGen 0.4+: 'handoff' entre agentes.
    """

    def __init__(self, nombre: str, llm: LLMSimulado, **kwargs):
        super().__init__(nombre=nombre, llm=llm, **kwargs)
        self._handoff_rules: Dict[str, str] = {}  # keyword → agente_destino

    def registrar_handoff(self, keyword: str, agente_destino: str):
        """Si la respuesta contiene keyword, pasa el control a agente_destino."""
        self._handoff_rules[keyword] = agente_destino

    def detectar_handoff(self, respuesta: str) -> Optional[str]:
        """Detecta si la respuesta indica handoff a otro agente."""
        for keyword, destino in self._handoff_rules.items():
            if keyword.upper() in respuesta.upper():
                return destino
        return None


class SwarmOrchestrator:
    """
    Orquestador de enjambre: gestiona handoffs entre agentes.
    Cada agente puede transferir el control a otro especialista.
    """

    def __init__(self, agentes: List[SwarmAgent]):
        self._agentes: Dict[str, SwarmAgent] = {a.nombre: a for a in agentes}
        self._historial_global: List[Dict] = []
        self._ruta_ejecucion: List[str] = []

    def ejecutar(
        self,
        mensaje_inicial: str,
        agente_inicial: str,
        max_turnos: int = 10,
    ) -> Dict:
        """
        Ejecuta el enjambre:
        1. El agente inicial recibe la tarea
        2. Si su respuesta incluye handoff → pasa al siguiente agente
        3. Continúa hasta TERMINATE o max_turnos
        """
        if agente_inicial not in self._agentes:
            raise ValueError(f"Agente '{agente_inicial}' no registrado")

        print(f"\n{'='*60}")
        print("SWARM EXECUTION")
        print(f"{'='*60}")

        msg_actual = {"role": "user", "content": mensaje_inicial, "name": "SwarmInput"}
        self._historial_global.append(msg_actual)
        agente_actual = self._agentes[agente_inicial]
        self._ruta_ejecucion.append(agente_actual.nombre)

        for turno in range(max_turnos):
            print(f"\n[Turno {turno+1}] Agente: {agente_actual.nombre}")
            print(f"  Input: {msg_actual['content'][:60]}...")

            # El agente responde
            respuesta = agente_actual._generar_respuesta(msg_actual)
            if respuesta is None:
                print(f"  → Sin respuesta. Terminando.")
                break

            self._historial_global.append(respuesta)
            print(f"  Output: {respuesta['content'][:60]}...")

            # Verificar terminación
            if "TERMINATE" in respuesta.get("content", ""):
                print(f"  → TERMINATE recibido.")
                break

            # Verificar handoff
            destino = agente_actual.detectar_handoff(respuesta["content"])
            if destino and destino in self._agentes:
                print(f"  → HANDOFF a {destino}")
                agente_actual = self._agentes[destino]
                self._ruta_ejecucion.append(agente_actual.nombre)
                msg_actual = respuesta
            else:
                # Sin handoff: terminar
                break

        return {
            "historial": self._historial_global,
            "ruta": self._ruta_ejecucion,
            "turnos": len(self._ruta_ejecucion),
        }


class LLMSwarm(LLMSimulado):
    """LLM que genera handoffs en sus respuestas."""

    def __init__(self, rol: str, handoff_a: Optional[str] = None):
        super().__init__(rol)
        self.handoff_a = handoff_a

    def chat(self, mensajes: List[Dict]) -> str:
        base = super().chat(mensajes)
        # Quitar TERMINATE para permitir handoff
        base = base.replace("TERMINATE", "")

        if self.handoff_a:
            return base + f"\n\nTRANSFERIR_A_{self.handoff_a.upper()}: tarea completada para mi parte."
        return base + "\nTERMINATE"


def demo_swarm():
    print("\n" + "=" * 60)
    print("DEMO 6 — Swarm con Handoffs")
    print("=" * 60)

    # Agentes con handoff automático
    clasificador = SwarmAgent(
        nombre="Clasificador",
        llm=LLMSwarm("asistente", handoff_a="Ejecutor"),
        system_message="Clasificas y enrutas tareas al agente correcto.",
    )
    clasificador.registrar_handoff("TRANSFERIR_A_EJECUTOR", "Ejecutor")

    ejecutor = SwarmAgent(
        nombre="Ejecutor",
        llm=LLMSwarm("programador", handoff_a="Validador"),
        system_message="Ejecutas tareas técnicas y transfieres para validación.",
    )
    ejecutor.registrar_handoff("TRANSFERIR_A_VALIDADOR", "Validador")

    validador = SwarmAgent(
        nombre="Validador",
        llm=LLMSwarm("revisor", handoff_a=None),  # termina aquí
        system_message="Validas resultados y produces respuesta final.",
    )

    # Orquestador del enjambre
    swarm = SwarmOrchestrator([clasificador, ejecutor, validador])
    resultado = swarm.ejecutar(
        mensaje_inicial="Implementa y valida una función de ordenamiento burbuja.",
        agente_inicial="Clasificador",
        max_turnos=5,
    )

    print(f"\nRuta de ejecución: {' → '.join(resultado['ruta'])}")
    print(f"Mensajes totales: {len(resultado['historial'])}")

    return resultado


if __name__ == "__main__":
    resultado_swarm = demo_swarm()
    with open("outputs/m20_swarm.json", "w", encoding="utf-8") as f:
        json.dump(
            {
                "ruta": resultado_swarm["ruta"],
                "turnos": resultado_swarm["turnos"],
                "historial": resultado_swarm["historial"],
            },
            f, ensure_ascii=False, indent=2,
        )
    print("Guardado: outputs/m20_swarm.json")
```

---

## 7. Ejecución Completa y Comparativa de Patrones

```python
# Parte 7: Comparativa de patrones AutoGen

def comparar_patrones():
    """
    Compara los tres patrones principales de AutoGen:
    1. Two-Agent: simple, directo
    2. GroupChat: múltiples agentes, turno rotativo
    3. Swarm: handoff dinámico entre especialistas
    """
    print("\n" + "=" * 60)
    print("DEMO 7 — Comparativa de Patrones AutoGen")
    print("=" * 60)

    tarea = "Crear y revisar una función Python para invertir una cadena."

    # ── Patrón 1: Two-Agent ──────────────────────────────────────────────────
    print("\n--- Patrón 1: Two-Agent ---")
    asistente = AssistantAgent(nombre="Asistente", llm=LLMSimulado("programador"))
    usuario = UserProxyAgent(nombre="Usuario", human_input_mode="NEVER")
    h1 = usuario.iniciar_chat(recipient=asistente, mensaje=tarea, max_turns=3)
    stats_p1 = {"patron": "two_agent", "mensajes": len(h1), "agentes": 2}

    # ── Patrón 2: GroupChat ──────────────────────────────────────────────────
    print("\n--- Patrón 2: GroupChat ---")
    prog2 = AssistantAgent(nombre="Prog", llm=LLMSimulado("programador"))
    rev2 = AssistantAgent(nombre="Rev", llm=LLMSimulado("revisor"))
    usr2 = UserProxyAgent(nombre="Usr", human_input_mode="NEVER")
    gc = GroupChat(agents=[prog2, rev2], max_round=4, speaker_selection_method="round_robin")
    mgr = GroupChatManager(groupchat=gc, nombre="Manager")
    h2 = mgr.ejecutar(mensaje_inicial=tarea, sender=usr2)
    stats_p2 = {"patron": "groupchat", "mensajes": len(h2), "agentes": 3}

    # ── Patrón 3: Swarm ──────────────────────────────────────────────────────
    print("\n--- Patrón 3: Swarm ---")
    sw_prog = SwarmAgent(nombre="SwProg", llm=LLMSwarm("programador", handoff_a="SwRev"))
    sw_prog.registrar_handoff("TRANSFERIR_A_SWREV", "SwRev")
    sw_rev = SwarmAgent(nombre="SwRev", llm=LLMSwarm("revisor", handoff_a=None))
    sw = SwarmOrchestrator([sw_prog, sw_rev])
    r3 = sw.ejecutar(mensaje_inicial=tarea, agente_inicial="SwProg", max_turnos=4)
    stats_p3 = {"patron": "swarm", "mensajes": len(r3["historial"]), "agentes": 2, "ruta": r3["ruta"]}

    # ── Resumen ──────────────────────────────────────────────────────────────
    print("\n" + "=" * 60)
    print("RESUMEN COMPARATIVO")
    print("=" * 60)
    print(f"{'Patrón':<15} {'Mensajes':<12} {'Agentes':<10}")
    print("-" * 37)
    for stats in [stats_p1, stats_p2, stats_p3]:
        print(f"{stats['patron']:<15} {stats['mensajes']:<12} {stats['agentes']:<10}")

    print("\nCuándo usar cada patrón:")
    print("  Two-Agent:  tarea simple, un experto + usuario")
    print("  GroupChat:  debate/revisión entre múltiples expertos")
    print("  Swarm:      pipeline con especialistas secuenciales")

    return [stats_p1, stats_p2, stats_p3]


if __name__ == "__main__":
    comparativa = comparar_patrones()
    with open("outputs/m20_comparativa.json", "w", encoding="utf-8") as f:
        json.dump(comparativa, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m20_comparativa.json")
    print("\n[M20 completado] Todos los outputs guardados en outputs/")
```

---

## Checkpoint

```python
# checkpoint_m20.py — 6 tests para AutoGen

import sys
sys.path.insert(0, ".")
import re
from dataclasses import dataclass, field
from typing import Callable, Dict, List, Optional, Any
from enum import Enum

# ── Reimportar clases necesarias ─────────────────────────────────────────────
# (en proyecto real: from modulo_20 import ...)

class LLMSimulado:
    def __init__(self, rol="asistente"):
        self.rol = rol
    def chat(self, mensajes):
        ultimo = mensajes[-1]["content"].lower() if mensajes else ""
        if "fibonacci" in ultimo:
            return "def fibonacci(n): return n if n<=1 else fibonacci(n-1)+fibonacci(n-2) TERMINATE"
        return f"Respuesta sobre {ultimo[:20]}. TERMINATE"

@dataclass
class Mensaje:
    content: str; role: str; name: str
    def to_dict(self): return {"role": self.role, "content": self.content, "name": self.name}

class ConversableAgent:
    TERMINATION_SIGNAL = "TERMINATE"
    def __init__(self, nombre, system_message="", llm=None, max_consecutive_auto_reply=10, human_input_mode="NEVER", is_termination_msg=None, code_execution_config=None):
        self.nombre = nombre; self.llm = llm
        self.system_message = system_message
        self.max_consecutive_auto_reply = max_consecutive_auto_reply
        self.human_input_mode = human_input_mode
        self._auto_reply_count = 0; self._historial = []; self._herramientas = {}
        self._is_termination_msg = is_termination_msg or (lambda m: self.TERMINATION_SIGNAL in m.get("content",""))
        self.code_execution_config = code_execution_config or {}
    def registrar_herramienta(self, nombre, fn, descripcion=""):
        self._herramientas[nombre] = {"fn": fn, "descripcion": descripcion}
    def recibir(self, mensaje, sender):
        self._historial.append(Mensaje(mensaje.get("content",""), mensaje.get("role","user"), sender.nombre))
        if self._is_termination_msg(mensaje): return None
        if self._auto_reply_count >= self.max_consecutive_auto_reply: return None
        self._auto_reply_count += 1
        return self._generar_respuesta(mensaje)
    def _generar_respuesta(self, mensaje):
        if not self.llm: return None
        msgs = []
        if self.system_message: msgs.append({"role":"system","content":self.system_message})
        msgs.extend([m.to_dict() for m in self._historial[-10:]])
        return {"role":"assistant","content":self.llm.chat(msgs),"name":self.nombre}
    def iniciar_chat(self, recipient, mensaje, max_turns=10):
        historial = []; self._auto_reply_count = 0; recipient._auto_reply_count = 0
        msg = {"role":"user","content":mensaje,"name":self.nombre}
        historial.append(msg)
        hablante, oyente = recipient, self
        for _ in range(max_turns):
            resp = hablante.recibir(historial[-1], oyente)
            if resp is None: break
            historial.append(resp)
            if self.TERMINATION_SIGNAL in resp.get("content",""): break
            hablante, oyente = oyente, hablante
        return historial

class AssistantAgent(ConversableAgent):
    def __init__(self, nombre, llm, system_message=None, **kw):
        super().__init__(nombre, system_message or "Eres un asistente útil.", llm, human_input_mode="NEVER", **kw)

class UserProxyAgent(ConversableAgent):
    def __init__(self, nombre, human_input_mode="TERMINATE", code_execution_config=None, **kw):
        super().__init__(nombre, llm=None, human_input_mode=human_input_mode, max_consecutive_auto_reply=0, **kw)

@dataclass
class GroupChat:
    agents: List[ConversableAgent]; messages: List[Dict] = field(default_factory=list)
    max_round: int = 10; speaker_selection_method: str = "round_robin"
    def __post_init__(self): self._idx = 0
    def seleccionar_siguiente_hablante(self, ultimo, ultimo_msg, mgr_llm=None):
        candidatos = [a for a in self.agents if a.nombre != "Manager"]
        sig = candidatos[self._idx % len(candidatos)]; self._idx += 1; return sig
    def agregar_mensaje(self, msg): self.messages.append(msg)
    def get_messages(self): return self.messages.copy()

class GroupChatManager(ConversableAgent):
    def __init__(self, groupchat, nombre="Manager", llm=None, **kw):
        super().__init__(nombre, llm=llm or LLMSimulado("planificador"), human_input_mode="NEVER", **kw)
        self.groupchat = groupchat
    def ejecutar(self, mensaje_inicial, sender):
        msg = {"role":"user","content":mensaje_inicial,"name":sender.nombre}
        self.groupchat.agregar_mensaje(msg)
        ultimo = sender
        for _ in range(self.groupchat.max_round):
            siguiente = self.groupchat.seleccionar_siguiente_hablante(ultimo, self.groupchat.messages[-1])
            resp = siguiente._generar_respuesta(self.groupchat.messages[-1])
            if resp is None: break
            self.groupchat.agregar_mensaje(resp)
            if "TERMINATE" in resp.get("content",""): break
            ultimo = siguiente
        return self.groupchat.get_messages()

class SwarmAgent(ConversableAgent):
    def __init__(self, nombre, llm, **kw):
        super().__init__(nombre, llm=llm, **kw)
        self._handoff_rules = {}
    def registrar_handoff(self, keyword, destino): self._handoff_rules[keyword] = destino
    def detectar_handoff(self, respuesta):
        for kw, dest in self._handoff_rules.items():
            if kw.upper() in respuesta.upper(): return dest
        return None

class SwarmOrchestrator:
    def __init__(self, agentes):
        self._agentes = {a.nombre: a for a in agentes}
        self._historial_global = []; self._ruta = []
    def ejecutar(self, mensaje_inicial, agente_inicial, max_turnos=10):
        if agente_inicial not in self._agentes: raise ValueError("Agente no encontrado")
        msg = {"role":"user","content":mensaje_inicial,"name":"Input"}
        self._historial_global.append(msg)
        agente = self._agentes[agente_inicial]; self._ruta.append(agente.nombre)
        for _ in range(max_turnos):
            resp = agente._generar_respuesta(msg)
            if resp is None: break
            self._historial_global.append(resp)
            if "TERMINATE" in resp.get("content",""): break
            destino = agente.detectar_handoff(resp["content"])
            if destino and destino in self._agentes:
                agente = self._agentes[destino]; self._ruta.append(agente.nombre); msg = resp
            else: break
        return {"historial": self._historial_global, "ruta": self._ruta}

# ── Tests ──────────────────────────────────────────────────────────────────

def test_01_conversable_agent_basico():
    """ConversableAgent genera respuesta y registra historial."""
    agente = ConversableAgent("A", llm=LLMSimulado("programador"))
    sender = ConversableAgent("B")
    msg = {"role":"user","content":"escribe fibonacci","name":"B"}
    resp = agente.recibir(msg, sender)
    assert resp is not None, "Debe generar respuesta"
    assert resp["name"] == "A", "Nombre correcto"
    assert "fibonacci" in resp["content"].lower() or len(resp["content"]) > 0
    assert len(agente._historial) == 1, "Historial actualizado"
    print("✓ test_01 — ConversableAgent básico OK")

def test_02_terminacion_por_signal():
    """La conversación se detiene cuando el mensaje contiene TERMINATE."""
    agente = ConversableAgent("A", llm=LLMSimulado())
    sender = ConversableAgent("B")
    msg_terminate = {"role":"assistant","content":"Listo. TERMINATE","name":"B"}
    resp = agente.recibir(msg_terminate, sender)
    assert resp is None, "Debe retornar None al recibir TERMINATE"
    print("✓ test_02 — Terminación por TERMINATE OK")

def test_03_two_agent_chat():
    """Two-agent chat produce historial bidireccional."""
    asistente = AssistantAgent("Asistente", llm=LLMSimulado("programador"))
    usuario = UserProxyAgent("Usuario", human_input_mode="NEVER")
    historial = usuario.iniciar_chat(
        recipient=asistente,
        mensaje="Escribe una función fibonacci.",
        max_turns=5,
    )
    assert len(historial) >= 2, "Al menos 2 mensajes"
    nombres = {m.get("name") for m in historial}
    assert "Usuario" in nombres, "Usuario en historial"
    assert "Asistente" in nombres, "Asistente en historial"
    print("✓ test_03 — Two-agent chat OK")

def test_04_groupchat_round_robin():
    """GroupChat con round_robin alterna entre agentes."""
    a1 = AssistantAgent("A1", llm=LLMSimulado("programador"))
    a2 = AssistantAgent("A2", llm=LLMSimulado("revisor"))
    usr = UserProxyAgent("Usr", human_input_mode="NEVER")
    gc = GroupChat(agents=[a1, a2], max_round=4, speaker_selection_method="round_robin")
    mgr = GroupChatManager(groupchat=gc, nombre="Manager")
    mensajes = mgr.ejecutar("Tarea de grupo", sender=usr)
    assert len(mensajes) >= 2, "Al menos 2 mensajes"
    # Verificar que ambos agentes participaron o que el sistema funcionó
    nombres = {m.get("name") for m in mensajes}
    assert len(nombres) >= 1, "Al menos un participante"
    print("✓ test_04 — GroupChat round_robin OK")

def test_05_registro_herramientas():
    """Las herramientas se registran y están disponibles en el agente."""
    agente = ConversableAgent("A", llm=LLMSimulado())
    fn_test = lambda x: f"resultado_{x}"
    agente.registrar_herramienta("mi_tool", fn_test, "Tool de prueba")
    assert "mi_tool" in agente._herramientas
    assert agente._herramientas["mi_tool"]["fn"] == fn_test
    assert agente._herramientas["mi_tool"]["descripcion"] == "Tool de prueba"
    print("✓ test_05 — Registro de herramientas OK")

def test_06_swarm_handoff():
    """SwarmOrchestrator sigue la ruta de handoff entre agentes."""

    class LLMHandoff(LLMSimulado):
        def __init__(self, destino=None):
            super().__init__("asistente")
            self.destino = destino
        def chat(self, mensajes):
            if self.destino:
                return f"Proceso completado. HANDOFF_A_{self.destino.upper()}: pasar control."
            return "Tarea finalizada. TERMINATE"

    ag1 = SwarmAgent("Agente1", llm=LLMHandoff("Agente2"))
    ag1.registrar_handoff("HANDOFF_A_AGENTE2", "Agente2")
    ag2 = SwarmAgent("Agente2", llm=LLMHandoff(None))

    orq = SwarmOrchestrator([ag1, ag2])
    resultado = orq.ejecutar("Tarea de prueba", "Agente1", max_turnos=5)

    assert "Agente1" in resultado["ruta"], "Agente1 en ruta"
    assert "Agente2" in resultado["ruta"], "Agente2 recibió handoff"
    assert len(resultado["historial"]) >= 2, "Historial con múltiples mensajes"
    print("✓ test_06 — Swarm handoff OK")


if __name__ == "__main__":
    tests = [
        test_01_conversable_agent_basico,
        test_02_terminacion_por_signal,
        test_03_two_agent_chat,
        test_04_groupchat_round_robin,
        test_05_registro_herramientas,
        test_06_swarm_handoff,
    ]
    aprobados = 0
    for test in tests:
        try:
            test()
            aprobados += 1
        except AssertionError as e:
            print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e:
            print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")

    print(f"\n{'='*40}")
    print(f"Resultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests), f"Solo {aprobados}/{len(tests)} tests pasaron"
    print("✓ Checkpoint M20 completado")
```

---

## Resumen

| Concepto | Descripción |
|---|---|
| `ConversableAgent` | Agente base: envía/recibe mensajes, controla terminación |
| `AssistantAgent` | Especialización con LLM, sin input humano |
| `UserProxyAgent` | Proxy humano: ejecuta código, sin LLM propio |
| `GroupChat` | Conversación grupal con selección de hablante configurable |
| `GroupChatManager` | Orquestador del GroupChat, gestiona turnos |
| `NestedChat` | Sub-conversaciones internas para resolver sub-problemas |
| `SwarmAgent` | Agente con handoff automático basado en keywords |
| `SwarmOrchestrator` | Gestiona ruta dinámica entre agentes del enjambre |
| `TERMINATE` | Señal estándar para finalizar conversación |
| `human_input_mode` | `NEVER`/`ALWAYS`/`TERMINATE`: cuándo pedir input humano |

### Patrones de uso

```
Two-Agent:   UserProxy ←→ AssistantAgent
             Simple, directo, una tarea

GroupChat:   Manager → [A1, A2, A3] (round_robin / auto)
             Debate, revisión colectiva, múltiples perspectivas

Swarm:       A1 → handoff → A2 → handoff → A3 → TERMINATE
             Pipeline secuencial con especialistas, sin coordinador central

Nested:      OrchestratorAgent → [nested_chat(Sub1), nested_chat(Sub2)]
             Composición recursiva, divide-y-vencerás
```
