# Tutorial Python — Página 11
## Módulo 4 · Ecosistema Python
### Archivos, JSON, CSV y pathlib

---

## Parte 1 — pathlib

`pathlib.Path` es la forma moderna y orientada a objetos de trabajar con rutas
de archivos. Es multiplataforma y más legible que `os.path`.

```python
from pathlib import Path

# ─────────────────────────────────────────────────────────────────────────────
# Crear y manipular rutas
# ─────────────────────────────────────────────────────────────────────────────
# Rutas relativas y absolutas
raiz = Path(".")                       # directorio actual (relativo)
home = Path.home()                     # /home/usuario
cwd = Path.cwd()                       # directorio de trabajo actual (absoluto)

# Construir rutas con el operador /
config_dir = home / ".config" / "mi_app"
archivo_log = config_dir / "app.log"

# Propiedades de una ruta
ruta = Path("/datos/proyecto/src/main.py")
print(ruta.name)          # main.py
print(ruta.stem)          # main
print(ruta.suffix)        # .py
print(ruta.suffixes)      # ['.py']
print(ruta.parent)        # /datos/proyecto/src
print(ruta.parts)         # ('/', 'datos', 'proyecto', 'src', 'main.py')

# Ruta de un archivo tar.gz
arch = Path("backup.tar.gz")
print(arch.suffixes)      # ['.tar', '.gz']
print(arch.stem)          # backup.tar


# ─────────────────────────────────────────────────────────────────────────────
# Verificar existencia y tipo
# ─────────────────────────────────────────────────────────────────────────────
ruta = Path("/datos/proyecto")

print(ruta.exists())      # True si existe
print(ruta.is_file())     # True si es un archivo
print(ruta.is_dir())      # True si es un directorio
print(ruta.is_symlink())  # True si es un enlace simbólico


# ─────────────────────────────────────────────────────────────────────────────
# Crear directorios y archivos
# ─────────────────────────────────────────────────────────────────────────────
# Crear directorio (y padres si no existen)
directorio = Path("salida/reportes/2024")
directorio.mkdir(parents=True, exist_ok=True)  # exist_ok evita error si ya existe

# Crear o actualizar un archivo (equivale a "touch")
archivo = Path("salida/ultimo_proceso.txt")
archivo.touch()

# Escribir texto directamente en un archivo
Path("salida/version.txt").write_text("1.0.0\n", encoding="utf-8")

# Leer texto directamente
contenido = Path("salida/version.txt").read_text(encoding="utf-8")
print(contenido.strip())   # 1.0.0


# ─────────────────────────────────────────────────────────────────────────────
# Listar y buscar archivos con glob
# ─────────────────────────────────────────────────────────────────────────────
directorio = Path(".")

# Listar hijos directos (no recursivo)
for hijo in directorio.iterdir():
    tipo = "DIR " if hijo.is_dir() else "FILE"
    print(f"[{tipo}] {hijo.name}")

# Buscar archivos con patrón (no recursivo)
for py in directorio.glob("*.py"):
    print(py)

# Buscar recursivamente — ** significa "cualquier subdirectorio"
for py in directorio.rglob("*.py"):
    print(py)

# Buscar todos los logs en subdirectorios
logs = list(Path("logs").rglob("*.log"))
print(f"Encontrados {len(logs)} archivos de log")
```

---

## Parte 2 — Leer y escribir archivos

