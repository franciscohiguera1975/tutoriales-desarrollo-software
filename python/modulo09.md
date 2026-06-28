# Tutorial Python — Página 9
## Módulo 3 · Programación Orientada a Objetos
### POO avanzada: dataclasses, decoradores, descriptores y slots

---

## Parte 1 — @dataclass

`@dataclass` genera automáticamente `__init__`, `__repr__` y `__eq__` a partir
de las anotaciones de clase. Es la forma moderna de crear clases de datos en
Python 3.7+ y se mejora significativamente en 3.10–3.12.

```python
from dataclasses import dataclass, field, KW_ONLY
from datetime import datetime

# ─────────────────────────────────────────────────────────────────────────────
# Dataclass básica — Python genera __init__, __repr__ y __eq__
# ─────────────────────────────────────────────────────────────────────────────
@dataclass
class Producto:
    nombre: str
    precio: float
    stock: int = 0                                    # valor por defecto simple

# Uso
p = Producto("Teclado", 49.99, stock=10)
print(p)           # Producto(nombre='Teclado', precio=49.99, stock=10)
print(p == Producto("Teclado", 49.99, stock=10))   # True — __eq__ generado


# ─────────────────────────────────────────────────────────────────────────────
# field() — control fino de cada campo
# ─────────────────────────────────────────────────────────────────────────────
@dataclass
class Pedido:
    id_pedido: int
    # default_factory para valores mutables (listas, dicts, etc.)
    articulos: list[str] = field(default_factory=list)
    # repr=False oculta el campo en __repr__
    _interno: str = field(default="", repr=False)
    # compare=False excluye el campo de __eq__ y __lt__
    fecha_creacion: datetime = field(
        default_factory=datetime.now,
        compare=False
    )

p1 = Pedido(1)
p2 = Pedido(1)
print(p1 == p2)    # True — fecha_creacion está excluida de la comparación


# ─────────────────────────────────────────────────────────────────────────────
# frozen=True — instancias inmutables (como namedtuple pero con type hints)
# ─────────────────────────────────────────────────────────────────────────────
@dataclass(frozen=True)
class Coordenada:
    latitud: float
    longitud: float

    def distancia_al_origen(self) -> float:
        return (self.latitud**2 + self.longitud**2) ** 0.5

coord = Coordenada(40.4168, -3.7038)
# coord.latitud = 0   # lanzaría FrozenInstanceError
print(coord.distancia_al_origen())   # 5.38...


# ─────────────────────────────────────────────────────────────────────────────
# __post_init__ — lógica después del __init__ generado
# ─────────────────────────────────────────────────────────────────────────────
@dataclass
class Usuario:
    nombre: str
    email: str
    rol: str = "usuario"
    # KW_ONLY: todo lo que sigue solo puede pasarse como keyword argument
    _: KW_ONLY
    activo: bool = True

    def __post_init__(self) -> None:
        # Normalizar email a minúsculas
        self.email = self.email.lower().strip()
        # Validar rol permitido
        roles_validos = {"usuario", "admin", "moderador"}
        if self.rol not in roles_validos:
            raise ValueError(
                f"Rol inválido: {self.rol!r}. Debe ser uno de {roles_validos}"
            )

u = Usuario("Ana", "ANA@EJEMPLO.COM", "admin")
print(u.email)   # ana@ejemplo.com
```

### Comparación: dataclass vs clase manual vs NamedTuple

```python
from typing import NamedTuple

# ─── 1. Clase manual ────────────────────────────────────────────────────────
class PuntoManual:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:
        return f"PuntoManual(x={self.x}, y={self.y})"

    def __eq__(self, otro: object) -> bool:
        if not isinstance(otro, PuntoManual):
            return NotImplemented
        return self.x == otro.x and self.y == otro.y
    # Requiere muchas líneas de código repetitivo (boilerplate)


# ─── 2. NamedTuple ──────────────────────────────────────────────────────────
class PuntoNT(NamedTuple):
    x: float
    y: float
    # inmutable, iterable, desempaquetable — menos flexible para lógica compleja


# ─── 3. Dataclass ───────────────────────────────────────────────────────────
@dataclass
class PuntoDC:
    x: float
    y: float
    # mutable por defecto, soporta herencia, __post_init__ y slots


# Cuándo usar cada uno:
# NamedTuple  → datos simples, inmutables, que se usan como tuplas
# dataclass   → la mayoría de los casos: clase de datos con lógica
# clase manual → cuando necesitas control total o __init__ muy complejo
```

