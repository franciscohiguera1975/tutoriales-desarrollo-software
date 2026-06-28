# Tutorial Python — Página 12
## Módulo 5 · Robustez
### Excepciones, logging y debugging profesional

---

## Parte 1 — Jerarquía de excepciones

Python tiene una jerarquía de clases de excepción. `BaseException` es la raíz;
`Exception` es la base de casi todas las excepciones de usuario. Capturar
`Exception` es lo habitual; capturar `BaseException` también atrapa
`KeyboardInterrupt` y `SystemExit`, lo cual casi nunca es lo que se quiere.

```
BaseException
├── SystemExit                  # sys.exit()
├── KeyboardInterrupt           # Ctrl+C
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── OSError (IOError, EnvironmentError)
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── RuntimeError
    │   └── RecursionError
    └── StopIteration
```

---

## Parte 2 — try / except / else / finally

```python
# ─────────────────────────────────────────────────────────────────────────────
# Estructura completa de manejo de excepciones
# ─────────────────────────────────────────────────────────────────────────────
from pathlib import Path

def cargar_configuracion(ruta: str) -> dict:
    """Carga un archivo JSON de configuración con manejo de errores completo."""
    import json

    try:
        # Código que puede lanzar excepciones
        contenido = Path(ruta).read_text(encoding="utf-8")
        config = json.loads(contenido)

    except FileNotFoundError:
        # Capturar un tipo específico de excepción
        print(f"[error] El archivo {ruta!r} no existe")
        return {}

    except json.JSONDecodeError as e:
        # 'as e' da acceso al objeto excepción con detalles
        print(f"[error] JSON inválido en línea {e.lineno}: {e.msg}")
        return {}

    except PermissionError:
        # Capturar otro tipo de error relacionado
        print(f"[error] Sin permisos para leer {ruta!r}")
        return {}

    except (OSError, ValueError) as e:
        # Capturar múltiples tipos en una sola cláusula
        print(f"[error] Error inesperado: {e}")
        return {}

    else:
        # else se ejecuta SOLO si NO hubo excepción en el try
        print(f"Configuración cargada: {len(config)} entradas")
        return config

    finally:
        # finally SIEMPRE se ejecuta, con o sin excepción
        # Ideal para liberar recursos (aunque 'with' es mejor para archivos)
        print("Intento de carga completado.")


# ─────────────────────────────────────────────────────────────────────────────
# raise — lanzar excepciones
# ─────────────────────────────────────────────────────────────────────────────
def validar_edad(edad: int) -> None:
    """Valida que la edad esté en un rango razonable."""
    if not isinstance(edad, int):
        raise TypeError(f"La edad debe ser un entero, recibido: {type(edad).__name__}")
    if edad < 0 or edad > 150:
        raise ValueError(f"Edad fuera de rango: {edad}. Debe estar entre 0 y 150.")


# ─────────────────────────────────────────────────────────────────────────────
# raise from — encadenamiento de excepciones (preserva la causa raíz)
# ─────────────────────────────────────────────────────────────────────────────
def conectar_a_base_datos(url: str) -> object:
    """Envuelve errores técnicos en excepciones de dominio."""
    try:
        # Simular un error de conexión
        if "localhost" not in url:
            raise OSError(f"No se puede conectar a {url}")
        return object()  # en la vida real devolvería una conexión real
    except OSError as causa:
        # raise X from Y: X es la nueva excepción, Y es la causa original
        # Python guardará la causa en excepcion.__cause__
        raise RuntimeError(
            f"No se pudo establecer conexión con la base de datos: {url}"
        ) from causa
```

---

## Parte 3 — Excepciones personalizadas

