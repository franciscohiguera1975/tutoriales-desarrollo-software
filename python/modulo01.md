# Tutorial Python — Página 1
## Módulo 1 · Entorno y Fundamentos
### Instalación, tipos de datos, variables y el sistema de tipos de Python

---

## ¿Qué es Python?

Python es un lenguaje de propósito general, interpretado, con tipado dinámico
y gestión automática de memoria. Creado por Guido van Rossum en 1991, hoy es
el lenguaje más usado del mundo según el índice TIOBE y el más demandado en
ciencia de datos, automatización, backend web e inteligencia artificial.

```
JavaScript / TypeScript  →  navegador + Node.js (frontend/fullstack)
Kotlin / Java            →  JVM + Android (aplicaciones empresariales)
Dart / Flutter           →  aplicaciones multiplataforma con un solo código
Python                   →  scripting, datos, ML/AI, backend, automatización
```

### ¿Por qué Python en 2025?

- El ecosistema de ML/AI (PyTorch, TensorFlow, scikit-learn) no tiene rival
- Velocidad de prototipado: menos boilerplate que Java/Kotlin, más expresivo que C++
- CPython 3.12+ tiene mejoras de rendimiento del 25% respecto a 3.10
- El GIL (Global Interpreter Lock) está siendo eliminado progresivamente en 3.13+
- Integración nativa con C/C++ vía `ctypes`, `cffi` y extensiones Cython

---

## Versiones del curso

| Herramienta | Versión |
|---|---|
| Python | **3.12+** (octubre 2023, soporte hasta 2028) |
| pip | **24.x** (gestor de paquetes oficial) |
| venv | incluido en la stdlib (entornos virtuales) |
| VS Code | última estable + extensión Python de Microsoft |
| Ruff | 0.4+ (linter ultrarrápido, sustituye a flake8 + isort) |
| mypy | 1.10+ (verificación estática de tipos) |

---

## Parte 1 — Instalación del entorno

### Instalar Python 3.12+

```bash
# macOS con Homebrew
brew install python@3.12

# Ubuntu / Debian
sudo apt update && sudo apt install python3.12 python3.12-venv python3-pip

# Windows — descargar instalador desde python.org
# Marcar "Add Python to PATH" durante la instalación

# Verificar instalación
python3 --version     # Python 3.12.x
python3 -m pip --version
```

### Crear y activar un entorno virtual

Los entornos virtuales aíslan las dependencias de cada proyecto,
evitando conflictos entre versiones de paquetes — equivalen
conceptualmente al `build.gradle.kts` local de un módulo Android.

```bash
# Crear entorno virtual en la carpeta .venv
python3 -m venv .venv

# Activar el entorno
# Linux / macOS
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# El prompt cambia para indicar que el entorno está activo:
# (.venv) usuario@equipo:~/proyecto$

# Instalar paquetes dentro del entorno
pip install requests fastapi ruff mypy

# Guardar dependencias del proyecto
pip freeze > requirements.txt

# Restaurar en otro equipo
pip install -r requirements.txt

# Desactivar el entorno
deactivate
```

### Extensiones recomendadas en VS Code

```
Python          (id: ms-python.python)              ← obligatoria
Pylance         (id: ms-python.vscode-pylance)      ← análisis de tipos
Ruff            (id: charliermarsh.ruff)             ← linter + formatter
mypy            (id: ms-python.mypy-type-checker)   ← verificación de tipos
Error Lens      (id: usernamehw.errorlens)           ← errores en línea
```

### Configuración de `pyproject.toml`

```toml
# pyproject.toml — archivo de configuración moderno del proyecto
# Equivale a build.gradle.kts en el ecosistema Android/Kotlin

[project]
name            = "mi-proyecto"
version         = "0.1.0"
description     = "Descripción del proyecto"
requires-python = ">=3.12"
dependencies    = [
    "requests>=2.31",
    "fastapi>=0.111",
]

[tool.ruff]
line-length    = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]   # errores, pyflakes, isort, pyupgrade

[tool.mypy]
python_version       = "3.12"
strict               = true
ignore_missing_stubs = true
```

