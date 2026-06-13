# Módulo 21 — MCP: Model Context Protocol — Arquitectura

> **Objetivo:** Comprender la arquitectura del Model Context Protocol (MCP): cómo conecta clientes LLM con servidores de herramientas mediante un protocolo estándar de mensajes JSON-RPC, habilitando extensibilidad sin modificar el modelo base.
>
> **Herramientas:** Python 3.11, json, asyncio (simulado)
>
> **Prerequisito:** M16 (Agentes ReAct), M20 (AutoGen)

---

## 1. Qué es MCP y por qué existe

MCP es un protocolo abierto (Anthropic, 2024) que define cómo un **cliente** (Claude, un IDE, un agente) se comunica con **servidores** que exponen herramientas, recursos y prompts. Sin MCP, cada integración requiere código ad-hoc; con MCP, cualquier cliente habla con cualquier servidor mediante el mismo protocolo.

```python
# modulo-21-mcp-arquitectura.py — Parte 1: Conceptos base

"""
Arquitectura MCP:

  ┌─────────────────────────────────────────────┐
  │              HOST (Claude, IDE…)            │
  │  ┌──────────┐    ┌──────────────────────┐  │
  │  │  Cliente │◄──►│   MCP Client Layer   │  │
  │  │   LLM    │    └──────────┬───────────┘  │
  └────────────────────────────│───────────────┘
                                │ JSON-RPC 2.0
                    ┌───────────▼───────────┐
                    │     MCP Server        │
                    │  ┌────┐ ┌──────────┐ │
                    │  │Tool│ │Resources │ │
                    │  └────┘ └──────────┘ │
                    └───────────────────────┘

Transporte: stdio | SSE (HTTP) | WebSocket
Protocolo:  JSON-RPC 2.0
Primitivas: Tools | Resources | Prompts | Sampling
"""

import json
import uuid
import hashlib
from dataclasses import dataclass, field, asdict
from typing import Any, Callable, Dict, List, Optional, Union
from enum import Enum


# ── Tipos de mensajes JSON-RPC 2.0 ───────────────────────────────────────────

class MCPMethod(str, Enum):
    """Métodos estándar del protocolo MCP."""
    # Ciclo de vida
    INITIALIZE          = "initialize"
    INITIALIZED         = "notifications/initialized"
    PING                = "ping"
    # Herramientas
    TOOLS_LIST          = "tools/list"
    TOOLS_CALL          = "tools/call"
    # Recursos
    RESOURCES_LIST      = "resources/list"
    RESOURCES_READ      = "resources/read"
    RESOURCES_SUBSCRIBE = "resources/subscribe"
    # Prompts
    PROMPTS_LIST        = "prompts/list"
    PROMPTS_GET         = "prompts/get"
    # Sampling (servidor pide al cliente generar texto)
    SAMPLING_CREATE     = "sampling/createMessage"
    # Logging
    LOGGING_MESSAGE     = "notifications/message"


@dataclass
class JSONRPCRequest:
    """Mensaje de solicitud JSON-RPC 2.0."""
    method: str
    params: Dict = field(default_factory=dict)
    id: Optional[str] = None
    jsonrpc: str = "2.0"

    def __post_init__(self):
        if self.id is None:
            self.id = str(uuid.uuid4())[:8]

    def to_dict(self) -> Dict:
        return {
            "jsonrpc": self.jsonrpc,
            "id": self.id,
            "method": self.method,
            "params": self.params,
        }

    def serialize(self) -> str:
        return json.dumps(self.to_dict())


@dataclass
class JSONRPCResponse:
    """Respuesta JSON-RPC 2.0 (éxito)."""
    id: str
    result: Any
    jsonrpc: str = "2.0"

    def to_dict(self) -> Dict:
        return {"jsonrpc": self.jsonrpc, "id": self.id, "result": self.result}

    def serialize(self) -> str:
        return json.dumps(self.to_dict())


@dataclass
class JSONRPCError:
    """Respuesta JSON-RPC 2.0 (error)."""
    id: str
    code: int
    message: str
    data: Any = None
    jsonrpc: str = "2.0"

    def to_dict(self) -> Dict:
        error: Dict[str, Any] = {"code": self.code, "message": self.message}
        if self.data is not None:
            error["data"] = self.data
        return {"jsonrpc": self.jsonrpc, "id": self.id, "error": error}

    def serialize(self) -> str:
        return json.dumps(self.to_dict())


@dataclass
class JSONRPCNotification:
    """Notificación JSON-RPC 2.0 (sin id, sin respuesta esperada)."""
    method: str
    params: Dict = field(default_factory=dict)
    jsonrpc: str = "2.0"

    def to_dict(self) -> Dict:
        return {"jsonrpc": self.jsonrpc, "method": self.method, "params": self.params}


# Códigos de error estándar MCP
class ErrorCode(int, Enum):
    PARSE_ERROR      = -32700
    INVALID_REQUEST  = -32600
    METHOD_NOT_FOUND = -32601
    INVALID_PARAMS   = -32602
    INTERNAL_ERROR   = -32603
    # MCP específicos
    RESOURCE_NOT_FOUND = -32002
    TOOL_NOT_FOUND     = -32003


# ── Demo: serialización de mensajes ─────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Mensajes JSON-RPC 2.0")
    print("=" * 60)

    # Request
    req = JSONRPCRequest(
        method=MCPMethod.TOOLS_LIST,
        params={},
    )
    print(f"Request:\n{json.dumps(req.to_dict(), indent=2)}\n")

    # Response
    resp = JSONRPCResponse(
        id=req.id,
        result={"tools": [{"name": "buscar_web", "description": "Busca en internet"}]},
    )
    print(f"Response:\n{json.dumps(resp.to_dict(), indent=2)}\n")

    # Error
    err = JSONRPCError(
        id=req.id,
        code=ErrorCode.METHOD_NOT_FOUND,
        message="Método no encontrado",
    )
    print(f"Error:\n{json.dumps(err.to_dict(), indent=2)}")

    with open("outputs/m21_jsonrpc_ejemplos.json", "w", encoding="utf-8") as f:
        json.dump({
            "request": req.to_dict(),
            "response": resp.to_dict(),
            "error": err.to_dict(),
        }, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m21_jsonrpc_ejemplos.json")
```

