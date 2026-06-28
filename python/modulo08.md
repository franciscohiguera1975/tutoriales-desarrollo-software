# Tutorial Python — Página 8
## Módulo 5 · Bucles
### `for`, `while`, `range`, `enumerate`, `zip` y list comprehensions

---

## Bucles en Python vs JavaScript

```
JavaScript                      Python
──────────────────────────────  ──────────────────────────────
for (let i = 0; i < 5; i++)     for i in range(5):
for (const x of lista)          for x in lista:
for (const k in obj)            for k in diccionario:
while (condicion)               while condicion:
lista.forEach(fn)               for x in lista:  (o comprehension)
lista.map(fn)                   [fn(x) for x in lista]
lista.filter(fn)                [x for x in lista if fn(x)]
```

---

## Parte 1 — `for` con iterables

En Python `for` itera directamente sobre los elementos de cualquier
iterable (lista, string, diccionario, rango, etc.). No necesitas el índice.

```python
# Iterar sobre una lista
frutas = ['manzana', 'naranja', 'pera', 'uva']

for fruta in frutas:
    print(fruta)

# Iterar sobre un string — carácter a carácter
for letra in "Python":
    print(letra, end=" ")    # end=" " evita el salto de línea
# P y t h o n

# Iterar sobre un diccionario
precios = {'Ratón': 25, 'Teclado': 50, 'Monitor': 280}

for producto in precios:             # itera sobre claves
    print(producto)

for precio in precios.values():      # itera sobre valores
    print(precio)

for producto, precio in precios.items():   # itera sobre pares — la más usada
    print(f"{producto}: {precio}€")
```

---

## Parte 2 — `range()` — generar secuencias de números

```python
# range(fin)           → 0, 1, 2, ..., fin-1
# range(inicio, fin)   → inicio, inicio+1, ..., fin-1
# range(inicio, fin, paso)

# Contar del 0 al 4
for i in range(5):
    print(i, end=" ")    # 0 1 2 3 4

# Contar del 1 al 5
for i in range(1, 6):
    print(i, end=" ")    # 1 2 3 4 5

# Contar de 2 en 2
for i in range(0, 11, 2):
    print(i, end=" ")    # 0 2 4 6 8 10

# Contar hacia atrás
for i in range(5, 0, -1):
    print(i, end=" ")    # 5 4 3 2 1

# Convertir range a lista
print(list(range(1, 6)))    # [1, 2, 3, 4, 5]
```

### `range` en situaciones reales

```python
# Tabla de multiplicar del 7
for i in range(1, 11):
    print(f"7 × {i:2d} = {7 * i:3d}")

# Generar IDs del 1 al 5
ids = list(range(1, 6))    # [1, 2, 3, 4, 5]

# Crear una lista de cuadrados
cuadrados = list(range(1, 6))
for i, n in enumerate(cuadrados):
    cuadrados[i] = n ** 2
print(cuadrados)    # [1, 4, 9, 16, 25]
```

---

## Parte 3 — `enumerate()` — índice y valor

Cuando necesitas el índice Y el valor al mismo tiempo,
usa `enumerate()` en lugar de `range(len(lista))`.

```python
# Sin enumerate — verboso y propenso a errores
nombres = ['Ana', 'Luis', 'María']
for i in range(len(nombres)):
    print(f"{i+1}. {nombres[i]}")

# Con enumerate — limpio e idiomático
for indice, nombre in enumerate(nombres):
    print(f"{indice+1}. {nombre}")

# start=1 para que el índice empiece en 1
for posicion, nombre in enumerate(nombres, start=1):
    print(f"{posicion}. {nombre}")
```

### Ejemplo — tabla de ranking

```python
def mostrar_ranking(puntuaciones):
    """
    puntuaciones: [(nombre, puntos), ...]
    """
    ordenadas = sorted(puntuaciones, key=lambda x: x[1], reverse=True)

    print("=== Ranking ===")
    for posicion, (nombre, puntos) in enumerate(ordenadas, start=1):
        medalla = {1: "🥇", 2: "🥈", 3: "🥉"}.get(posicion, f"{posicion}.")
        print(f"{medalla} {nombre:<12} {puntos:>6} pts")


puntuaciones = [
    ("Ana",    8500),
    ("Luis",   12000),
    ("María",  9750),
    ("Carlos", 7300),
]

mostrar_ranking(puntuaciones)
```

Salida:

