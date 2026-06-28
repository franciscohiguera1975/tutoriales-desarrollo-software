# Tutorial Python — Página 6
## Módulo 1 · Entorno y Fundamentos
### Funciones avanzadas — `*args`, `**kwargs`, lambdas, closures y generadores

La página 5 cubrió funciones básicas: definición, parámetros con valores por defecto, type hints y docstrings. Esta página cubre los mecanismos avanzados que permiten escribir código flexible, reutilizable y eficiente en memoria.

---

## 1. `*args` — número variable de argumentos posicionales

### El operador `*` en la firma de la función

Cuando una función declara `*args`, Python empaqueta todos los argumentos posicionales adicionales en una **tupla**. El nombre `args` es convención; podría llamarse `*valores` o `*elementos`.

```python
# Función que acepta cualquier cantidad de valores numéricos
def calcular_promedio(*valores: float) -> float:
    """Calcula el promedio de cualquier número de valores."""
    # args es una tupla — podemos iterar, len(), sum(), etc.
    if not valores:
        raise ValueError("Se requiere al menos un valor para calcular el promedio")
    return sum(valores) / len(valores)

# Llamadas con diferente número de argumentos
print(calcular_promedio(10))                          # 10.0
print(calcular_promedio(10, 20, 30))                  # 20.0
print(calcular_promedio(95.5, 87.3, 91.0, 78.8))     # 88.15

# Función que acepta cualquier número de métricas de monitoreo
def registrar_metricas(timestamp: str, *metricas: float) -> dict:
    """
    Registra un conjunto variable de métricas con su timestamp.
    metricas puede contener: CPU, RAM, disco, red, latencia, etc.
    """
    nombres = ["cpu", "ram", "disco", "red", "latencia"]
    # Emparejar cada métrica con su nombre usando zip
    registro = {
        "timestamp": timestamp,
        "metricas": dict(zip(nombres, metricas)),
        "total_metricas": len(metricas),
    }
    return registro

r1 = registrar_metricas("2024-01-15T10:00:00", 45.2, 78.1)
r2 = registrar_metricas("2024-01-15T10:00:05", 62.8, 91.3, 55.0, 12.4, 23.1)

print(f"\nRegistro 1: {r1}")
print(f"Registro 2: {r2}")

# Función para concatenar cualquier número de strings con separador
def construir_ruta(*segmentos: str, separador: str = "/") -> str:
    """Construye una ruta URL o de sistema de archivos desde segmentos."""
    # Filtrar segmentos vacíos y quitar slashes duplicados
    limpios = [seg.strip("/") for seg in segmentos if seg.strip("/")]
    return separador + separador.join(limpios)

print(construir_ruta("api", "v2", "usuarios", "42"))     # /api/v2/usuarios/42
print(construir_ruta("home", "user", "docs", separador="/"))  # /home/user/docs
```

> **Prueba esto:** Llama a `calcular_promedio()` sin argumentos. ¿Qué ocurre? Luego modifica la función para que en lugar de lanzar una excepción, retorne `0.0` cuando no se pasan valores. ¿Cuál comportamiento es más correcto para un sistema de monitoreo?

### Desempacar con `*` al llamar: `funcion(*mi_lista)`

```python
# El operador * en la llamada desempaca una lista/tupla en argumentos posicionales

def crear_conexion_db(host: str, puerto: int, usuario: str, password: str) -> str:
    """Simula la creación de una conexión a base de datos."""
    return f"postgresql://{usuario}:{password}@{host}:{puerto}/prod"

# Con argumentos explícitos (difícil de escalar)
url_1 = crear_conexion_db("db.prod.internal", 5432, "app_user", "s3cr3t")

# Con desempaquetado desde una lista (más flexible)
params_db = ["db.prod.internal", 5432, "app_user", "s3cr3t"]
url_2 = crear_conexion_db(*params_db)  # Equivalente a la llamada anterior

print(f"URL 1: {url_1}")
print(f"URL 2: {url_2}")
print(f"Iguales: {url_1 == url_2}")

# Caso real: combinar listas de hosts de diferentes fuentes
primarios = ["web-01", "web-02"]
secundarios = ["web-03", "web-04"]
todos_los_hosts = [*primarios, *secundarios]  # Desempacar para combinar listas

print(f"\nHosts combinados: {todos_los_hosts}")

# Desempacar rangos y generadores en llamadas
def sumar_rango(a: int, b: int, c: int) -> int:
    return a + b + c

coordenadas = range(1, 4)  # Genera 1, 2, 3
resultado = sumar_rango(*coordenadas)
print(f"\nSuma de rango desempaquetado: {resultado}")
```

> **Prueba esto:** Crea una lista `config = ["localhost", 8080, "admin", "pass123"]` y desempácala en la llamada a `crear_conexion_db`. Luego intenta añadir un quinto elemento a la lista y vuelve a llamarla con `*config`. ¿Qué error obtienes y por qué?

---

## 2. `**kwargs` — número variable de argumentos con nombre

### El operador `**` en la firma de la función

`**kwargs` empaqueta todos los argumentos de palabra clave adicionales en un **diccionario**. Es ideal para funciones de configuración y constructores flexibles.

