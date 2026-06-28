# Tutorial Python — Página 16
## Módulo 7 · HTTP y APIs REST
### FastAPI avanzado — autenticación, middleware y background tasks

---

## Middleware — CORS, logging y timing

```python
# middleware.py
# Middleware se ejecuta en cada request/response — como capas de cebolla

from fastapi import FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware
import time
import logging
import uuid

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)

app = FastAPI()

# ── CORS — Cross-Origin Resource Sharing ─────────────────────────
# Necesario cuando el frontend está en un dominio diferente al backend

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",    # React dev
        "http://localhost:5173",    # Vite dev
        "https://mi-app.com",       # Producción
    ],
    allow_credentials=True,         # permite cookies y auth headers
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["*"],            # permite cualquier header
)

# ── Middleware personalizado con @app.middleware ──────────────────

@app.middleware("http")
async def middleware_logging_timing(
    request:  Request,
    call_next,          # función que llama al siguiente middleware/endpoint
) -> Response:
    """Loggea cada request con su tiempo de respuesta."""
    # Generar ID único para rastrear el request en logs
    request_id = str(uuid.uuid4())[:8]

    inicio = time.perf_counter()
    logger.info(
        f"[{request_id}] {request.method} {request.url.path} — inicio"
    )

    # Llamar al siguiente middleware o al endpoint
    try:
        response = await call_next(request)
    except Exception as e:
        logger.error(f"[{request_id}] Error no manejado: {e}")
        raise

    duracion = (time.perf_counter() - inicio) * 1000
    logger.info(
        f"[{request_id}] {request.method} {request.url.path} "
        f"→ {response.status_code} ({duracion:.1f}ms)"
    )

    # Añadir headers de diagnóstico a la respuesta
    response.headers["X-Request-Id"]      = request_id
    response.headers["X-Process-Time-Ms"] = f"{duracion:.1f}"
    return response
```

---

## Depends() — dependencias globales y por router

```python
from fastapi import FastAPI, Depends, APIRouter, Request, HTTPException
from typing import Annotated

app = FastAPI()

# ── Dependencia de base de datos simulada ─────────────────────────

class ConexionDB:
    def __init__(self, nombre: str):
        self.nombre  = nombre
        self.abierta = True

    def cerrar(self) -> None:
        self.abierta = False

def obtener_db():
    """Dependencia generadora: crea y cierra la conexión automáticamente."""
    db = ConexionDB("postgres://localhost/miapp")
    try:
        yield db         # el endpoint recibe este objeto
    finally:
        db.cerrar()      # se ejecuta al terminar el request (o si hay error)

DBDep = Annotated[ConexionDB, Depends(obtener_db)]

# ── Dependencia de headers ────────────────────────────────────────

def verificar_version_api(
    request: Request,
) -> str:
    """Verifica que el cliente envíe el header de versión correcto."""
    version = request.headers.get("X-Api-Version", "1")
    if version not in {"1", "2"}:
        raise HTTPException(400, f"Versión de API no soportada: {version}")
    return version

VersionDep = Annotated[str, Depends(verificar_version_api)]

# ── Router con dependencia aplicada a todas sus rutas ─────────────

# dependencies=[...] aplica la dependencia a TODOS los endpoints del router
router_admin = APIRouter(
    prefix="/admin",
    tags=["Admin"],
    dependencies=[Depends(verificar_version_api)],   # aplicado globalmente
)

@router_admin.get("/stats")
def estadisticas_admin(db: DBDep) -> dict:
    return {"db": db.nombre, "estado": "activo"}

# ── Dependencia global en toda la aplicación ──────────────────────

# Se ejecuta en TODOS los endpoints de la app
app.include_router(
    router_admin,
    # dependencies=[Depends(verificar_version_api)],  # también válido aquí
)

@app.get("/publico")
def endpoint_publico() -> dict:
    return {"acceso": "libre"}

@app.get("/con-db")
def endpoint_con_db(db: DBDep) -> dict:
    return {"db": db.nombre, "conexion": db.abierta}
```