---

## Parte 2 — El REPL de Python

El REPL (Read-Eval-Print Loop) permite experimentar con código interactivamente
sin crear archivos. Esencial para explorar la API de una librería.

```bash
# Iniciar el REPL
python3

# Python 3.12.3 (main, Apr  9 2024, 08:09:14)
# >>> _
```

```python
# Dentro del REPL — se evalúa cada expresión al pulsar Enter
>>> 2 ** 32
4294967296

>>> "python".upper()
'PYTHON'

>>> [x ** 2 for x in range(6)]
[0, 1, 4, 9, 16, 25]

>>> import sys; sys.version
'3.12.3 (main, Apr  9 2024, 08:09:14) [GCC 13.2.0]'

# Salir del REPL
>>> exit()
```

---

## Parte 3 — Variables y tipos básicos

Python tiene tipado dinámico: el tipo se asocia al valor, no a la variable.
A diferencia de Kotlin donde `val nombre: String = "Ana"` fija el tipo en la
declaración, en Python `nombre = "Ana"` puede reasignarse a cualquier tipo
(aunque hacerlo es mala práctica en código de producción).

```python
# Variables — asignación directa, sin palabra clave declarativa
nombre   = "Ana García"       # str
edad     = 28                 # int
precio   = 89.99              # float
activo   = True               # bool
sin_dato = None               # NoneType — equivalente a null en Kotlin/Dart

# Múltiple asignación en una sola línea
x = y = z = 0

# Desestructuración (tuple unpacking)
latitud, longitud = 40.4168, -3.7038

# Intercambio sin variable temporal — Python idiomático
a, b = 10, 20
a, b = b, a
print(a, b)   # 20 10

# Verificar el tipo de un valor en tiempo de ejecución
print(type(nombre))    # <class 'str'>
print(type(edad))      # <class 'int'>
print(type(precio))    # <class 'float'>
print(type(activo))    # <class 'bool'>
print(type(sin_dato))  # <class 'NoneType'>

# isinstance() — comprobación de tipo segura (funciona con herencia)
print(isinstance(edad, int))          # True
print(isinstance(activo, int))        # True — bool es subclase de int
print(isinstance(precio, (int, float)))  # True — comprobación múltiple
```

> **Prueba esto:** Crea una variable `coordenadas` como tupla `(40.4168, -3.7038)` y desestructúrala en `lat, lon`. ¿Qué devuelve `type(coordenadas)`? ¿Puedes intercambiar `lat` y `lon` en una sola línea?

### Tipos numéricos en detalle

```python
# int — precisión arbitraria (no hay desbordamiento en Python)
entero_grande   = 10 ** 100           # 1 googol — sin overflow
entero_negativo = -42
binario         = 0b11001010          # 202 en base 2
hexadecimal     = 0xFF                # 255 en base 16
octal           = 0o17               # 15 en base 8

# Separador visual de miles — Python 3.6+ (mejora la legibilidad)
millon          = 1_000_000
dos_mil_millones = 2_000_000_000
ip_binaria      = 0b1100_0000_1010_1000_0000_0001_0000_0001

# float — punto flotante IEEE 754 de 64 bits (mismo que double en Kotlin)
pi           = 3.141592653589793
notacion_cient = 1.5e-10             # 0.00000000015
negativo     = -0.001

# Operaciones aritméticas — la división siempre retorna float
print(7 // 2)    # 3      — división entera (floor division)
print(7 %  2)    # 1      — módulo (resto de la división)
print(7 /  2)    # 3.5    — división real, siempre float
print(2 ** 10)   # 1024   — potencia (no existe ^ para potencia)
print(-7 // 2)   # -4     — floor division redondea hacia -∞

# Funciones matemáticas integradas
print(abs(-42))          # 42     — valor absoluto
print(round(3.7))        # 4      — redondeo
print(round(3.14159, 3)) # 3.142  — redondeo a 3 decimales
print(divmod(17, 5))     # (3, 2) — cociente y resto a la vez

# complex — números complejos (útiles en procesamiento de señales)
z = 3 + 4j
print(z.real)    # 3.0
print(z.imag)    # 4.0
print(abs(z))    # 5.0 — módulo del número complejo (teorema de Pitágoras)
```