```
=== Ranking ===
🥇 Luis         12000 pts
🥈 María         9750 pts
🥉 Ana           8500 pts
4. Carlos        7300 pts
```

---

## Parte 4 — `zip()` — iterar dos listas a la vez

`zip()` combina dos o más iterables en paralelo.

```python
nombres = ['Ana', 'Luis', 'María']
edades  = [28, 35, 22]
emails  = ['ana@ej.com', 'luis@ej.com', 'maria@ej.com']

# Iterar dos listas a la vez
for nombre, edad in zip(nombres, edades):
    print(f"{nombre}: {edad} años")

# Iterar tres listas a la vez
for nombre, edad, email in zip(nombres, edades, emails):
    print(f"{nombre} ({edad}) → {email}")

# Crear lista de tuplas con zip
pares = list(zip(nombres, edades))
print(pares)    # [('Ana', 28), ('Luis', 35), ('María', 22)]

# Crear diccionario con zip
claves  = ['nombre', 'edad', 'ciudad']
valores = ['Ana',    28,     'Madrid']
persona = dict(zip(claves, valores))
print(persona)  # {'nombre': 'Ana', 'edad': 28, 'ciudad': 'Madrid'}
```

---

## Parte 5 — `break`, `continue` y `else` en bucles

```python
# break — sale del bucle completamente
numeros = [3, 7, 2, 8, 1, 5]
for n in numeros:
    if n > 6:
        print(f"Primer número mayor que 6: {n}")
        break     # sale del for

# continue — salta la iteración actual
for n in range(1, 11):
    if n % 2 == 0:
        continue  # salta los pares
    print(n, end=" ")    # 1 3 5 7 9

# else en bucles — se ejecuta si el bucle terminó SIN break
# (no existe en JavaScript)
def buscar_primo(numeros):
    for n in numeros:
        if n > 1:
            for i in range(2, n):
                if n % i == 0:
                    break          # n no es primo
            else:
                print(f"{n} es primo")
                return n
    return None
```

---

## Parte 6 — `while`

```python
# while se usa cuando no sabes cuántas iteraciones necesitas

intentos   = 0
max_intentos = 3
conectado  = False

while not conectado and intentos < max_intentos:
    intentos += 1
    print(f"Intento {intentos} de conexión...")
    # Simular que conecta en el 3er intento
    if intentos == 3:
        conectado = True

if conectado:
    print("Conexión exitosa")
else:
    print("No se pudo conectar tras 3 intentos")

# while True — bucle infinito controlado con break
total = 0
print("Suma de números (0 para terminar):")
while True:
    numero = int(input("> "))
    if numero == 0:
        break
    total += numero

print(f"Total: {total}")
```

---

## Parte 7 — List comprehensions ⭐

Las **list comprehensions** son una de las características más
populares y pythónicas. Crean listas en una sola línea, reemplazando
`map()` y `filter()` de JavaScript.

```python
# Patrón: [expresión for elemento in iterable if condición]

# ── Sin comprehension (estilo JS) ────────────────────────────────
cuadrados = []
for n in range(1, 6):
    cuadrados.append(n ** 2)
print(cuadrados)    # [1, 4, 9, 16, 25]

# ── Con comprehension (estilo Python) ────────────────────────────
cuadrados = [n ** 2 for n in range(1, 6)]
print(cuadrados)    # [1, 4, 9, 16, 25]

# ── Con filtro (equivale a filter() de JS) ───────────────────────
pares = [n for n in range(1, 11) if n % 2 == 0]
print(pares)    # [2, 4, 6, 8, 10]

# ── Transformar strings ──────────────────────────────────────────
nombres = ['  ana  ', 'LUIS', 'María ']
limpios = [n.strip().title() for n in nombres]
print(limpios)  # ['Ana', 'Luis', 'María']

# ── Desde un diccionario ─────────────────────────────────────────
precios = {'Ratón': 25, 'Teclado': 50, 'Monitor': 280}

# Nombres de los productos que cuestan más de 40€
caros = [nombre for nombre, precio in precios.items() if precio > 40]
print(caros)    # ['Teclado', 'Monitor']
```

### Dictionary comprehension y set comprehension

```python
# Dict comprehension — crea un diccionario
numeros = [1, 2, 3, 4, 5]
cuadrados_dict = {n: n**2 for n in numeros}
print(cuadrados_dict)    # {1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# Invertir claves y valores de un diccionario
original  = {'a': 1, 'b': 2, 'c': 3}
invertido = {v: k for k, v in original.items()}
print(invertido)         # {1: 'a', 2: 'b', 3: 'c'}

# Set comprehension — como list comprehension pero con {}
letras = {letra.upper() for letra in "programacion"}
print(letras)    # {'P','R','O','G','A','M','C','I','N'} — sin duplicados
```

