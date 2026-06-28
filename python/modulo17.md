# Tutorial Python — Página 17
## Módulo 8 · Bases de Datos
### SQLAlchemy 2.0, SQLite, PostgreSQL y migraciones con Alembic

---

## SQLAlchemy 2.0 — Core vs ORM

```
SQLAlchemy 2.0 tiene dos capas complementarias:
────────────────────────────────────────────────────────────────────
Core (bajo nivel)            ORM (alto nivel)
───────────────────          ────────────────────────────────────────
SQL expresivo en Python      Modelos Python ↔ tablas SQL
insert(), select(), update() session.add(), session.get(), select()
Control total del SQL        Abstracción de relaciones
Ideal: bulk inserts,         Ideal: CRUD de entidades,
       queries complejas,           relaciones complejas
       rendimiento crítico          productividad

SQLAlchemy 1.x → 2.0 cambios clave:
  - session.query(Model) → session.execute(select(Model))
  - Mapped[] y mapped_column() en lugar de Column()
  - with Session() como context manager obligatorio
  - result.scalars().all() para obtener objetos ORM
```

### Instalación

```bash
# SQLAlchemy con driver para SQLite (incluido en Python)
pip install sqlalchemy

# Con driver para PostgreSQL
pip install sqlalchemy psycopg2-binary

# O con driver async para PostgreSQL
pip install sqlalchemy asyncpg

# Para las migraciones
pip install alembic
```

---

## Definir modelos con DeclarativeBase y Mapped[]

```python
# models.py
from sqlalchemy import (
    String, Integer, Float, Boolean, DateTime, Text,
    ForeignKey, func,
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
)
from datetime import datetime

# ── Base declarativa — todos los modelos heredan de aquí ──────────

class Base(DeclarativeBase):
    """Clase base para todos los modelos ORM."""
    pass

# ── Modelo de Usuario ─────────────────────────────────────────────

class Usuario(Base):
    __tablename__ = "usuarios"

    # Mapped[tipo] define el tipo Python y la columna SQL
    # mapped_column() reemplaza Column() de SQLAlchemy 1.x
    id:         Mapped[int]      = mapped_column(primary_key=True,
                                                  autoincrement=True)
    nombre:     Mapped[str]      = mapped_column(String(100))
    email:      Mapped[str]      = mapped_column(String(200), unique=True,
                                                  index=True)
    hash_pass:  Mapped[str]      = mapped_column(String(200))
    activo:     Mapped[bool]     = mapped_column(Boolean, default=True)
    rol:        Mapped[str]      = mapped_column(String(20), default="user")
    creado_en:  Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    # Mapped[tipo | None] → columna nullable
    ultimo_login: Mapped[datetime | None] = mapped_column(DateTime,
                                                            nullable=True)

    # Relación uno-a-muchos: un usuario tiene muchas tareas
    # back_populates enlaza la relación en el lado opuesto
    tareas: Mapped[list["Tarea"]] = relationship(
        "Tarea",
        back_populates="usuario",
        cascade="all, delete-orphan",    # borrar tareas al borrar usuario
        lazy="select",                   # carga lazy por defecto
    )

    def __repr__(self) -> str:
        return f"<Usuario id={self.id} email={self.email!r}>"

# ── Modelo de Tarea ───────────────────────────────────────────────

class Tarea(Base):
    __tablename__ = "tareas"

    id:          Mapped[int]      = mapped_column(primary_key=True,
                                                   autoincrement=True)
    titulo:      Mapped[str]      = mapped_column(String(200))
    descripcion: Mapped[str | None] = mapped_column(Text, nullable=True)
    completada:  Mapped[bool]     = mapped_column(Boolean, default=False)
    prioridad:   Mapped[int]      = mapped_column(Integer, default=1)  # 1-5
    creado_en:   Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    completado_en: Mapped[datetime | None] = mapped_column(DateTime,
                                                             nullable=True)

    # ForeignKey referencia la tabla y columna padre
    usuario_id: Mapped[int] = mapped_column(
        ForeignKey("usuarios.id", ondelete="CASCADE"),
        index=True,
    )

    # Relación muchos-a-uno: cada tarea pertenece a un usuario
    usuario: Mapped["Usuario"] = relationship(
        "Usuario",
        back_populates="tareas",
    )

    def __repr__(self) -> str:
        estado = "✓" if self.completada else "○"
        return f"<Tarea {estado} id={self.id} {self.titulo!r}>"
```

