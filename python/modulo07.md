# Tutorial Python — Página 7
## Módulo 2 · Colecciones y Programación Funcional
### `list`, `tuple`, `dict`, `set` y comprensiones

---

## `list` — lista ordenada y mutable

La colección más usada en Python. Equivale a `MutableList<T>` en Kotlin
y a `List<T>` de Dart (que también es mutable por defecto).

```python
# Creación
servidores: list[str] = ["web-01", "web-02", "db-01", "cache-01"]
numeros:    list[int] = list(range(1, 6))      # [1, 2, 3, 4, 5]
vacia:      list      = []
desde_iter: list[int] = list(range(0, 100, 10))  # [0, 10, 20, ..., 90]

# Acceso por índice (base 0, negativos desde el final)
print(servidores[0])    # web-01
print(servidores[-1])   # cache-01

# Longitud
print(len(servidores))  # 4
```

### Métodos completos de list

```python
registros: list[dict] = [
    {"id": 3, "host": "db-01",    "activo": True},
    {"id": 1, "host": "web-01",   "activo": True},
    {"id": 2, "host": "web-02",   "activo": False},
]

# append — añadir al final — O(1)
registros.append({"id": 4, "host": "cache-01", "activo": True})

# insert — insertar en posición — O(n)
registros.insert(0, {"id": 0, "host": "lb-01", "activo": True})

# extend — añadir todos los elementos de un iterable — O(k)
nuevos = [{"id": 5, "host": "worker-01", "activo": True}]
registros.extend(nuevos)

# remove — eliminar la primera ocurrencia — O(n)
# registros.remove({"id": 0, ...})  # elimina por valor igual

# pop — extraer y devolver por índice — O(1) al final, O(n) en el medio
ultimo = registros.pop()        # extrae el último
segundo = registros.pop(1)      # extrae el de índice 1

# index — índice de la primera ocurrencia — O(n)
hosts = ["web-01", "web-02", "db-01", "web-01"]
print(hosts.index("web-01"))     # 0
print(hosts.index("web-01", 1))  # 3 — buscar desde índice 1

# count — número de apariciones — O(n)
print(hosts.count("web-01"))     # 2

# sort — ordenar en su lugar (modifica la lista) — O(n log n)
ips = ["10.0.1.3", "10.0.0.1", "10.0.1.1", "10.0.0.5"]
ips.sort()                          # lexicográfico
ips.sort(key=lambda ip: [int(o) for o in ip.split(".")])  # numérico

# sorted — devuelve NUEVA lista ordenada, no modifica la original
ips_ordenadas = sorted(ips, reverse=True)  # descendente

# reverse — invertir en su lugar
ips.reverse()

# clear — vaciar la lista
# ips.clear()  # ips queda como []

# copy — copia superficial (shallow copy)
copia = ips.copy()   # equivale a ips[:]

# in / not in — comprobación de pertenencia — O(n)
print("10.0.0.1" in ips)       # True o False
```

### Slicing avanzado

```python
datos = list(range(10))   # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# [inicio:fin:paso]  — inicio incluido, fin excluido
print(datos[2:7])      # [2, 3, 4, 5, 6]
print(datos[::2])      # [0, 2, 4, 6, 8]   — índices pares
print(datos[1::2])     # [1, 3, 5, 7, 9]   — índices impares
print(datos[::-1])     # [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]   — invertida
print(datos[-3:])      # [7, 8, 9]   — últimos 3
print(datos[:-3])      # [0, 1, 2, 3, 4, 5, 6]   — todos excepto los 3 últimos
print(datos[2:8:3])    # [2, 5]   — de 2 a 7, de 3 en 3

# Asignación con slice — reemplazar un rango
bytes_paquete = [0x45, 0x00, 0x00, 0x28, 0xFF, 0xFF]
bytes_paquete[2:4] = [0x00, 0x64]   # reemplazar bytes 2 y 3
print(bytes_paquete)   # [0x45, 0x00, 0x00, 0x64, 0xFF, 0xFF]

# Deleción con slice
del bytes_paquete[4:]   # eliminar desde índice 4 hasta el final

# Slicing para ventanas deslizantes (rolling window)
serie_temporal = [10, 20, 35, 28, 45, 60, 52, 70, 85, 90]
tamano_ventana = 3
promedios = [
    sum(serie_temporal[i:i + tamano_ventana]) / tamano_ventana
    for i in range(len(serie_temporal) - tamano_ventana + 1)
]
print([round(p, 1) for p in promedios])
# [21.7, 27.7, 36.0, 44.3, 52.3, 60.7, 69.0, 81.7]
```

