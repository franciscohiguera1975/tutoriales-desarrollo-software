# Tutorial Python — Página 20
## Módulo 11 · Patrones y Buenas Prácticas
### Patrones de diseño, generadores avanzados y protocolo estructural

---

## Patrones de diseño en Python

Los patrones de diseño son soluciones probadas a problemas recurrentes.
Python, al ser dinámico, implementa algunos de forma más simple que
lenguajes como Java o C++. Varios patrones ya están en la stdlib.

```
Creacionales    → cómo crear objetos
Estructurales   → cómo componer objetos
De comportamiento → cómo interactúan los objetos
```

---

## Patrones creacionales

### Singleton — instancia única a nivel de módulo

Python no necesita una clase Singleton complicada:
**un módulo es un singleton por naturaleza** (se importa una sola vez).

```python
# config.py — Singleton mediante módulo
"""
Este módulo es importado una sola vez por Python.
Todos los imports de 'config' ven el mismo objeto.
"""
from dataclasses import dataclass, field
from pathlib import Path
import os


@dataclass
class Configuracion:
    entorno: str = "development"
    debug: bool = True
    base_url: str = "http://localhost:8000"
    db_url: str = "sqlite:///app.db"
    secret_key: str = "clave_desarrollo_insegura"
    max_conexiones: int = 10
    _cargada: bool = field(default=False, repr=False)

    def cargar_desde_entorno(self) -> None:
        """Cargar valores desde variables de entorno."""
        self.entorno   = os.getenv("APP_ENV", self.entorno)
        self.debug     = os.getenv("DEBUG", "true").lower() == "true"
        self.base_url  = os.getenv("BASE_URL", self.base_url)
        self.db_url    = os.getenv("DATABASE_URL", self.db_url)
        self.secret_key = os.getenv("SECRET_KEY", self.secret_key)
        self._cargada  = True

    @property
    def es_produccion(self) -> bool:
        return self.entorno == "production"


# Instancia única — se crea una sola vez cuando se importa el módulo
config = Configuracion()
config.cargar_desde_entorno()


# Uso en cualquier parte de la app:
# from config import config
# if config.debug:
#     logger.setLevel(logging.DEBUG)
```

```python
# Singleton con clase (cuando necesitas lazy init o herencia)
class ConexionRedis:
    _instancia: "ConexionRedis | None" = None

    def __new__(cls) -> "ConexionRedis":
        if cls._instancia is None:
            cls._instancia = super().__new__(cls)
            cls._instancia._inicializada = False
        return cls._instancia

    def conectar(self, host: str = "localhost", puerto: int = 6379) -> None:
        if not self._inicializada:
            # Solo conecta una vez, aunque se llame varias veces
            print(f"Conectando a Redis en {host}:{puerto}")
            self._host = host
            self._puerto = puerto
            self._inicializada = True
        else:
            print("Ya conectado — reutilizando conexión")


# Siempre es la misma instancia
r1 = ConexionRedis()
r2 = ConexionRedis()
assert r1 is r2  # True
```

### Factory — crear objetos según el tipo

```python
# factory.py
from abc import ABC, abstractmethod
from dataclasses import dataclass


# Jerarquía de notificaciones
class Notificacion(ABC):
    @abstractmethod
    def enviar(self, mensaje: str, destinatario: str) -> bool: ...

    @abstractmethod
    def nombre(self) -> str: ...


@dataclass
class NotificacionEmail(Notificacion):
    smtp_host: str = "smtp.ejemplo.com"
    smtp_puerto: int = 587

    def enviar(self, mensaje: str, destinatario: str) -> bool:
        print(f"[Email] → {destinatario}: {mensaje}")
        return True

    def nombre(self) -> str:
        return "email"


@dataclass
class NotificacionSMS(Notificacion):
    proveedor: str = "twilio"

    def enviar(self, mensaje: str, destinatario: str) -> bool:
        print(f"[SMS] → {destinatario}: {mensaje[:160]}")
        return True

    def nombre(self) -> str:
        return "sms"


@dataclass
class NotificacionPush(Notificacion):
    app_id: str = "mi_app"

    def enviar(self, mensaje: str, destinatario: str) -> bool:
        print(f"[Push] → {destinatario}: {mensaje}")
        return True

    def nombre(self) -> str:
        return "push"


# Factory — centraliza la creación
class FabricaNotificaciones:
    _registro: dict[str, type[Notificacion]] = {
        "email": NotificacionEmail,
        "sms":   NotificacionSMS,
        "push":  NotificacionPush,
    }

    @classmethod
    def crear(cls, tipo: str, **kwargs) -> Notificacion:
        """Crear una notificación del tipo especificado."""
        if tipo not in cls._registro:
            tipos = ", ".join(cls._registro)
            raise ValueError(f"Tipo '{tipo}' desconocido. Opciones: {tipos}")
        return cls._registro[tipo](**kwargs)

    @classmethod
    def registrar(cls, tipo: str, clase: type[Notificacion]) -> None:
        """Registrar un nuevo tipo — extensible sin modificar la factory."""
        cls._registro[tipo] = clase

    @classmethod
    def tipos_disponibles(cls) -> list[str]:
        return list(cls._registro.keys())


# Uso:
notif = FabricaNotificaciones.crear("email")
notif.enviar("Bienvenido al sistema", "usuario@ejemplo.com")
# [Email] → usuario@ejemplo.com: Bienvenido al sistema

# Registrar tipo personalizado
class NotificacionSlack(Notificacion):
    def enviar(self, mensaje: str, destinatario: str) -> bool:
        print(f"[Slack] → #{destinatario}: {mensaje}")
        return True
    def nombre(self) -> str:
        return "slack"

FabricaNotificaciones.registrar("slack", NotificacionSlack)
```

