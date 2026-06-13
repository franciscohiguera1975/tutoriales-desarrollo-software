# Módulo 24 — Integración MCP con Bases de Datos y APIs Externas

> **Objetivo:** Construir integraciones MCP robustas con sistemas reales: bases de datos relacionales y vectoriales, APIs REST con autenticación, y flujos de datos bidireccionales, aplicando patrones de resiliencia como retry, circuit breaker y timeout.
>
> **Herramientas:** Python 3.11, sqlite3, json, urllib
>
> **Prerequisito:** M22 (MCP Servidores), M12 (Vector Databases)

---

## 1. Integración MCP + Base de Datos Relacional con ORM

```python
# modulo-24-mcp-integraciones.py — Parte 1: MCP + SQL con ORM simulado

"""
Patrón de integración MCP ↔ Base de datos:

  Claude (cliente MCP)
       │
       │ tools/call: consultar / insertar / actualizar
       ▼
  MCPDatabaseServer
       │
       │ ORM / Query Builder
       ▼
  SQLite / PostgreSQL / MySQL
"""

import json
import uuid
import time
import sqlite3
import hashlib
import urllib.request
import urllib.error
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Union, Tuple
from enum import Enum
from contextlib import contextmanager


# ── Infraestructura JSON-RPC (mínima, reutilizada) ────────────────────────────

@dataclass
class JSONRPCRequest:
    method: str
    params: Dict = field(default_factory=dict)
    id: Optional[str] = None
    def __post_init__(self):
        if not self.id: self.id = str(uuid.uuid4())[:8]

@dataclass
class JSONRPCResponse:
    id: str; result: Any
    def to_dict(self): return {"jsonrpc":"2.0","id":self.id,"result":self.result}

@dataclass
class JSONRPCError:
    id: str; code: int; message: str
    def to_dict(self): return {"jsonrpc":"2.0","id":self.id,"error":{"code":self.code,"message":self.message}}


# ── ORM Mínimo ────────────────────────────────────────────────────────────────

class QueryBuilder:
    """
    Constructor de consultas SQL seguras (contra inyección).
    Usa parámetros posicionales en lugar de f-strings.
    """

    def __init__(self, tabla: str):
        self.tabla = tabla
        self._wheres: List[Tuple[str, Any]] = []
        self._order: Optional[str] = None
        self._limit: Optional[int] = None
        self._select_cols: List[str] = ["*"]

    def select(self, *cols: str) -> "QueryBuilder":
        self._select_cols = list(cols)
        return self

    def where(self, columna: str, valor: Any, op: str = "=") -> "QueryBuilder":
        """Agrega condición WHERE con parámetro seguro."""
        ALLOWED_OPS = {"=", "!=", ">", "<", ">=", "<=", "LIKE", "IN"}
        if op.upper() not in ALLOWED_OPS:
            raise ValueError(f"Operador no permitido: {op}")
        self._wheres.append((f"{columna} {op} ?", valor))
        return self

    def order_by(self, col: str, desc: bool = False) -> "QueryBuilder":
        direction = "DESC" if desc else "ASC"
        # Validar que la columna no contenga SQL injection
        if not col.replace("_", "").isalnum():
            raise ValueError(f"Nombre de columna inválido: {col}")
        self._order = f"{col} {direction}"
        return self

    def limit(self, n: int) -> "QueryBuilder":
        self._limit = int(n)
        return self

    def build_select(self) -> Tuple[str, List]:
        cols = ", ".join(self._select_cols)
        sql = f"SELECT {cols} FROM {self.tabla}"
        params = []
        if self._wheres:
            conditions = " AND ".join(cond for cond, _ in self._wheres)
            params = [val for _, val in self._wheres]
            sql += f" WHERE {conditions}"
        if self._order:
            sql += f" ORDER BY {self._order}"
        if self._limit:
            sql += f" LIMIT {self._limit}"
        return sql, params


class Database:
    """Conexión SQLite con gestión de contexto y helpers."""

    def __init__(self, db_path: str):
        self.db_path = db_path
        self._conn: Optional[sqlite3.Connection] = None

    @contextmanager
    def connection(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()

    def execute(self, sql: str, params: List = None) -> List[Dict]:
        with self.connection() as conn:
            cur = conn.execute(sql, params or [])
            if cur.description:
                cols = [d[0] for d in cur.description]
                return [dict(zip(cols, row)) for row in cur.fetchall()]
            return []

    def execute_write(self, sql: str, params: List = None) -> int:
        """Retorna rowcount."""
        with self.connection() as conn:
            cur = conn.execute(sql, params or [])
            return cur.rowcount

    def table_exists(self, tabla: str) -> bool:
        result = self.execute(
            "SELECT name FROM sqlite_master WHERE type='table' AND name=?", [tabla]
        )
        return len(result) > 0


class ORMMCPServer:
    """
    Servidor MCP que expone operaciones CRUD mediante ORM.
    Permite lectura y escritura (con control por tabla y operación).
    """

    def __init__(self, db: Database, read_only_tables: List[str], writable_tables: List[str]):
        self.db = db
        self._read_only = set(read_only_tables)
        self._writable = set(writable_tables)
        self._all_allowed = self._read_only | self._writable
        self._tools = self._build_tools()

    def _check_table(self, tabla: str, write: bool = False):
        if tabla not in self._all_allowed:
            raise PermissionError(f"Tabla '{tabla}' no autorizada")
        if write and tabla not in self._writable:
            raise PermissionError(f"Tabla '{tabla}' es de solo lectura")

    def _build_tools(self) -> Dict:

        def buscar(tabla: str, filtros: str = "", columnas: str = "*", orden: str = "", limite: int = 50) -> str:
            self._check_table(tabla)
            qb = QueryBuilder(tabla)
            if columnas != "*":
                qb.select(*[c.strip() for c in columnas.split(",")])

            # Parsear filtros simples "campo=valor,campo2=valor2"
            if filtros:
                for par in filtros.split(","):
                    par = par.strip()
                    if "=" in par:
                        k, v = par.split("=", 1)
                        qb.where(k.strip(), v.strip())

            if orden:
                desc = orden.startswith("-")
                col = orden.lstrip("-")
                qb.order_by(col, desc=desc)

            qb.limit(min(limite, 500))
            sql, params = qb.build_select()

            try:
                filas = self.db.execute(sql, params)
                return json.dumps({"filas": filas, "total": len(filas), "sql": sql}, ensure_ascii=False, default=str)
            except Exception as e:
                raise RuntimeError(f"Error en consulta: {e}")

        def insertar(tabla: str, datos: str) -> str:
            self._check_table(tabla, write=True)
            try:
                obj = json.loads(datos)
            except json.JSONDecodeError:
                raise ValueError("'datos' debe ser JSON válido")
            cols = ", ".join(obj.keys())
            placeholders = ", ".join("?" for _ in obj)
            sql = f"INSERT INTO {tabla} ({cols}) VALUES ({placeholders})"
            count = self.db.execute_write(sql, list(obj.values()))
            return json.dumps({"insertadas": count, "tabla": tabla})

        def actualizar(tabla: str, datos: str, filtros: str) -> str:
            self._check_table(tabla, write=True)
            if not filtros:
                raise ValueError("Se requieren filtros para UPDATE (seguridad)")
            try:
                obj = json.loads(datos)
            except json.JSONDecodeError:
                raise ValueError("'datos' debe ser JSON válido")
            set_clause = ", ".join(f"{k} = ?" for k in obj)
            where_parts, where_vals = [], []
            for par in filtros.split(","):
                if "=" in par:
                    k, v = par.strip().split("=", 1)
                    where_parts.append(f"{k.strip()} = ?")
                    where_vals.append(v.strip())
            where_clause = " AND ".join(where_parts)
            sql = f"UPDATE {tabla} SET {set_clause} WHERE {where_clause}"
            count = self.db.execute_write(sql, list(obj.values()) + where_vals)
            return json.dumps({"actualizadas": count, "tabla": tabla})

        def schema(tabla: str) -> str:
            self._check_table(tabla)
            result = self.db.execute(f"PRAGMA table_info({tabla})")
            return json.dumps({"tabla": tabla, "columnas": result}, ensure_ascii=False, indent=2)

        return {
            "buscar": {"fn": buscar, "desc": "Busca filas en una tabla con filtros opcionales"},
            "insertar": {"fn": insertar, "desc": "Inserta una nueva fila (JSON)"},
            "actualizar": {"fn": actualizar, "desc": "Actualiza filas que cumplan filtros"},
            "schema": {"fn": schema, "desc": "Obtiene esquema de una tabla"},
        }

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        if req.method == "initialize":
            return JSONRPCResponse(req.id, {"protocolVersion":"2024-11-05","serverInfo":{"name":"ORMServer","version":"1.0"},"capabilities":{"tools":{}}})
        if req.method == "tools/list":
            tools = [{"name": k, "description": v["desc"], "inputSchema": {"type": "object", "properties": {}}} for k, v in self._tools.items()]
            return JSONRPCResponse(req.id, {"tools": tools})
        if req.method == "tools/call":
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                return JSONRPCError(req.id, -32003, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name]["fn"](**args)
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": str(resultado)}], "isError": False})
            except PermissionError as e:
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": f"Acceso denegado: {e}"}], "isError": True})
            except Exception as e:
                return JSONRPCError(req.id, -32603, str(e))
        return JSONRPCError(req.id, -32601, "Método no soportado")


# ── Demo 1 ────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — MCP + ORM + SQLite")
    print("=" * 60)

    db = Database("outputs/m24_tienda.db")
    with db.connection() as conn:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS clientes (id INTEGER PRIMARY KEY, nombre TEXT, email TEXT, plan TEXT);
            CREATE TABLE IF NOT EXISTS pedidos (id INTEGER PRIMARY KEY, cliente_id INTEGER, producto TEXT, monto REAL, estado TEXT, fecha TEXT);
            INSERT OR IGNORE INTO clientes VALUES (1,'Ana García','ana@mail.com','premium');
            INSERT OR IGNORE INTO clientes VALUES (2,'Luis Pérez','luis@mail.com','básico');
            INSERT OR IGNORE INTO clientes VALUES (3,'María López','maria@mail.com','premium');
            INSERT OR IGNORE INTO pedidos VALUES (1,1,'Laptop',999.99,'entregado','2024-01-10');
            INSERT OR IGNORE INTO pedidos VALUES (2,1,'Mouse',29.99,'entregado','2024-01-15');
            INSERT OR IGNORE INTO pedidos VALUES (3,2,'Teclado',59.99,'pendiente','2024-01-20');
            INSERT OR IGNORE INTO pedidos VALUES (4,3,'Monitor',349.99,'enviado','2024-01-22');
        """)

    servidor = ORMMCPServer(db, read_only_tables=["clientes"], writable_tables=["pedidos"])

    consultas = [
        ("buscar", {"tabla": "clientes", "filtros": "plan=premium"}),
        ("buscar", {"tabla": "pedidos", "filtros": "estado=entregado", "columnas": "id,producto,monto", "orden": "-monto"}),
        ("schema", {"tabla": "pedidos"}),
        ("insertar", {"tabla": "pedidos", "datos": '{"cliente_id": 2, "producto": "Auriculares", "monto": 79.99, "estado": "pendiente", "fecha": "2024-01-25"}'}),
        # Intento de escritura en tabla de solo lectura (debe fallar)
        ("insertar", {"tabla": "clientes", "datos": '{"nombre": "Hacker"}'}),
    ]

    resultados = []
    for tool_name, args in consultas:
        req = JSONRPCRequest(method="tools/call", params={"name": tool_name, "arguments": args})
        resp = servidor.handle_request(req)
        texto = resp.result["content"][0]["text"] if hasattr(resp, "result") and "content" in resp.result else str(resp.to_dict())
        print(f"\n[{tool_name}({args})]")
        print(texto[:200])
        resultados.append({"tool": tool_name, "resultado": texto[:200]})

    with open("outputs/m24_orm_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m24_orm_demo.json")
```