---

## `tuple` — colección ordenada e inmutable

Las tuplas son secuencias inmutables. Se usan para representar datos
que no deben cambiar: coordenadas, registros de BD, pares clave-valor.

```python
# Creación — paréntesis opcionales, la coma es lo que hace la tupla
coordenadas:    tuple[float, float]        = (40.4168, -3.7038)
version:        tuple[int, int, int]       = (3, 12, 3)
endpoint:       tuple[str, int]            = ("api.ejemplo.com", 443)
singleton:      tuple[str]                 = ("solo",)    # la coma importa
vacia:          tuple                      = ()

# Desestructuración
lat, lon = coordenadas
mayor, menor, parche = version
print(f"Python {mayor}.{menor}.{parche}")   # Python 3.12.3

# Las tuplas son más rápidas y ligeras que las listas
# Usarlas cuando los datos son fijos (configuración, constantes)

# Tuplas anidadas — representar filas de base de datos
columnas = ("id", "nombre", "email", "rol", "activo")
filas: list[tuple[int, str, str, str, bool]] = [
    (1, "Ana García",     "ana@ejemplo.com",   "admin",  True),
    (2, "Bob Martínez",   "bob@ejemplo.com",   "user",   False),
    (3, "Carlos Ruiz",    "carlos@ejemplo.com", "user",  True),
]

for fila in filas:
    registro = dict(zip(columnas, fila))   # convertir a dict
    print(f"  {registro['nombre']:<20} [{registro['rol']}]")
```

### `namedtuple` y `NamedTuple`

```python
from typing import NamedTuple
from collections import namedtuple

# Forma moderna — NamedTuple con anotaciones (Python 3.6+)
class ConexionBD(NamedTuple):
    host:     str
    puerto:   int
    base:     str
    ssl:      bool  = True
    timeout:  int   = 30

conn = ConexionBD("10.0.1.1", 5432, "produccion")
print(conn.host)     # 10.0.1.1
print(conn.puerto)   # 5432
print(conn[0])       # 10.0.1.1 — también accesible por índice
print(conn._asdict())  # {'host': '10.0.1.1', 'puerto': 5432, ...}

# NamedTuple es inmutable — conn.host = "otro" lanzaría AttributeError
# Para crear una copia modificada:
conn_replica = conn._replace(host="10.0.1.2", base="replica")
print(conn_replica)

# Forma clásica — namedtuple() como función
PuntoRed = namedtuple("PuntoRed", ["x", "y", "etiqueta"])
nodo = PuntoRed(x=100, y=200, etiqueta="router-01")
print(nodo)   # PuntoRed(x=100, y=200, etiqueta='router-01')
```

---

## `dict` — diccionario clave → valor

El dict es la estructura de datos central de Python. En Python 3.7+
**mantiene el orden de inserción** (garantizado por la especificación).

```python
# Creación
config: dict[str, str | int | bool] = {
    "host":       "localhost",
    "puerto":     5432,
    "base_datos": "produccion",
    "ssl":        True,
    "timeout":    30,
}

# dict() desde pares clave-valor
config2 = dict(host="localhost", puerto=5432)

# dict desde lista de tuplas
pares = [("a", 1), ("b", 2), ("c", 3)]
desde_pares = dict(pares)
```

### Métodos completos de dict

