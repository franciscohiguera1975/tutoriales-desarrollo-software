# Tutorial Python — Página 3
## Módulo 3 · Colecciones
### Listas, tuplas, sets y diccionarios

---

## Las cuatro colecciones de Python

```
Colección    Mutable   Ordenada   Duplicados   Equivalente JS
──────────   ───────   ────────   ──────────   ──────────────
list         Sí        Sí         Sí           Array
tuple        No        Sí         Sí           Array congelado
set          Sí        No         No           Set (ES6)
dict         Sí        Sí*        Claves: No   Object / Map
```
*Los diccionarios mantienen el orden de inserción desde Python 3.7.

---

## Parte 1 — Listas (`list`)

La **lista** es la colección más usada. Es equivalente al array de JavaScript.

```python
# Crear una lista
frutas   = ['manzana', 'naranja', 'pera']
numeros  = [1, 2, 3, 4, 5]
mixta    = [1, 'hola', True, None, 3.14]  # puede mezclar tipos
vacia    = []

# Acceder por índice (igual que en JS)
print(frutas[0])      # manzana   — primer elemento
print(frutas[-1])     # pera      — último elemento
print(frutas[-2])     # naranja   — penúltimo

# Slicing — igual que en strings
print(frutas[0:2])    # ['manzana', 'naranja']
print(frutas[1:])     # ['naranja', 'pera']
print(frutas[::-1])   # ['pera', 'naranja', 'manzana'] — invertida
```

### Métodos principales

```python
colores = ['rojo', 'verde', 'azul']

# ── Añadir elementos ─────────────────────────────────────────────
colores.append('amarillo')         # añade al final — como push() en JS
colores.insert(1, 'morado')        # inserta en posición 1
colores.extend(['negro', 'blanco'])# añade todos los de otra lista

# ── Eliminar elementos ───────────────────────────────────────────
colores.remove('verde')            # elimina la primera ocurrencia por valor
ultimo = colores.pop()             # elimina y devuelve el último — como pop() en JS
segundo = colores.pop(1)           # elimina y devuelve el elemento en índice 1

# ── Buscar ───────────────────────────────────────────────────────
print('rojo' in colores)           # True  — operador in
print(colores.index('azul'))       # índice de 'azul' (lanza ValueError si no existe)
print(colores.count('rojo'))       # cuántas veces aparece

# ── Ordenar ──────────────────────────────────────────────────────
numeros = [3, 1, 4, 1, 5, 9, 2, 6]
numeros.sort()                     # ordena en el lugar (modifica la lista)
print(numeros)                     # [1, 1, 2, 3, 4, 5, 6, 9]

numeros.sort(reverse=True)         # orden descendente
print(numeros)                     # [9, 6, 5, 4, 3, 2, 1, 1]

# sorted() — devuelve una nueva lista sin modificar la original
original = [3, 1, 4, 1, 5]
ordenada = sorted(original)
print(original)                    # [3, 1, 4, 1, 5]  ← sin cambios
print(ordenada)                    # [1, 1, 3, 4, 5]

# ── Otros ────────────────────────────────────────────────────────
colores.reverse()                  # invierte en el lugar
copia = colores.copy()             # copia superficial — como [...arr] en JS
colores.clear()                    # vacía la lista
print(len(copia))                  # longitud — equivale a .length en JS
```

### Listas de listas (matrices)

```python
# Tabla de calificaciones: 3 estudiantes, 4 notas cada uno
notas = [
    [8, 7, 9, 6],   # Ana
    [5, 8, 7, 9],   # Luis
    [10, 9, 8, 10], # María
]

print(notas[0])       # [8, 7, 9, 6]        — fila de Ana
print(notas[0][2])    # 9                   — tercera nota de Ana
print(notas[1][0])    # 5                   — primera nota de Luis

# Promedio de Ana
promedio_ana = sum(notas[0]) / len(notas[0])
print(f"Promedio de Ana: {promedio_ana:.1f}")   # 7.5
```

---

## Parte 2 — Tuplas (`tuple`)