### Builder — construir objetos complejos paso a paso

```python
# builder.py
from dataclasses import dataclass, field
from typing import Self


@dataclass
class ConsultaSQL:
    """Representa una consulta SQL construida de forma fluida."""
    tabla: str = ""
    campos: list[str] = field(default_factory=list)
    condiciones: list[str] = field(default_factory=list)
    orden: str = ""
    limite: int | None = None
    desplazamiento: int = 0

    def construir(self) -> str:
        """Generar el SQL final."""
        campos_str = ", ".join(self.campos) if self.campos else "*"
        sql = f"SELECT {campos_str} FROM {self.tabla}"
        if self.condiciones:
            sql += " WHERE " + " AND ".join(self.condiciones)
        if self.orden:
            sql += f" ORDER BY {self.orden}"
        if self.limite is not None:
            sql += f" LIMIT {self.limite}"
        if self.desplazamiento:
            sql += f" OFFSET {self.desplazamiento}"
        return sql


class ConstructorConsulta:
    """Builder con interfaz fluida (method chaining)."""

    def __init__(self) -> None:
        self._consulta = ConsultaSQL()

    def de(self, tabla: str) -> Self:
        self._consulta.tabla = tabla
        return self

    def seleccionar(self, *campos: str) -> Self:
        self._consulta.campos = list(campos)
        return self

    def donde(self, condicion: str) -> Self:
        self._consulta.condiciones.append(condicion)
        return self

    def ordenar_por(self, campo: str, descendente: bool = False) -> Self:
        self._consulta.orden = f"{campo} {'DESC' if descendente else 'ASC'}"
        return self

    def limitar(self, n: int) -> Self:
        self._consulta.limite = n
        return self

    def pagina(self, numero: int, tamano: int = 10) -> Self:
        self._consulta.limite = tamano
        self._consulta.desplazamiento = (numero - 1) * tamano
        return self

    def construir(self) -> str:
        if not self._consulta.tabla:
            raise ValueError("Debes especificar una tabla con .de()")
        return self._consulta.construir()


# Uso con method chaining — muy legible:
sql = (
    ConstructorConsulta()
    .de("productos")
    .seleccionar("id", "nombre", "precio", "stock")
    .donde("activo = true")
    .donde("stock > 0")
    .ordenar_por("precio")
    .pagina(2, tamano=20)
    .construir()
)
print(sql)
# SELECT id, nombre, precio, stock FROM productos
# WHERE activo = true AND stock > 0
# ORDER BY precio ASC LIMIT 20 OFFSET 20
```

---

## Patrones estructurales

### Decorator — extender comportamiento con functools.wraps

```python
# decorator.py
import time
import functools
import logging
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")

logger = logging.getLogger(__name__)


# Decorator de tiempo de ejecución
def medir_tiempo(func: Callable[P, R]) -> Callable[P, R]:
    """Mide el tiempo de ejecución de cualquier función."""
    @functools.wraps(func)  # preserva nombre, docstring y firma
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> R:
        inicio = time.perf_counter()
        resultado = func(*args, **kwargs)
        duracion = time.perf_counter() - inicio
        logger.info(f"{func.__name__} tardó {duracion:.3f}s")
        return resultado
    return envoltura


# Decorator con parámetros — factory de decorators
def reintentar(
    max_intentos: int = 3,
    espera_seg: float = 1.0,
    excepciones: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """Reintenta la función si lanza una excepción."""
    def decorador(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        def envoltura(*args: P.args, **kwargs: P.kwargs) -> R:
            ultimo_error: Exception | None = None
            for intento in range(1, max_intentos + 1):
                try:
                    return func(*args, **kwargs)
                except excepciones as e:
                    ultimo_error = e
                    if intento < max_intentos:
                        logger.warning(
                            f"{func.__name__}: intento {intento}/{max_intentos} "
                            f"fallido: {e}. Reintentando en {espera_seg}s..."
                        )
                        time.sleep(espera_seg)
            raise RuntimeError(
                f"{func.__name__} falló tras {max_intentos} intentos"
            ) from ultimo_error
        return envoltura
    return decorador


# Cache con TTL (time-to-live)
def cache_ttl(segundos: int = 60) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """Cache con expiración por tiempo."""
    def decorador(func: Callable[P, R]) -> Callable[P, R]:
        _cache: dict = {}

        @functools.wraps(func)
        def envoltura(*args: P.args, **kwargs: P.kwargs) -> R:
            # Clave del cache basada en los argumentos
            clave = (args, tuple(sorted(kwargs.items())))
            ahora = time.time()

            if clave in _cache:
                valor, timestamp = _cache[clave]
                if ahora - timestamp < segundos:
                    return valor  # retornar del cache

            # Calcular y guardar en cache
            resultado = func(*args, **kwargs)
            _cache[clave] = (resultado, ahora)
            return resultado

        # Método para limpiar el cache manualmente
        def limpiar() -> None:
            _cache.clear()
        envoltura.limpiar = limpiar  # type: ignore

        return envoltura
    return decorador


# Uso de los decorators:
@medir_tiempo
@reintentar(max_intentos=3, espera_seg=0.5, excepciones=(ConnectionError,))
def llamar_api_externa(url: str) -> dict:
    """Llama a una API externa con medición y reintentos."""
    import httpx
    respuesta = httpx.get(url, timeout=10)
    respuesta.raise_for_status()
    return respuesta.json()


@cache_ttl(segundos=300)  # cache de 5 minutos
def obtener_configuracion_remota(entorno: str) -> dict:
    """Configuración costosa — se cachea por 5 minutos."""
    time.sleep(0.1)  # simular llamada lenta
    return {"entorno": entorno, "max_workers": 4}
```

