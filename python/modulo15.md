# Tutorial Python — Página 15
## Módulo 7 · HTTP y APIs REST
### Consumir APIs con httpx, construir APIs con FastAPI — fundamentos

---

## httpx — cliente HTTP moderno

```
httpx vs requests
───────────────────────────────────────────────────────────────────
httpx                              requests
──────────────────────             ──────────────────────
Sync y async en una librería       Solo sync
HTTP/2 soportado                   Solo HTTP/1.1
Timeouts granulares                Timeout global
Client reutilizable (session)      Session disponible
Type hints completos               Parciales
Mantenida activamente              Legacy (mantenimiento)
```

### Instalación

```bash
# Instalar httpx con soporte HTTP/2
pip install httpx[http2]

# Para async con asyncio (ya incluido en httpx)
# Para FastAPI con uvicorn
pip install "fastapi[standard]"
```

---

## httpx sync — GET, POST, PUT, DELETE

```python
import httpx

BASE_URL = "https://jsonplaceholder.typicode.com"

# ── GET básico ────────────────────────────────────────────────────

respuesta = httpx.get(f"{BASE_URL}/posts/1")

# raise_for_status() lanza HTTPStatusError si status >= 400
respuesta.raise_for_status()

# Acceder al JSON directamente como dict
post = respuesta.json()
print(post["title"])
print(respuesta.status_code)      # 200
print(respuesta.headers["content-type"])

# ── GET con query parameters ──────────────────────────────────────

# params se serializa automáticamente como ?userId=1&_limit=5
respuesta = httpx.get(
    f"{BASE_URL}/posts",
    params={"userId": 1, "_limit": 5},
    timeout=10.0,
)
posts = respuesta.json()
print(f"Posts recibidos: {len(posts)}")

# ── POST con JSON body ────────────────────────────────────────────

nuevo_post = {
    "title":  "Aprendiendo httpx",
    "body":   "httpx es el sucesor moderno de requests",
    "userId": 1,
}

respuesta = httpx.post(
    f"{BASE_URL}/posts",
    json=nuevo_post,               # serializa dict a JSON + Content-Type
    headers={"Authorization": "Bearer mi-token-secreto"},
    timeout=10.0,
)
respuesta.raise_for_status()
post_creado = respuesta.json()
print(f"Post creado con id: {post_creado['id']}")

# ── PUT — actualizar recurso completo ─────────────────────────────

respuesta = httpx.put(
    f"{BASE_URL}/posts/1",
    json={"title": "Título actualizado", "body": "Cuerpo nuevo", "userId": 1},
)
print(f"PUT status: {respuesta.status_code}")

# ── DELETE ────────────────────────────────────────────────────────

respuesta = httpx.delete(f"{BASE_URL}/posts/1")
print(f"DELETE status: {respuesta.status_code}")   # 200 o 204
```

---

## httpx.Client — sesión reutilizable

```python
import httpx

# Client mantiene conexiones abiertas (keep-alive) y comparte config
# SIEMPRE usar como context manager para cerrar conexiones

with httpx.Client(
    base_url="https://jsonplaceholder.typicode.com",
    headers={
        "Authorization": "Bearer mi-token",
        "Accept":        "application/json",
    },
    timeout=httpx.Timeout(
        connect=5.0,    # tiempo para establecer conexión
        read=30.0,      # tiempo para recibir la respuesta
        write=10.0,     # tiempo para enviar el request
        pool=5.0,       # tiempo esperando una conexión del pool
    ),
    follow_redirects=True,
    http2=True,         # habilitar HTTP/2
) as client:
    # Todas las llamadas comparten base_url, headers y timeout
    r1 = client.get("/posts/1")
    r2 = client.get("/posts/2")
    r3 = client.get("/users/1")

    r1.raise_for_status()
    r2.raise_for_status()
    r3.raise_for_status()

    print(r1.json()["title"])
    print(r3.json()["email"])
```

---

## httpx async — cliente asíncrono