Las **tuplas** son listas **inmutables** — sus elementos no pueden
cambiarse después de creadas. Son más rápidas que las listas y
se usan para datos que no deben modificarse.

```python
# Crear una tupla — con paréntesis (o sin ellos)
coordenadas = (40.4168, -3.7038)   # latitud, longitud de Madrid
rgb_rojo    = (255, 0, 0)
punto       = 3, 4                  # sin paréntesis también es tupla

# Acceso por índice — igual que lista
print(coordenadas[0])    # 40.4168
print(rgb_rojo[-1])      # 0

# Las tuplas son inmutables
# coordenadas[0] = 0   ← TypeError: 'tuple' object does not support item assignment

# Desempaquetado — muy idiomático en Python
lat, lon = coordenadas
print(f"Latitud: {lat}, Longitud: {lon}")

r, g, b = rgb_rojo
print(f"Rojo: {r}, Verde: {g}, Azul: {b}")

# Desempaquetado con * — captura el resto
primero, *resto = (1, 2, 3, 4, 5)
print(primero)   # 1
print(resto)     # [2, 3, 4, 5]  ← se convierte en lista

*inicio, ultimo = (1, 2, 3, 4, 5)
print(inicio)    # [1, 2, 3, 4]
print(ultimo)    # 5
```

### ¿Cuándo usar tupla en vez de lista?

```python
# USA TUPLA cuando los datos no deben cambiar:
COLORES_SEMAFORO  = ('rojo', 'amarillo', 'verde')
DIAS_LABORABLES   = ('lunes', 'martes', 'miércoles', 'jueves', 'viernes')
VERSION           = (3, 12, 1)     # major, minor, patch

# También para devolver múltiples valores desde una función
def coordenadas_ciudad(ciudad):
    datos = {
        'Madrid':    (40.4168, -3.7038),
        'Barcelona': (41.3851, 2.1734),
    }
    return datos.get(ciudad, (0.0, 0.0))   # devuelve tupla

lat, lon = coordenadas_ciudad('Madrid')
print(f"{lat:.4f}°N, {lon:.4f}°E")
```

---

## Parte 3 — Sets (`set`)

El **set** (conjunto) almacena elementos únicos sin orden definido.
Equivale al `Set` de JavaScript.

```python
# Crear un set — con llaves
tecnologias = {'Python', 'JavaScript', 'TypeScript', 'Python'}  # Python duplicado
print(tecnologias)   # {'Python', 'JavaScript', 'TypeScript'}  ← sin duplicados

# Desde una lista — forma habitual de eliminar duplicados
ids = [1, 2, 3, 2, 1, 4, 3]
ids_unicos = set(ids)
print(ids_unicos)    # {1, 2, 3, 4}  ← sin orden garantizado

# Set vacío — IMPORTANTE: {} crea un dict, no un set
set_vacio = set()    # ← forma correcta

# Operaciones de conjunto — muy potentes
grupo_a = {'Ana', 'Luis', 'María', 'Carlos'}
grupo_b = {'Luis', 'Carlos', 'Pedro', 'Elena'}

print(grupo_a | grupo_b)    # Unión        — todos los elementos
print(grupo_a & grupo_b)    # Intersección — elementos comunes
print(grupo_a - grupo_b)    # Diferencia   — en A pero no en B
print(grupo_a ^ grupo_b)    # Diferencia simétrica — en uno pero no en ambos
```

```python
# Métodos principales
tags = {'python', 'programacion', 'tutorial'}

tags.add('jest')              # añade un elemento
tags.discard('inexistente')   # elimina si existe — NO lanza error si no existe
tags.remove('tutorial')       # elimina — SÍ lanza KeyError si no existe
print('python' in tags)       # True  — operador in
print(len(tags))              # 2
```

### Ejemplo: eliminar duplicados y operaciones de conjuntos

