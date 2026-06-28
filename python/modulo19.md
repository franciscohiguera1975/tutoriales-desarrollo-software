# Tutorial Python — Página 19
## Módulo 10 · Herramientas del Ecosistema
### Linting, formateo, CLI con Typer y scripting avanzado

---

## El ecosistema de herramientas moderno

El desarrollo Python profesional va más allá del código: necesitas
herramientas que garanticen calidad, consistencia y productividad.
Esta página cubre las herramientas que usan los equipos en 2025.

```
Antes (4 herramientas separadas)    Ahora (todo en uno)
──────────────────────────────────  ──────────────────────
flake8  → linting de estilo         Ruff → linter + formatter
black   → formateo automático       (100–200x más rápido que black)
isort   → ordenar imports
mypy    → type checking (opcional)
```

---

## Ruff — linter y formatter ultrarrápido

Ruff está escrito en Rust. Es tan rápido que en proyectos grandes
la diferencia es de segundos a milisegundos.

```bash
# Instalar Ruff
pip install ruff

# O como herramienta de desarrollo
pip install --group dev ruff   # con pip 24+
```

### Uso básico

```bash
# Verificar el código (linting) — muestra errores
ruff check .

# Corregir automáticamente los errores que puede corregir
ruff check --fix .

# Formatear el código (equivale a black)
ruff format .

# Verificar sin modificar (útil en CI)
ruff format --check .

# Verificar un archivo específico
ruff check src/mi_modulo.py

# Ver qué reglas están activas
ruff rule E501
```

### Configuración en pyproject.toml

```toml
# pyproject.toml

[tool.ruff]
# Versión mínima de Python objetivo
target-version = "py312"

# Longitud máxima de línea (defecto: 88 como black)
line-length = 88

# Directorios a excluir
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "migrations",
    "*.pyi",
]

[tool.ruff.lint]
# Conjuntos de reglas a activar:
# E, W  = pycodestyle (errores de estilo)
# F     = pyflakes (errores lógicos: imports no usados, variables indefinidas)
# I     = isort (ordenar imports)
# N     = pep8-naming (convenciones de nombres)
# UP    = pyupgrade (modernizar código a Python 3.12)
# B     = flake8-bugbear (patrones problemáticos)
# SIM   = flake8-simplify (simplificar código)
# TCH   = flake8-type-checking (mover imports a TYPE_CHECKING)
select = ["E", "W", "F", "I", "N", "UP", "B", "SIM"]

# Reglas a ignorar
ignore = [
    "E501",  # línea muy larga — gestionado por el formatter
    "B008",  # no llamar funciones en argumentos por defecto (FastAPI lo usa)
]

# Permitir imports no usados en __init__.py
per-file-ignores = { "__init__.py" = ["F401"] }

# Correcciones automáticas habilitadas
fixable = ["ALL"]

[tool.ruff.lint.isort]
# Secciones de imports: stdlib, third-party, first-party
known-first-party = ["app", "src"]
force-sort-within-sections = true

[tool.ruff.format]
# Estilo de comillas (como black: dobles por defecto)
quote-style = "double"
# Indentar con espacios (no tabs)
indent-style = "space"
# Coma al final en colecciones multilínea
skip-magic-trailing-comma = false
```

### Qué corrige Ruff automáticamente

```python
# ── ANTES (código malo que Ruff detecta y corrige) ───────────────

import os, sys                      # E401: múltiples imports en una línea
import json
import os                           # F811: import duplicado

x = list()                          # C408: usar [] en vez de list()
y = dict()                          # C408: usar {} en vez de dict()

if x == True:                       # E712: comparar con True/False
    pass

if x == None:                       # E711: comparar con None (usar 'is')
    pass

nombre = "Juan"
edad    = 30                        # E221: alineación de espacios extra

def funcion( a, b ):               # E201/E202: espacios innecesarios
    return( a + b )                 # E211: espacio antes de paréntesis


# ── DESPUÉS (código corregido por ruff check --fix + ruff format) ─

import os
import sys
import json

x = []
y = {}

if x is True:
    pass

if x is None:
    pass

nombre = "Juan"
edad = 30


def funcion(a, b):
    return a + b
```