---

## 2. Primitivas MCP: Tools, Resources y Prompts

```python
# Parte 2: Las tres primitivas principales de MCP

# ── Tool ─────────────────────────────────────────────────────────────────────

@dataclass
class MCPToolParameter:
    """Parámetro de una herramienta MCP (JSON Schema)."""
    name: str
    type: str                          # "string" | "number" | "boolean" | "object"
    description: str
    required: bool = True
    enum: Optional[List[str]] = None  # valores permitidos

    def to_schema(self) -> Dict:
        schema: Dict[str, Any] = {"type": self.type, "description": self.description}
        if self.enum:
            schema["enum"] = self.enum
        return schema


@dataclass
class MCPTool:
    """
    Herramienta registrada en un servidor MCP.
    El cliente LLM puede invocarla pasando sus parámetros.
    """
    name: str
    description: str
    parameters: List[MCPToolParameter]
    fn: Callable                        # implementación
    annotations: Dict = field(default_factory=dict)  # readOnly, destructive, etc.

    def input_schema(self) -> Dict:
        """Genera el JSON Schema de inputs para el LLM."""
        props = {p.name: p.to_schema() for p in self.parameters}
        required = [p.name for p in self.parameters if p.required]
        return {
            "type": "object",
            "properties": props,
            "required": required,
        }

    def to_dict(self) -> Dict:
        """Representación para tools/list."""
        result: Dict[str, Any] = {
            "name": self.name,
            "description": self.description,
            "inputSchema": self.input_schema(),
        }
        if self.annotations:
            result["annotations"] = self.annotations
        return result

    def call(self, arguments: Dict) -> Any:
        """Ejecuta la herramienta con los argumentos dados."""
        return self.fn(**arguments)


# ── Resource ──────────────────────────────────────────────────────────────────

@dataclass
class MCPResource:
    """
    Recurso expuesto por un servidor MCP.
    Los recursos son datos que el cliente puede leer (archivos, BD, APIs…).
    """
    uri: str           # Identificador único: "file:///path" | "db://tabla" | "api://endpoint"
    name: str
    description: str
    mime_type: str = "text/plain"
    content: str = ""  # contenido (en demo, estático)
    annotations: Dict = field(default_factory=dict)

    def to_dict(self) -> Dict:
        return {
            "uri": self.uri,
            "name": self.name,
            "description": self.description,
            "mimeType": self.mime_type,
        }

    def read(self) -> Dict:
        """Retorna el contenido del recurso."""
        return {
            "uri": self.uri,
            "mimeType": self.mime_type,
            "text": self.content,
        }


# ── Prompt ────────────────────────────────────────────────────────────────────

@dataclass
class MCPPromptArgument:
    """Argumento de un prompt MCP."""
    name: str
    description: str
    required: bool = True


@dataclass
class MCPPrompt:
    """
    Template de prompt registrado en un servidor MCP.
    Permite compartir prompts reutilizables entre clientes.
    """
    name: str
    description: str
    arguments: List[MCPPromptArgument]
    template: str  # template con {placeholders}

    def to_dict(self) -> Dict:
        return {
            "name": self.name,
            "description": self.description,
            "arguments": [
                {"name": a.name, "description": a.description, "required": a.required}
                for a in self.arguments
            ],
        }

    def render(self, **kwargs) -> str:
        """Renderiza el template con los argumentos dados."""
        try:
            return self.template.format(**kwargs)
        except KeyError as e:
            raise ValueError(f"Argumento requerido no provisto: {e}")


# ── Demo: definir herramientas, recursos y prompts ───────────────────────────

def demo_primitivas():
    print("\n" + "=" * 60)
    print("DEMO 2 — Primitivas MCP")
    print("=" * 60)

    # Tool: calculadora
    tool_calc = MCPTool(
        name="calcular",
        description="Evalúa una expresión matemática simple",
        parameters=[
            MCPToolParameter("expresion", "string", "Expresión a evaluar, ej: 2+2*10"),
        ],
        fn=lambda expresion: str(eval(expresion)) if all(c in "0123456789+-*/(). " for c in expresion) else "Error",
        annotations={"readOnlyHint": True},
    )

    # Tool: clima simulado
    tool_clima = MCPTool(
        name="obtener_clima",
        description="Obtiene el clima actual de una ciudad",
        parameters=[
            MCPToolParameter("ciudad", "string", "Nombre de la ciudad"),
            MCPToolParameter("unidad", "string", "Unidad de temperatura", required=False, enum=["celsius", "fahrenheit"]),
        ],
        fn=lambda ciudad, unidad="celsius": f"Clima en {ciudad}: 22°{'C' if unidad=='celsius' else 'F'}, soleado",
    )

    # Resource
    resource_docs = MCPResource(
        uri="file:///docs/manual.txt",
        name="Manual de usuario",
        description="Documentación completa del sistema",
        mime_type="text/plain",
        content="# Manual\n\n## Sección 1\nContenido del manual...\n\n## Sección 2\nMás información.",
    )

    # Prompt
    prompt_analisis = MCPPrompt(
        name="analizar_codigo",
        description="Analiza código Python y sugiere mejoras",
        arguments=[
            MCPPromptArgument("codigo", "Código Python a analizar"),
            MCPPromptArgument("enfoque", "Aspecto a priorizar: rendimiento|legibilidad|seguridad", required=False),
        ],
        template=(
            "Analiza el siguiente código Python:\n\n"
            "```python\n{codigo}\n```\n\n"
            "Enfoque: {enfoque}\n\n"
            "Proporciona:\n1. Análisis de problemas\n2. Sugerencias de mejora\n3. Código refactorizado"
        ),
    )

    # Mostrar schemas
    print("Tool — input schema:")
    print(json.dumps(tool_calc.to_dict(), indent=2))

    print("\nResource — metadata:")
    print(json.dumps(resource_docs.to_dict(), indent=2))

    print("\nPrompt renderizado:")
    prompt_rendered = prompt_analisis.render(
        codigo="def suma(a,b): return a+b",
        enfoque="legibilidad",
    )
    print(prompt_rendered[:200] + "...")

    return {
        "tools": [tool_calc.to_dict(), tool_clima.to_dict()],
        "resources": [resource_docs.to_dict()],
        "prompts": [prompt_analisis.to_dict()],
    }


if __name__ == "__main__":
    primitivas = demo_primitivas()
    with open("outputs/m21_primitivas.json", "w", encoding="utf-8") as f:
        json.dump(primitivas, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m21_primitivas.json")
```