### Facade — simplificar una subsistema complejo

```python
# facade.py
"""
El patrón Facade provee una interfaz simple para un subsistema complejo.
Ejemplo: enviar un email implica configurar SMTP, construir el mensaje,
validar, adjuntar archivos — la fachada oculta todo eso.
"""
from dataclasses import dataclass
from pathlib import Path


class ValidadorEmail:
    """Subsistema: validación de emails."""
    def validar(self, email: str) -> bool:
        return "@" in email and "." in email.split("@")[-1]


class PlantillaEmail:
    """Subsistema: gestión de plantillas HTML."""
    _plantillas = {
        "bienvenida": "<h1>Bienvenido, {nombre}!</h1><p>{mensaje}</p>",
        "factura":    "<h1>Factura #{numero}</h1><p>Total: ${total}</p>",
        "alerta":     "<p style='color:red'><strong>Alerta:</strong> {texto}</p>",
    }

    def renderizar(self, plantilla: str, **variables) -> str:
        if plantilla not in self._plantillas:
            raise ValueError(f"Plantilla '{plantilla}' no encontrada")
        return self._plantillas[plantilla].format(**variables)


class ClienteSMTP:
    """Subsistema: envío SMTP (simplificado)."""
    def __init__(self, host: str, puerto: int, usuario: str, clave: str):
        self._host    = host
        self._puerto  = puerto
        self._usuario = usuario
        self._clave   = clave

    def conectar(self) -> bool:
        print(f"Conectando a {self._host}:{self._puerto}")
        return True

    def enviar(self, de: str, para: str, asunto: str,
               cuerpo: str) -> bool:
        print(f"Enviando: {de} → {para}: {asunto}")
        return True

    def desconectar(self) -> None:
        print("Conexión SMTP cerrada")


@dataclass
class ServicioEmail:
    """
    FACHADA: interfaz simple para enviar emails.
    Los clientes usan solo esta clase, sin saber nada de SMTP ni plantillas.
    """
    smtp_host:  str
    smtp_port:  int
    usuario:    str
    clave:      str
    remitente:  str

    def __post_init__(self):
        self._validador  = ValidadorEmail()
        self._plantillas = PlantillaEmail()

    def enviar_bienvenida(self, email: str, nombre: str) -> bool:
        """Enviar email de bienvenida usando la plantilla."""
        return self._enviar(
            destinatario=email,
            asunto=f"Bienvenido, {nombre}!",
            cuerpo=self._plantillas.renderizar(
                "bienvenida", nombre=nombre,
                mensaje="Gracias por registrarte."
            ),
        )

    def enviar_factura(self, email: str, numero: int, total: float) -> bool:
        return self._enviar(
            destinatario=email,
            asunto=f"Factura #{numero}",
            cuerpo=self._plantillas.renderizar(
                "factura", numero=numero, total=f"{total:.2f}"
            ),
        )

    def _enviar(self, destinatario: str, asunto: str, cuerpo: str) -> bool:
        """Método interno que orquesta el subsistema SMTP."""
        if not self._validador.validar(destinatario):
            raise ValueError(f"Email inválido: {destinatario}")

        cliente = ClienteSMTP(
            self._smtp_host, self._smtp_port,
            self._usuario, self._clave,
        )
        try:
            cliente.conectar()
            return cliente.enviar(
                self.remitente, destinatario, asunto, cuerpo
            )
        finally:
            cliente.desconectar()
```

---

## Patrones de comportamiento

### Strategy — algoritmos intercambiables