---

## pre-commit — hooks automáticos

`pre-commit` ejecuta verificaciones automáticamente antes de cada
`git commit`. Si alguna verificación falla, el commit no ocurre.

```bash
# Instalar pre-commit
pip install pre-commit

# Instalar los hooks en el repositorio
pre-commit install

# Ejecutar manualmente sobre todos los archivos
pre-commit run --all-files

# Actualizar las versiones de los hooks
pre-commit autoupdate
```

### Configuración .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml

repos:
  # ── Ruff: linting y formateo ──────────────────────────────────
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      # Linter — corrige automáticamente lo que puede
      - id: ruff
        args: [--fix]
      # Formatter
      - id: ruff-format

  # ── Verificaciones generales ──────────────────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      # No commitear claves privadas o secrets
      - id: detect-private-key
      # No commitear archivos con conflictos de merge sin resolver
      - id: check-merge-conflict
      # Archivos YAML válidos
      - id: check-yaml
      # Archivos TOML válidos
      - id: check-toml
      # Eliminar espacios al final de línea
      - id: trailing-whitespace
      # Asegurar newline al final del archivo
      - id: end-of-file-fixer
      # No commitear archivos grandes accidentalmente
      - id: check-added-large-files
        args: ["--maxkb=500"]

  # ── mypy: type checking (opcional pero recomendado) ──────────
  # - repo: https://github.com/pre-commit/mirrors-mypy
  #   rev: v1.10.0
  #   hooks:
  #     - id: mypy
  #       additional_dependencies: [pydantic, fastapi]
```

```bash
# Al hacer git commit, se ejecutan automáticamente:
git add src/mi_archivo.py
git commit -m "feat: agregar endpoint de usuarios"

# Si Ruff encuentra un error:
# ruff.....................................................................Failed
# - hook id: ruff
# - exit code: 1
# src/mi_archivo.py:5:1: F401 'os' imported but unused

# Corregir y volver a commitear
git add src/mi_archivo.py
git commit -m "feat: agregar endpoint de usuarios"
# ruff.....................................................................Passed
# ruff-format..............................................................Passed
# [main abc1234] feat: agregar endpoint de usuarios
```

---

## Typer — CLIs modernas con type hints

Typer convierte funciones Python en comandos CLI usando las anotaciones
de tipo como fuente de verdad. Cero boilerplate, autocompletado incluido.

```bash
pip install typer[all]  # incluye rich para colores y tabla de ayuda
```

### Comandos básicos

```python
# cli_basico.py
import typer
from typing import Annotated

# Crear la aplicación CLI
app = typer.Typer(
    name="gestor",
    help="Herramienta de gestión de proyectos",
    add_completion=True,  # habilita autocompletado en bash/zsh
)


# Argumento posicional — requerido
@app.command()
def saludar(
    nombre: Annotated[str, typer.Argument(help="Nombre de la persona")],
    veces: Annotated[int, typer.Option(help="Cuántas veces saludar")] = 1,
    mayusculas: Annotated[bool, typer.Option("--mayusculas/--no-mayusculas",
                          help="Saludar en mayúsculas")] = False,
) -> None:
    """Saludar a alguien desde la terminal."""
    mensaje = f"Hola, {nombre}!"
    if mayusculas:
        mensaje = mensaje.upper()
    for _ in range(veces):
        typer.echo(mensaje)


# Opción con valores por defecto
@app.command()
def info(
    detallado: Annotated[bool, typer.Option("--detallado", "-d",
               help="Mostrar información detallada")] = False,
) -> None:
    """Mostrar información del sistema."""
    typer.echo("Gestor de Proyectos v1.0")
    if detallado:
        import platform
        typer.echo(f"Python: {platform.python_version()}")
        typer.echo(f"Sistema: {platform.system()}")


if __name__ == "__main__":
    app()
```

```bash
# Uso:
python cli_basico.py saludar Ana
# Hola, Ana!