```python
inventario: dict[str, int] = {"teclado": 15, "monitor": 8, "raton": 22}

# get — acceso seguro con valor por defecto
print(inventario.get("teclado"))          # 15
print(inventario.get("auriculares"))      # None
print(inventario.get("auriculares", 0))   # 0

# setdefault — si la clave no existe, la crea con el valor dado
inventario.setdefault("auriculares", 0)   # inventario["auriculares"] = 0
inventario.setdefault("teclado", 999)     # no modifica — ya existe

# update — mezclar otro dict (o iterable de pares)
actualizaciones = {"teclado": 20, "camara": 5}
inventario.update(actualizaciones)

# Operador | (Python 3.9+) — merge sin modificar el original
nuevo_inventario = inventario | {"cable_hdmi": 30}
combinado = {"base": 1} | inventario   # el segundo tiene prioridad en conflictos

# pop — eliminar y devolver
cantidad_monitores = inventario.pop("monitor", 0)    # 0 si no existe

# popitem — eliminar y devolver el último elemento insertado
ultimo_clave, ultimo_valor = inventario.popitem()

# Iterar sobre el dict
for clave in inventario:               # itera sobre claves
    print(clave)

for clave, valor in inventario.items():    # claves Y valores
    print(f"  {clave}: {valor}")

for valor in inventario.values():      # solo valores
    print(valor)

for clave in inventario.keys():        # solo claves (como un set)
    print(clave)

# "in" comprueba claves — O(1) gracias a la tabla hash
print("teclado" in inventario)         # True
print(100 in inventario.values())      # True/False — O(n)

# Dict como acumulador de frecuencias
log_niveles = ["INFO", "ERROR", "WARN", "ERROR", "INFO", "INFO", "ERROR"]
conteo: dict[str, int] = {}
for nivel in log_niveles:
    conteo[nivel] = conteo.get(nivel, 0) + 1
print(conteo)   # {'INFO': 3, 'ERROR': 3, 'WARN': 1}

# Forma más idiomática con collections.Counter
from collections import Counter, defaultdict
conteo2 = Counter(log_niveles)
print(conteo2.most_common(2))   # [('INFO', 3), ('ERROR', 3)]

# defaultdict — evita KeyError al acceder a claves nuevas
errores_por_servicio: defaultdict[str, list[str]] = defaultdict(list)
errores_por_servicio["api-gateway"].append("timeout")
errores_por_servicio["api-gateway"].append("503")
errores_por_servicio["db-service"].append("connection refused")
```

---

## `set` — conjunto sin duplicados

Los sets son colecciones no ordenadas de elementos únicos.
Las operaciones de conjuntos (unión, intersección, diferencia) son O(min(m,n)).

```python
# Creación
permisos_usuario:  set[str] = {"leer", "escribir", "crear"}
permisos_admin:    set[str] = {"leer", "escribir", "crear", "eliminar", "admin"}
permisos_lectura:  set[str] = {"leer"}

# Añadir y eliminar
permisos_usuario.add("exportar")
permisos_usuario.discard("exportar")    # no lanza error si no existe
# permisos_usuario.remove("exportar")  # lanzaría KeyError si no existe

# Operaciones de conjuntos
union         = permisos_usuario | permisos_admin     # todos los permisos
interseccion  = permisos_usuario & permisos_admin     # en ambos
diferencia    = permisos_admin    - permisos_usuario  # solo en admin
simetrica     = permisos_usuario ^ permisos_admin     # en uno pero no en ambos

print(f"Solo admin tiene: {diferencia}")    # {'eliminar', 'admin'}

# Comprobaciones de subconjunto/superconjunto
print(permisos_lectura.issubset(permisos_usuario))   # True
print(permisos_admin.issuperset(permisos_usuario))   # True
print(permisos_usuario.isdisjoint({"banear"}))       # True — ningún elemento común

# Set para eliminar duplicados — conversión rápida
ips_con_duplicados = ["10.0.0.1", "10.0.0.2", "10.0.0.1", "10.0.0.3", "10.0.0.2"]
ips_unicas = list(set(ips_con_duplicados))   # orden no garantizado

# Para preservar el orden al deduplicar (Python 3.7+)
visto = set()
ips_dedup_ordenadas = []
for ip in ips_con_duplicados:
    if ip not in visto:
        visto.add(ip)
        ips_dedup_ordenadas.append(ip)
# ['10.0.0.1', '10.0.0.2', '10.0.0.3']

# frozenset — set inmutable, puede usarse como clave de dict
roles_necesarios = frozenset({"admin", "superuser"})
cache_permisos: dict[frozenset[str], list[str]] = {
    frozenset({"leer"}):             ["/api/get/*"],
    frozenset({"leer", "escribir"}): ["/api/get/*", "/api/post/*"],
}
```

---

## Comprensiones

### List comprehension