```python
import asyncio
import httpx

async def obtener_posts_usuario(user_id: int) -> list[dict]:
    """Descarga todos los posts de un usuario usando el cliente async."""
    async with httpx.AsyncClient(
        base_url="https://jsonplaceholder.typicode.com",
        timeout=10.0,
    ) as client:
        respuesta = await client.get("/posts", params={"userId": user_id})
        respuesta.raise_for_status()
        return respuesta.json()

async def main() -> None:
    # Descargar posts de múltiples usuarios concurrentemente
    tareas = [obtener_posts_usuario(i) for i in range(1, 6)]
    resultados = await asyncio.gather(*tareas)

    for i, posts in enumerate(resultados, 1):
        print(f"Usuario {i}: {len(posts)} posts")

asyncio.run(main())
```

---

## Manejo de errores HTTP

```python
import httpx

def llamar_api_segura(url: str) -> dict | None:
    """Maneja todos los tipos de error posibles en una llamada HTTP."""
    try:
        with httpx.Client(timeout=10.0) as client:
            respuesta = client.get(url)
            respuesta.raise_for_status()     # lanza HTTPStatusError si >= 400
            return respuesta.json()

    except httpx.HTTPStatusError as e:
        # Error de la API: 4xx o 5xx
        codigo   = e.response.status_code
        contenido = e.response.text[:200]
        print(f"Error HTTP {codigo}: {contenido}")

        if codigo == 401:
            print("Token inválido o expirado")
        elif codigo == 403:
            print("Sin permisos para este recurso")
        elif codigo == 404:
            print("Recurso no encontrado")
        elif codigo == 429:
            retry_after = e.response.headers.get("Retry-After", "?")
            print(f"Rate limit excedido. Esperar {retry_after}s")
        elif 500 <= codigo < 600:
            print("Error en el servidor — reintentar más tarde")

    except httpx.TimeoutException:
        # La petición tardó más del timeout configurado
        print(f"Timeout al llamar {url}")

    except httpx.ConnectError:
        # No se pudo establecer conexión (DNS, red, etc.)
        print(f"No se puede conectar a {url}")

    except httpx.RequestError as e:
        # Cualquier otro error de red
        print(f"Error de red: {type(e).__name__}: {e}")

    return None
```

---

## Parsear respuestas con Pydantic

```python
from pydantic import BaseModel, Field, HttpUrl, validator
import httpx

# Pydantic valida y convierte automáticamente el JSON a tipos Python

class Direccion(BaseModel):
    ciudad: str = Field(alias="city")
    pais:   str = Field(alias="zipcode")   # mapear campos con alias

class Usuario(BaseModel):
    id:      int
    nombre:  str    = Field(alias="name")
    email:   str
    telefono: str   = Field(alias="phone")

    class Config:
        populate_by_name = True   # permite usar alias o nombre original

class Post(BaseModel):
    id:       int
    user_id:  int    = Field(alias="userId")
    titulo:   str    = Field(alias="title")
    cuerpo:   str    = Field(alias="body")

    @property
    def resumen(self) -> str:
        """Primeras 50 letras del cuerpo."""
        return self.cuerpo[:50] + "..." if len(self.cuerpo) > 50 else self.cuerpo

def obtener_usuario(user_id: int) -> Usuario | None:
    with httpx.Client(base_url="https://jsonplaceholder.typicode.com") as c:
        r = c.get(f"/users/{user_id}")
        r.raise_for_status()
        return Usuario.model_validate(r.json())   # Pydantic v2

def obtener_posts(user_id: int) -> list[Post]:
    with httpx.Client(base_url="https://jsonplaceholder.typicode.com") as c:
        r = c.get("/posts", params={"userId": user_id})
        r.raise_for_status()
        # model_validate acepta lista de dicts
        return [Post.model_validate(p) for p in r.json()]

usuario = obtener_usuario(1)
if usuario:
    print(f"Usuario: {usuario.nombre} ({usuario.email})")
    posts = obtener_posts(usuario.id)
    for post in posts[:3]:
        print(f"  - {post.titulo[:40]}")
        print(f"    {post.resumen}")
```

---

## FastAPI — primera aplicación