```python
from pathlib import Path

# ─────────────────────────────────────────────────────────────────────────────
# Modos de apertura
# 'r'  → leer texto (default)
# 'w'  → escribir texto (sobreescribe)
# 'a'  → añadir al final (append)
# 'x'  → crear archivo nuevo, error si ya existe
# 'rb' → leer binario
# 'wb' → escribir binario
# ─────────────────────────────────────────────────────────────────────────────

# Lectura completa — adecuada para archivos pequeños
ruta = Path("datos/configuracion.txt")
with ruta.open(encoding="utf-8") as f:
    contenido = f.read()   # str con todo el archivo

# Lectura línea a línea — eficiente para archivos grandes (lazy)
with ruta.open(encoding="utf-8") as f:
    for numero, linea in enumerate(f, start=1):
        linea = linea.rstrip("\n")   # eliminar salto de línea
        print(f"{numero:4}: {linea}")

# Leer todas las líneas en una lista
with ruta.open(encoding="utf-8") as f:
    lineas = f.readlines()         # ['línea1\n', 'línea2\n', ...]
    lineas_limpias = [l.rstrip() for l in lineas]


# ─────────────────────────────────────────────────────────────────────────────
# Escritura de texto
# ─────────────────────────────────────────────────────────────────────────────
salida = Path("salida/informe.txt")
salida.parent.mkdir(parents=True, exist_ok=True)

# Sobreescribir
with salida.open("w", encoding="utf-8") as f:
    f.write("Informe de procesamiento\n")
    f.write(f"Fecha: 2024-01-15\n")

# Añadir al final (no sobreescribe)
with salida.open("a", encoding="utf-8") as f:
    f.write("Procesamiento completado.\n")

# Usar print() para escritura más cómoda
with salida.open("a", encoding="utf-8") as f:
    print("Total de registros: 1500", file=f)
    print("Errores: 0", file=f)


# ─────────────────────────────────────────────────────────────────────────────
# Leer línea a línea de forma lazy — eficiente para archivos de gigabytes
# ─────────────────────────────────────────────────────────────────────────────
from typing import Generator

def leer_lineas_validas(ruta: Path) -> Generator[str, None, None]:
    """Genera líneas no vacías y sin comentarios de forma lazy."""
    with ruta.open(encoding="utf-8") as f:
        for linea in f:
            linea = linea.strip()
            if linea and not linea.startswith("#"):  # saltar vacías y comentarios
                yield linea

# El archivo solo ocupa en memoria UNA línea a la vez
for linea in leer_lineas_validas(Path("datos/lista_urls.txt")):
    print(f"Procesando: {linea}")
```

---

## Parte 3 — JSON

```python
import json
from pathlib import Path
from datetime import datetime, date
from decimal import Decimal
from typing import Any

# ─────────────────────────────────────────────────────────────────────────────
# Operaciones básicas
# ─────────────────────────────────────────────────────────────────────────────
# json.loads — texto JSON → objeto Python
texto = '{"nombre": "Ana", "edad": 30, "activo": true}'
datos = json.loads(texto)
print(datos["nombre"])   # Ana
print(type(datos))       # <class 'dict'>

# json.dumps — objeto Python → texto JSON
datos = {"nombre": "Ana", "edad": 30, "tags": ["python", "data"]}
texto = json.dumps(datos)                             # sin formato
texto_bonito = json.dumps(datos, indent=2, ensure_ascii=False)
# ensure_ascii=False preserva caracteres como ñ, á, ü, 中文


# ─────────────────────────────────────────────────────────────────────────────
# json.load / json.dump — trabajar directamente con archivos
# ─────────────────────────────────────────────────────────────────────────────
ruta = Path("datos/config.json")

# Leer desde archivo
with ruta.open(encoding="utf-8") as f:
    config = json.load(f)

# Escribir a archivo
with ruta.open("w", encoding="utf-8") as f:
    json.dump(config, f, indent=2, ensure_ascii=False)


# ─────────────────────────────────────────────────────────────────────────────
# Encoder personalizado — serializar tipos que JSON no soporta nativamente
# ─────────────────────────────────────────────────────────────────────────────
class EncoderExtendido(json.JSONEncoder):
    """Añade soporte para datetime, date y Decimal."""

    def default(self, obj: Any) -> Any:
        if isinstance(obj, datetime):
            return obj.isoformat()         # "2024-01-15T10:30:00"
        if isinstance(obj, date):
            return obj.isoformat()         # "2024-01-15"
        if isinstance(obj, Decimal):
            return float(obj)              # puede perder precisión
        return super().default(obj)        # delegar al comportamiento por defecto


evento = {
    "nombre": "Conferencia Python",
    "fecha": datetime(2024, 6, 15, 9, 0),
    "precio": Decimal("149.99"),
}
texto = json.dumps(evento, cls=EncoderExtendido, indent=2)
print(texto)
# {
#   "nombre": "Conferencia Python",
#   "fecha": "2024-06-15T09:00:00",
#   "precio": 149.99
# }


# ─────────────────────────────────────────────────────────────────────────────
# Decoder personalizado — transformar tipos al cargar JSON
# ─────────────────────────────────────────────────────────────────────────────
def decodificar_fechas(dct: dict) -> dict:
    """Convierte automáticamente strings ISO 8601 a datetime."""
    for clave, valor in dct.items():
        if isinstance(valor, str):
            try:
                dct[clave] = datetime.fromisoformat(valor)
            except ValueError:
                pass  # no es una fecha, mantener como string
    return dct

texto_json = '{"nombre": "Ana", "alta": "2024-01-15T10:00:00"}'
datos = json.loads(texto_json, object_hook=decodificar_fechas)
print(type(datos["alta"]))   # <class 'datetime.datetime'>
```

