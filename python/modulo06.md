# Tutorial Python — Página 6
## Módulo 6 · Funciones
### `def`, parámetros nombrados, `*args`, `**kwargs` y lambdas

---

## Definir funciones con `def`

```python
# JavaScript                    # Python
# function saludar(nombre) {    def saludar(nombre):
#   return `Hola, ${nombre}`        return f"Hola, {nombre}"
# }

def saludar(nombre):
    return f"Hola, {nombre}"

print(saludar("Ana"))    # Hola, Ana
```

Las diferencias principales con JS:
- Se usa la palabra clave `def` (no `function`)
- El cuerpo va indentado, sin llaves
- No hay `return` implícito — una función sin `return` devuelve `None`

---

## Parte 1 — Parámetros y valores por defecto

```python
# Parámetros con valores por defecto — igual que en JS
def crear_usuario(nombre, rol="visitante", activo=True):
    return {"nombre": nombre, "rol": rol, "activo": activo}

# Llamadas posibles
print(crear_usuario("Ana"))                          # usa los defaults
print(crear_usuario("Luis", "admin"))                # sobreescribe rol
print(crear_usuario("María", "editor", False))       # sobreescribe ambos

# REGLA: los parámetros con default van siempre al final
# def funcion(obligatorio, opcional=valor) ← correcto
# def funcion(opcional=valor, obligatorio) ← SyntaxError
```

### Parámetros nombrados (keyword arguments)

Una de las características más útiles de Python: puedes especificar
los argumentos por nombre en cualquier orden.

```python
def enviar_email(destinatario, asunto, cuerpo, cc=None, html=False):
    modo = "HTML" if html else "texto plano"
    print(f"Para: {destinatario}")
    print(f"Asunto: {asunto}")
    print(f"CC: {cc or 'ninguno'}")
    print(f"Modo: {modo}")
    print(f"Cuerpo: {cuerpo[:50]}...")

# Llamada posicional — el orden importa
enviar_email("ana@ej.com", "Bienvenida", "Hola Ana...")

# Llamada con kwargs — el orden no importa
enviar_email(
    destinatario = "luis@ej.com",
    cuerpo       = "Confirme su cuenta...",
    asunto       = "Confirme su email",
    html         = True,
)

# Muy legible cuando hay muchos parámetros opcionales
enviar_email("maria@ej.com", "Hola", "Texto del mensaje", cc="jefe@ej.com")
```

---

## Parte 2 — Múltiples valores de retorno

En Python puedes devolver múltiples valores separados por coma.
En realidad devuelve una tupla, pero el desempaquetado hace que
parezca que devuelve varios valores.

```python
def estadisticas(numeros):
    """Devuelve mínimo, máximo y promedio de una lista."""
    if not numeros:
        return None, None, None
    return min(numeros), max(numeros), sum(numeros) / len(numeros)

# Desempaquetar el resultado
minimo, maximo, promedio = estadisticas([3, 7, 2, 8, 5])
print(f"Mín: {minimo}, Máx: {maximo}, Prom: {promedio:.2f}")
# Mín: 2, Máx: 8, Prom: 5.00

# También puedes ignorar valores con _
_, maximo, _ = estadisticas([3, 7, 2, 8, 5])
print(f"El máximo es: {maximo}")

# O recoger toda la tupla
resultado = estadisticas([1, 2, 3])
print(type(resultado))    # <class 'tuple'>
```

---

## Parte 3 — `*args` — número variable de argumentos posicionales

`*args` captura todos los argumentos posicionales extra en una **tupla**.
Equivale al operador rest `...args` de JavaScript.

```python
# JavaScript: function sumar(...numeros) { ... }
# Python:     def sumar(*numeros): ...

def sumar(*numeros):
    """Suma cualquier cantidad de números."""
    return sum(numeros)

print(sumar(1, 2))              # 3
print(sumar(1, 2, 3, 4, 5))    # 15
print(sumar())                  # 0

# Combinado con parámetros normales — *args siempre va al final
def presentar(saludo, *nombres):
    for nombre in nombres:
        print(f"{saludo}, {nombre}!")

presentar("Hola", "Ana", "Luis", "María")
# Hola, Ana!
# Hola, Luis!
# Hola, María!

# El operador * también "desempaqueta" una lista al llamar la función
numeros = [1, 2, 3, 4]
print(sumar(*numeros))    # equivale a sumar(1, 2, 3, 4)
```

---

## Parte 4 — `**kwargs` — número variable de argumentos nombrados

`**kwargs` captura todos los keyword arguments extra en un **diccionario**.