---

## JWT — autenticación con tokens

```python
# auth/jwt_utils.py
# pip install PyJWT python-multipart

import jwt           # PyJWT
from datetime import datetime, timedelta, timezone
from typing import Any

# ── Configuración ─────────────────────────────────────────────────

SECRETO_JWT   = "cambiar-esto-por-una-clave-secreta-larga-y-aleatoria"
ALGORITMO     = "HS256"
EXPIRACION_MIN = 30      # minutos

# ── Crear tokens ──────────────────────────────────────────────────

def crear_access_token(
    datos:      dict[str, Any],
    expira_en:  timedelta | None = None,
) -> str:
    """Genera un JWT con los datos dados y tiempo de expiración."""
    payload = datos.copy()
    expiracion = (
        datetime.now(timezone.utc) + expira_en
        if expira_en
        else datetime.now(timezone.utc) + timedelta(minutes=EXPIRACION_MIN)
    )
    payload["exp"] = expiracion
    payload["iat"] = datetime.now(timezone.utc)     # issued at
    payload["type"] = "access"

    return jwt.encode(payload, SECRETO_JWT, algorithm=ALGORITMO)

def crear_refresh_token(user_id: int) -> str:
    """Genera un refresh token con expiración de 7 días."""
    payload = {
        "sub":  str(user_id),
        "type": "refresh",
        "exp":  datetime.now(timezone.utc) + timedelta(days=7),
        "iat":  datetime.now(timezone.utc),
    }
    return jwt.encode(payload, SECRETO_JWT, algorithm=ALGORITMO)

# ── Verificar tokens ──────────────────────────────────────────────

def verificar_token(token: str, tipo: str = "access") -> dict[str, Any]:
    """Decodifica y valida un JWT. Lanza excepción si es inválido."""
    try:
        payload = jwt.decode(token, SECRETO_JWT, algorithms=[ALGORITMO])

        if payload.get("type") != tipo:
            raise ValueError(f"Tipo de token incorrecto: se esperaba {tipo}")

        return payload

    except jwt.ExpiredSignatureError:
        raise ValueError("El token ha expirado")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Token inválido: {e}")
```

---

## OAuth2PasswordBearer y flujo de login

```python
# auth/oauth2.py

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from typing import Annotated

# OAuth2PasswordBearer extrae el token del header Authorization: Bearer <token>
# tokenUrl es la URL del endpoint de login (para Swagger UI)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

class TokenResponse(BaseModel):
    access_token:  str
    refresh_token: str
    token_type:    str = "bearer"
    expires_in:    int = 1800       # segundos

class DatosToken(BaseModel):
    sub:  str              # subject — normalmente el user_id o email
    type: str

# ── Dependencia get_usuario_actual ───────────────────────────────

async def get_usuario_actual(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> dict:
    """Extrae y valida el usuario del JWT. Inyectado como dependencia."""
    credenciales_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="No autenticado o token inválido",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        from auth.jwt_utils import verificar_token
        payload = verificar_token(token, tipo="access")
        user_id = payload.get("sub")
        if not user_id:
            raise credenciales_error
        return {"id": int(user_id), "email": payload.get("email")}
    except ValueError:
        raise credenciales_error

UsuarioActualDep = Annotated[dict, Depends(get_usuario_actual)]
```

---

## Hashing de contraseñas con passlib/bcrypt

```python
# auth/password.py
# pip install "passlib[bcrypt]"

from passlib.context import CryptContext

# CryptContext gestiona múltiples algoritmos y migraciones
# bcrypt es el estándar recomendado para contraseñas
contexto_hash = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",      # marca como obsoleto algoritmos antiguos
    bcrypt__rounds=12,      # factor de trabajo — aumentar en producción
)

def hashear_password(password_plano: str) -> str:
    """Genera hash bcrypt de la contraseña. Incluye salt automáticamente."""
    return contexto_hash.hash(password_plano)

def verificar_password(password_plano: str, hash_guardado: str) -> bool:
    """Compara contraseña plana con hash almacenado de forma segura."""
    return contexto_hash.verify(password_plano, hash_guardado)

def necesita_rehash(hash_guardado: str) -> bool:
    """Retorna True si el hash usa parámetros obsoletos y debe rehachearse."""
    return contexto_hash.needs_update(hash_guardado)

# Ejemplo de uso:
# hash = hashear_password("mi_contraseña_segura")
# es_valido = verificar_password("mi_contraseña_segura", hash)
# assert es_valido is True
```