---

## Parte 4 — CSV

```python
import csv
from pathlib import Path

# ─────────────────────────────────────────────────────────────────────────────
# Leer CSV básico con csv.reader
# ─────────────────────────────────────────────────────────────────────────────
with Path("datos/ventas.csv").open(encoding="utf-8", newline="") as f:
    # newline="" es necesario para que csv maneje los saltos de línea correctamente
    lector = csv.reader(f, delimiter=",")
    cabecera = next(lector)            # leer la primera fila como cabecera
    print(f"Columnas: {cabecera}")

    for fila in lector:
        print(fila)   # ['2024-01-15', 'SKU-001', '5', '19.99']


# ─────────────────────────────────────────────────────────────────────────────
# DictReader — cada fila como diccionario (más legible)
# ─────────────────────────────────────────────────────────────────────────────
with Path("datos/ventas.csv").open(encoding="utf-8", newline="") as f:
    lector = csv.DictReader(f)
    for fila in lector:
        # fila es un dict: {"fecha": "2024-01-15", "sku": "SKU-001", ...}
        importe = float(fila["precio"]) * int(fila["cantidad"])
        print(f"{fila['fecha']}: {fila['sku']} → {importe:.2f} €")


# ─────────────────────────────────────────────────────────────────────────────
# Escribir CSV con csv.writer
# ─────────────────────────────────────────────────────────────────────────────
filas = [
    ["fecha", "producto", "cantidad", "precio"],
    ["2024-01-15", "Teclado", "3", "49.99"],
    ["2024-01-15", "Ratón", "5", "24.99"],
]

with Path("salida/reporte.csv").open("w", encoding="utf-8", newline="") as f:
    escritor = csv.writer(f, quoting=csv.QUOTE_MINIMAL)
    escritor.writerows(filas)


# ─────────────────────────────────────────────────────────────────────────────
# DictWriter — escribir desde lista de diccionarios
# ─────────────────────────────────────────────────────────────────────────────
datos = [
    {"nombre": "Ana García", "email": "ana@ejemplo.com", "rol": "admin"},
    {"nombre": "Luis Martín", "email": "luis@ejemplo.com", "rol": "usuario"},
]

with Path("salida/usuarios.csv").open("w", encoding="utf-8", newline="") as f:
    campos = ["nombre", "email", "rol"]
    escritor = csv.DictWriter(f, fieldnames=campos)
    escritor.writeheader()             # escribe la fila de cabecera
    escritor.writerows(datos)          # escribe todas las filas
```

---

