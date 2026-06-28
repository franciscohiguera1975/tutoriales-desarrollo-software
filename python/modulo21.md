# Tutorial Python — Página 21
## Módulo 12 · Arquitectura
### Clean Architecture en Python — capas, repositorios y casos de uso

---

## ¿Qué es Clean Architecture?

Clean Architecture divide la aplicación en capas con una regla fundamental:
**las dependencias solo apuntan hacia adentro**. Las capas internas
no saben nada de las capas externas:

```
┌──────────────────────────────────────────────────────────┐
│  Application (Presentación)                              │
│  FastAPI routers, schemas Pydantic, dependencias         │
├──────────────────────────────────────────────────────────┤
│  Domain (Negocio)                                        │
│  entidades, repositorios (Protocol), casos de uso        │
│  — SIN imports de FastAPI, SIN imports de httpx          │
├──────────────────────────────────────────────────────────┤
│  Data (Datos)                                            │
│  DTOs, implementación de repositorios, httpx, BD         │
└──────────────────────────────────────────────────────────┘

Regla: Data conoce Domain. Application conoce Domain.
       Nadie conoce Application desde adentro.
```

### Comparativa con el enfoque directo

```
Sin arquitectura (módulos 1-14)    Clean Architecture (módulo 18)
─────────────────────────────────  ─────────────────────────────────
Lógica mezclada en el router       Router solo llama casos de uso
DTO mezclado con dominio           DTO ≠ Entidad de dominio
HTTP client acoplado al router     Repositorio implementa protocolo
Sin casos de uso                   Cada operación = un caso de uso
Difícil de testear                 Cada capa se testea por separado
```

---

## Proyecto: `productos_api`

Construiremos una API que consume la misma fuente de datos
del tutorial de Flutter — la API de productos real:

```
GET https://higuera-billing-api.desarrollo-software.xyz/api/products/
    ?page=1&page_size=10
```

```bash
# Crear el proyecto
mkdir productos_api
cd productos_api

# Crear entorno virtual
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# Instalar dependencias
pip install fastapi uvicorn httpx pydantic
pip install --group dev pytest pytest-asyncio pytest-cov httpx
```

### pyproject.toml

```toml
# pyproject.toml
[build-system]
requires      = ["setuptools>=68"]
build-backend = "setuptools.backends.legacy:build"

[project]
name        = "productos-api"
version     = "0.1.0"
requires-python = ">=3.12"

dependencies = [
    "fastapi>=0.111",
    "uvicorn[standard]>=0.29",
    "httpx>=0.27",
    "pydantic>=2.7",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.2",
    "pytest-asyncio>=0.23",
    "pytest-cov>=5.0",
    "ruff>=0.4",
]

[tool.pytest.ini_options]
testpaths    = ["tests"]
asyncio_mode = "auto"
addopts      = "-v --tb=short"

[tool.coverage.run]
source = ["src"]
omit   = ["*/tests/*", "*/__init__.py"]

[tool.coverage.report]
fail_under   = 80
show_missing = true

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B"]
```

---

## Estructura del proyecto

```
productos_api/
├── pyproject.toml
├── src/
│   └── productos_api/
│       ├── __init__.py
│       │
│       ├── domain/                        ← CAPA DE DOMINIO
│       │   ├── __init__.py
│       │   ├── entities.py                ← entidad Producto (Pydantic)
│       │   ├── repositories.py            ← protocolo del repositorio
│       │   └── usecases.py                ← casos de uso
│       │
│       ├── data/                          ← CAPA DE DATOS
│       │   ├── __init__.py
│       │   ├── dtos.py                    ← DTOs que mapean el JSON de la API
│       │   └── repositories.py            ← implementación con httpx
│       │
│       └── application/                   ← CAPA DE APLICACIÓN
│           ├── __init__.py
│           ├── schemas.py                 ← schemas de respuesta Pydantic
│           ├── dependencies.py            ← inyección de dependencias
│           ├── routers.py                 ← FastAPI routers
│           └── main.py                    ← app FastAPI + lifespan
│
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_domain.py                     ← tests de entidades y casos de uso
    ├── test_data.py                       ← tests de DTOs y repositorio
    └── test_api.py                        ← tests de integración de la API
```

---

## Paso 1 — Capa Domain

La capa de dominio es Python puro: sin FastAPI, sin httpx, sin BD.
Se puede reutilizar en cualquier contexto.

### `src/productos_api/domain/entities.py`