---

## Parte 2 — Decoradores propios

Un decorador es una función que recibe otra función y devuelve una versión
mejorada. `functools.wraps` preserva el nombre y la documentación originales.

```python
import functools
import time
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")   # captura los parámetros de la función decorada
T = TypeVar("T")     # captura el tipo de retorno


# ─────────────────────────────────────────────────────────────────────────────
# Decorador simple — medir tiempo de ejecución
# ─────────────────────────────────────────────────────────────────────────────
def medir_tiempo(func: Callable[P, T]) -> Callable[P, T]:
    @functools.wraps(func)          # preserva __name__, __doc__, __module__
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
        inicio = time.perf_counter()
        resultado = func(*args, **kwargs)
        duracion = time.perf_counter() - inicio
        print(f"[tiempo] {func.__name__} tardó {duracion:.4f}s")
        return resultado
    return envoltura


@medir_tiempo
def calcular_primos(limite: int) -> list[int]:
    """Devuelve los primos hasta `limite` usando la criba de Eratóstenes."""
    criba = [True] * (limite + 1)
    criba[0] = criba[1] = False
    for i in range(2, int(limite**0.5) + 1):
        if criba[i]:
            for j in range(i * i, limite + 1, i):
                criba[j] = False
    return [n for n, es_primo in enumerate(criba) if es_primo]

primos = calcular_primos(100_000)
# [tiempo] calcular_primos tardó 0.0123s
print(calcular_primos.__name__)   # calcular_primos — wraps lo preserva


# ─────────────────────────────────────────────────────────────────────────────
# Decorador con argumentos — reintentar en caso de excepción
# Requiere tres niveles: fábrica → decorador → envoltura
# ─────────────────────────────────────────────────────────────────────────────
def reintentar(
    veces: int = 3,
    espera: float = 1.0,
    excepciones: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[P, T]], Callable[P, T]]:
    """Reintenta la función hasta `veces` veces si lanza alguna de `excepciones`."""
    def decorador(func: Callable[P, T]) -> Callable[P, T]:
        @functools.wraps(func)
        def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
            ultimo_error: Exception | None = None
            for intento in range(1, veces + 1):
                try:
                    return func(*args, **kwargs)
                except excepciones as e:
                    ultimo_error = e
                    print(f"[reintento {intento}/{veces}] {func.__name__} falló: {e}")
                    if intento < veces:
                        time.sleep(espera)
            # Si agotamos todos los intentos, relanzamos el último error
            raise RuntimeError(
                f"{func.__name__} falló tras {veces} intentos"
            ) from ultimo_error
        return envoltura
    return decorador


import random

@reintentar(veces=3, espera=0.1, excepciones=(ValueError,))
def servicio_inestable(valor: int) -> str:
    """Simula un servicio que falla aleatoriamente."""
    if random.random() < 0.7:         # 70% de probabilidad de fallo
        raise ValueError("Servicio no disponible temporalmente")
    return f"Resultado: {valor * 2}"


# ─────────────────────────────────────────────────────────────────────────────
# Decorador de caché manual (alternativa didáctica a functools.lru_cache)
# ─────────────────────────────────────────────────────────────────────────────
def cachear(func: Callable[P, T]) -> Callable[P, T]:
    """Memoriza resultados usando los argumentos como clave del diccionario."""
    _cache: dict[tuple, T] = {}

    @functools.wraps(func)
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
        # Construimos una clave hasheable con args y kwargs ordenados
        clave = (args, tuple(sorted(kwargs.items())))
        if clave not in _cache:
            _cache[clave] = func(*args, **kwargs)
            print(f"[cache] Miss — calculando {func.__name__}{args}")
        else:
            print(f"[cache] Hit  — {func.__name__}{args}")
        return _cache[clave]

    # Exponer el caché para inspección o limpieza externa
    envoltura.cache = _cache          # type: ignore[attr-defined]
    return envoltura


@cachear
def fibonacci(n: int) -> int:
    """Calcula el n-ésimo número de Fibonacci de forma recursiva."""
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

### Decoradores de clase

```python
# ─────────────────────────────────────────────────────────────────────────────
# Decorador que añade un logger a la clase y envuelve sus métodos públicos
# ─────────────────────────────────────────────────────────────────────────────
import logging