## Parte 5 — Formatos adicionales: TOML, INI y archivos temporales

```python
# ─────────────────────────────────────────────────────────────────────────────
# tomllib — leer archivos TOML (Python 3.11+, incluido en stdlib)
# ─────────────────────────────────────────────────────────────────────────────
import tomllib
from pathlib import Path

# Leer pyproject.toml u otro archivo TOML
with Path("pyproject.toml").open("rb") as f:   # DEBE abrirse en modo binario
    config = tomllib.load(f)

nombre = config["tool"]["poetry"]["name"]
version = config["tool"]["poetry"]["version"]
python_req = config["tool"]["poetry"]["dependencies"]["python"]
print(f"{nombre} v{version} — requiere Python {python_req}")


# ─────────────────────────────────────────────────────────────────────────────
# configparser — archivos .ini / .cfg (configuración estilo clásico)
# ─────────────────────────────────────────────────────────────────────────────
import configparser

# config.ini:
# [base_datos]
# host = localhost
# puerto = 5432
# nombre = mi_bd
#
# [api]
# url = https://api.ejemplo.com
# timeout = 30

parser = configparser.ConfigParser()
parser.read("config.ini", encoding="utf-8")

# Leer valores — siempre devuelve strings
host = parser["base_datos"]["host"]
puerto = parser.getint("base_datos", "puerto")     # conversión automática a int
timeout = parser.getfloat("api", "timeout")        # conversión automática a float
debug = parser.getboolean("general", "debug", fallback=False)  # valor por defecto

# Escribir un archivo .ini
nuevo_config = configparser.ConfigParser()
nuevo_config["general"] = {
    "version": "1.0.0",
    "debug": "false",
    "log_level": "INFO",
}
nuevo_config["base_datos"] = {
    "host": "localhost",
    "puerto": "5432",
}
with open("mi_config.ini", "w") as f:
    nuevo_config.write(f)


# ─────────────────────────────────────────────────────────────────────────────
# tempfile — archivos y directorios temporales
# ─────────────────────────────────────────────────────────────────────────────
import tempfile

# Archivo temporal — se elimina automáticamente al salir del with
with tempfile.NamedTemporaryFile(
    mode="w",
    suffix=".csv",
    delete=True,
    encoding="utf-8",
) as f:
    f.write("id,nombre\n1,Ana\n")
    ruta_temp = f.name
    print(f"Archivo temporal: {ruta_temp}")
# Aquí el archivo ya no existe

# Directorio temporal
with tempfile.TemporaryDirectory() as dir_temp:
    ruta_dir = Path(dir_temp)
    (ruta_dir / "datos.json").write_text('{"ok": true}')
    print(f"Directorio temporal: {dir_temp}")
# Aquí el directorio y su contenido ya no existen
```

---

## Parte 6 — shutil y zipfile