```python
# Sintaxis: [expresion for elemento in iterable if condicion]

# Básica
cuadrados = [x ** 2 for x in range(1, 11)]
# [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# Con filtro
puertos_altos = [p for p in range(1024, 65536) if p % 1000 == 0]
# [1000, 2000, ..., 65000]

# Transformación de strings
rutas_api = [
    "/api/v1/users", "/api/v1/posts", "/health", "/api/v2/users", "/metrics"
]
endpoints_v1 = [
    r.removeprefix("/api/v1")
    for r in rutas_api
    if r.startswith("/api/v1")
]
# ['/users', '/posts']

# Anidada — producto cartesiano
hosts   = ["web-01", "web-02"]
regiones = ["eu-west", "us-east"]
combinaciones = [f"{h}.{r}" for h in hosts for r in regiones]
# ['web-01.eu-west', 'web-01.us-east', 'web-02.eu-west', 'web-02.us-east']

# Con walrus — capturar el resultado de una expresión costosa
import re
patron = re.compile(r"ERROR: (.+)")
lineas = [
    "INFO: sistema iniciado",
    "ERROR: timeout en redis",
    "ERROR: disco al 95%",
    "DEBUG: cache miss",
]
mensajes_error = [
    m.group(1)
    for linea in lineas
    if (m := patron.search(linea))
]
# ['timeout en redis', 'disco al 95%']
```

### Dict comprehension

```python
# Sintaxis: {clave: valor for elemento in iterable if condicion}

# Invertir un dict (valores → claves)
puertos_servicio = {"HTTP": 80, "HTTPS": 443, "SSH": 22, "FTP": 21}
servicio_por_puerto = {v: k for k, v in puertos_servicio.items()}
print(servicio_por_puerto[443])   # HTTPS

# Filtrar y transformar un dict
activos_altos = {
    host: carga
    for host, carga in {
        "web-01": 0.82, "web-02": 0.31, "db-01": 0.95, "cache-01": 0.12
    }.items()
    if carga > 0.5
}
# {'web-01': 0.82, 'db-01': 0.95}

# Agrupar — dict de listas
eventos = [
    ("auth",    "login",   "ana"),
    ("api",     "request", "bob"),
    ("auth",    "logout",  "ana"),
    ("api",     "request", "carlos"),
    ("api",     "error",   "bob"),
]
por_servicio: dict[str, list[tuple]] = {}
for servicio, accion, usuario in eventos:
    por_servicio.setdefault(servicio, []).append((accion, usuario))
# {'auth': [('login', 'ana'), ('logout', 'ana')], 'api': [...]}

# Mismo resultado con defaultdict + comprensión
from collections import defaultdict
por_servicio2: defaultdict[str, list] = defaultdict(list)
for servicio, accion, usuario in eventos:
    por_servicio2[servicio].append((accion, usuario))
```

### Set comprehension

```python
# Sintaxis: {expresion for elemento in iterable if condicion}

log_lines = [
    "web-01 GET /api/users 200",
    "web-02 POST /api/users 201",
    "web-01 GET /api/posts 200",
    "web-03 GET /api/users 404",
    "web-02 GET /api/posts 200",
]

# Set de servidores únicos que aparecen en los logs
servidores_activos = {linea.split()[0] for linea in log_lines}
print(servidores_activos)   # {'web-01', 'web-02', 'web-03'}

# Endpoints con errores 4xx
endpoints_con_error = {
    linea.split()[2]
    for linea in log_lines
    if int(linea.split()[-1]) >= 400
}
print(endpoints_con_error)   # {'/api/users'}
```

### Generator expression

Las generator expressions son como list comprehensions pero perezosas —
no crean la lista completa en memoria, evalúan bajo demanda:

```python
# List comprehension — crea toda la lista inmediatamente
suma_cuadrados_lista = sum([x ** 2 for x in range(1_000_000)])

# Generator expression — evalúa x**2 de uno en uno, sin lista intermedia
suma_cuadrados_gen = sum(x ** 2 for x in range(1_000_000))   # sin corchetes

# Equivalentes en resultado, pero el generador usa mucho menos memoria
# Siempre preferir generador cuando solo se necesita iterar una vez

# Contar elementos sin crear lista completa
import os
total_archivos_log = sum(
    1
    for nombre in os.listdir("/var/log")
    if nombre.endswith(".log")
) if os.path.exists("/var/log") else 0

# any/all con generator expression — cortocircuito
valores = [0, 0, 0, 0, 1, 0]
tiene_activo = any(v > 0 for v in valores)      # True — para cuando encuentra el 1
todos_activos = all(v > 0 for v in valores)     # False — para en el primer 0
```

---

## `map()`, `filter()`, `reduce()`

En Python moderno se prefieren comprensiones por su legibilidad,
pero `map`/`filter` siguen siendo útiles en ciertos contextos:

```python
from functools import reduce

temperaturas_c = [0, 20, 37.5, 100, -10]

# map — aplica función a cada elemento, devuelve iterador
temps_f_map = list(map(lambda c: round(c * 9/5 + 32, 1), temperaturas_c))
temps_f_comp = [round(c * 9/5 + 32, 1) for c in temperaturas_c]   # equivalente

# filter — filtra elementos, devuelve iterador
positivas_filter = list(filter(lambda t: t > 0, temperaturas_c))
positivas_comp   = [t for t in temperaturas_c if t > 0]   # equivalente

# reduce — acumula un único valor (requiere importar de functools)
producto = reduce(lambda acc, x: acc * x, [1, 2, 3, 4, 5])
print(producto)   # 120 — equivale a 1*2*3*4*5

# Cuándo map() sí es preferible — con funciones ya definidas (sin lambda)
nombres = ["ana", "bob", "carlos"]
en_mayusculas = list(map(str.upper, nombres))       # más limpio que la comprensión
longitudes    = list(map(len, nombres))             # [3, 3, 6]
enteros       = list(map(int, ["1", "2", "3"]))    # conversión masiva
```

---

## `sorted()`, `max()`, `min()` con `key`

```python
# sorted() devuelve nueva lista; list.sort() modifica en su lugar
registros = [
    {"nombre": "redis-cache",  "ram_gb": 8,  "cpu": 0.12},
    {"nombre": "postgres-main","ram_gb": 32, "cpu": 0.78},
    {"nombre": "api-gateway",  "ram_gb": 4,  "cpu": 0.55},
    {"nombre": "worker-pool",  "ram_gb": 16, "cpu": 0.91},
]

# Por una sola clave
por_ram = sorted(registros, key=lambda r: r["ram_gb"])
por_cpu_desc = sorted(registros, key=lambda r: r["cpu"], reverse=True)

# Por múltiples criterios — tupla de claves
por_cpu_asc_ram_desc = sorted(
    registros,
    key=lambda r: (r["cpu"], -r["ram_gb"])   # cpu asc, ram desc
)

# max/min con clave
mas_ram     = max(registros, key=lambda r: r["ram_gb"])
menos_carga = min(registros, key=lambda r: r["cpu"])
print(f"Más RAM: {mas_ram['nombre']} ({mas_ram['ram_gb']}GB)")
print(f"Menos carga: {menos_carga['nombre']} ({menos_carga['cpu']:.0%})")

# Ordenar strings con locale — para ordenamiento natural en español
import locale
locale.setlocale(locale.LC_COLLATE, "es_ES.UTF-8") if False else None  # opcional
nombres_esp = ["ángel", "Carlos", "álvaro", "Beatriz", "ana"]
nombres_ordenados = sorted(nombres_esp, key=str.lower)
```

---

## `itertools` — herramientas de iteración

```python
import itertools

# chain — concatenar iterables sin crear lista
servidores_eu = ["web-eu-01", "web-eu-02"]
servidores_us = ["web-us-01", "web-us-02"]
todos = list(itertools.chain(servidores_eu, servidores_us))
# ['web-eu-01', 'web-eu-02', 'web-us-01', 'web-us-02']

# chain.from_iterable — aplanar lista de listas
regiones = [["web-eu-01", "web-eu-02"], ["web-us-01"], ["web-as-01", "web-as-02"]]
aplanado = list(itertools.chain.from_iterable(regiones))

# product — producto cartesiano (bucles anidados como una sola línea)
ambientes  = ["dev", "staging", "prod"]
servicios  = ["api", "worker", "cron"]
deploys = list(itertools.product(ambientes, servicios))
# [('dev', 'api'), ('dev', 'worker'), ..., ('prod', 'cron')]

# groupby — agrupar elementos CONSECUTIVOS por clave
# IMPORTANTE: los datos deben estar ordenados por la clave primero
eventos = [
    {"servicio": "api",    "nivel": "INFO"},
    {"servicio": "api",    "nivel": "ERROR"},
    {"servicio": "auth",   "nivel": "INFO"},
    {"servicio": "auth",   "nivel": "WARN"},
    {"servicio": "worker", "nivel": "INFO"},
]
eventos_ord = sorted(eventos, key=lambda e: e["servicio"])
for servicio, grupo in itertools.groupby(eventos_ord, key=lambda e: e["servicio"]):
    items = list(grupo)
    print(f"  {servicio}: {len(items)} eventos")

# islice — tomar N elementos de un iterador
primeras_10_ips = list(itertools.islice(
    (f"10.0.0.{i}" for i in range(1, 255)),
    10
))

# combinations — combinaciones sin repetición
nodos = ["A", "B", "C", "D"]
enlaces = list(itertools.combinations(nodos, 2))
# [('A','B'), ('A','C'), ('A','D'), ('B','C'), ('B','D'), ('C','D')]

# permutations — permutaciones
rutas_2_nodos = list(itertools.permutations(["A", "B", "C"], 2))
# [('A','B'), ('A','C'), ('B','A'), ('B','C'), ('C','A'), ('C','B')]

# accumulate — acumulados (suma acumulada, máximo acumulado, etc.)
import operator
bytes_recibidos = [1024, 2048, 512, 4096, 256]
totales_acumulados = list(itertools.accumulate(bytes_recibidos))
# [1024, 3072, 3584, 7680, 7936]

maximos_acumulados = list(itertools.accumulate(bytes_recibidos, func=max))
# [1024, 2048, 2048, 4096, 4096]
```