```python
# src/productos_api/domain/entities.py
"""
Entidades de dominio — representan el negocio, no el JSON ni la BD.
Sin dependencias externas. Lógica de negocio pura.
"""
from pydantic import BaseModel, field_validator, computed_field


class Producto(BaseModel):
    """
    Entidad de dominio Producto.
    No tiene campo 'price' como string — eso es un detalle del DTO.
    """
    id:        int
    nombre:    str
    precio:    float
    categoria: str
    activo:    bool
    stock:     int
    imagen_url: str | None = None

    # ── Lógica de negocio pura ────────────────────────────────────

    @computed_field
    @property
    def disponible(self) -> bool:
        """Disponible solo si está activo y tiene stock."""
        return self.activo and self.stock > 0

    @computed_field
    @property
    def nivel_stock(self) -> str:
        """Nivel descriptivo del stock."""
        return (
            "agotado" if self.stock == 0
            else "bajo"    if self.stock <= 5
            else "normal"  if self.stock <= 20
            else "alto"
        )

    @computed_field
    @property
    def precio_con_iva(self) -> float:
        """Precio con IVA del 16%."""
        return round(self.precio * 1.16, 2)

    @field_validator("precio")
    @classmethod
    def precio_positivo(cls, v: float) -> float:
        if v < 0:
            raise ValueError(f"El precio no puede ser negativo: {v}")
        return v

    @field_validator("stock")
    @classmethod
    def stock_no_negativo(cls, v: int) -> int:
        if v < 0:
            raise ValueError(f"El stock no puede ser negativo: {v}")
        return v

    def __hash__(self) -> int:
        return hash(self.id)

    def __eq__(self, otro: object) -> bool:
        if not isinstance(otro, Producto):
            return NotImplemented
        return self.id == otro.id


class PaginaProductos(BaseModel):
    """Resultado paginado de una consulta de productos."""
    items:    list[Producto]
    total:    int
    pagina:   int
    hay_mas:  bool

    @property
    def vacia(self) -> bool:
        return len(self.items) == 0
```

### `src/productos_api/domain/repositories.py`

```python
# src/productos_api/domain/repositories.py
"""
Protocolo del repositorio — define el CONTRATO, no la implementación.
La capa de dominio solo conoce este protocolo.
La implementación vive en la capa de datos.
"""
from typing import Protocol, runtime_checkable
from .entities import Producto, PaginaProductos


@runtime_checkable
class RepositorioProductos(Protocol):
    """
    Contrato del repositorio de productos.
    Cualquier clase que implemente estos métodos satisface el protocolo.
    No necesita heredar de esta clase.
    """

    async def listar(
        self,
        pagina: int = 1,
        tamano: int = 10,
        busqueda: str = "",
    ) -> PaginaProductos:
        """Obtener una página de productos."""
        ...

    async def obtener_por_id(self, id: int) -> Producto | None:
        """Obtener un producto por ID. Retorna None si no existe."""
        ...
```

### `src/productos_api/domain/usecases.py`

```python
# src/productos_api/domain/usecases.py
"""
Casos de uso — encapsulan UNA operación de negocio.
Reciben el repositorio por constructor para facilitar el testing.
No saben nada de FastAPI ni de HTTP.
"""
from .entities import Producto, PaginaProductos
from .repositories import RepositorioProductos


class ObtenerProductosUC:
    """
    Caso de uso: obtener una página de productos.
    Puede agregar reglas de negocio sin tocar el repositorio ni la API.
    """

    def __init__(self, repositorio: RepositorioProductos) -> None:
        self._repo = repositorio

    async def ejecutar(
        self,
        pagina: int = 1,
        tamano: int = 10,
        busqueda: str = "",
    ) -> PaginaProductos:
        # Validación de negocio: tamaño de página entre 1 y 100
        tamano = max(1, min(tamano, 100))
        pagina = max(1, pagina)

        return await self._repo.listar(
            pagina=pagina,
            tamano=tamano,
            busqueda=busqueda,
        )


class ObtenerProductoPorIdUC:
    """Caso de uso: obtener un producto específico por su ID."""

    def __init__(self, repositorio: RepositorioProductos) -> None:
        self._repo = repositorio

    async def ejecutar(self, id: int) -> Producto | None:
        if id <= 0:
            raise ValueError(f"El ID debe ser positivo, se recibió: {id}")
        return await self._repo.obtener_por_id(id)


class BuscarProductosUC:
    """
    Caso de uso: buscar productos por texto.
    Centraliza la lógica de búsqueda — incluida la validación.
    """

    LONGITUD_MINIMA_QUERY = 2

    def __init__(self, repositorio: RepositorioProductos) -> None:
        self._repo = repositorio

    async def ejecutar(self, query: str) -> list[Producto]:
        # Regla de negocio: no buscar con queries demasiado cortos
        query = query.strip()
        if len(query) < self.LONGITUD_MINIMA_QUERY:
            return []

        resultado = await self._repo.listar(
            busqueda=query,
            tamano=50,
        )
        return resultado.items
```

