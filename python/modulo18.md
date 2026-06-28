# Tutorial Python — Página 18
## Módulo 9 · Testing
### pytest — fixtures, mocks, parametrize y coverage

---

## ¿Por qué pytest?

Python incluye `unittest` en la librería estándar, pero `pytest` se convirtió en el
estándar de la industria porque su sintaxis es más limpia, los mensajes de error son
más informativos y su sistema de plugins es enorme. No necesitas clases: una función
que empieza con `test_` ya es un test.

```
unittest (stdlib)              pytest
───────────────────────────    ───────────────────────────────
class TestSuma(unittest.TC):   def test_suma():
    def test_suma(self):           assert suma(2, 3) == 5
        self.assertEqual(...)
```

---

## Instalación y convenciones

```bash
# Instalar pytest y plugins esenciales
pip install pytest pytest-cov pytest-asyncio httpx

# Convenciones de nombrado que pytest detecta automáticamente:
# - Archivos:    test_*.py  o  *_test.py
# - Funciones:   def test_nombre()
# - Clases:      class TestNombre   (sin heredar de nada)
# - Métodos:     def test_nombre(self)

# Ejecutar todos los tests
pytest

# Ejecutar un archivo específico
pytest tests/test_tareas.py

# Ejecutar un test específico
pytest tests/test_tareas.py::test_crear_tarea

# Modo verbose — muestra cada test
pytest -v

# Parar al primer fallo
pytest -x
```

---

## assert — mensajes de error informativos

pytest reescribe los `assert` de Python para mostrar exactamente qué falló:

```python
# tests/test_basico.py

def test_assert_basico():
    # pytest muestra el valor real vs esperado al fallar
    resultado = 2 + 2
    assert resultado == 4                    # pasa

def test_assert_lista():
    nombres = ["Ana", "Luis", "María"]
    assert "Luis" in nombres                 # pasa
    assert len(nombres) == 3                 # pasa

def test_assert_diccionario():
    tarea = {"id": 1, "titulo": "Comprar pan", "completada": False}
    assert tarea["titulo"] == "Comprar pan"  # pasa
    assert not tarea["completada"]           # pasa

# Al fallar un assert, pytest muestra:
# AssertionError: assert 4 == 5
#  +  where 4 = 2 + 2
# — mucho más útil que "AssertionError: False is not True"

def test_con_mensaje():
    x = 10
    # Mensaje personalizado cuando falla
    assert x > 0, f"Se esperaba x > 0 pero x = {x}"
```

---

## Fixtures — setup y teardown declarativo

Las fixtures son funciones que proveen datos o recursos a los tests.
Se inyectan por nombre en los parámetros de la función de test.

```python
# tests/test_fixtures.py
import pytest


# Fixture simple — devuelve datos
@pytest.fixture
def tarea_ejemplo():
    """Fixture que provee una tarea de ejemplo."""
    return {
        "id": 1,
        "titulo": "Estudiar pytest",
        "descripcion": "Aprender fixtures y mocks",
        "completada": False,
        "prioridad": "alta",
    }


# Fixture con teardown — yield separa setup de teardown
@pytest.fixture
def archivo_temporal(tmp_path):
    """Crea un archivo temporal y lo limpia al terminar."""
    # Setup: crear el archivo
    ruta = tmp_path / "datos_test.txt"
    ruta.write_text("datos de prueba")

    yield ruta  # el test recibe la ruta aquí

    # Teardown: se ejecuta después del test (aunque falle)
    if ruta.exists():
        ruta.unlink()
        print(f"\nArchivo {ruta} eliminado")


# Usar fixtures — se inyectan por nombre en los parámetros
def test_tarea_tiene_titulo(tarea_ejemplo):
    assert tarea_ejemplo["titulo"] == "Estudiar pytest"


def test_tarea_no_completada(tarea_ejemplo):
    assert not tarea_ejemplo["completada"]


def test_archivo_existe(archivo_temporal):
    assert archivo_temporal.exists()
    assert archivo_temporal.read_text() == "datos de prueba"


# Una fixture puede usar otra fixture
@pytest.fixture
def lista_tareas(tarea_ejemplo):
    """Fixture que depende de otra fixture."""
    return [
        tarea_ejemplo,
        {**tarea_ejemplo, "id": 2, "titulo": "Hacer ejercicio",
         "prioridad": "media"},
        {**tarea_ejemplo, "id": 3, "titulo": "Llamar al médico",
         "completada": True},
    ]


def test_lista_tiene_tres_tareas(lista_tareas):
    assert len(lista_tareas) == 3


def test_hay_una_tarea_completada(lista_tareas):
    completadas = [t for t in lista_tareas if t["completada"]]
    assert len(completadas) == 1
```

