# Tutorial Python — Página 3
## Módulo 1 · Entorno y Fundamentos
### `match/case` — Structural Pattern Matching (Python 3.10+)

---

## Introducción

El `match/case` de Python 3.10 (PEP 634) es fundamentalmente distinto del
`switch` clásico de C, Java o JavaScript. No es solo una comparación de igualdad:
es un motor de **desestructuración** que puede verificar la forma de un objeto,
extraer sus campos y ligarlos a variables en una sola operación.

### ¿Por qué es diferente al `switch` de otros lenguajes?

| Característica             | `switch` de C/Java | `when` de Kotlin | `match` de Python 3.10 |
|----------------------------|--------------------|------------------|------------------------|
| Comparación de igualdad    | Si                 | Si               | Si                      |
| Múltiples valores (`\|`)   | `case 1: case 2:`  | `1, 2 ->`        | `case 1 \| 2:`          |
| Guards (condición extra)   | No                 | `if condición`   | `case x if condición:` |
| Desestructuración de tipos | No                 | Parcial          | Si (completa)           |
| Desestructuración de dict  | No                 | No               | Si                      |
| Desestructuración de lista | No                 | No               | Si                      |
| Patrones de dataclasses    | No                 | No               | Si                      |
| Captura de subcampos       | No                 | Si               | Si                      |
| Wildcard (`_`)             | `default:`         | `else ->`        | `case _:`              |

### Comparativa: `if/elif` vs `match/case` para el mismo problema

```python
# ── Versión con if/elif ────────────────────────────────────────────────────
def describir_codigo_if(codigo: int) -> str:
    if codigo == 200:
        return "OK"
    elif codigo == 201:
        return "Created"
    elif codigo == 204:
        return "No Content"
    elif codigo == 301 or codigo == 302:
        return "Redirect"
    elif codigo == 400:
        return "Bad Request"
    elif codigo == 401:
        return "Unauthorized"
    elif codigo == 404:
        return "Not Found"
    elif codigo == 500:
        return "Internal Server Error"
    else:
        return f"Código {codigo}"


# ── Versión con match/case ─────────────────────────────────────────────────
def describir_codigo_match(codigo: int) -> str:
    match codigo:
        case 200:
            return "OK"
        case 201:
            return "Created"
        case 204:
            return "No Content"
        case 301 | 302:
            return "Redirect"
        case 400:
            return "Bad Request"
        case 401:
            return "Unauthorized"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:
            return f"Código {codigo}"

# Ambas producen los mismos resultados — la versión match es más legible
# cuando hay muchos casos y especialmente cuando se usa desestructuración
```

La ventaja del `match/case` se vuelve clara en los siguientes apartados, donde
el `if/elif` equivalente sería mucho más verboso.

---

## 1. `match` con valores literales

La forma más básica: comparar un valor con literales concretos.

```python
# Clasificar código de estado HTTP con literales exactos
def estado_http(codigo: int) -> str:
    """Devuelve una descripción detallada del código HTTP."""
    match codigo:
        case 200:
            return "200 OK — solicitud exitosa con contenido"
        case 201:
            return "201 Created — recurso creado correctamente"
        case 204:
            return "204 No Content — éxito sin cuerpo de respuesta"
        case 206:
            return "206 Partial Content — descarga parcial en progreso"
        case 301:
            return "301 Moved Permanently — redirección permanente"
        case 302:
            return "302 Found — redirección temporal"
        case 304:
            return "304 Not Modified — usar copia en caché"
        case 400:
            return "400 Bad Request — datos de entrada inválidos"
        case 401:
            return "401 Unauthorized — autenticación requerida"
        case 403:
            return "403 Forbidden — sin permisos, aunque autenticado"
        case 404:
            return "404 Not Found — el recurso no existe"
        case 409:
            return "409 Conflict — conflicto con el estado actual"
        case 422:
            return "422 Unprocessable Entity — validación de negocio fallida"
        case 429:
            return "429 Too Many Requests — límite de tasa excedido"
        case 500:
            return "500 Internal Server Error — fallo genérico del servidor"
        case 502:
            return "502 Bad Gateway — respuesta inválida de servicio upstream"
        case 503:
            return "503 Service Unavailable — servidor no disponible"
        case 504:
            return "504 Gateway Timeout — el servicio upstream no respondió"
        case _:
            # Wildcard — captura cualquier valor no listado arriba
            return f"Código HTTP {codigo} — consultar RFC 7231 / RFC 9110"


# Clasificar protocolo de red por número de puerto conocido
def protocolo_por_puerto(puerto: int) -> str:
    """Identifica el protocolo estándar por número de puerto."""
    match puerto:
        case 20 | 21:
            return f"Puerto {puerto}: FTP (File Transfer Protocol)"
        case 22:
            return "Puerto 22: SSH (Secure Shell)"
        case 23:
            return "Puerto 23: Telnet — NO USAR, sin cifrado"
        case 25:
            return "Puerto 25: SMTP (envío de correo)"
        case 53:
            return "Puerto 53: DNS (resolución de nombres)"
        case 80:
            return "Puerto 80: HTTP — tráfico web sin cifrado"
        case 110:
            return "Puerto 110: POP3 (recepción de correo)"
        case 143:
            return "Puerto 143: IMAP (acceso a correo remoto)"
        case 443:
            return "Puerto 443: HTTPS — tráfico web cifrado con TLS"
        case 3306:
            return "Puerto 3306: MySQL / MariaDB"
        case 5432:
            return "Puerto 5432: PostgreSQL"
        case 6379:
            return "Puerto 6379: Redis"
        case 27017:
            return "Puerto 27017: MongoDB"
        case _:
            if 1024 <= puerto <= 49151:
                return f"Puerto {puerto}: registrado (uso específico de aplicación)"
            elif puerto < 1024:
                return f"Puerto {puerto}: bien conocido — verificar /etc/services"
            else:
                return f"Puerto {puerto}: efímero (asignado dinámicamente)"


# Pruebas
codigos = [200, 201, 304, 401, 404, 429, 503, 418]
for codigo in codigos:
    print(estado_http(codigo))

print()
puertos = [22, 80, 443, 3306, 8080, 27017, 49200]
for p in puertos:
    print(protocolo_por_puerto(p))
```

> **Prueba esto:** Añade un caso para el puerto `8080` que devuelva `"Puerto 8080: HTTP alternativo (desarrollo/proxy)"`. ¿En qué posición de los `case` lo colocarías — antes o después del `case _:`? ¿Por qué?

---

### `case A | B:` — múltiples valores en una sola rama

El operador `|` en un patrón es el equivalente de `or` para valores literales.
Permite agrupar casos que tienen el mismo comportamiento.

