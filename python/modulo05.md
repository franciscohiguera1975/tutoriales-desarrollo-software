# Tutorial Python — Página 5
## Módulo 1 · Entorno y Fundamentos
### Funciones — definición, parámetros, `*args`, `**kwargs`, lambdas y closures

---

## Definir funciones

En Python las funciones se definen con `def`. No existe `fun` de Kotlin ni
`function` de JavaScript — solo `def`. El cuerpo se delimita por indentación.

```python
# Sintaxis básica
def saludar(nombre: str) -> str:
    """Docstring: describe qué hace la función."""
    return f"Hola, {nombre}"

print(saludar("Ana"))   # Hola, Ana

# Sin tipo de retorno anotado — implícitamente None
def registrar(mensaje: str) -> None:
    print(f"[LOG] {mensaje}")

# Múltiples valores de retorno — en realidad retorna una tupla
def dividir_con_resto(dividendo: int, divisor: int) -> tuple[int, int]:
    """Devuelve (cociente, resto) como tupla."""
    return dividendo // divisor, dividendo % divisor   # Python empaqueta en tupla

cociente, resto = dividir_con_resto(17, 5)   # desestructuración
print(f"17 ÷ 5 = {cociente} con resto {resto}")   # 17 ÷ 5 = 3 con resto 2

# Ignorar uno de los valores con _
cociente_solo, _ = dividir_con_resto(17, 5)

# Funciones como objetos de primera clase
def aplicar(funcion, valor):
    return funcion(valor)

print(aplicar(str.upper, "hola"))   # HOLA
print(aplicar(abs, -42))            # 42
```

---

## Tipos de parámetros

### Parámetros posicionales y con valor por defecto

```python
def crear_conexion(
    host:     str,
    puerto:   int   = 5432,       # valor por defecto
    timeout:  int   = 30,
    ssl:      bool  = True,
    nombre:   str   = "default",
) -> dict[str, str | int | bool]:
    """Crea una configuración de conexión a base de datos."""
    return {
        "host":    host,
        "puerto":  puerto,
        "timeout": timeout,
        "ssl":     ssl,
        "nombre":  nombre,
    }

# Posicional — por orden
conn1 = crear_conexion("10.0.1.1", 5432)

# Keyword argument — por nombre (orden no importa)
conn2 = crear_conexion(host="10.0.1.2", ssl=False, nombre="replica")

# Mezcla: posicionales primero, luego keyword
conn3 = crear_conexion("10.0.1.3", timeout=60, nombre="analytics")

print(conn1)
print(conn2)
```

### Parámetros keyword-only (después de `*`)

El `*` sin nombre obliga a que todos los argumentos siguientes
se pasen solo por nombre — evita bugs por orden incorrecto:

```python
def enviar_correo(
    destinatario: str,
    asunto:       str,
    *,                          # — todo lo que viene después es keyword-only
    cuerpo:       str,
    cc:           list[str] | None = None,
    prioridad:    str             = "normal",
    html:         bool            = False,
) -> bool:
    """
    Envía un correo. Los parámetros tras * son keyword-only:
    el llamador DEBE especificar el nombre, nunca por posición.
    """
    destinatarios = [destinatario] + (cc or [])
    modo = "HTML" if html else "texto plano"
    print(
        f"Enviando a {', '.join(destinatarios)}\n"
        f"  Asunto   : {asunto}\n"
        f"  Modo     : {modo}\n"
        f"  Prioridad: {prioridad}\n"
        f"  Cuerpo   : {cuerpo[:50]}..."
    )
    return True

# Correcto — 'cuerpo' obligatoriamente por nombre
enviar_correo(
    "ana@ejemplo.com",
    "Informe mensual",
    cuerpo="Adjunto el informe del mes de marzo...",
    cc=["jefe@ejemplo.com"],
    prioridad="alta",
)

# Incorrecto — lanzaría TypeError: toma 2 posicionales pero se dieron 3
# enviar_correo("ana@ejemplo.com", "Asunto", "cuerpo por posición")
```