```python
# main.py
# Instalar: pip install "fastapi[standard]"
# Ejecutar:  fastapi dev main.py   (con auto-reload)
#       o:   uvicorn main:app --reload

from fastapi import FastAPI
from pydantic import BaseModel

# FastAPI genera automáticamente:
#   /docs  → Swagger UI interactivo
#   /redoc → ReDoc (documentación alternativa)
#   /openapi.json → esquema OpenAPI 3.1

app = FastAPI(
    title="Mi Primera API",
    description="API de ejemplo con FastAPI",
    version="1.0.0",
)

class Item(BaseModel):
    nombre: str
    precio: float
    en_stock: bool = True

# Almacenamiento en memoria (para el ejemplo)
items_db: dict[int, Item] = {
    1: Item(nombre="Laptop",  precio=1200.0),
    2: Item(nombre="Monitor", precio=350.0),
}
next_id = 3

@app.get("/")
def raiz() -> dict[str, str]:
    return {"mensaje": "API funcionando", "docs": "/docs"}

@app.get("/items/{item_id}")
def obtener_item(item_id: int) -> Item:
    if item_id not in items_db:
        # FastAPI convierte automáticamente HTTPException en respuesta JSON
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail=f"Item {item_id} no encontrado")
    return items_db[item_id]

@app.post("/items", status_code=201)
def crear_item(item: Item) -> dict[str, object]:
    global next_id
    items_db[next_id] = item
    id_creado = next_id
    next_id += 1
    return {"id": id_creado, "item": item}
```

---

## Path parameters, query parameters y request body

```python
from fastapi import FastAPI, Query, Path, Body, HTTPException
from pydantic import BaseModel, Field
from typing import Annotated

app = FastAPI()

class Producto(BaseModel):
    nombre:      str   = Field(min_length=2, max_length=100)
    precio:      float = Field(gt=0, description="Precio en USD")
    categoria:   str
    descripcion: str | None = None

productos_db: dict[int, Producto] = {}

# ── Path parameters con validación ───────────────────────────────

@app.get("/productos/{producto_id}")
def obtener_producto(
    # Annotated + Path() para validar y documentar el parámetro
    producto_id: Annotated[int, Path(ge=1, description="ID del producto")],
) -> Producto:
    if producto_id not in productos_db:
        raise HTTPException(404, f"Producto {producto_id} no existe")
    return productos_db[producto_id]

# ── Query parameters con validación ──────────────────────────────

@app.get("/productos")
def listar_productos(
    # Query() valida y documenta los query params
    pagina:    Annotated[int, Query(ge=1, description="Número de página")] = 1,
    por_pagina: Annotated[int, Query(ge=1, le=100)] = 20,
    categoria: str | None = Query(None, description="Filtrar por categoría"),
    orden:     Annotated[str, Query(pattern="^(nombre|precio)$")] = "nombre",
) -> dict[str, object]:
    items = list(productos_db.values())

    # Filtrar por categoría si se especificó
    if categoria:
        items = [p for p in items if p.categoria == categoria]

    # Ordenar
    items.sort(key=lambda p: getattr(p, orden))

    # Paginar
    inicio = (pagina - 1) * por_pagina
    fin    = inicio + por_pagina
    pagina_items = items[inicio:fin]

    return {
        "total":     len(items),
        "pagina":    pagina,
        "por_pagina": por_pagina,
        "items":     pagina_items,
    }

# ── Request body con Pydantic ─────────────────────────────────────

@app.post("/productos", status_code=201)
def crear_producto(producto: Producto) -> dict[str, object]:
    nuevo_id = max(productos_db.keys(), default=0) + 1
    productos_db[nuevo_id] = producto
    return {"id": nuevo_id, **producto.model_dump()}

@app.put("/productos/{producto_id}")
def actualizar_producto(
    producto_id: Annotated[int, Path(ge=1)],
    producto:    Producto,
) -> Producto:
    if producto_id not in productos_db:
        raise HTTPException(404, f"Producto {producto_id} no existe")
    productos_db[producto_id] = producto
    return producto

@app.delete("/productos/{producto_id}", status_code=204)
def eliminar_producto(
    producto_id: Annotated[int, Path(ge=1)],
) -> None:
    if producto_id not in productos_db:
        raise HTTPException(404, f"Producto {producto_id} no existe")
    del productos_db[producto_id]
```

