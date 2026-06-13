# Tutorial Python — Página 4
## Módulo 4 · Control de flujo
### `if`/`elif`/`else`, operador ternario y `match` (Python 3.10+)

---

## `if` / `elif` / `else`

Python usa indentación (4 espacios) para definir los bloques.
No hay llaves `{}`. No hay `else if` — en Python es `elif`.

```python
# JavaScript                    # Python
# if (condición) {              if condición:
#   ...                             ...
# } else if (otra) {            elif otra:
#   ...                             ...
# } else {                      else:
#   ...                             ...
# }

temperatura = 38.5

if temperatura >= 40:
    print("Fiebre muy alta — acudir a urgencias")
elif temperatura >= 38:
    print("Fiebre — tomar medicación y descansar")
elif temperatura >= 37:
    print("Temperatura ligeramente elevada")
else:
    print("Temperatura normal")
```

### Comparadores en Python

```python
# Los mismos que en JS, con una diferencia importante
x = 5

# Igualdad y desigualdad
print(x == 5)    # True
print(x != 3)    # True

# Comparación numérica
print(x > 3)     # True
print(x <= 5)    # True

# DIFERENCIA con JS: Python NO tiene === ni !==
# En Python == siempre compara por valor (no por referencia)
print(1 == True)   # True  ← bool es int en Python
print(0 == False)  # True

# Para comparar identidad (referencia) usa "is"
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True  — mismo contenido
print(a is b)   # False — objetos distintos en memoria
print(a is c)   # True  — misma referencia

# SIEMPRE usa "is" para comparar con None, True o False
valor = None
if valor is None:      # ← forma correcta
    print("Sin valor")
# if valor == None:    # ← funciona, pero no es idiomático
```

### Operadores lógicos: `and`, `or`, `not`

```python
# En JS: && || !
# En Python: and or not  ← se escriben en palabras

edad    = 25
activo  = True
saldo   = 150

# and — ambas condiciones verdaderas
if edad >= 18 and activo:
    print("Usuario mayor de edad y activo")

# or — al menos una condición verdadera
if saldo > 100 or activo:
    print("Puede realizar la operación")

# not — niega la condición
if not activo:
    print("Usuario inactivo")

# Combinado
if edad >= 18 and (saldo > 50 or activo):
    print("Acceso permitido")

# Operadores de membresía e identidad
frutas = ['manzana', 'pera', 'uva']

if 'pera' in frutas:            # "in" — está en la colección
    print("Tenemos peras")

if 'melón' not in frutas:       # "not in" — no está en la colección
    print("No tenemos melón")
```

---

## Parte 1 — Operador ternario

Python tiene una versión del operador ternario pero con sintaxis diferente:

```python
# JavaScript:  condición ? valorSiTrue : valorSiFalse
# Python:       valorSiTrue if condición else valorSiFalse

edad = 20
estado = "mayor de edad" if edad >= 18 else "menor de edad"
print(estado)   # mayor de edad

# Ejemplo con números
puntos = 85
calificacion = "Aprobado" if puntos >= 60 else "Suspenso"

# Anidado — posible pero evita abusar, reduce la legibilidad
nota = 95
nivel = "Sobresaliente" if nota >= 90 else ("Notable" if nota >= 70 else "Aprobado")
```

---

## Parte 2 — Condiciones con colecciones

Python permite usar colecciones directamente en condiciones:

```python
# Verificar si una colección está vacía — forma idiomática
nombres = []

if nombres:                    # True si la lista NO está vacía
    print("Hay nombres")
else:
    print("Lista vacía")       # ← se imprime esto

# Equivalente menos idiomático:
# if len(nombres) > 0:  ← funciona, pero no es "pythónico"

# Con diccionarios
config = {}
if not config:
    print("Configuración no cargada")

# Con strings
mensaje = "  "
if mensaje.strip():            # strip elimina espacios — si queda vacío, es falsy
    print("El mensaje tiene contenido")
else:
    print("Mensaje vacío o solo espacios")
```

---

## Parte 3 — Ejemplo aplicado: clasificador de pedidos