---

## Scope de fixtures — cuándo se crean y destruyen

```python
# tests/test_scope.py
import pytest


# scope="function" (por defecto) — nueva instancia por cada test
@pytest.fixture(scope="function")
def conexion_ligera():
    """Se crea y destruye para CADA test — costoso si hay muchos."""
    print("\n→ Abriendo conexión ligera")
    conn = {"estado": "abierta", "operaciones": 0}
    yield conn
    print("\n← Cerrando conexión ligera")
    conn["estado"] = "cerrada"


# scope="module" — una instancia para todo el módulo (archivo)
@pytest.fixture(scope="module")
def conexion_modulo():
    """Se crea UNA VEZ para todos los tests del archivo."""
    print("\n→ Abriendo conexión de módulo")
    conn = {"estado": "abierta", "operaciones": 0}
    yield conn
    print("\n← Cerrando conexión de módulo")


# scope="session" — una instancia para toda la sesión de pytest
@pytest.fixture(scope="session")
def configuracion_global():
    """Se crea UNA VEZ para toda la ejecución de pytest."""
    return {
        "entorno": "test",
        "base_url": "http://localhost:8000",
        "timeout": 30,
    }


def test_conexion_a(conexion_ligera):
    conexion_ligera["operaciones"] += 1
    assert conexion_ligera["estado"] == "abierta"


def test_conexion_b(conexion_ligera):
    # Esta fixture empieza FRESCA — operations=0 de nuevo
    assert conexion_ligera["operaciones"] == 0


def test_config(configuracion_global):
    assert configuracion_global["entorno"] == "test"
```

---

## conftest.py — fixtures compartidas entre archivos

Las fixtures definidas en `conftest.py` están disponibles en todos
los tests del mismo directorio y subdirectorios, sin necesidad de importar.

```python
# tests/conftest.py
import pytest
import httpx
from fastapi.testclient import TestClient

# Importar la app FastAPI (ajustar según tu proyecto)
# from app.main import app


# ── Datos compartidos ─────────────────────────────────────────────

@pytest.fixture(scope="session")
def datos_usuario():
    """Usuario de prueba disponible en todos los tests."""
    return {
        "id": 1,
        "nombre": "Test User",
        "email": "test@ejemplo.com",
        "activo": True,
    }


@pytest.fixture
def tarea_base():
    """Tarea mínima válida para tests de creación."""
    return {
        "titulo": "Tarea de prueba",
        "descripcion": "Descripción de prueba",
        "prioridad": "media",
    }


# ── Cliente HTTP para tests de integración ────────────────────────

# @pytest.fixture(scope="module")
# def cliente():
#     """TestClient de FastAPI — no necesita servidor en ejecución."""
#     with TestClient(app) as c:
#         yield c


# ── Base de datos en memoria para tests ──────────────────────────

@pytest.fixture
def base_datos_mock():
    """Diccionario que simula una base de datos en memoria."""
    return {
        "tareas": [
            {"id": 1, "titulo": "Tarea 1", "completada": False},
            {"id": 2, "titulo": "Tarea 2", "completada": True},
        ],
        "proximo_id": 3,
    }
```

---

## @pytest.mark.parametrize — múltiples casos de prueba

`parametrize` permite ejecutar el mismo test con diferentes valores de entrada,
evitando duplicar código y haciendo los casos de prueba explícitos.

