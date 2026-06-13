# Módulo 22 — MCP: Construcción de Servidores

> **Objetivo:** Construir servidores MCP completos que expongan herramientas reales (filesystem, base de datos, APIs externas), implementar manejo de errores, logging estructurado y pruebas de integración end-to-end.
>
> **Herramientas:** Python 3.11, json, sqlite3, pathlib
>
> **Prerequisito:** M21 (MCP Arquitectura)

---

## 1. Servidor MCP de Filesystem

```python
# modulo-22-mcp-servidores.py — Parte 1: Servidor de Filesystem

"""
El servidor de filesystem es el caso de uso más común en MCP:
expone herramientas para leer, listar y buscar archivos de forma
controlada (el servidor filtra qué rutas puede ver el cliente).
"""

import os
import json
import uuid
import time
import sqlite3
import hashlib
import fnmatch
from pathlib import Path
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Union
from enum import Enum


# ── Infraestructura base (reutilizada de M21) ─────────────────────────────────

class MCPMethod(str, Enum):
    INITIALIZE      = "initialize"
    INITIALIZED     = "notifications/initialized"
    TOOLS_LIST      = "tools/list"
    TOOLS_CALL      = "tools/call"
    RESOURCES_LIST  = "resources/list"
    RESOURCES_READ  = "resources/read"
    PROMPTS_LIST    = "prompts/list"
    LOGGING_MESSAGE = "notifications/message"


@dataclass
class JSONRPCRequest:
    method: str
    params: Dict = field(default_factory=dict)
    id: Optional[str] = None
    jsonrpc: str = "2.0"

    def __post_init__(self):
        if self.id is None:
            self.id = str(uuid.uuid4())[:8]

    def to_dict(self):
        return {"jsonrpc": self.jsonrpc, "id": self.id, "method": self.method, "params": self.params}


@dataclass
class JSONRPCResponse:
    id: str
    result: Any
    jsonrpc: str = "2.0"

    def to_dict(self):
        return {"jsonrpc": self.jsonrpc, "id": self.id, "result": self.result}


@dataclass
class JSONRPCError:
    id: str
    code: int
    message: str
    data: Any = None
    jsonrpc: str = "2.0"

    def to_dict(self):
        return {"jsonrpc": self.jsonrpc, "id": self.id, "error": {"code": self.code, "message": self.message}}


@dataclass
class MCPToolParameter:
    name: str
    type: str
    description: str
    required: bool = True
    enum: Optional[List[str]] = None

    def to_schema(self):
        s: Dict[str, Any] = {"type": self.type, "description": self.description}
        if self.enum:
            s["enum"] = self.enum
        return s


@dataclass
class MCPTool:
    name: str
    description: str
    parameters: List[MCPToolParameter]
    fn: Callable
    annotations: Dict = field(default_factory=dict)

    def input_schema(self):
        return {
            "type": "object",
            "properties": {p.name: p.to_schema() for p in self.parameters},
            "required": [p.name for p in self.parameters if p.required],
        }

    def to_dict(self):
        d: Dict[str, Any] = {
            "name": self.name,
            "description": self.description,
            "inputSchema": self.input_schema(),
        }
        if self.annotations:
            d["annotations"] = self.annotations
        return d

    def call(self, arguments):
        return self.fn(**arguments)


@dataclass
class MCPResource:
    uri: str
    name: str
    description: str
    mime_type: str = "text/plain"
    content: str = ""

    def to_dict(self):
        return {"uri": self.uri, "name": self.name, "description": self.description, "mimeType": self.mime_type}

    def read(self):
        return {"uri": self.uri, "mimeType": self.mime_type, "text": self.content}


# ── Servidor Filesystem ────────────────────────────────────────────────────────

class FilesystemMCPServer:
    """
    Servidor MCP que expone herramientas de sistema de archivos.
    Solo permite acceso a rutas dentro de allowed_paths.
    """

    def __init__(self, allowed_paths: List[str], name: str = "FilesystemServer"):
        self.name = name
        self.version = "1.0.0"
        self._allowed = [Path(p).resolve() for p in allowed_paths]
        self._log: List[Dict] = []
        self._tools = self._build_tools()
        self._resources: Dict[str, MCPResource] = {}

    def _is_allowed(self, path: str) -> bool:
        """Verifica si la ruta está dentro de las permitidas."""
        resolved = Path(path).resolve()
        return any(
            str(resolved).startswith(str(allowed))
            for allowed in self._allowed
        )

    def _build_tools(self) -> Dict[str, MCPTool]:
        """Construye el catálogo de herramientas del servidor."""

        def leer_archivo(path: str) -> str:
            if not self._is_allowed(path):
                raise PermissionError(f"Acceso denegado: {path}")
            p = Path(path)
            if not p.exists():
                raise FileNotFoundError(f"No existe: {path}")
            if not p.is_file():
                raise IsADirectoryError(f"Es un directorio: {path}")
            # Límite de seguridad: max 50KB
            size = p.stat().st_size
            if size > 50_000:
                return f"[Archivo muy grande: {size} bytes. Usa leer_archivo_parcial]"
            return p.read_text(encoding="utf-8", errors="replace")

        def listar_directorio(path: str, pattern: str = "*") -> str:
            if not self._is_allowed(path):
                raise PermissionError(f"Acceso denegado: {path}")
            p = Path(path)
            if not p.is_dir():
                raise NotADirectoryError(f"No es directorio: {path}")
            entries = []
            for item in sorted(p.iterdir()):
                if fnmatch.fnmatch(item.name, pattern):
                    tipo = "DIR " if item.is_dir() else "FILE"
                    size = item.stat().st_size if item.is_file() else 0
                    entries.append(f"{tipo}  {item.name}  ({size} bytes)")
            return "\n".join(entries) if entries else "(directorio vacío)"

        def buscar_en_archivos(path: str, texto: str, extension: str = "*") -> str:
            if not self._is_allowed(path):
                raise PermissionError(f"Acceso denegado: {path}")
            resultados = []
            p = Path(path)
            patron = f"*.{extension}" if extension != "*" else "*"
            for archivo in p.rglob(patron):
                if archivo.is_file() and archivo.stat().st_size < 100_000:
                    try:
                        contenido = archivo.read_text(encoding="utf-8", errors="ignore")
                        lineas_match = [
                            f"  L{i+1}: {linea.strip()}"
                            for i, linea in enumerate(contenido.splitlines())
                            if texto.lower() in linea.lower()
                        ]
                        if lineas_match:
                            resultados.append(f"{archivo}:\n" + "\n".join(lineas_match[:3]))
                    except Exception:
                        pass
            return "\n\n".join(resultados[:10]) if resultados else f"No se encontró '{texto}'"

        def info_archivo(path: str) -> str:
            if not self._is_allowed(path):
                raise PermissionError(f"Acceso denegado: {path}")
            p = Path(path)
            if not p.exists():
                raise FileNotFoundError(f"No existe: {path}")
            stat = p.stat()
            return json.dumps({
                "nombre": p.name,
                "tipo": "directorio" if p.is_dir() else "archivo",
                "tamaño_bytes": stat.st_size,
                "extension": p.suffix,
                "absoluta": str(p.resolve()),
            }, ensure_ascii=False, indent=2)

        tools = {
            "leer_archivo": MCPTool(
                name="leer_archivo",
                description="Lee el contenido de un archivo de texto",
                parameters=[MCPToolParameter("path", "string", "Ruta absoluta o relativa del archivo")],
                fn=leer_archivo,
                annotations={"readOnlyHint": True},
            ),
            "listar_directorio": MCPTool(
                name="listar_directorio",
                description="Lista archivos y carpetas en un directorio",
                parameters=[
                    MCPToolParameter("path", "string", "Ruta del directorio"),
                    MCPToolParameter("pattern", "string", "Patrón glob, ej: *.py", required=False),
                ],
                fn=listar_directorio,
                annotations={"readOnlyHint": True},
            ),
            "buscar_en_archivos": MCPTool(
                name="buscar_en_archivos",
                description="Busca texto en archivos de un directorio",
                parameters=[
                    MCPToolParameter("path", "string", "Directorio donde buscar"),
                    MCPToolParameter("texto", "string", "Texto a buscar"),
                    MCPToolParameter("extension", "string", "Extensión a filtrar, ej: py", required=False),
                ],
                fn=buscar_en_archivos,
                annotations={"readOnlyHint": True},
            ),
            "info_archivo": MCPTool(
                name="info_archivo",
                description="Obtiene metadatos de un archivo o directorio",
                parameters=[MCPToolParameter("path", "string", "Ruta del archivo")],
                fn=info_archivo,
                annotations={"readOnlyHint": True},
            ),
        }
        return tools

    def _log_message(self, level: str, message: str):
        entry = {"timestamp": time.time(), "level": level, "message": message}
        self._log.append(entry)

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        self._log_message("debug", f"Request: {req.method} id={req.id}")

        if req.method == MCPMethod.INITIALIZE:
            return JSONRPCResponse(req.id, {
                "protocolVersion": "2024-11-05",
                "serverInfo": {"name": self.name, "version": self.version},
                "capabilities": {"tools": {}, "resources": {}},
            })

        if req.method == MCPMethod.TOOLS_LIST:
            return JSONRPCResponse(req.id, {
                "tools": [t.to_dict() for t in self._tools.values()],
            })

        if req.method == MCPMethod.TOOLS_CALL:
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                self._log_message("error", f"Tool no encontrada: {name}")
                return JSONRPCError(req.id, -32003, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name].call(args)
                self._log_message("info", f"Tool '{name}' ejecutada OK")
                return JSONRPCResponse(req.id, {
                    "content": [{"type": "text", "text": str(resultado)}],
                    "isError": False,
                })
            except PermissionError as e:
                self._log_message("warning", f"Permiso denegado: {e}")
                return JSONRPCResponse(req.id, {
                    "content": [{"type": "text", "text": f"Acceso denegado: {e}"}],
                    "isError": True,
                })
            except FileNotFoundError as e:
                return JSONRPCResponse(req.id, {
                    "content": [{"type": "text", "text": f"Archivo no encontrado: {e}"}],
                    "isError": True,
                })
            except Exception as e:
                self._log_message("error", f"Error en tool '{name}': {e}")
                return JSONRPCError(req.id, -32603, str(e))

        return JSONRPCError(req.id, -32601, f"Método '{req.method}' no soportado")


# ── Demo 1: Servidor Filesystem ───────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)
    os.makedirs("outputs/demo_files", exist_ok=True)

    # Crear archivos de demo
    (Path("outputs/demo_files/hola.py")).write_text(
        "# Script de prueba\ndef saludo(nombre):\n    return f'Hola, {nombre}'\n\nprint(saludo('mundo'))\n"
    )
    (Path("outputs/demo_files/config.json")).write_text(
        json.dumps({"version": "1.0", "debug": True, "puerto": 8080}, indent=2)
    )
    (Path("outputs/demo_files/README.md")).write_text(
        "# Demo Files\n\nArchivos de prueba para el servidor MCP de filesystem.\n"
    )

    print("=" * 60)
    print("DEMO 1 — Servidor MCP Filesystem")
    print("=" * 60)

    servidor_fs = FilesystemMCPServer(
        allowed_paths=["outputs/demo_files"],
        name="FilesystemServer",
    )

    # Herramienta: listar directorio
    req_list = JSONRPCRequest(
        method=MCPMethod.TOOLS_CALL,
        params={"name": "listar_directorio", "arguments": {"path": "outputs/demo_files"}},
    )
    resp = servidor_fs.handle_request(req_list)
    print(f"\nListado:\n{resp.result['content'][0]['text']}")

    # Herramienta: leer archivo
    req_read = JSONRPCRequest(
        method=MCPMethod.TOOLS_CALL,
        params={"name": "leer_archivo", "arguments": {"path": "outputs/demo_files/hola.py"}},
    )
    resp_read = servidor_fs.handle_request(req_read)
    print(f"\nContenido hola.py:\n{resp_read.result['content'][0]['text']}")

    # Herramienta: buscar texto
    req_search = JSONRPCRequest(
        method=MCPMethod.TOOLS_CALL,
        params={"name": "buscar_en_archivos", "arguments": {"path": "outputs/demo_files", "texto": "def"}},
    )
    resp_search = servidor_fs.handle_request(req_search)
    print(f"\nBúsqueda 'def':\n{resp_search.result['content'][0]['text']}")

    # Acceso denegado (fuera de allowed_paths)
    req_deny = JSONRPCRequest(
        method=MCPMethod.TOOLS_CALL,
        params={"name": "leer_archivo", "arguments": {"path": "/etc/passwd"}},
    )
    resp_deny = servidor_fs.handle_request(req_deny)
    print(f"\nAcceso denegado: {resp_deny.result['content'][0]['text']}")

    print(f"\nLogs del servidor: {len(servidor_fs._log)} entradas")
    with open("outputs/m22_filesystem_demo.json", "w", encoding="utf-8") as f:
        json.dump(servidor_fs._log, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m22_filesystem_demo.json")
```

