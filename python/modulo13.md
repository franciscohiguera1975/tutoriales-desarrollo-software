# Tutorial Python — Página 13
## Módulo 5 · Robustez
### Type hints, mypy y Pydantic v2

---

## Parte 1 — Type hints básicos

Python es dinámico, pero desde Python 3.5 permite anotar tipos. En 3.10–3.12
la sintaxis se simplifica significativamente. Los type hints no afectan al
rendimiento en tiempo de ejecución: son solo documentación y entrada para
herramientas de análisis estático como mypy o Pyright.

```python
# ─────────────────────────────────────────────────────────────────────────────
# Tipos primitivos y None
# ─────────────────────────────────────────────────────────────────────────────
nombre: str = "Ana"
edad: int = 30
precio: float = 49.99
activo: bool = True
sin_valor: None = None   # variable que siempre es None (raro, pero válido)

def saludar(nombre: str) -> str:
    """Los parámetros y el retorno se anotan con ->"""
    return f"Hola, {nombre}"

def registrar_evento(mensaje: str, nivel: str = "INFO") -> None:
    """None como retorno indica que la función no devuelve nada útil."""
    print(f"[{nivel}] {mensaje}")


# ─────────────────────────────────────────────────────────────────────────────
# Colecciones — usar la sintaxis moderna de Python 3.9+ (minúsculas)
# ─────────────────────────────────────────────────────────────────────────────
# Python 3.9+: list[int] en lugar de List[int] de typing
# Python 3.10+: int | None en lugar de Optional[int]

nombres: list[str] = ["Ana", "Luis", "María"]
precios: dict[str, float] = {"teclado": 49.99, "ratón": 24.99}
coordenadas: tuple[float, float] = (40.41, -3.70)   # tupla de longitud fija
tags: set[str] = {"python", "backend", "api"}

# Tupla de longitud variable (homogénea)
puntuaciones: tuple[int, ...] = (10, 20, 30, 40)

# Dict anidado
config: dict[str, dict[str, int | str]] = {
    "bd": {"host": "localhost", "puerto": 5432},
}


# ─────────────────────────────────────────────────────────────────────────────
# Optional y Union — Python 3.10+ usa el operador |
# ─────────────────────────────────────────────────────────────────────────────
# Antes (aún válido):
from typing import Optional, Union
def buscar_usuario_v1(id_: int) -> Optional[dict]:
    ...

# Python 3.10+ (preferido):
def buscar_usuario(id_: int) -> dict | None:
    """Devuelve el usuario o None si no existe."""
    ...

def procesar(valor: int | str | float) -> str:
    """Acepta múltiples tipos."""
    return str(valor)


# ─────────────────────────────────────────────────────────────────────────────
# Any — deshabilita la verificación de tipos para ese valor
# Usar con moderación; indica código que aún no está tipado
# ─────────────────────────────────────────────────────────────────────────────
from typing import Any

def parsear_dato_externo(raw: Any) -> str:
    """raw puede ser cualquier cosa que viene de una fuente externa."""
    return str(raw)
```

---

## Parte 2 — Tipos avanzados