```python
# Función de configuración que acepta parámetros opcionales ilimitados
def configurar_cliente_http(
    base_url: str,
    **opciones: object
) -> dict:
    """
    Crea una configuración de cliente HTTP con opciones variables.
    opciones puede incluir: timeout, retries, ssl_verify, headers, proxies, etc.
    """
    # Valores por defecto para opciones no especificadas
    config = {
        "base_url": base_url,
        "timeout": opciones.get("timeout", 30),
        "retries": opciones.get("retries", 3),
        "ssl_verify": opciones.get("ssl_verify", True),
        "headers": opciones.get("headers", {}),
    }

    # Añadir cualquier opción extra no contemplada
    opciones_extra = {k: v for k, v in opciones.items() if k not in config}
    config["opciones_extra"] = opciones_extra

    return config

# Llamadas con diferentes combinaciones de opciones
cliente_simple = configurar_cliente_http("https://api.empresa.com")
cliente_interno = configurar_cliente_http(
    "https://api.internal.empresa.com",
    timeout=5,
    ssl_verify=False,
    retries=1,
)
cliente_externo = configurar_cliente_http(
    "https://api.proveedor.com",
    timeout=60,
    retries=5,
    headers={"Authorization": "Bearer token123"},
    proxy="http://proxy.empresa.com:3128",
    verify_hostname=True,
)

for nombre, cliente in [("simple", cliente_simple), ("interno", cliente_interno), ("externo", cliente_externo)]:
    print(f"\nCliente {nombre}:")
    for k, v in cliente.items():
        print(f"  {k}: {v}")

# kwargs útil para logging estructurado
def log_evento(nivel: str, mensaje: str, **contexto: object) -> None:
    """Registra un evento con contexto estructurado variable."""
    campos = " ".join(f"{k}={v!r}" for k, v in contexto.items())
    print(f"[{nivel.upper()}] {mensaje} | {campos}")

log_evento("error", "Fallo en pago", usuario_id=42, monto=150.00, moneda="MXN", intento=3)
log_evento("info", "Usuario autenticado", usuario_id=42, ip="10.0.1.55", metodo="JWT")
```

> **Prueba esto:** Llama a `configurar_cliente_http` con un parámetro desconocido como `mi_opcion_rara=True`. ¿Dónde aparece en el diccionario retornado? Luego imprime solo las `opciones_extra` del `cliente_externo`.

### Desempacar con `**` al llamar: `funcion(**mi_dict)`

```python
# El operador ** en la llamada desempaca un diccionario en kwargs

def crear_usuario(nombre: str, email: str, rol: str = "usuario", activo: bool = True) -> dict:
    """Crea un registro de usuario en el sistema."""
    return {
        "nombre": nombre,
        "email": email,
        "rol": rol,
        "activo": activo,
        "id": hash(email) % 100000,  # ID simplificado para el ejemplo
    }

# Desde una API o base de datos, los datos llegan como diccionario
datos_usuario = {
    "nombre": "Ana García",
    "email": "ana@empresa.com",
    "rol": "admin",
    "activo": True,
}

# Sin desempacar — incómodo
usuario_1 = crear_usuario(
    nombre=datos_usuario["nombre"],
    email=datos_usuario["email"],
    rol=datos_usuario["rol"],
    activo=datos_usuario["activo"],
)

# Con desempacar — limpio y escalable
usuario_2 = crear_usuario(**datos_usuario)

print(f"Usuario creado: {usuario_2}")

# Combinar diccionarios al desempacar (Python 3.9+)
config_base = {"timeout": 30, "retries": 3, "ssl": True}
config_override = {"timeout": 60, "debug": True}

# El segundo dict sobreescribe claves del primero
config_final = {**config_base, **config_override}
print(f"\nConfig combinada: {config_final}")
```

> **Prueba esto:** Añade una clave `"rol_invalido": "superadmin"` al diccionario `datos_usuario` y vuelve a llamar `crear_usuario(**datos_usuario)`. ¿Qué error obtienes? ¿Cómo lo resolverías filtrando el diccionario antes de desempacar?

---

## 3. Combinar todos los tipos de parámetros

Python 3.8+ introduce `/` y `*` como separadores en la firma de funciones para control preciso del paso de argumentos.

```python
# Orden correcto de parámetros:
# def f(positional_only, /, normales, *args, keyword_only, **kwargs)

def crear_registro_auditoria(
    # Positional-only (antes del /): no pueden pasarse por nombre
    accion: str,
    recurso: str,
    /,
    # Normales: pueden pasarse por posición o por nombre
    usuario_id: int,
    timestamp: str = "ahora",
    # *args: colecta argumentos posicionales extras
    *metadatos: str,
    # Keyword-only (después de *args): DEBEN pasarse por nombre
    nivel: str = "INFO",
    notificar: bool = False,
    # **kwargs: colecta kwargs extra
    **contexto: object,
) -> dict:
    """
    Registra una acción de auditoría con máxima flexibilidad.

    positional_only: accion, recurso (no pueden nombrarse en la llamada)
    normales: usuario_id, timestamp
    *metadatos: etiquetas, categorías, etc. (posicionales extras)
    keyword_only: nivel, notificar (solo por nombre)
    **contexto: cualquier dato adicional
    """
    return {
        "accion": accion,
        "recurso": recurso,
        "usuario_id": usuario_id,
        "timestamp": timestamp,
        "metadatos": metadatos,
        "nivel": nivel,
        "notificar": notificar,
        "contexto": contexto,
    }

# Uso correcto: positional-only van primero, keyword-only por nombre
registro = crear_registro_auditoria(
    "DELETE",             # accion — positional-only
    "/api/usuario/42",    # recurso — positional-only
    usuario_id=1001,      # normal — por nombre
    timestamp="2024-01-15T10:30:00",
    "etiqueta-critica", "pii", "gdpr",  # *metadatos
    nivel="WARN",         # keyword-only — debe ser por nombre
    notificar=True,       # keyword-only
    ip_origen="203.0.113.1",  # **contexto
    session_id="abc-xyz-123",
)

print("Registro de auditoría:")
for campo, valor in registro.items():
    print(f"  {campo:<15}: {valor}")

# Intentar pasar positional-only por nombre causaría TypeError:
# crear_registro_auditoria(accion="DELETE", ...)  # TypeError!
```