---

## 3. Ciclo de Vida de la Conexión MCP

```python
# Parte 3: Handshake y ciclo de vida

@dataclass
class MCPClientInfo:
    name: str
    version: str

@dataclass
class MCPServerInfo:
    name: str
    version: str

@dataclass
class MCPCapabilities:
    """
    Capacidades que cliente y servidor intercambian durante initialize.
    Permite negociación de features opcionales.
    """
    tools: bool = True
    resources: bool = True
    prompts: bool = True
    sampling: bool = False    # el servidor puede pedir al cliente que genere texto
    logging: bool = True
    experimental: Dict = field(default_factory=dict)

    def to_dict(self) -> Dict:
        caps: Dict[str, Any] = {}
        if self.tools:     caps["tools"]     = {}
        if self.resources: caps["resources"] = {"subscribe": True}
        if self.prompts:   caps["prompts"]   = {}
        if self.sampling:  caps["sampling"]  = {}
        if self.logging:   caps["logging"]   = {}
        if self.experimental:
            caps["experimental"] = self.experimental
        return caps


class ConnectionState(str, Enum):
    DISCONNECTED = "disconnected"
    CONNECTING   = "connecting"
    CONNECTED    = "connected"
    ERROR        = "error"


class MCPConnection:
    """
    Simula el ciclo de vida de una conexión MCP:
    DISCONNECTED → CONNECTING → CONNECTED → DISCONNECTED

    En producción usa transporte stdio o SSE; aquí usamos paso directo.
    """

    def __init__(self, client_info: MCPClientInfo, server: "MCPServer"):
        self.client_info = client_info
        self.server = server
        self.state = ConnectionState.DISCONNECTED
        self.server_info: Optional[MCPServerInfo] = None
        self.server_capabilities: Optional[MCPCapabilities] = None
        self._message_log: List[Dict] = []

    def _send_request(self, request: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        """Envía request al servidor y obtiene respuesta."""
        self._message_log.append({"direction": "→ servidor", "msg": request.to_dict()})
        response = self.server.handle_request(request)
        self._message_log.append({"direction": "← servidor", "msg": response.to_dict()})
        return response

    def connect(self) -> bool:
        """
        Fase 1 — Handshake:
        cliente → initialize → servidor
        servidor → result(serverInfo, capabilities) → cliente
        cliente → notifications/initialized → servidor
        """
        self.state = ConnectionState.CONNECTING
        print(f"\n[MCP] Conectando {self.client_info.name} → {self.server.info.name}...")

        # Paso 1: initialize
        init_req = JSONRPCRequest(
            method=MCPMethod.INITIALIZE,
            params={
                "protocolVersion": "2024-11-05",
                "clientInfo": {"name": self.client_info.name, "version": self.client_info.version},
                "capabilities": MCPCapabilities(sampling=False).to_dict(),
            },
        )
        response = self._send_request(init_req)

        if isinstance(response, JSONRPCError):
            self.state = ConnectionState.ERROR
            print(f"[MCP] Error en initialize: {response.message}")
            return False

        # Extraer info del servidor
        result = response.result
        self.server_info = MCPServerInfo(
            name=result["serverInfo"]["name"],
            version=result["serverInfo"]["version"],
        )
        print(f"[MCP] Servidor: {self.server_info.name} v{self.server_info.version}")
        print(f"[MCP] Capacidades: {list(result.get('capabilities', {}).keys())}")

        # Paso 2: notificar que el cliente está listo
        notif = JSONRPCNotification(method=MCPMethod.INITIALIZED)
        self.server.handle_notification(notif)

        self.state = ConnectionState.CONNECTED
        print("[MCP] Conexión establecida ✓")
        return True

    def call_tool(self, name: str, arguments: Dict) -> Any:
        """Llama una herramienta en el servidor."""
        if self.state != ConnectionState.CONNECTED:
            raise RuntimeError("No conectado")

        req = JSONRPCRequest(
            method=MCPMethod.TOOLS_CALL,
            params={"name": name, "arguments": arguments},
        )
        response = self._send_request(req)

        if isinstance(response, JSONRPCError):
            raise RuntimeError(f"Error en tool call: {response.message}")

        return response.result

    def list_tools(self) -> List[Dict]:
        """Lista las herramientas disponibles."""
        req = JSONRPCRequest(method=MCPMethod.TOOLS_LIST, params={})
        response = self._send_request(req)
        if isinstance(response, JSONRPCError):
            return []
        return response.result.get("tools", [])

    def read_resource(self, uri: str) -> Dict:
        """Lee un recurso del servidor."""
        req = JSONRPCRequest(
            method=MCPMethod.RESOURCES_READ,
            params={"uri": uri},
        )
        response = self._send_request(req)
        if isinstance(response, JSONRPCError):
            raise RuntimeError(f"Error leyendo recurso: {response.message}")
        return response.result

    def disconnect(self):
        self.state = ConnectionState.DISCONNECTED
        print(f"[MCP] Desconectado de {self.server_info.name if self.server_info else '?'}")


# ── Servidor MCP mínimo para el demo ─────────────────────────────────────────

class MCPServer:
    """Implementación mínima de servidor MCP."""

    def __init__(self, name: str, version: str = "1.0.0"):
        self.info = MCPServerInfo(name, version)
        self.capabilities = MCPCapabilities()
        self._tools: Dict[str, MCPTool] = {}
        self._resources: Dict[str, MCPResource] = {}
        self._prompts: Dict[str, MCPPrompt] = {}
        self._initialized = False

    def register_tool(self, tool: MCPTool):
        self._tools[tool.name] = tool

    def register_resource(self, resource: MCPResource):
        self._resources[resource.uri] = resource

    def register_prompt(self, prompt: MCPPrompt):
        self._prompts[prompt.name] = prompt

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        """Router principal de mensajes."""
        method = req.method

        if method == MCPMethod.INITIALIZE:
            return JSONRPCResponse(
                id=req.id,
                result={
                    "protocolVersion": "2024-11-05",
                    "serverInfo": {"name": self.info.name, "version": self.info.version},
                    "capabilities": self.capabilities.to_dict(),
                },
            )

        if method == MCPMethod.PING:
            return JSONRPCResponse(id=req.id, result={})

        if method == MCPMethod.TOOLS_LIST:
            return JSONRPCResponse(
                id=req.id,
                result={"tools": [t.to_dict() for t in self._tools.values()]},
            )

        if method == MCPMethod.TOOLS_CALL:
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                return JSONRPCError(req.id, ErrorCode.TOOL_NOT_FOUND, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name].call(args)
                return JSONRPCResponse(
                    id=req.id,
                    result={"content": [{"type": "text", "text": str(resultado)}], "isError": False},
                )
            except Exception as e:
                return JSONRPCError(req.id, ErrorCode.INTERNAL_ERROR, str(e))

        if method == MCPMethod.RESOURCES_LIST:
            return JSONRPCResponse(
                id=req.id,
                result={"resources": [r.to_dict() for r in self._resources.values()]},
            )

        if method == MCPMethod.RESOURCES_READ:
            uri = req.params.get("uri")
            if uri not in self._resources:
                return JSONRPCError(req.id, ErrorCode.RESOURCE_NOT_FOUND, f"Recurso '{uri}' no encontrado")
            return JSONRPCResponse(
                id=req.id,
                result={"contents": [self._resources[uri].read()]},
            )

        if method == MCPMethod.PROMPTS_LIST:
            return JSONRPCResponse(
                id=req.id,
                result={"prompts": [p.to_dict() for p in self._prompts.values()]},
            )

        return JSONRPCError(req.id, ErrorCode.METHOD_NOT_FOUND, f"Método '{method}' no soportado")

    def handle_notification(self, notif: JSONRPCNotification):
        if notif.method == MCPMethod.INITIALIZED:
            self._initialized = True
            print(f"[Servidor] Cliente inicializado correctamente.")


def demo_ciclo_vida():
    print("\n" + "=" * 60)
    print("DEMO 3 — Ciclo de vida de conexión MCP")
    print("=" * 60)

    # Construir servidor con herramientas y recursos
    servidor = MCPServer("DemoServer", "1.0.0")
    servidor.register_tool(MCPTool(
        name="calcular",
        description="Evalúa expresión matemática",
        parameters=[MCPToolParameter("expresion", "string", "Expresión matemática")],
        fn=lambda expresion: eval(expresion) if all(c in "0123456789+-*/(). " for c in expresion) else "Error",
    ))
    servidor.register_resource(MCPResource(
        uri="file:///docs/guia.txt",
        name="Guía rápida",
        description="Guía de inicio rápido",
        content="# Guía Rápida\n\n1. Conectar\n2. Listar herramientas\n3. Llamar herramientas",
    ))

    # Conectar cliente
    cliente_info = MCPClientInfo("MiAgente", "0.1.0")
    conn = MCPConnection(cliente_info, servidor)
    ok = conn.connect()
    assert ok, "Conexión debe ser exitosa"

    # Listar herramientas
    tools = conn.list_tools()
    print(f"\nHerramientas disponibles: {[t['name'] for t in tools]}")

    # Llamar herramienta
    resultado = conn.call_tool("calcular", {"expresion": "2 + 2 * 10"})
    print(f"calcular(2 + 2 * 10) = {resultado['content'][0]['text']}")

    # Leer recurso
    doc = conn.read_resource("file:///docs/guia.txt")
    print(f"Recurso: {doc['contents'][0]['text'][:50]}...")

    # Desconectar
    conn.disconnect()

    return {
        "tools": tools,
        "tool_result": resultado,
        "message_count": len(conn._message_log),
    }


if __name__ == "__main__":
    resultado_ciclo = demo_ciclo_vida()
    with open("outputs/m21_ciclo_vida.json", "w", encoding="utf-8") as f:
        json.dump(resultado_ciclo, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m21_ciclo_vida.json")
```