```python
# tests/test_parametrize.py
import pytest


def validar_email(email: str) -> bool:
    """Validación simple de email."""
    return "@" in email and "." in email.split("@")[-1]


def calcular_descuento(precio: float, porcentaje: float) -> float:
    """Aplica un descuento a un precio."""
    if porcentaje < 0 or porcentaje > 100:
        raise ValueError(f"Porcentaje inválido: {porcentaje}")
    return round(precio * (1 - porcentaje / 100), 2)


# Caso simple — un parámetro con múltiples valores
@pytest.mark.parametrize("email", [
    "usuario@ejemplo.com",
    "otro.usuario@dominio.org",
    "test+tag@empresa.co",
])
def test_emails_validos(email):
    assert validar_email(email)


@pytest.mark.parametrize("email", [
    "sin-arroba.com",
    "sin-punto@dominio",
    "",
    "doble@@dominio.com",
])
def test_emails_invalidos(email):
    assert not validar_email(email)


# Múltiples parámetros — tuplas (entrada, salida esperada)
@pytest.mark.parametrize("precio, porcentaje, esperado", [
    (100.0,  10.0, 90.0),
    (200.0,  25.0, 150.0),
    (50.0,    0.0, 50.0),
    (99.99, 100.0, 0.0),
    (33.33,  33.0, 22.33),
])
def test_calcular_descuento(precio, porcentaje, esperado):
    resultado = calcular_descuento(precio, porcentaje)
    assert resultado == pytest.approx(esperado, abs=0.01)


# Casos que deben lanzar excepciones
@pytest.mark.parametrize("precio, porcentaje", [
    (100.0, -1.0),
    (100.0, 101.0),
])
def test_descuento_invalido(precio, porcentaje):
    with pytest.raises(ValueError, match="Porcentaje inválido"):
        calcular_descuento(precio, porcentaje)


# IDs personalizados para que los resultados sean legibles
@pytest.mark.parametrize("valor, esperado", [
    pytest.param([], False, id="lista_vacia"),
    pytest.param([1, 2, 3], True, id="lista_con_elementos"),
    pytest.param(None, False, id="ninguno"),
], )
def test_tiene_elementos(valor, esperado):
    resultado = bool(valor)
    assert resultado == esperado
```

---

## Monkeypatching — reemplazar comportamiento en tests

La fixture `monkeypatch` permite reemplazar atributos, funciones y variables
de entorno sin afectar el código de producción ni otros tests.

```python
# tests/test_monkeypatch.py
import pytest
import os


def obtener_entorno() -> str:
    """Lee el entorno desde variable de entorno."""
    return os.environ.get("APP_ENV", "development")


def leer_archivo_config(ruta: str) -> dict:
    """Lee configuración de un archivo."""
    import json
    with open(ruta) as f:
        return json.load(f)


class ServicioEmail:
    def enviar(self, destinatario: str, asunto: str, cuerpo: str) -> bool:
        """Envía email real — costoso en tests."""
        # En producción llamaría a una API de email
        print(f"Enviando email a {destinatario}...")
        return True


# Reemplazar variable de entorno
def test_entorno_produccion(monkeypatch):
    monkeypatch.setenv("APP_ENV", "production")
    assert obtener_entorno() == "production"


def test_entorno_sin_variable(monkeypatch):
    monkeypatch.delenv("APP_ENV", raising=False)
    assert obtener_entorno() == "development"


# Reemplazar una función completa
def test_leer_config_con_mock(monkeypatch):
    config_falsa = {"debug": True, "db": "sqlite:///test.db"}

    def mock_open(*args, **kwargs):
        import io
        import json
        return io.StringIO(json.dumps(config_falsa))

    monkeypatch.setattr("builtins.open", mock_open)
    # Ahora leer_archivo_config() devuelve datos falsos sin tocar el disco
    # resultado = leer_archivo_config("config.json")
    # assert resultado["debug"] is True


# Reemplazar un método de clase
def test_servicio_email_mock(monkeypatch):
    emails_enviados = []

    def enviar_mock(self, destinatario, asunto, cuerpo):
        emails_enviados.append({
            "destinatario": destinatario,
            "asunto": asunto,
        })
        return True

    monkeypatch.setattr(ServicioEmail, "enviar", enviar_mock)

    servicio = ServicioEmail()
    servicio.enviar("test@ejemplo.com", "Bienvenido", "Hola")

    assert len(emails_enviados) == 1
    assert emails_enviados[0]["destinatario"] == "test@ejemplo.com"
```

---

## unittest.mock — Mock, MagicMock, patch

`unittest.mock` es la librería estándar de mocking de Python.
`pytest` la integra perfectamente. Úsala cuando necesitas control fino.

