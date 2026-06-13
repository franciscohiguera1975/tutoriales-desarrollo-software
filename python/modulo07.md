# Tutorial Python — Página 7
## Módulo 7 · Clases y POO
### `class`, `__init__`, herencia, `@property` y métodos especiales

---

## Clases en Python vs JavaScript

```
JavaScript (ES6)                Python
──────────────────────────────  ──────────────────────────────
class Persona {                 class Persona:
  constructor(nombre, edad) {       def __init__(self, nombre, edad):
    this.nombre = nombre;               self.nombre = nombre
    this.edad   = edad;                 self.edad   = edad
  }
  saludar() {                       def saludar(self):
    return `Hola, ${this.nombre}`       return f"Hola, {self.nombre}"
  }
}                               (sin llave de cierre)
```

Las diferencias principales:
- `__init__` en lugar de `constructor`
- `self` en lugar de `this` — y hay que declararlo en cada método
- No hay `public`/`private`/`protected` — se usan convenciones de nombres
- La herencia usa paréntesis: `class Hijo(Padre)`

---

## Parte 1 — Clase básica

```python
class Producto:
    """Representa un producto del catálogo."""

    def __init__(self, nombre: str, precio: float, stock: int = 0):
        self.nombre = nombre
        self.precio = precio
        self.stock  = stock

    def esta_disponible(self) -> bool:
        return self.stock > 0

    def aplicar_descuento(self, porcentaje: float) -> None:
        """Modifica el precio aplicando un descuento."""
        if not 0 < porcentaje < 100:
            raise ValueError("El porcentaje debe estar entre 0 y 100")
        self.precio = round(self.precio * (1 - porcentaje / 100), 2)

    def __str__(self) -> str:
        """Lo que imprime print(producto)."""
        estado = "✓ disponible" if self.esta_disponible() else "✗ agotado"
        return f"{self.nombre} — {self.precio:.2f}€ ({estado})"

    def __repr__(self) -> str:
        """Representación técnica — usada en el REPL y en listas."""
        return f"Producto('{self.nombre}', {self.precio}, {self.stock})"


# Crear instancias
teclado = Producto("Teclado mecánico", 89.99, 15)
raton   = Producto("Ratón inalámbrico", 29.99)     # stock=0 por defecto

print(teclado)              # Teclado mecánico — 89.99€ (✓ disponible)
print(raton)                # Ratón inalámbrico — 29.99€ (✗ agotado)
print(teclado.esta_disponible())    # True
print(repr(teclado))        # Producto('Teclado mecánico', 89.99, 15)

teclado.aplicar_descuento(10)
print(teclado.precio)       # 80.99
```

---

## Parte 2 — Atributos de clase vs atributos de instancia

```python
class Contador:
    # Atributo de clase — compartido por TODAS las instancias
    total_creados = 0

    def __init__(self, nombre: str):
        # Atributos de instancia — propios de cada objeto
        self.nombre = nombre
        self.valor  = 0
        Contador.total_creados += 1    # incrementa el contador global

    def incrementar(self, cantidad: int = 1) -> None:
        self.valor += cantidad

    def __str__(self) -> str:
        return f"Contador '{self.nombre}': {self.valor}"


c1 = Contador("visitas")
c2 = Contador("clics")
c3 = Contador("errores")

c1.incrementar(100)
c2.incrementar(50)
c2.incrementar(25)

print(c1)                     # Contador 'visitas': 100
print(c2)                     # Contador 'clics': 75
print(Contador.total_creados) # 3  — atributo de clase
```

---

## Parte 3 — `@property` — getters y setters

`@property` convierte un método en un atributo calculado y permite
validar asignaciones con `@nombre.setter`.

