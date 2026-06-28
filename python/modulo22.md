# Tutorial Python — Página 22
## Módulo 12 · Arquitectura
### Proyecto integrador — API de e-commerce con FastAPI, SQLAlchemy y Clean Architecture

---

## Descripción del proyecto

Construiremos una API REST de e-commerce con usuarios, productos y órdenes.
El proyecto aplica todo lo visto en el curso: Clean Architecture, FastAPI,
SQLAlchemy 2.0, Alembic, JWT, Pydantic v2, pytest y Ruff.

```
GET  /api/productos/           ← lista paginada
GET  /api/productos/{id}/      ← detalle
POST /api/auth/registro/       ← crear cuenta
POST /api/auth/login/          ← obtener JWT
GET  /api/ordenes/             ← mis órdenes (requiere JWT)
POST /api/ordenes/             ← crear orden (requiere JWT)
```

---

## Estructura del proyecto

```
tienda_api/
├── pyproject.toml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
│
├── src/
│   └── tienda/
│       ├── __init__.py
│       │
│       ├── domain/                        ← CAPA DE DOMINIO
│       │   ├── __init__.py
│       │   ├── entidades.py               ← Producto, Usuario, Orden
│       │   ├── repositorios.py            ← Protocol de cada repositorio
│       │   └── casos_de_uso/
│       │       ├── __init__.py
│       │       ├── productos.py
│       │       ├── autenticacion.py
│       │       └── ordenes.py
│       │
│       ├── data/                          ← CAPA DE DATOS
│       │   ├── __init__.py
│       │   ├── modelos_db.py              ← modelos SQLAlchemy
│       │   ├── db.py                      ← engine, sesión
│       │   └── repositorios/
│       │       ├── __init__.py
│       │       ├── productos_repo.py
│       │       ├── usuarios_repo.py
│       │       └── ordenes_repo.py
│       │
│       └── app/                           ← CAPA DE APLICACIÓN
│           ├── __init__.py
│           ├── main.py                    ← FastAPI + lifespan
│           ├── dependencias.py            ← Depends() compartidas
│           ├── seguridad.py               ← JWT, hashing
│           ├── schemas/
│           │   ├── __init__.py
│           │   ├── productos.py
│           │   ├── usuarios.py
│           │   └── ordenes.py
│           └── routers/
│               ├── __init__.py
│               ├── productos.py
│               ├── auth.py
│               └── ordenes.py
│
└── tests/
    ├── conftest.py
    ├── domain/
    │   └── test_casos_de_uso.py
    └── app/
        └── test_routers.py
```

### `pyproject.toml`

```toml
[tool.poetry]
name        = "tienda-api"
version     = "0.1.0"
description = "API de e-commerce — proyecto integrador"

[tool.poetry.dependencies]
python            = "^3.12"
fastapi           = {extras = ["standard"], version = "^0.115"}
sqlalchemy        = "^2.0"
alembic           = "^1.13"
pydantic-settings = "^2.3"
passlib            = {extras = ["bcrypt"], version = "^1.7"}
PyJWT             = "^2.9"
httpx             = "^0.27"

[tool.poetry.group.dev.dependencies]
pytest            = "^8.3"
pytest-asyncio    = "^0.24"
pytest-cov        = "^5.0"
ruff              = "^0.6"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths    = ["tests"]

[tool.coverage.run]
source = ["src"]
omit   = ["*/alembic/*"]
```

```bash
poetry install
```

---

## Paso 1 — Capa Domain

### `src/tienda/domain/entidades.py`

```python
"""Entidades de dominio — Dart puro sin dependencias externas."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal
from enum import StrEnum


class EstadoOrden(StrEnum):
    PENDIENTE  = "pendiente"
    CONFIRMADA = "confirmada"
    ENVIADA    = "enviada"
    ENTREGADA  = "entregada"
    CANCELADA  = "cancelada"


@dataclass(frozen=True)
class Producto:
    id:        int
    nombre:    str
    precio:    Decimal
    categoria: str
    activo:    bool
    stock:     int

    @property
    def disponible(self) -> bool:
        return self.activo and self.stock > 0

    @property
    def nivel_stock(self) -> str:
        return (
            "Agotado" if self.stock == 0
            else "Bajo"    if self.stock <= 5
            else "Normal"  if self.stock <= 20
            else "Alto"
        )

    def __str__(self) -> str:
        return f"Producto({self.id}: {self.nombre} @ ${self.precio})"


@dataclass(frozen=True)
class Usuario:
    id:            int
    email:         str
    nombre:        str
    hash_password: str
    activo:        bool = True

    def __str__(self) -> str:
        return f"Usuario({self.id}: {self.email})"


@dataclass
class LineaOrden:
    producto:  Producto
    cantidad:  int
    precio_unitario: Decimal  # precio al momento de la compra

    @property
    def subtotal(self) -> Decimal:
        return self.precio_unitario * self.cantidad


@dataclass
class Orden:
    id:          int
    usuario_id:  int
    estado:      EstadoOrden
    creada_en:   datetime
    lineas:      list[LineaOrden] = field(default_factory=list)

    @property
    def total(self) -> Decimal:
        return sum(linea.subtotal for linea in self.lineas)

    @property
    def cantidad_items(self) -> int:
        return sum(linea.cantidad for linea in self.lineas)

    def puede_cancelar(self) -> bool:
        """Regla de negocio: solo se puede cancelar si no fue enviada."""
        return self.estado in (EstadoOrden.PENDIENTE, EstadoOrden.CONFIRMADA)
```