```python
# tests/test_mocks.py
from unittest.mock import Mock, MagicMock, patch, AsyncMock, call
import pytest


# ── Mock básico ──────────────────────────────────────────────────

def test_mock_basico():
    # Mock() acepta cualquier atributo y llamada sin configurar
    repositorio = Mock()

    # Configurar el valor de retorno
    repositorio.obtener_por_id.return_value = {
        "id": 1, "titulo": "Tarea mock"
    }

    # Llamar al mock
    tarea = repositorio.obtener_por_id(1)

    # Verificar resultado
    assert tarea["titulo"] == "Tarea mock"

    # Verificar que se llamó correctamente
    repositorio.obtener_por_id.assert_called_once_with(1)


# ── MagicMock — soporta métodos mágicos (__len__, __iter__, etc.) ─

def test_magic_mock():
    coleccion = MagicMock()
    coleccion.__len__.return_value = 5
    coleccion.__iter__.return_value = iter([1, 2, 3, 4, 5])

    assert len(coleccion) == 5
    assert list(coleccion) == [1, 2, 3, 4, 5]


# ── side_effect — lanzar excepciones o comportamiento dinámico ───

def test_mock_con_excepcion():
    servicio = Mock()
    servicio.conectar.side_effect = ConnectionError("Sin conexión")

    with pytest.raises(ConnectionError, match="Sin conexión"):
        servicio.conectar()


def test_mock_side_effect_dinamico():
    """side_effect con función — comportamiento diferente en cada llamada."""
    contador = {"n": 0}

    def respuesta_dinamica(id_tarea):
        contador["n"] += 1
        if id_tarea == 999:
            return None
        return {"id": id_tarea, "titulo": f"Tarea {id_tarea}"}

    repo = Mock()
    repo.buscar.side_effect = respuesta_dinamica

    assert repo.buscar(1)["titulo"] == "Tarea 1"
    assert repo.buscar(42)["titulo"] == "Tarea 42"
    assert repo.buscar(999) is None
    assert contador["n"] == 3


# ── patch — reemplazar durante el test con context manager ───────

class ClienteHTTP:
    def get(self, url: str) -> dict:
        import httpx
        respuesta = httpx.get(url)
        return respuesta.json()


def test_patch_como_context_manager():
    """patch() reemplaza el objeto solo dentro del bloque with."""
    with patch("httpx.get") as mock_get:
        mock_get.return_value.json.return_value = {
            "id": 1, "name": "Producto A"
        }
        mock_get.return_value.status_code = 200

        cliente = ClienteHTTP()
        # httpx.get está reemplazado — no hace llamada HTTP real
        # resultado = cliente.get("https://api.ejemplo.com/productos/1")
        # assert resultado["name"] == "Producto A"


# ── patch como decorador ─────────────────────────────────────────

@patch("os.path.exists")
@patch("os.path.getsize")
def test_patch_decorador(mock_getsize, mock_exists):
    """Decoradores se apilan de abajo hacia arriba — el más cercano va primero."""
    mock_exists.return_value = True
    mock_getsize.return_value = 1024

    import os
    assert os.path.exists("/archivo/falso.txt") is True
    assert os.path.getsize("/archivo/falso.txt") == 1024


# ── Verificar llamadas múltiples ─────────────────────────────────

def test_verificar_multiples_llamadas():
    servicio = Mock()
    servicio.procesar.return_value = "OK"

    # Simular múltiples llamadas
    for i in range(3):
        servicio.procesar(f"item_{i}")

    # Verificar número de llamadas
    assert servicio.procesar.call_count == 3

    # Verificar todas las llamadas en orden
    servicio.procesar.assert_has_calls([
        call("item_0"),
        call("item_1"),
        call("item_2"),
    ])
```

---

## AsyncMock — testear código asíncrono

```python
# tests/test_async.py
import pytest
from unittest.mock import AsyncMock, patch


# Función async a testear
async def obtener_usuario(id_usuario: int, cliente_http) -> dict | None:
    """Obtiene un usuario de la API."""
    respuesta = await cliente_http.get(f"/usuarios/{id_usuario}")
    if respuesta.status_code == 404:
        return None
    return respuesta.json()


async def crear_tarea_y_notificar(
    titulo: str,
    repo,
    notificador,
) -> dict:
    """Crea tarea y envía notificación — ambas operaciones son async."""
    tarea = await repo.crear({"titulo": titulo, "completada": False})
    await notificador.enviar(f"Tarea creada: {titulo}")
    return tarea


# ── Tests con pytest-asyncio ──────────────────────────────────────

@pytest.mark.asyncio
async def test_obtener_usuario_existente():
    """Testear función async con AsyncMock."""
    cliente_mock = AsyncMock()

    # Configurar respuesta del mock async
    cliente_mock.get.return_value.status_code = 200
    cliente_mock.get.return_value.json.return_value = {
        "id": 1,
        "nombre": "Ana García",
        "email": "ana@ejemplo.com",
    }

    usuario = await obtener_usuario(1, cliente_mock)

    assert usuario is not None
    assert usuario["nombre"] == "Ana García"
    cliente_mock.get.assert_called_once_with("/usuarios/1")


@pytest.mark.asyncio
async def test_obtener_usuario_no_existe():
    """Cuando la API retorna 404, la función devuelve None."""
    cliente_mock = AsyncMock()
    cliente_mock.get.return_value.status_code = 404

    usuario = await obtener_usuario(999, cliente_mock)

    assert usuario is None


@pytest.mark.asyncio
async def test_crear_tarea_llama_notificador():
    """Verificar que el notificador se llama después de crear la tarea."""
    repo_mock = AsyncMock()
    notificador_mock = AsyncMock()

    tarea_creada = {"id": 42, "titulo": "Mi tarea", "completada": False}
    repo_mock.crear.return_value = tarea_creada

    resultado = await crear_tarea_y_notificar(
        "Mi tarea", repo_mock, notificador_mock
    )

    # Verificar resultado
    assert resultado["id"] == 42

    # Verificar que se llamaron ambos métodos async
    repo_mock.crear.assert_called_once_with(
        {"titulo": "Mi tarea", "completada": False}
    )
    notificador_mock.enviar.assert_called_once_with("Tarea creada: Mi tarea")
```