```python
# strategy.py
from typing import Protocol
from dataclasses import dataclass


# Protocolo (interfaz) de la estrategia
class EstrategiaOrden(Protocol):
    def ordenar(self, elementos: list) -> list: ...


# Implementaciones concretas de la estrategia
class OrdenPorPrecio:
    def ordenar(self, elementos: list) -> list:
        return sorted(elementos, key=lambda x: x.get("precio", 0))


class OrdenPorNombre:
    def ordenar(self, elementos: list) -> list:
        return sorted(elementos, key=lambda x: x.get("nombre", ""))


class OrdenPorStock:
    def ordenar(self, elementos: list) -> list:
        return sorted(
            elementos,
            key=lambda x: x.get("stock", 0),
            reverse=True,  # mayor stock primero
        )


class OrdenPersonalizado:
    """Estrategia configurable mediante función de clave."""
    def __init__(self, clave_fn, descendente: bool = False):
        self._clave_fn    = clave_fn
        self._descendente = descendente

    def ordenar(self, elementos: list) -> list:
        return sorted(
            elementos,
            key=self._clave_fn,
            reverse=self._descendente,
        )


# Contexto que usa la estrategia
class CatalogProductos:
    def __init__(self, estrategia: EstrategiaOrden | None = None):
        self._estrategia = estrategia or OrdenPorNombre()
        self._productos: list[dict] = []

    def agregar(self, producto: dict) -> None:
        self._productos.append(producto)

    def cambiar_estrategia(self, estrategia: EstrategiaOrden) -> None:
        """Cambiar el algoritmo de ordenamiento en tiempo de ejecución."""
        self._estrategia = estrategia

    def listar(self) -> list[dict]:
        return self._estrategia.ordenar(self._productos)


# Uso:
catalogo = CatalogProductos()
catalogo.agregar({"nombre": "Monitor", "precio": 349.99, "stock": 5})
catalogo.agregar({"nombre": "Teclado", "precio":  89.99, "stock": 20})
catalogo.agregar({"nombre": "Mouse",   "precio":  29.99, "stock": 0})

# Ordenar por precio
catalogo.cambiar_estrategia(OrdenPorPrecio())
for p in catalogo.listar():
    print(f"${p['precio']:7.2f}  {p['nombre']}")

# Ordenar por nombre
catalogo.cambiar_estrategia(OrdenPorNombre())
for p in catalogo.listar():
    print(p["nombre"])
```

### Observer — notificar cambios a múltiples interesados

```python
# observer.py
from typing import Protocol, Callable
from dataclasses import dataclass, field


class Observador(Protocol):
    def actualizar(self, evento: str, datos: dict) -> None: ...


@dataclass
class SistemaEventos:
    """Sistema de eventos simple — el Observable."""
    _observadores: dict[str, list[Callable]] = field(default_factory=dict)

    def suscribir(self, evento: str, callback: Callable) -> None:
        """Registrar un observador para un evento."""
        if evento not in self._observadores:
            self._observadores[evento] = []
        self._observadores[evento].append(callback)

    def desuscribir(self, evento: str, callback: Callable) -> None:
        if evento in self._observadores:
            self._observadores[evento].remove(callback)

    def publicar(self, evento: str, **datos) -> None:
        """Notificar a todos los observadores del evento."""
        if evento in self._observadores:
            for callback in self._observadores[evento]:
                callback(evento=evento, **datos)


# Observadores concretos
class LoggerEventos:
    def __call__(self, evento: str, **datos) -> None:
        print(f"[LOG] {evento}: {datos}")


class NotificadorEmail:
    def __call__(self, evento: str, **datos) -> None:
        if evento == "usuario.creado":
            print(f"[EMAIL] Bienvenido {datos.get('nombre', 'usuario')}!")
        elif evento == "orden.completada":
            print(f"[EMAIL] Tu orden #{datos.get('id')} está lista!")


# Uso:
eventos = SistemaEventos()
logger   = LoggerEventos()
notif    = NotificadorEmail()

eventos.suscribir("usuario.creado",  logger)
eventos.suscribir("usuario.creado",  notif)
eventos.suscribir("orden.completada", logger)
eventos.suscribir("orden.completada", notif)

eventos.publicar("usuario.creado",   nombre="Ana", email="ana@ejemplo.com")
eventos.publicar("orden.completada", id=42, total=89.99)
```

---

## Generadores avanzados

### yield from — delegar a otro generador

```python
# generadores_avanzados.py
from typing import Generator, Iterator
from pathlib import Path


# yield from delega la iteración a un generador interno
def archivos_recursivos(directorio: str) -> Generator[Path, None, None]:
    """Recorrer todos los archivos de un directorio recursivamente."""
    ruta = Path(directorio)
    for elemento in ruta.iterdir():
        if elemento.is_dir():
            yield from archivos_recursivos(str(elemento))  # delegación
        else:
            yield elemento


def encadenar(*generadores) -> Generator:
    """Encadenar múltiples generadores (equivale a itertools.chain)."""
    for gen in generadores:
        yield from gen


# send() — comunicación bidireccional con el generador
def acumulador() -> Generator[float, float, str]:
    """
    Generador que acumula valores enviados con send().
    Retorna el total al cerrarse.
    """
    total = 0.0
    while True:
        valor = yield total  # pausa y espera el siguiente send()
        if valor is None:
            break
        total += valor
    return f"Total final: {total}"  # disponible en StopIteration.value


# Uso de send():
gen = acumulador()
next(gen)          # inicializar — avanza hasta el primer yield
gen.send(10.0)     # total = 10
gen.send(25.5)     # total = 35.5
gen.send(5.0)      # total = 40.5
try:
    gen.send(None)  # señal de fin
except StopIteration as e:
    print(e.value)  # "Total final: 40.5"


# throw() — lanzar excepción dentro del generador
def generador_con_manejo() -> Generator[int, None, None]:
    """Generador que puede manejar excepciones externas."""
    try:
        for i in range(10):
            yield i
    except ValueError as e:
        print(f"Generador recibió excepción: {e}")
        yield -1  # valor de error


gen = generador_con_manejo()
print(next(gen))           # 0
print(next(gen))           # 1
print(gen.throw(ValueError, "reiniciar"))  # maneja la excepción, yield -1
```