python cli_basico.py saludar Ana --veces 3 --mayusculas
# HOLA, ANA!
# HOLA, ANA!
# HOLA, ANA!

python cli_basico.py --help
python cli_basico.py saludar --help
```

### Subcomandos — grupos de aplicación

```python
# cli_subcomandos.py
import typer
from typing import Annotated

app = typer.Typer(help="CLI de gestión de proyectos")

# Subgrupo de comandos para tareas
tareas_app = typer.Typer(help="Gestionar tareas")
app.add_typer(tareas_app, name="tareas")

# Subgrupo para proyectos
proyectos_app = typer.Typer(help="Gestionar proyectos")
app.add_typer(proyectos_app, name="proyectos")


@tareas_app.command("listar")
def tareas_listar(
    estado: Annotated[str, typer.Option(
        help="Filtrar por estado: pendiente, completada, todas"
    )] = "todas",
) -> None:
    """Listar todas las tareas."""
    typer.echo(f"Listando tareas con estado: {estado}")


@tareas_app.command("crear")
def tareas_crear(
    titulo: Annotated[str, typer.Argument(help="Título de la tarea")],
    prioridad: Annotated[str, typer.Option(
        help="Prioridad: baja, media, alta"
    )] = "media",
) -> None:
    """Crear una nueva tarea."""
    typer.echo(f"Creando tarea: '{titulo}' con prioridad {prioridad}")


@proyectos_app.command("nuevo")
def proyectos_nuevo(
    nombre: Annotated[str, typer.Argument(help="Nombre del proyecto")],
) -> None:
    """Crear un nuevo proyecto."""
    typer.echo(f"Proyecto '{nombre}' creado")


if __name__ == "__main__":
    app()
```

```bash
python cli_subcomandos.py tareas listar --estado pendiente
python cli_subcomandos.py tareas crear "Revisar pull requests" --prioridad alta
python cli_subcomandos.py proyectos nuevo "API Rest"
python cli_subcomandos.py --help         # ayuda principal
python cli_subcomandos.py tareas --help  # ayuda del subgrupo
```

### Colores y confirmaciones con Typer + Rich

```python
# cli_colores.py
import typer
from rich.console import Console
from rich.table import Table
from rich import print as rprint

app = typer.Typer()
console = Console()


@app.command()
def borrar(
    nombre: Annotated[str, typer.Argument(help="Nombre del recurso")],
    forzar: Annotated[bool, typer.Option(
        "--forzar", "-f", help="No pedir confirmación"
    )] = False,
) -> None:
    """Borrar un recurso con confirmación."""
    if not forzar:
        # Pedir confirmación al usuario
        confirmar = typer.confirm(
            f"¿Seguro que quieres borrar '{nombre}'?",
            default=False,
        )
        if not confirmar:
            typer.echo("Operación cancelada.")
            raise typer.Exit(0)

    # Mostrar mensaje con color
    typer.secho(f"Borrando '{nombre}'...", fg=typer.colors.YELLOW)
    # ... lógica de borrado ...
    typer.secho(f"'{nombre}' eliminado.", fg=typer.colors.GREEN, bold=True)


@app.command()
def listar_proyectos() -> None:
    """Listar proyectos en una tabla Rich."""
    # Crear tabla con Rich
    tabla = Table(title="Proyectos activos", show_header=True)
    tabla.add_column("ID",       style="cyan",  no_wrap=True)
    tabla.add_column("Nombre",   style="bold")
    tabla.add_column("Estado",   style="green")
    tabla.add_column("Tareas",   justify="right")

    # Datos de ejemplo
    proyectos = [
        ("1", "API Rest",       "Activo",  "12"),
        ("2", "Frontend React", "Pausado", "8"),
        ("3", "Migración BD",   "Activo",  "5"),
    ]
    for proyecto in proyectos:
        tabla.add_row(*proyecto)

    console.print(tabla)


if __name__ == "__main__":
    app()
