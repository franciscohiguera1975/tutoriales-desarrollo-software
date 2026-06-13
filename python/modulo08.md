# Tutorial Python — Página 8
## Módulo 8 · Errores, módulos y entornos virtuales
### `try/except/finally`, excepciones propias, módulos y `venv`

---

## Manejo de errores en Python

Python usa `try/except` (no `try/catch` como en JavaScript).
Pero la idea es la misma: intentar ejecutar código que puede fallar
y capturar el error si ocurre.

```
JavaScript                      Python
──────────────────────────────  ──────────────────────────────
try {                           try:
  ...                               ...
} catch (error) {               except TipoError as e:
  ...                               ...
} finally {                     finally:
  ...                               ...
}
```

---

## Parte 1 — `try` / `except` básico

```python
# Sin manejo de errores — el programa se detiene
# resultado = int("abc")    ← ValueError: invalid literal for int()

# Con manejo de errores — el programa continúa
try:
    numero = int("abc")
except ValueError:
    print("No se pudo convertir a número")

print("El programa sigue ejecutándose")
```

### Capturar el mensaje del error con `as`

```python
try:
    resultado = 10 / 0
except ZeroDivisionError as e:
    print(f"Error: {e}")    # Error: division by zero
    print(type(e))          # <class 'ZeroDivisionError'>
```

### Capturar múltiples tipos de error

```python
def convertir_y_dividir(texto, divisor):
    try:
        numero = int(texto)        # puede lanzar ValueError
        resultado = numero / divisor  # puede lanzar ZeroDivisionError
        return resultado

    except ValueError:
        print(f"'{texto}' no es un número entero válido")
        return None

    except ZeroDivisionError:
        print("No se puede dividir entre cero")
        return None

print(convertir_y_dividir("10", 2))     # 5.0
print(convertir_y_dividir("abc", 2))    # error → None
print(convertir_y_dividir("10", 0))     # error → None

# Capturar varios tipos en una sola línea con tupla
try:
    ...
except (ValueError, TypeError) as e:
    print(f"Error de tipo o valor: {e}")
```

---

## Parte 2 — `else` y `finally`

```python
def leer_numero(texto):
    try:
        numero = int(texto)
    except ValueError as e:
        print(f"Error: {e}")
        return None
    else:
        # Se ejecuta SOLO si no hubo ningún error en el try
        print(f"Conversión exitosa: {numero}")
        return numero
    finally:
        # Se ejecuta SIEMPRE — haya error o no
        # Ideal para limpiar recursos (cerrar archivos, conexiones, etc.)
        print("Procesamiento finalizado")


leer_numero("42")     # exitoso — imprime el else y el finally
leer_numero("abc")    # falla   — imprime el except y el finally
```

---

## Parte 3 — Jerarquía de excepciones de Python

```
BaseException
├── SystemExit          ← sys.exit()
├── KeyboardInterrupt   ← Ctrl+C
└── Exception           ← todas las demás
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── IndexError   ← lista[índice fuera de rango]
    │   └── KeyError     ← dict['clave no existente']
    ├── TypeError        ← operación con tipo incorrecto
    ├── ValueError       ← tipo correcto, valor incorrecto
    ├── AttributeError   ← obj.atributo que no existe
    ├── NameError        ← variable no definida
    ├── FileNotFoundError
    └── RuntimeError
```

```python
# Capturar Exception captura casi todo (excepto SystemExit, etc.)
try:
    resultado = peligroso()
except Exception as e:
    print(f"Ocurrió un error inesperado: {type(e).__name__}: {e}")

# EVITA capturar BaseException o Exception sin razón —
# puede esconder bugs difíciles de encontrar
```

---

## Parte 4 — Lanzar errores con `raise`

```python
# raise lanza un error — equivale a throw en JS

def dividir(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("El divisor no puede ser cero")
    return a / b

def validar_edad(edad: int) -> None:
    if not isinstance(edad, int):
        raise TypeError(f"La edad debe ser un entero, no {type(edad).__name__}")
    if edad < 0 or edad > 120:
        raise ValueError(f"Edad fuera de rango: {edad}")

# Re-lanzar el error después de loguearlo
def procesar(datos):
    try:
        return calcular(datos)
    except ValueError as e:
        print(f"[LOG] Error de validación: {e}")
        raise    # re-lanza el mismo error sin perder el traceback
```

