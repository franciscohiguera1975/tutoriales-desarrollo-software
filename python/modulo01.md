# Tutorial Python — Página 1
## Módulo 1 · Introducción a Python
### ¿Qué es Python?, instalación, variables y tipos básicos

---

## ¿Qué es Python?

Python es un lenguaje de programación de propósito general, interpretado
y de tipado dinámico. Se usa en desarrollo web, ciencia de datos,
inteligencia artificial, automatización de tareas y scripting de sistemas.

```
JavaScript                      Python
─────────────────────────────   ─────────────────────────────
Ejecuta en el navegador         Ejecuta en el servidor / terminal
  (también en Node.js)          (también en el navegador con Pyodide)
Llaves { } para bloques         Indentación para bloques
Punto y coma opcional ;         Sin punto y coma
var / let / const               Solo nombres de variable
Tipado dinámico                 Tipado dinámico
async/await                     async/await
npm para paquetes               pip para paquetes
```

### ¿Por qué aprender Python si ya sabes JavaScript?

- Las estructuras de datos son muy similares (listas ≈ arrays, dicts ≈ objetos)
- La lógica es la misma — solo cambia la sintaxis
- Python domina en ciencia de datos, IA y automatización
- Los scripts de Python son muy cortos comparados con otros lenguajes

---

## Versiones del curso

| Herramienta | Versión |
|---|---|
| Python | **3.12+** |
| pip | Incluido con Python 3.12+ |
| venv | Incluido con Python 3.12+ |
| Editor | VS Code con extensión Python |

---

## Parte 1 — Instalación

```bash
# macOS con Homebrew
brew install python@3.12

# Ubuntu / Debian
sudo apt update && sudo apt install python3.12 python3-pip

# Windows — descargar desde python.org
# Marcar "Add Python to PATH" durante la instalación

# Verificar instalación
python3 --version    # Python 3.12.x
pip3 --version       # pip 24.x
```

### VS Code — extensión recomendada

```
Python    (id: ms-python.python)          ← obligatoria
Pylance   (id: ms-python.vscode-pylance)  ← autocompletado potente
Ruff      (id: charliermarsh.ruff)        ← linter y formateador
```

### Ejecutar un script Python

```bash
# Crear un archivo
echo 'print("Hola, Python")' > hola.py

# Ejecutarlo
python3 hola.py    # Hola, Python

# Modo interactivo (REPL) — ideal para probar cosas
python3
>>> 2 + 2
4
>>> print("hola")
hola
>>> exit()
```

---

## Parte 2 — Tu primer programa

```python
# hola.py
# En Python los comentarios usan # (igual que en bash)
# No hay // ni /* */ como en JavaScript

print("Hola, Python")         # equivale a console.log() en JS
print("La suma de 2+3 es", 2 + 3)

# input() lee texto del usuario — equivale a prompt() en el navegador
nombre = input("¿Cómo te llamas? ")
print("Hola,", nombre)
```

```bash
python3 hola.py
# Hola, Python
# La suma de 2+3 es 5
# ¿Cómo te llamas? Ana
# Hola, Ana
```

---

## Parte 3 — Variables y tipos básicos

En Python **no se declaran** las variables con `let`, `const` ni `var`.
Simplemente asignas y Python infiere el tipo.

```python
# JavaScript             # Python — equivalente
# let nombre = "Ana"     nombre = "Ana"
# const edad = 28        edad = 28
# let precio = 89.99     precio = 89.99
# let activo = true      activo = True    ← True/False con mayúscula
# let nada = null        nada = None      ← None en lugar de null

# Verificar el tipo — equivale a typeof en JS
print(type(nombre))      # <class 'str'>
print(type(edad))        # <class 'int'>
print(type(precio))      # <class 'float'>
print(type(activo))      # <class 'bool'>
print(type(nada))        # <class 'NoneType'>
```

### Los tipos básicos en Python

```python
# ── Entero (int) ─────────────────────────────────────────────────
poblacion = 8_000_000_000      # el _ separa miles — solo visual
temperatura = -15
binario = 0b1010               # 10 en binario
hexadecimal = 0xFF             # 255 en hexadecimal

# ── Decimal (float) ──────────────────────────────────────────────
precio = 29.99
pi = 3.141592653589793
notacion = 1.5e3               # 1500.0

# ── Booleano (bool) ──────────────────────────────────────────────
# En Python es True/False con mayúscula (en JS es true/false)
conectado = True
error = False

# Los booleanos son subclase de int en Python
print(True + True)    # 2 — curioso pero útil a veces
print(True * 5)       # 5

# ── None — el equivalente de null ────────────────────────────────
resultado = None
print(resultado is None)    # True  ← usa "is" para comparar con None
print(resultado == None)    # True  ← funciona pero no es idiomático
```

### Diferencia entre `int` y `float`