```bash
# Verificar que el dominio no tiene dependencias externas
python -c "from src.productos_api.domain import entities, repositories, usecases; print('OK')"
```

---

## Paso 2 — Capa Data

La capa de datos implementa los contratos definidos en el dominio.
Aquí sí viven las dependencias externas: httpx, JSON parsing.

### `src/productos_api/data/dtos.py`

```python
# src/productos_api/data/dtos.py
"""
DTOs — Data Transfer Objects.
Representan la estructura exacta del JSON que devuelve la API externa.
La conversión DTO → Entidad ocurre aquí, no en el dominio.
"""
from pydantic import BaseModel, Field
from ..domain.entities import Producto, PaginaProductos


class ProductoDTO(BaseModel):
    """
    Estructura EXACTA del JSON de la API.
    Los nombres de campos coinciden con el JSON (snake_case de la API).
    """
    id:            int
    name:          str
    price:         str           # La API devuelve precio como string
    category_name: str | None = None
    is_active:     bool = False
    stock:         int  = 0
    url_image:     str | None = None

    def a_entidad(self) -> Producto:
        """Convertir DTO a entidad de dominio."""
        return Producto(
            id=        self.id,
            nombre=    self.name,
            precio=    float(self.price),
            categoria= self.category_name or "Sin categoría",
            activo=    self.is_active,
            stock=     self.stock,
            imagen_url=self.url_image,
        )


class RespuestaPaginadaDTO(BaseModel):
    """Estructura de la respuesta paginada de la API."""
    count:    int
    next:     str | None = None
    previous: str | None = None
    results:  list[ProductoDTO] = Field(default_factory=list)

    @property
    def hay_mas(self) -> bool:
        return self.next is not None

    def a_pagina(self, pagina: int) -> PaginaProductos:
        """Convertir la respuesta paginada a la entidad de dominio."""
        return PaginaProductos(
            items=  [dto.a_entidad() for dto in self.results],
            total=  self.count,
            pagina= pagina,
            hay_mas=self.hay_mas,
        )
```

### `src/productos_api/data/repositories.py`

```python
# src/productos_api/data/repositories.py
"""
Implementación real del repositorio — hace las llamadas HTTP.
La capa de aplicación NUNCA importa esta clase directamente.
Solo la conoce como RepositorioProductos (el protocolo del dominio).
"""
import httpx
from ..domain.entities import Producto, PaginaProductos
from .dtos import ProductoDTO, RespuestaPaginadaDTO


class RepositorioProductosHTTP:
    """
    Implementación del repositorio usando la API HTTP externa.
    Satisface el protocolo RepositorioProductos sin heredar de él.
    """

    BASE_URL = "https://higuera-billing-api.desarrollo-software.xyz/api"

    def __init__(self, cliente: httpx.AsyncClient) -> None:
        # El cliente se inyecta — facilita el testing con mocks
        self._cliente = cliente

    async def listar(
        self,
        pagina: int = 1,
        tamano: int = 10,
        busqueda: str = "",
    ) -> PaginaProductos:
        """Llamar a la API y convertir la respuesta a entidades de dominio."""
        params: dict[str, str | int] = {
            "page":      pagina,
            "page_size": tamano,
        }
        if busqueda:
            params["search"] = busqueda

        try:
            respuesta = await self._cliente.get(
                f"{self.BASE_URL}/products/",
                params=params,
                timeout=15.0,
            )
            respuesta.raise_for_status()

            # Parsear con el DTO — validación automática con Pydantic
            dto = RespuestaPaginadaDTO.model_validate(respuesta.json())
            return dto.a_pagina(pagina)

        except httpx.HTTPStatusError as e:
            raise RuntimeError(
                f"Error de la API: {e.response.status_code}"
            ) from e
        except httpx.ConnectError as e:
            raise RuntimeError("Sin conexión a la API externa") from e
        except httpx.TimeoutException as e:
            raise RuntimeError("Tiempo de espera agotado") from e

    async def obtener_por_id(self, id: int) -> Producto | None:
        """Obtener un producto específico por ID."""
        try:
            respuesta = await self._cliente.get(
                f"{self.BASE_URL}/products/{id}/",
                timeout=15.0,
            )
            if respuesta.status_code == 404:
                return None
            respuesta.raise_for_status()

            dto = ProductoDTO.model_validate(respuesta.json())
            return dto.a_entidad()

        except httpx.HTTPStatusError as e:
            raise RuntimeError(
                f"Error de la API: {e.response.status_code}"
            ) from e
```

