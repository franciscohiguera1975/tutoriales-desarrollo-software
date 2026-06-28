# Tutorial Python

Curso completo de Python moderno (3.12+) — desde la instalación hasta arquitectura
de producción con FastAPI, SQLAlchemy y Clean Architecture.
El tutorial sigue la misma dinámica que los cursos de Flutter y Kotlin:
explicaciones directas, código real comentado en español y ejemplos aplicados a
problemas no triviales. Cada bloque de código incluye una sección **Prueba esto**
para reforzar el aprendizaje activo.

---

## Versiones del curso

| Herramienta       | Versión           |
|-------------------|-------------------|
| Python            | **3.12+**         |
| FastAPI           | 0.115+ (standard) |
| SQLAlchemy        | 2.0+              |
| Pydantic          | v2 (2.x)          |
| pydantic-settings | 2.x               |
| pytest            | 8.x               |
| Ruff              | 0.6+              |
| Poetry            | 1.8+              |
| Alembic           | 1.13+             |

---

## Índice

### Fundamentos (páginas 1–14)

| Página | Módulo | Tema |
|--------|--------|------|
| [01](modulo01.md) | Módulo 1 · Entorno | Instalación, tipos, variables, None, f-strings, operadores |
| [02](modulo02.md) | Módulo 1 · Entorno | Condicionales: simple, dos caminos, múltiples y anidadas |
| [03](modulo03.md) | Módulo 1 · Entorno | `match/case` — Structural Pattern Matching (Python 3.10+) |
| [04](modulo04.md) | Módulo 1 · Entorno | Bucles: `for`, `while`, `break`, `continue`, `else` |
| [05](modulo05.md) | Módulo 1 · Entorno | Funciones: definición, parámetros, type hints, retorno |
| [06](modulo06.md) | Módulo 1 · Entorno | Funciones avanzadas: `*args`, `**kwargs`, lambdas, closures, generadores |
| [07](modulo07.md) | Módulo 2 · Colecciones | `list`, `tuple`, `dict`, `set`, comprensiones, `itertools` |
| [08](modulo08.md) | Módulo 3 · POO | Clases, herencia, ABC, métodos dunder, encapsulación |
| [09](modulo09.md) | Módulo 3 · POO | POO avanzada: `@dataclass`, decoradores propios, descriptores, `__slots__` |
| [10](modulo10.md) | Módulo 4 · Ecosistema | Módulos, paquetes, pip, virtualenv, Poetry, `python-dotenv` |
| [11](modulo11.md) | Módulo 4 · Ecosistema | Archivos: `pathlib`, JSON, CSV, TOML, `shutil`, `zipfile` |
| [12](modulo12.md) | Módulo 5 · Robustez | Excepciones, context managers, `logging`, `pdb` |
| [13](modulo13.md) | Módulo 5 · Robustez | Type hints, `mypy`, Pydantic v2, `pydantic-settings` |
| [14](modulo14.md) | Módulo 6 · Concurrencia | GIL, `threading`, `multiprocessing`, `asyncio` |

### HTTP, APIs y Bases de Datos (páginas 15–17)

| Página | Módulo | Tema |
|--------|--------|------|
| [15](modulo15.md) | Módulo 7 · HTTP y APIs | `httpx`, FastAPI fundamentos, routers, `Depends()` |
| [16](modulo16.md) | Módulo 7 · HTTP y APIs | FastAPI avanzado: JWT, middleware, `BackgroundTasks`, lifespan |
| [17](modulo17.md) | Módulo 8 · Bases de datos | SQLAlchemy 2.0, relaciones, Alembic, integración FastAPI |

### Testing, Herramientas y Arquitectura (páginas 18–22)

| Página | Módulo | Tema |
|--------|--------|------|
| [18](modulo18.md) | Módulo 9 · Testing | pytest, fixtures, `mock`, `parametrize`, `pytest-cov` |
| [19](modulo19.md) | Módulo 10 · Herramientas | Ruff, pre-commit, Typer CLI, Rich |
| [20](modulo20.md) | Módulo 11 · Patrones | Design patterns, generadores avanzados, `Protocol` |
| [21](modulo21.md) | Módulo 12 · Arquitectura | Clean Architecture: Domain, Data, Application |
| [22](modulo22.md) | Módulo 12 · Arquitectura | Proyecto integrador: e-commerce completo con tests |

---

## Prerrequisitos

- Conocimientos básicos de programación (variables, bucles, funciones)
- Familiaridad con la terminal (bash/zsh)
- No se requiere experiencia previa con Python

---

## Cómo usar el tutorial

Cada página es autocontenida y referencia a la siguiente.
El código de todos los módulos está comentado en español y usa
Python 3.12+ con la sintaxis más moderna disponible.

Cada bloque de código incluye una sección **Prueba esto** con retos
concretos para reforzar el concepto recién visto.

Los ejemplos aplicados de cada módulo usan APIs reales:

```
GET https://higuera-billing-api.desarrollo-software.xyz/api/products/
    ?page=1&page_size=10
```