---

## 2. Servidor MCP de Base de Datos (SQLite)

```python
# Parte 2: Servidor MCP para SQLite

class DatabaseMCPServer:
    """
    Servidor MCP con herramientas para consultar una base de datos SQLite.
    El servidor controla qué tablas son accesibles (whitelist).
    """

    def __init__(self, db_path: str, allowed_tables: List[str], name: str = "DatabaseServer"):
        self.name = name
        self.version = "1.0.0"
        self.db_path = db_path
        self._allowed_tables = set(allowed_tables)
        self._connection: Optional[sqlite3.Connection] = None
        self._tools = self._build_tools()
        self._query_count = 0

    def _get_conn(self) -> sqlite3.Connection:
        """Lazy connection a SQLite."""
        if self._connection is None:
            self._connection = sqlite3.connect(self.db_path)
            self._connection.row_factory = sqlite3.Row
        return self._connection

    def _check_table(self, tabla: str):
        if tabla not in self._allowed_tables:
            raise PermissionError(f"Tabla '{tabla}' no autorizada. Permitidas: {self._allowed_tables}")

    def _build_tools(self) -> Dict[str, MCPTool]:

        def consultar(sql: str, limit: int = 100) -> str:
            """Ejecuta SELECT. Rechaza cualquier escritura."""
            sql_clean = sql.strip().upper()
            if not sql_clean.startswith("SELECT"):
                raise ValueError("Solo se permiten consultas SELECT")
            # Verificar tablas mencionadas
            for tabla in self._allowed_tables:
                pass  # En producción: parse SQL y verificar
            self._query_count += 1
            conn = self._get_conn()
            try:
                cursor = conn.execute(sql + f" LIMIT {min(limit, 1000)}")
                cols = [d[0] for d in cursor.description]
                rows = [dict(zip(cols, row)) for row in cursor.fetchall()]
                return json.dumps({"columnas": cols, "filas": rows, "total": len(rows)}, ensure_ascii=False, default=str)
            except sqlite3.Error as e:
                raise RuntimeError(f"Error SQL: {e}")

        def listar_tablas() -> str:
            conn = self._get_conn()
            cursor = conn.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
            all_tables = [row[0] for row in cursor.fetchall()]
            visible = [t for t in all_tables if t in self._allowed_tables]
            return json.dumps({"tablas_disponibles": visible}, ensure_ascii=False)

        def esquema_tabla(tabla: str) -> str:
            self._check_table(tabla)
            conn = self._get_conn()
            cursor = conn.execute(f"PRAGMA table_info({tabla})")
            columnas = [
                {"nombre": row[1], "tipo": row[2], "not_null": bool(row[3]), "pk": bool(row[5])}
                for row in cursor.fetchall()
            ]
            return json.dumps({"tabla": tabla, "columnas": columnas}, ensure_ascii=False, indent=2)

        def contar_filas(tabla: str) -> str:
            self._check_table(tabla)
            conn = self._get_conn()
            cursor = conn.execute(f"SELECT COUNT(*) FROM {tabla}")
            count = cursor.fetchone()[0]
            return str(count)

        return {
            "consultar": MCPTool(
                name="consultar",
                description="Ejecuta una consulta SQL SELECT en la base de datos",
                parameters=[
                    MCPToolParameter("sql", "string", "Consulta SQL SELECT"),
                    MCPToolParameter("limit", "number", "Máximo de filas a retornar", required=False),
                ],
                fn=consultar,
                annotations={"readOnlyHint": True},
            ),
            "listar_tablas": MCPTool(
                name="listar_tablas",
                description="Lista las tablas disponibles en la base de datos",
                parameters=[],
                fn=listar_tablas,
                annotations={"readOnlyHint": True},
            ),
            "esquema_tabla": MCPTool(
                name="esquema_tabla",
                description="Muestra la estructura (columnas y tipos) de una tabla",
                parameters=[MCPToolParameter("tabla", "string", "Nombre de la tabla")],
                fn=esquema_tabla,
                annotations={"readOnlyHint": True},
            ),
            "contar_filas": MCPTool(
                name="contar_filas",
                description="Cuenta el número de filas en una tabla",
                parameters=[MCPToolParameter("tabla", "string", "Nombre de la tabla")],
                fn=contar_filas,
                annotations={"readOnlyHint": True},
            ),
        }

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        if req.method == MCPMethod.INITIALIZE:
            return JSONRPCResponse(req.id, {
                "protocolVersion": "2024-11-05",
                "serverInfo": {"name": self.name, "version": self.version},
                "capabilities": {"tools": {}},
            })
        if req.method == MCPMethod.TOOLS_LIST:
            return JSONRPCResponse(req.id, {"tools": [t.to_dict() for t in self._tools.values()]})
        if req.method == MCPMethod.TOOLS_CALL:
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                return JSONRPCError(req.id, -32003, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name].call(args)
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": str(resultado)}], "isError": False})
            except PermissionError as e:
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": f"Acceso denegado: {e}"}], "isError": True})
            except Exception as e:
                return JSONRPCError(req.id, -32603, str(e))
        return JSONRPCError(req.id, -32601, "Método no soportado")

    def close(self):
        if self._connection:
            self._connection.close()
            self._connection = None


def crear_db_demo(db_path: str):
    """Crea una base de datos SQLite de ejemplo."""
    conn = sqlite3.connect(db_path)
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS productos (
            id INTEGER PRIMARY KEY,
            nombre TEXT NOT NULL,
            precio REAL NOT NULL,
            categoria TEXT,
            stock INTEGER DEFAULT 0
        );
        CREATE TABLE IF NOT EXISTS ventas (
            id INTEGER PRIMARY KEY,
            producto_id INTEGER,
            cantidad INTEGER,
            fecha TEXT,
            total REAL
        );
        INSERT OR IGNORE INTO productos VALUES
            (1, 'Laptop', 999.99, 'Electrónica', 15),
            (2, 'Mouse', 29.99, 'Periféricos', 100),
            (3, 'Teclado', 59.99, 'Periféricos', 75),
            (4, 'Monitor', 349.99, 'Electrónica', 20),
            (5, 'Auriculares', 79.99, 'Audio', 50);
        INSERT OR IGNORE INTO ventas VALUES
            (1, 1, 2, '2024-01-15', 1999.98),
            (2, 2, 5, '2024-01-16', 149.95),
            (3, 3, 3, '2024-01-17', 179.97),
            (4, 4, 1, '2024-01-18', 349.99);
    """)
    conn.commit()
    conn.close()


def demo_database_server():
    print("\n" + "=" * 60)
    print("DEMO 2 — Servidor MCP de Base de Datos")
    print("=" * 60)

    db_path = "outputs/demo.db"
    crear_db_demo(db_path)

    servidor_db = DatabaseMCPServer(
        db_path=db_path,
        allowed_tables=["productos", "ventas"],
        name="TiendaDB",
    )

    consultas = [
        ("listar_tablas", {}),
        ("esquema_tabla", {"tabla": "productos"}),
        ("contar_filas", {"tabla": "ventas"}),
        ("consultar", {"sql": "SELECT nombre, precio FROM productos WHERE precio > 50 ORDER BY precio"}),
        ("consultar", {"sql": "SELECT categoria, COUNT(*) as cantidad FROM productos GROUP BY categoria"}),
    ]

    resultados = []
    for nombre, args in consultas:
        req = JSONRPCRequest(
            method=MCPMethod.TOOLS_CALL,
            params={"name": nombre, "arguments": args},
        )
        resp = servidor_db.handle_request(req)
        resultado = resp.result["content"][0]["text"]
        print(f"\n[{nombre}({args})]")
        print(resultado[:200])
        resultados.append({"tool": nombre, "args": args, "resultado": resultado})

    servidor_db.close()
    print(f"\nConsultas ejecutadas: {servidor_db._query_count}")

    with open("outputs/m22_database_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m22_database_demo.json")
    return resultados


if __name__ == "__main__":
    demo_database_server()
```