```

---

## Rich — terminal como interfaz profesional

Rich transforma la salida de texto plano en interfaces visuales ricas.

```python
# demos_rich.py
from rich.console import Console
from rich.table import Table
from rich.progress import Progress, SpinnerColumn, TextColumn
from rich.panel import Panel
from rich.markdown import Markdown
from rich.syntax import Syntax
from rich.tree import Tree
from rich import print as rprint
import time

console = Console()


# ── Texto con formato ─────────────────────────────────────────────

def demo_texto():
    # Rich usa markup similar a BBCode
    console.print("[bold green]Éxito:[/bold green] Operación completada")
    console.print("[red]Error:[/red] Algo salió mal")
    console.print("[yellow]Advertencia:[/yellow] Revisa la configuración")
    console.print("[bold blue]Información[/bold blue]: conectado al servidor")

    # Estilos en línea
    rprint("El precio es [bold]$99.99[/bold] [green](descuento aplicado)[/green]")


# ── Tablas ────────────────────────────────────────────────────────

def demo_tabla():
    tabla = Table(title="Reporte de ventas", show_footer=True)

    tabla.add_column("Producto",  style="cyan",    footer="Total")
    tabla.add_column("Cantidad",  justify="right", footer="15")
    tabla.add_column("Precio",    justify="right")
    tabla.add_column("Subtotal",  justify="right", style="bold green",
                     footer="[bold]$1,250.00[/bold]")

    tabla.add_row("Teclado mecánico", "3",  "$89.99",  "$269.97")
    tabla.add_row("Monitor 27\"",     "2",  "$349.99", "$699.98")
    tabla.add_row("Mouse ergonómico", "10", "$28.05",  "$280.50")

    console.print(tabla)


# ── Progress bars ─────────────────────────────────────────────────

def demo_progress():
    # Progress bar simple
    with Progress() as progress:
        tarea = progress.add_task("[cyan]Procesando archivos...", total=100)
        while not progress.finished:
            progress.update(tarea, advance=10)
            time.sleep(0.1)

    # Progress bar con spinner personalizado
    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        transient=True,  # desaparece al terminar
    ) as progress:
        t1 = progress.add_task("Conectando a la API...", total=None)
        time.sleep(1)
        progress.update(t1, description="Descargando datos...")
        time.sleep(1)
        progress.update(t1, description="Procesando respuesta...")
        time.sleep(0.5)


# ── Paneles y markdown ────────────────────────────────────────────

def demo_paneles():
    # Panel con borde
    console.print(Panel(
        "[bold]Bienvenido al sistema de gestión[/bold]\n"
        "Versión 2.0 · Python 3.12 · FastAPI",
        title="[blue]Gestor de Proyectos[/blue]",
        subtitle="[green]Conectado[/green]",
        border_style="blue",
    ))

    # Markdown en terminal
    md = Markdown("""
# Instrucciones

1. Ejecuta `gestor init` para inicializar
2. Usa `gestor tareas listar` para ver tus tareas
3. Consulta `gestor --help` para más opciones

> **Nota**: Necesitas Python 3.12 o superior.
    """)
    console.print(md)


# ── Syntax highlighting ───────────────────────────────────────────

def demo_codigo():
    codigo = '''
def obtener_tareas(db: Session) -> list[Tarea]:
    """Obtener todas las tareas de la base de datos."""
    return db.query(Tarea).filter(Tarea.activa == True).all()
    '''
    syntax = Syntax(codigo, "python", theme="monokai", line_numbers=True)
    console.print(Panel(syntax, title="Código generado"))


# ── Árbol de directorios ──────────────────────────────────────────

def demo_arbol():
    arbol = Tree("[bold blue]mi_proyecto/[/bold blue]")

    src = arbol.add("[bold]src/[/bold]")
    src.add("main.py")
    src.add("models.py")
    api = src.add("[bold]api/[/bold]")
    api.add("routers.py")
    api.add("schemas.py")

    tests = arbol.add("[bold]tests/[/bold]")
    tests.add("test_api.py")
    tests.add("conftest.py")

    arbol.add("pyproject.toml")
    arbol.add("README.md")

    console.print(arbol)


# ── Logging con Rich ──────────────────────────────────────────────