```python
# ─────────────────────────────────────────────────────────────────────────────
# Jerarquía de excepciones propias de la aplicación
# ─────────────────────────────────────────────────────────────────────────────
class ErrorAplicacion(Exception):
    """Excepción base para todos los errores de la aplicación."""
    pass


class ErrorValidacion(ErrorAplicacion):
    """Error al validar datos de entrada."""

    def __init__(self, campo: str, valor: object, mensaje: str) -> None:
        self.campo = campo
        self.valor = valor
        self.mensaje = mensaje
        super().__init__(f"Validación fallida en '{campo}': {mensaje} (valor: {valor!r})")


class ErrorServicioExterno(ErrorAplicacion):
    """Error al comunicarse con un servicio externo."""

    def __init__(self, servicio: str, codigo_http: int, detalle: str = "") -> None:
        self.servicio = servicio
        self.codigo_http = codigo_http
        self.detalle = detalle
        super().__init__(
            f"Error {codigo_http} en servicio '{servicio}'"
            + (f": {detalle}" if detalle else "")
        )


class ErrorNoEncontrado(ErrorAplicacion):
    """Recurso solicitado no existe."""

    def __init__(self, recurso: str, id_: object) -> None:
        self.recurso = recurso
        self.id_ = id_
        super().__init__(f"{recurso} con id={id_!r} no encontrado")


# Uso
try:
    raise ErrorValidacion("email", "no-es-email", "formato inválido")
except ErrorValidacion as e:
    print(f"Campo: {e.campo}")    # email
    print(f"Valor: {e.valor}")    # no-es-email
    print(str(e))                 # Validación fallida en 'email': formato inválido (valor: 'no-es-email')
```

---

## Parte 4 — Context managers

```python
# ─────────────────────────────────────────────────────────────────────────────
# Implementar __enter__ y __exit__ en una clase
# ─────────────────────────────────────────────────────────────────────────────
import time

class TemporizadorContexto:
    """Context manager que mide el tiempo de un bloque de código."""

    def __init__(self, nombre: str = "bloque") -> None:
        self.nombre = nombre
        self.duracion: float = 0.0

    def __enter__(self) -> "TemporizadorContexto":
        # Se ejecuta al entrar al bloque with
        self._inicio = time.perf_counter()
        return self  # el valor que recibe 'as'

    def __exit__(
        self,
        tipo_exc: type | None,
        valor_exc: Exception | None,
        traza: object,
    ) -> bool:
        # Se ejecuta al salir del bloque with (con o sin excepción)
        self.duracion = time.perf_counter() - self._inicio
        print(f"[tiempo] {self.nombre}: {self.duracion:.4f}s")
        # Devolver True suprima la excepción; False (o None) la propaga
        return False

with TemporizadorContexto("cálculo pesado") as t:
    resultado = sum(range(10_000_000))
print(f"Duración guardada: {t.duracion:.4f}s")


# ─────────────────────────────────────────────────────────────────────────────
# @contextmanager — forma más corta usando un generador
# ─────────────────────────────────────────────────────────────────────────────
from contextlib import contextmanager, suppress
from pathlib import Path
import tempfile

@contextmanager
def archivo_temporal(sufijo: str = ".tmp"):
    """Crea un archivo temporal y lo elimina al terminar."""
    import tempfile
    archivo = tempfile.NamedTemporaryFile(
        suffix=sufijo,
        delete=False,
        mode="w",
        encoding="utf-8",
    )
    ruta = Path(archivo.name)
    try:
        archivo.close()
        yield ruta          # lo que recibe 'as' en el with
    finally:
        # El finally garantiza limpieza incluso si hay excepción
        with suppress(FileNotFoundError):  # suppress ignora excepciones específicas
            ruta.unlink()

with archivo_temporal(".json") as ruta:
    ruta.write_text('{"procesado": true}')
    contenido = ruta.read_text()
    print(f"Archivo temporal: {ruta}")
# Aquí ya no existe el archivo
```

---

## Parte 5 — Logging