```python
def usuarios_en_ambos_grupos(grupo1, grupo2):
    """Devuelve los usuarios que están en los dos grupos."""
    return set(grupo1) & set(grupo2)

def usuarios_solo_en_primero(grupo1, grupo2):
    """Devuelve los usuarios que solo están en el primer grupo."""
    return set(grupo1) - set(grupo2)

def eliminar_duplicados(lista):
    """Devuelve la lista sin duplicados (sin garantizar el orden)."""
    return list(set(lista))

# Uso
admins   = ['Ana', 'Luis', 'María']
editores = ['Luis', 'Carlos', 'María']

print(usuarios_en_ambos_grupos(admins, editores))  # {'Luis', 'María'}
print(usuarios_solo_en_primero(admins, editores))  # {'Ana'}
print(eliminar_duplicados([3, 1, 2, 1, 3, 4]))    # [1, 2, 3, 4] (orden puede variar)
```

---

## Parte 4 — Diccionarios (`dict`)

El **diccionario** almacena pares clave-valor.
Es el equivalente más cercano a los objetos/`Map` de JavaScript.

```python
# Crear un diccionario
producto = {
    'nombre':    'Teclado mecánico',
    'precio':    89.99,
    'stock':     15,
    'activo':    True,
}

# Acceder por clave
print(producto['nombre'])       # Teclado mecánico
print(producto['precio'])       # 89.99

# .get() — acceso seguro que no lanza KeyError si la clave no existe
print(producto.get('color'))           # None   — clave inexistente
print(producto.get('color', 'negro'))  # negro  — valor por defecto

# Modificar y añadir
producto['precio'] = 79.99      # modificar valor existente
producto['color']  = 'negro'    # añadir nueva clave

# Eliminar
del producto['color']           # elimina la clave
valor = producto.pop('activo')  # elimina y devuelve el valor
```

### Métodos principales

```python
persona = {'nombre': 'Ana', 'edad': 28, 'ciudad': 'Madrid'}

# Iterar — las tres formas
for clave in persona:                    # itera sobre claves
    print(clave)

for valor in persona.values():           # itera sobre valores
    print(valor)

for clave, valor in persona.items():     # itera sobre pares — la más usada
    print(f"{clave}: {valor}")

# Verificar existencia
print('nombre' in persona)              # True  — comprueba la CLAVE
print('Ana' in persona.values())        # True  — comprueba el VALOR

# Claves, valores y pares como listas
print(list(persona.keys()))     # ['nombre', 'edad', 'ciudad']
print(list(persona.values()))   # ['Ana', 28, 'Madrid']

# update() — combina otro diccionario (sobreescribe si hay claves iguales)
extras = {'email': 'ana@ejemplo.com', 'edad': 29}  # edad se sobreescribirá
persona.update(extras)
print(persona)
# {'nombre': 'Ana', 'edad': 29, 'ciudad': 'Madrid', 'email': 'ana@ejemplo.com'}

# Crear desde dos listas con zip()
claves  = ['nombre', 'edad', 'ciudad']
valores = ['Luis',   35,     'Sevilla']
nuevo   = dict(zip(claves, valores))
print(nuevo)   # {'nombre': 'Luis', 'edad': 35, 'ciudad': 'Sevilla'}
```

### Diccionarios anidados

```python
# Estructura común en APIs y bases de datos
inventario = {
    'A001': {'nombre': 'Teclado',  'precio': 89.99, 'stock': 15},
    'A002': {'nombre': 'Ratón',    'precio': 29.99, 'stock': 40},
    'A003': {'nombre': 'Monitor',  'precio': 299.99,'stock': 5 },
}

# Acceso anidado
print(inventario['A001']['nombre'])    # Teclado
print(inventario['A002']['precio'])    # 29.99

# Iterar sobre el inventario
for codigo, datos in inventario.items():
    print(f"{codigo}: {datos['nombre']} — {datos['precio']}€ (stock: {datos['stock']})")
```

---

## Parte 5 — Ejemplo aplicado: sistema de calificaciones