```python
def crear_perfil(nombre, **opciones):
    """Crea un perfil con campos adicionales opcionales."""
    perfil = {"nombre": nombre}
    perfil.update(opciones)   # añade todos los kwargs al diccionario
    return perfil

# Puedes pasar cualquier combinación de argumentos nombrados
p1 = crear_perfil("Ana", edad=28, ciudad="Madrid")
p2 = crear_perfil("Luis", edad=35, ciudad="Barcelona", premium=True)
p3 = crear_perfil("María")     # solo el nombre — kwargs vacío

print(p1)   # {'nombre': 'Ana', 'edad': 28, 'ciudad': 'Madrid'}
print(p2)   # {'nombre': 'Luis', 'edad': 35, 'ciudad': 'Barcelona', 'premium': True}
print(p3)   # {'nombre': 'María'}

# El operador ** también desempaqueta un diccionario al llamar la función
datos = {"edad": 30, "ciudad": "Sevilla"}
p4 = crear_perfil("Carlos", **datos)
print(p4)   # {'nombre': 'Carlos', 'edad': 30, 'ciudad': 'Sevilla'}
```

### Combinación completa de parámetros

```python
# El orden correcto de los parámetros en Python:
# def fn(posicionales, *args, keyword_only, **kwargs)

def log(nivel, *mensajes, separador="\n", **metadatos):
    """
    nivel:      string obligatorio
    *mensajes:  uno o más mensajes
    separador:  keyword-only con default (va después de *args)
    **metadatos: datos extra opcionales
    """
    cabecera = f"[{nivel.upper()}]"
    if metadatos:
        extras = " | ".join(f"{k}={v}" for k, v in metadatos.items())
        cabecera += f" ({extras})"
    texto = separador.join(str(m) for m in mensajes)
    print(f"{cabecera} {texto}")


log("info", "Servidor iniciado", "Puerto: 8080")
log("error", "Conexión fallida", usuario="Ana", intentos=3)
log("warn", "Memoria al 85%", separador=" — ", servidor="web-01")
```

---

## Parte 5 — Funciones lambda

Las **lambdas** son funciones anónimas de una línea.
Equivalen a las arrow functions `=>` de JavaScript.

```python
# JavaScript: (x) => x * 2
# Python:     lambda x: x * 2

doble = lambda x: x * 2
print(doble(5))    # 10

# Lambda con dos parámetros
sumar = lambda a, b: a + b
print(sumar(3, 4))    # 7

# Su uso más común: como argumento de sorted(), min(), max(), filter(), map()
productos = [
    {"nombre": "Monitor",  "precio": 280},
    {"nombre": "Ratón",    "precio": 25},
    {"nombre": "Teclado",  "precio": 50},
]

# sorted() con key — ordenar por precio
por_precio = sorted(productos, key=lambda p: p["precio"])
print([p["nombre"] for p in por_precio])    # ['Ratón', 'Teclado', 'Monitor']

# Orden descendente
por_precio_desc = sorted(productos, key=lambda p: p["precio"], reverse=True)
print([p["nombre"] for p in por_precio_desc])    # ['Monitor', 'Teclado', 'Ratón']

# min() y max() con key
mas_barato = min(productos, key=lambda p: p["precio"])
print(mas_barato["nombre"])    # Ratón
```

### `map()` y `filter()` con lambdas

```python
numeros = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map — transforma cada elemento (equivale a .map() de JS)
cuadrados = list(map(lambda n: n ** 2, numeros))
print(cuadrados)    # [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# filter — filtra elementos (equivale a .filter() de JS)
pares = list(filter(lambda n: n % 2 == 0, numeros))
print(pares)    # [2, 4, 6, 8, 10]

# En Python moderno se prefieren las comprehensions sobre map/filter:
cuadrados2 = [n**2 for n in numeros]        # ← más legible
pares2     = [n for n in numeros if n % 2 == 0]  # ← más legible
```

---

## Parte 6 — Funciones de primera clase

Las funciones en Python son objetos de primera clase: puedes pasarlas
como argumentos, devolverlas desde otras funciones y almacenarlas en variables.

```python
def aplicar(funcion, valor):
    """Aplica cualquier función a un valor."""
    return funcion(valor)

def doblar(n):
    return n * 2

def al_cuadrado(n):
    return n ** 2

print(aplicar(doblar,      5))    # 10
print(aplicar(al_cuadrado, 5))    # 25
print(aplicar(str,         42))   # "42"  ← str es una función también


# Funciones que devuelven funciones — closures
def crear_multiplicador(factor):
    """Devuelve una función que multiplica por factor."""
    def multiplicar(n):
        return n * factor
    return multiplicar

por_dos  = crear_multiplicador(2)
por_tres = crear_multiplicador(3)
por_diez = crear_multiplicador(10)

print(por_dos(5))    # 10
print(por_tres(5))   # 15
print(por_diez(5))   # 50
```

---

## Parte 7 — Type hints (anotaciones de tipo)

Python 3.5+ permite anotar los tipos de los parámetros y el retorno.
No son obligatorias, pero mejoran la legibilidad y el autocompletado.

```python
# Sin type hints
def calcular_descuento(precio, porcentaje):
    return precio * (1 - porcentaje / 100)

# Con type hints — mucho más claro
def calcular_descuento(precio: float, porcentaje: float) -> float:
    return precio * (1 - porcentaje / 100)

# Tipos comunes
def procesar(
    nombre:   str,
    edad:     int,
    activo:   bool = True,
    etiquetas: list[str] = None,
) -> dict:
    return {"nombre": nombre, "edad": edad, "activo": activo}

# Importar tipos más avanzados
from typing import Optional

def buscar(id: int) -> Optional[dict]:
    """Devuelve un dict si encuentra el elemento, None si no."""
    ...
```