### Parámetros positional-only (antes de `/`)

El `/` obliga a los parámetros anteriores a pasarse solo por posición.
Útil para APIs de bajo nivel o para imitar la firma de funciones C:

```python
def calcular_distancia(
    x1: float,
    y1: float,
    /,                   # x1 e y1 son positional-only
    x2: float,
    y2: float,
    *,                   # x2 e y2 son keyword-only
    unidad: str = "km",
) -> float:
    """
    Calcula la distancia euclidiana entre dos puntos.
    x1, y1: solo por posición.
    x2, y2: solo por nombre.
    """
    from math import sqrt
    distancia = sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2)
    factor = 1.0 if unidad == "km" else 0.621371   # km → millas
    return distancia * factor

# Correcto
d = calcular_distancia(0, 0, x2=3, y2=4, unidad="km")
print(f"Distancia: {d:.2f} km")   # 5.00 km

# Incorrecto — x1 es positional-only
# calcular_distancia(x1=0, y1=0, x2=3, y2=4)  # TypeError
```

---

## `*args` y `**kwargs`

### `*args` — número variable de argumentos posicionales

```python
def sumar(*numeros: float) -> float:
    """Suma una cantidad arbitraria de números."""
    # *numeros se convierte en una tupla dentro de la función
    return sum(numeros)

print(sumar(1, 2, 3))           # 6.0
print(sumar(10.5, 20.5, 30.0))  # 61.0
print(sumar())                  # 0.0

# Combinando parámetros normales con *args
def registrar_evento(nivel: str, *mensajes: str, separador: str = " | ") -> str:
    """Registra un evento con nivel y múltiples mensajes."""
    contenido = separador.join(mensajes)
    return f"[{nivel.upper()}] {contenido}"

print(registrar_evento("error", "timeout", "servidor: 10.0.0.1", "intento: 3/3"))
# [ERROR] timeout | servidor: 10.0.0.1 | intento: 3/3

# Desempaquetar una lista/tupla en una llamada con *
valores = [5, 10, 15, 20]
print(sumar(*valores))   # equivale a sumar(5, 10, 15, 20) → 50.0
```

### `**kwargs` — número variable de argumentos keyword

```python
def construir_query_sql(tabla: str, **condiciones: str | int | bool) -> str:
    """
    Construye dinámicamente una cláusula WHERE.
    Cada keyword argument se convierte en una condición.
    """
    # **condiciones es un dict dentro de la función
    if not condiciones:
        return f"SELECT * FROM {tabla}"

    clausulas = []
    for columna, valor in condiciones.items():
        if isinstance(valor, bool):
            clausulas.append(f"{columna} = {str(valor).upper()}")
        elif isinstance(valor, str):
            clausulas.append(f"{columna} = '{valor}'")
        else:
            clausulas.append(f"{columna} = {valor}")

    where = " AND ".join(clausulas)
    return f"SELECT * FROM {tabla} WHERE {where}"

print(construir_query_sql("usuarios"))
# SELECT * FROM usuarios

print(construir_query_sql("usuarios", activo=True, rol="admin"))
# SELECT * FROM usuarios WHERE activo = TRUE AND rol = 'admin'

print(construir_query_sql("pedidos", estado="pendiente", cliente_id=42, urgente=True))
# SELECT * FROM pedidos WHERE estado = 'pendiente' AND cliente_id = 42 AND urgente = TRUE

# Desempaquetar un dict en una llamada con **
parametros = {"activo": True, "departamento": "ingenieria", "nivel": 3}
print(construir_query_sql("empleados", **parametros))
```

### Combinar `*args` y `**kwargs` — función genérica de decoración