---

## 4. Transportes MCP: stdio, SSE y WebSocket

```python
# Parte 4: Capas de transporte

"""
MCP define el protocolo (JSON-RPC) independiente del transporte.
Los transportes oficiales son:

┌──────────┬───────────────────────────────────────────────────────┐
│ stdio    │ Proceso hijo: cliente escribe a stdin, lee de stdout  │
│          │ Ideal para servidores locales (archivos, base de datos)│
├──────────┼───────────────────────────────────────────────────────┤
│ SSE      │ HTTP: cliente conecta a /sse, recibe eventos          │
│ (HTTP)   │ Cliente envía POST a /messages                        │
│          │ Ideal para servidores remotos detrás de HTTPS         │
├──────────┼───────────────────────────────────────────────────────┤
│ WebSocket│ Bidireccional full-duplex                             │
│ (futuro) │ Para baja latencia y streaming                        │
└──────────┴───────────────────────────────────────────────────────┘
"""


class StdioTransport:
    """
    Simula el transporte stdio.
    En producción: subprocess.Popen con pipes stdin/stdout.
    """

    def __init__(self, server: MCPServer):
        self._server = server
        self._buffer: List[str] = []  # simula stdin del servidor

    def send(self, mensaje: str):
        """Cliente escribe al stdin del servidor."""
        self._buffer.append(mensaje)
        # En producción: process.stdin.write(mensaje + '\n')

    def receive(self) -> Optional[str]:
        """Cliente lee del stdout del servidor."""
        if not self._buffer:
            return None
        raw = self._buffer.pop(0)
        try:
            data = json.loads(raw)
            req = JSONRPCRequest(
                method=data["method"],
                params=data.get("params", {}),
                id=data.get("id"),
            )
            response = self._server.handle_request(req)
            return response.serialize()
        except Exception as e:
            return json.dumps({"error": str(e)})

    def intercambio(self, request: JSONRPCRequest) -> Dict:
        """Envío y recepción en un solo paso (demo simplificado)."""
        self.send(request.serialize())
        raw = self.receive()
        return json.loads(raw) if raw else {}


class SSETransport:
    """
    Simula el transporte HTTP + Server-Sent Events.

    Flujo real:
    1. GET /sse → abre stream de eventos
    2. POST /messages → envía requests JSON-RPC
    3. El servidor responde via SSE con: data: {...}\\n\\n
    """

    def __init__(self, server: MCPServer, base_url: str = "http://localhost:3000"):
        self._server = server
        self.base_url = base_url
        self._event_stream: List[str] = []

    def _post_message(self, request: JSONRPCRequest) -> Dict:
        """Simula POST /messages."""
        response = self._server.handle_request(request)
        # Servidor envía respuesta via SSE
        sse_event = f"data: {response.serialize()}\n\n"
        self._event_stream.append(sse_event)
        return response.to_dict()

    def _read_sse(self) -> List[Dict]:
        """Lee eventos del stream SSE."""
        events = []
        for event in self._event_stream:
            if event.startswith("data: "):
                payload = event[6:].strip()
                try:
                    events.append(json.loads(payload))
                except json.JSONDecodeError:
                    pass
        self._event_stream.clear()
        return events

    def intercambio(self, request: JSONRPCRequest) -> Dict:
        result = self._post_message(request)
        self._read_sse()  # consumir eventos pendientes
        return result


def demo_transportes():
    print("\n" + "=" * 60)
    print("DEMO 4 — Comparativa de Transportes")
    print("=" * 60)

    servidor = MCPServer("TransportDemo")
    servidor.register_tool(MCPTool(
        name="ping",
        description="Responde con pong",
        parameters=[],
        fn=lambda: "pong",
    ))

    req = JSONRPCRequest(method=MCPMethod.TOOLS_LIST, params={})

    # stdio
    print("\n[stdio]")
    stdio = StdioTransport(servidor)
    resp_stdio = stdio.intercambio(req)
    print(f"  Tools: {[t['name'] for t in resp_stdio.get('result', {}).get('tools', [])]}")

    # SSE
    print("\n[SSE]")
    sse = SSETransport(servidor, "http://mi-mcp-server.com")
    resp_sse = sse.intercambio(req)
    print(f"  Tools: {[t['name'] for t in resp_sse.get('result', {}).get('tools', [])]}")

    print(f"\nBase URL SSE: {sse.base_url}")
    print("Ambos transportes producen el mismo resultado ✓")

    return {
        "stdio_tools": resp_stdio.get("result", {}).get("tools", []),
        "sse_tools": resp_sse.get("result", {}).get("tools", []),
    }


if __name__ == "__main__":
    resultado_transporte = demo_transportes()
    with open("outputs/m21_transportes.json", "w", encoding="utf-8") as f:
        json.dump(resultado_transporte, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m21_transportes.json")
```