```python
# Clasificar códigos HTTP por familia con el operador |
def familia_http(codigo: int) -> tuple[str, str, str]:
    """
    Clasifica un código HTTP en su familia.
    Devuelve (familia, icono, descripción_corta).
    """
    match codigo:
        # Familia 2xx — éxito (los más comunes)
        case 200 | 201 | 202 | 203 | 204 | 206 | 207 | 208:
            return ("2xx ÉXITO", "[OK]", "La solicitud fue recibida y procesada")

        # Familia 3xx — redirección
        case 301 | 302 | 303 | 304 | 307 | 308:
            return ("3xx REDIR", "[->]", "El cliente debe tomar acción adicional")

        # Errores de cliente más comunes agrupados
        case 400 | 422:
            return ("4xx CLIENTE", "[!]", "Datos de entrada inválidos o no procesables")

        case 401 | 403:
            return ("4xx CLIENTE", "[X]", "Problema de autenticación o autorización")

        case 404 | 410:
            return ("4xx CLIENTE", "[?]", "El recurso no existe o fue eliminado")

        case 429:
            return ("4xx CLIENTE", "[~]", "Límite de tasa excedido — esperar y reintentar")

        # Resto de errores 4xx
        case 405 | 406 | 408 | 409 | 415 | 423 | 451:
            return ("4xx CLIENTE", "[!]", "Error del cliente — revisar documentación")

        # Errores de servidor más comunes
        case 500 | 502 | 503 | 504:
            return ("5xx SERVIDOR", "[!!]", "Fallo del lado del servidor")

        # Resto de errores 5xx
        case 501 | 505 | 507 | 511:
            return ("5xx SERVIDOR", "[!!]", "Capacidad o configuración del servidor")

        # Informativo — raramente visto en APIs REST
        case 100 | 101 | 102 | 103:
            return ("1xx INFO", "[i]", "Respuesta provisional")

        case _:
            return ("DESCONOCIDO", "[?]", f"Código {codigo} no es un código HTTP estándar")


# Simular respuestas de múltiples endpoints
endpoints_y_codigos = [
    ("/api/users",         200),
    ("/api/users",         201),
    ("/api/posts/99",      404),
    ("/api/auth/login",    401),
    ("/api/upload",        422),
    ("/api/search",        429),
    ("/api/admin/restart", 403),
    ("/api/health",        503),
    ("/api/legacy",        301),
]

print(f"{'ENDPOINT':<25} {'CÓDIGO':>6}  {'FAMILIA':<14} {'ICONO'}  DESCRIPCIÓN")
print("-" * 85)
for endpoint, codigo in endpoints_y_codigos:
    familia, icono, desc = familia_http(codigo)
    print(f"{endpoint:<25} {codigo:>6}  {familia:<14} {icono:<5}  {desc}")
```

> **Prueba esto:** Los códigos `204` y `304` se parecen pero pertenecen a familias distintas. ¿A qué familia asigna el código tu función? Verifica con `familia_http(204)` y `familia_http(304)`. ¿Cuál es la diferencia semántica entre los dos?

---

## 2. `match` con guards — condición adicional

Un **guard** es una condición extra que se evalúa después de que el patrón coincide.
Se escribe como `case patron if condicion:`. Si el guard es `False`, se prueba el
siguiente `case` como si el patrón no hubiera coincidido.

```python
# Clasificar latencia de red con rangos — guards para intervalos numéricos
def diagnosticar_latencia(host: str, latencia_ms: float) -> dict[str, str]:
    """
    Clasifica la latencia y sugiere la causa probable.
    Los guards permiten definir rangos como condiciones adicionales.
    """
    match latencia_ms:
        case ms if ms < 0:
            # Valor imposible — error de medición
            return {
                "nivel":  "ERROR",
                "causa":  "Sensor defectuoso o sincronización de tiempo incorrecta",
                "accion": "Verificar NTP y herramienta de medición",
            }
        case ms if ms < 1:
            return {
                "nivel":  "LOCAL",
                "causa":  f"Loopback o interfaz virtual — {ms:.3f}ms",
                "accion": "Normal para servicios locales",
            }
        case ms if ms < 5:
            return {
                "nivel":  "EXCELENTE",
                "causa":  f"Red local de alta velocidad — {ms:.1f}ms",
                "accion": "Sin acción requerida",
            }
        case ms if ms < 20:
            return {
                "nivel":  "MUY BUENA",
                "causa":  f"Red corporativa o datacenter — {ms:.1f}ms",
                "accion": "Sin acción requerida",
            }
        case ms if ms < 80:
            return {
                "nivel":  "BUENA",
                "causa":  f"Conexión nacional o CDN — {ms:.1f}ms",
                "accion": "Monitorear tendencia",
            }
        case ms if ms < 200:
            return {
                "nivel":  "ACEPTABLE",
                "causa":  f"Conexión intercontinental — {ms:.1f}ms",
                "accion": "Considerar réplica más cercana",
            }
        case ms if ms < 500:
            return {
                "nivel":  "LENTA",
                "causa":  f"Posible saturación de red o alta carga — {ms:.1f}ms",
                "accion": "Revisar ancho de banda y carga del servidor",
            }
        case ms if ms < 2000:
            return {
                "nivel":  "MUY LENTA",
                "causa":  f"Problema de red severo — {ms:.1f}ms",
                "accion": "Abrir incidente — verificar rutas de red",
            }
        case _:
            return {
                "nivel":  "INACEPTABLE",
                "causa":  f"Timeout virtual — {latencia_ms:.0f}ms",
                "accion": "El servicio está efectivamente caído desde la perspectiva del cliente",
            }


# Mediciones reales de una auditoría de red
mediciones = [
    ("localhost",            0.08),
    ("10.0.0.1",             0.4),
    ("redis.interno",        1.2),
    ("api.datacenter",       8.5),
    ("cdn.cloudflare.com",   35.0),
    ("api.europa.ejemplo",   120.0),
    ("api.asia.ejemplo",     280.0),
    ("servicio-saturado",    680.0),
    ("servicio-caido",     3200.0),
    ("sensor-roto",           -5.0),
]

for host, ms in mediciones:
    diag = diagnosticar_latencia(host, ms)
    print(f"\n{host} ({ms}ms):")
    print(f"  Nivel  : {diag['nivel']}")
    print(f"  Causa  : {diag['causa']}")
    print(f"  Acción : {diag['accion']}")
```

> **Prueba esto:** Modifica la función para que el nivel `"LENTA"` tenga un guard adicional que solo aplique si `ms < 500 and host.endswith(".interno")`, de modo que la red interna tenga un umbral más estricto. ¿Cómo queda el `case`?

---