> **Prueba esto:** Intenta llamar a la función con `accion="DELETE"` (por nombre). ¿Qué error obtienes? Luego elimina el `/` de la firma y repite la llamada. ¿Cambia el comportamiento?

---

## 4. Lambdas — funciones anónimas de una expresión

### Cuándo usar lambda y cuándo no

Lambda es apropiada para expresiones simples de una sola línea, especialmente como argumento `key=` en ordenaciones. Para cualquier lógica con más de una expresión, usa `def`.

```python
from functools import reduce

# Caso ideal para lambda: key= en sort/sorted
servidores = [
    {"nombre": "web-03", "carga": 72, "region": "us-east"},
    {"nombre": "web-01", "carga": 45, "region": "eu-west"},
    {"nombre": "web-05", "carga": 91, "region": "us-east"},
    {"nombre": "web-02", "carga": 30, "region": "eu-west"},
    {"nombre": "web-04", "carga": 58, "region": "us-east"},
]

# Ordenar por carga ascendente
por_carga = sorted(servidores, key=lambda s: s["carga"])
print("Servidores por carga (menor a mayor):")
for s in por_carga:
    print(f"  {s['nombre']}: {s['carga']}%")

# Ordenar con múltiples criterios: primero por región, luego por carga descendente
por_region_y_carga = sorted(
    servidores,
    key=lambda s: (s["region"], -s["carga"])  # Tupla: región asc, carga desc
)
print("\nServidores por región y carga descendente:")
for s in por_region_y_carga:
    print(f"  {s['region']:<12} {s['nombre']}: {s['carga']}%")

# Lambda útil para transformaciones rápidas en pipelines
niveles_prioridad = {"CRITICAL": 0, "ERROR": 1, "WARN": 2, "INFO": 3, "DEBUG": 4}
alertas = ["INFO: ping", "ERROR: timeout", "CRITICAL: caída", "WARN: alto uso", "DEBUG: trace"]

# Ordenar alertas por prioridad usando lambda
alertas_ordenadas = sorted(
    alertas,
    key=lambda a: niveles_prioridad.get(a.split(":")[0], 99)
)
print("\nAlertas por prioridad:")
for alerta in alertas_ordenadas:
    print(f"  {alerta}")
```

> **Prueba esto:** Añade un servidor `{"nombre": "web-06", "carga": 45, "region": "ap-south"}` a la lista. Al ordenar con el criterio de múltiples claves, ¿dónde aparece? ¿Puedes modificar el lambda para que dentro de cada región, los servidores con igual carga se ordenen alfabéticamente por nombre?

### `map()`, `filter()`, `reduce()` con lambdas — y por qué las comprensiones suelen ser mejores

```python
from functools import reduce

metricas_cpu = [45.2, 91.8, 23.4, 78.5, 55.0, 88.9, 12.1, 67.3]

# --- map(): aplicar función a cada elemento ---
# Con lambda (menos legible para transformaciones complejas)
metricas_redondeadas_map = list(map(lambda x: round(x), metricas_cpu))

# Con comprensión (más idiomático en Python)
metricas_redondeadas_comp = [round(x) for x in metricas_cpu]

print(f"map con lambda: {metricas_redondeadas_map}")
print(f"comprensión:    {metricas_redondeadas_comp}")

# --- filter(): conservar elementos que cumplen condición ---
# Con lambda
criticos_filter = list(filter(lambda x: x > 80, metricas_cpu))

# Con comprensión (generalmente preferido)
criticos_comp = [x for x in metricas_cpu if x > 80]

print(f"\nCríticos (>80%) con filter:       {criticos_filter}")
print(f"Críticos (>80%) con comprensión:  {criticos_comp}")

# --- reduce(): único caso donde lambda puede ser apropiado ---
# Calcular el producto acumulativo (no hay built-in como sum() para multiplicar)
valores = [2, 3, 4, 5]
producto = reduce(lambda acum, x: acum * x, valores)
print(f"\nProducto de {valores}: {producto}")  # 120

# Encontrar el máximo manualmente con reduce (aunque max() es mejor)
maximo = reduce(lambda a, b: a if a > b else b, metricas_cpu)
print(f"Máximo de métricas con reduce: {maximo}")
print(f"Máximo con max():              {max(metricas_cpu)}")
```

> **Prueba esto:** Reescribe la línea de `filter` con lambda para que filtre las métricas que estén entre 50 y 80 (inclusive). Escribe la versión con lambda y la versión con comprensión. ¿Cuál te parece más legible?