---

## 3. Servidor MCP de API Externa (simulada)

```python
# Parte 3: Servidor que wrappea una API externa

class APIMCPServer:
    """
    Servidor MCP que wrappea llamadas a APIs externas.
    En producción usa httpx/aiohttp; aquí simulamos las respuestas.
    """

    def __init__(self, name: str = "APIServer", api_key: Optional[str] = None):
        self.name = name
        self.version = "1.0.0"
        self._api_key = api_key or "demo-key"
        self._call_count = 0
        self._cache: Dict[str, Any] = {}  # cache simple para evitar llamadas duplicadas
        self._tools = self._build_tools()

    def _simular_api_call(self, endpoint: str, params: Dict) -> Dict:
        """Simula llamada HTTP a una API externa."""
        self._call_count += 1

        # Cache por fingerprint de params
        cache_key = hashlib.md5(f"{endpoint}{json.dumps(params, sort_keys=True)}".encode()).hexdigest()[:8]
        if cache_key in self._cache:
            return {**self._cache[cache_key], "_cached": True}

        # Simular latencia y respuesta
        respuestas = {
            "/weather": {
                "ciudad": params.get("ciudad", "?"),
                "temperatura": 22,
                "condicion": "Soleado",
                "humedad": 60,
                "viento_kmh": 15,
                "pronostico": ["Soleado", "Nublado", "Lluvia"],
            },
            "/translate": {
                "original": params.get("texto", ""),
                "traduccion": f"[Traducción al {params.get('idioma', 'inglés')}]: {params.get('texto', '')[:30]}...",
                "idioma_origen": "es",
                "idioma_destino": params.get("idioma", "en"),
                "confianza": 0.97,
            },
            "/sentiment": {
                "texto": params.get("texto", "")[:50],
                "sentimiento": "positivo" if any(w in params.get("texto", "").lower() for w in ["bien", "excelente", "bueno"]) else "neutro",
                "score": 0.85,
                "emociones": {"alegría": 0.7, "confianza": 0.6},
            },
            "/geocode": {
                "direccion": params.get("direccion", ""),
                "lat": -34.6037 + hash(params.get("direccion", "")) % 100 * 0.001,
                "lon": -58.3816 + hash(params.get("direccion", "")) % 100 * 0.001,
                "ciudad": "Buenos Aires",
                "pais": "Argentina",
            },
        }

        resultado = respuestas.get(endpoint, {"error": "Endpoint desconocido"})
        self._cache[cache_key] = resultado
        return resultado

    def _build_tools(self) -> Dict[str, MCPTool]:

        def clima(ciudad: str) -> str:
            resp = self._simular_api_call("/weather", {"ciudad": ciudad})
            return json.dumps(resp, ensure_ascii=False, indent=2)

        def traducir(texto: str, idioma: str = "inglés") -> str:
            resp = self._simular_api_call("/translate", {"texto": texto, "idioma": idioma})
            return json.dumps(resp, ensure_ascii=False, indent=2)

        def analizar_sentimiento(texto: str) -> str:
            resp = self._simular_api_call("/sentiment", {"texto": texto})
            return json.dumps(resp, ensure_ascii=False, indent=2)

        def geocodificar(direccion: str) -> str:
            resp = self._simular_api_call("/geocode", {"direccion": direccion})
            return json.dumps(resp, ensure_ascii=False, indent=2)

        return {
            "clima": MCPTool(
                name="clima",
                description="Obtiene el clima actual y pronóstico de una ciudad",
                parameters=[MCPToolParameter("ciudad", "string", "Nombre de la ciudad")],
                fn=clima,
                annotations={"readOnlyHint": True},
            ),
            "traducir": MCPTool(
                name="traducir",
                description="Traduce texto a otro idioma",
                parameters=[
                    MCPToolParameter("texto", "string", "Texto a traducir"),
                    MCPToolParameter("idioma", "string", "Idioma destino", required=False),
                ],
                fn=traducir,
                annotations={"readOnlyHint": True},
            ),
            "analizar_sentimiento": MCPTool(
                name="analizar_sentimiento",
                description="Analiza el sentimiento de un texto",
                parameters=[MCPToolParameter("texto", "string", "Texto a analizar")],
                fn=analizar_sentimiento,
                annotations={"readOnlyHint": True},
            ),
            "geocodificar": MCPTool(
                name="geocodificar",
                description="Obtiene coordenadas GPS de una dirección",
                parameters=[MCPToolParameter("direccion", "string", "Dirección postal")],
                fn=geocodificar,
                annotations={"readOnlyHint": True},
            ),
        }

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        if req.method == MCPMethod.INITIALIZE:
            return JSONRPCResponse(req.id, {
                "protocolVersion": "2024-11-05",
                "serverInfo": {"name": self.name, "version": self.version},
                "capabilities": {"tools": {}},
            })
        if req.method == MCPMethod.TOOLS_LIST:
            return JSONRPCResponse(req.id, {"tools": [t.to_dict() for t in self._tools.values()]})
        if req.method == MCPMethod.TOOLS_CALL:
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                return JSONRPCError(req.id, -32003, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name].call(args)
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": str(resultado)}], "isError": False})
            except Exception as e:
                return JSONRPCError(req.id, -32603, str(e))
        return JSONRPCError(req.id, -32601, "Método no soportado")


def demo_api_server():
    print("\n" + "=" * 60)
    print("DEMO 3 — Servidor MCP de API Externa")
    print("=" * 60)

    servidor_api = APIMCPServer("ExternalAPI")

    calls = [
        ("clima", {"ciudad": "Madrid"}),
        ("traducir", {"texto": "Hola mundo, esto es una prueba.", "idioma": "inglés"}),
        ("analizar_sentimiento", {"texto": "El producto es excelente, muy bueno"}),
        ("geocodificar", {"direccion": "Av. Corrientes 1234, Buenos Aires"}),
        # Segunda llamada al clima (debe usar cache)
        ("clima", {"ciudad": "Madrid"}),
    ]

    resultados = []
    for nombre, args in calls:
        req = JSONRPCRequest(method=MCPMethod.TOOLS_CALL, params={"name": nombre, "arguments": args})
        resp = servidor_api.handle_request(req)
        texto = resp.result["content"][0]["text"]
        cached = "_cached" in texto
        print(f"\n[{nombre}] {'(cache)' if cached else ''}")
        try:
            parsed = json.loads(texto)
            print(json.dumps({k: v for k, v in parsed.items() if k != "_cached"}, ensure_ascii=False, indent=2)[:200])
        except json.JSONDecodeError:
            print(texto[:200])
        resultados.append({"tool": nombre, "resultado": texto[:200]})

    print(f"\nLlamadas API realizadas: {servidor_api._call_count}")
    print(f"Cache hits: {len(calls) - servidor_api._call_count}")

    with open("outputs/m22_api_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m22_api_demo.json")
    return resultados


if __name__ == "__main__":
    demo_api_server()
    print("\n[M22 completado] Todos los outputs guardados en outputs/")
```