```python
# Clasificar temperatura de servidor con guards combinados
from dataclasses import dataclass


@dataclass
class LecturaSensor:
    """Lectura de un sensor de temperatura de servidor."""
    sensor_id:  str
    ubicacion:  str    # "cpu", "gpu", "disco", "ambiente"
    temp_c:     float
    tendencia:  str    # "subiendo", "estable", "bajando"


def evaluar_temperatura(lectura: LecturaSensor) -> tuple[str, str]:
    """
    Evalúa temperatura considerando la ubicación del sensor.
    Los guards pueden referenciar variables del scope exterior.
    """
    # Umbrales distintos por ubicación del sensor
    umbrales = {
        "cpu":      (70, 85, 95),    # (advertencia, crítico, emergencia)
        "gpu":      (75, 88, 100),
        "disco":    (45, 55, 65),
        "ambiente": (28, 35, 40),
    }
    advertencia, critico, emergencia = umbrales.get(lectura.ubicacion, (60, 75, 90))

    match lectura.temp_c:
        case t if t < 0:
            return ("ERROR SENSOR", "Temperatura imposible — verificar sensor")

        case t if t >= emergencia:
            nivel = "EMERGENCIA"
            msg = f"{t:.1f}°C — APAGADO DE EMERGENCIA INMINENTE en {lectura.ubicacion.upper()}"

        case t if t >= critico and lectura.tendencia == "subiendo":
            # Guard compuesto: temperatura crítica Y tendencia al alza
            nivel = "CRÍTICO ASCENDENTE"
            msg = f"{t:.1f}°C subiendo — refrigeración comprometida en {lectura.ubicacion}"

        case t if t >= critico:
            nivel = "CRÍTICO"
            msg = f"{t:.1f}°C — umbral crítico superado en {lectura.ubicacion}"

        case t if t >= advertencia:
            nivel = "ADVERTENCIA"
            msg = f"{t:.1f}°C — por encima del umbral de advertencia"

        case _:
            nivel = "OK"
            msg = f"{lectura.temp_c:.1f}°C — operación normal"

    return (nivel, msg)


# Lecturas de sensores del servidor
sensores = [
    LecturaSensor("CPU-0",   "cpu",      78.5, "subiendo"),
    LecturaSensor("CPU-1",   "cpu",      92.1, "subiendo"),
    LecturaSensor("GPU-0",   "gpu",      65.0, "estable"),
    LecturaSensor("HDD-0",   "disco",    58.0, "estable"),
    LecturaSensor("HDD-1",   "disco",    62.0, "subiendo"),
    LecturaSensor("AMB",     "ambiente", 31.0, "bajando"),
    LecturaSensor("CPU-ERR", "cpu",      -10.0, "estable"),
]

for sensor in sensores:
    nivel, mensaje = evaluar_temperatura(sensor)
    print(f"  [{sensor.sensor_id:<8}] {nivel:<22} {mensaje}")
```

> **Prueba esto:** Añade una lectura con `temp_c=97.0`, `ubicacion="gpu"` y `tendencia="bajando"`. ¿Qué nivel devuelve? ¿Por qué no activa el `case t if t >= critico and lectura.tendencia == "subiendo"`?

---

## 3. `match` con tipos — type patterns

Python `match` puede verificar el **tipo** de un valor usando `case str(var):`,
`case int(var):`, etc. Si el objeto es del tipo especificado, la variable queda
ligada al valor.

```python
# Procesar diferentes tipos de payload que llegan de una API
# En una API real, los payloads pueden ser de tipos muy distintos

def procesar_payload_api(payload: object) -> dict[str, object]:
    """
    Procesa un payload de API que puede venir en diferentes formatos.
    Devuelve siempre un dict normalizado con 'tipo', 'datos' y 'valido'.
    """
    match payload:
        case str(texto) if texto.strip():
            # String no vacío — probablemente JSON crudo o texto plano
            return {
                "tipo":  "texto",
                "datos": texto.strip(),
                "valido": True,
                "longitud": len(texto.strip()),
            }

        case str():
            # String vacío o solo espacios
            return {"tipo": "texto_vacio", "datos": None, "valido": False}

        case int(n) if n >= 0:
            # Entero no negativo — podría ser un ID
            return {"tipo": "id_entero", "datos": n, "valido": True}

        case int(n):
            # Entero negativo — inválido para un ID
            return {"tipo": "id_negativo", "datos": n, "valido": False}

        case float(f):
            # Valor flotante — métrica o coordenada
            return {
                "tipo":  "metrica_float",
                "datos": round(f, 6),
                "valido": True,
            }

        case bool(b):
            # bool ANTES de int — en Python, bool es subclase de int
            # sin esto, True/False caerían en el case int()
            return {"tipo": "bandera", "datos": b, "valido": True}

        case list() as lista if lista:
            # Lista no vacía
            return {
                "tipo":     "coleccion",
                "datos":    lista,
                "valido":   True,
                "elementos": len(lista),
            }

        case list():
            # Lista vacía
            return {"tipo": "coleccion_vacia", "datos": [], "valido": False}

        case dict() as d if d:
            # Dict no vacío — el caso más común en APIs JSON
            return {
                "tipo":  "objeto_json",
                "datos": d,
                "valido": True,
                "campos": list(d.keys()),
            }

        case None:
            # None explícito — ausencia de valor
            return {"tipo": "nulo", "datos": None, "valido": False}

        case _:
            # Tipo no esperado — bytes, set, objeto personalizado, etc.
            return {
                "tipo":  "desconocido",
                "datos": str(payload),
                "valido": False,
                "tipo_python": type(payload).__name__,
            }


# Payloads de prueba representando lo que podría llegar de una API
payloads = [
    '{"usuario": "Ana", "rol": "admin"}',    # JSON como string
    "",                                         # string vacío
    42,                                         # ID entero
    -7,                                         # ID inválido
    3.14159,                                    # métrica
    True,                                       # bandera booleana
    [1, 2, 3, "cuatro"],                        # array
    [],                                         # array vacío
    {"host": "10.0.0.1", "puerto": 8080},       # objeto dict
    None,                                       # nulo
    b"\x00\xFF\x42",                            # bytes binarios
]

for p in payloads:
    resultado = procesar_payload_api(p)
    tipo = resultado["tipo"]
    valido = "VALIDO" if resultado["valido"] else "INVALIDO"
    print(f"  {str(p)[:35]:<37} → [{valido}] tipo={tipo}")
```

> **Prueba esto:** ¿Por qué el `case bool(b):` debe colocarse **antes** del `case int(n):`? Mueve el `case bool(b):` después del `case int(n):` y prueba con `True` y `False`. ¿Qué ocurre y por qué?

---

## 4. `match` con desestructuración de diccionarios

El patrón de dict `case {"clave": patron}:` verifica que el diccionario tenga
esa clave con ese valor. Las claves no mencionadas en el patrón se **ignoran**.
Las variables en el patrón capturan los valores correspondientes.