```python
def con_reintentos(funcion, *args, max_reintentos: int = 3, **kwargs):
    """
    Ejecuta 'funcion' con los argumentos dados,
    reintentando hasta max_reintentos veces si falla.
    """
    ultimo_error = None
    for intento in range(1, max_reintentos + 1):
        try:
            resultado = funcion(*args, **kwargs)
            if intento > 1:
                print(f"  Éxito en el intento {intento}")
            return resultado
        except Exception as e:
            ultimo_error = e
            print(f"  Intento {intento}/{max_reintentos} fallido: {e}")
    raise RuntimeError(
        f"Todos los reintentos agotados. Último error: {ultimo_error}"
    )

# Simular función que falla las primeras dos veces
contador_llamadas = 0
def consultar_api(endpoint: str, timeout: int = 10) -> dict:
    global contador_llamadas
    contador_llamadas += 1
    if contador_llamadas < 3:
        raise ConnectionError(f"Timeout al conectar a {endpoint}")
    return {"datos": [1, 2, 3], "endpoint": endpoint}

resultado = con_reintentos(consultar_api, "/api/usuarios", timeout=5)
print(f"Resultado: {resultado}")
```

---

## Type hints en funciones

```python
from typing import Callable, Any
from collections.abc import Generator, Iterator

# Tipos de retorno avanzados
def obtener_config(clave: str) -> str | None:
    config = {"host": "localhost", "puerto": "5432"}
    return config.get(clave)

# Callable — tipo para funciones pasadas como argumentos
type Transformador = Callable[[str], str]
type Predicado      = Callable[[dict], bool]

def filtrar_y_transformar(
    registros:    list[dict],
    predicado:    Predicado,
    transformador: Transformador | None = None,
) -> list[dict]:
    """Filtra registros según predicado y opcionalmente transforma."""
    resultado = [r for r in registros if predicado(r)]
    if transformador:
        resultado = [{**r, "nombre": transformador(r["nombre"])} for r in resultado]
    return resultado

usuarios = [
    {"id": 1, "nombre": "ana garcía",  "activo": True,  "rol": "admin"},
    {"id": 2, "nombre": "bob martinez", "activo": False, "rol": "user"},
    {"id": 3, "nombre": "carlos ruiz",  "activo": True,  "rol": "user"},
]

activos = filtrar_y_transformar(
    usuarios,
    predicado=lambda u: u["activo"],
    transformador=str.title,
)
print(activos)
# [{'id': 1, 'nombre': 'Ana García', ...}, {'id': 3, 'nombre': 'Carlos Ruiz', ...}]
```

---

## Lambdas

Las lambdas son funciones anónimas de una sola expresión. Son útiles
para funciones simples de transformación o como argumento de `sorted()`,
`map()`, `filter()`:

```python
# Sintaxis: lambda parámetros: expresión
doblar    = lambda x: x * 2
cuadrado  = lambda x: x ** 2
normalizar = lambda s: s.strip().lower().replace(" ", "_")

print(doblar(5))              # 10
print(normalizar("  Mi Función  "))  # mi_función

# El caso de uso CORRECTO — pasar a sorted/max/min/map
servidores = [
    {"nombre": "web-01",   "carga": 0.82},
    {"nombre": "web-02",   "carga": 0.41},
    {"nombre": "db-01",    "carga": 0.95},
    {"nombre": "cache-01", "carga": 0.12},
]

# Ordenar por carga descendente
por_carga = sorted(servidores, key=lambda s: s["carga"], reverse=True)
for s in por_carga:
    print(f"  {s['nombre']}: {s['carga']:.0%}")

# Servidor con más carga
mas_cargado = max(servidores, key=lambda s: s["carga"])
print(f"Más cargado: {mas_cargado['nombre']}")

# Cuándo NO usar lambda — si la lógica es compleja, usar def
# MAL: lambda x: x if x > 0 else -x if x < 0 else 0
# BIEN:
def valor_absoluto_personalizado(x: float) -> float:
    if x > 0:
        return x
    elif x < 0:
        return -x
    return 0.0
```

---

## Funciones de orden superior