---

## 2. Patrones de Resiliencia: Retry, Timeout y Circuit Breaker

```python
# Parte 2: Resiliencia en integraciones MCP

import time


class RetryConfig:
    """Configuración de política de reintentos."""

    def __init__(
        self,
        max_intentos: int = 3,
        delay_inicial: float = 1.0,
        backoff: float = 2.0,         # multiplicador exponencial
        max_delay: float = 30.0,
        excepciones_reintentables: tuple = (ConnectionError, TimeoutError),
    ):
        self.max_intentos = max_intentos
        self.delay_inicial = delay_inicial
        self.backoff = backoff
        self.max_delay = max_delay
        self.excepciones_reintentables = excepciones_reintentables


def con_retry(fn: Callable, config: RetryConfig, *args, **kwargs) -> Any:
    """
    Ejecuta fn con retry exponencial.
    Reintenta solo en excepciones configuradas como reintentables.
    """
    delay = config.delay_inicial
    ultimo_error = None

    for intento in range(1, config.max_intentos + 1):
        try:
            return fn(*args, **kwargs)
        except config.excepciones_reintentables as e:
            ultimo_error = e
            if intento == config.max_intentos:
                break
            print(f"  [Retry] Intento {intento}/{config.max_intentos} falló: {e}. Esperando {delay:.1f}s...")
            time.sleep(min(delay, config.max_delay))
            delay *= config.backoff
        except Exception:
            raise  # no reintentar otras excepciones

    raise RuntimeError(f"Falló después de {config.max_intentos} intentos: {ultimo_error}")


class CircuitBreakerState(Enum):
    CLOSED   = "closed"    # Normal: todas las llamadas pasan
    OPEN     = "open"      # Abierto: todas las llamadas fallan inmediatamente
    HALF_OPEN = "half_open" # Semi-abierto: se permite una llamada de prueba


class CircuitBreaker:
    """
    Circuit Breaker pattern para integraciones frágiles.

    CLOSED → falla_threshold errores → OPEN
    OPEN   → timeout_reset segundos → HALF_OPEN
    HALF_OPEN → éxito → CLOSED | fallo → OPEN
    """

    def __init__(
        self,
        falla_threshold: int = 5,
        timeout_reset: float = 60.0,
        nombre: str = "CircuitBreaker",
    ):
        self.nombre = nombre
        self.falla_threshold = falla_threshold
        self.timeout_reset = timeout_reset
        self.state = CircuitBreakerState.CLOSED
        self._fallos = 0
        self._ultimo_fallo: Optional[float] = None
        self._llamadas_totales = 0
        self._llamadas_fallidas = 0

    def call(self, fn: Callable, *args, **kwargs) -> Any:
        """Ejecuta fn a través del circuit breaker."""
        self._llamadas_totales += 1

        if self.state == CircuitBreakerState.OPEN:
            # Verificar si es hora de intentar half-open
            if time.time() - self._ultimo_fallo > self.timeout_reset:
                self.state = CircuitBreakerState.HALF_OPEN
                print(f"  [CB:{self.nombre}] → HALF_OPEN")
            else:
                raise RuntimeError(f"Circuit breaker ABIERTO (espera {self.timeout_reset}s)")

        try:
            resultado = fn(*args, **kwargs)
            self._on_success()
            return resultado
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self._fallos = 0
        if self.state == CircuitBreakerState.HALF_OPEN:
            self.state = CircuitBreakerState.CLOSED
            print(f"  [CB:{self.nombre}] → CLOSED (recuperado)")

    def _on_failure(self):
        self._fallos += 1
        self._llamadas_fallidas += 1
        self._ultimo_fallo = time.time()
        if self._fallos >= self.falla_threshold:
            self.state = CircuitBreakerState.OPEN
            print(f"  [CB:{self.nombre}] → OPEN (umbral de {self.falla_threshold} fallos)")

    @property
    def stats(self) -> Dict:
        return {
            "estado": self.state.value,
            "fallos_consecutivos": self._fallos,
            "llamadas_totales": self._llamadas_totales,
            "llamadas_fallidas": self._llamadas_fallidas,
            "tasa_exito": (self._llamadas_totales - self._llamadas_fallidas) / max(1, self._llamadas_totales),
        }


class ResilientMCPClient:
    """
    Cliente MCP con resiliencia integrada:
    - Retry con backoff exponencial
    - Circuit breaker por servidor
    - Timeout configurable
    - Cache de respuestas exitosas
    """

    def __init__(self, retry_config: Optional[RetryConfig] = None):
        self.retry_config = retry_config or RetryConfig(max_intentos=3, delay_inicial=0.1)
        self._circuit_breakers: Dict[str, CircuitBreaker] = {}
        self._cache: Dict[str, Tuple[Any, float]] = {}  # key → (valor, timestamp)
        self._cache_ttl = 300.0  # 5 minutos
        self._llamadas: List[Dict] = []

    def _get_cb(self, servidor_id: str) -> CircuitBreaker:
        if servidor_id not in self._circuit_breakers:
            self._circuit_breakers[servidor_id] = CircuitBreaker(nombre=servidor_id)
        return self._circuit_breakers[servidor_id]

    def _cache_key(self, servidor_id: str, tool_name: str, args: Dict) -> str:
        return hashlib.md5(f"{servidor_id}:{tool_name}:{json.dumps(args, sort_keys=True)}".encode()).hexdigest()[:10]

    def _get_cache(self, key: str) -> Optional[Any]:
        if key in self._cache:
            valor, ts = self._cache[key]
            if time.time() - ts < self._cache_ttl:
                return valor
        return None

    def call_tool(self, servidor_id: str, servidor_fn: Callable, tool_name: str, args: Dict, cacheable: bool = True) -> Any:
        """
        Llama una herramienta con:
        1. Cache (si cacheable=True)
        2. Circuit breaker
        3. Retry
        """
        # Cache
        if cacheable:
            cache_key = self._cache_key(servidor_id, tool_name, args)
            cached = self._get_cache(cache_key)
            if cached is not None:
                self._llamadas.append({"tool": tool_name, "source": "cache"})
                return cached

        cb = self._get_cb(servidor_id)
        inicio = time.time()

        def _llamada():
            req = JSONRPCRequest(method="tools/call", params={"name": tool_name, "arguments": args})
            resp = servidor_fn(req)
            if hasattr(resp, "result") and "content" in resp.result:
                return resp.result["content"][0]["text"]
            raise RuntimeError("Respuesta inválida")

        try:
            resultado = cb.call(con_retry, self.retry_config, _llamada)
            duracion = time.time() - inicio

            if cacheable:
                self._cache[cache_key] = (resultado, time.time())

            self._llamadas.append({"tool": tool_name, "source": "server", "ms": int(duracion * 1000)})
            return resultado

        except RuntimeError as e:
            self._llamadas.append({"tool": tool_name, "source": "error", "error": str(e)})
            raise


def demo_resiliencia():
    print("\n" + "=" * 60)
    print("DEMO 2 — Resiliencia: Retry y Circuit Breaker")
    print("=" * 60)

    call_count = [0]
    fail_until = [3]  # Falla las primeras 3 llamadas

    def servidor_inestable(req: JSONRPCRequest) -> JSONRPCResponse:
        """Simula un servidor que falla a veces."""
        call_count[0] += 1
        if call_count[0] <= fail_until[0]:
            raise ConnectionError(f"Conexión rechazada (intento {call_count[0]})")
        return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": "éxito tras reintentos"}], "isError": False})

    cliente = ResilientMCPClient(
        retry_config=RetryConfig(max_intentos=4, delay_inicial=0.01, backoff=1.5)
    )

    print("\nTest 1: Retry hasta éxito")
    try:
        resultado = cliente.call_tool("servidor1", servidor_inestable, "ping", {}, cacheable=False)
        print(f"  Resultado: {resultado}")
        print(f"  Llamadas necesarias: {call_count[0]}")
    except Exception as e:
        print(f"  Error: {e}")

    print("\nTest 2: Cache (segunda llamada no llega al servidor)")
    call_count[0] = 0
    fail_until[0] = 0  # servidor siempre exitoso ahora

    def servidor_estable(req: JSONRPCRequest) -> JSONRPCResponse:
        call_count[0] += 1
        return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": f"datos_{call_count[0]}"}], "isError": False})

    r1 = cliente.call_tool("servidor2", servidor_estable, "datos", {"id": 1})
    r2 = cliente.call_tool("servidor2", servidor_estable, "datos", {"id": 1})  # cache
    print(f"  Primera llamada: {r1}, Segunda (cache): {r2}")
    print(f"  Llamadas al servidor: {call_count[0]} (esperado: 1)")

    print("\nEstadísticas del circuit breaker:")
    for nombre, cb in cliente._circuit_breakers.items():
        print(f"  {nombre}: {cb.stats}")

    return {
        "llamadas_totales": len(cliente._llamadas),
        "circuit_breakers": {k: v.stats for k, v in cliente._circuit_breakers.items()},
    }


if __name__ == "__main__":
    stats = demo_resiliencia()
    with open("outputs/m24_resiliencia.json", "w", encoding="utf-8") as f:
        json.dump(stats, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m24_resiliencia.json")
```