---

## Testear APIs FastAPI con TestClient

```python
# tests/test_api_tareas.py
import pytest
from fastapi.testclient import TestClient
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel


# ── App de ejemplo (normalmente importaría de app/main.py) ───────

class TareaIn(BaseModel):
    titulo: str
    descripcion: str = ""
    prioridad: str = "media"


class TareaOut(TareaIn):
    id: int
    completada: bool = False


# "Base de datos" en memoria para el ejemplo
_db: dict[int, TareaOut] = {}
_siguiente_id = 1

app_test = FastAPI()


@app_test.get("/tareas", response_model=list[TareaOut])
def listar_tareas():
    return list(_db.values())


@app_test.post("/tareas", response_model=TareaOut, status_code=201)
def crear_tarea(datos: TareaIn):
    global _siguiente_id
    tarea = TareaOut(id=_siguiente_id, **datos.model_dump())
    _db[_siguiente_id] = tarea
    _siguiente_id += 1
    return tarea


@app_test.get("/tareas/{id_tarea}", response_model=TareaOut)
def obtener_tarea(id_tarea: int):
    if id_tarea not in _db:
        raise HTTPException(status_code=404, detail="Tarea no encontrada")
    return _db[id_tarea]


# ── Fixtures ─────────────────────────────────────────────────────

@pytest.fixture(autouse=True)
def limpiar_db():
    """Limpiar la DB en memoria antes de cada test."""
    global _siguiente_id
    _db.clear()
    _siguiente_id = 1
    yield


@pytest.fixture
def cliente():
    """TestClient — no necesita servidor HTTP levantado."""
    return TestClient(app_test)


@pytest.fixture
def tarea_existente(cliente):
    """Crea una tarea y la devuelve para usar en tests que la necesitan."""
    respuesta = cliente.post("/tareas", json={
        "titulo": "Tarea existente",
        "descripcion": "Para tests de lectura y actualización",
        "prioridad": "alta",
    })
    return respuesta.json()


# ── Tests de la API ───────────────────────────────────────────────

def test_listar_tareas_vacio(cliente):
    respuesta = cliente.get("/tareas")
    assert respuesta.status_code == 200
    assert respuesta.json() == []


def test_crear_tarea(cliente):
    datos = {"titulo": "Nueva tarea", "prioridad": "alta"}
    respuesta = cliente.post("/tareas", json=datos)

    assert respuesta.status_code == 201
    cuerpo = respuesta.json()
    assert cuerpo["id"] == 1
    assert cuerpo["titulo"] == "Nueva tarea"
    assert cuerpo["completada"] is False


def test_crear_tarea_campos_requeridos(cliente):
    """Falta el campo 'titulo' — debe retornar 422."""
    respuesta = cliente.post("/tareas", json={"descripcion": "Sin título"})
    assert respuesta.status_code == 422


def test_obtener_tarea_existente(cliente, tarea_existente):
    id_tarea = tarea_existente["id"]
    respuesta = cliente.get(f"/tareas/{id_tarea}")

    assert respuesta.status_code == 200
    assert respuesta.json()["titulo"] == "Tarea existente"


def test_obtener_tarea_no_existe(cliente):
    respuesta = cliente.get("/tareas/999")
    assert respuesta.status_code == 404
    assert "no encontrada" in respuesta.json()["detail"]


def test_listar_tareas_despues_de_crear(cliente):
    """Crear varias tareas y verificar que aparecen en la lista."""
    for i in range(3):
        cliente.post("/tareas", json={"titulo": f"Tarea {i + 1}"})

    respuesta = cliente.get("/tareas")
    assert respuesta.status_code == 200
    assert len(respuesta.json()) == 3
```

---

## pytest-cov — cobertura de código