```python
# map() — aplica una función a cada elemento
temperaturas_c = [0, 20, 37, 100]
temperaturas_f = list(map(lambda c: c * 9/5 + 32, temperaturas_c))
print(temperaturas_f)   # [32.0, 68.0, 98.6, 212.0]

# filter() — filtra elementos según un predicado
puertos = [22, 80, 443, 8080, 3306, 9200, 6379, 27017]
puertos_privilegiados = list(filter(lambda p: p < 1024, puertos))
print(puertos_privilegiados)   # [22, 80, 443]

# En Python moderno se prefieren comprensiones de lista a map/filter:
# temperaturas_f   = [c * 9/5 + 32 for c in temperaturas_c]
# puertos_privil.  = [p for p in puertos if p < 1024]

# Pasar funciones como argumentos — diseño flexible
from functools import reduce

def pipeline(valor: str, *transformaciones: Callable[[str], str]) -> str:
    """Aplica una secuencia de transformaciones a un string."""
    resultado = valor
    for transformar in transformaciones:
        resultado = transformar(resultado)
    return resultado

texto_bruto = "  Servidor Web  Principal  "
limpio = pipeline(
    texto_bruto,
    str.strip,
    str.lower,
    lambda s: s.replace("  ", " "),
    lambda s: s.replace(" ", "-"),
)
print(limpio)   # servidor-web-principal
```

---

## Closures

Un closure es una función que captura y recuerda variables del
scope donde fue definida, incluso después de que ese scope termine:

```python
# Ejemplo básico de closure
def crear_contador(inicio: int = 0, paso: int = 1):
    """Fábrica de contadores independientes."""
    valor = inicio   # capturado por la función interna

    def incrementar() -> int:
        nonlocal valor    # 'nonlocal' permite modificar la variable capturada
        valor += paso
        return valor

    def reiniciar() -> None:
        nonlocal valor
        valor = inicio

    def obtener() -> int:
        return valor

    # Devolver múltiples funciones que comparten el estado
    return incrementar, reiniciar, obtener


# Dos contadores independientes con su propio estado
contar_solicitudes, resetear_solicitudes, total_solicitudes = crear_contador(0)
contar_errores,     resetear_errores,     total_errores     = crear_contador(0)

# Simular procesamiento de peticiones
for _ in range(5):
    contar_solicitudes()

for _ in range(2):
    contar_errores()

print(f"Solicitudes: {total_solicitudes()}")   # 5
print(f"Errores: {total_errores()}")           # 2
resetear_solicitudes()
print(f"Solicitudes tras reset: {total_solicitudes()}")  # 0


# Closure más útil — fábrica de validadores configurables
def crear_validador_ip(rango_permitido: str):
    """
    Retorna una función validadora para un rango específico.
    El rango queda capturado en el closure.
    """
    prefijo = rango_permitido.rstrip(".*")   # "192.168.1.*" → "192.168.1"

    def validar(ip: str) -> bool:
        partes = ip.split(".")
        if len(partes) != 4:
            return False
        try:
            octetos = [int(p) for p in partes]
        except ValueError:
            return False
        if not all(0 <= o <= 255 for o in octetos):
            return False
        return ip.startswith(prefijo)

    # Metadatos en la función retornada
    validar.__name__ = f"validar_rango_{rango_permitido}"
    validar.__doc__  = f"Valida que la IP esté en el rango {rango_permitido}"
    return validar


# Crear validadores para distintos rangos
es_red_interna  = crear_validador_ip("192.168.1.*")
es_red_dmz      = crear_validador_ip("10.0.0.*")
es_red_mgmt     = crear_validador_ip("172.16.0.*")

ips_prueba = ["192.168.1.100", "10.0.0.5", "172.16.0.1", "8.8.8.8", "192.168.2.1"]

for ip in ips_prueba:
    interna = "interna" if es_red_interna(ip) else "-"
    dmz     = "dmz"     if es_red_dmz(ip)     else "-"
    mgmt    = "mgmt"    if es_red_mgmt(ip)     else "-"
    print(f"  {ip:<18} interna:{interna:<8} dmz:{dmz:<5} mgmt:{mgmt}")
```