---

## Exception handlers personalizados

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError

app = FastAPI()

# ── Excepción de dominio personalizada ───────────────────────────

class ErrorNegocio(Exception):
    def __init__(self, mensaje: str, codigo: str, status: int = 400):
        self.mensaje = mensaje
        self.codigo  = codigo
        self.status  = status
        super().__init__(mensaje)

# ── Handlers ─────────────────────────────────────────────────────

@app.exception_handler(ErrorNegocio)
async def handler_error_negocio(
    request: Request, exc: ErrorNegocio,
) -> JSONResponse:
    """Convierte excepciones de negocio en respuestas JSON estructuradas."""
    return JSONResponse(
        status_code=exc.status,
        content={
            "error": {
                "tipo":    "error_negocio",
                "codigo":  exc.codigo,
                "mensaje": exc.mensaje,
                "path":    str(request.url.path),
            }
        },
    )

@app.exception_handler(RequestValidationError)
async def handler_validacion(
    request: Request, exc: RequestValidationError,
) -> JSONResponse:
    """Formatea errores de validación Pydantic de forma amigable."""
    errores = []
    for error in exc.errors():
        campo = " → ".join(str(loc) for loc in error["loc"])
        errores.append({
            "campo":   campo,
            "mensaje": error["msg"],
            "tipo":    error["type"],
        })
    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "tipo":    "validacion",
                "mensaje": "Datos de entrada inválidos",
                "detalles": errores,
            }
        },
    )