---

## 3. MCP con APIs Externas y Autenticación

```python
# Parte 3: Integración MCP con APIs REST autenticadas

class AuthMethod(Enum):
    API_KEY   = "api_key"
    BEARER    = "bearer"
    BASIC     = "basic"
    OAUTH2    = "oauth2"


@dataclass
class APICredentials:
    method: AuthMethod
    api_key: Optional[str] = None
    token: Optional[str] = None
    username: Optional[str] = None
    password: Optional[str] = None

    def headers(self) -> Dict[str, str]:
        if self.method == AuthMethod.API_KEY:
            return {"X-API-Key": self.api_key or ""}
        if self.method == AuthMethod.BEARER:
            return {"Authorization": f"Bearer {self.token or ''}"}
        if self.method == AuthMethod.BASIC:
            import base64
            creds = base64.b64encode(f"{self.username}:{self.password}".encode()).decode()
            return {"Authorization": f"Basic {creds}"}
        return {}


class RESTAPIClient:
    """
    Cliente REST genérico con autenticación, retry y logging.
    En producción usa httpx o aiohttp; aquí simulamos las respuestas.
    """

    def __init__(self, base_url: str, credentials: Optional[APICredentials] = None, timeout: float = 10.0):
        self.base_url = base_url.rstrip("/")
        self.credentials = credentials
        self.timeout = timeout
        self._requests_log: List[Dict] = []

    def _simular_respuesta(self, method: str, endpoint: str, data: Optional[Dict] = None) -> Tuple[int, Dict]:
        """Simula respuesta HTTP. En producción: urllib.request o httpx."""
        key = f"{method}:{endpoint}"
        respuestas = {
            "GET:/users": (200, {"users": [{"id": 1, "name": "Ana"}, {"id": 2, "name": "Luis"}]}),
            "GET:/users/1": (200, {"id": 1, "name": "Ana", "email": "ana@mail.com", "role": "admin"}),
            "GET:/products": (200, {"products": [{"id": 1, "name": "Laptop", "price": 999}]}),
            "POST:/orders": (201, {"id": 99, "status": "created", "total": data.get("total", 0) if data else 0}),
            "PUT:/users/1": (200, {"id": 1, "updated": True, **( data or {})}),
            "GET:/stats": (200, {"total_users": 150, "total_orders": 892, "revenue": 45230.5}),
        }
        return respuestas.get(key, (404, {"error": "Not found"}))

    def request(self, method: str, endpoint: str, data: Optional[Dict] = None, params: Optional[Dict] = None) -> Dict:
        """Realiza una petición HTTP."""
        url = f"{self.base_url}{endpoint}"
        headers = self.credentials.headers() if self.credentials else {}

        if params:
            param_str = "&".join(f"{k}={v}" for k, v in params.items())
            url = f"{url}?{param_str}"

        inicio = time.time()
        status_code, response_data = self._simular_respuesta(method, endpoint, data)
        duracion = time.time() - inicio

        self._requests_log.append({
            "method": method,
            "url": url,
            "status": status_code,
            "ms": int(duracion * 1000),
            "autenticado": bool(headers),
        })

        if status_code >= 400:
            raise RuntimeError(f"HTTP {status_code}: {response_data.get('error', 'Error')}")

        return response_data

    def get(self, endpoint: str, params: Optional[Dict] = None) -> Dict:
        return self.request("GET", endpoint, params=params)

    def post(self, endpoint: str, data: Dict) -> Dict:
        return self.request("POST", endpoint, data=data)

    def put(self, endpoint: str, data: Dict) -> Dict:
        return self.request("PUT", endpoint, data=data)


class RESTAPIMCPServer:
    """Servidor MCP que wrappea una API REST con autenticación."""

    def __init__(self, api_client: RESTAPIClient, name: str = "RESTAPIServer"):
        self.name = name
        self._api = api_client
        self._tools = self._build_tools()

    def _build_tools(self) -> Dict:

        def listar_usuarios() -> str:
            data = self._api.get("/users")
            return json.dumps(data, ensure_ascii=False)

        def obtener_usuario(id: str) -> str:
            data = self._api.get(f"/users/{id}")
            return json.dumps(data, ensure_ascii=False)

        def crear_orden(producto: str, cantidad: int, precio_unitario: float) -> str:
            total = cantidad * precio_unitario
            data = self._api.post("/orders", {"producto": producto, "cantidad": cantidad, "total": total})
            return json.dumps(data, ensure_ascii=False)

        def estadisticas() -> str:
            data = self._api.get("/stats")
            return json.dumps(data, ensure_ascii=False, indent=2)

        return {
            "listar_usuarios": {"fn": listar_usuarios},
            "obtener_usuario": {"fn": obtener_usuario},
            "crear_orden": {"fn": crear_orden},
            "estadisticas": {"fn": estadisticas},
        }

    def handle_request(self, req: JSONRPCRequest) -> Union[JSONRPCResponse, JSONRPCError]:
        if req.method == "tools/call":
            name = req.params.get("name")
            args = req.params.get("arguments", {})
            if name not in self._tools:
                return JSONRPCError(req.id, -32003, f"Tool '{name}' no encontrada")
            try:
                resultado = self._tools[name]["fn"](**args)
                return JSONRPCResponse(req.id, {"content": [{"type": "text", "text": str(resultado)}], "isError": False})
            except Exception as e:
                return JSONRPCError(req.id, -32603, str(e))
        return JSONRPCResponse(req.id, {"serverInfo": {"name": self.name}, "capabilities": {"tools": {}}})


def demo_api_integracion():
    print("\n" + "=" * 60)
    print("DEMO 3 — MCP + REST API con autenticación")
    print("=" * 60)

    # Cliente con API Key
    credenciales = APICredentials(method=AuthMethod.API_KEY, api_key="sk-demo-abc123")
    api_client = RESTAPIClient("https://api.miservicio.com", credenciales)
    servidor = RESTAPIMCPServer(api_client, "MiAPIServer")

    calls = [
        ("listar_usuarios", {}),
        ("obtener_usuario", {"id": "1"}),
        ("estadisticas", {}),
        ("crear_orden", {"producto": "Monitor", "cantidad": 2, "precio_unitario": 349.99}),
    ]

    resultados = []
    for tool, args in calls:
        req = JSONRPCRequest(method="tools/call", params={"name": tool, "arguments": args})
        resp = servidor.handle_request(req)
        texto = resp.result["content"][0]["text"]
        print(f"\n[{tool}]: {texto[:150]}")
        resultados.append({"tool": tool, "resultado": texto[:150]})

    print(f"\nPeticiones HTTP realizadas: {len(api_client._requests_log)}")
    print(f"Todas autenticadas: {all(r['autenticado'] for r in api_client._requests_log)}")

    with open("outputs/m24_api_integracion.json", "w", encoding="utf-8") as f:
        json.dump({"resultados": resultados, "request_log": api_client._requests_log}, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m24_api_integracion.json")

    print("\n[M24 completado] Todos los outputs guardados en outputs/")
    return resultados


if __name__ == "__main__":
    demo_api_integracion()
```