---

## Decoradores básicos

Un decorador es una función que envuelve a otra función para añadir
comportamiento sin modificarla. Se aplica con la sintaxis `@decorador`:

```python
import time
import functools
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
T = TypeVar("T")


def medir_tiempo(funcion: Callable[P, T]) -> Callable[P, T]:
    """Decorador que mide el tiempo de ejecución de una función."""
    @functools.wraps(funcion)   # preserva __name__, __doc__ etc.
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
        inicio = time.perf_counter()
        resultado = funcion(*args, **kwargs)
        duracion = time.perf_counter() - inicio
        print(f"[TIMER] {funcion.__name__} tardó {duracion * 1000:.3f}ms")
        return resultado
    return envoltura


def reintentar(max_intentos: int = 3, demora: float = 0.5):
    """Decorador parametrizado que reintenta la función si falla."""
    def decorador(funcion: Callable[P, T]) -> Callable[P, T]:
        @functools.wraps(funcion)
        def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
            for intento in range(1, max_intentos + 1):
                try:
                    return funcion(*args, **kwargs)
                except Exception as e:
                    if intento == max_intentos:
                        raise
                    print(f"  [{funcion.__name__}] Intento {intento} fallido: {e}. Reintentando...")
                    time.sleep(demora)
            raise RuntimeError("No debería llegar aquí")
        return envoltura
    return decorador


# Aplicar decoradores — el orden importa (se aplican de abajo hacia arriba)
@medir_tiempo
@reintentar(max_intentos=3, demora=0.1)
def consultar_base_datos(query: str) -> list[dict]:
    """Simula una consulta a base de datos con fallos intermitentes."""
    import random
    if random.random() < 0.6:   # 60% de probabilidad de fallo
        raise ConnectionError("Timeout de base de datos")
    return [{"id": 1, "query": query}]


# La función decorada se usa exactamente igual que la original
try:
    resultado = consultar_base_datos("SELECT * FROM usuarios LIMIT 10")
    print(f"Resultado: {resultado}")
except RuntimeError as e:
    print(f"Fallo definitivo: {e}")
```

---

## Generadores con `yield`

Los generadores producen valores de forma perezosa (lazy),
uno a la vez, sin cargar toda la secuencia en memoria:

```python
from collections.abc import Generator

def leer_chunks_archivo(
    ruta: str,
    tamano_chunk: int = 65_536,
) -> Generator[bytes, None, None]:
    """
    Genera chunks de un archivo binario sin cargarlo completo en RAM.
    Útil para procesar archivos de varios GB.
    """
    with open(ruta, "rb") as archivo:
        while chunk := archivo.read(tamano_chunk):
            yield chunk


def generar_ids_correlacion(prefijo: str = "req") -> Generator[str, None, None]:
    """Genera IDs únicos de correlación infinitamente."""
    import uuid
    while True:
        yield f"{prefijo}-{uuid.uuid4().hex[:12]}"


# Generar números de Fibonacci de forma perezosa
def fibonacci() -> Generator[int, None, None]:
    """Secuencia de Fibonacci infinita."""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b


# Tomar solo los primeros N de un generador infinito
from itertools import islice

primeros_10_fib = list(islice(fibonacci(), 10))
print(primeros_10_fib)
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Generador de rangos de IP para escaneo
def rango_ips(red: str, inicio: int = 1, fin: int = 254) -> Generator[str, None, None]:
    """Genera IPs de una red /24."""
    prefijo = ".".join(red.split(".")[:3])
    for ultimo_octeto in range(inicio, fin + 1):
        yield f"{prefijo}.{ultimo_octeto}"

# Usar el generador — no se generan todas las IPs a la vez
escaner = rango_ips("192.168.1.0")
primeras_5 = [next(escaner) for _ in range(5)]
print(primeras_5)   # ['192.168.1.1', '192.168.1.2', ..., '192.168.1.5']
```

