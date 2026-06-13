# Tutorial Python — Página 2
## Módulo 2 · Strings
### f-strings, slicing, métodos y comparativa con JavaScript

---

## Strings en Python

Los strings de Python son muy similares a los de JavaScript pero con
varias ventajas adicionales: el slicing es mucho más potente,
los f-strings son más limpios que los template literals, y hay
docenas de métodos útiles integrados.

```python
# Comillas simples o dobles — equivalentes (elige una y sé consistente)
nombre   = "Ana García"
apellido = 'López'

# String multilínea con triple comillas — equivale a los backticks en JS
descripcion = """
Este producto tiene:
- Garantía de 2 años
- Envío gratuito
- Devolución en 30 días
"""

# En Python los strings son inmutables (igual que en JS)
texto = "hola"
# texto[0] = "H"  ← ERROR — no puedes modificar caracteres individuales
texto = "Hola"    # solo puedes reasignar la variable entera
```

---

## Parte 1 — f-strings (la forma recomendada)

Los **f-strings** son la forma moderna de interpolar variables en Python 3.6+.
Son más limpios y rápidos que el operador `%` o el método `.format()`.

```python
# JavaScript                    # Python f-string
# `Hola, ${nombre}`             f"Hola, {nombre}"
# `Tienes ${edad} años`         f"Tienes {edad} años"
# `${precio.toFixed(2)}`        f"{precio:.2f}"

nombre = "Ana García"
edad   = 28
precio = 89.99

# f-string básico — la f va antes de las comillas
print(f"Hola, {nombre}")                       # Hola, Ana García

# Expresiones dentro de las llaves
print(f"En 5 años tendrás {edad + 5} años")   # En 5 años tendrás 33 años

# Llamadas a métodos
print(f"Nombre en mayúsculas: {nombre.upper()}")  # NOMBRE EN MAYÚSCULAS

# Formato numérico — especificadores de formato
precio_iva = precio * 1.21
print(f"Precio con IVA: {precio_iva:.2f}€")       # Precio con IVA: 108.89€
print(f"Número grande: {8_000_000:,}")             # Número grande: 8,000,000
print(f"Porcentaje: {0.857:.1%}")                  # Porcentaje: 85.7%
print(f"Rellenar con ceros: {42:05d}")             # Rellenar con ceros: 00042

# Alineación de texto
nombre1, nombre2 = "Ana", "Maximiliano"
print(f"{nombre1:<15} | {28}")   # Ana             | 28
print(f"{nombre2:<15} | {35}")   # Maximiliano     | 35

# Debug con = — muy útil para depurar (Python 3.8+)
x = 42
print(f"{x=}")          # x=42  ← imprime el nombre de la variable y su valor
print(f"{x*2=}")        # x*2=84
```

---

## Parte 2 — Concatenación y repetición

```python
# Concatenación con + (igual que en JS)
saludo = "Hola" + ", " + "mundo"    # "Hola, mundo"

# Repetición con * — no existe en JS
linea      = "-" * 40               # "----------------------------------------"
separador  = "=-" * 20              # "=-=-=-=-=-=-=-=-=-=-=-=-= ... "
print(linea)

# En Python NO puedes concatenar strings con números directamente
# nombre = "Edad: " + 28   # ← TypeError
nombre = "Edad: " + str(28)         # hay que convertir
# O mejor, usa f-strings:
nombre = f"Edad: {28}"              # más limpio
```

---

## Parte 3 — Slicing (rebanado)

El slicing es una de las características más poderosas de Python.
Permite extraer partes de un string con la notación `[inicio:fin:paso]`.

```python
texto = "Python es genial"
#        0123456789...

# Índice positivo — desde el inicio
print(texto[0])       # P  ← primer carácter
print(texto[7])       # e

# Índice negativo — desde el final
print(texto[-1])      # l  ← último carácter
print(texto[-6])      # g

# Slicing: texto[inicio:fin]  (fin NO se incluye)
print(texto[0:6])     # Python        ← del índice 0 al 5
print(texto[7:9])     # es
print(texto[10:])     # genial        ← hasta el final
print(texto[:6])      # Python        ← desde el inicio

# Slicing con paso: texto[inicio:fin:paso]
print(texto[::2])     # Pto sgnl      ← cada 2 caracteres
print(texto[::-1])    # laineg se nohtyP  ← invertir el string (¡muy útil!)

# Comparativa con JavaScript
# JS:     texto.substring(0, 6)    → "Python"
# Python: texto[0:6]               → "Python"

# JS:     texto.slice(-6)          → "genial"
# Python: texto[-6:]               → "genial"
```

### Ejemplo aplicado — procesar códigos de producto

