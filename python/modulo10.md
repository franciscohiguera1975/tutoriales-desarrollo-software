# Tutorial Python — Página 10
## Módulo 4 · Ecosistema Python
### Módulos, paquetes, pip, virtualenv y Poetry

---

## Parte 1 — Sistema de importación

Un **módulo** es cualquier archivo `.py`. Un **paquete** es un directorio con
un archivo `__init__.py`. Python sigue un orden determinado al resolver imports:
primero el directorio actual, luego las rutas en `sys.path`, y finalmente la
biblioteca estándar.

```python
# ─────────────────────────────────────────────────────────────────────────────
# Importaciones absolutas — siempre relativas a la raíz del proyecto
# ─────────────────────────────────────────────────────────────────────────────
import os                          # módulo de la biblioteca estándar
import json                        # otro módulo estándar
from pathlib import Path           # importar solo lo que se necesita
from collections import defaultdict, Counter

# Alias para nombres largos o colisiones
import numpy as np                 # convención universal en data science
import pandas as pd

# Importar todo — solo en el REPL o en __init__.py, nunca en módulos
# from modulo import *               # dificulta saber qué está disponible


# ─────────────────────────────────────────────────────────────────────────────
# Importaciones relativas — dentro de un paquete propio
# Solo funcionan dentro de un paquete (no en scripts ejecutados directamente)
# ─────────────────────────────────────────────────────────────────────────────
# Estructura del paquete:
#   miapp/
#     __init__.py
#     modelos/
#       __init__.py
#       usuario.py
#     servicios/
#       __init__.py
#       auth.py       ← este archivo usa importaciones relativas

# Dentro de miapp/servicios/auth.py:
# from ..modelos.usuario import Usuario    # sube un nivel, entra en modelos
# from . import utils                      # importa utils del mismo paquete
# from .validators import validar_email    # importa del mismo directorio


# ─────────────────────────────────────────────────────────────────────────────
# __all__ — controla qué se exporta cuando alguien hace "from modulo import *"
# ─────────────────────────────────────────────────────────────────────────────
# Dentro de miapp/modelos/__init__.py:
__all__ = ["Usuario", "Producto", "Pedido"]  # solo estas clases son públicas
# Los demás nombres (comenzando con _ o no listados) no se exportan


# ─────────────────────────────────────────────────────────────────────────────
# __init__.py — punto de entrada del paquete
# Sirve para re-exportar y simplificar el API público del paquete
# ─────────────────────────────────────────────────────────────────────────────
# Contenido típico de miapp/__init__.py:
# from .modelos.usuario import Usuario
# from .modelos.producto import Producto
# from .servicios.auth import autenticar
# __version__ = "1.0.0"
# __author__ = "Tu Nombre"

# Con esto, los usuarios pueden hacer:
# from miapp import Usuario            # en lugar de
# from miapp.modelos.usuario import Usuario
```

---

## Parte 2 — pip y gestión de dependencias

```bash
# ─────────────────────────────────────────────────────────────────────────────
# Comandos pip esenciales
# ─────────────────────────────────────────────────────────────────────────────

# Instalar un paquete
pip install requests

# Instalar una versión específica
pip install requests==2.31.0

# Instalar con restricción de versión mínima
pip install "httpx>=0.26"

# Instalar varios paquetes a la vez
pip install fastapi uvicorn pydantic

# Desinstalar
pip uninstall requests -y

# Ver qué hay instalado
pip list
pip list --outdated              # mostrar los que tienen actualizaciones

# Exportar dependencias actuales a un archivo
pip freeze > requirements.txt

# Instalar desde requirements.txt
pip install -r requirements.txt

# Ver información de un paquete instalado
pip show requests

# Buscar paquetes (busca en PyPI)
pip search "http client"         # obsoleto en versiones recientes
# Usar https://pypi.org en su lugar
```