@app.exception_handler(HTTPException)
async def handler_http(
    request: Request, exc: HTTPException,
) -> JSONResponse:
    """Formatea HTTPExceptions con estructura consistente."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "tipo":    "http",
                "status":  exc.status_code,
                "mensaje": exc.detail,
            }
        },
    )

# Uso:
@app.post("/comprar/{producto_id}")
def comprar(producto_id: int, cantidad: int) -> dict:
    if cantidad > 10:
        raise ErrorNegocio(
            mensaje="Máximo 10 unidades por compra",
            codigo="LIMITE_CANTIDAD",
            status=400,
        )
    return {"ok": True}
```

---

## Background Tasks

```python
from fastapi import FastAPI, BackgroundTasks, Depends
from pydantic import BaseModel, EmailStr
import asyncio
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

# ── Funciones de background ──────────────────────────────────────

async def enviar_email_bienvenida(email: str, nombre: str) -> None:
    """Simula el envío de email — no bloquea la respuesta HTTP."""
    await asyncio.sleep(2)     # simula latencia de SMTP
    logger.info(f"Email de bienvenida enviado a {email} ({nombre})")

def registrar_auditoria(
    accion:   str,
    user_id:  int,
    detalles: dict,
) -> None:
    """Guarda log de auditoría de forma asíncrona."""
    # En producción: escribir a base de datos, Kafka, etc.
    logger.info(f"AUDITORIA | {accion} | user={user_id} | {detalles}")

async def limpiar_tokens_expirados() -> None:
    """Limpieza periódica — útil como tarea post-logout."""
    await asyncio.sleep(0.1)
    logger.info("Tokens expirados limpiados")

# ── Endpoints que usan background tasks ──────────────────────────

class DatosRegistro(BaseModel):
    nombre: str
    email:  str      # en prod: EmailStr con pip install email-validator
    password: str

@app.post("/registro", status_code=201)
async def registrar_usuario(
    datos:      DatosRegistro,
    background: BackgroundTasks,         # FastAPI inyecta BackgroundTasks
) -> dict:
    """
    Registra usuario y envía email de bienvenida en background.
    La respuesta HTTP se devuelve ANTES de que se envíe el email.
    """
    # Simular creación en BD
    nuevo_user_id = 42

    # Añadir tarea en background — se ejecuta después de responder
    # Funciones async y sync son ambas válidas
    background.add_task(
        enviar_email_bienvenida,
        email=datos.email,
        nombre=datos.nombre,
    )
    background.add_task(
        registrar_auditoria,
        accion="REGISTRO",
        user_id=nuevo_user_id,
        detalles={"email": datos.email},
    )

    # Esta respuesta se envía INMEDIATAMENTE
    # El email se enviará después, sin que el cliente espere
    return {
        "id":      nuevo_user_id,
        "email":   datos.email,
        "mensaje": "Registro exitoso. Revisa tu email.",
    }

@app.post("/logout")
async def logout(background: BackgroundTasks) -> dict:
    background.add_task(limpiar_tokens_expirados)
    return {"mensaje": "Sesión cerrada"}
```

---

## Lifespan events — startup y shutdown

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncio
import httpx

# ── Estado global de la aplicación ───────────────────────────────
# Lifespan es la forma moderna (>= 3.9) de manejar startup/shutdown
# Reemplaza los deprecated @app.on_event("startup") y @app.on_event("shutdown")

class EstadoApp:
    """Contenedor de recursos compartidos de la aplicación."""
    http_client:  httpx.AsyncClient | None = None
    cache:        dict = {}
    inicializado: bool = False

estado = EstadoApp()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    El código ANTES del yield se ejecuta al iniciar.
    El código DESPUÉS del yield se ejecuta al apagar.
    """
    # ── STARTUP ──────────────────────────────────────────────────
    print("Iniciando aplicación...")

    # Crear cliente HTTP reutilizable
    estado.http_client = httpx.AsyncClient(
        timeout=30.0,
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    )

    # Precalentar caché con datos frecuentes
    try:
        r = await estado.http_client.get("https://httpbin.org/json")
        estado.cache["datos_base"] = r.json()
        print("Caché precalentado")
    except Exception as e:
        print(f"Advertencia: no se pudo precalentar caché: {e}")

    estado.inicializado = True
    print("Aplicación lista para recibir requests")

    yield   # la aplicación corre aquí

    # ── SHUTDOWN ──────────────────────────────────────────────────
    print("Apagando aplicación...")
    if estado.http_client:
        await estado.http_client.aclose()
    estado.cache.clear()
    print("Recursos liberados")

app = FastAPI(lifespan=lifespan)

@app.get("/salud")
async def verificar_salud() -> dict:
    return {
        "status":       "ok",
        "inicializado": estado.inicializado,
        "cache_keys":   list(estado.cache.keys()),
    }

@app.get("/datos-externos")
async def obtener_datos() -> dict:
    if not estado.http_client:
        from fastapi import HTTPException
        raise HTTPException(503, "Cliente HTTP no disponible")
    # Reutilizar el cliente compartido — no crear uno nuevo por request
    r = await estado.http_client.get("https://httpbin.org/uuid")
    return r.json()
```

---

## OpenAPI — tags y documentación

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(
    title="API de Gestión",
    description="""
## API de Gestión de Recursos

Permite crear, consultar, actualizar y eliminar recursos del sistema.

### Autenticación
Todos los endpoints protegidos requieren `Authorization: Bearer <token>`.

### Rate limits
- Tier free: 100 req/min
- Tier pro:  1000 req/min
    """,
    version="2.1.0",
    contact={
        "name":  "Equipo de Ingeniería",
        "email": "api@empresa.com",
        "url":   "https://empresa.com/api",
    },
    license_info={
        "name": "MIT",
        "url":  "https://opensource.org/licenses/MIT",
    },
    openapi_tags=[
        {"name": "Autenticación",
         "description": "Registro, login y manejo de tokens"},
        {"name": "Usuarios",
         "description": "CRUD de usuarios del sistema"},
        {"name": "Productos",
         "description": "Gestión del catálogo de productos"},
        {"name": "Salud",
         "description": "Endpoints de monitoreo y health check"},
    ],
)

@app.get("/salud", tags=["Salud"], include_in_schema=True)
def health_check() -> dict:
    return {"status": "ok"}
```