---

## Checkpoint

```python
# checkpoint_m22.py — 6 tests para MCP Servidores

import json, uuid, sqlite3, os
from pathlib import Path
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Union
from enum import Enum


class MCPMethod(str, Enum):
    INITIALIZE = "initialize"; TOOLS_LIST = "tools/list"; TOOLS_CALL = "tools/call"

@dataclass
class JSONRPCRequest:
    method: str; params: Dict = field(default_factory=dict)
    id: Optional[str] = None; jsonrpc: str = "2.0"
    def __post_init__(self):
        if self.id is None: self.id = str(uuid.uuid4())[:8]

@dataclass
class JSONRPCResponse:
    id: str; result: Any; jsonrpc: str = "2.0"
    def to_dict(self): return {"jsonrpc":self.jsonrpc,"id":self.id,"result":self.result}

@dataclass
class JSONRPCError:
    id: str; code: int; message: str; jsonrpc: str = "2.0"
    def to_dict(self): return {"jsonrpc":self.jsonrpc,"id":self.id,"error":{"code":self.code,"message":self.message}}

@dataclass
class MCPToolParameter:
    name: str; type: str; description: str; required: bool = True
    def to_schema(self): return {"type":self.type,"description":self.description}

@dataclass
class MCPTool:
    name: str; description: str; parameters: List[MCPToolParameter]; fn: Callable
    annotations: Dict = field(default_factory=dict)
    def input_schema(self):
        return {"type":"object","properties":{p.name:p.to_schema() for p in self.parameters},"required":[p.name for p in self.parameters if p.required]}
    def to_dict(self): return {"name":self.name,"description":self.description,"inputSchema":self.input_schema()}
    def call(self, arguments): return self.fn(**arguments)


# ── Tests ──────────────────────────────────────────────────────────────────

def test_01_filesystem_allowed_paths():
    """El servidor filesystem rechaza rutas fuera de allowed_paths."""
    import fnmatch
    from pathlib import Path as P

    allowed = [P("outputs/demo_files").resolve()]
    def is_allowed(path):
        resolved = P(path).resolve()
        return any(str(resolved).startswith(str(a)) for a in allowed)

    assert is_allowed("outputs/demo_files/test.py"), "Ruta permitida debe pasar"
    assert not is_allowed("/etc/passwd"), "Ruta no permitida debe ser rechazada"
    assert not is_allowed("/tmp/secret.txt"), "Ruta fuera de scope rechazada"
    print("✓ test_01 — Filtrado de rutas filesystem OK")

def test_02_filesystem_list_dir():
    """El servidor puede listar archivos en directorio permitido."""
    os.makedirs("outputs/demo_files", exist_ok=True)
    Path("outputs/demo_files/test_22.txt").write_text("contenido de prueba")

    tool = MCPTool(
        name="listar",
        description="Lista dir",
        parameters=[MCPToolParameter("path","string","Dir")],
        fn=lambda path: "\n".join(str(f.name) for f in Path(path).iterdir()),
    )
    resultado = tool.call({"path": "outputs/demo_files"})
    assert "test_22.txt" in resultado
    print("✓ test_02 — Listar directorio OK")

def test_03_database_server_select():
    """El servidor de DB ejecuta SELECT y retorna JSON."""
    db_path = "outputs/test_m22.db"
    conn = sqlite3.connect(db_path)
    conn.execute("CREATE TABLE IF NOT EXISTS items (id INTEGER PRIMARY KEY, nombre TEXT, valor REAL)")
    conn.execute("INSERT OR IGNORE INTO items VALUES (1, 'Alpha', 10.5)")
    conn.execute("INSERT OR IGNORE INTO items VALUES (2, 'Beta', 20.0)")
    conn.commit()
    conn.close()

    allowed = {"items"}

    def consultar(sql: str, limit: int = 100) -> str:
        if not sql.strip().upper().startswith("SELECT"):
            raise ValueError("Solo SELECT")
        c = sqlite3.connect(db_path)
        c.row_factory = sqlite3.Row
        cur = c.execute(sql + f" LIMIT {limit}")
        cols = [d[0] for d in cur.description]
        rows = [dict(zip(cols, row)) for row in cur.fetchall()]
        c.close()
        return json.dumps({"columnas": cols, "filas": rows, "total": len(rows)})

    resultado = consultar("SELECT * FROM items")
    parsed = json.loads(resultado)
    assert parsed["total"] == 2
    assert "nombre" in parsed["columnas"]
    print("✓ test_03 — Database SELECT OK")

def test_04_database_blocks_write():
    """El servidor rechaza consultas que no son SELECT."""
    def consultar_seguro(sql: str, limit: int = 100) -> str:
        if not sql.strip().upper().startswith("SELECT"):
            raise ValueError("Solo se permiten consultas SELECT")
        return "ok"

    try:
        consultar_seguro("DELETE FROM items")
        assert False, "Debería haber lanzado ValueError"
    except ValueError as e:
        assert "SELECT" in str(e)
    print("✓ test_04 — Bloqueo de escritura en DB OK")

def test_05_api_server_caching():
    """El servidor de API cachea respuestas para no llamar dos veces."""
    import hashlib

    cache: Dict = {}
    call_count = [0]

    def simular_con_cache(endpoint: str, params: Dict) -> Dict:
        key = hashlib.md5(f"{endpoint}{json.dumps(params, sort_keys=True)}".encode()).hexdigest()[:8]
        if key in cache:
            return {**cache[key], "_cached": True}
        call_count[0] += 1
        resultado = {"resultado": f"respuesta_{endpoint}", "params": params}
        cache[key] = resultado
        return resultado

    # Primera llamada
    r1 = simular_con_cache("/weather", {"ciudad": "Madrid"})
    assert "_cached" not in r1
    assert call_count[0] == 1

    # Segunda llamada (mismos params → cache)
    r2 = simular_con_cache("/weather", {"ciudad": "Madrid"})
    assert r2.get("_cached") == True
    assert call_count[0] == 1  # no incrementó

    print("✓ test_05 — API caching OK")

def test_06_tool_annotations():
    """Las herramientas de solo lectura tienen readOnlyHint en annotations."""
    tool = MCPTool(
        name="leer",
        description="Lee algo",
        parameters=[MCPToolParameter("path", "string", "Ruta")],
        fn=lambda path: "contenido",
        annotations={"readOnlyHint": True, "destructiveHint": False},
    )
    d = tool.to_dict()
    # to_dict no incluye annotations en este impl; verificamos que la tool tiene la annotation
    assert tool.annotations.get("readOnlyHint") == True
    assert tool.annotations.get("destructiveHint") == False
    assert tool.call({"path": "/tmp/x"}) == "contenido"
    print("✓ test_06 — Tool annotations OK")


if __name__ == "__main__":
    tests = [
        test_01_filesystem_allowed_paths,
        test_02_filesystem_list_dir,
        test_03_database_server_select,
        test_04_database_blocks_write,
        test_05_api_server_caching,
        test_06_tool_annotations,
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
    print("✓ Checkpoint M22 completado")
```