def demo_logging():
    import logging
    from rich.logging import RichHandler

    # Configurar logging con Rich
    logging.basicConfig(
        level=logging.DEBUG,
        format="%(message)s",
        datefmt="[%X]",
        handlers=[RichHandler(rich_tracebacks=True)],
    )
    log = logging.getLogger("rich")

    log.debug("Mensaje de debug")
    log.info("Servidor iniciado en :8000")
    log.warning("Memoria al 80% de capacidad")
    log.error("No se pudo conectar a la base de datos")
```

---

## subprocess — ejecutar comandos del sistema

```python
# subprocess_demo.py
import subprocess
import sys


# ── Ejecutar un comando y capturar la salida ──────────────────────

def ejecutar_comando(comando: list[str]) -> tuple[int, str, str]:
    """
    Ejecuta un comando y retorna (código_salida, stdout, stderr).
    Seguro: usa lista de strings, no shell=True.
    """
    resultado = subprocess.run(
        comando,
        capture_output=True,   # captura stdout y stderr
        text=True,             # decodifica bytes a string
        check=False,           # no lanza excepción si falla
    )
    return resultado.returncode, resultado.stdout, resultado.stderr


def verificar_git_instalado() -> bool:
    """Verificar si git está disponible en el sistema."""
    codigo, stdout, _ = ejecutar_comando(["git", "--version"])
    return codigo == 0


def obtener_rama_actual() -> str | None:
    """Obtener el nombre de la rama actual de git."""
    codigo, stdout, _ = ejecutar_comando(
        ["git", "rev-parse", "--abbrev-ref", "HEAD"]
    )
    if codigo == 0:
        return stdout.strip()
    return None


def ejecutar_tests(directorio: str = ".") -> bool:
    """Ejecutar pytest y retornar True si todos los tests pasan."""
    # subprocess.run con check=True lanza excepción si falla
    try:
        subprocess.run(
            [sys.executable, "-m", "pytest", directorio, "-v"],
            check=True,
        )
        return True
    except subprocess.CalledProcessError:
        return False


# ── Ejecutar con timeout ──────────────────────────────────────────

def ping(host: str, timeout_seg: int = 5) -> bool:
    """Verificar conectividad con un host usando ping."""
    import platform
    flag = "-n" if platform.system() == "Windows" else "-c"
    try:
        resultado = subprocess.run(
            ["ping", flag, "1", host],
            capture_output=True,
            timeout=timeout_seg,  # terminar si tarda más de N segundos
        )
        return resultado.returncode == 0
    except subprocess.TimeoutExpired:
        return False


# ── Ejemplo de uso ────────────────────────────────────────────────

if __name__ == "__main__":
    if verificar_git_instalado():
        rama = obtener_rama_actual()
        print(f"Rama actual: {rama}")
    else:
        print("git no encontrado")
```

---

## Ejemplo aplicado — CLI de gestión de proyectos

CLI completa con Typer + Rich: listar, crear, actualizar, eliminar y exportar.

```python
# gestor_proyectos/cli.py
"""
CLI de gestión de proyectos con Typer + Rich.
Persistencia en JSON local. Exportación a CSV y JSON.
"""
import json
import csv
import typer
from pathlib import Path
from datetime import datetime
from typing import Annotated
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich import print as rprint

# ── Configuración ─────────────────────────────────────────────────

APP_DIR  = Path.home() / ".gestor_proyectos"
DB_FILE  = APP_DIR / "proyectos.json"

app     = typer.Typer(name="gestor", help="Gestión de proyectos desde la terminal")
console = Console()


# ── Tipos de datos ────────────────────────────────────────────────

ESTADOS_VALIDOS = ["activo", "pausado", "completado", "cancelado"]

COLORES_ESTADO = {
    "activo":     "green",
    "pausado":    "yellow",
    "completado": "blue",
    "cancelado":  "red",
}


# ── Persistencia ──────────────────────────────────────────────────

def cargar_proyectos() -> list[dict]:
    """Cargar proyectos desde el archivo JSON."""
    APP_DIR.mkdir(parents=True, exist_ok=True)
    if not DB_FILE.exists():
        return []
    with open(DB_FILE) as f:
        return json.load(f)