```python
import logging
import logging.handlers
import sys
from pathlib import Path

# ─────────────────────────────────────────────────────────────────────────────
# Niveles de logging (de menor a mayor severidad):
# DEBUG    → información detallada para diagnóstico
# INFO     → confirmación de que las cosas van bien
# WARNING  → algo inesperado pero no fatal
# ERROR    → error que impide completar una operación
# CRITICAL → error grave que puede detener la aplicación
# ─────────────────────────────────────────────────────────────────────────────

# ─── Configuración básica (solo para scripts simples) ──────────────────────
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s — %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

logger = logging.getLogger(__name__)   # nombre = módulo actual
logger.debug("Modo depuración activo")
logger.info("Servicio iniciado en el puerto 8080")
logger.warning("Memoria al 85%% de uso")
logger.error("No se pudo conectar a Redis")
logger.critical("Base de datos principal no disponible")


# ─── Configuración avanzada con handlers ───────────────────────────────────
def configurar_logging(
    nivel: str = "INFO",
    archivo_log: Path | None = None,
) -> None:
    """Configura logging con handlers para consola y archivo rotativo."""
    nivel_num = getattr(logging, nivel.upper(), logging.INFO)

    # Formato detallado para archivos
    fmt_archivo = logging.Formatter(
        fmt="%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d — %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S",
    )
    # Formato compacto para consola
    fmt_consola = logging.Formatter(
        fmt="%(asctime)s [%(levelname)s] %(message)s",
        datefmt="%H:%M:%S",
    )

    # Logger raíz
    logger_raiz = logging.getLogger()
    logger_raiz.setLevel(logging.DEBUG)  # nivel mínimo global

    # Handler de consola — solo INFO y superior
    handler_consola = logging.StreamHandler(sys.stdout)
    handler_consola.setLevel(max(nivel_num, logging.INFO))
    handler_consola.setFormatter(fmt_consola)
    logger_raiz.addHandler(handler_consola)

    # Handler de archivo rotativo — rota cuando supera 5 MB, guarda 3 backups
    if archivo_log:
        archivo_log.parent.mkdir(parents=True, exist_ok=True)
        handler_archivo = logging.handlers.RotatingFileHandler(
            filename=archivo_log,
            maxBytes=5 * 1024 * 1024,   # 5 MB
            backupCount=3,
            encoding="utf-8",
        )
        handler_archivo.setLevel(logging.DEBUG)
        handler_archivo.setFormatter(fmt_archivo)
        logger_raiz.addHandler(handler_archivo)
```

### Configuración con dictConfig

```python
# ─────────────────────────────────────────────────────────────────────────────
# dictConfig — forma declarativa de configurar logging
# Equivalente a lo anterior pero más portable y configurable por entorno
# ─────────────────────────────────────────────────────────────────────────────
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,

    "formatters": {
        "detallado": {
            "format": "%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d — %(message)s",
            "datefmt": "%Y-%m-%dT%H:%M:%S",
        },
        "simple": {
            "format": "%(levelname)s — %(message)s",
        },
    },

    "handlers": {
        "consola": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "simple",
            "stream": "ext://sys.stdout",
        },
        "archivo": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "DEBUG",
            "formatter": "detallado",
            "filename": "logs/app.log",
            "maxBytes": 5_242_880,      # 5 MB
            "backupCount": 3,
            "encoding": "utf-8",
        },
    },

    "loggers": {
        "mi_app": {
            "level": "DEBUG",
            "handlers": ["consola", "archivo"],
            "propagate": False,   # no escalar al logger raíz
        },
    },

    "root": {
        "level": "WARNING",
        "handlers": ["consola"],
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
logger = logging.getLogger("mi_app.servicios.auth")
logger.info("Usuario autenticado: ana@ejemplo.com")
```

---

## Parte 6 — Debugging