```python
class Temperatura:
    """Almacena temperatura en Celsius con conversión automática."""

    def __init__(self, celsius: float):
        self._celsius = celsius    # _ indica atributo "privado" por convención

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, valor: float) -> None:
        if valor < -273.15:
            raise ValueError("Temperatura por debajo del cero absoluto")
        self._celsius = valor

    @property
    def fahrenheit(self) -> float:
        """Propiedad calculada — no almacena el valor, lo calcula."""
        return self._celsius * 9/5 + 32

    @property
    def kelvin(self) -> float:
        return self._celsius + 273.15

    def __str__(self) -> str:
        return (f"{self._celsius:.1f}°C = "
                f"{self.fahrenheit:.1f}°F = "
                f"{self.kelvin:.2f}K")


temp = Temperatura(100)
print(temp)              # 100.0°C = 212.0°F = 373.15K

temp.celsius = 0
print(temp.fahrenheit)   # 32.0 — propiedad calculada, no almacenada

# El setter valida el valor
try:
    temp.celsius = -300   # menor que el cero absoluto
except ValueError as e:
    print(e)              # Temperatura por debajo del cero absoluto
```

---

## Parte 4 — Herencia

```python
class Animal:
    """Clase base para todos los animales."""

    def __init__(self, nombre: str, edad: int):
        self.nombre = nombre
        self.edad   = edad

    def describir(self) -> str:
        return f"{self.nombre} tiene {self.edad} año(s)"

    def hablar(self) -> str:
        return "..."    # las subclases sobreescriben este método


class Perro(Animal):
    """Hereda de Animal y añade su propio comportamiento."""

    def __init__(self, nombre: str, edad: int, raza: str):
        super().__init__(nombre, edad)    # llama al __init__ del padre
        self.raza = raza

    def hablar(self) -> str:
        return "¡Guau!"    # sobreescribe el método del padre

    def describir(self) -> str:
        base = super().describir()    # llama al método del padre
        return f"{base} y es un {self.raza}"


class Gato(Animal):
    def __init__(self, nombre: str, edad: int, interior: bool = True):
        super().__init__(nombre, edad)
        self.interior = interior

    def hablar(self) -> str:
        return "¡Miau!"

    def describir(self) -> str:
        tipo = "de interior" if self.interior else "de exterior"
        return f"{super().describir()} ({tipo})"


# Uso
animales = [
    Perro("Rex",   3, "Labrador"),
    Gato("Misi",   5),
    Perro("Buddy", 1, "Golden Retriever"),
    Gato("Negra",  2, interior=False),
]

for animal in animales:
    print(f"{animal.nombre}: {animal.hablar()} — {animal.describir()}")
```

Salida:

```
Rex: ¡Guau! — Rex tiene 3 año(s) y es un Labrador
Misi: ¡Miau! — Misi tiene 5 año(s) (de interior)
Buddy: ¡Guau! — Buddy tiene 1 año(s) y es un Golden Retriever
Negra: ¡Miau! — Negra tiene 2 año(s) (de exterior)
```

### Verificar tipos con `isinstance` e `issubclass`

```python
rex = Perro("Rex", 3, "Labrador")

print(isinstance(rex, Perro))     # True  — es un Perro
print(isinstance(rex, Animal))    # True  — también es un Animal (herencia)
print(isinstance(rex, Gato))      # False

print(issubclass(Perro, Animal))  # True
print(issubclass(Gato, Animal))   # True
print(issubclass(Animal, Perro))  # False
```

---

## Parte 5 — Métodos especiales (dunder methods)

Los métodos `__nombre__` (double underscore = "dunder") definen cómo
se comporta la clase con operadores y funciones de Python.