def guardar_proyectos(proyectos: list[dict]) -> None:
    """Guardar proyectos en el archivo JSON."""
    APP_DIR.mkdir(parents=True, exist_ok=True)
    with open(DB_FILE, "w") as f:
        json.dump(proyectos, f, indent=2, ensure_ascii=False)


def siguiente_id(proyectos: list[dict]) -> int:
    """Obtener el siguiente ID disponible."""
    if not proyectos:
        return 1
    return max(p["id"] for p in proyectos) + 1


# ── Comandos ──────────────────────────────────────────────────────

@app.command("listar")
def listar_proyectos(
    estado: Annotated[str | None, typer.Option(
        help="Filtrar por estado: activo, pausado, completado, cancelado"
    )] = None,
    json_output: Annotated[bool, typer.Option(
        "--json", help="Salida en formato JSON"
    )] = False,
) -> None:
    """Listar todos los proyectos."""
    proyectos = cargar_proyectos()

    # Filtrar por estado si se especificó
    if estado:
        if estado not in ESTADOS_VALIDOS:
            rprint(f"[red]Estado inválido:[/red] {estado}")
            rprint(f"Estados válidos: {', '.join(ESTADOS_VALIDOS)}")
            raise typer.Exit(1)
        proyectos = [p for p in proyectos if p["estado"] == estado]

    if not proyectos:
        console.print(Panel(
            "[yellow]No hay proyectos que mostrar[/yellow]",
            title="Gestor de Proyectos",
        ))
        return

    if json_output:
        # Salida JSON para pipelines
        typer.echo(json.dumps(proyectos, indent=2, ensure_ascii=False))
        return

    # Tabla Rich
    tabla = Table(title=f"Proyectos ({len(proyectos)})", show_footer=False)
    tabla.add_column("ID",       style="cyan",  width=5)
    tabla.add_column("Nombre",   style="bold",  min_width=20)
    tabla.add_column("Estado",   min_width=12)
    tabla.add_column("Tareas",   justify="right", width=8)
    tabla.add_column("Creado",   width=12)

    for p in proyectos:
        color  = COLORES_ESTADO.get(p["estado"], "white")
        estado_texto = f"[{color}]{p['estado']}[/{color}]"
        tabla.add_row(
            str(p["id"]),
            p["nombre"],
            estado_texto,
            str(p.get("num_tareas", 0)),
            p.get("creado", "")[:10],
        )

    console.print(tabla)


@app.command("crear")
def crear_proyecto(
    nombre: Annotated[str, typer.Argument(help="Nombre del proyecto")],
    descripcion: Annotated[str, typer.Option(
        "--desc", "-d", help="Descripción del proyecto"
    )] = "",
    estado: Annotated[str, typer.Option(
        help="Estado inicial del proyecto"
    )] = "activo",
) -> None:
    """Crear un nuevo proyecto."""
    if estado not in ESTADOS_VALIDOS:
        rprint(f"[red]Estado inválido:[/red] {estado}")
        raise typer.Exit(1)

    proyectos = cargar_proyectos()

    # Verificar nombre duplicado
    if any(p["nombre"].lower() == nombre.lower() for p in proyectos):
        rprint(f"[red]Ya existe un proyecto con el nombre:[/red] '{nombre}'")
        raise typer.Exit(1)

    nuevo = {
        "id":          siguiente_id(proyectos),
        "nombre":      nombre,
        "descripcion": descripcion,
        "estado":      estado,
        "num_tareas":  0,
        "creado":      datetime.now().isoformat(),
    }
    proyectos.append(nuevo)
    guardar_proyectos(proyectos)

    rprint(f"[green]Proyecto creado:[/green] [bold]{nombre}[/bold] (ID: {nuevo['id']})")