```bash
# Instalar
pip install pytest-cov

# Ejecutar tests con reporte de cobertura en terminal
pytest --cov=app tests/

# Reporte HTML detallado (ver qué líneas no se cubren)
pytest --cov=app --cov-report=html tests/
# Abre htmlcov/index.html en el navegador

# Reporte con líneas no cubiertas en terminal
pytest --cov=app --cov-report=term-missing tests/

# Fallar si la cobertura baja del umbral
pytest --cov=app --cov-fail-under=80 tests/
```

Configuración en `pyproject.toml` para no tener que escribir flags cada vez:

```toml
# pyproject.toml

[tool.pytest.ini_options]
# Directorio de tests
testpaths = ["tests"]
# Flags por defecto
addopts = "-v --tb=short"
# Modo asyncio para pytest-asyncio
asyncio_mode = "auto"

[tool.coverage.run]
# Directorio a medir
source = ["app"]
# Excluir archivos que no necesitan cobertura
omit = [
    "app/migrations/*",
    "app/tests/*",
    "*/__init__.py",
]

[tool.coverage.report]
# Umbral mínimo — falla si la cobertura baja de este porcentaje
fail_under = 80
# Mostrar líneas no cubiertas
show_missing = true
# Excluir líneas específicas del reporte
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "@abstractmethod",
]
```

---

## Ejemplo aplicado — Suite de tests para API de tareas

Esta es la suite completa para testear la API del Módulo 8.
Incluye tests unitarios de la lógica, de integración de la API
y mocks de dependencias externas.