---

## Response models y status codes

```python
from fastapi import FastAPI, status
from pydantic import BaseModel, Field
from typing import Annotated

app = FastAPI()

# ── Response models — lo que devuelve la API ──────────────────────

class ProductoCrear(BaseModel):
    """Datos que el cliente envía al crear un producto."""
    nombre:    str   = Field(min_length=2)
    precio:    float = Field(gt=0)
    categoria: str

class ProductoPublico(BaseModel):
    """Datos que la API devuelve — puede excluir campos internos."""
    id:        int
    nombre:    str
    precio:    float
    categoria: str
    # No incluir campos como: costo_interno, proveedor_id, etc.

class ListaProductos(BaseModel):
    total:  int
    pagina: int
    items:  list[ProductoPublico]

db: dict[int, dict] = {}

# response_model filtra automáticamente los campos del modelo de respuesta
# status_code define el código HTTP de respuesta exitosa
@app.post(
    "/productos",
    response_model=ProductoPublico,
    status_code=status.HTTP_201_CREATED,
    summary="Crear un producto nuevo",
    response_description="El producto creado",
)
def crear_producto(datos: ProductoCrear) -> dict:
    nuevo_id = len(db) + 1
    producto = {"id": nuevo_id, **datos.model_dump()}
    db[nuevo_id] = producto
    return producto    # FastAPI filtra con response_model automáticamente

@app.get(
    "/productos",
    response_model=ListaProductos,
    summary="Listar productos con paginación",
)
def listar(pagina: int = 1, por_pagina: int = 20) -> dict:
    items  = list(db.values())
    inicio = (pagina - 1) * por_pagina
    return {
        "total":  len(items),
        "pagina": pagina,
        "items":  items[inicio:inicio + por_pagina],
    }
```

---

## Routers — organizar rutas en módulos

```python
# ── routers/productos.py ──────────────────────────────────────────

from fastapi import APIRouter, HTTPException, Path
from pydantic import BaseModel, Field
from typing import Annotated

router = APIRouter(
    prefix="/productos",          # prefijo para todas las rutas
    tags=["Productos"],           # agrupación en Swagger UI
    responses={
        404: {"description": "Producto no encontrado"},
        422: {"description": "Datos inválidos"},
    },
)

class Producto(BaseModel):
    nombre: str = Field(min_length=2)
    precio: float = Field(gt=0)

_db: dict[int, Producto] = {}

@router.get("/")
def listar() -> list[Producto]:
    return list(_db.values())

@router.get("/{id}")
def obtener(id: Annotated[int, Path(ge=1)]) -> Producto:
    if id not in _db:
        raise HTTPException(404, f"Producto {id} no existe")
    return _db[id]

@router.post("/", status_code=201)
def crear(producto: Producto) -> dict:
    nuevo_id = len(_db) + 1
    _db[nuevo_id] = producto
    return {"id": nuevo_id, **producto.model_dump()}

# ── routers/usuarios.py ──────────────────────────────────────────

from fastapi import APIRouter

router_usuarios = APIRouter(prefix="/usuarios", tags=["Usuarios"])

@router_usuarios.get("/")
def listar_usuarios() -> list[dict]:
    return [{"id": 1, "nombre": "Admin"}]

# ── main.py — registrar routers ───────────────────────────────────

from fastapi import FastAPI
# from routers.productos import router         # importar el router
# from routers.usuarios  import router_usuarios

app = FastAPI(title="API Catálogo")

# app.include_router(router)
# app.include_router(router_usuarios)
```

---

## Dependency Injection básica

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Annotated

app = FastAPI()

# ── Dependencias — funciones que FastAPI llama automáticamente ─────

def verificar_api_key(
    x_api_key: Annotated[str | None, Header()] = None,
) -> str:
    """Dependencia: verifica que el header X-Api-Key sea válido."""
    claves_validas = {"clave-secreta-123", "otra-clave-456"}
    if x_api_key not in claves_validas:
        raise HTTPException(401, "API key inválida o ausente")
    return x_api_key