---

## Parte 5 — Excepciones propias

Define tus propias clases de error heredando de `Exception`:

```python
# Excepciones personalizadas para un sistema de pagos
class ErrorPago(Exception):
    """Clase base para todos los errores de pago."""
    pass

class SaldoInsuficienteError(ErrorPago):
    def __init__(self, saldo_actual: float, importe: float):
        self.saldo_actual = saldo_actual
        self.importe      = importe
        super().__init__(
            f"Saldo insuficiente: tienes {saldo_actual:.2f}€, "
            f"necesitas {importe:.2f}€"
        )

class TarjetaBloqueadaError(ErrorPago):
    def __init__(self, ultimos_digitos: str):
        self.ultimos_digitos = ultimos_digitos
        super().__init__(f"Tarjeta terminada en {ultimos_digitos} bloqueada")


def procesar_pago(importe: float, saldo: float, tarjeta_activa: bool, digitos: str):
    if not tarjeta_activa:
        raise TarjetaBloqueadaError(digitos)
    if saldo < importe:
        raise SaldoInsuficienteError(saldo, importe)
    return saldo - importe


# Capturar por jerarquía
try:
    nuevo_saldo = procesar_pago(100, 50, True, "1234")
except SaldoInsuficienteError as e:
    print(f"Saldo insuficiente: necesitas {e.importe - e.saldo_actual:.2f}€ más")
except TarjetaBloqueadaError as e:
    print(f"Llama al banco — {e}")
except ErrorPago as e:
    print(f"Error de pago genérico: {e}")
```

---

## Parte 6 — Context managers y `with`

`with` garantiza que los recursos se liberan aunque ocurra un error.
Equivale a `try/finally` pero más limpio.

```python
# Leer un archivo — con with se cierra automáticamente
# Aunque ocurra un error en el interior, el archivo se cierra
with open("datos.txt", "r", encoding="utf-8") as archivo:
    contenido = archivo.read()
    print(contenido)
# Aquí el archivo ya está cerrado

# Sin with — necesitas try/finally manual:
archivo = open("datos.txt", "r")
try:
    contenido = archivo.read()
finally:
    archivo.close()   # hay que cerrar siempre

# Escribir un archivo
with open("salida.txt", "w", encoding="utf-8") as f:
    f.write("Primera línea\n")
    f.write("Segunda línea\n")

# Leer línea a línea — eficiente para archivos grandes
with open("datos.txt", "r", encoding="utf-8") as f:
    for linea in f:
        print(linea.strip())    # strip() elimina el \n del final
```

---

## Parte 7 — Módulos e importaciones

### Importar módulos de la biblioteca estándar

```python
# Forma 1 — importar el módulo completo
import math
print(math.sqrt(16))       # 4.0
print(math.pi)             # 3.141592...

# Forma 2 — importar funciones específicas (evita el prefijo)
from math import sqrt, pi, ceil, floor
print(sqrt(25))            # 5.0 — sin "math."

# Forma 3 — importar con alias
import datetime as dt
hoy = dt.date.today()
print(hoy)                 # 2024-03-15

from datetime import datetime as DT
ahora = DT.now()
print(ahora.strftime("%H:%M:%S"))   # hora actual formateada
```

### Módulos de la biblioteca estándar más útiles