```python
# Procesar mensajes de un servidor WebSocket
# Los mensajes llegan como dicts con estructura variable

def procesar_mensaje_ws(mensaje: dict) -> str:
    """
    Router de mensajes WebSocket.
    Los patrones de dict solo verifican las claves especificadas —
    el resto de campos del dict se ignoran automáticamente.
    """
    match mensaje:
        # Ping — solo verificar el tipo, no hay datos adicionales
        case {"tipo": "ping"}:
            return "pong"

        # Ping con identificador de correlación — capturar el id
        case {"tipo": "ping", "id": id_ping}:
            return f"pong (correlación: {id_ping})"

        # Suscripción a canal — capturar nombre del canal y opciones
        case {"tipo": "suscribir", "canal": canal, "opciones": {"filtro": filtro}}:
            return f"Suscrito a '{canal}' con filtro '{filtro}'"

        # Suscripción sin opciones específicas
        case {"tipo": "suscribir", "canal": canal}:
            return f"Suscrito a '{canal}' (sin filtros)"

        # Publicar mensaje en canal
        case {"tipo": "publicar", "canal": canal, "datos": datos}:
            bytes_datos = len(str(datos))
            return f"Publicado en '{canal}': {bytes_datos} bytes"

        # Autenticación con token JWT
        case {"tipo": "auth", "token": str(token)} if len(token) >= 32:
            return f"Autenticación con token válido: {token[:8]}..."

        # Autenticación con token inválido (demasiado corto)
        case {"tipo": "auth", "token": token}:
            return f"Error de auth: token inválido (longitud {len(str(token))})"

        # Mensaje de error con código y descripción
        case {"tipo": "error", "codigo": int(codigo), "descripcion": desc}:
            return f"Error WebSocket {codigo}: {desc}"

        # Desconexión limpia
        case {"tipo": "desconexion", "codigo": codigo, "razon": razon}:
            return f"Desconexión [{codigo}]: {razon}"

        # Tipo de mensaje desconocido — capturar el tipo para el log
        case {"tipo": tipo_desconocido}:
            return f"Tipo de mensaje no reconocido: '{tipo_desconocido}'"

        # Dict sin campo 'tipo' — protocolo incorrecto
        case _:
            return f"Mensaje malformado: {list(mensaje.keys())}"


# Secuencia de mensajes WebSocket de prueba
mensajes = [
    {"tipo": "ping"},
    {"tipo": "ping", "id": "corr-001", "timestamp": 1716000000},
    {"tipo": "suscribir", "canal": "alertas.produccion"},
    {"tipo": "suscribir", "canal": "logs", "opciones": {"filtro": "ERROR"}},
    {"tipo": "publicar",  "canal": "metricas", "datos": {"cpu": 87.4, "mem": 61.2}},
    {"tipo": "auth", "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.payload"},
    {"tipo": "auth", "token": "corto"},
    {"tipo": "error", "codigo": 4001, "descripcion": "Token expirado"},
    {"tipo": "desconexion", "codigo": 1000, "razon": "Cierre normal"},
    {"tipo": "heartbeat", "ts": 1716000060},   # tipo desconocido
    {"usuario": "Ana", "accion": "click"},      # sin campo 'tipo'
]

for msg in mensajes:
    respuesta = procesar_mensaje_ws(msg)
    tipo_msg = msg.get("tipo", "—")
    print(f"  [{tipo_msg:<15}] → {respuesta}")
```

> **Prueba esto:** El patrón `{"tipo": "suscribir", "canal": canal, "opciones": {"filtro": filtro}}` es un dict anidado. Prueba con `{"tipo": "suscribir", "canal": "logs", "opciones": {"filtro": "WARN", "formato": "json"}}` — ¿el `"formato": "json"` extra en las opciones provoca un error o se ignora?

---

## 5. `match` con secuencias — listas y tuplas

El patrón de secuencia permite desestructurar listas y tuplas. `[a, b]` coincide
con exactamente dos elementos. `[primero, *resto]` captura el primero y el resto
como lista. `[]` coincide con una secuencia vacía.

```python
# Parsear líneas de log con estructura variable
# Los logs pueden tener diferente número de campos según el formato

def parsear_linea_log(campos: list[str]) -> dict[str, str]:
    """
    Interpreta una línea de log ya dividida en campos.
    Soporta múltiples formatos según el número de campos.
    """
    match campos:
        case []:
            # Línea vacía o en blanco
            return {"tipo": "vacia", "contenido": ""}

        case [mensaje]:
            # Un solo campo — mensaje sin estructura
            return {"tipo": "texto_plano", "mensaje": mensaje}

        case [nivel, mensaje] if nivel in ("ERROR", "WARN", "INFO", "DEBUG", "TRACE"):
            # Dos campos: nivel + mensaje (formato simple)
            return {"tipo": "simple", "nivel": nivel, "mensaje": mensaje}

        case [fecha, hora, nivel, mensaje]:
            # Cuatro campos: fecha, hora, nivel, mensaje (formato estándar)
            return {
                "tipo":     "estandar",
                "fecha":    fecha,
                "hora":     hora,
                "nivel":    nivel,
                "mensaje":  mensaje,
            }

        case [fecha, hora, nivel, servicio, mensaje]:
            # Cinco campos: incluye el nombre del servicio
            return {
                "tipo":     "con_servicio",
                "fecha":    fecha,
                "hora":     hora,
                "nivel":    nivel,
                "servicio": servicio,
                "mensaje":  mensaje,
            }

        case [fecha, hora, nivel, servicio, *resto] if len(resto) > 1:
            # Más de 5 campos — el mensaje puede contener espacios (resto tiene varios elementos)
            mensaje_completo = " ".join(resto)
            return {
                "tipo":     "extendido",
                "fecha":    fecha,
                "hora":     hora,
                "nivel":    nivel,
                "servicio": servicio,
                "mensaje":  mensaje_completo,
                "campos_extra": len(resto),
            }

        case [primero, *resto]:
            # Cualquier otra estructura — capturar todo
            return {
                "tipo":    "desconocido",
                "primero": primero,
                "resto":   resto,
                "total_campos": 1 + len(resto),
            }


# Líneas de log reales divididas en campos
lineas = [
    [],
    ["Inicio del sistema"],
    ["ERROR", "Conexión rechazada por el servidor"],
    ["2024-01-15", "14:32:01", "INFO", "Servidor iniciado en el puerto 8080"],
    ["2024-01-15", "14:32:05", "ERROR", "auth-svc", "Token JWT inválido para usuario 42"],
    ["2024-01-15", "14:32:10", "WARN", "db-svc", "Conexión", "a", "PostgreSQL", "tardó", "850ms"],
    ["METADATO", "campo1", "campo2", "campo3"],
]

for campos in lineas:
    resultado = parsear_linea_log(campos)
    tipo = resultado["tipo"]
    print(f"\n  Campos: {campos}")
    print(f"  Tipo  : {tipo}")
    if "nivel" in resultado:
        print(f"  Nivel : {resultado['nivel']}")
    if "mensaje" in resultado:
        print(f"  Msg   : {resultado['mensaje']}")
```