### `src/tienda/domain/repositorios.py`

```python
"""Contratos de repositorios — Protocol (duck typing estático)."""

from __future__ import annotations

from typing import Protocol, runtime_checkable

from .entidades import Orden, Producto, Usuario


@runtime_checkable
class IProductosRepo(Protocol):
    def obtener_lista(
        self,
        *,
        pagina:   int = 1,
        por_pag:  int = 10,
        busqueda: str = "",
    ) -> tuple[list[Producto], int]:
        """Devuelve (items, total_count)."""
        ...

    def obtener_por_id(self, producto_id: int) -> Producto | None: ...


@runtime_checkable
class IUsuariosRepo(Protocol):
    def obtener_por_email(self, email: str) -> Usuario | None: ...
    def crear(self, email: str, nombre: str, hash_password: str) -> Usuario: ...


@runtime_checkable
class IOrdenesRepo(Protocol):
    def obtener_por_usuario(self, usuario_id: int) -> list[Orden]: ...
    def crear(
        self,
        usuario_id: int,
        lineas: list[tuple[int, int]],  # (producto_id, cantidad)
    ) -> Orden: ...
    def actualizar_estado(self, orden_id: int, estado: str) -> Orden: ...
```

### `src/tienda/domain/casos_de_uso/productos.py`

```python
"""Casos de uso para productos."""

from __future__ import annotations

from dataclasses import dataclass

from tienda.domain.entidades import Producto
from tienda.domain.repositorios import IProductosRepo


@dataclass
class ObtenerProductosUC:
    repo: IProductosRepo

    def ejecutar(
        self,
        *,
        pagina:   int = 1,
        por_pag:  int = 10,
        busqueda: str = "",
    ) -> tuple[list[Producto], int]:
        if pagina < 1:
            raise ValueError("El número de página debe ser mayor a 0")
        if por_pag < 1 or por_pag > 100:
            raise ValueError("El tamaño de página debe estar entre 1 y 100")
        return self.repo.obtener_lista(
            pagina=pagina, por_pag=por_pag, busqueda=busqueda
        )


@dataclass
class ObtenerProductoPorIdUC:
    repo: IProductosRepo

    def ejecutar(self, producto_id: int) -> Producto:
        producto = self.repo.obtener_por_id(producto_id)
        if producto is None:
            raise ValueError(f"Producto {producto_id} no encontrado")
        return producto
```

### `src/tienda/domain/casos_de_uso/autenticacion.py`

```python
"""Casos de uso de autenticación — sin JWT, sin hashing (eso es infraestructura)."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol


class IHasher(Protocol):
    def hashear(self, texto: str) -> str: ...
    def verificar(self, texto: str, hash_: str) -> bool: ...


from tienda.domain.entidades import Usuario
from tienda.domain.repositorios import IUsuariosRepo


@dataclass
class RegistrarUsuarioUC:
    repo:   IUsuariosRepo
    hasher: IHasher

    def ejecutar(self, email: str, nombre: str, password: str) -> Usuario:
        # Regla de negocio: email único
        if self.repo.obtener_por_email(email) is not None:
            raise ValueError(f"El email '{email}' ya está registrado")
        if len(password) < 8:
            raise ValueError("La contraseña debe tener al menos 8 caracteres")

        hash_pw = self.hasher.hashear(password)
        return self.repo.crear(email=email, nombre=nombre, hash_password=hash_pw)


@dataclass
class AutenticarUsuarioUC:
    repo:   IUsuariosRepo
    hasher: IHasher

    def ejecutar(self, email: str, password: str) -> Usuario:
        usuario = self.repo.obtener_por_email(email)
        if usuario is None:
            raise ValueError("Credenciales inválidas")
        if not usuario.activo:
            raise ValueError("La cuenta está desactivada")
        if not self.hasher.verificar(password, usuario.hash_password):
            raise ValueError("Credenciales inválidas")
        return usuario
```

---

## Paso 2 — Capa Data

### `src/tienda/data/db.py`

```python
"""Configuración de la base de datos."""

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from tienda.app.config import configuracion

# engine síncrono — para la API se puede usar async con aiosqlite/asyncpg
motor = create_engine(
    configuracion.database_url,
    connect_args={"check_same_thread": False},  # solo para SQLite
    echo=configuracion.debug,
)

SesionLocal = sessionmaker(autocommit=False, autoflush=False, bind=motor)
```