```python
class Carrito:
    def __init__(self):
        self._items = []

    def agregar(self, nombre: str, precio: float, cantidad: int = 1):
        self._items.append({"nombre": nombre, "precio": precio, "cantidad": cantidad})

    # len(carrito) — qué devuelve la función len()
    def __len__(self) -> int:
        return sum(item["cantidad"] for item in self._items)

    # str(carrito) y print(carrito)
    def __str__(self) -> str:
        if not self._items:
            return "Carrito vacío"
        lineas = [f"  · {i['nombre']} x{i['cantidad']} — {i['precio']*i['cantidad']:.2f}€"
                  for i in self._items]
        return "Carrito:\n" + "\n".join(lineas) + f"\n  Total: {self.total:.2f}€"

    # if carrito — es truthy si tiene items
    def __bool__(self) -> bool:
        return len(self._items) > 0

    # carrito1 + carrito2 — combina dos carritos
    def __add__(self, otro: "Carrito") -> "Carrito":
        nuevo = Carrito()
        nuevo._items = self._items + otro._items
        return nuevo

    @property
    def total(self) -> float:
        return sum(i["precio"] * i["cantidad"] for i in self._items)


c1 = Carrito()
c1.agregar("Teclado", 50, 1)
c1.agregar("Ratón",   25, 2)

c2 = Carrito()
c2.agregar("Monitor", 280, 1)

print(len(c1))        # 3  — total de artículos
print(bool(c1))       # True
print(bool(Carrito())) # False — carrito vacío

combinado = c1 + c2
print(combinado)
```

---

## Parte 6 — Clases de datos con `@dataclass`

`@dataclass` (Python 3.7+) genera automáticamente `__init__`, `__str__`,
`__repr__` y `__eq__` a partir de las anotaciones de tipo.
Equivale a los data classes de Kotlin.

```python
from dataclasses import dataclass, field

@dataclass
class Punto:
    x: float
    y: float

    def distancia_al_origen(self) -> float:
        return (self.x**2 + self.y**2) ** 0.5


@dataclass
class Pedido:
    id:       int
    cliente:  str
    items:    list = field(default_factory=list)  # lista mutable como default
    enviado:  bool = False

    def agregar_item(self, item: str) -> None:
        self.items.append(item)

    @property
    def resumen(self) -> str:
        return f"Pedido #{self.id} de {self.cliente} — {len(self.items)} artículos"


# @dataclass genera __init__ automáticamente
p = Punto(3.0, 4.0)
print(p)                          # Punto(x=3.0, y=4.0)
print(p.distancia_al_origen())    # 5.0

# Comparación automática
print(Punto(1, 2) == Punto(1, 2))    # True  — __eq__ generado por @dataclass
print(Punto(1, 2) == Punto(3, 4))    # False

pedido = Pedido(id=1, cliente="Ana")
pedido.agregar_item("Teclado")
pedido.agregar_item("Ratón")
print(pedido.resumen)    # Pedido #1 de Ana — 2 artículos
```

---

## Ejemplo completo: sistema de biblioteca

```python
from dataclasses import dataclass, field
from datetime import date, timedelta


@dataclass
class Libro:
    isbn:   str
    titulo: str
    autor:  str

    def __str__(self) -> str:
        return f'"{self.titulo}" — {self.autor} [{self.isbn}]'


class Prestamo:
    DIAS_PRESTAMO = 14

    def __init__(self, libro: Libro, socio: str):
        self.libro        = libro
        self.socio        = socio
        self.fecha_inicio = date.today()
        self.fecha_limite = date.today() + timedelta(days=self.DIAS_PRESTAMO)
        self.devuelto     = False

    @property
    def en_mora(self) -> bool:
        return not self.devuelto and date.today() > self.fecha_limite

    def __str__(self) -> str:
        estado = "devuelto" if self.devuelto else (
            "EN MORA" if self.en_mora else "activo"
        )
        return f"{self.socio} → {self.libro.titulo} [{estado}]"


class Biblioteca:
    def __init__(self, nombre: str):
        self.nombre   = nombre
        self._libros   = {}      # isbn → Libro
        self._prestamos = []

    def registrar(self, libro: Libro) -> None:
        self._libros[libro.isbn] = libro

    def prestar(self, isbn: str, socio: str) -> Prestamo:
        if isbn not in self._libros:
            raise KeyError(f"Libro {isbn} no registrado")
        prestamos_activos = [p for p in self._prestamos
                             if p.libro.isbn == isbn and not p.devuelto]
        if prestamos_activos:
            raise RuntimeError(f"El libro ya está prestado")
        prestamo = Prestamo(self._libros[isbn], socio)
        self._prestamos.append(prestamo)
        return prestamo

    def devolver(self, isbn: str, socio: str) -> None:
        for prestamo in self._prestamos:
            if prestamo.libro.isbn == isbn and prestamo.socio == socio and not prestamo.devuelto:
                prestamo.devuelto = True
                return
        raise LookupError(f"Préstamo no encontrado")

    def prestamos_activos(self) -> list:
        return [p for p in self._prestamos if not p.devuelto]


# Uso
bib = Biblioteca("Biblioteca Municipal")
bib.registrar(Libro("978-0-13-468599-1", "Clean Code",    "Robert C. Martin"))
bib.registrar(Libro("978-0-13-235088-4", "Clean Coder",   "Robert C. Martin"))
bib.registrar(Libro("978-0-20-163361-0", "Design Patterns","GoF"))

p1 = bib.prestar("978-0-13-468599-1", "Ana García")
p2 = bib.prestar("978-0-20-63361-0",  "Luis Pérez")

for p in bib.prestamos_activos():
    print(p)
```