---

## Paso 3 — Capa Application

La capa de aplicación conecta el mundo HTTP con el dominio.
Aquí vive FastAPI, los schemas de respuesta y la inyección de dependencias.

### `src/productos_api/application/schemas.py`

```python
# src/productos_api/application/schemas.py
"""
Schemas de respuesta de la API — lo que el cliente final recibe.
Son diferentes a las entidades de dominio: pueden tener más o menos campos,
campos renombrados para el API público, etc.
"""
from pydantic import BaseModel


class ProductoSchema(BaseModel):
    """Schema de respuesta para un producto individual."""
    id:          int
    nombre:      str
    precio:      float
    precio_iva:  float
    categoria:   str
    activo:      bool
    stock:       int
    nivel_stock: str
    disponible:  bool
    imagen_url:  str | None = None


class PaginaProductosSchema(BaseModel):
    """Schema de respuesta paginada."""
    items:   list[ProductoSchema]
    total:   int
    pagina:  int
    hay_mas: bool
    vacia:   bool


class ErrorSchema(BaseModel):
    """Schema de respuesta de error."""
    detalle: str
    codigo:  int
```

### `src/productos_api/application/dependencies.py`

```python
# src/productos_api/application/dependencies.py
"""
Inyección de dependencias de FastAPI.
Aquí se cablea la arquitectura: qué implementación usa cada capa.
Cambiar de HTTP a una BD local solo requiere modificar este archivo.
"""
from typing import Annotated
from fastapi import Depends, Request
import httpx

from ..domain.repositories import RepositorioProductos
from ..domain.usecases import (
    ObtenerProductosUC,
    ObtenerProductoPorIdUC,
    BuscarProductosUC,
)
from ..data.repositories import RepositorioProductosHTTP


def obtener_cliente_http(request: Request) -> httpx.AsyncClient:
    """
    Obtener el cliente HTTP del estado de la app.
    El cliente se crea en el lifespan y se reutiliza para todas las peticiones.
    """
    return request.app.state.http_client


def obtener_repositorio(
    cliente: Annotated[httpx.AsyncClient, Depends(obtener_cliente_http)],
) -> RepositorioProductos:
    """
    Obtener el repositorio de productos.
    Retorna la INTERFAZ (Protocol), no la implementación.
    El router no sabe que por debajo hay HTTP.
    """
    return RepositorioProductosHTTP(cliente)


def obtener_uc_listar(
    repo: Annotated[RepositorioProductos, Depends(obtener_repositorio)],
) -> ObtenerProductosUC:
    """Caso de uso: obtener lista de productos."""
    return ObtenerProductosUC(repo)


def obtener_uc_por_id(
    repo: Annotated[RepositorioProductos, Depends(obtener_repositorio)],
) -> ObtenerProductoPorIdUC:
    """Caso de uso: obtener producto por ID."""
    return ObtenerProductoPorIdUC(repo)


def obtener_uc_buscar(
    repo: Annotated[RepositorioProductos, Depends(obtener_repositorio)],
) -> BuscarProductosUC:
    """Caso de uso: buscar productos."""
    return BuscarProductosUC(repo)
```

### `src/productos_api/application/routers.py`