```python
import shutil
import zipfile
from pathlib import Path

# ─────────────────────────────────────────────────────────────────────────────
# shutil — operaciones de alto nivel sobre archivos y directorios
# ─────────────────────────────────────────────────────────────────────────────

# Copiar un archivo
shutil.copy2("origen.txt", "copia.txt")           # copy2 preserva metadatos
shutil.copy2("archivo.txt", "directorio/")        # copia dentro del directorio

# Copiar árbol de directorios completo
shutil.copytree("src/", "backup_src/")

# Mover / renombrar
shutil.move("viejo_nombre.txt", "nuevo_nombre.txt")
shutil.move("archivo.txt", "otro_directorio/archivo.txt")

# Eliminar árbol de directorios (recursivo, IRREVERSIBLE)
shutil.rmtree("directorio_a_eliminar")

# Comprimir un directorio en ZIP/TAR
shutil.make_archive(
    base_name="backup_2024",          # nombre sin extensión
    format="zip",                     # zip, tar, gztar, bztar, xztar
    root_dir=".",                     # directorio raíz
    base_dir="mi_proyecto",           # directorio a comprimir
)
# Genera: backup_2024.zip

# Extraer un archivo comprimido
shutil.unpack_archive("backup_2024.zip", extract_dir="extraido/")


# ─────────────────────────────────────────────────────────────────────────────
# zipfile — crear y leer archivos ZIP con control granular
# ─────────────────────────────────────────────────────────────────────────────

# Crear un ZIP
with zipfile.ZipFile("reporte.zip", "w", compression=zipfile.ZIP_DEFLATED) as zf:
    zf.write("datos/ventas.csv", arcname="ventas.csv")    # arcname = nombre dentro del ZIP
    zf.write("datos/config.json", arcname="config.json")
    # Añadir contenido en memoria sin archivo físico
    zf.writestr("README.txt", "Archivo de reporte generado automáticamente.\n")

# Leer el contenido de un ZIP
with zipfile.ZipFile("reporte.zip", "r") as zf:
    print(zf.namelist())                 # ['ventas.csv', 'config.json', 'README.txt']
    info = zf.getinfo("ventas.csv")
    print(f"Tamaño: {info.file_size} bytes")

    # Extraer todo
    zf.extractall("extraido/")

    # Extraer un archivo específico a memoria
    contenido = zf.read("README.txt").decode("utf-8")
    print(contenido)
```

---

## Ejemplo aplicado — Procesador de logs en JSON

Lee un directorio de logs en formato JSON Lines, filtra por nivel y fecha,
agrega estadísticas y escribe un reporte en CSV.