> **Prueba esto:** ¿Cuánto es `10 ** 100`? ¿Y `-7 // 2`? Verifica que `0b11001010 == 202`. Usa el separador de miles para escribir la velocidad de la luz en m/s: `299_792_458`.

### Strings

```python
# Strings — inmutables, Unicode por defecto en Python 3
saludo    = "Hola, mundo"
saludo2   = 'Hola, mundo'    # comillas simples y dobles son equivalentes
multilinea = """
Este es un string de varias líneas.
Útil para docstrings y bloques de texto.
Preserva los saltos de línea y la indentación.
"""

# Escape sequences
ruta_windows = "C:\\Users\\Ana\\Documentos"
tabulacion   = "columna1\tcolumna2\tcolumna3"
nueva_linea  = "primera linea\nsegunda linea"

# Raw strings — los backslashes no se interpretan como escape
ruta_raw   = r"C:\Users\Ana\Documentos"    # útil en rutas Windows
patron_re  = r"\d{3}-\d{2}-\d{4}"         # imprescindible en regex

# Indexación (base 0, índices negativos desde el final)
texto = "Python 3.12"
print(texto[0])       # P
print(texto[-1])      # 2       — último carácter
print(texto[-4:])     # 3.12    — últimos 4 caracteres

# Slicing [inicio:fin:paso]
print(texto[7:])      # 3.12    — desde índice 7 hasta el final
print(texto[:6])      # Python  — desde el inicio hasta índice 5
print(texto[::2])     # Pto .2  — cada 2 caracteres
print(texto[::-1])    # 21.3 nohtyP — invertido (paso -1)
print(texto[2:8:2])   # to 3    — desde 2 hasta 7, de 2 en 2

# Métodos de string más útiles en sistemas reales
url = "  https://api.ejemplo.com/v1/usuarios  "
print(url.strip())               # elimina espacios al inicio y final
print(url.strip().lower())       # normalizar URL completa

cabecera = "Content-Type: application/json; charset=utf-8"
tipo, _, resto = cabecera.partition(": ")   # divide en el primer ": "
print(tipo)    # Content-Type
print(resto)   # application/json; charset=utf-8

print("GET /api/v1 HTTP/1.1".split())         # ['GET', '/api/v1', 'HTTP/1.1']
print(", ".join(["Ana", "Luis", "María"]))    # Ana, Luis, María
print("nginx error".replace("error", "warn")) # nginx warn
print("Python".center(20, "="))               # =======Python=======
print("token".startswith("tok"))              # True
print("request.log".endswith(".log"))         # True
print("error " * 3)                           # error error error
print("python".capitalize())                  # Python
print("  \n  ".isspace())                     # True — solo espacios/newlines
```

> **Prueba esto:** Dado `cabecera = "Authorization: Bearer eyJhbGc..."`, usa `partition()` para separar el nombre del encabezado del valor. ¿Qué pasa si usas `split(": ")` en su lugar? ¿En qué caso podría dar un resultado incorrecto?

### f-strings — interpolación moderna de strings

Los f-strings (Python 3.6+) son la forma estándar y más eficiente
de construir strings con valores dinámicos:

```python
nombre  = "Ana"
edad    = 28
sueldo  = 45_750.50
activo  = True

# Sintaxis básica — cualquier expresión Python dentro de {}
print(f"Nombre: {nombre}, Edad: {edad}")
# Nombre: Ana, Edad: 28

# Expresiones aritméticas y llamadas a métodos
print(f"El año que viene tendrá {edad + 1} años")
print(f"En mayúsculas: {nombre.upper()}")

# Especificadores de formato — idénticos a format()
print(f"Sueldo: {sueldo:,.2f} €")        # 45,750.50 €
print(f"Sueldo: {sueldo:>15,.2f} €")     # alineado a derecha, 15 chars
print(f"Hex: {255:#010x}")               # 0x000000ff
print(f"Binario: {10:08b}")              # 00001010
print(f"Porcentaje: {0.875:.1%}")        # 87.5%

# Expresión condicional inline
print(f"Estado: {'activo' if activo else 'inactivo'}")

# Formato de fecha sin librería extra
from datetime import datetime
ahora = datetime(2025, 3, 15, 14, 30, 0)
print(f"Fecha: {ahora:%d/%m/%Y %H:%M}")  # 15/03/2025 14:30

# Depuración rápida con = (Python 3.8+)
x = 42
print(f"{x = }")          # x = 42  — imprime nombre y valor
print(f"{len(nombre) = }") # len(nombre) = 3

# Reporte con alineación de columnas
reporte = (
    f"=== Informe de usuario ===\n"
    f"{'Nombre':<10}: {nombre:<20}\n"    # alineado a la izquierda
    f"{'Edad':<10}: {edad:>5} años\n"   # alineado a la derecha
    f"{'Sueldo':<10}: {sueldo:>12,.2f} €"
)
print(reporte)
```

Salida del reporte:
```
=== Informe de usuario ===
Nombre    : Ana
Edad      :    28 años
Sueldo    :   45,750.50 €
```

> **Prueba esto:** Formatea `0.99` como porcentaje con 1 decimal (`"99.0%"`). Luego imprime el número `255` en hexadecimal (`0xff`), binario (`11111111`) y octal con relleno de ceros a 6 dígitos. Usa el modo debug `f"{variable = }"` para inspeccionar el valor de `sueldo`.

---

## Parte 4 — El sistema None

`None` es el único valor del tipo `NoneType`. Es el equivalente a `null`
en Kotlin y Dart. En Python, la convención es comparar con `is None`,
no con `== None`.

```python
# None — ausencia de valor
resultado      = None
configuracion  = None

# La comparación CORRECTA es con 'is' (identidad de objeto)
if resultado is None:
    print("Sin resultado todavía")

if configuracion is not None:
    print("Configuración cargada")

# None como retorno implícito de funciones sin return
def registrar_evento(mensaje: str) -> None:
    print(f"[LOG] {mensaje}")
    # sin 'return' → devuelve None implícitamente

valor = registrar_evento("sistema iniciado")
print(valor is None)   # True

# None en diccionarios — campos opcionales
registro: dict[str, str | None] = {
    "nombre":     "servidor-01",
    "ip":         "192.168.1.10",
    "ultima_ip":  None,       # campo todavía no disponible
    "region":     None,
}

# get() con valor por defecto cuando el campo puede ser None
region    = registro.get("region") or "sin-region"
ultima_ip = registro.get("ultima_ip") or "primera asignación"
print(region)     # sin-region
print(ultima_ip)  # primera asignación

# CUIDADO: 'or' falla si el valor es 0, "" o False (todos son falsy)
puerto = 0
display = puerto or 8080     # Incorrecto: ignora puerto=0
display = puerto if puerto is not None else 8080   # Correcto
```

> **Prueba esto:** Define `timeout = 0` y compara `timeout or 30` vs `timeout if timeout is not None else 30`. ¿Por qué son distintos? Ahora haz lo mismo con `timeout = None`. ¿Coinciden los resultados?

---

## Parte 5 — Tipado dinámico y type hints

Python es de tipado dinámico pero soporta anotaciones de tipo (PEP 484).
Los type hints no se comprueban en tiempo de ejecución por defecto —
son para herramientas como mypy y el autocompletado del IDE.