```python
# En JS todos los números son float. En Python hay dos tipos distintos.
a = 10       # int
b = 10.0     # float
c = 10 / 3   # float → 3.3333... (en JS igual)
d = 10 // 3  # int   → 3 (división entera — equivale a Math.floor(10/3))
e = 10 % 3   # int   → 1 (módulo — igual que en JS)
f = 2 ** 8   # int   → 256 (potencia — en JS sería 2 ** 8 también)

print(type(a))   # <class 'int'>
print(type(c))   # <class 'float'>
print(type(d))   # <class 'int'>
```

---

## Parte 4 — Asignación múltiple y constantes

```python
# Asignación múltiple en una línea
x, y, z = 1, 2, 3
print(x, y, z)    # 1 2 3

# Intercambiar valores — en JS necesitarías una variable temporal
a, b = 10, 20
a, b = b, a       # swap elegante en Python
print(a, b)       # 20 10

# Asignar el mismo valor a varias variables
contador = total = puntos = 0
print(contador, total, puntos)    # 0 0 0

# Constantes — Python no tiene const
# Por convención se escriben en MAYÚSCULAS (pero no son inmutables)
MAX_INTENTOS = 3
PI = 3.14159
URL_BASE = "https://api.ejemplo.com"
```

---

## Parte 5 — Conversión de tipos

```python
# Conversión explícita — equivale a parseInt(), parseFloat() en JS

# String → número
edad   = int("28")           # 28 (int)
precio = float("89.99")      # 89.99 (float)

# Número → string
texto = str(42)              # "42"
texto2 = str(3.14)           # "3.14"

# Número → booleano
print(bool(0))               # False  ← 0 es falsy
print(bool(1))               # True
print(bool(-1))              # True   ← cualquier número no cero es truthy

# String → booleano
print(bool(""))              # False  ← string vacío es falsy
print(bool("hola"))          # True

# Conversión segura — int() lanza ValueError si el string no es un número
try:
    numero = int("abc")      # lanza ValueError
except ValueError:
    print("No se pudo convertir")

# Verificar si un string es un número antes de convertir
texto = "123"
if texto.isdigit():
    numero = int(texto)
    print(numero)            # 123
```

---

## Parte 6 — Valores truthy y falsy

Python tiene el mismo concepto que JavaScript. Los valores falsy son:

```python
# Valores falsy en Python
valores_falsy = [False, 0, 0.0, "", [], {}, set(), None, 0j]

for valor in valores_falsy:
    if not valor:
        print(f"{repr(valor)} es falsy")

# Valores truthy — todo lo demás
# Incluye listas no vacías, strings no vacíos, números distintos de 0

# En JS:            En Python:
# if (lista.length) if lista:      ← forma idiomática en Python
# if (obj)          if diccionario:
# if (str !== '')   if texto:
```

---

## Parte 7 — El REPL como herramienta de aprendizaje

El REPL (Read-Eval-Print Loop) de Python es ideal para experimentar:

```python
# Abrir en terminal: python3
>>> 10 / 3
3.3333333333333335
>>> 10 // 3
3
>>> type(3.14)
<class 'float'>
>>> isinstance(True, int)   # ← bool es subclase de int
True
>>> "hola".upper()
'HOLA'
>>> help(str.split)         # ← documentación en el REPL
```

---

## Ejercicios propuestos

1. **Tipos y conversiones** — Crea variables de cada tipo básico (`int`, `float`,
   `bool`, `str`, `None`). Imprime su valor y su tipo con `type()`. Convierte
   el `int` a `float`, el `float` a `str`, y el `str` de vuelta a `float`.
   Imprime el resultado de cada conversión.

2. **Calculadora básica** — Escribe un programa que pida dos números con `input()`,
   los convierta a `float` con `float()`, y muestre: la suma, la resta, el
   producto, la división, la división entera y el módulo. Usa `print()` con
   mensajes descriptivos.

3. **Intercambio sin variable temporal** — Demuestra el swap de Python:
   pide dos valores al usuario, muéstralos, intercámbialos con `a, b = b, a`
   y muéstralos de nuevo. Compara mentalmente con cómo lo harías en JavaScript.

4. **Explorar truthy/falsy** — Crea una lista con al menos 8 valores distintos
   (mezcla de strings, números, `None`, listas vacías, etc.). Usa un bucle
   para imprimir cada valor y si es truthy o falsy usando `if valor: print("truthy")
   else: print("falsy")`.

---

## Resumen de la página 1

- Python no usa `var`/`let`/`const` — las variables se crean con simple asignación. El tipo se infiere automáticamente.
- Los tipos básicos son `int`, `float`, `bool`, `str` y `NoneType`. `bool` es subclase de `int` — `True == 1` y `False == 0`.
- `None` es el equivalente de `null`. Compáralo siempre con `is None`, nunca con `== None`.
- `True`/`False` llevan mayúscula (diferente a JavaScript donde son `true`/`false`).
- La división `/` siempre devuelve `float`. La división entera `//` devuelve `int`. La potencia es `**`.
- La asignación múltiple `a, b = 1, 2` y el swap `a, b = b, a` son modismos propios de Python.
- Los valores falsy son: `False`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `None`. Todo lo demás es truthy.

---

> **Siguiente página →** Página 2: Strings en Python — f-strings, slicing,
> métodos y cómo se comparan con los template literals de JavaScript.