---

## 5. Cliente MCP Completo con Selección de Herramientas por LLM

```python
# Parte 5: Cliente MCP que usa LLM para seleccionar herramientas

class LLMConMCP:
    """
    LLM simulado que recibe lista de herramientas disponibles
    y decide cuál usar basándose en la consulta del usuario.
    """

    def decidir_herramienta(self, consulta: str, tools: List[Dict]) -> Optional[Dict]:
        """Retorna {"name": ..., "arguments": ...} o None si no usa herramientas."""
        consulta_l = consulta.lower()
        nombres_tools = {t["name"]: t for t in tools}

        if "calcula" in consulta_l or "cuánto" in consulta_l or "resultado" in consulta_l:
            # Extraer expresión numérica
            import re
            nums = re.findall(r'\d+[\+\-\*/\d\s\.]+\d+', consulta)
            expr = nums[0].strip() if nums else "1+1"
            if "calcular" in nombres_tools:
                return {"name": "calcular", "arguments": {"expresion": expr}}

        if "clima" in consulta_l or "temperatura" in consulta_l or "tiempo" in consulta_l:
            ciudades = ["Madrid", "Buenos Aires", "Ciudad de México", "Lima", "Bogotá"]
            ciudad = next((c for c in ciudades if c.lower() in consulta_l), "Madrid")
            if "obtener_clima" in nombres_tools:
                return {"name": "obtener_clima", "arguments": {"ciudad": ciudad, "unidad": "celsius"}}

        if "busca" in consulta_l or "información" in consulta_l:
            if "buscar_web" in nombres_tools:
                return {"name": "buscar_web", "arguments": {"query": consulta[:50]}}

        return None

    def generar_respuesta(self, consulta: str, contexto_tool: Optional[str] = None) -> str:
        if contexto_tool:
            return f"Basándome en los resultados: {contexto_tool}. Consulta resuelta."
        return f"Respuesta directa a: {consulta[:50]}. No se requirieron herramientas."


class MCPClientCompleto:
    """
    Cliente MCP completo que integra:
    1. Conexión con servidor
    2. Descubrimiento de herramientas
    3. LLM para selección de herramientas
    4. Ejecución y síntesis de respuesta
    """

    def __init__(self, client_info: MCPClientInfo, llm: LLMConMCP):
        self.client_info = client_info
        self.llm = llm
        self._conexiones: Dict[str, MCPConnection] = {}
        self._all_tools: List[Dict] = []

    def conectar(self, nombre: str, servidor: MCPServer) -> bool:
        """Conecta a un servidor MCP y cachea sus herramientas."""
        conn = MCPConnection(self.client_info, servidor)
        if conn.connect():
            self._conexiones[nombre] = conn
            tools = conn.list_tools()
            # Añadir metadata de qué servidor provee cada herramienta
            for tool in tools:
                tool["_server"] = nombre
            self._all_tools.extend(tools)
            print(f"  → {len(tools)} herramientas de '{nombre}' disponibles")
            return True
        return False

    def procesar_consulta(self, consulta: str) -> Dict:
        """
        Flujo completo:
        1. LLM decide si usar herramienta
        2. Envía tool call al servidor correcto
        3. Genera respuesta final con el resultado
        """
        print(f"\n[Cliente] Consulta: {consulta}")

        # Paso 1: el LLM decide
        tool_call = self.llm.decidir_herramienta(consulta, self._all_tools)

        resultado_tool = None
        if tool_call:
            nombre_tool = tool_call["name"]
            args = tool_call["arguments"]

            # Encontrar qué servidor tiene la herramienta
            server_name = next(
                (t["_server"] for t in self._all_tools if t["name"] == nombre_tool),
                None,
            )

            if server_name and server_name in self._conexiones:
                print(f"[Cliente] Invocando {nombre_tool} en servidor '{server_name}'")
                try:
                    resp = self._conexiones[server_name].call_tool(nombre_tool, args)
                    resultado_tool = resp["content"][0]["text"]
                    print(f"[Cliente] Resultado: {resultado_tool}")
                except Exception as e:
                    resultado_tool = f"Error: {e}"

        # Paso 2: LLM sintetiza respuesta
        respuesta_final = self.llm.generar_respuesta(consulta, resultado_tool)
        print(f"[Cliente] Respuesta: {respuesta_final}")

        return {
            "consulta": consulta,
            "tool_usado": tool_call["name"] if tool_call else None,
            "resultado_tool": resultado_tool,
            "respuesta": respuesta_final,
        }

    def desconectar_todo(self):
        for conn in self._conexiones.values():
            conn.disconnect()


def demo_cliente_completo():
    print("\n" + "=" * 60)
    print("DEMO 5 — Cliente MCP completo con LLM")
    print("=" * 60)

    # Servidor con múltiples herramientas
    servidor_utilidades = MCPServer("Utilidades", "2.0.0")
    servidor_utilidades.register_tool(MCPTool(
        name="calcular",
        description="Evalúa expresiones matemáticas",
        parameters=[MCPToolParameter("expresion", "string", "Expresión matemática")],
        fn=lambda expresion: eval(expresion) if all(c in "0123456789+-*/(). " for c in expresion) else "Error",
    ))
    servidor_utilidades.register_tool(MCPTool(
        name="obtener_clima",
        description="Clima actual de una ciudad",
        parameters=[
            MCPToolParameter("ciudad", "string", "Ciudad"),
            MCPToolParameter("unidad", "string", "celsius|fahrenheit", required=False),
        ],
        fn=lambda ciudad, unidad="celsius": f"En {ciudad}: 22°C, soleado",
    ))

    # Cliente con LLM
    cliente = MCPClientCompleto(
        client_info=MCPClientInfo("AgentePrueba", "1.0"),
        llm=LLMConMCP(),
    )
    cliente.conectar("utilidades", servidor_utilidades)

    # Procesar consultas
    consultas = [
        "¿Cuánto es 15 * 4 + 7?",
        "¿Qué clima hay en Madrid?",
        "¿Qué es el machine learning?",  # sin herramienta
    ]

    resultados = []
    for consulta in consultas:
        r = cliente.procesar_consulta(consulta)
        resultados.append(r)

    cliente.desconectar_todo()
    return resultados


if __name__ == "__main__":
    resultados_cliente = demo_cliente_completo()
    with open("outputs/m21_cliente_completo.json", "w", encoding="utf-8") as f:
        json.dump(resultados_cliente, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m21_cliente_completo.json")
    print("\n[M21 completado] Todos los outputs guardados en outputs/")
```