---

## Engine, Session y create_all

```python
# database.py
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker, Session
from models import Base

# ── Crear el engine ───────────────────────────────────────────────

# SQLite — archivo local (desarrollo)
DATABASE_URL_SQLITE = "sqlite:///./app.db"

# PostgreSQL — producción
# DATABASE_URL_PG = "postgresql://usuario:password@localhost:5432/midb"

# echo=True loggea todos los SQL generados — útil en desarrollo
engine = create_engine(
    DATABASE_URL_SQLITE,
    echo=True,
    connect_args={"check_same_thread": False},  # solo necesario en SQLite
)

# ── SessionMaker — fábrica de sesiones ───────────────────────────

# autocommit=False → las transacciones se manejan manualmente
# autoflush=False  → no sincroniza automáticamente antes de cada query
SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False,
    expire_on_commit=False,   # los objetos siguen accesibles tras commit
)

# ── Crear tablas (solo para desarrollo — en prod usar Alembic) ────

def crear_tablas() -> None:
    """Crea todas las tablas definidas en los modelos."""
    Base.metadata.create_all(bind=engine)
    print("Tablas creadas")

def verificar_conexion() -> bool:
    """Verifica que la conexión a la BD funcione."""
    try:
        with engine.connect() as conn:
            conn.execute(text("SELECT 1"))
        return True
    except Exception as e:
        print(f"Error de conexión: {e}")
        return False
```

---

## CRUD completo — add, commit, get, select, update, delete

```python
# crud.py
from sqlalchemy.orm import Session
from sqlalchemy import select
from models import Usuario, Tarea
from datetime import datetime, timezone

# ── CREATE ────────────────────────────────────────────────────────

def crear_usuario(
    db: Session,
    nombre: str,
    email: str,
    hash_pass: str,
) -> Usuario:
    """Crea y persiste un nuevo usuario."""
    usuario = Usuario(nombre=nombre, email=email, hash_pass=hash_pass)
    db.add(usuario)
    db.commit()
    db.refresh(usuario)    # recarga desde BD para obtener id, timestamps, etc.
    return usuario

def crear_tarea(
    db: Session,
    usuario_id: int,
    titulo: str,
    descripcion: str | None = None,
    prioridad: int = 1,
) -> Tarea:
    tarea = Tarea(
        titulo=titulo,
        descripcion=descripcion,
        prioridad=prioridad,
        usuario_id=usuario_id,
    )
    db.add(tarea)
    db.commit()
    db.refresh(tarea)
    return tarea

# ── READ ──────────────────────────────────────────────────────────

def obtener_usuario_por_id(db: Session, user_id: int) -> Usuario | None:
    """Obtiene un usuario por su PK — usa el caché de identidad."""
    return db.get(Usuario, user_id)    # forma óptima para PK

def obtener_usuario_por_email(db: Session, email: str) -> Usuario | None:
    """Busca un usuario por email usando select()."""
    stmt = select(Usuario).where(Usuario.email == email)
    return db.scalars(stmt).first()    # None si no existe

def listar_usuarios(
    db: Session,
    activos: bool | None = None,
    limite: int = 100,
    offset: int = 0,
) -> list[Usuario]:
    stmt = select(Usuario)
    if activos is not None:
        stmt = stmt.where(Usuario.activo == activos)
    stmt = stmt.order_by(Usuario.id).limit(limite).offset(offset)
    return list(db.scalars(stmt).all())

def listar_tareas_usuario(
    db: Session,
    usuario_id: int,
    solo_pendientes: bool = False,
    pagina: int = 1,
    por_pagina: int = 20,
) -> list[Tarea]:
    stmt = (
        select(Tarea)
        .where(Tarea.usuario_id == usuario_id)
    )
    if solo_pendientes:
        stmt = stmt.where(Tarea.completada == False)
    stmt = (
        stmt
        .order_by(Tarea.prioridad.desc(), Tarea.creado_en.desc())
        .limit(por_pagina)
        .offset((pagina - 1) * por_pagina)
    )
    return list(db.scalars(stmt).all())

# ── UPDATE ────────────────────────────────────────────────────────

def completar_tarea(db: Session, tarea_id: int) -> Tarea | None:
    """Marca una tarea como completada y guarda la fecha."""
    tarea = db.get(Tarea, tarea_id)
    if not tarea:
        return None
    tarea.completada    = True
    tarea.completado_en = datetime.now(timezone.utc)
    db.commit()
    db.refresh(tarea)
    return tarea

def actualizar_tarea(
    db: Session,
    tarea_id: int,
    **campos,
) -> Tarea | None:
    """Actualiza campos arbitrarios de una tarea."""
    tarea = db.get(Tarea, tarea_id)
    if not tarea:
        return None
    for campo, valor in campos.items():
        if hasattr(tarea, campo):
            setattr(tarea, campo, valor)
    db.commit()
    db.refresh(tarea)
    return tarea

# ── DELETE ────────────────────────────────────────────────────────

def eliminar_tarea(db: Session, tarea_id: int) -> bool:
    """Elimina una tarea. Retorna True si existía."""
    tarea = db.get(Tarea, tarea_id)
    if not tarea:
        return False
    db.delete(tarea)
    db.commit()
    return True
```