---

## Ejercicios propuestos

1. **Clase `CuentaBancaria`** — Crea una clase con `titular`, `saldo` y
   los métodos `depositar(monto)`, `retirar(monto)` (lanza error si no hay
   fondos) y `__str__`. Añade un atributo de clase `total_cuentas` que cuente
   cuántas cuentas se han creado. Usa `@property` para el `saldo` y valida
   que nunca sea negativo en el setter.

2. **Jerarquía de figuras** — Crea la clase base `Figura` con el método
   abstracto `area()` y `perimetro()`. Luego crea `Circulo(radio)`,
   `Rectangulo(ancho, alto)` y `Triangulo(base, altura, lado1, lado2, lado3)`.
   Implementa `__str__` en cada una e incluye `isinstance` en una función
   `describir_figura(fig)` que imprima el tipo y las métricas.

3. **`@dataclass` de inventario** — Usa `@dataclass` para crear `Articulo`
   con `id`, `nombre`, `precio` y `stock`. Crea `Almacen` como clase normal
   con una lista de `Articulo`. Implementa `buscar(nombre)`, `reabastecer(id, cantidad)`,
   y `__len__` que devuelva el número de artículos distintos.

4. **Métodos especiales** — Añade a la clase `Carrito` del ejemplo los métodos
   `__contains__` (para poder usar `"Teclado" in carrito`),
   `__iter__` (para poder hacer `for item in carrito`) y
   `__getitem__` (para poder hacer `carrito[0]`).

---

## Resumen de la página 7

- `class Nombre:` define una clase. `__init__(self, ...)` es el constructor. `self` equivale a `this` en JS pero hay que declararlo en cada método.
- Los atributos de **instancia** se definen en `__init__` con `self.nombre`. Los atributos de **clase** se definen fuera de `__init__` y se comparten entre todas las instancias.
- El prefijo `_` en un atributo indica que es "privado" por convención — Python no lo oculta, pero señala que no debería usarse desde fuera.
- `@property` crea atributos calculados y permite validación con `@nombre.setter`.
- La herencia usa paréntesis: `class Hijo(Padre)`. `super().__init__(...)` llama al constructor del padre.
- `isinstance(obj, Clase)` comprueba si un objeto es de esa clase o de una subclase — más seguro que comparar tipos con `type()`.
- Los **métodos dunder** (`__str__`, `__len__`, `__add__`, etc.) definen el comportamiento con operadores y funciones de Python.
- `@dataclass` genera automáticamente `__init__`, `__repr__` y `__eq__` — ideal para clases que principalmente almacenan datos.

---

> **Siguiente página →** Página 8: Manejo de errores, módulos y entornos
> virtuales — `try/except/finally`, crear y usar módulos propios, `pip` y `venv`.