---

## Ejemplo aplicado — API con autenticación JWT completa

```python
"""
auth_completa.py — API con registro, login, refresh y endpoints protegidos
Ejecutar: uvicorn auth_completa:app --reload
"""

from __future__ import annotations

import uuid
from datetime import datetime, timedelta, timezone
from contextlib import asynccontextmanager
from typing import Annotated, Any

import jwt
from fastapi import (FastAPI, Depends, HTTPException, status,
                     APIRouter, BackgroundTasks)
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from passlib.context import CryptContext
from pydantic import BaseModel, EmailStr, Field

# ── Configuración ─────────────────────────────────────────────────

JWT_SECRET     = "supersecreto-cambiar-en-produccion-usar-openssl-rand"
JWT_ALGORITHM  = "HS256"
ACCESS_EXPIRE  = timedelta(minutes=30)
REFRESH_EXPIRE = timedelta(days=7)

pwd_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2  = OAuth2PasswordBearer(tokenUrl="/auth/login")

# ── Modelos ───────────────────────────────────────────────────────

class UsuarioRegistro(BaseModel):
    nombre:   str   = Field(min_length=2, max_length=100)
    email:    str   = Field(min_length=5)         # EmailStr con email-validator
    password: str   = Field(min_length=8)

class UsuarioDB(BaseModel):
    id:            int
    nombre:        str
    email:         str
    hash_password: str
    activo:        bool = True
    rol:           str  = "user"

class UsuarioPublico(BaseModel):
    id:     int
    nombre: str
    email:  str
    rol:    str

class TokenResponse(BaseModel):
    access_token:  str
    refresh_token: str
    token_type:    str = "bearer"

class RefreshRequest(BaseModel):
    refresh_token: str

# ── Base de datos en memoria ──────────────────────────────────────

_usuarios: dict[int, UsuarioDB] = {}
_siguiente_id: int = 1

# Seed: usuario admin
_usuarios[1] = UsuarioDB(
    id=1, nombre="Admin", email="admin@ejemplo.com",
    hash_password=pwd_ctx.hash("admin1234"),
    rol="admin",
)
_siguiente_id = 2

# ── Utilidades JWT ────────────────────────────────────────────────

def crear_token(datos: dict[str, Any], expira: timedelta) -> str:
    payload = datos | {
        "exp": datetime.now(timezone.utc) + expira,
        "iat": datetime.now(timezone.utc),
        "jti": str(uuid.uuid4()),     # JWT ID único — para blacklist
    }
    return jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGORITHM)

def decodificar_token(token: str) -> dict[str, Any]:
    try:
        return jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expirado")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Token inválido")

# ── Dependencias ──────────────────────────────────────────────────

async def get_usuario_actual(
    token: Annotated[str, Depends(oauth2)],
) -> UsuarioDB:
    payload = decodificar_token(token)
    if payload.get("type") != "access":
        raise HTTPException(401, "Se requiere access token")
    user_id = int(payload.get("sub", 0))
    usuario = _usuarios.get(user_id)
    if not usuario or not usuario.activo:
        raise HTTPException(401, "Usuario no encontrado o inactivo")
    return usuario

async def requiere_admin(
    usuario: Annotated[UsuarioDB, Depends(get_usuario_actual)],
) -> UsuarioDB:
    if usuario.rol != "admin":
        raise HTTPException(403, "Se requieren permisos de administrador")
    return usuario

UsuarioActualDep = Annotated[UsuarioDB, Depends(get_usuario_actual)]
AdminDep         = Annotated[UsuarioDB, Depends(requiere_admin)]

# ── Router de autenticación ───────────────────────────────────────

router_auth = APIRouter(prefix="/auth", tags=["Autenticación"])

@router_auth.post("/registro", response_model=UsuarioPublico,
                  status_code=201)
def registrar(
    datos: UsuarioRegistro,
    background: BackgroundTasks,
) -> UsuarioDB:
    global _siguiente_id

    # Verificar email único
    if any(u.email == datos.email for u in _usuarios.values()):
        raise HTTPException(400, "El email ya está registrado")

    # Crear usuario con hash de contraseña
    nuevo = UsuarioDB(
        id=_siguiente_id,
        nombre=datos.nombre,
        email=datos.email,
        hash_password=pwd_ctx.hash(datos.password),
    )
    _usuarios[_siguiente_id] = nuevo
    _siguiente_id += 1

    # Enviar email de bienvenida en background
    background.add_task(
        lambda: print(f"[BG] Email de bienvenida enviado a {datos.email}")
    )

    return nuevo

@router_auth.post("/login", response_model=TokenResponse)
def login(
    form: Annotated[OAuth2PasswordRequestForm, Depends()],
) -> dict:
    # OAuth2PasswordRequestForm espera form-data con username y password
    usuario = next(
        (u for u in _usuarios.values() if u.email == form.username),
        None,
    )
    if not usuario or not pwd_ctx.verify(form.password, usuario.hash_password):
        raise HTTPException(
            status.HTTP_401_UNAUTHORIZED,
            "Credenciales incorrectas",
            headers={"WWW-Authenticate": "Bearer"},
        )

    access_token  = crear_token(
        {"sub": str(usuario.id), "email": usuario.email, "type": "access"},
        ACCESS_EXPIRE,
    )
    refresh_token = crear_token(
        {"sub": str(usuario.id), "type": "refresh"},
        REFRESH_EXPIRE,
    )
    return {
        "access_token":  access_token,
        "refresh_token": refresh_token,
    }

@router_auth.post("/refresh", response_model=TokenResponse)
def refrescar_token(body: RefreshRequest) -> dict:
    """Genera un nuevo access_token a partir de un refresh_token válido."""
    payload = decodificar_token(body.refresh_token)

    if payload.get("type") != "refresh":
        raise HTTPException(401, "Se requiere refresh token")

    user_id = int(payload.get("sub", 0))
    usuario = _usuarios.get(user_id)
    if not usuario or not usuario.activo:
        raise HTTPException(401, "Usuario no válido")

    nuevo_access = crear_token(
        {"sub": str(usuario.id), "email": usuario.email, "type": "access"},
        ACCESS_EXPIRE,
    )
    nuevo_refresh = crear_token(
        {"sub": str(usuario.id), "type": "refresh"},
        REFRESH_EXPIRE,
    )
    return {"access_token": nuevo_access, "refresh_token": nuevo_refresh}

@router_auth.get("/me", response_model=UsuarioPublico)
def perfil_actual(usuario: UsuarioActualDep) -> UsuarioDB:
    """Retorna los datos del usuario autenticado."""
    return usuario

# ── Router de usuarios (admin) ────────────────────────────────────

router_usuarios = APIRouter(prefix="/usuarios", tags=["Usuarios"])

@router_usuarios.get("/", response_model=list[UsuarioPublico])
def listar_usuarios(admin: AdminDep) -> list[UsuarioDB]:
    """Solo administradores pueden listar todos los usuarios."""
    return list(_usuarios.values())

@router_usuarios.get("/{user_id}", response_model=UsuarioPublico)
def obtener_usuario(
    user_id: int,
    usuario_actual: UsuarioActualDep,
) -> UsuarioDB:
    """Un usuario puede ver su propio perfil; admin puede ver cualquiera."""
    if usuario_actual.id != user_id and usuario_actual.rol != "admin":
        raise HTTPException(403, "Sin permisos para ver este perfil")
    usuario = _usuarios.get(user_id)
    if not usuario:
        raise HTTPException(404, f"Usuario {user_id} no encontrado")
    return usuario

# ── Lifespan y App principal ──────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    print("API de autenticación iniciada")
    print(f"Usuarios precargados: {len(_usuarios)}")
    yield
    print("API apagada")

app = FastAPI(
    title="Auth API",
    description="API con autenticación JWT completa",
    version="1.0.0",
    lifespan=lifespan,
)

app.include_router(router_auth)
app.include_router(router_usuarios)

@app.get("/", tags=["Salud"])
def raiz() -> dict:
    return {"api": "Auth API v1.0", "docs": "/docs"}

# ── Endpoint protegido de ejemplo ─────────────────────────────────

@app.get("/privado", tags=["Protegido"])
def dato_privado(usuario: UsuarioActualDep) -> dict:
    return {
        "mensaje": f"Hola {usuario.nombre}, eres {usuario.rol}",
        "user_id": usuario.id,
    }

@app.get("/solo-admin", tags=["Protegido"])
def solo_admins(admin: AdminDep) -> dict:
    return {
        "mensaje": "Zona de administración",
        "total_usuarios": len(_usuarios),
    }
```