---

## Ejemplo aplicado — analizador de logs de servidor

```python
"""
analizador_logs.py
Parsea, agrupa, filtra y produce estadísticas de logs de acceso HTTP.
Demuestra: list, dict, set, comprensiones, Counter, itertools, sorted.
"""
import re
import itertools
from collections import Counter, defaultdict
from dataclasses import dataclass
from datetime import datetime

# ── Modelo ───────────────────────────────────────────────────────────────────

@dataclass(frozen=True)
class EntradaAcceso:
    ip:           str
    timestamp:    datetime
    metodo:       str
    ruta:         str
    codigo:       int
    bytes_resp:   int
    user_agent:   str


# ── Parseo ───────────────────────────────────────────────────────────────────

# Formato de log de nginx: Combined Log Format
PATRON_NGINX = re.compile(
    r'(?P<ip>\d+\.\d+\.\d+\.\d+) - - '
    r'\[(?P<timestamp>[^\]]+)\] '
    r'"(?P<metodo>\w+) (?P<ruta>[^\s]+) HTTP/\d\.\d" '
    r'(?P<codigo>\d+) '
    r'(?P<bytes>\d+) '
    r'"[^"]*" '
    r'"(?P<ua>[^"]*)"'
)

FORMATO_TIMESTAMP = "%d/%b/%Y:%H:%M:%S %z"


def parsear_log_nginx(lineas: list[str]) -> list[EntradaAcceso]:
    """Parsea líneas de log nginx en objetos estructurados."""
    entradas = []
    for linea in lineas:
        m = PATRON_NGINX.match(linea.strip())
        if not m:
            continue
        try:
            entradas.append(EntradaAcceso(
                ip         = m["ip"],
                timestamp  = datetime.strptime(m["timestamp"], FORMATO_TIMESTAMP),
                metodo     = m["metodo"],
                ruta       = m["ruta"],
                codigo     = int(m["codigo"]),
                bytes_resp = int(m["bytes"]),
                user_agent = m["ua"],
            ))
        except (ValueError, KeyError):
            continue
    return entradas


# ── Análisis ─────────────────────────────────────────────────────────────────

def analizar_logs(entradas: list[EntradaAcceso]) -> dict:
    """Produce un análisis completo de las entradas de log."""

    if not entradas:
        return {}

    # ── Conteos básicos con Counter ──
    codigos      = Counter(e.codigo for e in entradas)
    metodos      = Counter(e.metodo for e in entradas)
    top_rutas    = Counter(e.ruta   for e in entradas).most_common(5)
    ips_unicas   = {e.ip for e in entradas}    # set comprehension

    # ── IPs con más solicitudes (posibles bots/ataques) ──
    top_ips      = Counter(e.ip for e in entradas).most_common(5)

    # ── Errores 4xx y 5xx ──
    errores      = [e for e in entradas if e.codigo >= 400]
    errores_4xx  = [e for e in errores  if e.codigo < 500]
    errores_5xx  = [e for e in errores  if e.codigo >= 500]

    # ── Rutas con más errores ──
    rutas_error  = Counter(e.ruta for e in errores).most_common(5)

    # ── Bytes transferidos ──
    total_bytes  = sum(e.bytes_resp for e in entradas)
    max_respuesta = max(entradas, key=lambda e: e.bytes_resp)

    # ── Distribución por hora ──
    por_hora: dict[int, int] = defaultdict(int)
    for entrada in entradas:
        por_hora[entrada.timestamp.hour] += 1

    hora_pico = max(por_hora, key=lambda h: por_hora[h])

    # ── Tasa de error ──
    tasa_error = len(errores) / len(entradas) * 100

    # ── IPs que solo causan errores (posibles escáneres) ──
    ips_con_exito = {e.ip for e in entradas if e.codigo < 400}
    ips_solo_error = ips_unicas - ips_con_exito

    # ── Agrupar errores 5xx por ruta con itertools.groupby ──
    errores_5xx_ord = sorted(errores_5xx, key=lambda e: e.ruta)
    errores_5xx_por_ruta = {
        ruta: len(list(grupo))
        for ruta, grupo in itertools.groupby(errores_5xx_ord, key=lambda e: e.ruta)
    }

    return {
        "total":            len(entradas),
        "ips_unicas":       len(ips_unicas),
        "codigos":          dict(codigos),
        "metodos":          dict(metodos),
        "top_rutas":        top_rutas,
        "top_ips":          top_ips,
        "errores_4xx":      len(errores_4xx),
        "errores_5xx":      len(errores_5xx),
        "rutas_con_error":  rutas_error,
        "tasa_error_pct":   round(tasa_error, 2),
        "total_mb":         round(total_bytes / 1_048_576, 2),
        "max_respuesta_kb": round(max_respuesta.bytes_resp / 1024, 1),
        "hora_pico":        hora_pico,
        "dist_por_hora":    dict(sorted(por_hora.items())),
        "ips_solo_error":   ips_solo_error,
        "errores_5xx_ruta": errores_5xx_por_ruta,
    }


def imprimir_resumen(stats: dict) -> None:
    """Formatea e imprime las estadísticas de análisis."""
    print("╔══════════════════════════════════════════════════╗")
    print("║           ANÁLISIS DE LOGS DE ACCESO            ║")
    print("╠══════════════════════════════════════════════════╣")
    print(f"║  Solicitudes totales : {stats['total']:<27}║")
    print(f"║  IPs únicas          : {stats['ips_unicas']:<27}║")
    print(f"║  Tasa de error       : {stats['tasa_error_pct']:.2f}%{' ' * 23}║")
    print(f"║  Total transferido   : {stats['total_mb']:.2f} MB{' ' * 20}║")
    print(f"║  Hora pico           : {stats['hora_pico']:02d}:00h{' ' * 21}║")
    print("╠══════════════════════════════════════════════════╣")
    print("║  Códigos HTTP:                                   ║")
    for codigo, count in sorted(stats["codigos"].items()):
        barra = "█" * min(20, count // max(1, stats["total"] // 20))
        print(f"║    {codigo}  {count:>5}  {barra:<20}          ║")
    print("╠══════════════════════════════════════════════════╣")
    print("║  Top 5 rutas:                                    ║")
    for ruta, count in stats["top_rutas"]:
        print(f"║    {count:>5}×  {ruta:<40}║")
    if stats["ips_solo_error"]:
        print("╠══════════════════════════════════════════════════╣")
        print("║  IPs solo con errores (posibles escáneres):      ║")
        for ip in sorted(stats["ips_solo_error"])[:5]:
            print(f"║    {ip:<47}║")
    print("╚══════════════════════════════════════════════════╝")


# ── Datos de prueba ──────────────────────────────────────────────────────────

LOG_NGINX_MUESTRA = [
    '10.0.1.5 - - [15/Mar/2025:14:01:03 +0000] "GET /api/users HTTP/1.1" 200 1452 "-" "Mozilla/5.0"',
    '10.0.1.6 - - [15/Mar/2025:14:01:04 +0000] "POST /api/login HTTP/1.1" 200 256 "-" "curl/7.84"',
    '10.0.2.1 - - [15/Mar/2025:14:01:05 +0000] "GET /api/users HTTP/1.1" 401 89 "-" "python-requests/2.31"',
    '10.0.2.1 - - [15/Mar/2025:14:01:06 +0000] "GET /api/admin HTTP/1.1" 403 120 "-" "python-requests/2.31"',
    '10.0.1.5 - - [15/Mar/2025:14:01:07 +0000] "GET /api/posts HTTP/1.1" 200 8920 "-" "Mozilla/5.0"',
    '10.0.1.7 - - [15/Mar/2025:14:01:08 +0000] "GET /api/users HTTP/1.1" 500 0 "-" "Go-http-client/1.1"',
    '10.0.1.8 - - [15/Mar/2025:14:01:09 +0000] "DELETE /api/posts/42 HTTP/1.1" 404 110 "-" "axios/1.6"',
    '10.0.1.5 - - [15/Mar/2025:14:01:10 +0000] "GET /health HTTP/1.1" 200 18 "-" "kube-probe/1.28"',
    '10.0.2.1 - - [15/Mar/2025:14:01:11 +0000] "GET /api/users/1 HTTP/1.1" 403 89 "-" "python-requests/2.31"',
    '10.0.1.9 - - [15/Mar/2025:14:01:12 +0000] "POST /api/users HTTP/1.1" 503 0 "-" "Mozilla/5.0"',
]

entradas = parsear_log_nginx(LOG_NGINX_MUESTRA)
estadisticas = analizar_logs(entradas)
imprimir_resumen(estadisticas)
```