---

## Filtros avanzados — where, and_, or_, like, in_

```python
from sqlalchemy import select, and_, or_, func
from sqlalchemy.orm import Session
from models import Tarea, Usuario

def buscar_tareas(
    db: Session,
    usuario_id: int,
    texto:     str | None    = None,
    prioridades: list[int] | None = None,
    completada: bool | None  = None,
) -> list[Tarea]:
    """Búsqueda avanzada de tareas con filtros combinados."""

    # Condiciones que se van acumulando
    condiciones = [Tarea.usuario_id == usuario_id]

    if texto:
        # like() con % para búsqueda parcial (case-insensitive con ilike en PG)
        condiciones.append(
            or_(
                Tarea.titulo.like(f"%{texto}%"),
                Tarea.descripcion.like(f"%{texto}%"),
            )
        )

    if prioridades:
        # in_() equivale a WHERE prioridad IN (1, 2, 3)
        condiciones.append(Tarea.prioridad.in_(prioridades))

    if completada is not None:
        condiciones.append(Tarea.completada == completada)

    # and_() combina todas las condiciones con AND
    stmt = (
        select(Tarea)
        .where(and_(*condiciones))
        .order_by(Tarea.prioridad.desc())
    )
    return list(db.scalars(stmt).all())

def contar_tareas_por_usuario(db: Session) -> list[dict]:
    """Agrupa y cuenta tareas por usuario."""
    stmt = (
        select(
            Usuario.nombre,
            func.count(Tarea.id).label("total_tareas"),
            func.sum(Tarea.completada.cast(Integer)).label("completadas"),
        )
        .join(Tarea, Tarea.usuario_id == Usuario.id, isouter=True)
        .group_by(Usuario.id, Usuario.nombre)
        .order_by(func.count(Tarea.id).desc())
    )
    # execute() para queries que no retornan modelos ORM directamente
    rows = db.execute(stmt).all()
    return [
        {"nombre": r.nombre, "total": r.total_tareas,
         "completadas": r.completadas or 0}
        for r in rows
    ]
```

---

## Paginación y relaciones con eager loading