### `src/tienda/data/modelos_db.py`

```python
"""Modelos SQLAlchemy — representación de la BD, separada de las entidades de dominio."""

from __future__ import annotations

from datetime import datetime, timezone
from decimal import Decimal

from sqlalchemy import (
    Boolean, DateTime, ForeignKey, Integer,
    Numeric, String, func,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class ProductoDB(Base):
    __tablename__ = "productos"

    id:        Mapped[int]     = mapped_column(Integer, primary_key=True, index=True)
    nombre:    Mapped[str]     = mapped_column(String(200), nullable=False)
    precio:    Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
    categoria: Mapped[str]     = mapped_column(String(100), nullable=False, default="General")
    activo:    Mapped[bool]    = mapped_column(Boolean, default=True)
    stock:     Mapped[int]     = mapped_column(Integer, default=0)

    # Relación inversa
    lineas: Mapped[list[LineaOrdenDB]] = relationship(back_populates="producto")


class UsuarioDB(Base):
    __tablename__ = "usuarios"

    id:            Mapped[int]  = mapped_column(Integer, primary_key=True, index=True)
    email:         Mapped[str]  = mapped_column(String(254), unique=True, index=True)
    nombre:        Mapped[str]  = mapped_column(String(150), nullable=False)
    hash_password: Mapped[str]  = mapped_column(String(256), nullable=False)
    activo:        Mapped[bool] = mapped_column(Boolean, default=True)

    # Relación: un usuario tiene muchas órdenes
    ordenes: Mapped[list[OrdenDB]] = relationship(back_populates="usuario")


class OrdenDB(Base):
    __tablename__ = "ordenes"

    id:         Mapped[int]      = mapped_column(Integer, primary_key=True, index=True)
    usuario_id: Mapped[int]      = mapped_column(ForeignKey("usuarios.id"), nullable=False)
    estado:     Mapped[str]      = mapped_column(String(20), default="pendiente")
    creada_en:  Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
    )

    # Relaciones
    usuario: Mapped[UsuarioDB]        = relationship(back_populates="ordenes")
    lineas:  Mapped[list[LineaOrdenDB]] = relationship(
        back_populates="orden", cascade="all, delete-orphan"
    )


class LineaOrdenDB(Base):
    __tablename__ = "lineas_orden"

    id:              Mapped[int]     = mapped_column(Integer, primary_key=True)
    orden_id:        Mapped[int]     = mapped_column(ForeignKey("ordenes.id"), nullable=False)
    producto_id:     Mapped[int]     = mapped_column(ForeignKey("productos.id"), nullable=False)
    cantidad:        Mapped[int]     = mapped_column(Integer, nullable=False)
    precio_unitario: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)

    # Relaciones
    orden:    Mapped[OrdenDB]    = relationship(back_populates="lineas")
    producto: Mapped[ProductoDB] = relationship(back_populates="lineas")
```

### `src/tienda/data/repositorios/productos_repo.py`

```python
"""Implementación del repositorio de productos con SQLAlchemy."""

from __future__ import annotations

from decimal import Decimal

from sqlalchemy import func, or_, select
from sqlalchemy.orm import Session

from tienda.data.modelos_db import ProductoDB
from tienda.domain.entidades import Producto


class ProductosRepo:
    """Implementación concreta. La UI solo conoce IProductosRepo."""

    def __init__(self, sesion: Session) -> None:
        self._db = sesion

    # ── Mapeo DB → Dominio ────────────────────────────────────────────
    @staticmethod
    def _a_dominio(modelo: ProductoDB) -> Producto:
        return Producto(
            id=modelo.id,
            nombre=modelo.nombre,
            precio=Decimal(str(modelo.precio)),
            categoria=modelo.categoria,
            activo=modelo.activo,
            stock=modelo.stock,
        )

    # ── Operaciones ───────────────────────────────────────────────────
    def obtener_lista(
        self,
        *,
        pagina:   int = 1,
        por_pag:  int = 10,
        busqueda: str = "",
    ) -> tuple[list[Producto], int]:
        query = select(ProductoDB).where(ProductoDB.activo.is_(True))

        if busqueda:
            patron = f"%{busqueda}%"
            query = query.where(
                or_(
                    ProductoDB.nombre.ilike(patron),
                    ProductoDB.categoria.ilike(patron),
                )
            )

        # Total antes de paginar
        total = self._db.scalar(
            select(func.count()).select_from(query.subquery())
        ) or 0

        # Paginar
        offset = (pagina - 1) * por_pag
        modelos = self._db.scalars(
            query.offset(offset).limit(por_pag).order_by(ProductoDB.id)
        ).all()

        return [self._a_dominio(m) for m in modelos], total

    def obtener_por_id(self, producto_id: int) -> Producto | None:
        modelo = self._db.get(ProductoDB, producto_id)
        return self._a_dominio(modelo) if modelo else None
```