```python
# src/productos_api/application/routers.py
"""
Routers de FastAPI — solo coordinan HTTP ↔ casos de uso.
No tienen lógica de negocio ni acceso directo a datos.
"""
from typing import Annotated
from fastapi import APIRouter, Depends, HTTPException, Query

from ..domain.usecases import (
    ObtenerProductosUC,
    ObtenerProductoPorIdUC,
    BuscarProductosUC,
)
from .dependencies import (
    obtener_uc_listar,
    obtener_uc_por_id,
    obtener_uc_buscar,
)
from .schemas import ProductoSchema, PaginaProductosSchema

router = APIRouter(prefix="/productos", tags=["Productos"])


def _producto_a_schema(p) -> ProductoSchema:
    """Convertir entidad de dominio a schema de respuesta."""
    return ProductoSchema(
        id=         p.id,
        nombre=     p.nombre,
        precio=     p.precio,
        precio_iva= p.precio_con_iva,
        categoria=  p.categoria,
        activo=     p.activo,
        stock=      p.stock,
        nivel_stock=p.nivel_stock,
        disponible= p.disponible,
        imagen_url= p.imagen_url,
    )


@router.get(
    "",
    response_model=PaginaProductosSchema,
    summary="Listar productos",
    description="Retorna una página de productos con información completa.",
)
async def listar_productos(
    pagina:  Annotated[int, Query(ge=1, description="Número de página")] = 1,
    tamano:  Annotated[int, Query(ge=1, le=100, description="Productos por página")] = 10,
    uc:      Annotated[ObtenerProductosUC, Depends(obtener_uc_listar)] = ...,
) -> PaginaProductosSchema:
    try:
        pagina_dom = await uc.ejecutar(pagina=pagina, tamano=tamano)
        return PaginaProductosSchema(
            items=  [_producto_a_schema(p) for p in pagina_dom.items],
            total=  pagina_dom.total,
            pagina= pagina_dom.pagina,
            hay_mas=pagina_dom.hay_mas,
            vacia=  pagina_dom.vacia,
        )
    except RuntimeError as e:
        raise HTTPException(status_code=503, detail=str(e)) from e


@router.get(
    "/{id_producto}",
    response_model=ProductoSchema,
    summary="Obtener producto por ID",
)
async def obtener_producto(
    id_producto: int,
    uc: Annotated[ObtenerProductoPorIdUC, Depends(obtener_uc_por_id)] = ...,
) -> ProductoSchema:
    try:
        producto = await uc.ejecutar(id_producto)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e)) from e
    except RuntimeError as e:
        raise HTTPException(status_code=503, detail=str(e)) from e

    if producto is None:
        raise HTTPException(status_code=404, detail="Producto no encontrado")

    return _producto_a_schema(producto)


@router.get(
    "/buscar/{query}",
    response_model=list[ProductoSchema],
    summary="Buscar productos por texto",
)
async def buscar_productos(
    query: str,
    uc:    Annotated[BuscarProductosUC, Depends(obtener_uc_buscar)] = ...,
) -> list[ProductoSchema]:
    try:
        productos = await uc.ejecutar(query)
        return [_producto_a_schema(p) for p in productos]
    except RuntimeError as e:
        raise HTTPException(status_code=503, detail=str(e)) from e
```

---

## Paso 4 — Integración completa con main.py

```python
# src/productos_api/application/main.py
"""
Punto de entrada de la aplicación.
Configura el lifespan (apertura y cierre de recursos),
registra los routers y define el estado global.
"""
from contextlib import asynccontextmanager
from typing import AsyncGenerator

import httpx
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .routers import router


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """
    Lifespan: código que se ejecuta al iniciar y apagar la app.
    Aquí creamos y destruimos recursos compartidos.
    """
    # ── INICIO ────────────────────────────────────────────────────
    print("Iniciando aplicación...")

    # Crear cliente HTTP con configuración óptima
    # Un solo cliente para toda la app — reutiliza conexiones (connection pool)
    app.state.http_client = httpx.AsyncClient(
        headers={
            "Accept":     "application/json",
            "User-Agent": "ProductosAPI/1.0",
        },
        timeout=httpx.Timeout(10.0, connect=5.0),
        limits=httpx.Limits(
            max_connections=20,
            max_keepalive_connections=10,
        ),
    )
    print("Cliente HTTP inicializado")

    yield  # aquí se ejecuta la app

    # ── APAGADO ───────────────────────────────────────────────────
    print("Cerrando aplicación...")
    await app.state.http_client.aclose()
    print("Cliente HTTP cerrado")


def crear_app() -> FastAPI:
    """Factory de la app FastAPI — facilita el testing."""
    app = FastAPI(
        title="API de Productos",
        description="Ejemplo de Clean Architecture con FastAPI",
        version="1.0.0",
        lifespan=lifespan,
    )

    # CORS para desarrollo
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Registrar routers
    app.include_router(router, prefix="/api/v1")

    @app.get("/", tags=["Health"])
    async def raiz():
        return {"estado": "OK", "version": "1.0.0"}

    @app.get("/health", tags=["Health"])
    async def health():
        return {"status": "healthy"}

    return app


app = crear_app()
```