```python
# tests/test_suite_completa.py
"""
Suite de tests para la API de tareas del módulo anterior.
Demuestra: unit tests, integration tests y mocking de BD.
"""
import pytest
from unittest.mock import MagicMock, AsyncMock, patch
from fastapi.testclient import TestClient
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, field_validator
from typing import Annotated


# ── Modelos de dominio ────────────────────────────────────────────

class Tarea(BaseModel):
    id: int
    titulo: str
    descripcion: str = ""
    prioridad: str = "media"
    completada: bool = False
    etiquetas: list[str] = []

    @field_validator("prioridad")
    @classmethod
    def validar_prioridad(cls, v: str) -> str:
        opciones = {"baja", "media", "alta", "critica"}
        if v not in opciones:
            raise ValueError(f"Prioridad debe ser una de: {opciones}")
        return v

    @field_validator("titulo")
    @classmethod
    def validar_titulo(cls, v: str) -> str:
        if len(v.strip()) < 3:
            raise ValueError("El título debe tener al menos 3 caracteres")
        return v.strip()

    @property
    def es_urgente(self) -> bool:
        return self.prioridad in ("alta", "critica")


# ── Lógica de negocio — pura, sin FastAPI ────────────────────────

class ServicioTareas:
    def __init__(self, repositorio):
        self._repo = repositorio

    def obtener_tareas_urgentes(self) -> list[Tarea]:
        """Retorna solo las tareas urgentes no completadas."""
        todas = self._repo.listar()
        return [
            t for t in todas
            if t.es_urgente and not t.completada
        ]

    def completar_tarea(self, id_tarea: int) -> Tarea:
        """Marca una tarea como completada."""
        tarea = self._repo.obtener(id_tarea)
        if tarea is None:
            raise ValueError(f"Tarea {id_tarea} no encontrada")
        if tarea.completada:
            raise ValueError(f"La tarea {id_tarea} ya está completada")

        tarea_actualizada = tarea.model_copy(update={"completada": True})
        return self._repo.guardar(tarea_actualizada)

    def estadisticas(self) -> dict:
        """Calcula estadísticas del conjunto de tareas."""
        todas = self._repo.listar()
        completadas = [t for t in todas if t.completada]
        pendientes = [t for t in todas if not t.completada]

        return {
            "total": len(todas),
            "completadas": len(completadas),
            "pendientes": len(pendientes),
            "por_prioridad": {
                prioridad: len([t for t in pendientes
                                if t.prioridad == prioridad])
                for prioridad in ("baja", "media", "alta", "critica")
            },
        }


# ══════════════════════════════════════════════════════════════════
# UNIT TESTS — testean lógica de negocio con mocks
# ══════════════════════════════════════════════════════════════════

class TestTareaModelo:
    """Tests unitarios del modelo Tarea — sin BD ni HTTP."""

    def test_tarea_valida(self):
        tarea = Tarea(id=1, titulo="Comprar pan", prioridad="alta")
        assert tarea.titulo == "Comprar pan"
        assert not tarea.completada

    def test_titulo_muy_corto_falla(self):
        with pytest.raises(Exception):  # ValidationError
            Tarea(id=1, titulo="AB")

    def test_titulo_se_limpia(self):
        tarea = Tarea(id=1, titulo="  Tarea con espacios  ")
        assert tarea.titulo == "Tarea con espacios"

    def test_prioridad_invalida_falla(self):
        with pytest.raises(Exception):  # ValidationError
            Tarea(id=1, titulo="Test", prioridad="urgentisima")

    @pytest.mark.parametrize("prioridad, esperado", [
        ("alta",    True),
        ("critica", True),
        ("media",   False),
        ("baja",    False),
    ])
    def test_es_urgente(self, prioridad, esperado):
        tarea = Tarea(id=1, titulo="Test", prioridad=prioridad)
        assert tarea.es_urgente == esperado


class TestServicioTareas:
    """Tests unitarios del servicio — repositorio mockeado."""

    @pytest.fixture
    def repo_mock(self):
        """Repositorio falso que controla lo que devuelve."""
        return MagicMock()

    @pytest.fixture
    def servicio(self, repo_mock):
        return ServicioTareas(repo_mock)

    @pytest.fixture
    def tareas_ejemplo(self):
        return [
            Tarea(id=1, titulo="Fix bug crítico", prioridad="critica"),
            Tarea(id=2, titulo="Revisar PR", prioridad="alta"),
            Tarea(id=3, titulo="Actualizar docs", prioridad="baja"),
            Tarea(id=4, titulo="Deploy", prioridad="critica",
                  completada=True),
        ]

    def test_obtener_tareas_urgentes(self, servicio, repo_mock,
                                     tareas_ejemplo):
        repo_mock.listar.return_value = tareas_ejemplo

        urgentes = servicio.obtener_tareas_urgentes()

        # Debe retornar solo las urgentes NO completadas
        assert len(urgentes) == 2
        assert all(t.es_urgente for t in urgentes)
        assert all(not t.completada for t in urgentes)

    def test_completar_tarea_exitoso(self, servicio, repo_mock):
        tarea = Tarea(id=1, titulo="Tarea pendiente", prioridad="media")
        tarea_completada = tarea.model_copy(update={"completada": True})

        repo_mock.obtener.return_value = tarea
        repo_mock.guardar.return_value = tarea_completada

        resultado = servicio.completar_tarea(1)

        assert resultado.completada is True
        repo_mock.obtener.assert_called_once_with(1)
        repo_mock.guardar.assert_called_once()

    def test_completar_tarea_no_existe(self, servicio, repo_mock):
        repo_mock.obtener.return_value = None

        with pytest.raises(ValueError, match="no encontrada"):
            servicio.completar_tarea(999)

    def test_completar_tarea_ya_completada(self, servicio, repo_mock):
        tarea_ya_completada = Tarea(
            id=1, titulo="Ya hecha", prioridad="media", completada=True
        )
        repo_mock.obtener.return_value = tarea_ya_completada

        with pytest.raises(ValueError, match="ya está completada"):
            servicio.completar_tarea(1)

    def test_estadisticas(self, servicio, repo_mock, tareas_ejemplo):
        repo_mock.listar.return_value = tareas_ejemplo

        stats = servicio.estadisticas()

        assert stats["total"] == 4
        assert stats["completadas"] == 1
        assert stats["pendientes"] == 3
        assert stats["por_prioridad"]["critica"] == 1  # 1 critica pendiente
        assert stats["por_prioridad"]["alta"] == 1
        assert stats["por_prioridad"]["baja"] == 1
        assert stats["por_prioridad"]["media"] == 0


# ══════════════════════════════════════════════════════════════════
# INTEGRATION TESTS — testean la API completa con TestClient
# ══════════════════════════════════════════════════════════════════

# App de integración (simplificada)
_tareas_db: dict[int, dict] = {}
_id_counter = 1
app_integracion = FastAPI(title="API Tareas Test")


@app_integracion.get("/tareas", response_model=list[dict])
def api_listar():
    return list(_tareas_db.values())


@app_integracion.post("/tareas", status_code=201)
def api_crear(datos: dict):
    global _id_counter
    tarea = {"id": _id_counter, "completada": False, **datos}
    _tareas_db[_id_counter] = tarea
    _id_counter += 1
    return tarea


@app_integracion.patch("/tareas/{id_tarea}/completar")
def api_completar(id_tarea: int):
    if id_tarea not in _tareas_db:
        raise HTTPException(404, "No encontrada")
    _tareas_db[id_tarea]["completada"] = True
    return _tareas_db[id_tarea]


@pytest.fixture(autouse=True)
def resetear_estado():
    """Resetear la DB en memoria entre cada test."""
    global _id_counter
    _tareas_db.clear()
    _id_counter = 1
    yield


@pytest.fixture
def api():
    return TestClient(app_integracion)


class TestIntegracionAPI:
    def test_flujo_completo(self, api):
        """Test de flujo: crear → listar → completar."""
        # 1. Crear tarea
        resp = api.post("/tareas", json={
            "titulo": "Escribir tests de integración",
            "prioridad": "alta",
        })
        assert resp.status_code == 201
        id_tarea = resp.json()["id"]

        # 2. Verificar que aparece en la lista
        resp = api.get("/tareas")
        assert len(resp.json()) == 1
        assert resp.json()[0]["completada"] is False

        # 3. Completar la tarea
        resp = api.patch(f"/tareas/{id_tarea}/completar")
        assert resp.status_code == 200
        assert resp.json()["completada"] is True

    def test_completar_no_existente(self, api):
        resp = api.patch("/tareas/999/completar")
        assert resp.status_code == 404
```