def con_logging(cls: type) -> type:
    """Añade un logger nombrado a la clase y registra cada llamada pública."""
    cls.logger = logging.getLogger(cls.__name__)  # type: ignore[attr-defined]

    for nombre, metodo in list(vars(cls).items()):
        if callable(metodo) and not nombre.startswith("_"):
            def crear_envuelto(fn: Callable) -> Callable:
                @functools.wraps(fn)
                def envuelto(self, *args, **kwargs):
                    self.logger.debug("Llamando %s", fn.__name__)
                    return fn(self, *args, **kwargs)
                return envuelto
            setattr(cls, nombre, crear_envuelto(metodo))
    return cls


@con_logging
class Repositorio:
    def guardar(self, dato: dict) -> None:
        print(f"Guardando: {dato}")

    def buscar(self, id_: int) -> dict:
        return {"id": id_, "dato": "ejemplo"}
```

---

## Parte 3 — Descriptores

Un descriptor es cualquier objeto que implementa `__get__`, `__set__` o
`__delete__`. Es el mecanismo interno detrás de `@property`, `@classmethod`
y `@staticmethod`.

```python
# ─────────────────────────────────────────────────────────────────────────────
# Descriptor validador — reutilizable en múltiples clases
# ─────────────────────────────────────────────────────────────────────────────
class CampoPositivo:
    """Descriptor que valida que el valor sea un número mayor que cero."""

    def __set_name__(self, owner: type, name: str) -> None:
        # Python llama a __set_name__ al definir la clase
        self.nombre_publico = name
        self.nombre_privado = f"_{name}"   # almacenamos con prefijo _

    def __get__(self, obj: object, objtype: type | None = None) -> float:
        if obj is None:
            # Acceso desde la clase, no desde una instancia
            return self  # type: ignore[return-value]
        return getattr(obj, self.nombre_privado, 0.0)

    def __set__(self, obj: object, valor: float) -> None:
        if not isinstance(valor, int | float):
            raise TypeError(
                f"{self.nombre_publico} debe ser numérico, no {type(valor).__name__}"
            )
        if valor <= 0:
            raise ValueError(
                f"{self.nombre_publico} debe ser positivo, recibido: {valor}"
            )
        setattr(obj, self.nombre_privado, float(valor))

    def __delete__(self, obj: object) -> None:
        raise AttributeError(f"No se puede eliminar '{self.nombre_publico}'")


class Factura:
    # Los descriptores se declaran como atributos de clase
    subtotal: float = CampoPositivo()     # type: ignore[assignment]
    impuesto: float = CampoPositivo()     # type: ignore[assignment]

    def __init__(self, subtotal: float, impuesto: float) -> None:
        self.subtotal = subtotal      # invoca CampoPositivo.__set__
        self.impuesto = impuesto

    def total(self) -> float:
        return self.subtotal + self.impuesto


f = Factura(100.0, 21.0)
print(f.total())           # 121.0
# Factura(-10, 5)          # ValueError: subtotal debe ser positivo


# ─────────────────────────────────────────────────────────────────────────────
# __slots__ — optimización de memoria
# Sin __slots__: cada instancia tiene un __dict__ (más memoria, más lento)
# Con __slots__: Python usa descriptores internos de posición fija
# ─────────────────────────────────────────────────────────────────────────────
class PuntoSlots:
    __slots__ = ("x", "y", "z")   # solo se permiten estos tres atributos

    def __init__(self, x: float, y: float, z: float = 0.0) -> None:
        self.x = x
        self.y = y
        self.z = z

    def magnitud(self) -> float:
        return (self.x**2 + self.y**2 + self.z**2) ** 0.5


# Comparación de consumo de memoria
import sys

class SinSlots:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

p_sin = SinSlots(1.0, 2.0, 3.0)
p_con = PuntoSlots(1.0, 2.0, 3.0)

# p_sin tiene __dict__ interno (~232 bytes extra)
# p_con no tiene __dict__ — acceso hasta 2x más rápido en bucles masivos
print(hasattr(p_sin, "__dict__"))   # True
print(hasattr(p_con, "__dict__"))   # False
```

---

## Ejemplo aplicado — Sistema de validación con decoradores

Sistema completo que combina decoradores de validación, medición de tiempo,
reintentos y caché para una API de procesamiento de pedidos.

```python
"""
sistema_pedidos.py
Sistema de validación y procesamiento de pedidos con decoradores encadenados.
"""
from __future__ import annotations