```python
def calcular_estadisticas(calificaciones):
    """
    Recibe un dict {nombre: [notas...]} y devuelve estadísticas.
    """
    resultado = {}

    for nombre, notas in calificaciones.items():
        promedio = sum(notas) / len(notas)

        resultado[nombre] = {
            'notas':    notas,
            'promedio': round(promedio, 2),
            'maxima':   max(notas),
            'minima':   min(notas),
            'aprobado': promedio >= 5,
        }

    return resultado


def ranking(estadisticas):
    """Devuelve los nombres ordenados de mayor a menor promedio."""
    return sorted(
        estadisticas.keys(),
        key=lambda nombre: estadisticas[nombre]['promedio'],
        reverse=True
    )


# Uso
calificaciones = {
    'Ana':   [8, 7, 9, 10, 8],
    'Luis':  [4, 6, 5, 7,  4],
    'María': [10, 9, 10, 8, 9],
    'Carlos':[3, 4, 2, 5,  3],
}

stats = calcular_estadisticas(calificaciones)
orden = ranking(stats)

print("=== Resultados ===")
for posicion, nombre in enumerate(orden, start=1):
    datos = stats[nombre]
    estado = "APROBADO" if datos['aprobado'] else "SUSPENDIDO"
    print(f"{posicion}. {nombre:<10} Promedio: {datos['promedio']:>5.2f} — {estado}")
```

Salida:

```
=== Resultados ===
1. María      Promedio:  9.20 — APROBADO
2. Ana        Promedio:  8.40 — APROBADO
3. Luis       Promedio:  5.20 — APROBADO
4. Carlos     Promedio:  3.40 — SUSPENDIDO
```

---

## Ejercicios propuestos

1. **Agenda de contactos** — Crea una agenda como diccionario
   `{nombre: {"telefono": ..., "email": ...}}`. Escribe las funciones
   `agregar_contacto(agenda, nombre, telefono, email)`,
   `buscar_contacto(agenda, nombre)` que devuelva el contacto o `None`,
   y `listar_contactos(agenda)` que imprima todos ordenados por nombre.

2. **Estadísticas de ventas** — Tienes una lista de ventas:
   `[{"producto": "Ratón", "precio": 25, "cantidad": 3}, ...]`.
   Calcula el total vendido por producto usando un diccionario acumulador.
   El resultado debe ser `{"Ratón": 75, "Teclado": 180, ...}`.

3. **Eliminar duplicados conservando el orden** — `set()` no conserva el
   orden. Escribe `sin_duplicados(lista)` que devuelva los elementos únicos
   en el mismo orden en que aparecen por primera vez, sin usar `set()`
   (pista: usa un diccionario o comprueba con `in` mientras construyes la lista resultado).

4. **Operaciones de conjuntos** — Dadas dos listas de palabras clave de
   artículos de blog, escribe funciones que usen operadores de sets para:
   encontrar las palabras comunes, las palabras que solo aparecen en el
   primer artículo, y todas las palabras únicas de los dos artículos juntos.

---

## Resumen de la página 3

- **Lista** (`[]`): ordenada, mutable, permite duplicados — equivale al array de JS. Usa `append`, `remove`, `pop`, `sort`, `len`.
- **Tupla** (`()`): ordenada, inmutable — usa para datos que no deben cambiar. Permite desempaquetado `a, b = tupla`.
- **Set** (`{}`): sin orden, sin duplicados — ideal para eliminar duplicados y operaciones de conjuntos (`|`, `&`, `-`, `^`).
- **Dict** (`{clave: valor}`): pares clave-valor, ordenado por inserción (Python 3.7+) — equivale a objetos/Map de JS. Usa `.get(clave, defecto)` para acceso seguro.
- `in` comprueba membresía en todas las colecciones: `'Ana' in lista`, `'nombre' in dict` (comprueba claves).
- `len()` devuelve la longitud de cualquier colección — equivale a `.length` en JS.
- Desempaquetado `a, b, *resto = coleccion` es idiomático en Python para extraer elementos de listas y tuplas.

---

> **Siguiente página →** Página 4: Control de flujo — `if`/`elif`/`else`,
> el operador ternario, `match` (Python 3.10+) y comparadores de Python.