```python
# Sin type hints — válido pero opaco para el IDE
def calcular_descuento(precio, descuento):
    return precio * (1 - descuento / 100)

# Con type hints — Python 3.12 estilo moderno
def calcular_descuento(precio: float, descuento: float) -> float:
    """Aplica un descuento porcentual al precio base."""
    return precio * (1 - descuento / 100)

# Tipo unión con | (Python 3.10+) — equivale a Union[str, None]
def buscar_usuario(id_usuario: int) -> str | None:
    usuarios = {1: "Ana", 2: "Luis", 3: "Carlos"}
    return usuarios.get(id_usuario)    # None si no existe

# Tipos de colecciones — usar minúsculas (Python 3.9+)
def procesar_ips(direcciones: list[str]) -> dict[str, bool]:
    """Verifica qué IPs están en el rango local."""
    return {ip: ip.startswith("192.168.") for ip in direcciones}

# TypeAlias — nombres para tipos complejos reutilizables (Python 3.12)
type MatrizFlotante = list[list[float]]
type Configuracion  = dict[str, str | int | bool | None]
type ResultadoRed   = tuple[bool, str, int]   # (exito, mensaje, codigo)

# Variables con type hints explícitos
nombre:    str        = "servidor-01"
puerto:    int        = 8080
activo:    bool       = True
etiquetas: list[str]  = ["produccion", "web", "nginx"]
config:    Configuracion = {"timeout": 30, "reintentos": 3, "debug": False}

# isinstance() — verificar tipo en runtime cuando sea necesario
def procesar(valor: int | str | bytes) -> str:
    if isinstance(valor, int):
        return f"Número entero: {valor}"
    elif isinstance(valor, bytes):
        return f"Bytes ({len(valor)}): {valor.hex()}"
    return f"Texto: {valor.upper()}"

print(procesar(42))             # Número entero: 42
print(procesar("hola"))         # Texto: HOLA
print(procesar(b"\xca\xfe"))    # Bytes (2): cafe
```

> **Prueba esto:** Añade un cuarto caso a `procesar()` para manejar `float` y que retorne `f"Decimal: {valor:.4f}"`. ¿Qué pasa si llamas `procesar(True)`? ¿Por qué? (Pista: `bool` es subclase de `int`.)

---

## Parte 6 — Operadores

### Aritméticos y de comparación

```python
a, b = 17, 5

# Aritméticos
print(a + b)    # 22     — suma
print(a - b)    # 12     — resta
print(a * b)    # 85     — multiplicación
print(a / b)    # 3.4    — división real (siempre float)
print(a // b)   # 3      — división entera (floor)
print(a % b)    # 2      — módulo (resto)
print(a ** b)   # 1419857 — potencia

# Asignación compuesta
contador  = 0
contador += 1     # contador = 1
contador *= 10    # contador = 10
contador //= 3    # contador = 3
contador **= 2    # contador = 9

# Comparación — siempre devuelven bool
print(10 > 5)     # True
print(10 == 10)   # True
print(10 != 5)    # True
print(3 <= 3)     # True
print("abc" < "abd")  # True — comparación lexicográfica

# Encadenamiento de comparaciones — Python idiomático
temperatura = 36.8
print(36.0 <= temperatura <= 37.5)   # True — temperatura normal
print(0 <= 200 < 256)                # True — byte válido
print(1 < 2 < 3 < 4)                # True — cadena de comparaciones
```

> **Prueba esto:** ¿Cuánto es `10 ** 3 // 7`? ¿Y `-10 % 3`? Verifica que `0 <= 255 < 256` es `True` pero `0 <= 256 < 256` es `False`. Escribe el equivalente de la segunda expresión sin encadenamiento.

### Operadores lógicos

```python
# and, or, not  (Python) ≡  &&, ||, !  (Kotlin/Dart/JavaScript)
activo   = True
validado = False

print(activo and validado)   # False
print(activo or  validado)   # True
print(not activo)            # False

# Evaluación en cortocircuito — Python solo evalúa lo necesario
def conectar_bd() -> bool:
    print("Intentando conexión...")
    return True

# Si 'activo' es False, Python NO llama a conectar_bd()
resultado = activo and conectar_bd()   # imprime "Intentando conexión..."

# 'or' como valor de respaldo — cuidado con valores falsy (0, "", [], {})
nombre   = ""
mostrar  = nombre or "Anónimo"     # "Anónimo" porque "" es falsy

# Más seguro: comprobar explícitamente None
nombre2  = ""
mostrar2 = nombre2 if nombre2 is not None else "Anónimo"  # ""

# 'and' para ejecutar solo si el objeto existe
archivo  = None
# archivo.close()   # AttributeError
archivo and archivo.close()   # seguro — si archivo es None, no ejecuta
```