```python
from typing import Callable, TypeVar, Generic, TypedDict, Protocol, Literal, Final, ClassVar
from collections.abc import Sequence, Mapping, Iterator, Generator

# ─────────────────────────────────────────────────────────────────────────────
# Callable — tipo de una función
# ─────────────────────────────────────────────────────────────────────────────
# Callable[[tipo_arg1, tipo_arg2], tipo_retorno]
TrasformadorStr = Callable[[str], str]
Predicado = Callable[[dict], bool]
Manejador = Callable[[Exception], None]

def aplicar_transformacion(texto: str, fn: TrasformadorStr) -> str:
    return fn(texto)

resultado = aplicar_transformacion("hola mundo", str.upper)
print(resultado)   # HOLA MUNDO


# ─────────────────────────────────────────────────────────────────────────────
# TypeVar y Generic — funciones y clases genéricas
# ─────────────────────────────────────────────────────────────────────────────
T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

def primero(elementos: list[T]) -> T | None:
    """Devuelve el primer elemento de cualquier lista, o None si está vacía."""
    return elementos[0] if elementos else None

# Clase genérica
class Pila(Generic[T]):
    """Pila LIFO genérica que funciona con cualquier tipo."""

    def __init__(self) -> None:
        self._items: list[T] = []

    def apilar(self, item: T) -> None:
        self._items.append(item)

    def desapilar(self) -> T:
        if not self._items:
            raise IndexError("La pila está vacía")
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

    def __len__(self) -> int:
        return len(self._items)


pila_enteros: Pila[int] = Pila()
pila_enteros.apilar(1)
pila_enteros.apilar(2)
print(pila_enteros.desapilar())   # 2


# ─────────────────────────────────────────────────────────────────────────────
# TypedDict — diccionario con tipos por clave
# Alternativa ligera a dataclass cuando se trabaja con dicts JSON
# ─────────────────────────────────────────────────────────────────────────────
class UsuarioDict(TypedDict):
    id: int
    nombre: str
    email: str
    activo: bool

class UsuarioDictOpcional(TypedDict, total=False):
    # total=False hace que todos los campos sean opcionales
    telefono: str
    avatar_url: str

class UsuarioCompleto(UsuarioDict, UsuarioDictOpcional):
    pass  # hereda campos requeridos y opcionales

usuario: UsuarioCompleto = {
    "id": 1,
    "nombre": "Ana",
    "email": "ana@ejemplo.com",
    "activo": True,
    # telefono es opcional
}


# ─────────────────────────────────────────────────────────────────────────────
# Protocol — duck typing estático (structural subtyping)
# Una clase satisface un Protocol si tiene los métodos indicados,
# sin necesidad de heredar explícitamente
# ─────────────────────────────────────────────────────────────────────────────
from typing import runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    """Cualquier objeto con un método to_dict() satisface este protocolo."""
    def to_dict(self) -> dict[str, Any]:
        ...

class Producto:
    def __init__(self, nombre: str, precio: float) -> None:
        self.nombre = nombre
        self.precio = precio

    def to_dict(self) -> dict[str, Any]:
        return {"nombre": self.nombre, "precio": self.precio}

class Pedido:
    def __init__(self, id_: int) -> None:
        self.id_ = id_

    def to_dict(self) -> dict[str, Any]:
        return {"id": self.id_}

def serializar(obj: Serializable) -> str:
    """Funciona con cualquier objeto que tenga to_dict(), sin herencia."""
    import json
    return json.dumps(obj.to_dict())

# Tanto Producto como Pedido satisfacen Serializable sin heredar de él
print(serializar(Producto("Teclado", 49.99)))
print(isinstance(Producto("x", 1.0), Serializable))   # True (con @runtime_checkable)


# ─────────────────────────────────────────────────────────────────────────────
# Literal, Final y ClassVar
# ─────────────────────────────────────────────────────────────────────────────
Direccion = Literal["norte", "sur", "este", "oeste"]
NivelLog = Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]

def mover(direccion: Direccion, pasos: int) -> None:
    print(f"Moviendo {pasos} paso(s) hacia el {direccion}")

# Final — variable que no debe reasignarse
MAXIMO_REINTENTOS: Final = 3
URL_BASE: Final[str] = "https://api.ejemplo.com/v1"

class Configuracion:
    # ClassVar — atributo de clase, no de instancia
    _instancias: ClassVar[int] = 0

    def __init__(self) -> None:
        Configuracion._instancias += 1
```

---

## Parte 3 — TYPE_CHECKING y mypy

```python
# ─────────────────────────────────────────────────────────────────────────────
# TYPE_CHECKING — evitar imports circulares en type hints
# El bloque solo se ejecuta cuando mypy analiza el código, no en runtime
# ─────────────────────────────────────────────────────────────────────────────
from __future__ import annotations   # permite referencias hacia adelante en strings
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # Estos imports solo existen para mypy, no se ejecutan en producción
    from mi_app.modelos.pedido import Pedido
    from mi_app.servicios.auth import SesionUsuario

class ServicioPedidos:
    def procesar(self, pedido: "Pedido") -> None:   # comillas por forward reference
        ...

    def con_sesion(self, sesion: "SesionUsuario") -> None:
        ...
```