```python
# ─────────────────────────────────────────────────────────────────────────────
# breakpoint() — depurador interactivo de Python (pdb)
# Disponible desde Python 3.7+ como función built-in
# ─────────────────────────────────────────────────────────────────────────────
def procesar_pedido(pedido: dict) -> float:
    total = 0.0
    for linea in pedido.get("lineas", []):
        # breakpoint()  # ← descomentar para pausar aquí en modo debug
        cantidad = linea["cantidad"]
        precio = linea["precio"]
        total += cantidad * precio
    return total

# Comandos pdb más útiles:
# n  (next)     → siguiente línea
# s  (step)     → entrar a la función
# c  (continue) → continuar hasta el siguiente breakpoint
# p expr        → imprimir valor de expr
# pp expr       → pretty-print
# l  (list)     → mostrar el código alrededor
# w  (where)    → mostrar el stack trace
# q  (quit)     → salir del depurador

# Variable de entorno para cambiar el depurador
# PYTHONBREAKPOINT=0           → deshabilitar breakpoints
# PYTHONBREAKPOINT=ipdb.set_trace → usar ipdb (más potente)
# PYTHONBREAKPOINT=pudb.set_trace  → usar pudb (visual en terminal)
```

---

## Ejemplo aplicado — Sistema de importación con reintentos, logging y reporte de errores