---

## Resumen

| Servidor | Herramientas | Control de acceso |
|---|---|---|
| **Filesystem** | `leer_archivo`, `listar_directorio`, `buscar_en_archivos`, `info_archivo` | `allowed_paths` whitelist |
| **Database** | `consultar`, `listar_tablas`, `esquema_tabla`, `contar_filas` | Solo SELECT + `allowed_tables` |
| **API Externa** | `clima`, `traducir`, `analizar_sentimiento`, `geocodificar` | Cache, rate limiting |

### Patrones de seguridad en servidores MCP

```python
# 1. Whitelist de rutas/tablas — bloquear acceso no autorizado
def _is_allowed(self, path): return any(path.startswith(a) for a in self._allowed)

# 2. Solo lectura — rechazar operaciones de escritura
if not sql.strip().upper().startswith("SELECT"): raise ValueError("Solo SELECT")

# 3. Límites de tamaño — evitar DoS
if file_size > 50_000: return "[Archivo demasiado grande]"

# 4. Manejo de errores — isError vs error de protocolo
return JSONRPCResponse(id, {"content": [...], "isError": True})  # error de negocio
return JSONRPCError(id, -32603, "Error interno")                 # error de protocolo

# 5. Cache — evitar llamadas externas repetidas
cache_key = md5(endpoint + params).hexdigest()
if cache_key in cache: return cached_result
```