```
# requirements.txt — ejemplo real con versiones fijadas
requests==2.31.0
httpx==0.27.0
fastapi==0.110.0
uvicorn[standard]==0.29.0
pydantic==2.6.4
python-dotenv==1.0.1
sqlalchemy==2.0.28
alembic==1.13.1
```

---

## Parte 3 — Entornos virtuales

```bash
# ─────────────────────────────────────────────────────────────────────────────
# venv — incluido en Python 3.3+, sin instalación extra
# ─────────────────────────────────────────────────────────────────────────────

# Crear un entorno virtual en el directorio .venv
python3 -m venv .venv

# Activar (Linux / macOS)
source .venv/bin/activate

# Activar (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Activar (Windows CMD)
.venv\Scripts\activate.bat

# Verificar que estamos en el entorno virtual
which python           # /ruta/al/proyecto/.venv/bin/python

# Desactivar
deactivate

# Eliminar el entorno virtual (simplemente borrar el directorio)
rm -rf .venv


# ─────────────────────────────────────────────────────────────────────────────
# pyenv — gestiona múltiples versiones de Python en el sistema
# ─────────────────────────────────────────────────────────────────────────────

# Instalar pyenv (Linux/macOS)
curl https://pyenv.run | bash

# Listar versiones disponibles
pyenv install --list

# Instalar una versión específica
pyenv install 3.12.3
pyenv install 3.11.9

# Establecer la versión global del sistema
pyenv global 3.12.3

# Establecer la versión local para un proyecto (crea .python-version)
cd mi_proyecto
pyenv local 3.11.9

# Ver versión activa
pyenv version          # 3.11.9 (set by /ruta/.python-version)
```

---

## Parte 4 — Poetry

Poetry es el gestor de dependencias y entornos moderno de Python. Combina
lo que hacen pip + venv + setuptools en una sola herramienta con resolución
de dependencias determinista.

```bash
# ─────────────────────────────────────────────────────────────────────────────
# Instalación de Poetry
# ─────────────────────────────────────────────────────────────────────────────
curl -sSL https://install.python-poetry.org | python3 -

# Crear nuevo proyecto
poetry new mi_proyecto
# Genera la estructura:
#   mi_proyecto/
#     pyproject.toml
#     README.md
#     mi_proyecto/
#       __init__.py
#     tests/
#       __init__.py

# Inicializar Poetry en un proyecto existente
cd proyecto_existente
poetry init                      # guía interactiva


# ─────────────────────────────────────────────────────────────────────────────
# Gestión de dependencias
# ─────────────────────────────────────────────────────────────────────────────

# Agregar dependencia de producción
poetry add fastapi
poetry add "httpx>=0.27"

# Agregar dependencia de desarrollo (solo para desarrollo, no en producción)
poetry add --group dev pytest pytest-asyncio ruff mypy

# Agregar dependencia de testing
poetry add --group test coverage pytest-cov

# Instalar todas las dependencias (incluye dev y test por defecto)
poetry install

# Instalar solo dependencias de producción (para despliegue)
poetry install --only main

# Actualizar dependencias
poetry update                    # actualiza todo
poetry update httpx              # actualiza solo httpx

# Ver árbol de dependencias
poetry show --tree

# Ejecutar un comando dentro del entorno de Poetry
poetry run python mi_script.py
poetry run pytest
poetry run ruff check .
```