@app.command("actualizar")
def actualizar_proyecto(
    id_proyecto: Annotated[int, typer.Argument(help="ID del proyecto")],
    nombre: Annotated[str | None, typer.Option(
        "--nombre", "-n", help="Nuevo nombre"
    )] = None,
    estado: Annotated[str | None, typer.Option(
        "--estado", "-e", help="Nuevo estado"
    )] = None,
    descripcion: Annotated[str | None, typer.Option(
        "--desc", "-d", help="Nueva descripción"
    )] = None,
) -> None:
    """Actualizar un proyecto existente."""
    proyectos = cargar_proyectos()

    # Buscar el proyecto
    proyecto = next((p for p in proyectos if p["id"] == id_proyecto), None)
    if not proyecto:
        rprint(f"[red]Proyecto no encontrado:[/red] ID {id_proyecto}")
        raise typer.Exit(1)

    # Aplicar cambios
    if nombre:
        proyecto["nombre"] = nombre
    if estado:
        if estado not in ESTADOS_VALIDOS:
            rprint(f"[red]Estado inválido:[/red] {estado}")
            raise typer.Exit(1)
        proyecto["estado"] = estado
    if descripcion:
        proyecto["descripcion"] = descripcion

    guardar_proyectos(proyectos)
    rprint(f"[green]Proyecto actualizado:[/green] ID {id_proyecto}")


@app.command("eliminar")
def eliminar_proyecto(
    id_proyecto: Annotated[int, typer.Argument(help="ID del proyecto")],
    forzar: Annotated[bool, typer.Option(
        "--forzar", "-f", help="No pedir confirmación"
    )] = False,
) -> None:
    """Eliminar un proyecto."""
    proyectos = cargar_proyectos()

    proyecto = next((p for p in proyectos if p["id"] == id_proyecto), None)
    if not proyecto:
        rprint(f"[red]Proyecto no encontrado:[/red] ID {id_proyecto}")
        raise typer.Exit(1)

    if not forzar:
        confirmar = typer.confirm(
            f"¿Eliminar el proyecto '{proyecto['nombre']}'?",
            default=False,
        )
        if not confirmar:
            typer.echo("Cancelado.")
            raise typer.Exit(0)

    proyectos = [p for p in proyectos if p["id"] != id_proyecto]
    guardar_proyectos(proyectos)
    rprint(f"[green]Proyecto eliminado:[/green] '{proyecto['nombre']}'")


@app.command("exportar")
def exportar_proyectos(
    formato: Annotated[str, typer.Option(
        help="Formato de exportación: csv o json"
    )] = "csv",
    salida: Annotated[str, typer.Option(
        "--salida", "-o", help="Archivo de salida"
    )] = "",
) -> None:
    """Exportar proyectos a CSV o JSON."""
    proyectos = cargar_proyectos()
    if not proyectos:
        rprint("[yellow]No hay proyectos para exportar[/yellow]")
        return

    # Nombre de archivo por defecto
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    nombre_archivo = salida or f"proyectos_{timestamp}.{formato}"

    if formato == "json":
        with open(nombre_archivo, "w") as f:
            json.dump(proyectos, f, indent=2, ensure_ascii=False)

    elif formato == "csv":
        campos = ["id", "nombre", "descripcion", "estado",
                  "num_tareas", "creado"]
        with open(nombre_archivo, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=campos,
                                    extrasaction="ignore")
            writer.writeheader()
            writer.writerows(proyectos)
    else:
        rprint(f"[red]Formato no soportado:[/red] {formato}")
        raise typer.Exit(1)

    rprint(f"[green]Exportado:[/green] {nombre_archivo} "
           f"([bold]{len(proyectos)}[/bold] proyectos)")


# Callback para mostrar versión
@app.callback()
def main(
    version: Annotated[bool, typer.Option(
        "--version", "-v", help="Mostrar versión", is_eager=True
    )] = False,
) -> None:
    """Gestor de Proyectos — CLI profesional con Typer + Rich."""
    if version:
        rprint("[bold]Gestor de Proyectos[/bold] v1.0.0")
        raise typer.Exit(0)


if __name__ == "__main__":
    app()