---

## 5. Funciones de orden superior

Una función de orden superior acepta funciones como argumentos o las retorna como resultado. Este patrón es fundamental para pipelines de datos y middleware.

```python
from typing import Callable

# Pipeline de transformaciones: lista de funciones que se aplican en secuencia
def ejecutar_pipeline(datos: list, *transformaciones: Callable) -> list:
    """
    Aplica una serie de transformaciones en orden a una lista de datos.
    Cada transformación recibe la lista actual y retorna la lista transformada.
    """
    resultado = datos
    for i, transformacion in enumerate(transformaciones, start=1):
        nombre = transformacion.__name__
        resultado = transformacion(resultado)
        print(f"  Paso {i} ({nombre}): {len(resultado)} elementos")
    return resultado

# Definir transformaciones individuales
def filtrar_invalidos(registros: list[dict]) -> list[dict]:
    """Elimina registros sin campo 'id' o con 'id' nulo."""
    return [r for r in registros if r.get("id") is not None]

def normalizar_emails(registros: list[dict]) -> list[dict]:
    """Convierte emails a minúsculas y elimina espacios."""
    return [{**r, "email": r.get("email", "").strip().lower()} for r in registros]

def filtrar_activos(registros: list[dict]) -> list[dict]:
    """Conserva solo registros con campo 'activo' en True."""
    return [r for r in registros if r.get("activo", False)]

def agregar_dominio(registros: list[dict]) -> list[dict]:
    """Extrae el dominio del email y lo añade como campo separado."""
    resultado = []
    for r in registros:
        email = r.get("email", "")
        dominio = email.split("@")[1] if "@" in email else "desconocido"
        resultado.append({**r, "dominio": dominio})
    return resultado

# Dataset inicial con datos sucios
usuarios_raw = [
    {"id": 1,    "email": " Ana@Empresa.COM ",  "activo": True},
    {"id": None, "email": "invalido@test.com",  "activo": True},   # sin id
    {"id": 2,    "email": "CARLOS@Empresa.COM", "activo": False},  # inactivo
    {"id": 3,    "email": "  ELENA@Startup.io ", "activo": True},
    {"id": 4,    "email": "pedro@otra.net",      "activo": True},
    {"id": 5,    "email": "sindominio",           "activo": True},  # email raro
]

print(f"Dataset inicial: {len(usuarios_raw)} registros")
print("Ejecutando pipeline:")
usuarios_limpios = ejecutar_pipeline(
    usuarios_raw,
    filtrar_invalidos,   # Paso 1
    normalizar_emails,   # Paso 2
    filtrar_activos,     # Paso 3
    agregar_dominio,     # Paso 4
)

print("\nResultado final:")
for u in usuarios_limpios:
    print(f"  ID {u['id']}: {u['email']} | dominio: {u['dominio']}")
```

> **Prueba esto:** Crea una nueva función de transformación `def filtrar_por_dominio(registros)` que solo conserve usuarios cuyo dominio sea `"empresa.com"`. Añádela como último paso al pipeline y verifica cuántos usuarios quedan.

---

## 6. Closures — capturar el entorno

### Qué es un closure

Un closure es una función que "recuerda" las variables del ámbito en que fue creada, incluso cuando ese ámbito ya no está activo. Es la base de decoradores, fábricas de funciones y muchos patrones de diseño.

```python
# Fábrica de multiplicadores con closure
def crear_multiplicador(factor: float) -> Callable[[float], float]:
    """
    Retorna una función que multiplica cualquier número por `factor`.
    La función retornada 'recuerda' el valor de factor (closure).
    """
    # factor queda capturado en el closure
    def multiplicar(valor: float) -> float:
        return valor * factor  # Usa factor del scope externo
    return multiplicar

# Cada llamada crea un closure independiente
duplicar    = crear_multiplicador(2)
triplicar   = crear_multiplicador(3)
porcentaje  = crear_multiplicador(0.01)  # Convierte % a proporción

print(f"duplicar(15)    = {duplicar(15)}")     # 30
print(f"triplicar(15)   = {triplicar(15)}")    # 45
print(f"porcentaje(75)  = {porcentaje(75)}")   # 0.75 (75%)

# Caso real: generadores de URLs para diferentes entornos
def crear_url_builder(base_url: str, version: str) -> Callable[[str], str]:
    """Retorna una función que construye URLs para un entorno específico."""
    def construir_url(endpoint: str) -> str:
        return f"{base_url}/api/{version}/{endpoint.lstrip('/')}"
    return construir_url

url_prod = crear_url_builder("https://api.empresa.com", "v2")
url_dev  = crear_url_builder("http://localhost:8080", "v1")

endpoints = ["usuarios", "productos/42", "pedidos?status=activo"]

print("\nURLs de producción:")
for ep in endpoints:
    print(f"  {url_prod(ep)}")

print("\nURLs de desarrollo:")
for ep in endpoints:
    print(f"  {url_dev(ep)}")
```

> **Prueba esto:** Crea un `crear_url_builder` para un entorno de staging con URL `"https://staging.empresa.com"` y versión `"v2-beta"`. Genera las URLs para los mismos endpoints y compara las tres versiones (prod, dev, staging) en paralelo con `zip`.