---

## Paso 3 — Capa Application

### `src/tienda/app/config.py`

```python
"""Configuración con pydantic-settings — carga .env automáticamente."""

from pydantic_settings import BaseSettings, SettingsConfigDict


class Configuracion(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    database_url: str  = "sqlite:///./tienda.db"
    secret_key:   str  = "cambia-este-secreto-en-produccion"
    algoritmo_jwt: str = "HS256"
    token_exp_min: int = 60
    debug:        bool = False


configuracion = Configuracion()
```

### `src/tienda/app/seguridad.py`

```python
"""Hashing de passwords y generación/verificación de JWT."""

from __future__ import annotations

from datetime import datetime, timedelta, timezone

import jwt
from passlib.context import CryptContext

from tienda.app.config import configuracion

# Contexto de hashing con bcrypt
_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hashear_password(password: str) -> str:
    return _ctx.hash(password)


def verificar_password(password: str, hash_: str) -> bool:
    return _ctx.verify(password, hash_)


def crear_token(datos: dict) -> str:
    """Crea un JWT con expiración configurada."""
    carga = datos.copy()
    expira = datetime.now(timezone.utc) + timedelta(
        minutes=configuracion.token_exp_min
    )
    carga["exp"] = expira
    return jwt.encode(
        carga,
        configuracion.secret_key,
        algorithm=configuracion.algoritmo_jwt,
    )


def decodificar_token(token: str) -> dict:
    """Decodifica y valida el JWT. Lanza jwt.InvalidTokenError si es inválido."""
    return jwt.decode(
        token,
        configuracion.secret_key,
        algorithms=[configuracion.algoritmo_jwt],
    )
```

### `src/tienda/app/dependencias.py`

```python
"""Dependencias compartidas inyectadas con Depends()."""

from __future__ import annotations

import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session

from tienda.app.seguridad import decodificar_token
from tienda.data.db import SesionLocal
from tienda.data.repositorios.productos_repo import ProductosRepo
from tienda.data.repositorios.usuarios_repo import UsuariosRepo
from tienda.domain.casos_de_uso.autenticacion import AutenticarUsuarioUC, RegistrarUsuarioUC
from tienda.domain.casos_de_uso.productos import ObtenerProductoPorIdUC, ObtenerProductosUC
from tienda.domain.entidades import Usuario

oauth2_esquema = OAuth2PasswordBearer(tokenUrl="/api/auth/login")


# ── Base de datos ─────────────────────────────────────────────────────
def obtener_db():
    """Generador: abre la sesión y la cierra al finalizar el request."""
    db = SesionLocal()
    try:
        yield db
    finally:
        db.close()


# ── Casos de uso ──────────────────────────────────────────────────────
def productos_uc(db: Session = Depends(obtener_db)) -> ObtenerProductosUC:
    return ObtenerProductosUC(repo=ProductosRepo(db))


def producto_por_id_uc(db: Session = Depends(obtener_db)) -> ObtenerProductoPorIdUC:
    return ObtenerProductoPorIdUC(repo=ProductosRepo(db))


# ── Autenticación ─────────────────────────────────────────────────────
def usuario_actual(
    token: str = Depends(oauth2_esquema),
    db:    Session = Depends(obtener_db),
) -> Usuario:
    """Extrae el usuario del JWT. Lanza 401 si el token es inválido."""
    excepcion = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token inválido o expirado",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        carga = decodificar_token(token)
        email: str | None = carga.get("sub")
        if email is None:
            raise excepcion
    except jwt.InvalidTokenError:
        raise excepcion

    repo    = UsuariosRepo(db)
    usuario = repo.obtener_por_email(email)
    if usuario is None or not usuario.activo:
        raise excepcion

    return usuario
```

### `src/tienda/app/routers/productos.py`

```python
"""Router de productos."""

from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, Query, status

from tienda.app.dependencias import producto_por_id_uc, productos_uc
from tienda.app.schemas.productos import ProductoDetalle, ProductoPaginado
from tienda.domain.casos_de_uso.productos import (
    ObtenerProductoPorIdUC,
    ObtenerProductosUC,
)

router = APIRouter(prefix="/api/productos", tags=["Productos"])


@router.get("/", response_model=ProductoPaginado)
def listar_productos(
    pagina:   int = Query(1, ge=1),
    por_pag:  int = Query(10, ge=1, le=100),
    busqueda: str = Query(""),
    uc: ObtenerProductosUC = Depends(productos_uc),
):
    """Lista productos activos con paginación y búsqueda opcional."""
    items, total = uc.ejecutar(pagina=pagina, por_pag=por_pag, busqueda=busqueda)
    return ProductoPaginado(
        total=total,
        pagina=pagina,
        por_pagina=por_pag,
        hay_mas=(pagina * por_pag) < total,
        items=[ProductoDetalle.model_validate(p.__dict__) for p in items],
    )


@router.get("/{producto_id}/", response_model=ProductoDetalle)
def obtener_producto(
    producto_id: int,
    uc: ObtenerProductoPorIdUC = Depends(producto_por_id_uc),
):
    """Obtiene un producto por ID."""
    try:
        producto = uc.ejecutar(producto_id)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(e))
    return ProductoDetalle.model_validate(producto.__dict__)
```