```python
def clasificar_pedido(pedido):
    """
    Clasifica un pedido según su estado, importe y prioridad.
    pedido = {"estado": ..., "importe": ..., "cliente_vip": ...}
    """
    estado        = pedido.get("estado", "desconocido")
    importe       = pedido.get("importe", 0)
    cliente_vip   = pedido.get("cliente_vip", False)

    # Verificar estado primero
    if estado == "cancelado":
        return "❌ Pedido cancelado — no procesar"

    if estado not in ("pendiente", "confirmado", "enviado"):
        return f"⚠️  Estado desconocido: {estado}"

    # Clasificar por importe y tipo de cliente
    if importe <= 0:
        return "⛔ Importe inválido"

    if cliente_vip and importe >= 500:
        prioridad = "URGENTE"
    elif cliente_vip or importe >= 300:
        prioridad = "ALTA"
    elif importe >= 100:
        prioridad = "MEDIA"
    else:
        prioridad = "NORMAL"

    return f"✅ [{prioridad}] Pedido {estado} — {importe:.2f}€"


# Pruebas
pedidos = [
    {"estado": "confirmado", "importe": 750,  "cliente_vip": True},
    {"estado": "pendiente",  "importe": 45,   "cliente_vip": False},
    {"estado": "cancelado",  "importe": 200,  "cliente_vip": False},
    {"estado": "enviado",    "importe": 320,  "cliente_vip": False},
    {"estado": "error",      "importe": 100,  "cliente_vip": False},
]

for p in pedidos:
    print(clasificar_pedido(p))
```

Salida:

```
✅ [URGENTE] Pedido confirmado — 750.00€
✅ [NORMAL] Pedido pendiente — 45.00€
❌ Pedido cancelado — no procesar
✅ [ALTA] Pedido enviado — 320.00€
⚠️  Estado desconocido: error
```

---

## Parte 4 — `match` (Python 3.10+)

`match` es la versión moderna del `switch` de otros lenguajes.
Es más potente: puede comparar por valor, tipo, estructura y condiciones.

```python
# Forma básica — comparar contra valores exactos
def describir_codigo_http(codigo):
    match codigo:
        case 200:
            return "OK — solicitud exitosa"
        case 201:
            return "Created — recurso creado"
        case 301 | 302:               # múltiples valores con |
            return "Redirección"
        case 400:
            return "Bad Request — datos inválidos"
        case 401:
            return "Unauthorized — sin autenticación"
        case 403:
            return "Forbidden — sin permiso"
        case 404:
            return "Not Found — recurso no existe"
        case 500:
            return "Internal Server Error"
        case _:                       # _ es el caso por defecto (como default)
            return f"Código HTTP desconocido: {codigo}"

print(describir_codigo_http(200))    # OK — solicitud exitosa
print(describir_codigo_http(404))    # Not Found — recurso no existe
print(describir_codigo_http(418))    # Código HTTP desconocido: 418
```

### `match` con condiciones (guards)

```python
def clasificar_temperatura(grados):
    match grados:
        case t if t < 0:
            return f"{t}°C — bajo cero, heladas posibles"
        case t if t < 15:
            return f"{t}°C — frío"
        case t if t < 25:
            return f"{t}°C — agradable"
        case t if t < 35:
            return f"{t}°C — caluroso"
        case t:
            return f"{t}°C — muy caluroso"

for temp in [-5, 10, 22, 31, 38]:
    print(clasificar_temperatura(temp))
```

### `match` con estructuras (pattern matching)

Una de las características más potentes: desempaquetar y verificar
la estructura de objetos o listas al mismo tiempo.

```python
def procesar_comando(comando):
    """
    comando puede ser:
    - ("salir",)
    - ("mover", dirección)
    - ("atacar", objetivo, daño)
    - ("inventario", "mostrar") o ("inventario", "vaciar")
    """
    match comando:
        case ("salir",):
            return "Cerrando el juego..."

        case ("mover", direccion) if direccion in ("norte", "sur", "este", "oeste"):
            return f"Moviéndote hacia el {direccion}"

        case ("mover", direccion):
            return f"Dirección '{direccion}' no válida"

        case ("atacar", objetivo, dano) if dano > 0:
            return f"Atacas a {objetivo} causando {dano} de daño"

        case ("inventario", "mostrar"):
            return "Mostrando inventario..."

        case ("inventario", "vaciar"):
            return "Inventario vaciado"

        case _:
            return f"Comando desconocido: {comando}"


comandos = [
    ("mover",     "norte"),
    ("atacar",    "dragón", 50),
    ("inventario","mostrar"),
    ("salir",),
    ("volar",),
]

for cmd in comandos:
    print(procesar_comando(cmd))
```

Salida:

```
Moviéndote hacia el norte
Atacas a dragón causando 50 de daño
Mostrando inventario...
Cerrando el juego...
Comando desconocido: ('volar',)
```

---

## Parte 5 — Ejemplo completo: validador de formulario