import functools
import logging
import time
import random
from dataclasses import dataclass, field
from typing import Callable, TypeVar, ParamSpec

# ── Configuración de logging ─────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s — %(message)s",
)
logger = logging.getLogger("sistema_pedidos")

P = ParamSpec("P")
T = TypeVar("T")


# ── Dataclasses de dominio ───────────────────────────────────────────────────
@dataclass
class LineaPedido:
    sku: str
    cantidad: int
    precio_unitario: float

    def subtotal(self) -> float:
        return self.cantidad * self.precio_unitario


@dataclass
class Pedido:
    id_pedido: str
    cliente: str
    lineas: list[LineaPedido] = field(default_factory=list)
    estado: str = "pendiente"

    def total(self) -> float:
        return sum(linea.subtotal() for linea in self.lineas)

    def agregar_linea(self, linea: LineaPedido) -> None:
        self.lineas.append(linea)


# ── Decoradores de infraestructura ───────────────────────────────────────────
def registrar(func: Callable[P, T]) -> Callable[P, T]:
    """Registra entrada, salida y errores de cada llamada al servicio."""
    @functools.wraps(func)
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
        logger.info("→ %s llamado", func.__name__)
        try:
            resultado = func(*args, **kwargs)
            logger.info("← %s completado", func.__name__)
            return resultado
        except Exception as e:
            logger.error("✗ %s falló: %s", func.__name__, e)
            raise
    return envoltura


def validar_pedido(func: Callable[P, T]) -> Callable[P, T]:
    """Valida que el primer argumento sea un Pedido con líneas y total válido."""
    @functools.wraps(func)
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
        pedido = args[0] if args else kwargs.get("pedido")
        if not isinstance(pedido, Pedido):
            raise TypeError(f"Se esperaba Pedido, se recibió {type(pedido).__name__}")
        if not pedido.lineas:
            raise ValueError(f"Pedido {pedido.id_pedido} no tiene líneas")
        if pedido.total() <= 0:
            raise ValueError(
                f"Pedido {pedido.id_pedido} tiene total inválido: {pedido.total()}"
            )
        return func(*args, **kwargs)
    return envoltura


def reintentar_en_fallo(
    veces: int = 3,
    espera: float = 0.5,
) -> Callable[[Callable[P, T]], Callable[P, T]]:
    """Reintenta la función si lanza ConnectionError o TimeoutError."""
    def decorador(func: Callable[P, T]) -> Callable[P, T]:
        @functools.wraps(func)
        def envoltura(*args: P.args, **kwargs: P.kwargs) -> T:
            for intento in range(1, veces + 1):
                try:
                    return func(*args, **kwargs)
                except (ConnectionError, TimeoutError) as e:
                    logger.warning(
                        "Intento %d/%d falló en %s: %s",
                        intento, veces, func.__name__, e
                    )
                    if intento == veces:
                        raise
                    time.sleep(espera * intento)   # espera progresiva
            raise RuntimeError("Código inalcanzable")
        return envoltura
    return decorador


def cachear_resultado(func: Callable[P, T]) -> Callable[P, T]:
    """Caché simple usando el id_pedido como clave de almacenamiento."""
    _cache: dict[str, T] = {}

    @functools.wraps(func)
    def envoltura(pedido: Pedido, *args: P.args, **kwargs: P.kwargs) -> T:
        if pedido.id_pedido in _cache:
            logger.info("[caché] Resultado guardado para pedido %s", pedido.id_pedido)
            return _cache[pedido.id_pedido]
        resultado = func(pedido, *args, **kwargs)
        _cache[pedido.id_pedido] = resultado
        return resultado

    # Exponer método para limpiar el caché desde el exterior
    envoltura.limpiar_cache = lambda: _cache.clear()  # type: ignore[attr-defined]
    return envoltura