### `src/tienda/app/schemas/productos.py`

```python
"""Schemas Pydantic para la capa de aplicación."""

from __future__ import annotations

from decimal import Decimal

from pydantic import BaseModel, ConfigDict


class ProductoDetalle(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id:          int
    nombre:      str
    precio:      Decimal
    categoria:   str
    activo:      bool
    stock:       int
    disponible:  bool
    nivel_stock: str


class ProductoPaginado(BaseModel):
    total:     int
    pagina:    int
    por_pagina: int
    hay_mas:   bool
    items:     list[ProductoDetalle]
```

### `src/tienda/app/routers/auth.py`

```python
"""Router de autenticación."""

from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from tienda.app.dependencias import obtener_db
from tienda.app.schemas.usuarios import TokenRespuesta, UsuarioCrear, UsuarioRespuesta
from tienda.app.seguridad import crear_token, hashear_password, verificar_password
from tienda.data.repositorios.usuarios_repo import UsuariosRepo
from tienda.domain.casos_de_uso.autenticacion import AutenticarUsuarioUC, RegistrarUsuarioUC

# Adaptador del hasher para los casos de uso
class _Hasher:
    def hashear(self, texto: str) -> str:
        return hashear_password(texto)
    def verificar(self, texto: str, hash_: str) -> bool:
        return verificar_password(texto, hash_)


router = APIRouter(prefix="/api/auth", tags=["Autenticación"])


@router.post("/registro/", response_model=UsuarioRespuesta,
             status_code=status.HTTP_201_CREATED)
def registrar(datos: UsuarioCrear, db: Session = Depends(obtener_db)):
    """Registra un nuevo usuario."""
    uc = RegistrarUsuarioUC(repo=UsuariosRepo(db), hasher=_Hasher())
    try:
        usuario = uc.ejecutar(
            email=datos.email, nombre=datos.nombre, password=datos.password
        )
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))
    db.commit()
    db.refresh(db.get(type(usuario), usuario.id))  # refrescar desde BD
    return UsuarioRespuesta(id=usuario.id, email=usuario.email, nombre=usuario.nombre)


@router.post("/login/", response_model=TokenRespuesta)
def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db:   Session = Depends(obtener_db),
):
    """Autentica y devuelve un JWT Bearer."""
    uc = AutenticarUsuarioUC(repo=UsuariosRepo(db), hasher=_Hasher())
    try:
        usuario = uc.ejecutar(email=form.username, password=form.password)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(e),
            headers={"WWW-Authenticate": "Bearer"},
        )
    token = crear_token({"sub": usuario.email, "nombre": usuario.nombre})
    return TokenRespuesta(access_token=token, token_type="bearer")
```

### `src/tienda/app/main.py`

```python
"""Punto de entrada de la aplicación FastAPI."""

from __future__ import annotations

from contextlib import asynccontextmanager

from fastapi import FastAPI

from tienda.app.routers import auth, ordenes, productos
from tienda.data.db import motor
from tienda.data.modelos_db import Base


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Crea las tablas al arrancar (en producción usa Alembic)."""
    Base.metadata.create_all(bind=motor)
    yield
    # cleanup al apagar si fuera necesario


app = FastAPI(
    title="Tienda API",
    description="API de e-commerce — proyecto integrador del curso Python",
    version="0.1.0",
    lifespan=lifespan,
)

# Registrar routers
app.include_router(productos.router)
app.include_router(auth.router)
app.include_router(ordenes.router)


@app.get("/", tags=["Health"])
def raiz():
    return {"estado": "ok", "version": "0.1.0"}
```

```bash
# Ejecutar el servidor de desarrollo
poetry run uvicorn tienda.app.main:app --reload

# Documentación interactiva
# http://localhost:8000/docs
```

---

## Paso 4 — Tests

### `tests/conftest.py`

```python
"""Fixtures compartidas entre todos los tests."""

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from tienda.app.dependencias import obtener_db
from tienda.app.main import app
from tienda.data.modelos_db import Base

# BD en memoria para tests — no contamina la BD real
MOTOR_TEST = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
)
SesionTest = sessionmaker(bind=MOTOR_TEST)


@pytest.fixture(autouse=True)
def crear_tablas():
    """Crea y destruye las tablas en cada test."""
    Base.metadata.create_all(bind=MOTOR_TEST)
    yield
    Base.metadata.drop_all(bind=MOTOR_TEST)


@pytest.fixture
def db():
    """Sesión de BD para tests unitarios de repositorios."""
    sesion = SesionTest()
    try:
        yield sesion
    finally:
        sesion.close()


@pytest.fixture
def cliente(db):
    """TestClient con la BD de test inyectada."""
    def obtener_db_test():
        yield db

    app.dependency_overrides[obtener_db] = obtener_db_test
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

### `tests/domain/test_casos_de_uso.py`

```python
"""Tests unitarios de los casos de uso — sin BD, sin HTTP."""