---

## Ejercicios propuestos

1. **Índice invertido de texto** — Dado un texto largo (usa Lorem Ipsum o una
   noticia), construye un índice invertido: un `dict[str, set[int]]` donde
   cada palabra mapea al conjunto de números de línea donde aparece. Normaliza
   palabras a minúsculas y elimina puntuación. Usa dict comprehension y set
   comprehension. Permite buscar qué líneas contienen una palabra dada.

2. **Procesador de inventario con Counter y defaultdict** — Lee una lista de
   movimientos de almacén: `[("entrada", "teclado", 50), ("salida", "teclado", 8), ...]`.
   Usa `defaultdict(int)` para mantener el stock actual. Usa `Counter` para
   contar movimientos por tipo y por producto. Detecta los 3 productos con
   más rotación y los que tienen stock <= 5 con set comprehension.

3. **Matriz de distancias con itertools** — Dada una lista de ciudades con
   coordenadas `(nombre, lat, lon)`, usa `itertools.combinations(ciudades, 2)`
   para calcular la distancia haversine entre cada par. Guarda los resultados
   en un `dict[frozenset[str], float]`. Encuentra la ruta más corta que visite
   todas las ciudades probando `itertools.permutations` (para N <= 8).

4. **Pipeline de transformación de CSV** — Dado un CSV de métricas de
   servidores (simulado como string multilinea), usa comprensiones para:
   parsear cada fila en un dict, filtrar servidores con CPU > 80%,
   calcular percentiles 50/90/99 de latencia sin numpy (solo con `sorted` y
   slicing), agrupar por región con `defaultdict`, y exportar los resultados
   como lista de dicts ordenada por carga descendente.