---

## Ejercicios propuestos

1. **Blacklist de tokens** — Implementa un sistema de logout real: cuando el
   usuario hace logout, guarda el `jti` (JWT ID) del token en un `set` de
   tokens revocados. En la dependencia `get_usuario_actual`, verifica que
   el `jti` no esté en la blacklist antes de autenticar. Usa un `set` en
   memoria con TTL: limpia los `jti` expirados cada 5 minutos con una tarea
   de background.

2. **Rate limiting por usuario** — Implementa un middleware que limite las
   peticiones a 60 req/min por IP o por `user_id` (si está autenticado).
   Usa un `dict[str, deque]` para almacenar los timestamps de los últimos
   requests de cada cliente. Devuelve 429 con el header `Retry-After` cuando
   se supere el límite.

3. **Permisos granulares con scopes** — Extiende el sistema de autenticación
   para incluir scopes en el JWT (`read:productos`, `write:productos`,
   `admin:usuarios`). Crea una función `requiere_scope(scope: str)` que
   retorne una dependencia que verifique si el token tiene ese scope.
   Aplica los scopes a los endpoints de lectura y escritura del catálogo.

4. **Tests con httpx.AsyncClient** — Escribe tests de integración usando
   `pytest` y `httpx.AsyncClient` (con `transport=ASGITransport(app=app)`
   para testear sin levantar servidor). Cubre: registro exitoso, registro
   con email duplicado, login correcto, login incorrecto, acceso con token
   válido, acceso con token expirado y acceso de viewer a endpoint de admin.