> **Prueba esto:** Evalúa `False and print("hola")` y `True or print("hola")` en el REPL. ¿Cuándo se ejecuta el `print`? Esto ilustra el cortocircuito. Luego prueba `[] or "vacío"` y `[1] or "vacío"` — ¿cuál devuelve qué y por qué?

### El operador walrus `:=`

El operador walrus (Python 3.8+) asigna y evalúa en la misma expresión.
Especialmente útil en bucles `while` y comprensiones de lista:

```python
import re

lineas_log = [
    "2025-03-15 14:01:03 ERROR: timeout en conexión a redis (attempt 3/3)",
    "2025-03-15 14:01:05 INFO: solicitud /api/users procesada en 45ms",
    "2025-03-15 14:01:07 ERROR: disco /dev/sda1 al 95% de capacidad",
    "2025-03-15 14:01:09 DEBUG: cache hit ratio 87.3%",
    "2025-03-15 14:01:11 WARN: memoria disponible por debajo del 20%",
]

patron_error = re.compile(r"ERROR: (.+)")

# Sin walrus — requiere variable intermedia por separado
errores_v1 = []
for linea in lineas_log:
    coincidencia = patron_error.search(linea)
    if coincidencia:
        errores_v1.append(coincidencia.group(1))

# Con walrus — asigna 'm' y evalúa su veracidad en la misma expresión
errores_v2 = [
    m.group(1)
    for linea in lineas_log
    if (m := patron_error.search(linea))
]

print(errores_v2)
# ['timeout en conexión a redis (attempt 3/3)', 'disco /dev/sda1 al 95% de capacidad']

# Walrus en while — procesar chunks de un stream hasta vaciarlo
import io
datos_stream = io.BytesIO(b"paquete-de-red-simulado-en-bytes-del-buffer")

numero_chunk = 0
while chunk := datos_stream.read(8):   # sale cuando read() devuelve b""
    numero_chunk += 1
    print(f"Chunk {numero_chunk:02d}: {chunk.hex()} ({len(chunk)} bytes)")
```

> **Prueba esto:** Reescribe el ejemplo de `errores_v2` sin walrus, usando un `for` normal con una variable `m` y un `if`. ¿Cuántas líneas adicionales necesitas? Luego usa el walrus en un `while input("> ") != "salir":` (simulado con una lista de strings) para procesar entradas hasta encontrar `"salir"`.

---

## Ejemplo aplicado — analizador de métricas de sistema