# ── Servicio de procesamiento ─────────────────────────────────────────────────
class ProcesadorPedidos:
    """Los decoradores se aplican de abajo hacia arriba al definir el método."""

    @registrar
    @validar_pedido
    @reintentar_en_fallo(veces=3, espera=0.2)
    @cachear_resultado
    def procesar(self, pedido: Pedido) -> dict:
        """Envía el pedido al sistema externo y devuelve la confirmación."""
        # Simular fallo de red ocasional (30% de probabilidad)
        if random.random() < 0.3:
            raise ConnectionError("API externa no disponible")

        # Procesamiento real — aquí iría la llamada HTTP
        pedido.estado = "procesado"
        return {
            "id_pedido": pedido.id_pedido,
            "cliente": pedido.cliente,
            "total": pedido.total(),
            "estado": pedido.estado,
        }

    @registrar
    def generar_factura(self, pedido: Pedido) -> str:
        """Genera una factura en texto para el pedido procesado."""
        lineas = "\n".join(
            f"  {l.sku:.<30} {l.cantidad:3} x {l.precio_unitario:8.2f}"
            f" = {l.subtotal():10.2f}"
            for l in pedido.lineas
        )
        return (
            f"FACTURA — Pedido {pedido.id_pedido}\n"
            f"Cliente: {pedido.cliente}\n"
            f"{'─' * 60}\n"
            f"{lineas}\n"
            f"{'─' * 60}\n"
            f"TOTAL: {pedido.total():>50.2f}\n"
        )


# ── Demostración ──────────────────────────────────────────────────────────────
if __name__ == "__main__":
    # Crear pedido de prueba con tres líneas
    pedido = Pedido(id_pedido="PED-2024-001", cliente="Empresa ABC S.L.")
    pedido.agregar_linea(LineaPedido("SKU-001", 5, 19.99))
    pedido.agregar_linea(LineaPedido("SKU-002", 2, 149.50))
    pedido.agregar_linea(LineaPedido("SKU-003", 10, 4.75))

    procesador = ProcesadorPedidos()

    # Primera llamada — procesará y cacheará el resultado
    resultado = procesador.procesar(pedido)
    print(resultado)

    # Segunda llamada — devuelve el resultado desde el caché
    resultado2 = procesador.procesar(pedido)
    assert resultado == resultado2

    # Generar factura del pedido procesado
    factura = procesador.generar_factura(pedido)
    print(factura)
```

---

## Ejercicios propuestos

1. **Inventario con descriptores** — Crea una clase `Inventario` que use descriptores
   personalizados para validar que `stock` sea un entero no negativo y `precio`
   sea un float positivo. El descriptor debe incluir un historial de los últimos
   10 valores asignados al atributo, accesible mediante un método `historial()`.

2. **Decorador de permisos** — Implementa `@requiere_rol("admin", "supervisor")` que
   compruebe si el usuario actual (almacenado en una variable global de contexto)
   tiene uno de los roles indicados. Si no lo tiene, lanza `PermissionError`. Úsalo
   en una clase `SistemaRRHH` con los métodos `despedir`, `contratar` y `ver_nominas`.

3. **Dataclass árbol de configuración** — Diseña un sistema con tres dataclasses
   anidadas: `ConfigBD` (host, puerto, nombre_bd, usuario), `ConfigAPI`
   (url_base, timeout, reintentos) y `Config` (bd, api, debug, version). Añade
   `__post_init__` que valide que el timeout sea positivo y que la URL empiece
   por `https://`. Implementa un método `desde_env()` que cargue los valores
   desde variables de entorno.

4. **Decorador singleton con argumentos** — Implementa `@singleton` que convierta
   cualquier clase en un singleton. Luego crea `@singleton_por_clave(attr)` que
   devuelva la misma instancia para el mismo valor del atributo indicado. Por
   ejemplo, la misma instancia de `ConexionBD` para el mismo `nombre_bd`, pero
   instancias distintas para bases de datos distintas.

---

## Resumen de la página 6

- `@dataclass` genera `__init__`, `__repr__` y `__eq__` automáticamente; `field()` da control fino de cada atributo
- `frozen=True` crea instancias inmutables y hashables; `__post_init__` permite validar o transformar tras la inicialización
- Un decorador es una función de orden superior: recibe una función y devuelve otra mejorada; `functools.wraps` preserva la firma original
- Los decoradores con argumentos requieren tres niveles de función: la fábrica, el decorador y la envoltura
- Los decoradores se encadenan y se aplican de abajo hacia arriba (`@a @b @c` aplica primero `c`, luego `b`, luego `a`)
- Un descriptor implementa `__get__`/`__set__`/`__delete__` y permite reutilizar lógica de validación en múltiples clases sin repetir código
- `__set_name__` se llama automáticamente al definir la clase y permite que el descriptor conozca su propio nombre de atributo
- `__slots__` elimina el `__dict__` de las instancias, reduce el uso de memoria y acelera el acceso a atributos en clases de alto rendimiento

---

> **Siguiente página →** Página 10: Módulos, paquetes, pip, virtualenv y Poetry.