from __future__ import annotations

from decimal import Decimal
from unittest.mock import MagicMock

import pytest

from tienda.domain.casos_de_uso.autenticacion import AutenticarUsuarioUC, RegistrarUsuarioUC
from tienda.domain.casos_de_uso.productos import ObtenerProductosUC
from tienda.domain.entidades import Producto, Usuario


# ── Helpers ───────────────────────────────────────────────────────────

def _producto_base(**kwargs) -> Producto:
    defaults = dict(
        id=1, nombre="Teclado", precio=Decimal("89.99"),
        categoria="Periféricos", activo=True, stock=10,
    )
    return Producto(**{**defaults, **kwargs})


def _usuario_base(**kwargs) -> Usuario:
    defaults = dict(
        id=1, email="ana@ejemplo.com", nombre="Ana",
        hash_password="$2b$12$...", activo=True,
    )
    return Usuario(**{**defaults, **kwargs})


class _HasherFake:
    """Hasher falso para tests: no hace bcrypt real."""
    def hashear(self, texto: str) -> str:
        return f"hash:{texto}"
    def verificar(self, texto: str, hash_: str) -> bool:
        return hash_ == f"hash:{texto}"


# ── Tests de productos ────────────────────────────────────────────────

class TestObtenerProductosUC:

    def test_llama_repositorio_con_defaults(self):
        repo = MagicMock()
        repo.obtener_lista.return_value = ([], 0)

        uc = ObtenerProductosUC(repo=repo)
        items, total = uc.ejecutar()

        repo.obtener_lista.assert_called_once_with(
            pagina=1, por_pag=10, busqueda=""
        )
        assert items == []
        assert total == 0

    def test_pagina_menor_a_uno_lanza_error(self):
        uc = ObtenerProductosUC(repo=MagicMock())
        with pytest.raises(ValueError, match="mayor a 0"):
            uc.ejecutar(pagina=0)

    def test_por_pag_mayor_a_100_lanza_error(self):
        uc = ObtenerProductosUC(repo=MagicMock())
        with pytest.raises(ValueError, match="entre 1 y 100"):
            uc.ejecutar(por_pag=101)


# ── Tests de autenticación ────────────────────────────────────────────

class TestRegistrarUsuarioUC:

    def test_crea_usuario_con_hash(self):
        repo   = MagicMock()
        hasher = _HasherFake()
        repo.obtener_por_email.return_value = None
        repo.crear.return_value = _usuario_base()

        uc = RegistrarUsuarioUC(repo=repo, hasher=hasher)
        uc.ejecutar(email="ana@ejemplo.com", nombre="Ana", password="secreto123")

        repo.crear.assert_called_once_with(
            email="ana@ejemplo.com",
            nombre="Ana",
            hash_password="hash:secreto123",
        )

    def test_email_duplicado_lanza_error(self):
        repo   = MagicMock()
        repo.obtener_por_email.return_value = _usuario_base()

        uc = RegistrarUsuarioUC(repo=repo, hasher=_HasherFake())
        with pytest.raises(ValueError, match="ya está registrado"):
            uc.ejecutar(email="ana@ejemplo.com", nombre="Ana", password="secreto123")

    def test_password_corta_lanza_error(self):
        repo = MagicMock()
        repo.obtener_por_email.return_value = None

        uc = RegistrarUsuarioUC(repo=repo, hasher=_HasherFake())
        with pytest.raises(ValueError, match="8 caracteres"):
            uc.ejecutar(email="x@y.com", nombre="X", password="abc")


class TestAutenticarUsuarioUC:

    def test_credenciales_correctas_devuelven_usuario(self):
        repo   = MagicMock()
        hasher = _HasherFake()
        repo.obtener_por_email.return_value = _usuario_base(
            hash_password="hash:mi_password"
        )

        uc = AutenticarUsuarioUC(repo=repo, hasher=hasher)
        usuario = uc.ejecutar(email="ana@ejemplo.com", password="mi_password")

        assert usuario.email == "ana@ejemplo.com"

    def test_password_incorrecta_lanza_error(self):
        repo = MagicMock()
        repo.obtener_por_email.return_value = _usuario_base(
            hash_password="hash:correcta"
        )

        uc = AutenticarUsuarioUC(repo=repo, hasher=_HasherFake())
        with pytest.raises(ValueError, match="Credenciales inválidas"):
            uc.ejecutar(email="ana@ejemplo.com", password="incorrecta")