```toml
# ─────────────────────────────────────────────────────────────────────────────
# pyproject.toml — configuración de mypy
# ─────────────────────────────────────────────────────────────────────────────
[tool.mypy]
python_version = "3.12"
strict = true                     # habilita todas las comprobaciones estrictas

# Opciones individuales que incluye --strict:
# warn_return_any = true
# warn_unused_ignores = true
# disallow_untyped_defs = true
# disallow_incomplete_defs = true
# check_untyped_defs = true
# no_implicit_optional = true

ignore_missing_imports = true     # ignorar paquetes sin stubs
exclude = ["tests/", "scripts/"]  # excluir directorios

# Relajar reglas para módulos específicos (terceros sin tipos)
[[tool.mypy.overrides]]
module = ["sqlalchemy.*", "alembic.*"]
ignore_missing_imports = true
```

```bash
# Ejecutar mypy
mypy mi_app/

# Ver solo errores (sin notas)
mypy mi_app/ --no-notes

# Verificar un archivo concreto
mypy mi_app/servicios/auth.py

# Generar informe HTML
mypy mi_app/ --html-report reporte_tipos/
```

---

## Parte 4 — Pydantic v2

Pydantic es la librería de validación de datos más popular de Python. La versión 2
(2.0+) es hasta 50x más rápida que v1 gracias a su núcleo escrito en Rust.

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from pydantic import EmailStr, AnyHttpUrl
from datetime import datetime
from decimal import Decimal
from typing import Annotated

# ─────────────────────────────────────────────────────────────────────────────
# BaseModel básico — Pydantic valida y convierte automáticamente
# ─────────────────────────────────────────────────────────────────────────────
class Usuario(BaseModel):
    id: int
    nombre: str
    email: str   # en la vida real: EmailStr (requiere pip install pydantic[email])
    activo: bool = True
    fecha_alta: datetime | None = None

# Pydantic convierte tipos automáticamente (coerción)
u = Usuario(id="1", nombre="Ana", email="ana@ejemplo.com")
print(u.id)           # 1 (int, no "1")
print(u.activo)       # True
print(u.model_dump()) # {'id': 1, 'nombre': 'Ana', 'email': 'ana@ejemplo.com', ...}

# Errores de validación — lanza ValidationError con detalles
from pydantic import ValidationError
try:
    Usuario(id="no_es_numero", nombre="Ana", email="ana@ejemplo.com")
except ValidationError as e:
    print(e.error_count())   # número de errores
    for error in e.errors():
        print(f"  Campo: {error['loc']}, tipo: {error['type']}, msg: {error['msg']}")


# ─────────────────────────────────────────────────────────────────────────────
# Field() — control avanzado de cada campo
# ─────────────────────────────────────────────────────────────────────────────
StrNoVacia = Annotated[str, Field(min_length=1, max_length=200)]

class Producto(BaseModel):
    nombre: StrNoVacia                               # tipo reutilizable con Annotated
    precio: Decimal = Field(gt=0, decimal_places=2)  # mayor que 0, 2 decimales
    stock: int = Field(default=0, ge=0)              # mayor o igual a 0
    sku: str = Field(pattern=r"^SKU-[A-Z0-9]{3,10}$")  # regex
    descripcion: str | None = Field(default=None, max_length=500)
    # alias — usar nombre diferente en JSON/dict de entrada
    precio_iva: float = Field(alias="price_with_vat", default=0.0)

    class Config:
        # populate_by_name = True  # aceptar tanto el alias como el nombre Python
        frozen = True   # instancias inmutables (como dataclass frozen)


# ─────────────────────────────────────────────────────────────────────────────
# field_validator — validación personalizada por campo
# ─────────────────────────────────────────────────────────────────────────────
class Pedido(BaseModel):
    id_pedido: str
    email_cliente: str
    lineas: list[dict]   # simplificado para el ejemplo
    descuento_pct: float = Field(default=0.0, ge=0.0, le=100.0)

    @field_validator("email_cliente")
    @classmethod
    def validar_email(cls, v: str) -> str:
        """Normaliza el email y verifica su formato básico."""
        v = v.lower().strip()
        if "@" not in v or "." not in v.split("@")[1]:
            raise ValueError(f"Email inválido: {v!r}")
        return v

    @field_validator("id_pedido")
    @classmethod
    def validar_id_pedido(cls, v: str) -> str:
        """El ID debe seguir el formato PED-YYYY-NNNNNN."""
        import re
        if not re.match(r"^PED-\d{4}-\d{1,6}$", v):
            raise ValueError(
                f"Formato de ID inválido: {v!r}. Se esperaba PED-YYYY-NNNNNN"
            )
        return v.upper()