```toml
# ─────────────────────────────────────────────────────────────────────────────
# pyproject.toml — archivo de configuración central (generado por Poetry)
# ─────────────────────────────────────────────────────────────────────────────
[tool.poetry]
name = "mi-api"
version = "0.1.0"
description = "API REST con FastAPI"
authors = ["Tu Nombre <tu@email.com>"]
readme = "README.md"
packages = [{include = "mi_api"}]

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.110.0"
uvicorn = {extras = ["standard"], version = "^0.29.0"}
pydantic = "^2.6.4"
sqlalchemy = "^2.0.28"
python-dotenv = "^1.0.1"

[tool.poetry.group.dev.dependencies]
ruff = "^0.3.0"
mypy = "^1.9.0"

[tool.poetry.group.test.dependencies]
pytest = "^8.1.0"
pytest-asyncio = "^0.23.0"
httpx = "^0.27.0"     # cliente HTTP para tests de la API

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# ── Configuración de herramientas en el mismo archivo ─────────────────────
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP"]   # errores, flake8, isort, naming, pyupgrade

[tool.mypy]
python_version = "3.12"
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

---

## Parte 5 — Variables de entorno con python-dotenv

```python
# ─────────────────────────────────────────────────────────────────────────────
# .env — archivo de configuración local (NUNCA subir a git)
# ─────────────────────────────────────────────────────────────────────────────
# DATABASE_URL=postgresql://user:pass@localhost:5432/mi_db
# SECRET_KEY=mi_clave_super_secreta_en_produccion_cambiar
# DEBUG=true
# API_BASE_URL=https://api.ejemplo.com
# MAX_WORKERS=4

import os
from dotenv import load_dotenv

# Cargar el archivo .env en las variables de entorno del proceso
load_dotenv()   # busca .env en el directorio actual y los padres

# Leer variables con os.getenv (devuelve None si no existe)
url_bd = os.getenv("DATABASE_URL")
clave_secreta = os.getenv("SECRET_KEY", "clave_default_para_desarrollo")
debug = os.getenv("DEBUG", "false").lower() == "true"
max_workers = int(os.getenv("MAX_WORKERS", "2"))

# Leer variables obligatorias — lanzar error si no están definidas
def obtener_env(nombre: str) -> str:
    """Obtiene una variable de entorno obligatoria o lanza un error claro."""
    valor = os.getenv(nombre)
    if valor is None:
        raise EnvironmentError(
            f"Variable de entorno requerida no definida: {nombre}\n"
            f"Comprueba tu archivo .env o las variables del sistema."
        )
    return valor

DATABASE_URL = obtener_env("DATABASE_URL")
SECRET_KEY = obtener_env("SECRET_KEY")
```

```
# .gitignore — entradas esenciales para proyectos Python
.env
.env.local
.env.*.local
.venv/
__pycache__/
*.pyc
*.pyo
.pytest_cache/
.mypy_cache/
.ruff_cache/
dist/
build/
*.egg-info/
```

---

## Parte 6 — Estructura recomendada de proyecto Python moderno

```
mi_api/                          # raíz del repositorio
├── .env                         # variables de entorno locales (no en git)
├── .env.example                 # plantilla del .env (sí en git)
├── .gitignore
├── pyproject.toml               # configuración del proyecto y dependencias
├── poetry.lock                  # versiones exactas bloqueadas
├── README.md
│
├── mi_api/                      # código fuente del paquete
│   ├── __init__.py              # versión y exports públicos
│   ├── main.py                  # punto de entrada (app FastAPI, etc.)
│   ├── config.py                # configuración centralizada con Pydantic
│   │
│   ├── modelos/                 # definiciones de datos / ORM
│   │   ├── __init__.py
│   │   ├── usuario.py
│   │   └── producto.py
│   │
│   ├── servicios/               # lógica de negocio
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── pedidos.py
│   │
│   ├── repositorios/            # acceso a base de datos
│   │   ├── __init__.py
│   │   └── usuario_repo.py
│   │
│   └── rutas/                   # endpoints HTTP (routers de FastAPI)
│       ├── __init__.py
│       ├── usuarios.py
│       └── productos.py
│
└── tests/
    ├── __init__.py
    ├── conftest.py              # fixtures compartidos de pytest
    ├── test_usuarios.py
    └── test_productos.py