---

## Ejercicios propuestos

1. **Tests para validador de contraseñas** — Crea la función `validar_contrasena(pwd: str) -> dict` que retorne `{"valida": bool, "errores": list[str]}` verificando: longitud mínima 8, al menos una mayúscula, al menos un número, al menos un carácter especial. Escribe al menos 10 tests con `@pytest.mark.parametrize` cubriendo casos válidos e inválidos, y verifica que `errores` contiene exactamente los mensajes correctos cuando falla cada regla.

2. **Mock de servicio externo de clima** — Crea `ServicioClima` con el método `obtener_temperatura(ciudad: str) -> float` que internamente hace `httpx.get(...)`. Crea `AlertaTemperatura` que usa `ServicioClima` y envía una notificación cuando la temperatura supera 35°C. Testea `AlertaTemperatura` mockeando `ServicioClima` sin tocar HTTP. Usa `AsyncMock` si implementas el servicio como async y testea con `pytest-asyncio`.

3. **Fixture con base de datos SQLite** — Crea una fixture de scope `"function"` que cree una base de datos SQLite en memoria usando `sqlite3`, ejecute un script SQL de schema, y la destruya al terminar. Crea un repositorio `RepositorioProductosSQLite` con métodos `insertar`, `obtener`, `listar` y `eliminar`. Escribe tests de integración para cada método usando la fixture. Verifica que los datos persisten entre llamadas dentro del mismo test pero no entre tests diferentes.

4. **Coverage al 90%** — Toma el módulo `ServicioTareas` del ejemplo aplicado. Ejecuta `pytest --cov=. --cov-report=term-missing` y analiza las líneas no cubiertas. Agrega los tests necesarios hasta alcanzar al menos 90% de cobertura. Configura `pyproject.toml` con `fail_under = 90` para que el CI falle si la cobertura baja. Incluye al menos un test que use `pytest.mark.parametrize` con los 4 niveles de prioridad.

---

## Resumen de la página 15

- `pytest` detecta automáticamente tests en archivos `test_*.py` y funciones `test_*()` — no necesitas registrar nada ni heredar de clases.
- Las **fixtures** se inyectan por nombre en los parámetros — `scope="function"` (por defecto) crea una instancia nueva por test; `scope="session"` crea solo una para toda la ejecución. El `yield` dentro de una fixture separa el setup del teardown.
- `conftest.py` es el lugar para fixtures compartidas — pytest lo detecta automáticamente en cada directorio, sin imports.
- `@pytest.mark.parametrize` evita duplicar tests: un test con 10 casos parametrizados es más mantenible que 10 funciones separadas.
- `monkeypatch` es la fixture oficial de pytest para reemplazar atributos, funciones y variables de entorno — scope de función, sin efectos secundarios entre tests.
- `unittest.mock.patch` es más potente para mocking complejo: `side_effect` permite lanzar excepciones o comportamiento dinámico; `assert_called_once_with` verifica la llamada exacta.
- `AsyncMock` es el equivalente async de `Mock` — necesario para testear funciones `async def` que llaman a otras coroutines.
- `TestClient` de FastAPI permite testear endpoints HTTP sin levantar un servidor real — los tests son síncronos aunque la app sea async.
- `pytest-cov` mide la cobertura de código; configurar `fail_under` en `pyproject.toml` convierte la cobertura en un requisito del CI/CD.

---

> **Siguiente página →** Página 19: Herramientas — Ruff, pre-commit, Typer CLI y Rich.