```bash
# Ejecutar el servidor
uvicorn src.productos_api.application.main:app --reload

# Endpoints disponibles:
# GET /api/v1/productos          → lista paginada
# GET /api/v1/productos/1        → producto por ID
# GET /api/v1/productos/buscar/teclado → búsqueda
# GET /                          → health check
# GET /docs                      → Swagger UI automático
```

---

## Paso 5 — Tests de la arquitectura

La ventaja real de Clean Architecture es que cada capa se testea
de forma independiente sin necesitar las demás.

```python
# tests/test_domain.py
"""Tests unitarios de la capa de dominio — sin HTTP, sin BD."""
import pytest
from pydantic import ValidationError
from src.productos_api.domain.entities import Producto, PaginaProductos
from src.productos_api.domain.usecases import (
    ObtenerProductosUC,
    ObtenerProductoPorIdUC,
    BuscarProductosUC,
)


# ── Fixtures ─────────────────────────────────────────────────────

@pytest.fixture
def producto_activo() -> Producto:
    return Producto(
        id=1, nombre="Teclado mecánico", precio=89.99,
        categoria="Periféricos", activo=True, stock=10,
    )


@pytest.fixture
def producto_sin_stock() -> Producto:
    return Producto(
        id=2, nombre="Monitor agotado", precio=349.99,
        categoria="Monitores", activo=True, stock=0,
    )


@pytest.fixture
def repositorio_mock():
    """Mock simple del repositorio usando MagicMock."""
    from unittest.mock import AsyncMock
    return AsyncMock()


# ── Tests de entidades ────────────────────────────────────────────

class TestProducto:
    def test_disponible_cuando_activo_y_con_stock(self, producto_activo):
        assert producto_activo.disponible is True

    def test_no_disponible_sin_stock(self, producto_sin_stock):
        assert producto_sin_stock.disponible is False

    def test_no_disponible_cuando_inactivo(self):
        p = Producto(id=3, nombre="Inactivo", precio=10.0,
                     categoria="C", activo=False, stock=5)
        assert p.disponible is False

    @pytest.mark.parametrize("stock, nivel", [
        (0,  "agotado"),
        (1,  "bajo"),
        (5,  "bajo"),
        (6,  "normal"),
        (20, "normal"),
        (21, "alto"),
    ])
    def test_nivel_stock(self, stock, nivel):
        p = Producto(id=1, nombre="T", precio=1.0,
                     categoria="C", activo=True, stock=stock)
        assert p.nivel_stock == nivel

    def test_precio_con_iva(self, producto_activo):
        assert producto_activo.precio_con_iva == pytest.approx(104.39, abs=0.01)

    def test_precio_negativo_falla(self):
        with pytest.raises(ValidationError):
            Producto(id=1, nombre="T", precio=-10.0,
                     categoria="C", activo=True, stock=0)

    def test_igualdad_por_id(self):
        p1 = Producto(id=1, nombre="A", precio=10.0,
                      categoria="C", activo=True, stock=1)
        p2 = Producto(id=1, nombre="B", precio=20.0,
                      categoria="D", activo=False, stock=0)
        assert p1 == p2  # mismo ID → iguales


# ── Tests de casos de uso ─────────────────────────────────────────

class TestObtenerProductosUC:
    @pytest.mark.asyncio
    async def test_llama_al_repositorio(self, repositorio_mock):
        from src.productos_api.domain.entities import PaginaProductos
        repositorio_mock.listar.return_value = PaginaProductos(
            items=[], total=0, pagina=1, hay_mas=False
        )
        uc = ObtenerProductosUC(repositorio_mock)
        await uc.ejecutar(pagina=1, tamano=10)

        repositorio_mock.listar.assert_called_once_with(
            pagina=1, tamano=10, busqueda=""
        )

    @pytest.mark.asyncio
    async def test_limita_tamano_maximo(self, repositorio_mock):
        """El UC no permite tamaños de página mayores a 100."""
        from src.productos_api.domain.entities import PaginaProductos
        repositorio_mock.listar.return_value = PaginaProductos(
            items=[], total=0, pagina=1, hay_mas=False
        )
        uc = ObtenerProductosUC(repositorio_mock)
        await uc.ejecutar(tamano=500)

        # Debe haber limitado el tamaño a 100
        llamada = repositorio_mock.listar.call_args
        assert llamada.kwargs["tamano"] == 100


class TestBuscarProductosUC:
    @pytest.mark.asyncio
    async def test_query_muy_corto_retorna_lista_vacia(self,
                                                        repositorio_mock):
        uc = BuscarProductosUC(repositorio_mock)
        resultado = await uc.ejecutar("a")
        assert resultado == []
        repositorio_mock.listar.assert_not_called()

    @pytest.mark.asyncio
    async def test_espacios_solos_retorna_lista_vacia(self,
                                                       repositorio_mock):
        uc = BuscarProductosUC(repositorio_mock)
        resultado = await uc.ejecutar("   ")
        assert resultado == []
```