```

---

## Ejemplo aplicado — Crear un paquete con CLI usando estructura moderna

```python
# ─────────────────────────────────────────────────────────────────────────────
# Estructura del paquete herramienta_csv/
# ─────────────────────────────────────────────────────────────────────────────
# herramienta_csv/
#   __init__.py
#   cli.py          ← punto de entrada de la CLI
#   procesador.py   ← lógica principal
#   formatos.py     ← funciones de formato y salida


# herramienta_csv/__init__.py
"""Herramienta de procesamiento de archivos CSV desde la línea de comandos."""
__version__ = "0.1.0"
__all__ = ["ProcesadorCSV"]


# herramienta_csv/procesador.py
"""Lógica de negocio para leer, filtrar y agregar datos CSV."""
from __future__ import annotations

import csv
from pathlib import Path
from collections import defaultdict
from typing import Callable


class ProcesadorCSV:
    """Lee y transforma archivos CSV con una API funcional encadenada."""

    def __init__(self, ruta: Path | str) -> None:
        self.ruta = Path(ruta)
        self._filas: list[dict[str, str]] = []
        self._cargar()

    def _cargar(self) -> None:
        """Carga todas las filas del CSV en memoria."""
        if not self.ruta.exists():
            raise FileNotFoundError(f"Archivo no encontrado: {self.ruta}")
        with self.ruta.open(encoding="utf-8", newline="") as f:
            lector = csv.DictReader(f)
            self._filas = list(lector)

    def filtrar(self, condicion: Callable[[dict], bool]) -> "ProcesadorCSV":
        """Filtra filas que cumplen la condición. Devuelve self para encadenamiento."""
        self._filas = [fila for fila in self._filas if condicion(fila)]
        return self

    def agrupar_y_sumar(self, columna_grupo: str, columna_valor: str) -> dict[str, float]:
        """Agrupa por columna_grupo y suma los valores de columna_valor."""
        resultado: dict[str, float] = defaultdict(float)
        for fila in self._filas:
            grupo = fila.get(columna_grupo, "sin_grupo")
            try:
                valor = float(fila.get(columna_valor, "0").replace(",", "."))
                resultado[grupo] += valor
            except ValueError:
                pass  # ignorar filas con valores no numéricos
        return dict(sorted(resultado.items(), key=lambda x: x[1], reverse=True))

    def exportar_csv(self, destino: Path | str) -> None:
        """Exporta las filas actuales a un nuevo archivo CSV."""
        destino = Path(destino)
        if not self._filas:
            raise ValueError("No hay filas para exportar")
        destino.parent.mkdir(parents=True, exist_ok=True)
        with destino.open("w", encoding="utf-8", newline="") as f:
            escritor = csv.DictWriter(f, fieldnames=self._filas[0].keys())
            escritor.writeheader()
            escritor.writerows(self._filas)

    @property
    def total_filas(self) -> int:
        return len(self._filas)


# herramienta_csv/cli.py
"""Punto de entrada de la línea de comandos."""
import argparse
import sys
from pathlib import Path
from . import __version__
from .procesador import ProcesadorCSV


def crear_parser() -> argparse.ArgumentParser:
    """Define los argumentos aceptados por la CLI."""
    parser = argparse.ArgumentParser(
        prog="herramienta-csv",
        description="Procesa, filtra y agrega archivos CSV.",
    )
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    parser.add_argument("archivo", type=Path, help="Ruta al archivo CSV de entrada")
    parser.add_argument(
        "--filtrar-columna",
        metavar="COL",
        help="Nombre de columna para filtrar",
    )
    parser.add_argument(
        "--filtrar-valor",
        metavar="VAL",
        help="Valor a mantener en la columna indicada",
    )
    parser.add_argument(
        "--agrupar",
        metavar="COL",
        help="Columna por la que agrupar",
    )
    parser.add_argument(
        "--sumar",
        metavar="COL",
        help="Columna cuyos valores se suman por grupo",
    )
    parser.add_argument(
        "--salida",
        type=Path,
        metavar="RUTA",
        help="Ruta del archivo CSV de salida",
    )
    return parser