# ─────────────────────────────────────────────────────────────────────────────
# model_validator — validación que involucra múltiples campos
# ─────────────────────────────────────────────────────────────────────────────
class RangoFechas(BaseModel):
    fecha_inicio: datetime
    fecha_fin: datetime
    descripcion: str = ""

    @model_validator(mode="after")
    def validar_rango(self) -> "RangoFechas":
        """Verifica que fecha_fin sea posterior a fecha_inicio."""
        if self.fecha_fin <= self.fecha_inicio:
            raise ValueError(
                f"fecha_fin ({self.fecha_fin}) debe ser posterior a "
                f"fecha_inicio ({self.fecha_inicio})"
            )
        return self


# ─────────────────────────────────────────────────────────────────────────────
# Serialización y deserialización JSON
# ─────────────────────────────────────────────────────────────────────────────
class EventoAPI(BaseModel):
    id: int
    nombre: str
    timestamp: datetime
    datos: dict[str, Any]

# Deserializar desde JSON
json_str = '{"id": 1, "nombre": "login", "timestamp": "2024-01-15T10:30:00", "datos": {}}'
evento = EventoAPI.model_validate_json(json_str)
print(evento.timestamp)   # datetime(2024, 1, 15, 10, 30, 0)

# Serializar a JSON
json_output = evento.model_dump_json(indent=2)

# Serializar a dict
dict_output = evento.model_dump()
dict_iso = evento.model_dump(mode="json")   # convierte datetime a ISO string

# Copiar con cambios
evento_modificado = evento.model_copy(update={"nombre": "logout"})
```

---

## Ejemplo aplicado — Sistema de configuración con Pydantic

Carga variables de entorno, las valida con Pydantic y expone una configuración
tipada disponible en toda la aplicación.

```python
"""
config.py
Sistema de configuración tipada que carga .env, valida con Pydantic
y expone toda la configuración de la aplicación como un singleton.
"""
from __future__ import annotations

import os
from functools import lru_cache
from typing import Annotated, Any, Literal

from pydantic import (
    AnyHttpUrl,
    BaseModel,
    Field,
    PostgresDsn,
    RedisDsn,
    field_validator,
    model_validator,
)
from pydantic_settings import BaseSettings, SettingsConfigDict


# ── Sub-configuraciones ───────────────────────────────────────────────────────
class ConfigBase_Datos(BaseModel):
    """Configuración de la base de datos PostgreSQL."""
    url: PostgresDsn                              # valida que sea una URL PostgreSQL
    pool_size: int = Field(default=10, ge=1, le=100)
    max_overflow: int = Field(default=5, ge=0, le=50)
    pool_timeout: int = Field(default=30, ge=1)
    echo_sql: bool = False                        # loguear las queries SQL (solo dev)

    @property
    def url_str(self) -> str:
        return str(self.url)


class ConfigRedis(BaseModel):
    """Configuración de Redis para caché y colas."""
    url: RedisDsn = Field(default="redis://localhost:6379/0")   # type: ignore[assignment]
    timeout_conexion: int = Field(default=5, ge=1)
    max_conexiones: int = Field(default=50, ge=1)


class ConfigSeguridad(BaseModel):
    """Configuración de seguridad y autenticación JWT."""
    clave_secreta: str = Field(min_length=32)
    algoritmo_jwt: Literal["HS256", "HS384", "HS512", "RS256"] = "HS256"
    duracion_token_minutos: int = Field(default=60, ge=1, le=1440)  # máx 24h
    duracion_refresh_dias: int = Field(default=7, ge=1, le=30)
    cors_origenes: list[str] = Field(default_factory=list)

    @field_validator("clave_secreta")
    @classmethod
    def validar_clave_secreta(cls, v: str) -> str:
        """En producción la clave debe ser suficientemente compleja."""
        if v == "cambiar_en_produccion":
            raise ValueError(
                "La clave secreta no puede ser el valor por defecto. "
                "Genera una clave segura con: python -c \"import secrets; print(secrets.token_hex(32))\""
            )
        return v