```python
"""
importador_datos.py
Importa registros desde un CSV, valida cada fila, reintenta fallos
de red y genera un reporte detallado de errores con logging estructurado.
"""
from __future__ import annotations

import csv
import json
import logging
import logging.config
import time
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Any


# ── Configuración de logging ─────────────────────────────────────────────────
logging.config.dictConfig({
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "logging.Formatter",
            "fmt": '{"ts":"%(asctime)s","nivel":"%(levelname)s",'
                   '"modulo":"%(name)s","msg":"%(message)s"}',
            "datefmt": "%Y-%m-%dT%H:%M:%S",
        },
        "legible": {
            "format": "%(asctime)s [%(levelname)-8s] %(name)s — %(message)s",
            "datefmt": "%H:%M:%S",
        },
    },
    "handlers": {
        "consola": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "legible",
        },
        "archivo": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "DEBUG",
            "formatter": "json",
            "filename": "logs/importador.log",
            "maxBytes": 10_485_760,   # 10 MB
            "backupCount": 5,
            "encoding": "utf-8",
        },
    },
    "root": {"level": "DEBUG", "handlers": ["consola", "archivo"]},
})

logger = logging.getLogger("importador")


# ── Excepciones de dominio ────────────────────────────────────────────────────
class ErrorImportacion(Exception):
    """Error base del proceso de importación."""


class ErrorValidacionFila(ErrorImportacion):
    """Una fila del CSV no supera la validación."""

    def __init__(self, numero_fila: int, campo: str, detalle: str) -> None:
        self.numero_fila = numero_fila
        self.campo = campo
        self.detalle = detalle
        super().__init__(f"Fila {numero_fila}, campo '{campo}': {detalle}")


class ErrorEnvioRegistro(ErrorImportacion):
    """Fallo al enviar un registro al sistema destino."""


# ── Modelos de datos ──────────────────────────────────────────────────────────
@dataclass
class Registro:
    id_externo: str
    nombre: str
    email: str
    importe: float
    fecha: datetime


@dataclass
class ResultadoImportacion:
    total_leidos: int = 0
    exitosos: int = 0
    fallidos: int = 0
    errores: list[dict[str, Any]] = field(default_factory=list)
    inicio: datetime = field(default_factory=datetime.now)
    fin: datetime | None = None

    @property
    def duracion_segundos(self) -> float:
        if self.fin is None:
            return 0.0
        return (self.fin - self.inicio).total_seconds()

    @property
    def tasa_exito_pct(self) -> float:
        if self.total_leidos == 0:
            return 0.0
        return (self.exitosos / self.total_leidos) * 100


# ── Validación ────────────────────────────────────────────────────────────────
def validar_fila(numero_fila: int, fila: dict[str, str]) -> Registro:
    """Valida y convierte una fila de CSV a un objeto Registro."""
    import re

    # Validar id_externo
    id_externo = fila.get("id", "").strip()
    if not id_externo:
        raise ErrorValidacionFila(numero_fila, "id", "campo vacío")

    # Validar nombre
    nombre = fila.get("nombre", "").strip()
    if len(nombre) < 2:
        raise ErrorValidacionFila(numero_fila, "nombre", "demasiado corto")

    # Validar email
    email = fila.get("email", "").strip().lower()
    if not re.match(r"^[\w.+-]+@[\w-]+\.[a-z]{2,}$", email):
        raise ErrorValidacionFila(numero_fila, "email", f"formato inválido: {email!r}")

    # Validar y convertir importe
    try:
        importe = float(fila.get("importe", "").replace(",", "."))
        if importe < 0:
            raise ValueError("negativo")
    except ValueError:
        raise ErrorValidacionFila(
            numero_fila, "importe", f"no es un número válido: {fila.get('importe')!r}"
        )

    # Validar y convertir fecha
    try:
        fecha = datetime.strptime(fila.get("fecha", ""), "%Y-%m-%d")
    except ValueError:
        raise ErrorValidacionFila(
            numero_fila, "fecha", f"formato incorrecto, se esperaba YYYY-MM-DD"
        )

    return Registro(id_externo, nombre, email, importe, fecha)


# ── Envío con reintentos ──────────────────────────────────────────────────────
def enviar_registro(
    registro: Registro,
    max_reintentos: int = 3,
    espera_base: float = 0.5,
) -> None:
    """Envía un registro al sistema destino con reintentos exponenciales."""
    ultimo_error: Exception | None = None

    for intento in range(1, max_reintentos + 1):
        try:
            # Simulación: en la vida real sería una llamada HTTP
            if hash(registro.id_externo) % 10 == 0:   # 10% de fallos simulados
                raise ConnectionError("Timeout al conectar con API destino")
            logger.debug("Registro enviado: id=%s", registro.id_externo)
            return  # éxito

        except (ConnectionError, TimeoutError) as e:
            ultimo_error = e
            logger.warning(
                "Intento %d/%d fallido para id=%s: %s",
                intento, max_reintentos, registro.id_externo, e,
            )
            if intento < max_reintentos:
                time.sleep(espera_base * (2 ** (intento - 1)))   # backoff exponencial

    raise ErrorEnvioRegistro(
        f"No se pudo enviar id={registro.id_externo} tras {max_reintentos} intentos"
    ) from ultimo_error


# ── Proceso principal ─────────────────────────────────────────────────────────
def importar_csv(ruta: Path) -> ResultadoImportacion:
    """Importa todos los registros de un CSV con validación y reintentos."""
    resultado = ResultadoImportacion()
    logger.info("Iniciando importación desde %s", ruta)

    try:
        with ruta.open(encoding="utf-8", newline="") as f:
            lector = csv.DictReader(f)

            for numero_fila, fila_raw in enumerate(lector, start=2):
                resultado.total_leidos += 1

                try:
                    registro = validar_fila(numero_fila, fila_raw)
                    enviar_registro(registro)
                    resultado.exitosos += 1

                except ErrorValidacionFila as e:
                    resultado.fallidos += 1
                    resultado.errores.append({
                        "fila": e.numero_fila,
                        "tipo": "validacion",
                        "campo": e.campo,
                        "detalle": e.detalle,
                    })
                    logger.warning("Validación fallida fila %d: %s", numero_fila, e)

                except ErrorEnvioRegistro as e:
                    resultado.fallidos += 1
                    resultado.errores.append({
                        "fila": numero_fila,
                        "tipo": "envio",
                        "detalle": str(e),
                    })
                    logger.error("Envío fallido fila %d: %s", numero_fila, e)

    except FileNotFoundError:
        logger.critical("Archivo no encontrado: %s", ruta)
        raise
    finally:
        resultado.fin = datetime.now()

    logger.info(
        "Importación completada: %d/%d exitosos (%.1f%%) en %.2fs",
        resultado.exitosos, resultado.total_leidos,
        resultado.tasa_exito_pct, resultado.duracion_segundos,
    )
    return resultado


def exportar_reporte_errores(resultado: ResultadoImportacion, ruta: Path) -> None:
    """Guarda el reporte de errores en un archivo JSON."""
    ruta.parent.mkdir(parents=True, exist_ok=True)
    reporte = {
        "fecha_ejecucion": resultado.inicio.isoformat(),
        "duracion_segundos": resultado.duracion_segundos,
        "total_leidos": resultado.total_leidos,
        "exitosos": resultado.exitosos,
        "fallidos": resultado.fallidos,
        "tasa_exito_pct": resultado.tasa_exito_pct,
        "errores": resultado.errores,
    }
    ruta.write_text(json.dumps(reporte, indent=2, ensure_ascii=False), encoding="utf-8")
    logger.info("Reporte de errores guardado en %s", ruta)


if __name__ == "__main__":
    resultado = importar_csv(Path("datos/registros.csv"))
    if resultado.errores:
        exportar_reporte_errores(
            resultado,
            Path(f"salida/errores_{datetime.now():%Y%m%d_%H%M}.json"),
        )
```