> **Prueba esto:** El patrón `[primero, *resto]` también coincide con una lista de un solo elemento (`resto` sería `[]`). Pero en el código hay un `case [mensaje]:` antes. ¿Qué ocurre si reordenas los `case` y pones `[primero, *resto]` primero? ¿Por qué Python evalúa los patrones en orden?

---

```python
# Parsear comandos de administración como tuplas
# Los comandos llegan como tuplas: (accion, *argumentos)

def ejecutar_comando(cmd: tuple) -> str:
    """
    Interpreta comandos de administración del servidor.
    Las tuplas permiten una sintaxis limpia para comandos estructurados.
    """
    match cmd:
        case ("ayuda",):
            return "Comandos disponibles: start, stop, restart, status, config get, config set"

        case ("status",):
            return "Sistema operativo — todos los servicios activos"

        case ("start", servicio):
            return f"Iniciando servicio: {servicio}"

        case ("stop", servicio):
            return f"Deteniendo servicio: {servicio}"

        case ("restart", servicio, "--forzar"):
            return f"Reinicio forzado de {servicio} (sin esperar conexiones activas)"

        case ("restart", servicio):
            return f"Reinicio graceful de {servicio} (esperando conexiones activas)"

        case ("config", "get", clave):
            return f"Obteniendo configuración: {clave}"

        case ("config", "set", clave, valor):
            return f"Configurando {clave} = {valor}"

        case ("escalar", servicio, str(replicas)) if replicas.isdigit():
            n = int(replicas)
            return f"Escalando {servicio} a {n} réplica{'s' if n != 1 else ''}"

        case ("log", "tail", str(n_lineas)) if n_lineas.isdigit():
            return f"Mostrando las últimas {n_lineas} líneas del log"

        case (accion, *args):
            return f"Comando desconocido: '{accion}' con args {args}"


# Comandos de prueba
comandos: list[tuple] = [
    ("ayuda",),
    ("status",),
    ("start",    "nginx"),
    ("stop",     "redis"),
    ("restart",  "api-gateway"),
    ("restart",  "worker", "--forzar"),
    ("config",   "get", "max_connections"),
    ("config",   "set", "timeout_ms", "5000"),
    ("escalar",  "web-service", "5"),
    ("log",      "tail", "100"),
    ("backup",   "postgres", "/mnt/backups"),
]

for cmd in comandos:
    resultado = ejecutar_comando(cmd)
    print(f"  {str(cmd):<45} → {resultado}")
```

> **Prueba esto:** Añade un caso para `("escalar", servicio, replicas)` cuando `replicas` no es un número (ej: `("escalar", "web", "mucho")`). El mensaje debe ser `"Error: el número de réplicas debe ser un entero positivo"`. ¿Dónde en el orden de `case` lo colocarías?

---

## 6. `match` con dataclasses

Cuando Python ve `case MiClase(campo=patron):` en un `match`, verifica que:
1. El objeto es una instancia de `MiClase`.
2. El campo `campo` del objeto coincide con `patron`.

Esto es equivalente a `isinstance(obj, MiClase) and obj.campo == patron` pero
en una sola expresión. Se pueden capturar múltiples campos a la vez.

```python
from dataclasses import dataclass
from enum import Enum, auto


class TipoEvento(Enum):
    """Tipos de eventos de red que puede generar el sistema."""
    CONEXION        = auto()
    DESCONEXION     = auto()
    DATOS           = auto()
    ERROR           = auto()
    AUTENTICACION   = auto()
    TIMEOUT         = auto()


class NivelSeguridad(Enum):
    """Nivel de seguridad del cliente conectado."""
    PUBLICO     = auto()
    AUTENTICADO = auto()
    ADMIN       = auto()


@dataclass
class EventoRed:
    """Evento generado por el sistema de red."""
    tipo:        TipoEvento
    cliente_ip:  str
    nivel:       NivelSeguridad = NivelSeguridad.PUBLICO
    bytes_:      int            = 0
    codigo:      int            = 0
    detalle:     str | None     = None


def gestionar_evento_red(evento: EventoRed) -> dict[str, object]:
    """
    Gestiona eventos de red usando match con dataclasses.
    Retorna un dict con la acción tomada y si se debe registrar en auditoría.
    """
    match evento:
        # Nueva conexión de administrador — requiere registro de auditoría
        case EventoRed(tipo=TipoEvento.CONEXION, nivel=NivelSeguridad.ADMIN, cliente_ip=ip):
            return {
                "accion":    f"Conexión de administrador desde {ip} — alerta de seguridad",
                "auditoria": True,
                "prioridad": "ALTA",
            }

        # Conexión normal autenticada
        case EventoRed(tipo=TipoEvento.CONEXION, nivel=NivelSeguridad.AUTENTICADO, cliente_ip=ip):
            return {
                "accion":    f"Conexión autenticada desde {ip}",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Conexión pública
        case EventoRed(tipo=TipoEvento.CONEXION, cliente_ip=ip):
            return {
                "accion":    f"Conexión pública desde {ip} — aplicar rate limiting",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Transferencia masiva (más de 1 MiB) — revisar posible exfiltración
        case EventoRed(tipo=TipoEvento.DATOS, bytes_=b, cliente_ip=ip) if b > 1_048_576:
            mib = b / 1_048_576
            return {
                "accion":    f"Transferencia masiva de {ip}: {mib:.2f} MiB — verificar",
                "auditoria": True,
                "prioridad": "MEDIA",
            }

        # Transferencia normal
        case EventoRed(tipo=TipoEvento.DATOS, bytes_=b, cliente_ip=ip):
            return {
                "accion":    f"Datos de {ip}: {b:,} bytes procesados",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Error de autenticación — posible ataque de fuerza bruta
        case EventoRed(tipo=TipoEvento.AUTENTICACION, codigo=401, cliente_ip=ip):
            return {
                "accion":    f"Fallo de auth desde {ip} — incrementar contador de intentos",
                "auditoria": True,
                "prioridad": "ALTA",
            }

        # Autenticación exitosa
        case EventoRed(tipo=TipoEvento.AUTENTICACION, codigo=200, cliente_ip=ip):
            return {
                "accion":    f"Auth exitosa para {ip}",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Error con detalle disponible
        case EventoRed(tipo=TipoEvento.ERROR, detalle=str(d), cliente_ip=ip):
            return {
                "accion":    f"Error en {ip}: {d}",
                "auditoria": True,
                "prioridad": "MEDIA",
            }

        # Timeout de conexión
        case EventoRed(tipo=TipoEvento.TIMEOUT, cliente_ip=ip):
            return {
                "accion":    f"Timeout en conexión de {ip}",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Desconexión
        case EventoRed(tipo=TipoEvento.DESCONEXION, cliente_ip=ip):
            return {
                "accion":    f"Desconexión de {ip}",
                "auditoria": False,
                "prioridad": "BAJA",
            }

        # Caso no manejado
        case _:
            return {
                "accion":    f"Evento no gestionado: {evento.tipo}",
                "auditoria": True,
                "prioridad": "MEDIA",
            }


# Secuencia de eventos de red
eventos = [
    EventoRed(TipoEvento.CONEXION,      "10.0.0.1",  NivelSeguridad.PUBLICO),
    EventoRed(TipoEvento.CONEXION,      "10.0.0.2",  NivelSeguridad.AUTENTICADO),
    EventoRed(TipoEvento.CONEXION,      "192.168.1.1", NivelSeguridad.ADMIN),
    EventoRed(TipoEvento.AUTENTICACION, "10.0.0.5",  codigo=401),
    EventoRed(TipoEvento.AUTENTICACION, "10.0.0.5",  codigo=200),
    EventoRed(TipoEvento.DATOS,         "10.0.0.2",  bytes_=4_096),
    EventoRed(TipoEvento.DATOS,         "10.0.0.2",  bytes_=5_242_880),
    EventoRed(TipoEvento.ERROR,         "10.0.0.7",  detalle="Connection reset by peer"),
    EventoRed(TipoEvento.TIMEOUT,       "10.0.0.9"),
    EventoRed(TipoEvento.DESCONEXION,   "10.0.0.2"),
]

print(f"{'TIPO':<20} {'IP':<16} {'PRIORIDAD':<8} {'AUDITORÍA'}  ACCIÓN")
print("-" * 90)
for ev in eventos:
    resultado = gestionar_evento_red(ev)
    audit = "SI" if resultado["auditoria"] else "NO"
    print(f"{ev.tipo.name:<20} {ev.cliente_ip:<16} {resultado['prioridad']:<8} "
          f"{'['+audit+']':<10} {resultado['accion']}")
```