---

## Ejemplo aplicado — pipeline funcional de transformación de datos

```python
"""
pipeline_datos.py
Sistema de transformación de datos en pipeline usando funciones
de orden superior, closures y generadores. Simula ETL de logs.
"""
from __future__ import annotations

import re
from collections.abc import Callable, Generator, Iterable
from dataclasses import dataclass
from datetime import datetime


# ── Modelos ─────────────────────────────────────────────────────────────────

@dataclass
class EntradaLog:
    timestamp:  datetime
    nivel:      str
    servicio:   str
    mensaje:    str
    duracion_ms: float | None = None


@dataclass
class MetricaLog:
    servicio:    str
    nivel:       str
    hora:        str    # HH:MM
    conteo:      int
    duracion_avg: float | None


# ── Etapa EXTRACT ────────────────────────────────────────────────────────────

PATRON_LOG = re.compile(
    r"(?P<fecha>\d{4}-\d{2}-\d{2}) (?P<hora>\d{2}:\d{2}:\d{2})\s+"
    r"(?P<nivel>DEBUG|INFO|WARN|ERROR|FATAL)\s+"
    r"\[(?P<servicio>[^\]]+)\]\s+"
    r"(?P<mensaje>.+?)"
    r"(?:\s+duration=(?P<duracion>\d+)ms)?$"
)

def parsear_lineas(lineas: Iterable[str]) -> Generator[EntradaLog, None, None]:
    """Parsea líneas de log crudas en EntradaLog estructurados."""
    for num_linea, linea in enumerate(lineas, start=1):
        linea = linea.strip()
        if not linea:
            continue
        m = PATRON_LOG.match(linea)
        if m is None:
            continue   # línea malformada — ignorar
        yield EntradaLog(
            timestamp   = datetime.fromisoformat(f"{m['fecha']}T{m['hora']}"),
            nivel       = m["nivel"],
            servicio    = m["servicio"],
            mensaje     = m["mensaje"],
            duracion_ms = float(m["duracion"]) if m["duracion"] else None,
        )


# ── Etapa TRANSFORM ──────────────────────────────────────────────────────────

type FiltroLog      = Callable[[EntradaLog], bool]
type TransformLog   = Callable[[EntradaLog], EntradaLog]

def crear_filtro_nivel(niveles_minimos: list[str]) -> FiltroLog:
    """Closure: retorna un filtro que acepta solo los niveles especificados."""
    niveles_set = set(niveles_minimos)
    def filtrar(entrada: EntradaLog) -> bool:
        return entrada.nivel in niveles_set
    filtrar.__name__ = f"filtro_nivel_{'+'.join(sorted(niveles_minimos))}"
    return filtrar


def crear_filtro_servicio(patron: str) -> FiltroLog:
    """Closure: retorna un filtro que acepta servicios que coincidan con el patrón."""
    regex = re.compile(patron, re.IGNORECASE)
    def filtrar(entrada: EntradaLog) -> bool:
        return bool(regex.search(entrada.servicio))
    return filtrar


def crear_sanitizador(campos_sensibles: list[str]) -> TransformLog:
    """
    Closure: retorna una función que reemplaza datos sensibles en mensajes.
    Útil para ofuscar tokens, contraseñas, IPs antes de exportar logs.
    """
    patron_combinado = re.compile(
        "|".join(rf"(?<={campo}=)\S+" for campo in campos_sensibles),
        re.IGNORECASE,
    )
    def sanitizar(entrada: EntradaLog) -> EntradaLog:
        from dataclasses import replace
        mensaje_limpio = patron_combinado.sub("***REDACTED***", entrada.mensaje)
        return replace(entrada, mensaje=mensaje_limpio)
    return sanitizar


def aplicar_pipeline(
    entradas:       Iterable[EntradaLog],
    filtros:        list[FiltroLog],
    transformaciones: list[TransformLog],
) -> Generator[EntradaLog, None, None]:
    """
    Aplica filtros y transformaciones en pipeline.
    Cada entrada pasa por todos los filtros (AND) y luego por todas las transformaciones.
    """
    for entrada in entradas:
        # Aplicar todos los filtros — si alguno falla, descartar
        if not all(f(entrada) for f in filtros):
            continue
        # Aplicar transformaciones en cadena
        resultado = entrada
        for transformar in transformaciones:
            resultado = transformar(resultado)
        yield resultado


# ── Etapa LOAD ───────────────────────────────────────────────────────────────

def agrupar_por_servicio_hora(
    entradas: Iterable[EntradaLog],
) -> dict[tuple[str, str, str], MetricaLog]:
    """
    Agrega entradas por (servicio, nivel, hora) y calcula métricas.
    Devuelve un dict de MetricaLog para su exportación.
    """
    acumulador: dict[tuple[str, str, str], list[EntradaLog]] = {}

    for entrada in entradas:
        clave = (entrada.servicio, entrada.nivel, entrada.timestamp.strftime("%H:%M"))
        acumulador.setdefault(clave, []).append(entrada)

    resultado = {}
    for (servicio, nivel, hora), grupo in acumulador.items():
        duraciones = [e.duracion_ms for e in grupo if e.duracion_ms is not None]
        duracion_avg = sum(duraciones) / len(duraciones) if duraciones else None
        resultado[(servicio, nivel, hora)] = MetricaLog(
            servicio=servicio, nivel=nivel, hora=hora,
            conteo=len(grupo), duracion_avg=duracion_avg,
        )
    return resultado


# ── Programa principal ───────────────────────────────────────────────────────

LOGS_CRUDOS = """
2025-03-15 14:01:03  INFO  [api-gateway]   GET /api/users auth=Bearer.eyJhbGc duration=45ms
2025-03-15 14:01:04  ERROR [user-service]  DB timeout password=secreto123 duration=3000ms
2025-03-15 14:01:05  WARN  [api-gateway]   Rate limit alcanzado para ip=192.168.1.50 duration=2ms
2025-03-15 14:01:06  INFO  [auth-service]  Login exitoso token=eyJhbGciOiJIUzI duration=12ms
2025-03-15 14:01:07  ERROR [user-service]  Conexión rechazada duration=5010ms
2025-03-15 14:01:08  DEBUG [api-gateway]   Cache hit ratio=87%
2025-03-15 14:01:09  FATAL [user-service]  Disco lleno — sistema detenido
2025-03-15 14:02:01  INFO  [api-gateway]   GET /health duration=3ms
2025-03-15 14:02:05  ERROR [auth-service]  Token inválido api_key=sk-prod-abc123 duration=8ms
"""

# Construir el pipeline declarativamente
filtros = [
    crear_filtro_nivel(["WARN", "ERROR", "FATAL"]),
    # crear_filtro_servicio("api-gateway"),  # descomentar para filtrar por servicio
]
transformaciones = [
    crear_sanitizador(["password", "token", "api_key", "auth"]),
]

# Ejecutar el pipeline en modo lazy (generadores)
entradas_crudas = parsear_lineas(LOGS_CRUDOS.strip().splitlines())
entradas_procesadas = aplicar_pipeline(entradas_crudas, filtros, transformaciones)

print("=== Entradas procesadas ===")
lista_procesada = list(entradas_procesadas)   # materializar para reutilizar
for entrada in lista_procesada:
    duracion = f" [{entrada.duracion_ms:.0f}ms]" if entrada.duracion_ms else ""
    print(f"  [{entrada.nivel:<5}] [{entrada.servicio}] {entrada.mensaje}{duracion}")

print("\n=== Métricas por servicio/nivel/hora ===")
metricas = agrupar_por_servicio_hora(lista_procesada)
for m in sorted(metricas.values(), key=lambda x: (x.servicio, x.nivel)):
    duracion = f"avg {m.duracion_avg:.0f}ms" if m.duracion_avg else "sin métricas"
    print(f"  {m.hora}  {m.servicio:<15} {m.nivel:<6} × {m.conteo}  {duracion}")
```