---

## Resumen de la página 4

- `list` es la colección mutable ordenada central de Python. `append` es O(1), `insert`/`remove` son O(n). El slicing `[inicio:fin:paso]` genera nuevas listas sin modificar la original.
- `tuple` es inmutable y más eficiente que `list`. `NamedTuple` añade nombres a los campos manteniendo la inmutabilidad y compatibilidad con indexación numérica.
- `dict` mantiene el orden de inserción desde Python 3.7. `get(clave, defecto)` evita KeyError. `setdefault` inicializa claves nuevas. El operador `|` (Python 3.9+) fusiona dicts sin modificar los originales.
- `set` garantiza unicidad y permite operaciones matemáticas de conjuntos (`|`, `&`, `-`, `^`) en O(min(m,n)). `frozenset` es la versión inmutable, usable como clave de dict.
- Las comprensiones `[e for x in it if cond]` son más idiomáticas y generalmente más rápidas que `map`/`filter`. Las generator expressions `(e for x in it)` son lazy y ahorran memoria cuando solo se itera una vez.
- `Counter` de `collections` es un dict especializado para frecuencias. `defaultdict` elimina la necesidad de comprobar si una clave existe antes de usarla.
- `itertools.chain`, `product`, `groupby`, `combinations` y `accumulate` permiten operaciones sobre iterables sin cargar todo en memoria — esenciales para procesamiento de datos a escala.
- `sorted(clave=lambda...)` devuelve nueva lista ordenada; `list.sort(key=...)` ordena en su lugar. Ambos aceptan tuplas como clave para ordenamiento multi-criterio.

---

> **Siguiente página →** Página 8: Programación Orientada a Objetos — clases, herencia y polimorfismo.
> clases, herencia, MRO, polimorfismo, ABC e interfaces, y métodos dunder.