> **Prueba esto:** Crea un `EventoRed` con `tipo=TipoEvento.ERROR` pero `detalle=None`. ¿Qué `case` lo captura? ¿Por qué el patrón `case EventoRed(tipo=TipoEvento.ERROR, detalle=str(d))` no coincide cuando `detalle=None`?

---

## 7. Wildcard y OR patterns

```python
# Diferencia entre case _: (wildcard) y case variable: (captura)

def clasificar_valor(valor: object) -> str:
    match valor:
        case 0:
            # Literal exacto — solo coincide con el entero 0
            return "cero exacto"

        case True | False:
            # OR pattern con literales bool — antes que int por herencia
            return f"booleano: {valor}"

        case int(n) | float(n):
            # OR pattern con tipos — captura en la variable 'n'
            # NOTA: la variable de captura debe llamarse igual en ambos lados del |
            return f"número: {n}"

        case str(s) if s:
            # Tipo str con guard — string no vacío
            return f"texto: {s!r}"

        case nombre:
            # CAPTURA — a diferencia de _, esta variable LIGA el valor
            # Es equivalente a case _ pero con nombre — útil para usar en el cuerpo
            return f"otro tipo capturado: {type(nombre).__name__} = {nombre!r}"


# Ejemplo integral — router de comandos que combina todos los patrones
from dataclasses import dataclass
from typing import Any


@dataclass
class ComandoAdmin:
    """Comando de administración con tipo y argumentos tipados."""
    nombre: str
    args:   list[Any]
    nivel:  str = "usuario"   # "usuario", "moderador", "admin"


def router_completo(comando: object) -> str:
    """
    Router que combina literales, guards, tipos, dicts, secuencias y dataclasses.
    Muestra cómo combinar todos los patrones en un solo match.
    """
    match comando:
        # --- Dataclass con campo específico y guard ---
        case ComandoAdmin(nombre="shutdown", nivel="admin"):
            return "Apagando el sistema — requiere confirmación en 30s"

        case ComandoAdmin(nombre="shutdown"):
            return "Error: el comando 'shutdown' requiere nivel 'admin'"

        case ComandoAdmin(nombre=n, args=[], nivel="admin"):
            return f"Ejecutando '{n}' sin argumentos (modo admin)"

        case ComandoAdmin(nombre=n, args=a, nivel=lvl):
            return f"Comando '{n}' con {len(a)} arg(s) — nivel: {lvl}"

        # --- Dict con estructura conocida ---
        case {"accion": "recargar", "servicio": str(svc)}:
            return f"Recargando configuración de {svc}"

        case {"accion": accion, "objetivo": objetivo}:
            return f"Acción '{accion}' sobre objetivo '{objetivo}'"

        # --- Secuencia con desestructuración ---
        case ["exec", *partes] if partes:
            comando_shell = " ".join(partes)
            return f"Ejecutando shell: {comando_shell}"

        case [unico]:
            return f"Comando de un elemento: {unico}"

        # --- Literales y tipos simples ---
        case str(texto) if texto.startswith("/"):
            return f"Ruta de comando: {texto}"

        case int(codigo) if 1000 <= codigo <= 9999:
            return f"Código de operación: {codigo}"

        # --- Wildcard con captura (para logging) ---
        case valor_desconocido:
            tipo = type(valor_desconocido).__name__
            return f"Comando no reconocido ({tipo}): {valor_desconocido!r}"


# Pruebas del router completo
entradas: list[object] = [
    ComandoAdmin("shutdown",  [],           "admin"),
    ComandoAdmin("shutdown",  [],           "usuario"),
    ComandoAdmin("status",    [],           "admin"),
    ComandoAdmin("reiniciar", ["nginx"],    "moderador"),
    {"accion": "recargar",  "servicio": "nginx"},
    {"accion": "bloquear",  "objetivo": "10.0.0.99"},
    ["exec", "systemctl", "status", "nginx"],
    ["/etc/nginx/nginx.conf"],
    "/api/v2/usuarios",
    5001,
    None,
]

for entrada in entradas:
    resultado = router_completo(entrada)
    print(f"  {str(entrada)[:45]:<47} → {resultado}")
```

> **Prueba esto:** En el patrón `case int(n) | float(n):`, ¿qué ocurre si cambias la variable a `case int(n) | float(m):`? Python exige que las variables de captura tengan el mismo nombre en ambos lados de un OR pattern — prueba el error que da y lee el mensaje.

---

## Ejemplo aplicado — router de comandos de servidor

Un sistema de administración que recibe comandos en diferentes formatos desde
distintas fuentes: CLI (tuplas), API REST (dicts), eventos internos (dataclasses)
y mensajes de texto simples. Combina todos los patrones de `match` en un sistema real.