```python
def validar_formulario(datos):
    """
    Valida los datos de un formulario de registro.
    Devuelve {"valido": bool, "errores": [mensajes]}
    """
    errores = []

    # Validar nombre
    nombre = datos.get("nombre", "").strip()
    if not nombre:
        errores.append("El nombre es obligatorio")
    elif len(nombre) < 2:
        errores.append("El nombre debe tener al menos 2 caracteres")

    # Validar email
    email = datos.get("email", "").strip().lower()
    if not email:
        errores.append("El email es obligatorio")
    elif "@" not in email or "." not in email.split("@")[-1]:
        errores.append("El email no tiene un formato válido")

    # Validar edad
    edad = datos.get("edad")
    if edad is None:
        errores.append("La edad es obligatoria")
    elif not isinstance(edad, int):
        errores.append("La edad debe ser un número entero")
    elif edad < 18:
        errores.append("Debes ser mayor de edad para registrarte")
    elif edad > 120:
        errores.append("La edad introducida no es válida")

    # Validar contraseña
    pwd = datos.get("contrasena", "")
    if not pwd:
        errores.append("La contraseña es obligatoria")
    elif len(pwd) < 8:
        errores.append("La contraseña debe tener al menos 8 caracteres")
    elif pwd.isdigit():
        errores.append("La contraseña no puede ser solo números")

    return {"valido": len(errores) == 0, "errores": errores}


# Casos de prueba
casos = [
    {"nombre": "Ana García", "email": "ana@ej.com", "edad": 25, "contrasena": "Segura123"},
    {"nombre": "",            "email": "sinArroba",  "edad": 15, "contrasena": "abc"},
    {"nombre": "Luis",        "email": "luis@ok.com","edad": 30, "contrasena": "12345678"},
]

for caso in casos:
    resultado = validar_formulario(caso)
    if resultado["valido"]:
        print(f"✅ {caso.get('nombre') or '(sin nombre)'}: formulario válido")
    else:
        print(f"❌ {caso.get('nombre') or '(sin nombre)'}:")
        for error in resultado["errores"]:
            print(f"   · {error}")
```

---

## Ejercicios propuestos

1. **Semáforo de batería** — Escribe `estado_bateria(porcentaje, cargando)`
   que use `if/elif/else` para devolver: `"Crítica"` si < 10%,
   `"Baja"` si < 30%, `"Media"` si < 60%, `"Alta"` si < 90%, `"Completa"` si >= 90%.
   Si `cargando` es `True`, añade `" (cargando)"` al final con el operador ternario.

2. **Clasificador de triángulos** — Escribe `tipo_triangulo(a, b, c)`
   que devuelva `"equilátero"`, `"isósceles"`, `"escaleno"` o
   `"no es un triángulo"` (cuando los lados no cumplen la desigualdad triangular:
   la suma de dos lados siempre debe ser mayor que el tercero).

3. **match para días** — Usa `match` para escribir `tipo_dia(nombre)`
   que devuelva `"laboral"`, `"fin de semana"` o `"desconocido"`.
   Maneja mayúsculas/minúsculas convirtiendo a minúsculas antes del `match`.
   Agrupa lunes-viernes con `|` y sábado-domingo con otro `|`.

4. **Calculadora con validación** — Escribe `calcular(a, operador, b)`
   que use `match` para los operadores `+`, `-`, `*`, `/`, `**` y `%`.
   Valida con `if` antes del `match` que `a` y `b` son números.
   Para `/` y `%`, lanza `ValueError` si `b == 0`.

---

## Resumen de la página 4

- `if`/`elif`/`else` usa **indentación** en lugar de llaves. No existe `else if` — en Python es `elif`.
- Los operadores lógicos se escriben en palabras: `and`, `or`, `not` — en JS son `&&`, `||`, `!`.
- Para comparar con `None`, `True` o `False` usa `is` en lugar de `==`: `if valor is None`.
- El operador ternario invierte el orden: `valor_si_true if condición else valor_si_false`.
- Las colecciones vacías son falsy: `if lista:` es equivalente a `if len(lista) > 0:`.
- `in` y `not in` comprueban membresía en cualquier colección: `if "x" in lista`.
- `match` (Python 3.10+) es más potente que el `switch` de JS: admite guards (`if condición`), múltiples valores (`case a | b`) y desempaquetado de estructuras.

---

> **Siguiente página →** Página 5: Bucles — `for`, `while`, `range`,
> `enumerate`, `zip` y las poderosas list comprehensions de Python.