```python
"""
analizador_sistema.py
Parsea un informe de métricas en formato clave=valor y emite diagnóstico.
Demuestra: variables, tipos, f-strings, None, type hints y operadores.
"""
from datetime import datetime

# Alias de tipo para mayor legibilidad
type MetricasSistema = dict[str, float | str | int]


def parsear_informe(texto_informe: str) -> MetricasSistema:
    """
    Parsea un informe en formato 'clave = valor' línea a línea.
    Ignora líneas en blanco y comentarios (# al inicio).
    Convierte valores numéricos automáticamente.
    """
    metricas: MetricasSistema = {}

    for linea in texto_informe.strip().splitlines():
        linea = linea.strip()

        # Ignorar líneas vacías y comentarios
        if not linea or linea.startswith("#"):
            continue
        if "=" not in linea:
            continue

        # partition() es más seguro que split("=", 1) para este caso
        clave, _, valor = linea.partition("=")
        clave = clave.strip()
        valor = valor.strip()

        # Conversión automática de tipos numéricos
        try:
            metricas[clave] = int(valor)
        except ValueError:
            try:
                metricas[clave] = float(valor)
            except ValueError:
                metricas[clave] = valor   # queda como string

    return metricas


def clasificar_umbral(valor: float, critico: float, alto: float) -> str:
    """Clasifica un porcentaje según umbrales de alerta."""
    if valor >= critico:
        return "CRÍTICO"
    elif valor >= alto:
        return "ALTO"
    return "NORMAL"


def generar_diagnostico(metricas: MetricasSistema) -> str:
    """
    Produce un informe de diagnóstico legible con alineación de columnas.
    Devuelve el string completo para facilitar pruebas unitarias.
    """
    cpu    = float(metricas.get("cpu_pct", 0))
    memoria = float(metricas.get("mem_pct", 0))
    disco  = float(metricas.get("disco_pct", 0))
    nombre = str(metricas.get("host", "desconocido"))
    uptime = metricas.get("uptime_dias", "?")

    # Clasificar cada métrica con sus umbrales específicos
    est_cpu    = clasificar_umbral(cpu,    critico=90, alto=70)
    est_mem    = clasificar_umbral(memoria, critico=90, alto=75)
    est_disco  = clasificar_umbral(disco,  critico=95, alto=80)

    ahora = datetime.now().strftime("%Y-%m-%d %H:%M")

    # Determinar estado global del servidor
    estados = [est_cpu, est_mem, est_disco]
    estado_global = (
        "CRÍTICO" if "CRÍTICO" in estados
        else "DEGRADADO" if "ALTO" in estados
        else "SALUDABLE"
    )

    lineas = [
        "╔══════════════════════════════════════════════╗",
        f"║  DIAGNÓSTICO · {ahora}               ║",
        "╠══════════════════════════════════════════════╣",
        f"║  Host    : {nombre:<34}║",
        f"║  Uptime  : {str(uptime) + ' días':<34}║",
        f"║  Estado  : {estado_global:<34}║",
        "╠══════════════════════════════════════════════╣",
        f"║  CPU     : {cpu:>6.1f}%  ► {est_cpu:<26}║",
        f"║  Memoria : {memoria:>6.1f}%  ► {est_mem:<26}║",
        f"║  Disco   : {disco:>6.1f}%  ► {est_disco:<26}║",
        "╚══════════════════════════════════════════════╝",
    ]
    return "\n".join(lineas)


def detectar_alertas(
    metricas: MetricasSistema,
    umbrales: dict[str, float],
) -> list[str]:
    """Devuelve lista de alertas para métricas que superan sus umbrales."""
    alertas = []
    for clave, umbral in umbrales.items():
        # Walrus: asigna el valor y comprueba si supera el umbral
        if isinstance(valor := float(metricas.get(clave, 0)), float):
            if valor > umbral:
                alertas.append(
                    f"{clave}: {valor:.1f}% — supera umbral de {umbral:.0f}%"
                )
    return alertas


# ── Programa principal ──────────────────────────────────────────────────────

INFORME_TEXTO = """
# Informe generado por el agente de monitorización v2.1
# Timestamp: 2025-03-15T14:30:00Z

host          = servidor-web-03
uptime_dias   = 127
cpu_pct       = 78.4
mem_pct       = 91.2
disco_pct     = 64.8
red_mbps      = 450.0
temperatura_c = 67.3
procesos      = 312
"""

UMBRALES_ALERTA: dict[str, float] = {
    "cpu_pct":   70.0,
    "mem_pct":   85.0,
    "disco_pct": 80.0,
}

metricas = parsear_informe(INFORME_TEXTO)
print(generar_diagnostico(metricas))

alertas = detectar_alertas(metricas, UMBRALES_ALERTA)
if alertas:
    print("\n  ALERTAS ACTIVAS:")
    for alerta in alertas:
        print(f"  → {alerta}")
else:
    print("\n  Sin alertas activas.")
```

Salida esperada:
```
╔══════════════════════════════════════════════╗
║  DIAGNÓSTICO · 2025-03-15 14:30               ║
╠══════════════════════════════════════════════╣
║  Host    : servidor-web-03                   ║
║  Uptime  : 127 días                          ║
║  Estado  : DEGRADADO                         ║
╠══════════════════════════════════════════════╣
║  CPU     :   78.4%  ► ALTO                   ║
║  Memoria :   91.2%  ► CRÍTICO                ║
║  Disco   :   64.8%  ► NORMAL                 ║
╚══════════════════════════════════════════════╝

  ALERTAS ACTIVAS:
  → cpu_pct: 78.4% — supera umbral de 70%
  → mem_pct: 91.2% — supera umbral de 85%
```