---

## Checkpoint

```python
# checkpoint_m21.py — 6 tests para MCP Arquitectura

import json
import uuid
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Union
from enum import Enum


# ── Reimport mínimo ──────────────────────────────────────────────────────────

class MCPMethod(str, Enum):
    INITIALIZE  = "initialize"
    INITIALIZED = "notifications/initialized"
    TOOLS_LIST  = "tools/list"
    TOOLS_CALL  = "tools/call"
    RESOURCES_READ = "resources/read"

class ErrorCode(int, Enum):
    METHOD_NOT_FOUND = -32601
    TOOL_NOT_FOUND   = -32003

@dataclass
class JSONRPCRequest:
    method: str; params: Dict = field(default_factory=dict)
    id: Optional[str] = None; jsonrpc: str = "2.0"
    def __post_init__(self):
        if self.id is None: self.id = str(uuid.uuid4())[:8]
    def to_dict(self): return {"jsonrpc":self.jsonrpc,"id":self.id,"method":self.method,"params":self.params}
    def serialize(self): return json.dumps(self.to_dict())

@dataclass
class JSONRPCResponse:
    id: str; result: Any; jsonrpc: str = "2.0"
    def to_dict(self): return {"jsonrpc":self.jsonrpc,"id":self.id,"result":self.result}

@dataclass
class JSONRPCError:
    id: str; code: int; message: str; data: Any = None; jsonrpc: str = "2.0"
    def to_dict(self): return {"jsonrpc":self.jsonrpc,"id":self.id,"error":{"code":self.code,"message":self.message}}

@dataclass
class MCPToolParameter:
    name: str; type: str; description: str; required: bool = True; enum: Optional[List[str]] = None
    def to_schema(self):
        s = {"type":self.type,"description":self.description}
        if self.enum: s["enum"] = self.enum
        return s

@dataclass
class MCPTool:
    name: str; description: str; parameters: List[MCPToolParameter]; fn: Callable
    annotations: Dict = field(default_factory=dict)
    def input_schema(self):
        return {"type":"object","properties":{p.name:p.to_schema() for p in self.parameters},"required":[p.name for p in self.parameters if p.required]}
    def to_dict(self): return {"name":self.name,"description":self.description,"inputSchema":self.input_schema()}
    def call(self, arguments): return self.fn(**arguments)

@dataclass
class MCPResource:
    uri: str; name: str; description: str; mime_type: str = "text/plain"; content: str = ""
    def to_dict(self): return {"uri":self.uri,"name":self.name,"description":self.description,"mimeType":self.mime_type}
    def read(self): return {"uri":self.uri,"mimeType":self.mime_type,"text":self.content}

@dataclass
class MCPPromptArgument:
    name: str; description: str; required: bool = True

@dataclass
class MCPPrompt:
    name: str; description: str; arguments: List[MCPPromptArgument]; template: str
    def render(self, **kwargs): return self.template.format(**kwargs)

@dataclass
class MCPCapabilities:
    tools: bool = True; resources: bool = True; prompts: bool = True; sampling: bool = False; logging: bool = True
    def to_dict(self):
        c = {}
        if self.tools: c["tools"] = {}
        if self.resources: c["resources"] = {}
        if self.prompts: c["prompts"] = {}
        return c

@dataclass
class MCPServerInfo:
    name: str; version: str

class MCPServer:
    def __init__(self, name, version="1.0.0"):
        self.info = MCPServerInfo(name, version); self.capabilities = MCPCapabilities()
        self._tools = {}; self._resources = {}; self._initialized = False
    def register_tool(self, tool): self._tools[tool.name] = tool
    def register_resource(self, res): self._resources[res.uri] = res
    def handle_request(self, req):
        if req.method == MCPMethod.INITIALIZE:
            return JSONRPCResponse(req.id, {"protocolVersion":"2024-11-05","serverInfo":{"name":self.info.name,"version":self.info.version},"capabilities":self.capabilities.to_dict()})
        if req.method == MCPMethod.TOOLS_LIST:
            return JSONRPCResponse(req.id, {"tools":[t.to_dict() for t in self._tools.values()]})
        if req.method == MCPMethod.TOOLS_CALL:
            name = req.params.get("name"); args = req.params.get("arguments", {})
            if name not in self._tools: return JSONRPCError(req.id, ErrorCode.TOOL_NOT_FOUND, f"Tool '{name}' no encontrada")
            try:
                res = self._tools[name].call(args)
                return JSONRPCResponse(req.id, {"content":[{"type":"text","text":str(res)}],"isError":False})
            except Exception as e: return JSONRPCError(req.id, -32603, str(e))
        if req.method == MCPMethod.RESOURCES_READ:
            uri = req.params.get("uri")
            if uri not in self._resources: return JSONRPCError(req.id, -32002, "Recurso no encontrado")
            return JSONRPCResponse(req.id, {"contents":[self._resources[uri].read()]})
        return JSONRPCError(req.id, ErrorCode.METHOD_NOT_FOUND, "Método no soportado")
    def handle_notification(self, notif): self._initialized = True


# ── Tests ────────────────────────────────────────────────────────────────────

def test_01_jsonrpc_request_serialization():
    """JSONRPCRequest se serializa con campos correctos."""
    req = JSONRPCRequest(method="tools/list", params={"cursor": None})
    d = req.to_dict()
    assert d["jsonrpc"] == "2.0"
    assert d["method"] == "tools/list"
    assert "id" in d
    assert req.id is not None
    raw = req.serialize()
    parsed = json.loads(raw)
    assert parsed["method"] == "tools/list"
    print("✓ test_01 — JSONRPCRequest serialización OK")

def test_02_mcp_tool_schema():
    """MCPTool genera JSON Schema correcto para el LLM."""
    tool = MCPTool(
        name="sumar",
        description="Suma dos números",
        parameters=[
            MCPToolParameter("a", "number", "Primer número"),
            MCPToolParameter("b", "number", "Segundo número"),
        ],
        fn=lambda a, b: a + b,
    )
    schema = tool.input_schema()
    assert schema["type"] == "object"
    assert "a" in schema["properties"]
    assert "b" in schema["properties"]
    assert "a" in schema["required"]
    resultado = tool.call({"a": 3, "b": 4})
    assert resultado == 7
    print("✓ test_02 — MCPTool schema y ejecución OK")

def test_03_mcp_resource_read():
    """MCPResource retorna contenido correcto."""
    recurso = MCPResource(
        uri="file:///test.txt",
        name="Test",
        description="Archivo de prueba",
        content="Hola mundo desde MCP",
    )
    d = recurso.to_dict()
    assert d["uri"] == "file:///test.txt"
    contenido = recurso.read()
    assert contenido["text"] == "Hola mundo desde MCP"
    assert contenido["mimeType"] == "text/plain"
    print("✓ test_03 — MCPResource lectura OK")

def test_04_mcp_prompt_render():
    """MCPPrompt renderiza template con argumentos."""
    prompt = MCPPrompt(
        name="test_prompt",
        description="Prompt de prueba",
        arguments=[MCPPromptArgument("tema", "Tema a analizar")],
        template="Analiza el tema: {tema}. Sé conciso.",
    )
    rendered = prompt.render(tema="inteligencia artificial")
    assert "inteligencia artificial" in rendered
    assert "Analiza el tema:" in rendered
    print("✓ test_04 — MCPPrompt render OK")

def test_05_server_tools_call():
    """MCPServer maneja tools/call correctamente."""
    servidor = MCPServer("TestServer")
    servidor.register_tool(MCPTool(
        name="duplicar",
        description="Duplica un número",
        parameters=[MCPToolParameter("n", "number", "Número a duplicar")],
        fn=lambda n: int(n) * 2,
    ))

    req = JSONRPCRequest(method=MCPMethod.TOOLS_CALL, params={"name": "duplicar", "arguments": {"n": 5}})
    resp = servidor.handle_request(req)
    assert isinstance(resp, JSONRPCResponse)
    assert resp.result["isError"] == False
    assert resp.result["content"][0]["text"] == "10"
    print("✓ test_05 — MCPServer tools/call OK")

def test_06_server_initialize_handshake():
    """El handshake initialize retorna serverInfo y capabilities."""
    servidor = MCPServer("HandshakeServer", "2.0.0")
    req = JSONRPCRequest(
        method=MCPMethod.INITIALIZE,
        params={"protocolVersion": "2024-11-05", "clientInfo": {"name": "Test", "version": "1.0"}, "capabilities": {}},
    )
    resp = servidor.handle_request(req)
    assert isinstance(resp, JSONRPCResponse)
    result = resp.result
    assert result["serverInfo"]["name"] == "HandshakeServer"
    assert result["serverInfo"]["version"] == "2.0.0"
    assert result["protocolVersion"] == "2024-11-05"
    assert "capabilities" in result
    print("✓ test_06 — Initialize handshake OK")


if __name__ == "__main__":
    tests = [
        test_01_jsonrpc_request_serialization,
        test_02_mcp_tool_schema,
        test_03_mcp_resource_read,
        test_04_mcp_prompt_render,
        test_05_server_tools_call,
        test_06_server_initialize_handshake,
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
    print("✓ Checkpoint M21 completado")
```

---

## Resumen

| Concepto | Descripción |
|---|---|
| **JSON-RPC 2.0** | Protocolo base: request/response/notification con `id`, `method`, `params` |
| **Tool** | Acción invocable: nombre + JSON Schema de inputs + función |
| **Resource** | Datos legibles: URI + MIME type + contenido |
| **Prompt** | Template reutilizable: nombre + argumentos + texto con placeholders |
| **initialize** | Handshake: intercambia versión, info y capacidades |
| **TERMINATE** | Señal de fin (en MCP: el cliente cierra la conexión) |
| **stdio** | Transporte local: stdin/stdout de proceso hijo |
| **SSE** | Transporte remoto: HTTP + Server-Sent Events |
| **Capabilities** | Negociación: qué features soporta cada lado |
| **Sampling** | Feature avanzada: el servidor pide al cliente que genere texto |

```
Flujo típico:
  Cliente → initialize → Servidor
  Servidor → {serverInfo, capabilities} → Cliente
  Cliente → notifications/initialized → Servidor
  Cliente → tools/list → Servidor → {tools:[...]}
  LLM decide usar tool → Cliente → tools/call → Servidor → {content}
  LLM sintetiza respuesta final
```