### Closure como contador o acumulador con estado

```python
# Contador de eventos por categoría usando closures
def crear_contador_eventos(categoria: str, alerta_en: int = 10):
    """
    Retorna un conjunto de funciones que comparten estado (el conteo).
    Este patrón es útil para métricas sin usar clases.
    """
    # Estado encapsulado en el closure
    conteo = [0]  # Lista para poder modificar desde funciones internas
    historial = []

    def registrar(mensaje: str = "") -> int:
        """Incrementa el contador y registra el evento."""
        conteo[0] += 1
        historial.append({"n": conteo[0], "msg": mensaje})
        if conteo[0] == alerta_en:
            print(f"  [ALERTA] Categoría '{categoria}' alcanzó {alerta_en} eventos")
        return conteo[0]

    def obtener_total() -> int:
        """Retorna el conteo actual."""
        return conteo[0]

    def obtener_historial() -> list:
        """Retorna una copia del historial de eventos."""
        return historial.copy()

    def resetear() -> None:
        """Resetea el contador y el historial."""
        conteo[0] = 0
        historial.clear()

    # Retornar las funciones como un "objeto" de closure
    return registrar, obtener_total, obtener_historial, resetear

# Crear contadores independientes por categoría
registrar_error, total_errores, hist_errores, reset_errores = crear_contador_eventos("ERROR", alerta_en=3)
registrar_aviso, total_avisos, hist_avisos, _              = crear_contador_eventos("WARN",  alerta_en=5)

# Simular procesamiento de eventos
eventos = ["ERROR", "WARN", "ERROR", "ERROR", "WARN", "INFO", "ERROR", "WARN"]
for evento in eventos:
    if evento == "ERROR":
        n = registrar_error(f"Error en iteración {total_errores() + 1}")
    elif evento == "WARN":
        registrar_aviso()

print(f"\nTotal errores: {total_errores()}")
print(f"Total avisos:  {total_avisos()}")
print("\nHistorial de errores:")
for entrada in hist_errores():
    print(f"  #{entrada['n']}: {entrada['msg']}")
```

> **Prueba esto:** Crea un tercer contador para la categoría `"CRITICAL"` con `alerta_en=1` (que alerte en el primer evento crítico). Registra 2 eventos críticos y verifica que la alerta solo se dispara en el primero (el que iguala `alerta_en`).

### `nonlocal` — modificar la variable del scope externo

```python
# Sin nonlocal: la variable interna sombrea (oculta) a la externa
def crear_contador_sin_nonlocal():
    cuenta = 0
    def incrementar():
        # cuenta = cuenta + 1  # UnboundLocalError: se interpreta como local
        pass  # Por eso usamos la lista [0] en el ejemplo anterior
    return incrementar

# Con nonlocal: acceso explícito a la variable del scope externo
def crear_contador_con_nonlocal(inicio: int = 0, paso: int = 1):
    """Contador usando nonlocal — más limpio que la lista [0]."""
    cuenta = inicio  # Variable del scope externo

    def incrementar(cantidad: int = 1) -> int:
        nonlocal cuenta  # Declarar que usamos la variable del scope externo
        cuenta += cantidad * paso
        return cuenta

    def decrementar(cantidad: int = 1) -> int:
        nonlocal cuenta
        cuenta -= cantidad * paso
        return cuenta

    def valor_actual() -> int:
        return cuenta  # Solo leer no requiere nonlocal

    return incrementar, decrementar, valor_actual

# Contador de reintentos con paso personalizado
inc, dec, val = crear_contador_con_nonlocal(inicio=0, paso=1)

print("Contador con nonlocal:")
print(f"  Incrementar: {inc()}")   # 1
print(f"  Incrementar: {inc()}")   # 2
print(f"  Incrementar: {inc(3)}")  # 5 (suma 3 pasos)
print(f"  Decrementar: {dec()}")   # 4
print(f"  Valor actual: {val()}")  # 4

# Caso real: rastreador de progreso de carga de archivos
def crear_rastreador_progreso(total_bytes: int):
    """Closure para rastrear el progreso de carga sin estado global."""
    bytes_enviados = 0

    def actualizar(chunk_size: int) -> float:
        nonlocal bytes_enviados
        bytes_enviados += chunk_size
        porcentaje = (bytes_enviados / total_bytes) * 100
        return min(porcentaje, 100.0)  # No superar 100%

    def completado() -> bool:
        return bytes_enviados >= total_bytes

    return actualizar, completado

# Simular carga de un archivo de 1MB en chunks
progreso, esta_listo = crear_rastreador_progreso(1_048_576)  # 1MB en bytes

chunks = [262_144, 262_144, 262_144, 262_144]  # 4 chunks de 256KB
for i, chunk in enumerate(chunks, 1):
    pct = progreso(chunk)
    print(f"  Chunk {i}: {pct:.1f}% completado")

print(f"  Carga completada: {esta_listo()}")
```

> **Prueba esto:** Elimina la línea `nonlocal bytes_enviados` del closure `actualizar`. ¿Qué error obtienes al intentar hacer `bytes_enviados += chunk_size`? ¿Por qué Python no puede resolver la referencia sin `nonlocal`?

---

## 7. Generadores — `yield`

### Por qué usar generadores