> **Prueba esto:** Añade `red_mbps = 450.0` con umbral `400.0` en `UMBRALES_ALERTA`. ¿Aparece en las alertas? Luego modifica `clasificar_umbral()` para que devuelva `"ADVERTENCIA"` cuando el valor esté entre `alto` y un nuevo parámetro `medio`. Actualiza `generar_diagnostico()` para reflejar el nuevo nivel.

---

## Ejercicios propuestos

1. **Conversor de unidades de red** — Define variables para velocidades de red
   en bits por segundo (bps) usando separadores de miles (`1_000_000_000`).
   Conviértelas a Kbps, Mbps y Gbps con aritmética de Python. Muestra el
   resultado con f-strings formateados: separador de miles, 3 decimales y
   alineación de columnas. Ejemplo: `1_000_000_000` bps → `"1,000.000 Mbps"`
   o `"1.000 Gbps"`.

2. **Validador de dirección IPv4** — Dado un string como `"192.168.1.256"`,
   verifica que: tiene exactamente 4 partes separadas por `.`, cada parte es
   convertible a `int`, y cada entero está en el rango `0..255`. Usa `split()`,
   `int()`, `try/except`, operadores de comparación encadenados y f-strings
   para mostrar un mensaje claro: `"IP válida: 192.168.1.1"` o
   `"IP inválida: octeto 256 fuera de rango"`.

3. **Calculadora de descuentos escalonados** — Dadas las variables
   `precio_base: float` y `cantidad: int`, calcula el precio final aplicando:
   0% si `cantidad < 10`, 5% si `10 <= cantidad < 50`, 10% si `50 <= cantidad < 100`,
   15% si `cantidad >= 100`. Usa el operador walrus para capturar la tasa en
   la misma expresión. Muestra el desglose completo con f-strings.

4. **Informe de tokens de API** — Crea un `dict` con datos de una API key:
   `nombre`, `token` (string de 32 chars), `llamadas_restantes` (int),
   `vence` (string de fecha), `activa` (bool). Usa f-strings con alineación
   para imprimir un recuadro de informe. Si `llamadas_restantes < 100` o
   `activa is False`, añade una línea de advertencia usando operadores
   lógicos correctos. Demuestra la diferencia entre `or` con `None` y
   con `0` (falsy).

---

## Resumen de la página 1

- Python 3.12+ es el estándar del curso. `venv` aísla dependencias por proyecto; `pyproject.toml` es el archivo de configuración moderno equivalente a `build.gradle.kts` en Kotlin.
- Las variables se asignan directamente sin `var`/`val`. El tipo es dinámico e inferido. Los type hints (`nombre: str`) son opcionales en runtime pero esenciales para mypy y el IDE.
- `int` tiene precisión arbitraria (sin overflow). `float` es IEEE 754 de 64 bits. `str` es Unicode inmutable. `bool` es subclase de `int` (`True == 1`, `False == 0`).
- Los f-strings `f"..."` son la forma estándar de interpolar valores: admiten expresiones, llamadas a métodos y especificadores de formato (`{valor:.2f}`, `{texto:<15}`, `{n:#010x}`).
- `None` es el equivalente a `null`. La comparación correcta es `is None` / `is not None`. Usar `== None` funciona pero es mala práctica según PEP 8.
- Los type hints modernos usan `|` para uniones (`str | None`) y minúsculas para genéricos (`list[str]`, `dict[str, int]`). `type Alias = ...` define alias reutilizables (Python 3.12).
- El operador walrus `:=` asigna y evalúa simultáneamente — ideal en bucles `while chunk := f.read(8)` y comprensiones con condición de filtrado.
- Los operadores lógicos son `and`, `or`, `not` (no `&&`, `||`, `!`). Las comparaciones se encadenan: `0 <= x < 256`. El cortocircuito evita evaluaciones innecesarias.

---





> **Siguiente página →** Página 2: Condicionales — `if` simple, dos caminos, múltiples y anidadas.