### Generator pipelines — procesar datos en cadena sin cargar en memoria

```python
# pipeline.py
"""
Los generator pipelines procesan datos elemento a elemento.
Nunca cargan el dataset completo en memoria — ideales para
archivos grandes o streams de datos.
"""
import csv
from typing import Iterator
from pathlib import Path


# Cada función es una etapa del pipeline — recibe y produce iteradores
def leer_csv(ruta: str) -> Iterator[dict]:
    """Etapa 1: leer líneas del CSV como dicts."""
    with open(ruta, newline="", encoding="utf-8") as f:
        yield from csv.DictReader(f)


def filtrar_activos(filas: Iterator[dict]) -> Iterator[dict]:
    """Etapa 2: filtrar solo los registros activos."""
    for fila in filas:
        if fila.get("activo", "").lower() == "true":
            yield fila


def normalizar_precios(filas: Iterator[dict]) -> Iterator[dict]:
    """Etapa 3: convertir precio de string a float y aplicar IVA."""
    for fila in filas:
        try:
            precio_base = float(fila["precio"])
            yield {
                **fila,
                "precio":     precio_base,
                "precio_iva": round(precio_base * 1.16, 2),
            }
        except (ValueError, KeyError):
            continue  # saltar filas con datos inválidos


def enriquecer(filas: Iterator[dict]) -> Iterator[dict]:
    """Etapa 4: agregar campos calculados."""
    for fila in filas:
        stock = int(fila.get("stock", 0))
        yield {
            **fila,
            "disponible":    stock > 0,
            "nivel_stock":   "bajo" if stock <= 5 else "normal",
        }


def a_lotes(iterable: Iterator, tamano: int = 100) -> Iterator[list]:
    """Etapa N: agrupar elementos en lotes para inserts masivos."""
    lote = []
    for elemento in iterable:
        lote.append(elemento)
        if len(lote) >= tamano:
            yield lote
            lote = []
    if lote:
        yield lote  # último lote (posiblemente incompleto)


def procesar_productos(ruta_csv: str) -> None:
    """
    Pipeline completo — procesa un CSV de cualquier tamaño
    usando memoria constante (solo un elemento a la vez en RAM).
    """
    # Construir el pipeline — ningún generador ejecuta todavía
    filas       = leer_csv(ruta_csv)
    activos     = filtrar_activos(filas)
    con_precios = normalizar_precios(activos)
    enriquecidos = enriquecer(con_precios)
    lotes       = a_lotes(enriquecidos, tamano=50)

    # La ejecución real ocurre aquí, elemento a elemento
    total_procesados = 0
    for lote in lotes:
        # Aquí irían los inserts a la base de datos
        print(f"Insertando lote de {len(lote)} productos")
        total_procesados += len(lote)

    print(f"Total procesados: {total_procesados}")
```

---

## Protocol — structural subtyping (duck typing estático)

`Protocol` permite definir interfaces sin herencia.
Si un objeto tiene los métodos correctos, satisface el protocolo —
aunque no herede de él. El type checker (mypy/pyright) lo verifica.