```

### `tests/app/test_routers.py`

```python
"""Tests de integración — prueban los routers de FastAPI contra la BD en memoria."""

from __future__ import annotations

from decimal import Decimal

import pytest

from tienda.data.modelos_db import ProductoDB, UsuarioDB
from tienda.app.seguridad import hashear_password


# ── Helpers para poblar la BD de test ─────────────────────────────────

def _crear_producto(db, nombre="Teclado", precio=89.99, stock=10, activo=True):
    p = ProductoDB(
        nombre=nombre, precio=Decimal(str(precio)),
        categoria="Periféricos", activo=activo, stock=stock,
    )
    db.add(p)
    db.commit()
    db.refresh(p)
    return p


def _crear_usuario(db, email="ana@test.com", password="secreto123"):
    u = UsuarioDB(
        email=email, nombre="Ana",
        hash_password=hashear_password(password),
    )
    db.add(u)
    db.commit()
    db.refresh(u)
    return u


# ── Tests de productos ────────────────────────────────────────────────

class TestProductosRouter:

    def test_lista_vacia(self, cliente):
        resp = cliente.get("/api/productos/")
        assert resp.status_code == 200
        data = resp.json()
        assert data["total"] == 0
        assert data["items"] == []

    def test_lista_con_productos(self, cliente, db):
        _crear_producto(db, nombre="Teclado")
        _crear_producto(db, nombre="Ratón")

        resp = cliente.get("/api/productos/")
        assert resp.status_code == 200
        assert resp.json()["total"] == 2

    def test_busqueda_filtra_por_nombre(self, cliente, db):
        _crear_producto(db, nombre="Teclado mecánico")
        _crear_producto(db, nombre="Monitor 4K")

        resp = cliente.get("/api/productos/?busqueda=teclado")
        assert resp.json()["total"] == 1
        assert resp.json()["items"][0]["nombre"] == "Teclado mecánico"

    def test_producto_no_encontrado_devuelve_404(self, cliente):
        resp = cliente.get("/api/productos/999/")
        assert resp.status_code == 404

    def test_producto_inactivo_no_aparece_en_lista(self, cliente, db):
        _crear_producto(db, nombre="Activo",   activo=True)
        _crear_producto(db, nombre="Inactivo", activo=False)

        resp = cliente.get("/api/productos/")
        assert resp.json()["total"] == 1


# ── Tests de autenticación ────────────────────────────────────────────

class TestAuthRouter:

    def test_registro_exitoso(self, cliente):
        resp = cliente.post("/api/auth/registro/", json={
            "email": "nuevo@test.com",
            "nombre": "Nuevo",
            "password": "secreto123",
        })
        assert resp.status_code == 201
        assert resp.json()["email"] == "nuevo@test.com"

    def test_registro_email_duplicado(self, cliente, db):
        _crear_usuario(db, email="ana@test.com")
        resp = cliente.post("/api/auth/registro/", json={
            "email": "ana@test.com",
            "nombre": "Ana2",
            "password": "secreto123",
        })
        assert resp.status_code == 400

    def test_login_exitoso_devuelve_token(self, cliente, db):
        _crear_usuario(db, email="ana@test.com", password="secreto123")
        resp = cliente.post("/api/auth/login/", data={
            "username": "ana@test.com",
            "password": "secreto123",
        })
        assert resp.status_code == 200
        assert "access_token" in resp.json()

    def test_login_password_incorrecta(self, cliente, db):
        _crear_usuario(db, email="ana@test.com", password="correcta")
        resp = cliente.post("/api/auth/login/", data={
            "username": "ana@test.com",
            "password": "incorrecta",
        })
        assert resp.status_code == 401
```

```bash
# Ejecutar todos los tests con cobertura
poetry run pytest --cov=src --cov-report=html

# Solo tests unitarios
poetry run pytest tests/domain/

# Solo tests de integración
poetry run pytest tests/app/

# Con verbose
poetry run pytest -v
```

---

## Diagrama final de la arquitectura

```
src/tienda/
│
├── domain/                     ← SIN dependencias externas
│   ├── entidades.py            │  Producto, Usuario, Orden
│   ├── repositorios.py         │  IProductosRepo, IUsuariosRepo (Protocol)
│   └── casos_de_uso/           │  ObtenerProductosUC, RegistrarUsuarioUC...
│
├── data/                       ← Conoce Domain, no conoce App
│   ├── modelos_db.py           │  ProductoDB, UsuarioDB (SQLAlchemy)
│   ├── db.py                   │  engine, SesionLocal
│   └── repositorios/           │  ProductosRepo, UsuariosRepo (implementan Protocol)
│
└── app/                        ← Conoce Domain, orquesta Data
    ├── main.py                 │  FastAPI + lifespan
    ├── config.py               │  pydantic-settings
    ├── seguridad.py            │  JWT, bcrypt
    ├── dependencias.py         │  Depends() — inyección de dependencias
    ├── schemas/                │  Pydantic I/O (no son entidades de dominio)
    └── routers/                │  productos, auth, ordenes