---

## Parte 8 — Ejemplo completo: análisis de ventas

```python
ventas = [
    {"producto": "Teclado",  "precio": 50,  "cantidad": 3, "categoria": "periféricos"},
    {"producto": "Ratón",    "precio": 25,  "cantidad": 8, "categoria": "periféricos"},
    {"producto": "Monitor",  "precio": 280, "cantidad": 2, "categoria": "pantallas"},
    {"producto": "Cable USB","precio": 8,   "cantidad": 15,"categoria": "accesorios"},
    {"producto": "Webcam",   "precio": 45,  "cantidad": 5, "categoria": "periféricos"},
    {"producto": "Hub USB",  "precio": 20,  "cantidad": 10,"categoria": "accesorios"},
]

# 1. Total por producto con comprehension
totales = {
    v["producto"]: v["precio"] * v["cantidad"]
    for v in ventas
}

# 2. Solo los productos que generan más de 100€ de ingreso
rentables = [
    v["producto"]
    for v in ventas
    if v["precio"] * v["cantidad"] > 100
]

# 3. Total por categoría — requiere un bucle normal
por_categoria = {}
for venta in ventas:
    cat    = venta["categoria"]
    importe = venta["precio"] * venta["cantidad"]
    por_categoria[cat] = por_categoria.get(cat, 0) + importe

# 4. Mostrar resultados
print("=== Ingresos por producto ===")
for producto, total in sorted(totales.items(), key=lambda x: x[1], reverse=True):
    print(f"  {producto:<12} {total:>8.2f}€")

print(f"\nProductos rentables (>100€): {', '.join(rentables)}")

print("\n=== Ingresos por categoría ===")
for cat, total in sorted(por_categoria.items(), key=lambda x: x[1], reverse=True):
    print(f"  {cat:<15} {total:>8.2f}€")

print(f"\nTotal general: {sum(totales.values()):.2f}€")
```

---

## Ejercicios propuestos

1. **Tabla de multiplicar** — Usa bucles `for` anidados con `range` para
   imprimir la tabla de multiplicar del 1 al 5 × 1 al 5. Usa `enumerate`
   para numerar las filas y f-strings para alinear las columnas.

2. **Filtrar y transformar** — Dada la lista de diccionarios del ejemplo,
   escribe en una sola list comprehension que devuelva los nombres de los
   productos con precio entre 20 y 100€, en mayúsculas y ordenados
   alfabéticamente (pista: ordena después con `sorted()`).

3. **Menú interactivo** — Usa `while True` con `break` para crear un
   menú de consola con opciones: 1) sumar número al total, 2) ver total,
   3) limpiar total, 0) salir. Valida que la opción sea un número con
   `try/except` antes de convertir con `int()`.

4. **Frecuencia de palabras** — Usa un bucle y un diccionario para contar
   cuántas veces aparece cada palabra en un texto (convierte a minúsculas
   y elimina puntuación con `replace`). Luego usa un dict comprehension
   para filtrar solo las palabras que aparecen más de una vez.

---

## Resumen de la página 5

- `for x in coleccion` itera directamente sobre los elementos sin necesitar índice — más limpio que el `for` clásico de JS.
- `range(inicio, fin, paso)` genera secuencias de enteros. `range(5)` → 0,1,2,3,4.
- `enumerate(lista, start=1)` da el índice y el valor al mismo tiempo — evita `range(len(lista))`.
- `zip(lista1, lista2)` combina dos iterables en paralelo — ideal para crear diccionarios desde dos listas.
- `break` sale del bucle, `continue` salta la iteración. `else` en un bucle se ejecuta si terminó **sin** `break` — no existe en JS.
- **List comprehension** `[expr for x in it if cond]` reemplaza `map` + `filter` de JS en una línea limpia y legible.
- **Dict comprehension** `{k: v for k, v in ...}` crea diccionarios de forma concisa.
- Usa `for` cuando conoces el número de iteraciones, `while` cuando dependes de una condición que puede cambiar.

---

> **Siguiente página →** Página 9: POO avanzada — dataclasses, decoradores propios y descriptores.
> nombrados, valores por defecto, `*args`, `**kwargs` y funciones lambda.