def paginacion(
    pagina:     int = 1,
    por_pagina: int = 20,
) -> dict[str, int]:
    """Dependencia reutilizable: extrae y valida parámetros de paginación."""
    if pagina < 1:
        raise HTTPException(400, "La página debe ser >= 1")
    if por_pagina < 1 or por_pagina > 100:
        raise HTTPException(400, "por_pagina debe estar entre 1 y 100")
    return {"pagina": pagina, "por_pagina": por_pagina, "offset": (pagina - 1) * por_pagina}

# Annotated + Depends — la forma idiomática de declarar dependencias
ApiKeyDep  = Annotated[str, Depends(verificar_api_key)]
PagDep     = Annotated[dict, Depends(paginacion)]

@app.get("/datos-privados")
def datos_privados(api_key: ApiKeyDep) -> dict:
    # api_key ya fue verificada por la dependencia
    return {"mensaje": "Datos confidenciales", "clave_usada": api_key}

@app.get("/items")
def listar_items(pag: PagDep) -> dict:
    # pag contiene {"pagina": N, "por_pagina": N, "offset": N}
    items_totales = list(range(100))   # simula 100 items
    items_pagina  = items_totales[pag["offset"]:pag["offset"] + pag["por_pagina"]]
    return {"pagina": pag["pagina"], "items": items_pagina}
```

---

## Ejemplo aplicado — Catálogo de productos (cliente + servidor)

```python
"""
catalogo_api.py — API completa de catálogo de productos
Ejecutar: uvicorn catalogo_api:app --reload
Cliente:  python catalogo_cliente.py
"""

# ══════════════════════════════════════════════════════════════════
#  SERVIDOR — catalogo_api.py
# ══════════════════════════════════════════════════════════════════

from fastapi import FastAPI, HTTPException, Query, Path, Depends
from pydantic import BaseModel, Field
from typing import Annotated
import uuid

app = FastAPI(
    title="Catálogo de Productos",
    description="API REST para gestión de inventario",
    version="1.0.0",
)

# ── Modelos ───────────────────────────────────────────────────────

class ProductoBase(BaseModel):
    nombre:      str   = Field(min_length=2, max_length=200,
                                description="Nombre del producto")
    descripcion: str   = Field(default="", max_length=1000)
    precio:      float = Field(gt=0, description="Precio en USD")
    categoria:   str   = Field(min_length=2, max_length=50)
    stock:       int   = Field(ge=0, default=0)

class ProductoCrear(ProductoBase):
    pass

class Producto(ProductoBase):
    id:  str   # UUID como string

class BusquedaParams(BaseModel):
    """Parámetros de búsqueda como dependencia."""
    q:         str | None = None
    categoria: str | None = None
    precio_min: float | None = Query(None, ge=0)
    precio_max: float | None = Query(None, ge=0)

# ── Base de datos en memoria ──────────────────────────────────────

_productos: dict[str, Producto] = {
    "p1": Producto(id="p1", nombre="Laptop Pro 16",
                   precio=1299.99, categoria="electronica", stock=15,
                   descripcion="Laptop de alto rendimiento con 32GB RAM"),
    "p2": Producto(id="p2", nombre="Monitor UltraWide 34",
                   precio=599.99, categoria="electronica", stock=8),
    "p3": Producto(id="p3", nombre="Teclado Mecánico TKL",
                   precio=129.99, categoria="perifericos", stock=50),
    "p4": Producto(id="p4", nombre="Mouse Ergonómico",
                   precio=79.99,  categoria="perifericos", stock=30),
}

# ── Dependencia de búsqueda ───────────────────────────────────────

def params_busqueda(
    q:          str | None = Query(None, description="Búsqueda de texto"),
    categoria:  str | None = Query(None),
    precio_min: float | None = Query(None, ge=0),
    precio_max: float | None = Query(None, ge=0),
) -> dict:
    return {"q": q, "categoria": categoria,
            "precio_min": precio_min, "precio_max": precio_max}

BusquedaDep = Annotated[dict, Depends(params_busqueda)]

# ── Endpoints ─────────────────────────────────────────────────────