---

## Ejercicios propuestos

1. **Jerarquía de excepciones de API REST** — Define una jerarquía de excepciones para
   una API REST: `ErrorHTTP` (base, con `codigo` y `detalle`), `Error400` (mala
   petición, con lista de `errores_validacion`), `Error401` (no autenticado), `Error403`
   (sin permisos), `Error404` (no encontrado), `Error503` (servicio no disponible, con
   `tiempo_espera_segundos`). Escribe un handler que las capture y las convierta a
   respuestas JSON adecuadas.

2. **Context manager de transacción** — Implementa un context manager `transaccion(bd)`
   que llame a `bd.begin()` al entrar y a `bd.commit()` al salir sin errores. Si ocurre
   una excepción, llama a `bd.rollback()` y propaga el error. Escribe los tests con
   una clase `BDSimulada` que registre las llamadas.

3. **Rotación manual de logs** — Sin usar `RotatingFileHandler`, implementa una función
   `abrir_log_rotativo(base, max_mb, max_archivos)` que devuelva un context manager.
   Al cerrar, si el archivo supera `max_mb`, renombra los archivos existentes (`.1`,
   `.2`, ...) y elimina los que superan `max_archivos`. Usa `pathlib` y `shutil`.

4. **Decorador @intentar_n_veces con backoff** — Crea un decorador que reintente la
   función con espera exponencial y jitter aleatorio (para evitar tormentas de
   reintentos). Debe aceptar los parámetros `max_intentos`, `espera_base`,
   `multiplicador`, `excepciones` y `en_error` (callback opcional que recibe el
   intento y la excepción). Loguea cada intento con el logger del módulo que lo usa.

---

## Resumen de la página 9

- `BaseException` es la raíz; capturar `Exception` es lo habitual; `BaseException` también captura `SystemExit` y `KeyboardInterrupt`
- `try/except/else/finally`: `else` solo se ejecuta si no hubo excepción; `finally` siempre se ejecuta y es el lugar para liberar recursos
- `raise X from Y` encadena excepciones preservando la causa original; Python la muestra en el traceback con "The above exception was the direct cause of"
- Las excepciones personalizadas deben heredar de `Exception` (o una de sus subclases) y añadir atributos con información de contexto útil para el manejo
- Un context manager implementa `__enter__`/`__exit__`; `@contextmanager` permite escribirlo como un generador con `yield`; `contextlib.suppress` ignora excepciones específicas
- El módulo `logging` tiene cinco niveles (DEBUG, INFO, WARNING, ERROR, CRITICAL); configurar con `dictConfig` es la forma más portable y mantenible
- `RotatingFileHandler` rota los logs por tamaño; `TimedRotatingFileHandler` los rota por tiempo; ambos evitan que los archivos de log crezcan indefinidamente
- `breakpoint()` inicia `pdb`; la variable `PYTHONBREAKPOINT` permite cambiar el depurador sin tocar el código; `ipdb` y `pudb` son alternativas más potentes

---

> **Siguiente página →** Página 13: Type hints, mypy y Pydantic v2.