```python
"""
router_servidor.py
Router de comandos de administración de servidor.
Acepta comandos en múltiples formatos y los despacha a los manejadores correctos.
"""
from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Any


class OrigenComando(Enum):
    """De dónde viene el comando."""
    CLI         = auto()   # interfaz de línea de comandos
    API_REST    = auto()   # petición HTTP a la API de administración
    SCHEDULER   = auto()   # tarea programada
    WEBSOCKET   = auto()   # cliente web en tiempo real
    INTERNO     = auto()   # el propio sistema (auto-gestión)


class PrioridadComando(Enum):
    """Prioridad de ejecución del comando."""
    CRITICA  = 0
    ALTA     = 1
    NORMAL   = 2
    BAJA     = 3


@dataclass
class ComandoSistema:
    """Comando estructurado del sistema interno."""
    operacion:  str
    servicio:   str
    parametros: dict[str, Any] = field(default_factory=dict)
    prioridad:  PrioridadComando = PrioridadComando.NORMAL


@dataclass
class ResultadoDespacho:
    """Resultado del despacho de un comando."""
    ejecutado:     bool
    mensaje:       str
    requiere_ack:  bool = False
    prioridad:     PrioridadComando = PrioridadComando.NORMAL


def despachar_comando_cli(partes: list[str]) -> ResultadoDespacho:
    """Maneja comandos que vienen de la CLI como lista de strings."""
    match partes:
        case ["help"] | ["--help"] | ["-h"]:
            return ResultadoDespacho(
                True, "Comandos: start, stop, restart, status, scale, logs, config"
            )

        case ["start", servicio] | ["iniciar", servicio]:
            return ResultadoDespacho(
                True, f"Iniciando servicio '{servicio}'...", requiere_ack=True
            )

        case ["stop", servicio, "--force"] | ["detener", servicio, "--forzar"]:
            return ResultadoDespacho(
                True, f"Deteniendo '{servicio}' forzosamente",
                requiere_ack=True, prioridad=PrioridadComando.ALTA
            )

        case ["stop", servicio] | ["detener", servicio]:
            return ResultadoDespacho(
                True, f"Deteniendo '{servicio}' graciosamente (timeout 30s)",
                requiere_ack=True
            )

        case ["scale", servicio, str(n)] if n.isdigit() and int(n) > 0:
            return ResultadoDespacho(
                True, f"Escalando '{servicio}' a {n} instancias"
            )

        case ["logs", servicio, "--tail", str(n)] if n.isdigit():
            return ResultadoDespacho(
                True, f"Mostrando últimas {n} líneas de logs de '{servicio}'"
            )

        case ["logs", servicio]:
            return ResultadoDespacho(True, f"Logs de '{servicio}' (últimas 50 líneas)")

        case ["config", "get", clave]:
            return ResultadoDespacho(True, f"config.{clave} = <valor>")

        case ["config", "set", clave, valor]:
            return ResultadoDespacho(
                True, f"Configurando {clave}={valor}", requiere_ack=True
            )

        case [cmd, *_]:
            return ResultadoDespacho(False, f"Comando desconocido: '{cmd}'")

        case []:
            return ResultadoDespacho(False, "Sin comando — usa 'help'")


def despachar_comando_api(payload: dict) -> ResultadoDespacho:
    """Maneja comandos que vienen de la API REST como dicts."""
    match payload:
        case {"accion": "healthcheck"}:
            return ResultadoDespacho(True, "OK — todos los servicios operativos")

        case {"accion": "reiniciar", "servicio": str(svc), "razon": str(razon)}:
            return ResultadoDespacho(
                True, f"Reiniciando '{svc}': {razon}",
                requiere_ack=True, prioridad=PrioridadComando.ALTA
            )

        case {"accion": "reiniciar", "servicio": str(svc)}:
            return ResultadoDespacho(
                True, f"Reiniciando '{svc}' sin motivo registrado",
                requiere_ack=True
            )

        case {"accion": "escalar", "servicio": str(svc), "replicas": int(n)} if n > 0:
            return ResultadoDespacho(True, f"Escalando '{svc}' a {n} réplicas")

        case {"accion": "notificar", "canal": str(canal), "mensaje": str(msg)}:
            return ResultadoDespacho(True, f"Notificación enviada a #{canal}: {msg[:40]}")

        case {"accion": str(accion)}:
            return ResultadoDespacho(False, f"Acción API desconocida: '{accion}'")

        case _:
            return ResultadoDespacho(False, "Payload de API malformado — falta 'accion'")


def despachar_comando_interno(cmd: ComandoSistema) -> ResultadoDespacho:
    """Maneja comandos del sistema interno como dataclasses."""
    match cmd:
        case ComandoSistema(operacion="auto_escalar", servicio=svc,
                            parametros={"cpu_pct": float(cpu)}) if cpu > 80:
            replicas = int(cpu / 20)   # heurística simple
            return ResultadoDespacho(
                True, f"Auto-escalando '{svc}' a {replicas} réplicas (CPU {cpu:.1f}%)",
                prioridad=PrioridadComando.ALTA
            )

        case ComandoSistema(operacion="limpiar_cache", servicio=svc):
            return ResultadoDespacho(True, f"Caché de '{svc}' limpiada")

        case ComandoSistema(operacion="rotacion_logs", servicio=svc,
                            prioridad=PrioridadComando.CRITICA):
            return ResultadoDespacho(
                True, f"Rotación urgente de logs de '{svc}'",
                prioridad=PrioridadComando.CRITICA
            )

        case ComandoSistema(operacion=op, servicio=svc):
            return ResultadoDespacho(True, f"Operación interna '{op}' en '{svc}'")


def router_principal(origen: OrigenComando, comando: object) -> ResultadoDespacho:
    """
    Router principal — despacha según el origen y el tipo del comando.
    Combina match sobre enum y delegación a funciones especializadas.
    """
    match (origen, comando):
        # CLI: el comando llega como lista de strings
        case (OrigenComando.CLI, list(partes)):
            return despachar_comando_cli(partes)

        # API REST: el comando llega como dict
        case (OrigenComando.API_REST, dict(payload)):
            return despachar_comando_api(payload)

        # Sistema interno: comando estructurado como dataclass
        case (OrigenComando.INTERNO | OrigenComando.SCHEDULER, ComandoSistema() as cmd):
            return despachar_comando_interno(cmd)

        # WebSocket: el comando llega como string
        case (OrigenComando.WEBSOCKET, str(texto)) if texto.startswith("/"):
            partes = texto.lstrip("/").split()
            return despachar_comando_cli(partes)

        # Combinación no manejada
        case (origen_val, tipo_cmd):
            return ResultadoDespacho(
                False,
                f"Combinación no soportada: origen={origen_val.name}, tipo={type(tipo_cmd).__name__}"
            )


# ── Pruebas del router completo ──────────────────────────────────────────────

comandos_prueba: list[tuple[OrigenComando, object]] = [
    (OrigenComando.CLI,       ["help"]),
    (OrigenComando.CLI,       ["start", "nginx"]),
    (OrigenComando.CLI,       ["stop", "worker", "--force"]),
    (OrigenComando.CLI,       ["scale", "api", "5"]),
    (OrigenComando.CLI,       ["logs", "api", "--tail", "100"]),
    (OrigenComando.CLI,       ["config", "set", "timeout_ms", "3000"]),
    (OrigenComando.API_REST,  {"accion": "healthcheck"}),
    (OrigenComando.API_REST,  {"accion": "reiniciar", "servicio": "redis", "razon": "fuga de memoria detectada"}),
    (OrigenComando.API_REST,  {"accion": "escalar", "servicio": "workers", "replicas": 8}),
    (OrigenComando.INTERNO,   ComandoSistema("auto_escalar", "api-gateway", {"cpu_pct": 87.5})),
    (OrigenComando.INTERNO,   ComandoSistema("limpiar_cache", "redis")),
    (OrigenComando.SCHEDULER, ComandoSistema("rotacion_logs", "nginx", prioridad=PrioridadComando.CRITICA)),
    (OrigenComando.WEBSOCKET, "/restart nginx"),
    (OrigenComando.WEBSOCKET, "/logs api"),
]

print(f"\n{'ORIGEN':<12} {'EJECUTADO':<10} {'PRIORIDAD':<10} {'ACK':<5}  MENSAJE")
print("=" * 80)
for origen, cmd in comandos_prueba:
    resultado = router_principal(origen, cmd)
    ack = "SI" if resultado.requiere_ack else "NO"
    ok = "OK" if resultado.ejecutado else "ERR"
    print(f"{origen.name:<12} [{ok}] {'':>3}    {resultado.prioridad.name:<10} {ack:<5}  {resultado.mensaje}")
```