Un generador produce valores de uno en uno, bajo demanda, sin cargar toda la secuencia en memoria. Ideal para archivos grandes, streams, pipelines de datos infinitos o costosos de computar.

```python
from datetime import date, timedelta
from typing import Generator

# Generador de rangos de fechas
def rango_fechas(
    fecha_inicio: date,
    fecha_fin: date,
    paso_dias: int = 1
) -> Generator[date, None, None]:
    """
    Genera fechas entre inicio y fin (inclusive) con paso variable.
    No crea la lista completa — genera cada fecha al ser pedida.
    """
    fecha_actual = fecha_inicio
    while fecha_actual <= fecha_fin:
        yield fecha_actual  # Suspende aquí y retorna la fecha
        fecha_actual += timedelta(days=paso_dias)

# Iterar sobre un rango de fechas sin crear lista
inicio = date(2024, 1, 1)
fin    = date(2024, 1, 10)

print("Fechas para informe semanal:")
for fecha in rango_fechas(inicio, fin, paso_dias=2):
    print(f"  {fecha.strftime('%Y-%m-%d (%A)')}")

# Generador de chunks para procesar archivos grandes
def chunks_de_archivo(
    datos: list,          # En producción sería un archivo real
    tamanio_chunk: int
) -> Generator[list, None, None]:
    """
    Divide una secuencia en chunks del tamaño especificado.
    Útil para procesar archivos grandes sin cargarlos en memoria.
    """
    total = len(datos)
    for inicio in range(0, total, tamanio_chunk):
        fin = min(inicio + tamanio_chunk, total)
        yield datos[inicio:fin]  # Retorna un chunk a la vez

# Simular procesamiento de 10,000 registros en chunks
registros = list(range(1, 10_001))  # Simula datos de un archivo
CHUNK_SIZE = 2_500

print(f"\nProcesando {len(registros)} registros en chunks de {CHUNK_SIZE}:")
for numero_chunk, chunk in enumerate(chunks_de_archivo(registros, CHUNK_SIZE), start=1):
    suma = sum(chunk)
    print(f"  Chunk {numero_chunk}: registros {chunk[0]}-{chunk[-1]} | suma: {suma:,}")

# Verificar que el generador es lazy: next() para obtener un valor a la vez
gen = rango_fechas(date(2024, 6, 1), date(2024, 12, 31), paso_dias=30)
print(f"\nPrimer valor del generador: {next(gen)}")
print(f"Segundo valor              : {next(gen)}")
print(f"(El resto aún no se ha calculado)")
```

> **Prueba esto:** Cambia `tamanio_chunk` a 3000 en la llamada a `chunks_de_archivo`. ¿Cuántos chunks se generan ahora? ¿El último chunk tiene exactamente 3000 elementos? Calcula cuántos tiene.

### `yield from` — delegar a otro generador

```python
import os
from pathlib import Path

# Generador que recorre archivos en un directorio y subdirectorios
def archivos_en_directorio(
    directorio: str,
    extensiones: tuple[str, ...] = (".py", ".md", ".txt")
) -> Generator[Path, None, None]:
    """
    Recorre recursivamente un directorio y genera rutas de archivos
    que coincidan con las extensiones dadas.
    """
    ruta = Path(directorio)

    for elemento in ruta.iterdir():
        if elemento.is_file() and elemento.suffix in extensiones:
            yield elemento  # Generar archivo directamente
        elif elemento.is_dir():
            # yield from delega al generador del subdirectorio
            # Equivale a: for archivo in archivos_en_directorio(...): yield archivo
            yield from archivos_en_directorio(str(elemento), extensiones)

# Generador compuesto usando yield from
def todos_los_archivos_python(*directorios: str) -> Generator[Path, None, None]:
    """Combina generadores de múltiples directorios con yield from."""
    for directorio in directorios:
        if Path(directorio).exists():
            yield from archivos_en_directorio(directorio, extensiones=(".py",))

# Ejemplo con yield from en estructura de datos anidada
def aplanar(coleccion) -> Generator:
    """Aplana una estructura anidada arbitrariamente profunda."""
    for elemento in coleccion:
        if isinstance(elemento, (list, tuple)):
            yield from aplanar(elemento)  # Recursión con yield from
        else:
            yield elemento

# Aplanar una estructura de alertas anidadas
alertas_anidadas = [
    "CRITICAL: caída DB",
    ["ERROR: timeout web-01", "ERROR: timeout web-02"],
    [
        "WARN: alto uso CPU",
        ["INFO: reinicio programado", "INFO: backup completado"],
    ],
]

print("Alertas aplanadas:")
for alerta in aplanar(alertas_anidadas):
    print(f"  {alerta}")
```

> **Prueba esto:** Modifica la función `aplanar` para que también aplane `set` (conjuntos) y `dict` (iterando sobre las claves). Prueba con `aplanar([{"a", "b"}, [1, 2], {"x": 1, "y": 2}])`.

### `send()` para pasar valores al generador