```python
def analizar_codigo(codigo):
    """
    Códigos con formato: "CAT-SUBCOD-AAAAMMDD"
    Ejemplo:             "ELE-TEC001-20240315"
    """
    categoria   = codigo[:3]          # "ELE"
    subcodigo   = codigo[4:10]        # "TEC001"
    fecha_raw   = codigo[11:]         # "20240315"

    año  = fecha_raw[:4]             # "2024"
    mes  = fecha_raw[4:6]            # "03"
    dia  = fecha_raw[6:]             # "15"

    return {
        "categoria":  categoria,
        "subcodigo":  subcodigo,
        "fecha":      f"{dia}/{mes}/{año}",
    }

info = analizar_codigo("ELE-TEC001-20240315")
print(info)
# {'categoria': 'ELE', 'subcodigo': 'TEC001', 'fecha': '15/03/2024'}
```

---

## Parte 4 — Métodos de string más importantes

```python
texto = "  Hola, Mundo!  "

# ── Limpieza ─────────────────────────────────────────────────────
print(texto.strip())           # "Hola, Mundo!"   — elimina espacios de ambos lados
print(texto.lstrip())          # "Hola, Mundo!  " — solo izquierda (left)
print(texto.rstrip())          # "  Hola, Mundo!" — solo derecha (right)

# ── Búsqueda ─────────────────────────────────────────────────────
frase = "Python es poderoso y Python es sencillo"
print(frase.find("Python"))    # 0   — primera ocurrencia (índice)
print(frase.rfind("Python"))   # 22  — última ocurrencia
print(frase.count("Python"))   # 2   — cuántas veces aparece
print("Python" in frase)       # True ← el operador "in" — muy idiomático
print("Java" not in frase)     # True

# ── Transformación ───────────────────────────────────────────────
print("hola".upper())          # HOLA
print("HOLA".lower())          # hola
print("ana garcía".title())    # Ana García  ← capitaliza cada palabra
print("hola MUNDO".swapcase()) # HOLA mundo

# ── Verificación (devuelven True/False) ──────────────────────────
print("123".isdigit())         # True  — solo contiene dígitos
print("abc".isalpha())         # True  — solo letras
print("abc123".isalnum())      # True  — letras o dígitos
print("  ".isspace())          # True  — solo espacios
print("Hola".startswith("Ho")) # True
print("Hola".endswith("la"))   # True

# ── División y unión ─────────────────────────────────────────────
# split() — divide un string en lista (equivale a split() de JS)
csv = "Ana,Luis,María,Carlos"
nombres = csv.split(",")
print(nombres)                 # ['Ana', 'Luis', 'María', 'Carlos']

lineas = "línea 1\nlínea 2\nlínea 3".split("\n")
print(lineas)                  # ['línea 1', 'línea 2', 'línea 3']

# split() sin argumento divide por cualquier espacio en blanco
palabras = "  hola   mundo  ".split()
print(palabras)                # ['hola', 'mundo']  — elimina espacios extra

# join() — une una lista en string con un separador (¡es el inverso de split!)
# En JS: lista.join(",")
# En Python: ",".join(lista) ← el separador llama a join
print(", ".join(nombres))      # Ana, Luis, María, Carlos
print(" | ".join(nombres))     # Ana | Luis | María | Carlos
print("".join(["H","o","l","a"]))   # Hola

# ── Reemplazo ────────────────────────────────────────────────────
url = "https://www.ejemplo.com/pagina"
print(url.replace("www.", ""))         # https://ejemplo.com/pagina
print(url.replace("e", "E", 1))       # rEemplaza solo la primera "e"

# ── Relleno ──────────────────────────────────────────────────────
print("42".zfill(5))          # 00042 — rellena con ceros por la izquierda
print("Hola".ljust(10, "-"))  # Hola------
print("Hola".rjust(10, "-"))  # ------Hola
print("Hola".center(10, "-")) # ---Hola---
```

---

## Parte 5 — Strings multilínea y raw strings

```python
# String multilínea — con triple comillas
sql = """
    SELECT nombre, email
    FROM usuarios
    WHERE activo = TRUE
    ORDER BY nombre
"""

html = '''
<div class="tarjeta">
    <h2>Título</h2>
    <p>Contenido aquí</p>
</div>
'''

# Raw string — ignora los caracteres de escape (la r va antes de las comillas)
# En JS: String.raw`C:\Users\Ana`
# Útil para rutas de Windows y expresiones regulares
ruta_windows = r"C:\Users\Ana\Documents"   # sin r sería un error
patron_regex  = r"\d{4}-\d{2}-\d{2}"      # patrón de fecha

print(ruta_windows)   # C:\Users\Ana\Documents  (sin interpretar \U \A \D)
print("\n")           # salto de línea — el \n SÍ se interpreta sin r
print(r"\n")          # \n           — con r NO se interpreta
```

---

## Parte 6 — Caracteres especiales

```python
# Secuencias de escape (igual que en JS)
print("Hola\nMundo")     # salto de línea
print("Col1\tCol2")      # tabulación
print("Dice \"hola\"")   # comillas dentro de comillas dobles
print('Dice \'hola\'')   # comillas dentro de comillas simples
print("C:\\ruta\\archivo") # barra invertida

# Alternativa — mezcla de comillas para evitar el escape
print('Dice "hola" con comillas dobles')
print("Dice 'hola' con comillas simples")
```