---

## Resumen de la página 13

- `CORSMiddleware` se añade con `app.add_middleware()` — debe ser el primero en procesarse para manejar preflight OPTIONS.
- `@app.middleware("http")` con `call_next` permite interceptar cada request — ideal para logging, timing y enriquecimiento de headers.
- Las dependencias generadoras con `yield` gestionan recursos automáticamente — el bloque `finally` se ejecuta siempre, incluso en errores.
- `PyJWT` con `HS256` genera tokens compactos — incluir siempre `exp`, `iat` y `jti` para seguridad; usar `RS256` en sistemas distribuidos.
- `passlib.CryptContext` con bcrypt es el estándar para hashear contraseñas — nunca almacenar contraseñas en texto plano ni usar MD5/SHA1.
- `BackgroundTasks.add_task()` ejecuta funciones después de enviar la respuesta HTTP — ideal para emails, logs de auditoría y limpieza.
- El patrón `lifespan` con `asynccontextmanager` reemplaza `@app.on_event` — garantiza que los recursos se inicialicen y liberen correctamente.
- Los `exception_handler` globales dan formato consistente a todos los errores — separar errores de validación, HTTP y de negocio en respuestas claras.

---

> **Siguiente página →** Página 17: Bases de datos — SQLAlchemy 2.0, Alembic y FastAPI.
> SQLAlchemy 2.0, modelos con `Mapped[]`, relaciones, migraciones con
> Alembic e integración completa con FastAPI.