```python
# protocolo.py
from typing import Protocol, runtime_checkable
from dataclasses import dataclass


# Definir el protocolo — la "interfaz" de Python moderno
@runtime_checkable
class Repositorio(Protocol):
    """Protocolo para repositorios de datos."""

    def obtener(self, id: int) -> dict | None: ...

    def guardar(self, entidad: dict) -> dict: ...

    def eliminar(self, id: int) -> bool: ...

    def listar(self, pagina: int = 1, tamano: int = 10) -> list[dict]: ...


# Implementación 1: en memoria
class RepositorioMemoria:
    """No hereda de Repositorio — solo implementa los métodos."""

    def __init__(self):
        self._datos: dict[int, dict] = {}
        self._siguiente_id = 1

    def obtener(self, id: int) -> dict | None:
        return self._datos.get(id)

    def guardar(self, entidad: dict) -> dict:
        if "id" not in entidad:
            entidad = {"id": self._siguiente_id, **entidad}
            self._siguiente_id += 1
        self._datos[entidad["id"]] = entidad
        return entidad

    def eliminar(self, id: int) -> bool:
        if id in self._datos:
            del self._datos[id]
            return True
        return False

    def listar(self, pagina: int = 1, tamano: int = 10) -> list[dict]:
        todos = list(self._datos.values())
        inicio = (pagina - 1) * tamano
        return todos[inicio:inicio + tamano]


# Implementación 2: archivo JSON
import json
from pathlib import Path


class RepositorioJSON:
    """Otra implementación del mismo protocolo — sin herencia."""

    def __init__(self, archivo: str = "datos.json"):
        self._archivo = Path(archivo)
        if not self._archivo.exists():
            self._archivo.write_text("[]")

    def _cargar(self) -> list[dict]:
        return json.loads(self._archivo.read_text())

    def _guardar_todos(self, datos: list[dict]) -> None:
        self._archivo.write_text(json.dumps(datos, indent=2))

    def obtener(self, id: int) -> dict | None:
        return next((d for d in self._cargar() if d.get("id") == id), None)

    def guardar(self, entidad: dict) -> dict:
        datos = self._cargar()
        existente = next(
            (i for i, d in enumerate(datos) if d.get("id") == entidad.get("id")),
            None
        )
        if existente is not None:
            datos[existente] = entidad
        else:
            if "id" not in entidad:
                max_id = max((d.get("id", 0) for d in datos), default=0)
                entidad = {"id": max_id + 1, **entidad}
            datos.append(entidad)
        self._guardar_todos(datos)
        return entidad

    def eliminar(self, id: int) -> bool:
        datos = self._cargar()
        nuevos = [d for d in datos if d.get("id") != id]
        if len(nuevos) < len(datos):
            self._guardar_todos(nuevos)
            return True
        return False

    def listar(self, pagina: int = 1, tamano: int = 10) -> list[dict]:
        todos  = self._cargar()
        inicio = (pagina - 1) * tamano
        return todos[inicio:inicio + tamano]


# Función que acepta CUALQUIER implementación del protocolo
def servicio_generico(repo: Repositorio) -> None:
    """
    Acepta cualquier objeto que satisfaga el protocolo Repositorio.
    No importa si es RepositorioMemoria, RepositorioJSON o cualquier otro.
    """
    producto = repo.guardar({"nombre": "Teclado", "precio": 89.99})
    print(f"Guardado con ID: {producto['id']}")

    recuperado = repo.obtener(producto["id"])
    print(f"Recuperado: {recuperado}")


# Verificar en tiempo de ejecución (por @runtime_checkable)
repo_mem = RepositorioMemoria()
repo_json = RepositorioJSON("/tmp/test_repositorio.json")

assert isinstance(repo_mem, Repositorio)   # True — satisface el protocolo
assert isinstance(repo_json, Repositorio)  # True

servicio_generico(repo_mem)
servicio_generico(repo_json)
```

### TypeVar con bounds y __slots__

```python
# tipos_avanzados.py
from typing import TypeVar, Generic


# TypeVar con bound — restricción de tipo
T = TypeVar("T")
Comparable = TypeVar("Comparable", bound="Ordenable")


class Ordenable(Protocol):
    def __lt__(self, otro: "Ordenable") -> bool: ...
    def __le__(self, otro: "Ordenable") -> bool: ...


def minimo(a: Comparable, b: Comparable) -> Comparable:
    """Retorna el mínimo — funciona con cualquier tipo Comparable."""
    return a if a < b else b


# __slots__ — optimiza memoria eliminando __dict__
class PuntoOptimizado:
    """
    __slots__ declara los atributos explícitamente.
    - Elimina el dict de instancia → ~40% menos memoria
    - Acceso a atributos más rápido
    - No puede tener atributos arbitrarios después de la creación
    """
    __slots__ = ("x", "y", "z")

    def __init__(self, x: float, y: float, z: float = 0.0) -> None:
        self.x = x
        self.y = y
        self.z = z

    def distancia(self, otro: "PuntoOptimizado") -> float:
        return ((self.x - otro.x)**2 +
                (self.y - otro.y)**2 +
                (self.z - otro.z)**2) ** 0.5

    def __repr__(self) -> str:
        return f"Punto({self.x}, {self.y}, {self.z})"


# __init_subclass__ — validar subclases en tiempo de definición
class PluginBase:
    """Clase base que valida que las subclases implementen lo requerido."""
    _plugins: dict[str, type["PluginBase"]] = {}

    def __init_subclass__(cls, nombre: str, **kwargs) -> None:
        super().__init_subclass__(**kwargs)

        # Validar que el nombre no esté duplicado
        if nombre in cls._plugins:
            raise ValueError(f"Plugin '{nombre}' ya registrado")

        # Validar que tenga el método requerido
        if not hasattr(cls, "ejecutar"):
            raise TypeError(
                f"Plugin '{nombre}' debe implementar el método 'ejecutar'"
            )

        cls._plugins[nombre] = cls
        cls.nombre_plugin = nombre
        print(f"Plugin registrado: '{nombre}'")


class PluginCSV(PluginBase, nombre="csv"):
    def ejecutar(self, datos: list) -> str:
        return ",".join(str(d) for d in datos)


class PluginJSON(PluginBase, nombre="json"):
    def ejecutar(self, datos: list) -> str:
        import json
        return json.dumps(datos)
```

---

## Ejemplo aplicado — Pipeline ETL con generadores y Strategy

Sistema completo de ETL (Extract, Transform, Load) que procesa datos
de múltiples fuentes usando generadores y el patrón Strategy.