tests/
    ├── conftest.py             ← BD en memoria, TestClient
    ├── domain/                 ← tests unitarios (MagicMock, sin BD)
    └── app/                    ← tests de integración (BD test real)
```

---

## Ejercicios propuestos

1. **Router de órdenes** — Implementa `src/tienda/app/routers/ordenes.py` con:
   `GET /api/ordenes/` (mis órdenes, requiere JWT) y `POST /api/ordenes/`
   (crear orden con lista de `{producto_id, cantidad}`). Valida stock disponible
   en el caso de uso. Descuenta stock tras confirmar la orden.

2. **Alembic** — Configura Alembic con `alembic init alembic`. Ajusta `env.py`
   para usar el `motor` de `tienda.data.db` y los `Base.metadata`. Genera la
   migración inicial con `alembic revision --autogenerate -m "tablas iniciales"`.
   Aplícala con `alembic upgrade head`. Añade una columna `descripcion: str | None`
   a `ProductoDB` y genera una segunda migración.

3. **Caché con `functools.lru_cache`** — Crea un `ProductosRepoConCache` que
   envuelva `ProductosRepo`. Cachea el resultado de `obtener_por_id` durante 60s
   usando `cachetools.TTLCache`. En el provider de FastAPI, decide cuál repo
   inyectar mediante una variable de entorno `USAR_CACHE=true`.

4. **Exportar a CSV** — Añade un endpoint `GET /api/productos/exportar.csv`
   que devuelva todos los productos en formato CSV con `StreamingResponse`.
   Usa `csv.DictWriter` y un `io.StringIO` en memoria. Incluye cabeceras de
   respuesta correctas (`Content-Disposition`, `Content-Type: text/csv`).

---

## Resumen de la página 19

- La **Capa Domain** usa `@dataclass(frozen=True)` para entidades inmutables y `Protocol` para los contratos de repositorios — sin imports de FastAPI, SQLAlchemy ni httpx.
- Los **casos de uso** encapsulan una operación de negocio y reciben sus dependencias por constructor — son testeables con `MagicMock` sin base de datos.
- La **Capa Data** implementa los `Protocol` con SQLAlchemy. El mapeo DB→Dominio ocurre en el repositorio, no en el router.
- La **Capa Application** (FastAPI) solo conoce Domain: llama casos de uso, recibe entidades, devuelve schemas Pydantic.
- `Depends()` inyecta repositorios y casos de uso sin acoplar el router a una implementación concreta — `app.dependency_overrides` permite sustituirlos en tests.
- La **BD de test** (`sqlite:///:memory:`) se crea y destruye en cada test con `autouse=True` — ningún test contamina a otro.
- Los **tests unitarios de Domain** no necesitan BD ni servidor: usan `MagicMock` para el repositorio e `_HasherFake` para el hasher.
- Los **tests de integración** usan `TestClient` con la BD en memoria — prueban el stack completo desde HTTP hasta SQLite sin levantar un servidor real.

---

## Resumen del curso

A lo largo de las 19 páginas construiste una base sólida en Python moderno:

```
Módulo  1 (págs 1-3)  → Entorno, tipos, control de flujo, funciones
Módulo  2 (pág  4)    → Colecciones: list, dict, set, comprensiones
Módulo  3 (págs 5-6)  → POO: clases, herencia, dataclasses, decoradores
Módulo  4 (págs 7-8)  → Ecosistema: Poetry, pathlib, JSON, CSV
Módulo  5 (págs 9-10) → Robustez: excepciones, logging, Pydantic v2
Módulo  6 (pág  11)   → Concurrencia: asyncio, threading, multiprocessing
Módulo  7 (págs 12-13)→ HTTP: httpx + FastAPI (básico y avanzado con JWT)
Módulo  8 (pág  14)   → Bases de datos: SQLAlchemy 2.0 + Alembic
Módulo  9 (pág  15)   → Testing: pytest, fixtures, mocks, coverage
Módulo 10 (pág  16)   → Herramientas: Ruff, pre-commit, Typer, Rich
Módulo 11 (pág  17)   → Patrones: generadores, Protocol, design patterns
Módulo 12 (págs 18-19)→ Arquitectura: Clean Architecture + proyecto integrador
```

### Próximos pasos recomendados

```
Backend avanzado    →  Django + Django REST Framework
                       gRPC con grpcio
                       GraphQL con Strawberry

Datos y ML          →  NumPy, Pandas, Matplotlib
                       scikit-learn, PyTorch, Transformers

Infraestructura     →  Docker + docker-compose
                       PostgreSQL en producción
                       Redis para caché y Celery para tareas

Observabilidad      →  OpenTelemetry, Prometheus, Grafana
                       Sentry para errores en producción
```