```python
import os           # sistema de archivos y variables de entorno
import sys          # argumentos de línea de comandos
import json         # serializar/deserializar JSON
import re           # expresiones regulares
import random       # números aleatorios
import time         # tiempo y pausas
import pathlib      # rutas de archivo modernas
import collections  # Counter, defaultdict, deque
import itertools    # combinaciones, permutaciones
import functools    # reduce, lru_cache, partial

# Ejemplos rápidos
import json
datos = {"nombre": "Ana", "edad": 28}
json_str = json.dumps(datos, ensure_ascii=False, indent=2)
print(json_str)

recuperado = json.loads(json_str)
print(recuperado["nombre"])    # Ana

import random
print(random.randint(1, 10))           # número entre 1 y 10
print(random.choice(["A", "B", "C"]))  # elemento aleatorio
lista = [1, 2, 3, 4, 5]
random.shuffle(lista)
print(lista)                           # orden aleatorio

import re
emails = "Contactos: ana@ej.com y luis@otro.es"
encontrados = re.findall(r'[\w.]+@[\w.]+', emails)
print(encontrados)    # ['ana@ej.com', 'luis@otro.es']
```

### Crear tus propios módulos

```python
# src/calculadora.py — un módulo propio
def sumar(a, b):
    return a + b

def restar(a, b):
    return a - b

PI = 3.14159

# main.py — importa el módulo propio
from src.calculadora import sumar, PI

print(sumar(3, 4))    # 7
print(PI)             # 3.14159
```

---

## Parte 8 — Entornos virtuales con `venv`

Un **entorno virtual** aisla las dependencias de un proyecto para que
no interfieran con otros proyectos o con el Python del sistema.

```bash
# Crear el entorno virtual
python3 -m venv .venv

# Activar el entorno
source .venv/bin/activate       # macOS / Linux
.venv\Scripts\activate          # Windows

# Verificar que estás en el entorno
which python                    # debe apuntar a .venv/bin/python
pip list                        # muestra los paquetes instalados

# Instalar dependencias
pip install requests            # instala en el entorno virtual
pip install flask sqlalchemy    # varios paquetes
pip install "fastapi[all]"      # paquete con extras

# Guardar las dependencias en un archivo
pip freeze > requirements.txt

# Instalar desde requirements.txt (en otro equipo o CI/CD)
pip install -r requirements.txt

# Desactivar el entorno
deactivate
```

### Estructura recomendada del proyecto

```
mi-proyecto/
├── .venv/                 ← entorno virtual (no subir a git)
├── src/
│   ├── __init__.py
│   ├── calculadora.py
│   └── utilidades.py
├── tests/
│   ├── __init__.py
│   └── test_calculadora.py
├── .gitignore             ← incluye .venv/
├── requirements.txt
└── README.md
```

```gitignore
# .gitignore
.venv/
__pycache__/
*.pyc
.env
```

---

## Parte 9 — Ejemplo completo: lector de CSV con errores

```python
import csv
import json
from pathlib import Path


class ErrorArchivo(Exception):
    """Error base para operaciones de archivo."""
    pass

class ArchivoNoEncontradoError(ErrorArchivo):
    pass

class FormatoInvalidoError(ErrorArchivo):
    pass


def leer_productos_csv(ruta: str) -> list[dict]:
    """
    Lee un archivo CSV de productos y devuelve una lista de dicts.
    Formato esperado: nombre,precio,stock
    """
    archivo = Path(ruta)

    if not archivo.exists():
        raise ArchivoNoEncontradoError(f"No se encontró: {ruta}")

    if archivo.suffix.lower() != ".csv":
        raise FormatoInvalidoError(f"Se esperaba .csv, se recibió {archivo.suffix}")

    productos = []
    errores   = []

    with open(archivo, "r", encoding="utf-8") as f:
        lector = csv.DictReader(f)

        for num_linea, fila in enumerate(lector, start=2):
            try:
                producto = {
                    "nombre": fila["nombre"].strip(),
                    "precio": float(fila["precio"]),
                    "stock":  int(fila["stock"]),
                }
                if producto["precio"] < 0:
                    raise ValueError("El precio no puede ser negativo")
                productos.append(producto)

            except (ValueError, KeyError) as e:
                errores.append(f"Línea {num_linea}: {e}")

    if errores:
        print(f"⚠️  Se encontraron {len(errores)} errores:")
        for err in errores:
            print(f"   · {err}")

    return productos


def guardar_json(datos: list, ruta: str) -> None:
    """Guarda una lista de dicts como JSON."""
    with open(ruta, "w", encoding="utf-8") as f:
        json.dump(datos, f, ensure_ascii=False, indent=2)
    print(f"✅ Guardado en {ruta}")


# Uso con manejo de errores
try:
    productos = leer_productos_csv("productos.csv")
    print(f"Se cargaron {len(productos)} productos")
    guardar_json(productos, "productos.json")

except ArchivoNoEncontradoError as e:
    print(f"❌ Archivo no encontrado: {e}")

except FormatoInvalidoError as e:
    print(f"❌ Formato incorrecto: {e}")

except PermissionError:
    print("❌ Sin permisos para leer el archivo")

except Exception as e:
    print(f"❌ Error inesperado: {type(e).__name__}: {e}")
    raise    # re-lanza para no perder el traceback en desarrollo
```