```python
# etl_pipeline.py
"""
Pipeline ETL con:
- Generadores para procesamiento en streaming (sin cargar en RAM)
- Patrón Strategy para fuentes de datos intercambiables
- Protocol para las interfaces de cada etapa
"""
import json
import csv
import io
from typing import Iterator, Protocol
from dataclasses import dataclass


# ── Protocolos de las etapas ──────────────────────────────────────

class FuenteDatos(Protocol):
    """Estrategia de extracción — de dónde vienen los datos."""
    def extraer(self) -> Iterator[dict]: ...
    def nombre(self) -> str: ...


class Transformacion(Protocol):
    """Una transformación individual en el pipeline."""
    def transformar(self, filas: Iterator[dict]) -> Iterator[dict]: ...


class Destino(Protocol):
    """Estrategia de carga — a dónde van los datos."""
    def cargar(self, filas: Iterator[dict]) -> int: ...


# ── Fuentes de datos (Extract) ────────────────────────────────────

class FuenteCSV:
    """Leer datos de un archivo CSV."""
    def __init__(self, contenido: str):
        self._contenido = contenido

    def extraer(self) -> Iterator[dict]:
        reader = csv.DictReader(io.StringIO(self._contenido))
        yield from reader

    def nombre(self) -> str:
        return "CSV"


class FuenteJSON:
    """Leer datos de una cadena JSON (lista de objetos)."""
    def __init__(self, json_str: str):
        self._datos = json.loads(json_str)

    def extraer(self) -> Iterator[dict]:
        yield from self._datos

    def nombre(self) -> str:
        return "JSON"


class FuenteGenerada:
    """Generar datos sintéticos — útil para tests y demos."""
    def __init__(self, cantidad: int = 100):
        self._cantidad = cantidad

    def extraer(self) -> Iterator[dict]:
        for i in range(1, self._cantidad + 1):
            yield {
                "id":      str(i),
                "nombre":  f"Producto {i}",
                "precio":  f"{9.99 + i * 10:.2f}",
                "stock":   str(i % 50),
                "activo":  "true" if i % 3 != 0 else "false",
            }

    def nombre(self) -> str:
        return "generada"


# ── Transformaciones (Transform) ─────────────────────────────────

class NormalizarTipos:
    """Convertir strings a sus tipos correctos."""
    def transformar(self, filas: Iterator[dict]) -> Iterator[dict]:
        for fila in filas:
            yield {
                **fila,
                "precio": float(fila.get("precio", 0)),
                "stock":  int(fila.get("stock", 0)),
                "activo": fila.get("activo", "false").lower() == "true",
                "id":     int(fila.get("id", 0)),
            }


class FiltrarInactivos:
    """Excluir registros inactivos."""
    def transformar(self, filas: Iterator[dict]) -> Iterator[dict]:
        for fila in filas:
            if fila.get("activo", False):
                yield fila


class AgregarCamposCalculados:
    """Enriquecer con campos derivados."""
    IVA = 0.16

    def transformar(self, filas: Iterator[dict]) -> Iterator[dict]:
        for fila in filas:
            precio = fila.get("precio", 0.0)
            stock  = fila.get("stock",  0)
            yield {
                **fila,
                "precio_con_iva": round(precio * (1 + self.IVA), 2),
                "valor_inventario": round(precio * stock, 2),
                "nivel_stock": (
                    "agotado" if stock == 0
                    else "bajo"    if stock <= 5
                    else "normal"  if stock <= 20
                    else "alto"
                ),
            }


# ── Destinos (Load) ───────────────────────────────────────────────

class DestinoMemoria:
    """Cargar resultados en una lista en memoria."""
    def __init__(self):
        self.resultados: list[dict] = []

    def cargar(self, filas: Iterator[dict]) -> int:
        count = 0
        for fila in filas:
            self.resultados.append(fila)
            count += 1
        return count


class DestinoConsola:
    """Imprimir resultados — útil para depuración."""
    def __init__(self, max_filas: int = 10):
        self._max = max_filas

    def cargar(self, filas: Iterator[dict]) -> int:
        count = 0
        for fila in filas:
            if count < self._max:
                campos = ["id", "nombre", "precio", "nivel_stock"]
                print({k: fila[k] for k in campos if k in fila})
            count += 1
        if count > self._max:
            print(f"... y {count - self._max} filas más")
        return count


# ── Pipeline orquestador ──────────────────────────────────────────

@dataclass
class PipelineETL:
    """Orquesta las tres fases: Extract → Transform → Load."""
    fuente:          FuenteDatos
    transformaciones: list[Transformacion]
    destino:         Destino

    def ejecutar(self) -> dict:
        """Ejecutar el pipeline completo y retornar estadísticas."""
        import time
        inicio = time.perf_counter()

        # 1. Extraer
        datos = self.fuente.extraer()

        # 2. Aplicar transformaciones en cadena
        for transformacion in self.transformaciones:
            datos = transformacion.transformar(datos)

        # 3. Cargar
        total = self.destino.cargar(datos)

        duracion = time.perf_counter() - inicio
        return {
            "fuente":    self.fuente.nombre(),
            "registros": total,
            "duracion_ms": round(duracion * 1000, 2),
        }


# ── Ejecutar el pipeline ──────────────────────────────────────────

if __name__ == "__main__":
    # Pipeline 1: leer datos generados y transformar
    destino_mem = DestinoMemoria()

    pipeline = PipelineETL(
        fuente=FuenteGenerada(cantidad=500),
        transformaciones=[
            NormalizarTipos(),
            FiltrarInactivos(),
            AgregarCamposCalculados(),
        ],
        destino=destino_mem,
    )

    stats = pipeline.ejecutar()
    print(f"Pipeline completado: {stats}")
    print(f"Primeros 3 resultados:")
    for r in destino_mem.resultados[:3]:
        print(f"  {r['nombre']}: ${r['precio']} "
              f"(con IVA: ${r['precio_con_iva']}) "
              f"[{r['nivel_stock']}]")

    # Pipeline 2: cambiar la fuente a JSON sin modificar nada más
    datos_json = json.dumps([
        {"id": "1", "nombre": "Monitor", "precio": "349.99",
         "stock": "3", "activo": "true"},
        {"id": "2", "nombre": "Webcam",  "precio":  "79.99",
         "stock": "0", "activo": "true"},
    ])

    destino_consola = DestinoConsola()
    pipeline2 = PipelineETL(
        fuente=FuenteJSON(datos_json),
        transformaciones=[
            NormalizarTipos(),
            FiltrarInactivos(),
            AgregarCamposCalculados(),
        ],
        destino=destino_consola,
    )
    stats2 = pipeline2.ejecutar()
    print(f"\nPipeline JSON: {stats2}")
```