---

## Checkpoint

```python
# checkpoint_m24.py — 6 tests para MCP Integraciones

import json, uuid, sqlite3, time, hashlib
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Tuple, Union
from enum import Enum
from contextlib import contextmanager

@dataclass
class JSONRPCRequest:
    method: str; params: Dict = field(default_factory=dict); id: Optional[str] = None
    def __post_init__(self):
        if not self.id: self.id = str(uuid.uuid4())[:8]

@dataclass
class JSONRPCResponse:
    id: str; result: Any

@dataclass
class JSONRPCError:
    id: str; code: int; message: str

class QueryBuilder:
    def __init__(self, tabla): self.tabla = tabla; self._wheres = []; self._limit = None; self._select_cols = ["*"]
    def where(self, col, val, op="="):
        ALLOWED = {"=","!=",">","<",">=","<=","LIKE"}
        if op.upper() not in ALLOWED: raise ValueError(f"Op inválido: {op}")
        self._wheres.append((f"{col} {op} ?", val)); return self
    def limit(self, n): self._limit = int(n); return self
    def build_select(self):
        sql = f"SELECT {', '.join(self._select_cols)} FROM {self.tabla}"
        params = []
        if self._wheres:
            sql += " WHERE " + " AND ".join(c for c,_ in self._wheres)
            params = [v for _,v in self._wheres]
        if self._limit: sql += f" LIMIT {self._limit}"
        return sql, params

class RetryConfig:
    def __init__(self, max_intentos=3, delay_inicial=0.01, backoff=2.0, max_delay=30.0):
        self.max_intentos = max_intentos; self.delay_inicial = delay_inicial
        self.backoff = backoff; self.max_delay = max_delay
        self.excepciones_reintentables = (ConnectionError, TimeoutError)

def con_retry(fn, config, *args, **kwargs):
    delay = config.delay_inicial; ultimo_error = None
    for intento in range(1, config.max_intentos + 1):
        try: return fn(*args, **kwargs)
        except config.excepciones_reintentables as e:
            ultimo_error = e
            if intento == config.max_intentos: break
            time.sleep(min(delay, config.max_delay)); delay *= config.backoff
        except Exception: raise
    raise RuntimeError(f"Falló tras {config.max_intentos} intentos: {ultimo_error}")

class CircuitBreakerState(Enum):
    CLOSED = "closed"; OPEN = "open"; HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, falla_threshold=5, timeout_reset=60.0, nombre="CB"):
        self.nombre = nombre; self.falla_threshold = falla_threshold
        self.timeout_reset = timeout_reset; self.state = CircuitBreakerState.CLOSED
        self._fallos = 0; self._ultimo_fallo = None
    def call(self, fn, *args, **kwargs):
        if self.state == CircuitBreakerState.OPEN:
            if time.time() - self._ultimo_fallo > self.timeout_reset: self.state = CircuitBreakerState.HALF_OPEN
            else: raise RuntimeError("Circuit breaker ABIERTO")
        try:
            r = fn(*args, **kwargs); self._fallos = 0
            if self.state == CircuitBreakerState.HALF_OPEN: self.state = CircuitBreakerState.CLOSED
            return r
        except Exception:
            self._fallos += 1; self._ultimo_fallo = time.time()
            if self._fallos >= self.falla_threshold: self.state = CircuitBreakerState.OPEN
            raise

class AuthMethod(Enum):
    API_KEY = "api_key"; BEARER = "bearer"

@dataclass
class APICredentials:
    method: AuthMethod; api_key: Optional[str] = None; token: Optional[str] = None
    def headers(self):
        if self.method == AuthMethod.API_KEY: return {"X-API-Key": self.api_key or ""}
        if self.method == AuthMethod.BEARER: return {"Authorization": f"Bearer {self.token or ''}"}
        return {}


def test_01_query_builder_where():
    """QueryBuilder genera SQL parametrizado correcto."""
    qb = QueryBuilder("productos")
    qb.where("categoria", "electrónica").where("precio", 100, ">").limit(10)
    sql, params = qb.build_select()
    assert "WHERE" in sql
    assert "categoria = ?" in sql
    assert "precio > ?" in sql
    assert "LIMIT 10" in sql
    assert params == ["electrónica", 100]
    print("✓ test_01 — QueryBuilder OK")

def test_02_query_builder_bloquea_op_invalido():
    """QueryBuilder rechaza operadores SQL peligrosos."""
    qb = QueryBuilder("t")
    try:
        qb.where("col", "val", "DROP")
        assert False, "Debería lanzar ValueError"
    except ValueError as e:
        assert "Op inválido" in str(e) or "inválido" in str(e).lower()
    print("✓ test_02 — QueryBuilder seguridad OK")

def test_03_retry_exito_tras_fallos():
    """con_retry reintenta y tiene éxito si la función eventualmente funciona."""
    intentos = [0]
    def fn_inestable():
        intentos[0] += 1
        if intentos[0] < 3: raise ConnectionError("Fallo simulado")
        return "éxito"
    config = RetryConfig(max_intentos=5, delay_inicial=0.001)
    resultado = con_retry(fn_inestable, config)
    assert resultado == "éxito"
    assert intentos[0] == 3
    print("✓ test_03 — Retry con éxito OK")

def test_04_retry_falla_max_intentos():
    """con_retry lanza excepción cuando se agotan los intentos."""
    def fn_siempre_falla():
        raise ConnectionError("Siempre falla")
    config = RetryConfig(max_intentos=3, delay_inicial=0.001)
    try:
        con_retry(fn_siempre_falla, config)
        assert False, "Debería lanzar excepción"
    except RuntimeError as e:
        assert "Falló" in str(e)
    print("✓ test_04 — Retry agotado OK")

def test_05_circuit_breaker_abre():
    """CircuitBreaker se abre tras el umbral de fallos."""
    cb = CircuitBreaker(falla_threshold=3, timeout_reset=60.0)
    def fn_falla(): raise RuntimeError("Error")
    for _ in range(3):
        try: cb.call(fn_falla)
        except Exception: pass
    assert cb.state == CircuitBreakerState.OPEN
    assert cb._fallos == 3
    print("✓ test_05 — CircuitBreaker apertura OK")

def test_06_api_credentials_headers():
    """APICredentials genera headers de autenticación correctos."""
    api_key_creds = APICredentials(AuthMethod.API_KEY, api_key="sk-test-123")
    headers_ak = api_key_creds.headers()
    assert headers_ak.get("X-API-Key") == "sk-test-123"

    bearer_creds = APICredentials(AuthMethod.BEARER, token="my-jwt-token")
    headers_b = bearer_creds.headers()
    assert headers_b.get("Authorization") == "Bearer my-jwt-token"
    print("✓ test_06 — APICredentials headers OK")


if __name__ == "__main__":
    import os; os.makedirs("outputs", exist_ok=True)
    tests = [
        test_01_query_builder_where,
        test_02_query_builder_bloquea_op_invalido,
        test_03_retry_exito_tras_fallos,
        test_04_retry_falla_max_intentos,
        test_05_circuit_breaker_abre,
        test_06_api_credentials_headers,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}")
    print(f"Resultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M24 completado")
```

---

## Resumen

| Patrón | Problema que resuelve | Implementación clave |
|---|---|---|
| **QueryBuilder** | Inyección SQL | Parámetros `?` posicionales, whitelist de operadores |
| **Read-only vs Writable** | Control de acceso | Whitelist de tablas por operación |
| **Retry exponencial** | Fallos transitorios de red | Backoff: `delay *= backoff` entre intentos |
| **Circuit Breaker** | Cascada de fallos | `CLOSED → OPEN → HALF_OPEN` según tasa de error |
| **Cache con TTL** | Reducir carga en APIs | `{key: (valor, timestamp)}`, expirar con TTL |
| **APICredentials** | Autenticación segura | Headers por método (API key, Bearer, Basic) |

```python
# Flujo completo MCP → Resiliencia → API

cliente_mcp
    → call_tool("obtener_usuario", {"id": 42})
        → cache_hit? → retornar inmediatamente
        → circuit_breaker.call(
            con_retry(
                fn=servidor.handle_request,
                config=RetryConfig(max_intentos=3)
            )
          )
              → api_client.get("/users/42", headers=auth_headers)
              → respuesta JSON
```