```python
# tests/test_data.py
"""Tests de la capa de datos — DTOs y conversión."""
import pytest
from src.productos_api.data.dtos import ProductoDTO, RespuestaPaginadaDTO


class TestProductoDTO:
    def test_parsea_json_completo(self):
        json_api = {
            "id": 42, "name": "Monitor UHD",
            "price": "349.99", "category_name": "Monitores",
            "is_active": True, "stock": 8,
            "url_image": "https://img.test/monitor.jpg",
        }
        dto = ProductoDTO.model_validate(json_api)
        assert dto.id == 42
        assert dto.name == "Monitor UHD"
        assert dto.is_active is True

    def test_conversion_a_entidad(self):
        dto = ProductoDTO(
            id=1, name="Teclado", price="89.99",
            category_name="Periféricos", is_active=True, stock=5,
        )
        entidad = dto.a_entidad()
        assert entidad.precio == 89.99
        assert entidad.nombre == "Teclado"
        assert entidad.categoria == "Periféricos"

    def test_categoria_por_defecto_sin_categoria(self):
        dto = ProductoDTO(id=1, name="T", price="10.00",
                          is_active=True, stock=0)
        entidad = dto.a_entidad()
        assert entidad.categoria == "Sin categoría"

    def test_respuesta_paginada(self):
        json_paginado = {
            "count": 100,
            "next": "https://api.ejemplo.com/products/?page=2",
            "previous": None,
            "results": [
                {"id": 1, "name": "Prod 1", "price": "10.00",
                 "is_active": True, "stock": 5},
            ],
        }
        dto = RespuestaPaginadaDTO.model_validate(json_paginado)
        assert dto.hay_mas is True
        pagina = dto.a_pagina(1)
        assert pagina.total == 100
        assert pagina.hay_mas is True
        assert len(pagina.items) == 1
```

```python
# tests/test_api.py
"""Tests de integración de la API — mocking del repositorio."""
import pytest
from unittest.mock import AsyncMock
from fastapi.testclient import TestClient

from src.productos_api.application.main import crear_app
from src.productos_api.domain.entities import Producto, PaginaProductos
from src.productos_api.application.dependencies import obtener_repositorio


@pytest.fixture
def repo_mock():
    return AsyncMock()


@pytest.fixture
def cliente(repo_mock):
    """TestClient con el repositorio mockeado — sin HTTP real."""
    app = crear_app()

    # Sobreescribir la dependencia del repositorio
    app.dependency_overrides[obtener_repositorio] = lambda: repo_mock

    # TestClient maneja el lifespan automáticamente
    with TestClient(app) as c:
        yield c


@pytest.fixture
def producto_ejemplo() -> Producto:
    return Producto(
        id=1, nombre="Teclado mecánico", precio=89.99,
        categoria="Periféricos", activo=True, stock=10,
    )


class TestAPIProductos:
    def test_listar_productos_exitoso(self, cliente, repo_mock,
                                      producto_ejemplo):
        repo_mock.listar.return_value = PaginaProductos(
            items=[producto_ejemplo], total=1, pagina=1, hay_mas=False
        )
        resp = cliente.get("/api/v1/productos")
        assert resp.status_code == 200
        datos = resp.json()
        assert datos["total"] == 1
        assert datos["items"][0]["nombre"] == "Teclado mecánico"
        assert "precio_iva" in datos["items"][0]

    def test_obtener_por_id_existe(self, cliente, repo_mock, producto_ejemplo):
        repo_mock.obtener_por_id.return_value = producto_ejemplo
        resp = cliente.get("/api/v1/productos/1")
        assert resp.status_code == 200
        assert resp.json()["id"] == 1

    def test_obtener_por_id_no_existe(self, cliente, repo_mock):
        repo_mock.obtener_por_id.return_value = None
        resp = cliente.get("/api/v1/productos/999")
        assert resp.status_code == 404

    def test_listar_cuando_api_falla(self, cliente, repo_mock):
        repo_mock.listar.side_effect = RuntimeError("Sin conexión")
        resp = cliente.get("/api/v1/productos")
        assert resp.status_code == 503

    def test_paginacion(self, cliente, repo_mock):
        repo_mock.listar.return_value = PaginaProductos(
            items=[], total=0, pagina=2, hay_mas=False
        )
        resp = cliente.get("/api/v1/productos?pagina=2&tamano=5")
        assert resp.status_code == 200
        # Verificar que se pasaron los parámetros correctos
        repo_mock.listar.assert_called_once_with(
            pagina=2, tamano=5, busqueda=""
        )
```