```python
# Generador de estadísticas acumulativas que recibe nuevos valores con send()
def acumulador_estadisticas() -> Generator[dict, float, str]:
    """
    Generador coroutine que acumula estadísticas de métricas en tiempo real.

    Uso:
        gen = acumulador_estadisticas()
        next(gen)          # Inicializar (avanzar al primer yield)
        stats = gen.send(45.2)  # Enviar valor y recibir estadísticas
    """
    # Estado interno del generador
    total = 0.0
    cantidad = 0
    minimo = float("inf")
    maximo = float("-inf")

    while True:
        # yield expresa dos cosas:
        # 1. Retorna las estadísticas actuales al caller
        # 2. Recibe el nuevo valor enviado con send()
        nuevo_valor = yield {
            "count": cantidad,
            "suma": round(total, 2),
            "promedio": round(total / cantidad, 2) if cantidad > 0 else 0,
            "min": round(minimo, 2) if minimo != float("inf") else None,
            "max": round(maximo, 2) if maximo != float("-inf") else None,
        }

        if nuevo_valor is None:
            break  # Terminar si se envía None

        # Actualizar estadísticas con el nuevo valor
        total += nuevo_valor
        cantidad += 1
        minimo = min(minimo, nuevo_valor)
        maximo = max(maximo, nuevo_valor)

# Uso del generador con send()
metricas_stream = [45.2, 78.9, 23.1, 91.5, 56.7, 12.3, 88.4, 34.6]

gen = acumulador_estadisticas()
next(gen)  # Inicializar: avanzar hasta el primer yield

print("Acumulando métricas en tiempo real:")
print(f"{'Valor':<10} {'Count':<8} {'Promedio':<12} {'Min':<10} {'Max':<10}")
print("-" * 50)

for metrica in metricas_stream:
    stats = gen.send(metrica)  # Enviar valor y recibir estadísticas actualizadas
    print(
        f"{metrica:<10.1f} "
        f"{stats['count']:<8} "
        f"{stats['promedio']:<12.2f} "
        f"{stats['min']:<10.1f} "
        f"{stats['max']:<10.1f}"
    )

gen.close()  # Limpiar el generador
```

> **Prueba esto:** Añade una estadística adicional al diccionario que retorna el generador: `"desviacion_estimada"` calculada como `maximo - minimo`. ¿Cómo cambia el rango conforme llegan más métricas?

---

## Ejemplo aplicado — pipeline de procesamiento de datos con generadores

```python
# Pipeline lazy que procesa un CSV línea a línea sin cargarlo completo en memoria
# Cada etapa es un generador que consume el generador anterior

import csv
import io
from typing import Generator, Iterator

# Simular un CSV de métricas de servidores (en producción sería un archivo real)
CSV_SIMULADO = """timestamp,servidor,cpu,ram,disco,latencia_ms
2024-01-15T10:00:01,web-01,45.2,78.1,55.0,23
2024-01-15T10:00:01,web-02,91.8,82.3,60.1,145
2024-01-15T10:00:01,db-01,23.4,91.5,88.2,8
2024-01-15T10:00:02,web-01,47.1,78.5,55.0,25
2024-01-15T10:00:02,web-02,,82.1,60.2,130
2024-01-15T10:00:02,cache-01,12.3,45.6,30.0,3
2024-01-15T10:00:03,web-01,INVALIDO,79.0,55.1,22
2024-01-15T10:00:03,db-01,88.9,92.1,88.5,12
2024-01-15T10:00:03,web-02,95.2,83.0,60.3,201
2024-01-15T10:00:04,cache-01,11.8,44.9,30.0,2
"""

# ETAPA 1: Fuente de datos — leer líneas del CSV (generador)
def leer_csv(contenido: str) -> Generator[dict, None, None]:
    """Lee un CSV y genera filas como diccionarios (lazy)."""
    lector = csv.DictReader(io.StringIO(contenido))
    for fila in lector:
        yield fila  # Una fila a la vez — sin cargar todo el archivo

# ETAPA 2: Filtrado — eliminar filas con datos faltantes o inválidos
def filtrar_invalidos(filas: Iterator[dict]) -> Generator[dict, None, None]:
    """Descarta filas con campos vacíos o valores no numéricos en métricas."""
    campos_numericos = ["cpu", "ram", "disco", "latencia_ms"]
    descartadas = 0
    for fila in filas:
        valida = True
        for campo in campos_numericos:
            valor = fila.get(campo, "").strip()
            if not valor:
                print(f"  [FILTRO] Descartando {fila['servidor']} — campo '{campo}' vacío")
                valida = False
                break
            try:
                float(valor)  # Verificar que es numérico
            except ValueError:
                print(f"  [FILTRO] Descartando {fila['servidor']} — '{campo}'='{valor}' no es número")
                valida = False
                break
        if valida:
            yield fila

# ETAPA 3: Transformación — convertir tipos y añadir campos calculados
def transformar(filas: Iterator[dict]) -> Generator[dict, None, None]:
    """Convierte strings a tipos numéricos y añade campos calculados."""
    for fila in filas:
        yield {
            "timestamp": fila["timestamp"],
            "servidor":  fila["servidor"],
            "cpu":       float(fila["cpu"]),
            "ram":       float(fila["ram"]),
            "disco":     float(fila["disco"]),
            "latencia":  int(fila["latencia_ms"]),
            # Campo calculado: nivel de carga compuesto
            "carga_total": round((float(fila["cpu"]) + float(fila["ram"])) / 2, 1),
            "alerta":    float(fila["cpu"]) > 90 or float(fila["ram"]) > 90,
        }

# ETAPA 4: Filtrar solo servidores bajo alta carga para alertas
def filtrar_alta_carga(filas: Iterator[dict], umbral: float = 80.0) -> Generator[dict, None, None]:
    """Pasa solo las filas donde la carga total supera el umbral."""
    for fila in filas:
        if fila["carga_total"] >= umbral:
            yield fila

# EJECUTAR EL PIPELINE COMPLETO (todo es lazy hasta que se consume)
print("=" * 60)
print("PIPELINE DE PROCESAMIENTO DE MÉTRICAS (lazy)")
print("=" * 60)

# Construir el pipeline: encadenar generadores
etapa_1 = leer_csv(CSV_SIMULADO)
etapa_2 = filtrar_invalidos(etapa_1)
etapa_3 = transformar(etapa_2)
etapa_4 = filtrar_alta_carga(etapa_3, umbral=80.0)

# Acumular estadísticas finales al consumir el pipeline
print("\nServidores bajo alta carga:")
print(f"{'Timestamp':<22} {'Servidor':<12} {'CPU%':<8} {'RAM%':<8} {'Carga':<8} {'Alerta'}")
print("-" * 70)

total_alertas = 0
registros_procesados = 0

for registro in etapa_4:  # Solo aquí se ejecuta el pipeline completo
    registros_procesados += 1
    if registro["alerta"]:
        total_alertas += 1
    marca_alerta = "(!)" if registro["alerta"] else "   "
    print(
        f"{registro['timestamp']:<22} "
        f"{registro['servidor']:<12} "
        f"{registro['cpu']:<8.1f} "
        f"{registro['ram']:<8.1f} "
        f"{registro['carga_total']:<8.1f} "
        f"{marca_alerta}"
    )

print("-" * 70)
print(f"Registros en alta carga: {registros_procesados} | Alertas críticas: {total_alertas}")
print("\nVentaja del pipeline lazy: el CSV nunca se cargó completo en memoria.")
print("Cada registro se procesó una sola vez, pasando por todas las etapas.")
```