```python
from sqlalchemy import select, func
from sqlalchemy.orm import Session, selectinload, joinedload
from models import Usuario, Tarea

# ── Paginación ────────────────────────────────────────────────────

def paginar_tareas(
    db: Session,
    usuario_id: int,
    pagina: int = 1,
    por_pagina: int = 20,
) -> dict:
    """Retorna una página de tareas con metadatos de paginación."""

    # Contar total de registros
    total_stmt = (
        select(func.count())
        .select_from(Tarea)
        .where(Tarea.usuario_id == usuario_id)
    )
    total: int = db.scalar(total_stmt) or 0   # scalar() retorna primer valor

    # Obtener la página
    items_stmt = (
        select(Tarea)
        .where(Tarea.usuario_id == usuario_id)
        .order_by(Tarea.creado_en.desc())
        .offset((pagina - 1) * por_pagina)
        .limit(por_pagina)
    )
    items = list(db.scalars(items_stmt).all())

    return {
        "total":        total,
        "pagina":       pagina,
        "por_pagina":   por_pagina,
        "total_paginas": -(-total // por_pagina),  # ceil
        "items":        items,
    }

# ── Eager loading — evitar el problema N+1 ───────────────────────

def obtener_usuarios_con_tareas(db: Session) -> list[Usuario]:
    """
    Carga usuarios con sus tareas en una sola query (JOIN).
    Sin eager loading: N queries (1 por usuario) — problema N+1.
    Con selectinload: 2 queries (1 usuarios, 1 tareas) — eficiente.
    """
    stmt = (
        select(Usuario)
        # selectinload: SELECT separado para la relación — mejor para listas
        .options(selectinload(Usuario.tareas))
        .where(Usuario.activo == True)
        .order_by(Usuario.nombre)
    )
    return list(db.scalars(stmt).all())

def obtener_tarea_con_usuario(db: Session, tarea_id: int) -> Tarea | None:
    """Carga la tarea con su usuario en un solo JOIN."""
    stmt = (
        select(Tarea)
        # joinedload: un solo JOIN — mejor para relaciones muchos-a-uno
        .options(joinedload(Tarea.usuario))
        .where(Tarea.id == tarea_id)
    )
    return db.scalars(stmt).first()
```

---

## Alembic — migraciones de base de datos

```bash
# Inicializar Alembic en el proyecto
alembic init alembic

# Estructura creada:
# alembic/
#   env.py          ← configuración de las migraciones
#   script.py.mako  ← plantilla para archivos de migración
#   versions/       ← aquí se guardan las migraciones
# alembic.ini       ← configuración principal
```

```python
# alembic/env.py — configuración de Alembic

from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# Importar los modelos para que Alembic los detecte
from models import Base   # Base declarativa con todos los modelos

config = context.config
fileConfig(config.config_file_name)

# target_metadata le dice a Alembic qué tablas manejar
target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """Genera el SQL sin conectarse a la BD."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    """Aplica las migraciones conectándose a la BD."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
        )
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```bash
# alembic.ini — cambiar la URL de conexión
# sqlalchemy.url = sqlite:///./app.db
# sqlalchemy.url = postgresql://user:pass@localhost/midb

# Crear una migración automática (detecta diferencias con los modelos)
alembic revision --autogenerate -m "crear tablas usuarios y tareas"

# Aplicar todas las migraciones pendientes
alembic upgrade head

# Ver estado actual
alembic current

# Retroceder una migración
alembic downgrade -1

# Ver historial
alembic history --verbose
```

```python
# alembic/versions/001_crear_tablas_usuarios_y_tareas.py
# Archivo generado automáticamente por --autogenerate

from alembic import op
import sqlalchemy as sa

revision = '001abc'
down_revision = None    # primera migración
branch_labels = None
depends_on = None