---

## Parte 8 — Ejemplo completo: pipeline de procesamiento

```python
def limpiar_nombre(nombre: str) -> str:
    """Elimina espacios y convierte a Title Case."""
    return nombre.strip().title()

def validar_email(email: str) -> bool:
    """Validación básica de formato email."""
    email = email.strip().lower()
    return "@" in email and "." in email.split("@")[-1]

def calcular_descuento(precio: float, porcentaje: float) -> float:
    """Aplica un descuento y devuelve el precio final."""
    if not 0 <= porcentaje <= 100:
        raise ValueError(f"Porcentaje inválido: {porcentaje}")
    return round(precio * (1 - porcentaje / 100), 2)

def procesar_pedidos(pedidos: list, tasa_descuento: float = 0) -> dict:
    """
    Procesa una lista de pedidos y devuelve un resumen.
    Cada pedido: {"cliente": str, "email": str, "importe": float}
    """
    validos   = []
    invalidos = []

    for pedido in pedidos:
        nombre = limpiar_nombre(pedido.get("cliente", ""))
        email  = pedido.get("email", "")
        importe = pedido.get("importe", 0)

        if not nombre or not validar_email(email) or importe <= 0:
            invalidos.append(pedido)
            continue

        importe_final = calcular_descuento(importe, tasa_descuento)
        validos.append({
            "cliente": nombre,
            "email":   email.strip().lower(),
            "importe": importe_final,
        })

    total = sum(p["importe"] for p in validos)

    return {
        "procesados": len(validos),
        "rechazados": len(invalidos),
        "total":      round(total, 2),
        "pedidos":    validos,
    }


pedidos = [
    {"cliente": "  ana garcia  ", "email": "ANA@EJ.COM",  "importe": 150.00},
    {"cliente": "Luis",           "email": "sinArroba",   "importe": 80.00},
    {"cliente": "MARÍA LÓPEZ",    "email": "maria@ej.com","importe": 200.00},
    {"cliente": "",               "email": "x@x.com",     "importe": 50.00},
]

resultado = procesar_pedidos(pedidos, tasa_descuento=10)
print(f"Procesados: {resultado['procesados']}")
print(f"Rechazados: {resultado['rechazados']}")
print(f"Total:      {resultado['total']}€")
for p in resultado["pedidos"]:
    print(f"  {p['cliente']:<20} {p['importe']:>8.2f}€")
```

---

## Ejercicios propuestos

1. **Calculadora flexible** — Escribe `calcular(operacion, *numeros)` donde
   `operacion` es un string (`"suma"`, `"producto"`, `"maximo"`, `"minimo"`)
   y `*numeros` acepta cualquier cantidad de números. Usa un diccionario
   de funciones `{nombre: lambda}` para seleccionar la operación.

2. **Formateador configurable** — Escribe `formatear_tabla(datos, **opciones)`
   donde `datos` es una lista de dicts y `**opciones` puede incluir
   `separador` (default `" | "`), `mayusculas` (default `False`) y
   `max_filas` (default `None`). Aplica cada opción en la función.

3. **Clausuras** — Escribe `crear_validador(minimo, maximo)` que devuelva
   una función que recibe un número y devuelve `True` si está en el rango.
   Crea tres validadores: para edades (0-120), para porcentajes (0-100)
   y para temperaturas (-60 a 60). Pruébalos con `filter()` sobre una lista.

4. **Decorador manual** — Sin usar `@`, escribe una función
   `con_log(funcion)` que reciba una función y devuelva una nueva función
   que imprime `"Llamando a [nombre]"` antes de ejecutarla y
   `"[nombre] terminó"` después. Envuelve `calcular_descuento` con ella
   y verifica que los mensajes aparecen correctamente.

---

## Resumen de la página 6

- `def nombre(params):` define una función. Sin `return` explícito devuelve `None`.
- Los parámetros con valor por defecto deben ir **al final**: `def fn(req, opt=valor)`.
- Los **keyword arguments** permiten pasar argumentos por nombre en cualquier orden — mejoran la legibilidad con funciones de muchos parámetros.
- Las funciones pueden devolver **múltiples valores** separados por coma — en realidad devuelven una tupla que se desempaqueta automáticamente.
- `*args` captura argumentos posicionales extra en una tupla. `**kwargs` captura keyword arguments extra en un diccionario.
- `*` y `**` también sirven para **desempaquetar** listas y diccionarios al llamar funciones.
- Las **lambdas** son funciones anónimas de una línea — su uso principal es como argumento de `sorted()`, `min()`, `max()`, `map()` y `filter()`.
- Los **type hints** (`param: tipo -> retorno`) no son obligatorios pero mejoran la legibilidad y el soporte del editor.

---

> **Siguiente página →** Página 7: Clases y programación orientada a objetos —
> `class`, `__init__`, herencia, `@property` y métodos especiales.