> **Prueba esto:** Añade una quinta etapa al pipeline: un generador `def agregar_recomendacion(filas)` que añada un campo `"accion"` con el valor `"escalar"` si `cpu > 90`, `"monitorear"` si `carga_total > 85`, o `"ok"` en otro caso. Insértala entre la etapa 3 y la etapa 4.

---

## Ejercicios propuestos

**Ejercicio 1 — Función de reporte flexible con `*args` y `**kwargs`**
Escribe una función `generar_reporte(titulo, *secciones, formato="texto", **metadatos)` que reciba un título, un número variable de secciones de contenido, el formato de salida y metadatos opcionales (autor, fecha, versión). La función debe imprimir el reporte formateado usando todos los parámetros.

**Ejercicio 2 — Sistema de caché con closures**
Implementa una función `crear_cache(max_entradas=100)` que retorne tres funciones: `guardar(clave, valor)`, `obtener(clave)` y `limpiar()`. Usa un closure para encapsular el diccionario de caché. Añade lógica para eliminar la entrada más antigua cuando se alcanza `max_entradas`.

**Ejercicio 3 — Generador de secuencia de Fibonacci infinita**
Escribe un generador `fibonacci()` que genere la secuencia de Fibonacci indefinidamente (0, 1, 1, 2, 3, 5, 8...). Luego escribe una función `primeros_fibonacci(n)` que use `itertools.islice` para tomar solo los primeros `n` valores. Finalmente, filtra solo los pares usando una comprensión de generador.

**Ejercicio 4 — Pipeline de validación con lambdas**
Crea una función `validar_con_pipeline(valor, *validadores)` donde cada validador es una función que recibe un valor y retorna `(bool, str)` — si es válido y el mensaje de error. Define 3 validadores usando lambda para: longitud mínima, solo caracteres alfanuméricos, y no empezar con número. Úsalos para validar nombres de usuario.

---

## Resumen de la página 6

- **`*args`** empaqueta argumentos posicionales variables en una tupla; `*` en la llamada desempaca listas/tuplas en argumentos individuales.
- **`**kwargs`** empaqueta argumentos de palabra clave variables en un diccionario; `**` en la llamada desempaca diccionarios en kwargs.
- El orden correcto de parámetros es: `positional_only /`, normales, `*args`, `keyword_only`, `**kwargs`. El `/` y `*` como separadores dan control preciso sobre cómo se pueden pasar los argumentos.
- **Lambda** es apropiada para expresiones de una línea en `key=`, `map()` y `filter()`; para lógica compleja, las comprensiones o funciones `def` son más legibles.
- Las **funciones de orden superior** (que reciben o retornan funciones) permiten construir pipelines de transformación flexibles y reutilizables.
- Un **closure** captura las variables del ámbito donde fue creado; `nonlocal` permite modificarlas desde la función interna, a diferencia de la lectura que no lo requiere.
- Los **generadores** con `yield` producen valores bajo demanda sin cargar todo en memoria; son la herramienta correcta para procesar streams, archivos grandes y pipelines de datos.
- **`yield from`** delega la generación a otro iterable o generador, simplificando la recursión y composición de generadores.
- **`send()`** convierte un generador en una coroutine bidireccional capaz de recibir valores del caller en cada iteración.

---

> **Siguiente página →** Página 7: Colecciones — list, tuple, dict, set y comprensiones completas.