---

## Parte 7 — Ejemplo aplicado: formatear una factura

```python
def formatear_factura(pedido):
    lineas = []
    lineas.append("=" * 45)
    lineas.append(f"{'FACTURA':^45}")    # centrado en 45 chars
    lineas.append("=" * 45)
    lineas.append(f"Cliente: {pedido['cliente']}")
    lineas.append(f"Fecha:   {pedido['fecha']}")
    lineas.append("-" * 45)
    lineas.append(f"{'Producto':<20} {'Cant':>5} {'Precio':>8} {'Total':>10}")
    lineas.append("-" * 45)

    subtotal = 0
    for item in pedido["items"]:
        total_item = item["precio"] * item["cantidad"]
        subtotal += total_item
        lineas.append(
            f"{item['nombre']:<20} {item['cantidad']:>5} "
            f"{item['precio']:>8.2f} {total_item:>10.2f}"
        )

    iva = subtotal * 0.21
    total = subtotal + iva

    lineas.append("-" * 45)
    lineas.append(f"{'Subtotal':>34} {subtotal:>10.2f}")
    lineas.append(f"{'IVA (21%)':>34} {iva:>10.2f}")
    lineas.append("=" * 45)
    lineas.append(f"{'TOTAL':>34} {total:>10.2f}")
    lineas.append("=" * 45)

    return "\n".join(lineas)


pedido = {
    "cliente": "Ana García",
    "fecha":   "15/03/2024",
    "items": [
        {"nombre": "Teclado mecánico", "cantidad": 1, "precio": 89.99},
        {"nombre": "Ratón inalámbrico","cantidad": 2, "precio": 29.99},
        {"nombre": "Alfombrilla",      "cantidad": 1, "precio": 12.50},
    ],
}

print(formatear_factura(pedido))
```

Salida:

```
=============================================
                   FACTURA
=============================================
Cliente: Ana García
Fecha:   15/03/2024
---------------------------------------------
Producto              Cant   Precio      Total
---------------------------------------------
Teclado mecánico         1    89.99      89.99
Ratón inalámbrico        2    29.99      59.98
Alfombrilla              1    12.50      12.50
---------------------------------------------
                         Subtotal     162.47
                         IVA (21%)     34.12
=============================================
                            TOTAL     196.59
=============================================
```

---

## Ejercicios propuestos

1. **Formatear DNI** — Escribe `formatear_dni(dni)` que reciba un string
   como `"12345678A"` o `"12345678 A"` (con o sin espacio) y lo devuelva
   siempre en el formato `"12345678-A"` en mayúsculas. Usa `strip()`,
   `replace()` y slicing.

2. **Invertir palabras** — Escribe `invertir_palabras(frase)` que invierta
   el orden de las palabras de una frase usando `split()`, slicing (`[::-1]`)
   y `join()`. `"hola mundo cruel"` → `"cruel mundo hola"`. No uses `reversed()`.

3. **Censurar email** — Escribe `censurar_email(email)` que muestre solo
   los 2 primeros caracteres del usuario y el dominio completo:
   `"ana.garcia@gmail.com"` → `"an**@gmail.com"`. Usa `find()` y slicing.

4. **Tabla de precios** — Dado una lista de productos `[{"nombre": ..., "precio": ...}]`,
   genera una tabla formateada con f-strings donde el nombre esté alineado
   a la izquierda en 25 caracteres y el precio a la derecha con 2 decimales
   y el símbolo €. Añade una línea de total al final.

---

## Resumen de la página 2

- Los f-strings (`f"texto {variable}"`) son la forma recomendada de interpolar valores en strings. Admiten expresiones completas, llamadas a métodos y especificadores de formato.
- Los especificadores de formato más útiles: `:.2f` (2 decimales), `:,` (separador de miles), `:.1%` (porcentaje), `:>10` (alineado a la derecha en 10 chars), `:<10` (izquierda), `:^10` (centrado).
- El slicing `texto[inicio:fin:paso]` es más potente que `substring()` en JS. `texto[::-1]` invierte el string.
- `in` es el operador para comprobar si un substring está en otro: `"hola" in frase`.
- `split(sep)` divide un string en lista. `sep.join(lista)` une una lista en string — en Python el separador es el que llama a `join`, no la lista.
- Los raw strings con `r"..."` ignoran los caracteres de escape — imprescindibles para rutas de Windows y expresiones regulares.
- Los strings son inmutables: no puedes modificar un carácter con `texto[0] = "X"`. Debes crear un string nuevo.

---

> **Siguiente página →** Página 3: Colecciones — listas, tuplas, sets
> y diccionarios, con sus métodos más importantes y sus equivalentes
> en JavaScript.