@app.get("/productos", response_model=dict, tags=["Productos"])
def listar_productos(
    pagina:     Annotated[int, Query(ge=1)] = 1,
    por_pagina: Annotated[int, Query(ge=1, le=100)] = 20,
    busqueda:   BusquedaDep = Depends(params_busqueda),
) -> dict:
    """Lista productos con paginación y filtros opcionales."""
    items = list(_productos.values())

    # Aplicar filtros
    if busqueda["q"]:
        q = busqueda["q"].lower()
        items = [p for p in items
                 if q in p.nombre.lower() or q in p.descripcion.lower()]

    if busqueda["categoria"]:
        items = [p for p in items
                 if p.categoria == busqueda["categoria"]]

    if busqueda["precio_min"] is not None:
        items = [p for p in items if p.precio >= busqueda["precio_min"]]

    if busqueda["precio_max"] is not None:
        items = [p for p in items if p.precio <= busqueda["precio_max"]]

    # Paginar
    total  = len(items)
    offset = (pagina - 1) * por_pagina
    pagina_items = items[offset:offset + por_pagina]

    return {
        "total":        total,
        "pagina":       pagina,
        "por_pagina":   por_pagina,
        "total_paginas": -(-total // por_pagina),  # ceil division
        "items":        pagina_items,
    }

@app.get("/productos/{producto_id}", response_model=Producto, tags=["Productos"])
def obtener_producto(
    producto_id: Annotated[str, Path(description="ID del producto")],
) -> Producto:
    """Obtiene un producto por su ID."""
    if producto_id not in _productos:
        raise HTTPException(404, f"Producto '{producto_id}' no encontrado")
    return _productos[producto_id]

@app.post("/productos", response_model=Producto,
          status_code=201, tags=["Productos"])
def crear_producto(datos: ProductoCrear) -> Producto:
    """Crea un nuevo producto."""
    nuevo_id = str(uuid.uuid4())[:8]    # ID corto para el ejemplo
    producto = Producto(id=nuevo_id, **datos.model_dump())
    _productos[nuevo_id] = producto
    return producto

@app.put("/productos/{producto_id}", response_model=Producto, tags=["Productos"])
def actualizar_producto(
    producto_id: str,
    datos: ProductoCrear,
) -> Producto:
    """Reemplaza un producto completo."""
    if producto_id not in _productos:
        raise HTTPException(404, f"Producto '{producto_id}' no encontrado")
    producto = Producto(id=producto_id, **datos.model_dump())
    _productos[producto_id] = producto
    return producto

@app.delete("/productos/{producto_id}", status_code=204, tags=["Productos"])
def eliminar_producto(producto_id: str) -> None:
    """Elimina un producto."""
    if producto_id not in _productos:
        raise HTTPException(404, f"Producto '{producto_id}' no encontrado")
    del _productos[producto_id]

@app.get("/categorias", tags=["Utilidades"])
def listar_categorias() -> list[str]:
    """Lista todas las categorías disponibles."""
    return sorted({p.categoria for p in _productos.values()})

# ══════════════════════════════════════════════════════════════════
#  CLIENTE — catalogo_cliente.py
#  (en la práctica: archivo separado)
# ══════════════════════════════════════════════════════════════════

import httpx

class ClienteCatalogo:
    """Cliente Python para la API del catálogo."""

    def __init__(self, base_url: str = "http://localhost:8000") -> None:
        self._client = httpx.Client(
            base_url=base_url,
            timeout=10.0,
            headers={"Content-Type": "application/json"},
        )

    def __enter__(self): return self
    def __exit__(self, *args): self._client.close()

    def listar(self, pagina: int = 1, q: str | None = None,
               categoria: str | None = None) -> dict:
        params: dict = {"pagina": pagina}
        if q:         params["q"]         = q
        if categoria: params["categoria"] = categoria
        r = self._client.get("/productos", params=params)
        r.raise_for_status()
        return r.json()

    def obtener(self, producto_id: str) -> dict:
        r = self._client.get(f"/productos/{producto_id}")
        r.raise_for_status()
        return r.json()

    def crear(self, nombre: str, precio: float, categoria: str,
              descripcion: str = "", stock: int = 0) -> dict:
        r = self._client.post("/productos", json={
            "nombre": nombre, "precio": precio,
            "categoria": categoria, "descripcion": descripcion,
            "stock": stock,
        })
        r.raise_for_status()
        return r.json()

    def eliminar(self, producto_id: str) -> bool:
        r = self._client.delete(f"/productos/{producto_id}")
        return r.status_code == 204

# Demostración del cliente
def demo_cliente() -> None:
    with ClienteCatalogo() as catalogo:
        # Listar todos los productos
        resultado = catalogo.listar()
        print(f"Total: {resultado['total']} productos")
        for item in resultado["items"]:
            print(f"  [{item['id']}] {item['nombre']} — ${item['precio']:.2f}")

        # Buscar electrónicos
        electronica = catalogo.listar(categoria="electronica")
        print(f"\nElectrónica: {electronica['total']} productos")

        # Crear nuevo producto
        nuevo = catalogo.crear(
            nombre="Auriculares Inalámbricos",
            precio=199.99,
            categoria="perifericos",
            stock=25,
        )
        print(f"\nCreado: {nuevo['id']} — {nuevo['nombre']}")

        # Obtener por ID
        producto = catalogo.obtener(nuevo["id"])
        print(f"Obtenido: {producto['nombre']}")

# demo_cliente()   # descomentar para ejecutar
```

---

## Ejercicios propuestos

1. **Cliente async con reintentos** — Reescribe el `ClienteCatalogo` como una
   clase async que use `httpx.AsyncClient`. Añade un método `listar_todas_paginas()`
   que haga paginación automática hasta obtener todos los productos, usando
   `asyncio.gather()` para descargar múltiples páginas en paralelo una vez
   conocido el total.

2. **PATCH — actualización parcial** — Añade el endpoint `PATCH /productos/{id}`
   que permita actualizar solo los campos enviados (no todos). Usa un modelo
   Pydantic con todos los campos opcionales (`Optional`) y `model_dump(exclude_unset=True)`
   para aplicar solo los campos presentes en el request.

3. **Middleware de autenticación como dependencia** — Crea una dependencia
   `get_usuario_actual(token: str = Header(...))` que valide un JWT simple
   (puedes usar `PyJWT`). Aplícala a todos los endpoints de escritura
   (POST, PUT, DELETE) usando `dependencies=[Depends(get_usuario_actual)]`
   en el `APIRouter`.

4. **Caché con TTL** — Implementa una dependencia `CacheHTTP` que use un
   diccionario en memoria con TTL de 60 segundos para cachear las respuestas
   GET del catálogo. Si la clave existe y no ha expirado, retorna el resultado
   cacheado; si no, llama al endpoint real. Añade un header `X-Cache: HIT/MISS`.

---

## Resumen de la página 12

- `httpx` es el sucesor moderno de `requests` — soporta sync y async, HTTP/2, y timeouts granulares en una sola librería.
- `httpx.Client` y `httpx.AsyncClient` son sesiones reutilizables — mantienen conexiones keep-alive y comparten configuración entre llamadas.
- `raise_for_status()` convierte respuestas 4xx/5xx en excepciones `HTTPStatusError` — siempre llamarlo antes de procesar la respuesta.
- Pydantic `model_validate()` parsea y valida JSON de APIs externas automáticamente — elimina el boilerplate de conversión de tipos.
- `FastAPI` genera Swagger UI en `/docs` y `/redoc` automáticamente a partir de las type hints y modelos Pydantic — cero configuración adicional.
- `Path()`, `Query()`, `Body()` con `Annotated` permiten validar, documentar y agregar restricciones a los parámetros de forma declarativa.
- `response_model` filtra los campos de la respuesta — permite tener modelos internos con más datos que el modelo público expuesto.
- `APIRouter` con `prefix` y `tags` organiza las rutas en módulos separados; `app.include_router()` las registra en la aplicación principal.

---

> **Siguiente página →** Página 16: FastAPI avanzado — autenticación JWT, middleware y background tasks.
> Middleware, autenticación JWT, OAuth2, hashing de contraseñas,
> background tasks y lifespan events.