```python
"""
procesador_logs.py
Lee logs JSON Lines, filtra, agrega por servicio y escribe CSV de reporte.
"""
from __future__ import annotations

import csv
import json
import sys
from collections import defaultdict
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from pathlib import Path
from typing import Generator


# ── Modelo de un evento de log ────────────────────────────────────────────────
@dataclass
class EventoLog:
    timestamp: datetime
    nivel: str
    servicio: str
    mensaje: str
    duracion_ms: float = 0.0
    error: str | None = None

    @classmethod
    def desde_dict(cls, datos: dict) -> "EventoLog":
        """Construye un EventoLog desde un diccionario JSON."""
        return cls(
            timestamp=datetime.fromisoformat(datos["timestamp"]),
            nivel=datos.get("nivel", "INFO").upper(),
            servicio=datos.get("servicio", "desconocido"),
            mensaje=datos.get("mensaje", ""),
            duracion_ms=float(datos.get("duracion_ms", 0)),
            error=datos.get("error"),
        )


# ── Lector lazy de JSON Lines ─────────────────────────────────────────────────
def leer_json_lines(ruta: Path) -> Generator[EventoLog, None, None]:
    """Lee un archivo JSON Lines línea a línea (una línea = un objeto JSON)."""
    with ruta.open(encoding="utf-8") as f:
        for numero_linea, linea in enumerate(f, start=1):
            linea = linea.strip()
            if not linea:
                continue
            try:
                datos = json.loads(linea)
                yield EventoLog.desde_dict(datos)
            except (json.JSONDecodeError, KeyError) as e:
                print(
                    f"[aviso] Línea {numero_linea} ignorada en {ruta.name}: {e}",
                    file=sys.stderr
                )


# ── Procesador y agregador ────────────────────────────────────────────────────
@dataclass
class EstadisticasServicio:
    servicio: str
    total_eventos: int = 0
    errores: int = 0
    advertencias: int = 0
    duracion_total_ms: float = 0.0
    duracion_max_ms: float = 0.0
    mensajes_error: list[str] = field(default_factory=list)

    @property
    def duracion_media_ms(self) -> float:
        if self.total_eventos == 0:
            return 0.0
        return self.duracion_total_ms / self.total_eventos

    @property
    def tasa_error_pct(self) -> float:
        if self.total_eventos == 0:
            return 0.0
        return (self.errores / self.total_eventos) * 100


def procesar_directorio(
    dir_logs: Path,
    desde: datetime | None = None,
    hasta: datetime | None = None,
    niveles: set[str] | None = None,
) -> dict[str, EstadisticasServicio]:
    """
    Procesa todos los archivos .jsonl del directorio y devuelve estadísticas
    agrupadas por servicio.
    """
    stats: dict[str, EstadisticasServicio] = defaultdict(
        lambda: EstadisticasServicio(servicio="")
    )

    archivos_log = sorted(dir_logs.glob("*.jsonl"))
    if not archivos_log:
        print(f"[aviso] No se encontraron archivos .jsonl en {dir_logs}")
        return {}

    total_eventos = 0
    total_ignorados = 0

    for archivo in archivos_log:
        for evento in leer_json_lines(archivo):
            # Filtrar por rango de fechas
            if desde and evento.timestamp < desde:
                total_ignorados += 1
                continue
            if hasta and evento.timestamp > hasta:
                total_ignorados += 1
                continue
            # Filtrar por nivel
            if niveles and evento.nivel not in niveles:
                total_ignorados += 1
                continue

            # Agregar estadísticas por servicio
            if evento.servicio not in stats:
                stats[evento.servicio] = EstadisticasServicio(servicio=evento.servicio)

            s = stats[evento.servicio]
            s.total_eventos += 1
            s.duracion_total_ms += evento.duracion_ms
            s.duracion_max_ms = max(s.duracion_max_ms, evento.duracion_ms)

            if evento.nivel == "ERROR":
                s.errores += 1
                if evento.error:
                    # Guardar hasta 5 mensajes de error únicos por servicio
                    if len(s.mensajes_error) < 5 and evento.error not in s.mensajes_error:
                        s.mensajes_error.append(evento.error)
            elif evento.nivel == "WARNING":
                s.advertencias += 1

            total_eventos += 1

    print(f"Procesados: {total_eventos} eventos, ignorados: {total_ignorados}")
    return dict(stats)


def exportar_reporte_csv(
    stats: dict[str, EstadisticasServicio],
    ruta_salida: Path,
) -> None:
    """Escribe el reporte de estadísticas en un archivo CSV."""
    ruta_salida.parent.mkdir(parents=True, exist_ok=True)

    campos = [
        "servicio",
        "total_eventos",
        "errores",
        "advertencias",
        "tasa_error_pct",
        "duracion_media_ms",
        "duracion_max_ms",
        "mensajes_error",
    ]

    with ruta_salida.open("w", encoding="utf-8", newline="") as f:
        escritor = csv.DictWriter(f, fieldnames=campos)
        escritor.writeheader()
        # Ordenar por tasa de error descendente
        for s in sorted(stats.values(), key=lambda x: x.tasa_error_pct, reverse=True):
            escritor.writerow({
                "servicio": s.servicio,
                "total_eventos": s.total_eventos,
                "errores": s.errores,
                "advertencias": s.advertencias,
                "tasa_error_pct": f"{s.tasa_error_pct:.2f}",
                "duracion_media_ms": f"{s.duracion_media_ms:.1f}",
                "duracion_max_ms": f"{s.duracion_max_ms:.1f}",
                "mensajes_error": " | ".join(s.mensajes_error),
            })

    print(f"Reporte exportado a: {ruta_salida}")


# ── Punto de entrada ──────────────────────────────────────────────────────────
if __name__ == "__main__":
    dir_logs = Path("logs")
    dir_salida = Path("salida")

    # Procesar solo las últimas 24 horas
    ahora = datetime.now()
    hace_24h = ahora - timedelta(hours=24)

    print(f"Procesando logs desde {hace_24h:%Y-%m-%d %H:%M} hasta {ahora:%Y-%m-%d %H:%M}")

    estadisticas = procesar_directorio(
        dir_logs=dir_logs,
        desde=hace_24h,
        hasta=ahora,
        niveles={"ERROR", "WARNING", "INFO"},
    )

    if estadisticas:
        exportar_reporte_csv(
            stats=estadisticas,
            ruta_salida=dir_salida / f"reporte_{ahora:%Y%m%d_%H%M}.csv",
        )

        # Imprimir resumen en consola
        print("\nResumen por servicio:")
        print(f"{'Servicio':<20} {'Eventos':>8} {'Errores':>8} {'Tasa error':>12} {'Media ms':>10}")
        print("─" * 62)
        for s in sorted(estadisticas.values(), key=lambda x: x.errores, reverse=True):
            print(
                f"{s.servicio:<20} {s.total_eventos:>8} {s.errores:>8} "
                f"{s.tasa_error_pct:>11.1f}% {s.duracion_media_ms:>10.1f}"
            )
```