```bash
# Ejecutar todos los tests
pytest tests/ -v

# Con cobertura
pytest tests/ --cov=src --cov-report=term-missing

# Solo la capa de dominio (muy rápidos — sin mocks externos)
pytest tests/test_domain.py -v
```

---

## Ejercicios propuestos

1. **Caso de uso: productos por categoría** — Crea `ObtenerProductosPorCategoriaUC` en la capa de dominio. El caso de uso recibe `categoria: str` y retorna todos los productos de esa categoría. Agrega el método `listar_por_categoria` al protocolo `RepositorioProductos` y su implementación en `RepositorioProductosHTTP`. Crea el endpoint `GET /api/v1/productos/categoria/{nombre}` y sus tests correspondientes mockeando el repositorio.

2. **Repositorio Fake para desarrollo** — Crea `RepositorioProductosFake` en `src/productos_api/data/repositories.py` con datos hardcodeados (10 productos ficticios). Modifica `dependencies.py` para que cuando la variable de entorno `USE_FAKE_REPO=true` esté configurada, se use el fake en lugar del HTTP real. Escribe tests que verifiquen que ambos repositorios satisfacen el protocolo `RepositorioProductos` usando `isinstance(repo, RepositorioProductos)`.

3. **Cache de resultados** — Agrega un repositorio decorador `RepositorioConCache` que envuelva a `RepositorioProductos` y cachee los resultados en memoria por 5 minutos. El decorador debe implementar el mismo protocolo. Para invalidar el cache usa el patrón TTL del módulo 17. Escribe tests que verifiquen que: (a) la primera llamada va al repositorio subyacente, (b) la segunda llamada dentro del TTL usa el cache, (c) después del TTL vuelve al repositorio.

4. **Logging estructurado** — Agrega logging a la capa de aplicación usando `logging` estándar con formato JSON. En cada router, loguea: método HTTP, ruta, tiempo de respuesta y status code. En los casos de uso, loguea cuándo se llama cada uno y con qué parámetros. Usa `structlog` o un formatter JSON personalizado. Agrega un middleware de FastAPI que loguee todas las peticiones con un request_id único (usa `uuid.uuid4()`).

---

## Resumen de la página 18

- **Clean Architecture** divide en tres capas: Domain (lógica de negocio pura, sin dependencias externas), Data (implementación de repositorios, DTOs, HTTP), Application (FastAPI, schemas, inyección de dependencias).
- La **regla de dependencias**: Domain no conoce a nadie; Data conoce a Domain; Application conoce a Domain. Nunca hacia afuera.
- Las **entidades de dominio** con Pydantic tienen validadores (`@field_validator`) y campos calculados (`@computed_field`) que expresan las reglas de negocio sin código externo.
- El **Protocol** `RepositorioProductos` es el contrato — define qué operaciones existen. La implementación `RepositorioProductosHTTP` lo satisface sin heredar de él.
- Los **casos de uso** encapsulan una sola operación de negocio. Reciben el repositorio por constructor, lo que hace trivial el testing con mocks.
- Los **DTOs** en la capa Data representan el JSON de la API externa. El método `.a_entidad()` realiza la conversión y aislamiento de los detalles del JSON.
- La **inyección de dependencias** de FastAPI (`Depends`) cablea la arquitectura: `obtener_repositorio` retorna el Protocol, no la implementación concreta.
- `dependency_overrides` en los tests permite reemplazar cualquier dependencia sin modificar el código de producción — los tests de la API usan un mock del repositorio, no HTTP real.
- El **lifespan** de FastAPI (`asynccontextmanager`) crea y destruye recursos compartidos (el cliente httpx con connection pool) de forma limpia.

---

> **Siguiente página →** Página 22: Proyecto integrador — API de e-commerce completa.