---

## Ejercicios propuestos

**1. Parser de versiones semánticas**

Escribe `def analizar_version(version: str) -> dict` que use `match` con secuencias
para parsear versiones en formato `"MAYOR.MENOR.PARCHE"` o `"MAYOR.MENOR"` o solo
`"MAYOR"`. Primero divide el string con `.split(".")`. Luego usa `match` sobre la
lista resultante:

- `[mayor, menor, parche]` → dict con los tres componentes como `int`
- `[mayor, menor]` → dict con `parche=0` por defecto
- `[mayor]` → dict con `menor=0`, `parche=0`
- Cualquier otro formato → `{"error": "Formato de versión inválido"}`

Incluye un guard que verifique que todos los componentes sean numéricos con `.isdigit()`.
Prueba con `"3.12.0"`, `"3.10"`, `"4"`, `"1.2.3.4"`, `"alpha.beta"`.

**2. Clasificador de eventos de CI/CD**

Dado un sistema de integración continua que emite eventos como dicts, escribe
`def manejar_evento_ci(evento: dict) -> str` usando `match` con desestructuración
de dicts para manejar:

- `{"tipo": "push", "rama": "main", "autor": autor}` → `"Deploy a producción por {autor}"`
- `{"tipo": "push", "rama": rama}` → `"Build en rama {rama}"`
- `{"tipo": "pr_abierto", "titulo": t, "numero": n}` → `"PR #{n}: {t}"`
- `{"tipo": "test_fallido", "suite": s, "fallos": int(n)} if n > 5` → `"CI roto: {n} fallos en {s}"`
- `{"tipo": "test_fallido", "suite": s, "fallos": n}` → `"{n} fallos en {s} — revisar"`
- `{"tipo": "deploy_exitoso", "entorno": e, "version": v}` → `"Deploy {v} en {e}"`

**3. Intérprete de expresiones matemáticas**

Usando `match` con secuencias de tuplas, escribe `def evaluar(expr: tuple) -> float`
que soporte:

- `("+", a, b)` → suma
- `("-", a, b)` → resta
- `("*", a, b)` → multiplicación
- `("/", a, b) if b != 0` → división
- `("/", _, 0)` → error división por cero
- `("neg", a)` → negación (`-a`)
- `("abs", a)` → valor absoluto
- `(op,)` donde `op` es `float | int` → el propio valor

Donde `a` y `b` pueden ser a su vez tuplas anidadas. Ejemplo:
`evaluar(("+", ("*", 3.0, 4.0), ("-", 10.0, 2.0)))` → `20.0`

**4. Validador de esquema JSON simple**

Escribe `def validar_esquema(datos: object, esquema: dict) -> list[str]` que
use `match` con tipos para verificar que `datos` cumple con el esquema:

El esquema es un dict como `{"nombre": str, "edad": int, "activo": bool}`.
La función debe verificar que `datos` es un dict y que cada clave del esquema
existe en `datos` con el tipo correcto. Devuelve lista de errores (vacía si todo OK).

Usa `match` para clasificar el tipo de error en cada campo:
- Si el campo no existe → `"Campo '{campo}' requerido"`
- Si el tipo no coincide → `"'{campo}' debe ser {tipo} pero es {tipo_actual}"`
- Si el valor es `None` → `"'{campo}' no puede ser None"`

---

## Resumen de la página 3

- `match/case` (Python 3.10+) no es un `switch` de igualdad simple — es un motor de desestructuración que puede verificar tipo, forma y contenido de un objeto en una sola expresión.
- Los patrones se evalúan en orden de arriba a abajo. El primero que coincide gana. Si ningún patrón coincide, el `match` simplemente no hace nada (no hay error).
- El operador `|` en un patrón agrupa múltiples valores en una sola rama: `case 400 | 422 | 415:`. Todos los lados del `|` deben capturar las mismas variables.
- Los **guards** (`case patron if condicion:`) añaden una condición booleana extra. Si el guard falla, Python sigue buscando el siguiente `case`, como si el patrón no hubiera coincidido.
- Los **patrones de tipo** `case str(s):`, `case int(n):` verifican el tipo del objeto y ligan el valor a la variable. `bool` es subclase de `int` — colocar `case bool(b):` antes de `case int(n):`.
- Los **patrones de dict** `case {"clave": valor}:` verifican que esas claves existan con esos valores. Las claves no mencionadas se ignoran — esto permite patrones parciales sobre dicts con muchos campos.
- Los **patrones de secuencia** `case [a, b]:` y `case [primero, *resto]:` desestructuran listas y tuplas. `*resto` captura cero o más elementos como lista.
- Los **patrones de dataclass** `case MiClase(campo=v):` verifican `isinstance` y desestructuran campos en una sola expresión. Son mucho más limpios que múltiples `isinstance` + accesos a atributos.
- `case _:` es el wildcard que captura cualquier valor sin ligarlo a una variable. `case nombre:` captura cualquier valor y lo liga a la variable `nombre` para usarla en el cuerpo.

---

> **Siguiente página →** Página 4: Bucles — `for`, `while`, `break`, `continue`, `else` en bucles, `enumerate`, `zip`, `range` y comprensiones de lista.