---

## Ejercicios propuestos

1. **Decorador de caché LRU manual** — Implementa un decorador `@cache(max_size=128)`
   que recuerde los últimos `max_size` resultados de una función. Usa un `dict`
   como almacén y una lista para rastrear el orden (LRU = evictar el más antiguo).
   Aplícalo a una función `fibonacci_recursivo(n: int) -> int` y compara el tiempo
   con y sin caché para `n=35`.

2. **Fábrica de validadores con closures** — Crea `def crear_validador(**reglas)`
   que acepte keyword arguments como `min_longitud=8`, `max_longitud=64`,
   `requiere_mayuscula=True`, `requiere_numero=True`, `requiere_especial=True`.
   Devuelve una función `validar(texto: str) -> tuple[bool, list[str]]` que
   retorna si es válido y la lista de errores. Útil para validar contraseñas.

3. **Pipeline de procesamiento de CSV** — Escribe un generador `leer_csv(ruta)`
   que lea un archivo CSV línea a línea sin cargarlo completo (usa `yield`).
   Crea funciones `filtrar(predicado)` y `transformar(funcion)` que devuelven
   generadores. Compón el pipeline para leer un CSV de 1M+ filas, filtrar
   las que tengan `estado="activo"` y transformar el campo `precio` a `float`.
   Mide el uso de memoria con `tracemalloc`.