class ConfigAPI(BaseModel):
    """Configuración del servidor HTTP."""
    host: str = "0.0.0.0"
    puerto: int = Field(default=8000, ge=1, le=65535)
    workers: int = Field(default=1, ge=1)
    timeout_peticion: int = Field(default=30, ge=1)
    max_tamano_body_mb: int = Field(default=10, ge=1)

    @property
    def max_tamano_body_bytes(self) -> int:
        return self.max_tamano_body_mb * 1024 * 1024


# ── Configuración principal usando pydantic-settings ─────────────────────────
# pydantic-settings carga automáticamente desde variables de entorno y .env
class Configuracion(BaseSettings):
    """
    Configuración completa de la aplicación.
    Los valores se cargan en este orden (mayor prioridad primero):
    1. Variables de entorno del sistema
    2. Archivo .env
    3. Valores por defecto en el modelo
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_nested_delimiter="__",   # BD__URL → bd.url (anidado)
        case_sensitive=False,
        extra="ignore",              # ignorar variables de entorno desconocidas
    )

    # ── Entorno ────────────────────────────────────────────────────────────
    entorno: Literal["desarrollo", "staging", "produccion"] = "desarrollo"
    debug: bool = False
    nombre_app: str = "Mi Aplicación"
    version: str = "0.1.0"

    # ── Sub-configuraciones (se cargan con BD__URL, REDIS__URL, etc.) ─────
    bd: ConfigBase_Datos
    redis: ConfigRedis = Field(default_factory=ConfigRedis)   # type: ignore[call-arg]
    seguridad: ConfigSeguridad
    api: ConfigAPI = Field(default_factory=ConfigAPI)   # type: ignore[call-arg]

    # ── Servicios externos ─────────────────────────────────────────────────
    url_servicio_email: AnyHttpUrl | None = None
    url_servicio_pagos: AnyHttpUrl | None = None
    api_key_servicio_pagos: str | None = None

    @model_validator(mode="after")
    def validar_produccion(self) -> "Configuracion":
        """Reglas adicionales que solo aplican en producción."""
        if self.entorno == "produccion":
            if self.debug:
                raise ValueError("debug=True no está permitido en producción")
            if self.api.workers < 2:
                raise ValueError("Se recomienda al menos 2 workers en producción")
            if self.url_servicio_email is None:
                raise ValueError("url_servicio_email es obligatorio en producción")
        return self

    @property
    def es_desarrollo(self) -> bool:
        return self.entorno == "desarrollo"

    @property
    def es_produccion(self) -> bool:
        return self.entorno == "produccion"

    def resumen(self) -> dict[str, Any]:
        """Devuelve un resumen de la configuración sin datos sensibles."""
        return {
            "app": self.nombre_app,
            "version": self.version,
            "entorno": self.entorno,
            "debug": self.debug,
            "api_host": self.api.host,
            "api_puerto": self.api.puerto,
            "api_workers": self.api.workers,
            "bd_pool_size": self.bd.pool_size,
        }


# ── Singleton de configuración ────────────────────────────────────────────────
@lru_cache(maxsize=1)
def obtener_config() -> Configuracion:
    """
    Devuelve la instancia única de configuración.
    lru_cache garantiza que solo se crea una vez durante la vida del proceso.
    """
    return Configuracion()   # type: ignore[call-arg]


# ── Uso en la aplicación ──────────────────────────────────────────────────────
# En cualquier módulo:
# from mi_app.config import obtener_config
#
# config = obtener_config()
# print(config.bd.url_str)
# print(config.es_produccion)
# print(config.resumen())
#
# En tests (sobreescribir valores):
# from pydantic_settings import BaseSettings
# from unittest.mock import patch
#
# with patch.dict(os.environ, {"ENTORNO": "desarrollo", "DEBUG": "true"}):
#     config_test = Configuracion()   # type: ignore[call-arg]


# ── Ejemplo de archivo .env compatible ────────────────────────────────────────
# ENTORNO=produccion
# DEBUG=false
# NOMBRE_APP=Mi API de Producción
# VERSION=1.2.0
#
# BD__URL=postgresql+psycopg://usuario:contraseña@db.ejemplo.com:5432/mi_bd
# BD__POOL_SIZE=20
# BD__ECHO_SQL=false
#
# REDIS__URL=redis://cache.ejemplo.com:6379/0
#
# SEGURIDAD__CLAVE_SECRETA=a1b2c3d4e5f6...aqui_va_la_clave_real
# SEGURIDAD__DURACION_TOKEN_MINUTOS=30
# SEGURIDAD__CORS_ORIGENES=["https://app.ejemplo.com","https://admin.ejemplo.com"]
#
# API__HOST=0.0.0.0
# API__PUERTO=8000
# API__WORKERS=4
#
# URL_SERVICIO_EMAIL=https://email.ejemplo.com
# URL_SERVICIO_PAGOS=https://pagos.ejemplo.com
# API_KEY_SERVICIO_PAGOS=pk_live_...
```

---

## Ejercicios propuestos

1. **API de usuarios con Pydantic** — Crea un sistema de modelos Pydantic para una
   API REST de usuarios: `UsuarioCrear` (nombre, email, contraseña — validar fortaleza
   de contraseña), `UsuarioActualizar` (todos los campos opcionales), `UsuarioRespuesta`
   (sin contraseña, con fecha de alta). Implementa un `model_validator` que verifique
   que la contraseña tiene al menos una mayúscula, un número y un carácter especial.

2. **Parser de configuración tipado** — Usando `TypedDict` y `Protocol`, implementa un
   sistema de configuración que pueda cargarse desde JSON, TOML o variables de entorno.
   El `Protocol` debe tener un método `cargar() -> dict[str, Any]`. Implementa tres
   clases que lo satisfagan: `CargadorJSON`, `CargadorTOML`, `CargadorEnv`. Usa mypy
   para verificar que todo está correctamente tipado.

3. **Contenedor genérico con tipos** — Implementa una clase `Resultado[T, E]` genérica
   que represente el resultado de una operación que puede tener éxito (`T`) o fallar
   (`E`). Incluye métodos `map(fn)` (transforma el valor si es éxito), `map_error(fn)`
   (transforma el error si es fallo), `unwrap()` (devuelve el valor o lanza la excepción),
   `unwrap_or(default)` (devuelve el valor o el default). Tipa todo correctamente.

4. **Validador de esquema JSON con Pydantic** — Crea un sistema que reciba un esquema
   en formato dict y construya dinámicamente un modelo Pydantic para validarlo usando
   `create_model`. Soporta campos de tipo `str`, `int`, `float`, `bool`, con opción de
   marcar campos como requeridos u opcionales. Escribe tests con `pytest` que verifiquen
   casos válidos e inválidos.

---

## Resumen de la página 10

- Los type hints no afectan al rendimiento en runtime: son documentación para herramientas como mypy, Pyright y editores
- Python 3.10+ permite `int | None` en lugar de `Optional[int]` y `list[int]` en lugar de `List[int]`; preferir la sintaxis moderna
- `TypeVar` y `Generic[T]` permiten clases y funciones genéricas que funcionan con cualquier tipo preservando la información de tipos
- `TypedDict` anota diccionarios con claves y tipos fijos; es más ligero que un dataclass cuando se trabaja con datos JSON
- `Protocol` permite duck typing estático: una clase satisface un Protocol si tiene los métodos indicados, sin herencia explícita
- `mypy --strict` activa todas las comprobaciones; `TYPE_CHECKING` permite importar tipos sin ejecutar el código en runtime para evitar imports circulares
- Pydantic v2 valida y convierte automáticamente los tipos al instanciar modelos; `field_validator` valida campos individuales y `model_validator` valida relaciones entre campos
- `pydantic-settings` combina Pydantic con carga de `.env` y variables de entorno; `@lru_cache` convierte la función de configuración en un singleton eficiente

---

> **Siguiente página →** Página 14: Concurrencia — threading, multiprocessing y asyncio.