```

```bash
# Uso completo de la CLI:
python -m gestor_proyectos.cli crear "API Rest" --desc "Backend con FastAPI"
python -m gestor_proyectos.cli crear "Dashboard" --estado pausado
python -m gestor_proyectos.cli listar
python -m gestor_proyectos.cli listar --estado activo
python -m gestor_proyectos.cli actualizar 1 --estado completado
python -m gestor_proyectos.cli exportar --formato json --salida backup.json
python -m gestor_proyectos.cli exportar --formato csv
python -m gestor_proyectos.cli eliminar 2
python -m gestor_proyectos.cli --version
```

---

## Ejercicios propuestos

1. **CLI de tareas con prioridades** — Extiende el ejemplo de CLI para gestionar tareas dentro de cada proyecto. Agrega comandos `tareas agregar <id_proyecto> <titulo>`, `tareas completar <id_tarea>` y `tareas listar <id_proyecto>`. Usa `@app.add_typer` para crear un subgrupo `tareas` dentro del CLI principal. Muestra las tareas con una tabla Rich con columnas para ID, título, prioridad y estado (con color según prioridad).

2. **Hook de pre-commit personalizado** — Crea un script Python `scripts/verificar_docstrings.py` que recorra todos los archivos `.py` del proyecto y verifique que todas las funciones públicas (sin `_` al inicio) tienen docstring. Haz que retorne código de salida 1 si encuentra funciones sin docstring y liste cuáles son. Configura este script como un hook local en `.pre-commit-config.yaml` que se ejecute en cada commit.

3. **Reporte de coverage en terminal** — Crea un comando Typer `cobertura` que ejecute `pytest --cov=app --cov-report=json` usando `subprocess.run`, lea el archivo `coverage.json` generado y muestre los resultados en una tabla Rich con columnas: Módulo, Statements, Miss, Covered%. Resalta en rojo los módulos con cobertura menor al 80% y en verde los que están sobre el 90%.

4. **Configuración de Ruff con reglas personalizadas** — Configura Ruff en `pyproject.toml` para un proyecto FastAPI que: (a) active las reglas `UP` para modernizar el código a Python 3.12 (`match` en vez de `if/elif`, `X | Y` en vez de `Optional[X]`, etc.); (b) ignore `B008` para permitir `Depends()` en los parámetros; (c) configure `per-file-ignores` para permitir imports no usados en `__init__.py`. Toma código legacy con `Optional[str]`, `Union[int, str]` y `Dict[str, Any]` y verifica que Ruff propone las correcciones correctas.

---

## Resumen de la página 16

- **Ruff** reemplaza flake8, black e isort en una sola herramienta escrita en Rust — 100x más rápido. `ruff check --fix` corrige automáticamente; `ruff format` formatea como black.
- La configuración de Ruff en `pyproject.toml` bajo `[tool.ruff.lint]` y `[tool.ruff.format]` permite activar conjuntos de reglas como `UP` (modernización), `B` (bug-bear) y `I` (isort).
- **pre-commit** instala hooks con `pre-commit install` y los ejecuta automáticamente en cada `git commit`. Si un hook falla, el commit no ocurre — garantiza código limpio en el repositorio.
- **Typer** convierte funciones Python en comandos CLI usando las anotaciones de tipo: `Argument` para posicionales, `Option` para flags con `--nombre`. `Annotated[tipo, typer.Option(...)]` es la forma moderna de declarar parámetros.
- Los **subcomandos** con `typer.Typer()` y `app.add_typer(sub_app, name="subcomando")` permiten crear CLIs jerárquicas como `git commit`, `git push`.
- **Rich** transforma la salida de texto plano: `Table` para datos tabulares, `Progress` para barras de progreso, `Panel` para destacar información, `Syntax` para código con colores, `RichHandler` para logging visual.
- `subprocess.run(["comando", "arg"], capture_output=True, text=True)` es la forma segura de ejecutar comandos del sistema — nunca uses `shell=True` con input no confiable.
- La combinación Typer + Rich + Ruff + pre-commit es el stack estándar para herramientas internas de equipo y automatización de tareas.

---

> **Siguiente página →** Página 20: Patrones de diseño, generadores avanzados y Protocol.