4. **Función `curry` genérica** — Implementa `curry(funcion)` que convierte
   una función de N argumentos en una cadena de funciones de 1 argumento.
   Ejemplo: `suma_curried = curry(lambda a, b, c: a + b + c)`; después
   `suma_curried(1)(2)(3)` devuelve `6`. Usa `inspect.signature` para
   obtener el número de parámetros. Prueba con funciones de 2, 3 y 4 args.

---

## Resumen de la página 3

- Las funciones Python se definen con `def`. Retornan `None` implícitamente si no tienen `return`. Pueden retornar múltiples valores mediante tuplas que se desestructuran en la llamada.
- Los parámetros tras `*` son keyword-only: el llamador DEBE usar el nombre. Los parámetros antes de `/` son positional-only: no se puede usar el nombre. Esto permite APIs más expresivas y seguras.
- `*args` captura argumentos posicionales extras como tupla. `**kwargs` captura keyword arguments extras como dict. Ambos pueden desempaquetarse en otra llamada con `*lista` y `**dict`.
- Las lambdas son funciones anónimas de una expresión. Son apropiadas como argumento de `sorted(key=...)`, `max(key=...)` y similares. Para lógica compleja, usar `def`.
- Los closures capturan variables del scope externo. `nonlocal` permite modificar la variable capturada. Son la base para fábricas de funciones y callbacks configurables.
- Los decoradores envuelven funciones con `@decorator`. `@functools.wraps` preserva los metadatos de la función original (`__name__`, `__doc__`). Los decoradores parametrizados son funciones que devuelven decoradores.
- Los generadores con `yield` producen valores de forma perezosa — esenciales para procesar grandes volúmenes de datos sin cargar todo en memoria. `itertools.islice` permite tomar N elementos de un generador infinito.
- `Callable[[int, str], bool]` y `ParamSpec` son los tipos para anotar funciones que aceptan otras funciones como parámetros — imprescindible para decoradores tipados correctamente.

---

> **Siguiente página →** Página 6: Funciones avanzadas — `*args`, `**kwargs`, lambdas, closures y generadores. — *args, **kwargs, lambdas, closures y generadores.
> `list`, `tuple`, `dict`, `set`, comprensiones, `map/filter/reduce`
> e `itertools`.