---

## Ejercicios propuestos

1. **Conversor seguro** — Escribe `convertir_a_int(texto)` que use
   `try/except` para devolver el entero si la conversión es posible,
   o `None` si no lo es. Luego escribe `convertir_lista(textos)` que use
   una list comprehension con esa función para convertir una lista mixta
   como `["1", "dos", "3", "cuatro", "5"]` → `[1, None, 3, None, 5]`.

2. **Excepciones propias** — Crea la jerarquía `ErrorStock(Exception)` →
   `StockInsuficienteError` y `ProductoNoEncontradoError`. Úsalas en
   una clase `Almacen` con los métodos `retirar(nombre, cantidad)` y
   `consultar(nombre)`. Demuestra que puedes capturarlas por el tipo
   específico o por el tipo base `ErrorStock`.

3. **Leer y escribir JSON** — Escribe `cargar_config(ruta)` que use
   `with open(...)` para leer un archivo JSON y devuelva el diccionario.
   Si el archivo no existe, devuelva una configuración por defecto.
   Si el JSON es inválido, lance un `FormatoInvalidoError` propio.

4. **Proyecto completo** — Crea un pequeño gestor de tareas de consola
   usando todo lo aprendido: clase `Tarea` con `@dataclass`, clase
   `GestorTareas` con `agregar`, `completar` y `listar`, manejo de errores
   con excepciones propias, guardado/carga en JSON con `with open`,
   y un entorno virtual con `venv` para ejecutarlo.

---

## Recapitulación del curso

| Módulo | Tema |
|---|---|
| 1 | Variables, tipos básicos, `None`, truthy/falsy |
| 2 | Strings, f-strings, slicing, métodos |
| 3 | Listas, tuplas, sets, diccionarios |
| 4 | `if/elif/else`, operador ternario, `match` |
| 5 | `for`, `while`, `range`, `enumerate`, `zip`, comprehensions |
| 6 | Funciones, `*args`, `**kwargs`, lambdas, type hints |
| 7 | Clases, herencia, `@property`, `@dataclass`, métodos dunder |
| 8 | Errores, módulos, `venv`, archivos con `with` |

---

## Resumen de la página 8

- `try/except` captura errores — `except TipoError as e` da acceso al mensaje. Captura siempre el tipo más específico posible.
- `else` en `try` se ejecuta solo si no hubo error. `finally` se ejecuta siempre — ideal para liberar recursos.
- `raise` lanza un error explícitamente. `raise` sin argumentos re-lanza el error actual.
- Las **excepciones propias** heredan de `Exception` y permiten añadir atributos con información del error.
- `with` garantiza que el recurso se cierra aunque ocurra un error — imprescindible para archivos y conexiones.
- Los **módulos** se importan con `import modulo` o `from modulo import funcion`. Crea los tuyos propios con archivos `.py`.
- `venv` crea entornos virtuales aislados. `pip freeze > requirements.txt` guarda las dependencias para reproducir el entorno.

---

> **¡Fin del tutorial!** Ahora conoces los fundamentos de Python.
> El siguiente paso natural es elegir una especialización:
> **Desarrollo web** con FastAPI o Django · **Ciencia de datos** con Pandas y NumPy
> · **Automatización** con scripts y la biblioteca estándar
> · **Pruebas** con pytest — el equivalente a Jest en Python.