---

## Ejercicios propuestos

1. **Command pattern para operaciones invertibles** — Implementa el patrón Command para un editor de texto en memoria. Define el protocolo `Comando` con métodos `ejecutar()` y `deshacer()`. Crea los comandos: `ComandoInsertar(posicion, texto)`, `ComandoEliminar(posicion, longitud)` y `ComandoReemplazar(posicion, longitud, texto_nuevo)`. Crea la clase `EditorTexto` con una pila de comandos ejecutados y métodos `ejecutar(cmd)` y `deshacer()`. Escribe tests que verifiquen que después de ejecutar y deshacer 5 operaciones el texto vuelve al estado original.

2. **Generator pipeline para logs** — Crea un pipeline de procesamiento de logs que: (a) lea líneas de un archivo de log de Nginx; (b) parsee cada línea con una expresión regular extrayendo: IP, timestamp, método, ruta, status code, bytes; (c) filtre solo los errores 4xx y 5xx; (d) agrupe por ruta y cuente ocurrencias; (e) emita alertas para rutas con más de 10 errores. Cada etapa debe ser un generador. El pipeline debe procesar un archivo de 100,000 líneas sin cargarlo entero en memoria.

3. **Protocol con múltiples implementaciones** — Define el protocolo `CacheBackend` con métodos `get(clave)`, `set(clave, valor, ttl_seg)` y `delete(clave)`. Implementa tres backends: `CacheMemoria` (dict + expiración manual), `CacheRedis` (mockeado con dict en tests) y `CacheNulo` (siempre retorna None, útil para deshabilitar cache). Crea la función `con_cache(backend: CacheBackend)` que retorna un decorator. Escribe tests que verifiquen que los tres backends son intercambiables.

4. **Builder para consultas HTTP** — Crea `ConstructorPeticion` con interfaz fluida para construir peticiones HTTP: `.get(url)`, `.post(url)`, `.con_header(clave, valor)`, `.con_parametro(clave, valor)`, `.con_token_bearer(token)`, `.con_timeout(segundos)`, `.con_cuerpo(datos: dict)`, `.construir()`. El método `construir()` debe retornar un objeto `Peticion` con todos los datos validados. Agrega un método `.ejecutar()` que use `httpx` para hacer la petición real. Escribe tests unitarios mockeando `httpx` para verificar que los parámetros se pasan correctamente.

---

## Resumen de la página 17

- El **Singleton** más idiomático en Python es un módulo: se importa una sola vez. La versión con `__new__` es útil cuando necesitas lazy initialization o herencia.
- El **Factory** centraliza la creación de objetos y permite registrar nuevos tipos sin modificar el código existente — el principio Open/Closed en acción.
- El **Builder** con method chaining (`return self` en cada método) permite construir objetos complejos de forma legible y paso a paso.
- `functools.wraps` es obligatorio en decorators para preservar `__name__`, `__doc__` y la firma de la función original — sin él, las herramientas de debugging y documentación fallan.
- `yield from` delega la iteración a otro generador o iterable; en generators pipelines, cada etapa recibe y produce un iterador — el procesamiento es completamente lazy.
- `send()` convierte un generador en un canal de comunicación bidireccional; el valor enviado es el resultado del `yield` en el generador receptor.
- `Protocol` es la forma moderna de duck typing estático — define una interfaz sin herencia. `@runtime_checkable` permite usar `isinstance()` con protocolos.
- `__slots__` elimina el diccionario de instancia de cada objeto, reduciendo el uso de memoria en ~40% — ideal para clases que se instancian millones de veces.
- El patrón **Strategy** con `Protocol` permite cambiar el algoritmo en tiempo de ejecución pasando una implementación diferente — sin condicionales en el contexto.

---

> **Siguiente página →** Página 21: Clean Architecture en Python.