def main() -> None:
    """Función principal de la CLI."""
    parser = crear_parser()
    args = parser.parse_args()

    try:
        procesador = ProcesadorCSV(args.archivo)
        print(f"Cargadas {procesador.total_filas} filas de {args.archivo.name}")

        # Aplicar filtro si se especificó
        if args.filtrar_columna and args.filtrar_valor:
            procesador.filtrar(
                lambda fila: fila.get(args.filtrar_columna) == args.filtrar_valor
            )
            print(f"Tras filtrar: {procesador.total_filas} filas")

        # Agrupar y sumar si se especificó
        if args.agrupar and args.sumar:
            totales = procesador.agrupar_y_sumar(args.agrupar, args.sumar)
            print(f"\nTotales por {args.agrupar}:")
            for grupo, total in totales.items():
                print(f"  {grupo:<30} {total:>12.2f}")

        # Exportar resultado si se especificó salida
        if args.salida:
            procesador.exportar_csv(args.salida)
            print(f"\nExportado a: {args.salida}")

    except FileNotFoundError as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)


# pyproject.toml — registrar el comando CLI
# [tool.poetry.scripts]
# herramienta-csv = "herramienta_csv.cli:main"
#
# Uso tras instalar:
# herramienta-csv ventas.csv --filtrar-columna region --filtrar-valor "Madrid"
# herramienta-csv ventas.csv --agrupar categoria --sumar importe --salida resultado.csv
```

---

## Ejercicios propuestos

1. **Paquete de utilidades matemáticas** — Crea un paquete `matutils` con tres módulos:
   `estadistica.py` (media, mediana, desviación estándar), `geometria.py` (área y
   perímetro de figuras básicas) y `conversiones.py` (temperatura, distancia, peso).
   Organiza `__init__.py` para que todo sea accesible con `from matutils import media`.

2. **Migración de pip a Poetry** — Dado un proyecto con `requirements.txt`, escribe
   un script Python que lo lea y genere los comandos `poetry add` equivalentes,
   separando las dependencias de producción (sin marcadores) de las de desarrollo
   (marcadas con `; extra == "dev"`).

3. **Gestor de configuración con .env** — Implementa una clase `Configuracion` que
   cargue todas sus variables desde un archivo `.env`, valide que las requeridas
   existan, convierta automáticamente tipos (`"true"` a `bool`, `"42"` a `int`) y
   genere un `.env.example` con los nombres de las variables pero sin los valores.

4. **CLI de análisis de logs** — Usando `argparse` y la estructura de paquete moderna,
   crea una herramienta `analiza-logs` que lea un archivo de logs línea a línea,
   cuente errores por nivel (`ERROR`, `WARNING`, `INFO`), filtre por rango de fechas
   y exporte un resumen en JSON.

---

## Resumen de la página 7

- Un módulo es un archivo `.py`; un paquete es un directorio con `__init__.py`; las importaciones absolutas son preferibles a las relativas salvo dentro de un paquete propio
- `__all__` controla qué nombres se exportan con `from modulo import *` y documenta el API público del módulo
- `pip freeze > requirements.txt` exporta las versiones exactas; instalar desde `requirements.txt` garantiza reproducibilidad
- `venv` crea un entorno aislado por proyecto; `pyenv` gestiona múltiples versiones de Python en el sistema
- Poetry combina gestión de dependencias, entornos virtuales y empaquetado en una sola herramienta con `pyproject.toml`
- `poetry add --group dev` separa dependencias de desarrollo de las de producción; `poetry install --only main` instala solo producción
- `python-dotenv` carga variables de entorno desde `.env`; nunca subas `.env` a git (añádelo al `.gitignore`)
- La estructura moderna de proyecto separa código fuente, tests y configuración en directorios distintos con `pyproject.toml` como centro de verdad

---

> **Siguiente página →** Página 11: Archivos, JSON, CSV y pathlib.