def upgrade() -> None:
    """Crea las tablas usuarios y tareas."""
    op.create_table(
        "usuarios",
        sa.Column("id",           sa.Integer(),     nullable=False),
        sa.Column("nombre",       sa.String(100),   nullable=False),
        sa.Column("email",        sa.String(200),   nullable=False),
        sa.Column("hash_pass",    sa.String(200),   nullable=False),
        sa.Column("activo",       sa.Boolean(),     nullable=False),
        sa.Column("rol",          sa.String(20),    nullable=False),
        sa.Column("creado_en",    sa.DateTime(),    server_default=sa.func.now()),
        sa.Column("ultimo_login", sa.DateTime(),    nullable=True),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("email"),
    )
    op.create_index("ix_usuarios_email", "usuarios", ["email"])

    op.create_table(
        "tareas",
        sa.Column("id",           sa.Integer(),     nullable=False),
        sa.Column("titulo",       sa.String(200),   nullable=False),
        sa.Column("descripcion",  sa.Text(),        nullable=True),
        sa.Column("completada",   sa.Boolean(),     nullable=False),
        sa.Column("prioridad",    sa.Integer(),     nullable=False),
        sa.Column("creado_en",    sa.DateTime(),    server_default=sa.func.now()),
        sa.Column("completado_en", sa.DateTime(),   nullable=True),
        sa.Column("usuario_id",   sa.Integer(),     nullable=False),
        sa.ForeignKeyConstraint(["usuario_id"], ["usuarios.id"],
                                 ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_tareas_usuario_id", "tareas", ["usuario_id"])

def downgrade() -> None:
    """Elimina las tablas en orden inverso (respetando FK)."""
    op.drop_table("tareas")
    op.drop_table("usuarios")
```

---

## Integración con FastAPI — dependencia de sesión

```python
# fastapi_db.py
# Patrón estándar para inyectar la sesión de BD en FastAPI

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import Annotated, Generator
from database import SessionLocal, engine
from models import Base

# ── Dependencia generadora de sesión ─────────────────────────────

def obtener_db() -> Generator[Session, None, None]:
    """
    Crea una sesión por request y la cierra al terminar.
    yield permite que FastAPI gestione el ciclo de vida.
    """
    db = SessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()    # rollback automático si hay error
        raise
    finally:
        db.close()       # siempre cerrar la sesión

# Tipo reutilizable para los endpoints
DBDep = Annotated[Session, Depends(obtener_db)]

app = FastAPI()

# ── Crear tablas al iniciar (solo en desarrollo) ──────────────────

@app.on_event("startup")   # en prod usar lifespan + alembic upgrade head
def startup() -> None:
    Base.metadata.create_all(bind=engine)
```

---

## Ejemplo aplicado — API de To-Do con FastAPI + SQLAlchemy + Alembic

```python
"""
todo_api.py — API completa de tareas con autenticación y base de datos
Ejecutar: uvicorn todo_api:app --reload
"""

from __future__ import annotations

from contextlib import asynccontextmanager
from datetime import datetime, timezone
from typing import Annotated, Generator

from fastapi import (FastAPI, Depends, HTTPException, APIRouter,
                     status, Query, Path)
from passlib.context import CryptContext
from pydantic import BaseModel, Field
from sqlalchemy import (create_engine, select, func, String, Integer,
                        Boolean, DateTime, Text, ForeignKey)
from sqlalchemy.orm import (DeclarativeBase, Mapped, mapped_column,
                             relationship, Session, sessionmaker,
                             selectinload)

# ── Base de datos ─────────────────────────────────────────────────

DATABASE_URL = "sqlite:///./todo.db"

engine       = create_engine(DATABASE_URL, echo=False,
                              connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autocommit=False,
                             autoflush=False, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# ── Modelos ORM ───────────────────────────────────────────────────

class UsuarioDB(Base):
    __tablename__ = "usuarios"

    id:        Mapped[int]  = mapped_column(primary_key=True)
    nombre:    Mapped[str]  = mapped_column(String(100))
    email:     Mapped[str]  = mapped_column(String(200), unique=True, index=True)
    hash_pass: Mapped[str]  = mapped_column(String(200))
    activo:    Mapped[bool] = mapped_column(Boolean, default=True)

    tareas: Mapped[list["TareaDB"]] = relationship(
        back_populates="usuario",
        cascade="all, delete-orphan",
        lazy="select",
    )

class TareaDB(Base):
    __tablename__ = "tareas"

    id:           Mapped[int]          = mapped_column(primary_key=True)
    titulo:       Mapped[str]          = mapped_column(String(200))
    descripcion:  Mapped[str | None]   = mapped_column(Text, nullable=True)
    completada:   Mapped[bool]         = mapped_column(Boolean, default=False)
    prioridad:    Mapped[int]          = mapped_column(Integer, default=1)
    creado_en:    Mapped[datetime]     = mapped_column(
        DateTime, default=lambda: datetime.now(timezone.utc)
    )
    completado_en: Mapped[datetime | None] = mapped_column(DateTime,
                                                            nullable=True)
    usuario_id:   Mapped[int]          = mapped_column(
        ForeignKey("usuarios.id", ondelete="CASCADE"), index=True
    )

    usuario: Mapped["UsuarioDB"] = relationship(back_populates="tareas")

# ── Esquemas Pydantic ─────────────────────────────────────────────

class UsuarioCrear(BaseModel):
    nombre:   str = Field(min_length=2, max_length=100)
    email:    str = Field(min_length=5)
    password: str = Field(min_length=8)

class UsuarioRespuesta(BaseModel):
    id:     int
    nombre: str
    email:  str

    model_config = {"from_attributes": True}    # permite crear desde ORM

class TareaCrear(BaseModel):
    titulo:      str          = Field(min_length=1, max_length=200)
    descripcion: str | None   = None
    prioridad:   int          = Field(default=1, ge=1, le=5)

class TareaActualizar(BaseModel):
    titulo:      str | None = Field(None, min_length=1, max_length=200)
    descripcion: str | None = None
    prioridad:   int | None = Field(None, ge=1, le=5)

class TareaRespuesta(BaseModel):
    id:           int
    titulo:       str
    descripcion:  str | None
    completada:   bool
    prioridad:    int
    creado_en:    datetime
    completado_en: datetime | None
    usuario_id:   int

    model_config = {"from_attributes": True}

class PaginaTareas(BaseModel):
    total:        int
    pagina:       int
    por_pagina:   int
    total_paginas: int
    items:        list[TareaRespuesta]

# ── Utilidades ────────────────────────────────────────────────────

pwd_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")

def obtener_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()

DBDep = Annotated[Session, Depends(obtener_db)]

# Autenticación simplificada (sin JWT para este ejemplo)
def get_usuario_actual(
    db: DBDep,
    user_id: Annotated[int, Query(description="ID del usuario autenticado")],
) -> UsuarioDB:
    usuario = db.get(UsuarioDB, user_id)
    if not usuario or not usuario.activo:
        raise HTTPException(401, "Usuario no autenticado")
    return usuario

UsuarioActualDep = Annotated[UsuarioDB, Depends(get_usuario_actual)]

# ── Lifespan ──────────────────────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)
    # Seed de usuario demo
    with SessionLocal() as db:
        if not db.scalars(select(UsuarioDB)).first():
            demo = UsuarioDB(
                nombre="Demo User",
                email="demo@ejemplo.com",
                hash_pass=pwd_ctx.hash("demo1234"),
            )
            db.add(demo)
            db.commit()
            print("Usuario demo creado: demo@ejemplo.com / demo1234")
    yield
    print("API apagada")

# ── Routers ───────────────────────────────────────────────────────

router_usuarios = APIRouter(prefix="/usuarios", tags=["Usuarios"])
router_tareas   = APIRouter(prefix="/tareas",   tags=["Tareas"])

# ── Endpoints de Usuarios ─────────────────────────────────────────

@router_usuarios.post("/", response_model=UsuarioRespuesta, status_code=201)
def crear_usuario(datos: UsuarioCrear, db: DBDep) -> UsuarioDB:
    if db.scalars(
        select(UsuarioDB).where(UsuarioDB.email == datos.email)
    ).first():
        raise HTTPException(400, "El email ya está registrado")

    usuario = UsuarioDB(
        nombre=datos.nombre,
        email=datos.email,
        hash_pass=pwd_ctx.hash(datos.password),
    )
    db.add(usuario)
    db.commit()
    db.refresh(usuario)
    return usuario

@router_usuarios.get("/{user_id}", response_model=UsuarioRespuesta)
def obtener_usuario(
    user_id: Annotated[int, Path(ge=1)],
    db: DBDep,
) -> UsuarioDB:
    usuario = db.get(UsuarioDB, user_id)
    if not usuario:
        raise HTTPException(404, f"Usuario {user_id} no encontrado")
    return usuario

@router_usuarios.get("/{user_id}/estadisticas")
def estadisticas_usuario(
    user_id: Annotated[int, Path(ge=1)],
    db: DBDep,
) -> dict:
    """Retorna estadísticas de tareas del usuario."""
    usuario = db.get(UsuarioDB, user_id)
    if not usuario:
        raise HTTPException(404, "Usuario no encontrado")

    stmt = (
        select(
            func.count(TareaDB.id).label("total"),
            func.sum(TareaDB.completada.cast(Integer)).label("completadas"),
        )
        .where(TareaDB.usuario_id == user_id)
    )
    row = db.execute(stmt).one()
    total      = row.total or 0
    completadas = int(row.completadas or 0)

    return {
        "usuario":     usuario.nombre,
        "total":       total,
        "completadas": completadas,
        "pendientes":  total - completadas,
        "porcentaje":  round(completadas / total * 100, 1) if total else 0,
    }

# ── Endpoints de Tareas ───────────────────────────────────────────

@router_tareas.get("/", response_model=PaginaTareas)
def listar_tareas(
    usuario: UsuarioActualDep,
    db:      DBDep,
    pagina:          Annotated[int, Query(ge=1)]       = 1,
    por_pagina:      Annotated[int, Query(ge=1, le=50)] = 20,
    solo_pendientes: bool = False,
    prioridad_min:   Annotated[int | None, Query(ge=1, le=5)] = None,
) -> dict:
    condiciones = [TareaDB.usuario_id == usuario.id]
    if solo_pendientes:
        condiciones.append(TareaDB.completada == False)
    if prioridad_min is not None:
        condiciones.append(TareaDB.prioridad >= prioridad_min)

    from sqlalchemy import and_
    # Total
    total: int = db.scalar(
        select(func.count()).select_from(TareaDB).where(and_(*condiciones))
    ) or 0

    # Página
    items = list(db.scalars(
        select(TareaDB)
        .where(and_(*condiciones))
        .order_by(TareaDB.prioridad.desc(), TareaDB.creado_en.desc())
        .offset((pagina - 1) * por_pagina)
        .limit(por_pagina)
    ).all())

    return {
        "total":        total,
        "pagina":       pagina,
        "por_pagina":   por_pagina,
        "total_paginas": -(-total // por_pagina),
        "items":        items,
    }

@router_tareas.post("/", response_model=TareaRespuesta, status_code=201)
def crear_tarea(
    datos:   TareaCrear,
    usuario: UsuarioActualDep,
    db:      DBDep,
) -> TareaDB:
    tarea = TareaDB(
        titulo=datos.titulo,
        descripcion=datos.descripcion,
        prioridad=datos.prioridad,
        usuario_id=usuario.id,
    )
    db.add(tarea)
    db.commit()
    db.refresh(tarea)
    return tarea

@router_tareas.get("/{tarea_id}", response_model=TareaRespuesta)
def obtener_tarea(
    tarea_id: Annotated[int, Path(ge=1)],
    usuario:  UsuarioActualDep,
    db:       DBDep,
) -> TareaDB:
    tarea = db.get(TareaDB, tarea_id)
    if not tarea:
        raise HTTPException(404, f"Tarea {tarea_id} no encontrada")
    if tarea.usuario_id != usuario.id:
        raise HTTPException(403, "Sin permisos para esta tarea")
    return tarea

@router_tareas.patch("/{tarea_id}", response_model=TareaRespuesta)
def actualizar_tarea(
    tarea_id: Annotated[int, Path(ge=1)],
    datos:    TareaActualizar,
    usuario:  UsuarioActualDep,
    db:       DBDep,
) -> TareaDB:
    tarea = db.get(TareaDB, tarea_id)
    if not tarea:
        raise HTTPException(404, "Tarea no encontrada")
    if tarea.usuario_id != usuario.id:
        raise HTTPException(403, "Sin permisos")

    # Actualizar solo los campos enviados
    for campo, valor in datos.model_dump(exclude_unset=True).items():
        setattr(tarea, campo, valor)

    db.commit()
    db.refresh(tarea)
    return tarea

@router_tareas.post("/{tarea_id}/completar", response_model=TareaRespuesta)
def completar_tarea(
    tarea_id: Annotated[int, Path(ge=1)],
    usuario:  UsuarioActualDep,
    db:       DBDep,
) -> TareaDB:
    tarea = db.get(TareaDB, tarea_id)
    if not tarea:
        raise HTTPException(404, "Tarea no encontrada")
    if tarea.usuario_id != usuario.id:
        raise HTTPException(403, "Sin permisos")
    if tarea.completada:
        raise HTTPException(400, "La tarea ya está completada")

    tarea.completada    = True
    tarea.completado_en = datetime.now(timezone.utc)
    db.commit()
    db.refresh(tarea)
    return tarea

@router_tareas.delete("/{tarea_id}", status_code=204)
def eliminar_tarea(
    tarea_id: Annotated[int, Path(ge=1)],
    usuario:  UsuarioActualDep,
    db:       DBDep,
) -> None:
    tarea = db.get(TareaDB, tarea_id)
    if not tarea:
        raise HTTPException(404, "Tarea no encontrada")
    if tarea.usuario_id != usuario.id:
        raise HTTPException(403, "Sin permisos")
    db.delete(tarea)
    db.commit()

# ── App principal ─────────────────────────────────────────────────

app = FastAPI(
    title="To-Do API",
    description="API de gestión de tareas con FastAPI + SQLAlchemy 2.0",
    version="1.0.0",
    lifespan=lifespan,
)

app.include_router(router_usuarios)
app.include_router(router_tareas)

@app.get("/", tags=["Salud"])
def raiz() -> dict:
    return {
        "api": "To-Do API v1.0",
        "docs": "/docs",
        "nota": "Pasar ?user_id=1 en tareas para autenticar",
    }
```

---

## Ejercicios propuestos

1. **Soft delete** — Modifica el modelo `TareaDB` para añadir una columna
   `eliminada_en: Mapped[datetime | None]`. En lugar de borrar registros,
   el endpoint DELETE debe marcar `eliminada_en = datetime.now()`. Añade
   un filtro global (`where(TareaDB.eliminada_en == None)`) a todos los
   `select()` para que las tareas eliminadas no aparezcan. Crea un endpoint
   `GET /tareas/papelera` que liste solo las eliminadas.

2. **Migración nueva columna** — Añade el campo `etiquetas: Mapped[str | None]`
   al modelo `TareaDB` para almacenar etiquetas separadas por coma (ej: `"trabajo,urgente"`).
   Genera la migración con `alembic revision --autogenerate -m "add etiquetas to tareas"`,
   revisa el archivo generado, aplícala con `alembic upgrade head` y verifica
   con `alembic current`. Añade un endpoint de búsqueda por etiqueta.

3. **Índice compuesto y full-text search** — Añade un índice compuesto en
   `(usuario_id, completada, prioridad)` al modelo `TareaDB` usando
   `__table_args__ = (Index("ix_usuario_estado_prio", "usuario_id", "completada", "prioridad"),)`.
   Genera la migración. Para SQLite, implementa búsqueda de texto con
   `Tarea.titulo.contains(texto)`. Para PostgreSQL, usa `func.to_tsvector`
   con un índice GIN.

4. **Modelo Categoría con relación muchos-a-muchos** — Crea un modelo
   `Categoria` y una tabla asociativa `tarea_categoria` para la relación
   muchos-a-muchos entre `Tarea` y `Categoria`. Usa `relationship()` con
   `secondary=tarea_categoria`. Añade endpoints para: listar tareas de una
   categoría, añadir/quitar categorías a una tarea. Genera las migraciones
   con Alembic.

---

## Resumen de la página 14

- SQLAlchemy 2.0 usa `Mapped[tipo]` y `mapped_column()` para definir columnas con type hints completos — más seguro y con mejor soporte IDE que la API de 1.x.
- `relationship()` con `back_populates` enlaza ambos lados de una relación — `cascade="all, delete-orphan"` propaga borrados automáticamente.
- `session.get(Modelo, pk)` es la forma óptima para buscar por PK usando el caché de identidad; `select()` con `where()` para cualquier otro filtro.
- `selectinload()` resuelve el problema N+1 con una segunda query; `joinedload()` hace un JOIN — preferir `selectinload` para colecciones y `joinedload` para relaciones muchos-a-uno.
- Alembic gestiona el historial de migraciones con `--autogenerate` para detectar diferencias entre modelos y BD; `upgrade head` y `downgrade -1` controlan la versión del esquema.
- La dependencia generadora `obtener_db()` con `yield`, `rollback()` en except y `close()` en finally garantiza que cada request tenga su propia sesión limpia.
- `model_dump(exclude_unset=True)` en PATCH permite actualizar solo los campos enviados — patrón esencial para implementar actualizaciones parciales correctamente.
- `func.count()`, `func.sum()` y `group_by()` permiten estadísticas directamente en SQL sin cargar todos los registros en memoria — siempre preferir agregaciones en BD.

---

> **Siguiente página →** Página 18: Testing con pytest — fixtures, mocks, parametrize y coverage.
> pytest, fixtures, mocking con `unittest.mock`, testing de APIs con
> `httpx.AsyncClient` y `TestClient`, cobertura con `pytest-cov`.