---

## Ejercicios propuestos

1. **Migrador de CSV a JSON** — Lee un directorio con múltiples archivos CSV (todos
   con la misma estructura) y los combina en un único archivo JSON Lines, convirtiendo
   automáticamente columnas numéricas a `int`/`float` y fechas en formato `YYYY-MM-DD`
   a objetos `datetime`. Reporta cuántas filas procesó y cuántas falló.

2. **Organizador de archivos** — Escribe un script que recorra un directorio de
   descargas y mueva cada archivo al subdirectorio correspondiente según su extensión
   (`.pdf` → `documentos/`, `.jpg`/`.png` → `imagenes/`, `.py` → `codigo/`, etc.).
   Usa `shutil.move` y `pathlib`. Si el archivo destino ya existe, añade un sufijo
   numérico para evitar sobreescribir.

3. **Caché de resultados en JSON** — Implementa un decorador `@cachear_en_disco(ruta)`
   que serialice el resultado de una función en un archivo JSON usando el hash de los
   argumentos como nombre de archivo. Si el archivo existe, devuelve el resultado
   cacheado sin llamar a la función. Usa un encoder personalizado para soportar
   `datetime` y `Decimal`.

4. **Comprimidor de backups** — Crea una función `hacer_backup(directorio, max_backups)`
   que comprima el directorio en un ZIP con nombre `backup_YYYYMMDD_HHMMSS.zip` en
   una carpeta `backups/`. Si ya hay más de `max_backups` archivos ZIP en esa carpeta,
   elimina el más antiguo. Usa `zipfile` para crear el ZIP y `pathlib` para gestionar
   los archivos.

---

## Resumen de la página 8

- `pathlib.Path` es la API moderna para rutas: multiplataforma, legible y orientada a objetos; usa `/` para construir rutas y métodos como `.exists()`, `.is_dir()`, `.rglob()`
- Siempre usar `with open(...)` para garantizar que el archivo se cierra aunque ocurra una excepción; especificar siempre `encoding="utf-8"` y `newline=""` para CSV
- Leer archivos grandes línea a línea con un bucle `for linea in f` es "lazy" y consume memoria constante independientemente del tamaño del archivo
- `json.loads`/`json.dumps` trabajan con strings; `json.load`/`json.dump` trabajan directamente con archivos; `ensure_ascii=False` preserva caracteres no ASCII
- Un encoder personalizado (`JSONEncoder.default`) permite serializar tipos como `datetime`, `Decimal` o clases propias a JSON
- `csv.DictReader` y `csv.DictWriter` permiten trabajar con CSV usando diccionarios en lugar de listas, lo que hace el código más legible y robusto
- `tomllib` (Python 3.11+) lee archivos TOML en modo binario; `configparser` lee archivos `.ini`; ambos son parte de la biblioteca estándar
- `shutil` cubre operaciones de alto nivel (copiar, mover, comprimir árbol); `zipfile` da control granular sobre archivos ZIP incluyendo escritura en memoria

---

> **Siguiente página →** Página 12: Excepciones, logging y debugging profesional.
