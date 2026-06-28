# Tutorial Python — Página 2
## Módulo 1 · Entorno y Fundamentos
### Condicionales — simple, dos caminos, múltiples y anidadas

---

Los condicionales son la primera herramienta de control de flujo: permiten que el
programa tome decisiones en función del estado del sistema. Python usa indentación
de 4 espacios (nunca tabs) para delimitar bloques — no hay llaves.

Esta página cubre únicamente `if`. El `match/case` se trata en la página 3.

---

## 1. Condicional simple — solo `if`, sin `else`

Un `if` sin `else` no hace nada cuando la condición es falsa. Es la forma correcta
cuando la acción es **opcional o secundaria**: añadir un log, registrar un evento,
acumular en una lista. No cambia el flujo principal del programa.

```python
# Registrar advertencia cuando el uso de CPU supera el umbral
uso_cpu: float = 87.4   # porcentaje de uso en el último segundo

if uso_cpu > 80.0:
    print(f"[WARN] CPU al {uso_cpu:.1f}% — monitorear de cerca")

# El programa continúa aquí independientemente de la condición
print("Ciclo de monitoreo completado")
```

> **Prueba esto:** Cambia el valor de `uso_cpu` a `60.0` y ejecuta. ¿Qué líneas se imprimen? Luego prueba con `80.0` exacto — ¿entra o no entra en el `if`?

Más patrones de `if` simple — las tres razones más comunes para usarlo:

```python
# --- Caso 1: añadir a una lista de alertas (acumulador opcional) ---
alertas: list[str] = []
latencia_ms: float = 340.0
error_rate: float = 2.5
espacio_disco_pct: float = 91.0

if latencia_ms > 300:
    alertas.append(f"Latencia elevada: {latencia_ms:.0f}ms")

if error_rate > 1.0:
    alertas.append(f"Tasa de errores: {error_rate:.1f}%")

if espacio_disco_pct > 90:
    alertas.append(f"Disco casi lleno: {espacio_disco_pct:.1f}% usado")

# Solo imprimir si hay algo que reportar
if alertas:
    print("=== ALERTAS ACTIVAS ===")
    for alerta in alertas:
        print(f"  · {alerta}")

# --- Caso 2: logging condicional por nivel ---
NIVEL_LOG = "DEBUG"   # configuración del entorno

mensaje_debug = "Conexión establecida con 10.0.1.5:5432"

if NIVEL_LOG == "DEBUG":
    print(f"[DEBUG] {mensaje_debug}")

# --- Caso 3: efecto colateral — enviar notificación si se supera umbral ---
memoria_libre_mb: int = 128
UMBRAL_CRITICO_MB: int = 256

if memoria_libre_mb < UMBRAL_CRITICO_MB:
    # En producción aquí iría: enviar_alerta_slack(memoria_libre_mb)
    print(f"[CRÍTICO] Memoria libre: {memoria_libre_mb}MB — por debajo de {UMBRAL_CRITICO_MB}MB")
```

> **Prueba esto:** Modifica el código para que también registre una alerta si `latencia_ms` supera los `500ms` con el mensaje `"Latencia crítica"`. ¿Cuántas alertas hay en total ahora?

---

## 2. Condicional de dos caminos — `if/else`

Cuando hay exactamente **dos posibles resultados** y siempre se elige uno, se usa
`if/else`. La garantía es que exactamente una de las dos ramas se ejecuta.

```python
# Clasificar el resultado de un healthcheck HTTP
codigo_respuesta: int = 503
endpoint: str = "https://api.ejemplo.com/health"

if codigo_respuesta < 400:
    # Rama éxito — el servicio responde con código 2xx o 3xx
    estado = "OPERATIVO"
    accion = "continuar monitoreo normal"
else:
    # Rama error — el servicio devuelve 4xx o 5xx
    estado = "DEGRADADO O CAÍDO"
    accion = "escalar al equipo de guardia"

print(f"Servicio {endpoint}")
print(f"  Estado : {estado}")
print(f"  Código : {codigo_respuesta}")
print(f"  Acción : {accion}")
```

> **Prueba esto:** Cambia `codigo_respuesta` a `200` y luego a `404`. ¿Cambia la rama ejecutada en ambos casos? ¿Qué ocurre con `399`?

---

### El operador ternario — `a if condición else b`

Para **asignar un valor en función de una condición** en una sola línea. Úsalo
cuando la expresión es corta y clara. Evítalo cuando la lógica es compleja o
requiere efectos colaterales.

```python
# Cuándo SÍ usar el ternario — asignación simple y legible
latencia_ms: float = 145.0
nivel: str = "LENTO" if latencia_ms > 100 else "RÁPIDO"
print(f"Latencia {latencia_ms}ms → {nivel}")

# Formatear bytes en unidad legible
tamano_bytes: int = 2_097_152   # 2 MB
tamano_str: str = f"{tamano_bytes / 1_048_576:.1f} MiB" if tamano_bytes >= 1_048_576 else f"{tamano_bytes / 1_024:.1f} KiB"
print(f"Tamaño: {tamano_str}")

# Seleccionar entorno de base de datos
entorno: str = "produccion"
db_host: str = "db.produccion.interno" if entorno == "produccion" else "localhost"
print(f"Conectando a: {db_host}")

# Cuándo NO usar el ternario — demasiado anidado, difícil de leer
# MAL: valor = "A" if x > 10 else "B" if x > 5 else "C" if x > 0 else "D"
# BIEN: usar if/elif/elif/else para este caso
valor_x: int = 7
if valor_x > 10:
    etiqueta = "A"
elif valor_x > 5:
    etiqueta = "B"
elif valor_x > 0:
    etiqueta = "C"
else:
    etiqueta = "D"
print(f"Etiqueta para {valor_x}: {etiqueta}")
```

> **Prueba esto:** Reescribe la asignación de `db_host` para que también devuelva `"db.staging.interno"` cuando el entorno sea `"staging"`. ¿Sigue siendo legible como ternario o conviene usar `if/elif/else`?

---

### `if` con `None` check — el patrón correcto

`None` en Python representa la ausencia de valor. Comprobarlo correctamente evita
bugs sutiles.

```python
# Función que puede devolver un valor o None
def buscar_token_sesion(user_id: int) -> str | None:
    """Busca el token en caché. Devuelve None si no existe o expiró."""
    tokens_activos: dict[int, str] = {
        1: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9",
        3: "eyJzdWIiOiIzIiwiaWF0IjoxNzE2MDAwMDAwfQ",
    }
    return tokens_activos.get(user_id)   # .get() devuelve None si no existe


# CORRECTO — comparación explícita con 'is None'
token = buscar_token_sesion(1)
if token is not None:
    print(f"Token encontrado: {token[:20]}...")
else:
    print("Sin token — redirigir al login")

# CORRECTO — para el usuario 99 que no tiene token
token_nuevo = buscar_token_sesion(99)
if token_nuevo is None:
    print("Generando nuevo token para el usuario 99")

# EL PELIGRO DEL OPERADOR 'or' — trata None Y string vacío igual
# Esto puede ocultar bugs si una cadena vacía "" es un valor válido
timeout_configurado: int | None = 0   # timeout de 0 segundos (válido)
timeout_efectivo = timeout_configurado or 30   # BUG: 0 es falsy, usa 30 en vez de 0!
print(f"Timeout (con 'or'): {timeout_efectivo}")   # Imprime 30, incorrecto

# CORRECTO — comprobar None explícitamente con 'is None'
timeout_efectivo_correcto = timeout_configurado if timeout_configurado is not None else 30
print(f"Timeout (correcto): {timeout_efectivo_correcto}")   # Imprime 0, correcto

# TAMBIÉN INCORRECTO — usar == None en vez de 'is None'
# None es un singleton; 'is' compara identidad, '==' compara valor
# Algunos objetos sobreescriben __eq__ y pueden devolver True al comparar con None
# PEP 8 exige 'is None' / 'is not None'
if token is not None:   # siempre esta forma, no: token != None
    print("Token válido")
```

> **Prueba esto:** Llama a `buscar_token_sesion(3)` y comprueba el token. Luego llama con `user_id=0`. ¿Es el resultado `None` o un token? Explica por qué `tokens_activos.get(0)` devuelve `None` aunque el usuario `0` no exista explícitamente en el diccionario.

---

## 3. Condicionales múltiples — `if/elif/elif/else`

Cuando hay más de dos caminos posibles. Python evalúa las condiciones de arriba
a abajo y entra **solo en la primera rama cuya condición sea verdadera**. El `else`
final actúa como captura de todo lo que no coincidió.

```python
# Clasificar código de respuesta HTTP por familia
def clasificar_codigo_http(codigo: int) -> tuple[str, str]:
    """Devuelve (categoría, descripción) para un código HTTP."""

    if codigo < 100 or codigo > 599:
        # Fuera del rango estándar HTTP
        categoria = "INVÁLIDO"
        descripcion = f"El código {codigo} no es un código HTTP estándar"
    elif codigo < 200:
        # 1xx — Respuestas informativas
        categoria = "INFORMATIVO"
        descripcion = "Solicitud recibida, procesando"
    elif codigo < 300:
        # 2xx — Éxito
        categoria = "ÉXITO"
        descripcion = "Solicitud completada correctamente"
    elif codigo < 400:
        # 3xx — Redirecciones
        categoria = "REDIRECCIÓN"
        descripcion = "El recurso se ha movido o el cliente debe tomar acción"
    elif codigo < 500:
        # 4xx — Error del cliente
        categoria = "ERROR CLIENTE"
        descripcion = "La solicitud contiene datos incorrectos o el cliente carece de permisos"
    else:
        # 5xx — Error del servidor
        categoria = "ERROR SERVIDOR"
        descripcion = "El servidor falló al procesar una solicitud válida"

    return categoria, descripcion


# Probar con distintos códigos
codigos_prueba = [99, 101, 200, 201, 301, 304, 400, 401, 403, 404, 429, 500, 502, 503, 600]
for codigo in codigos_prueba:
    cat, desc = clasificar_codigo_http(codigo)
    print(f"  {codigo:>3}  [{cat:<14}]  {desc}")
```

> **Prueba esto:** Añade un caso específico para `codigo == 418` (I'm a teapot) entre el bloque 4xx y el else final, con la descripción `"El servidor es una tetera — RFC 2324"`. Verifica que sigue funcionando correctamente con `200`, `404` y `500`.

---

### Truthy y Falsy — tabla de valores que evalúan como `False`

Python no necesita comparaciones explícitas con `True`/`False`/`0`/`""`.
Cualquier valor puede usarse directamente en una condición.

| Valor         | Tipo        | Es Falsy |
|---------------|-------------|----------|
| `False`       | `bool`      | Sí       |
| `None`        | `NoneType`  | Sí       |
| `0`           | `int`       | Sí       |
| `0.0`         | `float`     | Sí       |
| `0j`          | `complex`   | Sí       |
| `""`          | `str`       | Sí       |
| `[]`          | `list`      | Sí       |
| `{}`          | `dict`      | Sí       |
| `set()`       | `set`       | Sí       |
| `()`          | `tuple`     | Sí       |
| `b""`         | `bytes`     | Sí       |
| Todo lo demás | cualquiera  | No (truthy) |

```python
# Demostración de todos los valores falsy
valores_falsy = [False, None, 0, 0.0, 0j, "", [], {}, set(), (), b""]
valores_truthy = [True, 1, -1, 0.001, "0", " ", [0], {0: 0}, {0}, (0,), b"0"]

print("=== VALORES FALSY ===")
for v in valores_falsy:
    resultado = "falsy" if not v else "truthy"   # 'not v' invierte el valor booleano
    print(f"  {repr(v):<12} → {resultado}")

print("\n=== VALORES TRUTHY ===")
for v in valores_truthy:
    resultado = "truthy" if v else "falsy"
    print(f"  {repr(v):<12} → {resultado}")

# Aprovechando truthiness en código real
cola_mensajes: list[str] = []
respuesta_api: dict | None = None
nombre_host: str = ""

# Idiomático: no hace falta '== []' ni 'len(cola_mensajes) == 0'
if not cola_mensajes:
    print("\nCola vacía — esperando mensajes")

# Idiomático para None
if respuesta_api is None:
    print("Sin respuesta de la API aún")

# Idiomático para string vacío
host_efectivo = nombre_host if nombre_host else "localhost"
print(f"Conectando a: {host_efectivo}")
```

> **Prueba esto:** ¿Qué devuelve `bool([0])`? ¿Y `bool([])`? Explica por qué una lista con un elemento `0` es truthy aunque su único elemento sea falsy.

---

### Encadenamiento de comparaciones — `0 <= x < 256`

Python permite encadenar comparaciones matemáticamente: `a < b < c` es equivalente
a `a < b and b < c` pero más legible y sin evaluar `b` dos veces.

```python
# Validar que un valor está en un rango — sin encadenamiento vs con encadenamiento
def validar_byte(valor: int) -> bool:
    """Un byte válido está entre 0 y 255 inclusive."""
    # Sin encadenamiento (verboso)
    # return valor >= 0 and valor <= 255

    # Con encadenamiento (legible, Pythónico)
    return 0 <= valor <= 255


# Validar componentes de una dirección IP
def validar_ip(ip: str) -> bool:
    """Valida el formato básico de una dirección IPv4."""
    partes = ip.split(".")

    # Primero verificar que hay exactamente 4 partes
    if len(partes) != 4:
        return False

    for parte in partes:
        # Verificar que cada parte es numérica
        if not parte.isdigit():
            return False
        octeto = int(parte)
        # Encadenamiento para validar rango de octeto
        if not (0 <= octeto <= 255):
            return False

    return True


def clasificar_temperatura_cpu(temp_celsius: float) -> str:
    """Clasifica la temperatura de CPU con rangos encadenados."""
    if temp_celsius < 0:
        return "SENSOR ERROR — temperatura negativa imposible"
    elif 0 <= temp_celsius < 40:
        return "FRÍA — posible problema de ventilación inverso"
    elif 40 <= temp_celsius < 65:
        return "NORMAL — operación óptima"
    elif 65 <= temp_celsius < 80:
        return "CÁLIDA — monitorear carga de trabajo"
    elif 80 <= temp_celsius < 90:
        return "CALIENTE — reducir carga o mejorar refrigeración"
    else:   # temp_celsius >= 90
        return "CRÍTICA — riesgo de throttling o apagado de emergencia"


# Pruebas
ips_prueba = ["192.168.1.1", "10.0.0.256", "172.16", "0.0.0.0", "255.255.255.255", "abc.def.ghi.jkl"]
for ip in ips_prueba:
    print(f"  {ip:<20} → {'VÁLIDA' if validar_ip(ip) else 'INVÁLIDA'}")

print()
temperaturas = [-5.0, 25.0, 52.0, 71.3, 85.0, 94.7]
for temp in temperaturas:
    print(f"  {temp:>6.1f}°C → {clasificar_temperatura_cpu(temp)}")
```

> **Prueba esto:** Python también permite `1 < 2 < 3 < 4 < 5`. Evalúa `1 < 2 > 0` en el intérprete — ¿da `True` o `False`? ¿Por qué? Hint: `1 < 2` es True y `2 > 0` es True.

---

## 4. Condicionales anidadas — `if` dentro de `if`

Los condicionales anidados tienen sentido cuando las condiciones son **dependientes
en secuencia**: primero comprobar si un objeto existe, luego si tiene permisos,
luego si la IP está en lista blanca. Sin embargo, más de 2-3 niveles de anidamiento
suelen indicar que el código necesita refactorización.

```python
# Caso real: validar acceso a un recurso de API
# Pregunta 1: ¿está activa la cuenta?
# Pregunta 2: ¿tiene el permiso requerido?
# Pregunta 3: ¿la IP está en la lista blanca?

def verificar_acceso_anidado(
    cuenta_activa: bool,
    permisos: list[str],
    permiso_requerido: str,
    ip_cliente: str,
    ips_permitidas: list[str],
) -> str:
    """Versión con anidamiento explícito — útil cuando hay lógica específica en cada nivel."""

    if cuenta_activa:
        # Solo verificamos permisos si la cuenta está activa
        if permiso_requerido in permisos:
            # Solo verificamos la IP si el permiso es correcto
            if ip_cliente in ips_permitidas:
                return "ACCESO CONCEDIDO"
            else:
                return f"ACCESO DENEGADO — IP {ip_cliente} no permitida"
        else:
            return f"ACCESO DENEGADO — permiso '{permiso_requerido}' no asignado"
    else:
        return "ACCESO DENEGADO — cuenta inactiva"


# Pruebas del validador anidado
casos = [
    (False, ["lectura"],        "escritura", "10.0.0.5",  ["10.0.0.1", "10.0.0.5"]),
    (True,  ["lectura"],        "escritura", "10.0.0.5",  ["10.0.0.1", "10.0.0.5"]),
    (True,  ["lectura", "escritura"], "escritura", "10.0.1.99", ["10.0.0.1", "10.0.0.5"]),
    (True,  ["lectura", "escritura"], "escritura", "10.0.0.1",  ["10.0.0.1", "10.0.0.5"]),
]

for activa, perms, perm_req, ip, ips_ok in casos:
    resultado = verificar_acceso_anidado(activa, perms, perm_req, ip, ips_ok)
    print(f"  {resultado}")
```

> **Prueba esto:** Añade un cuarto nivel de anidamiento que verifique que `ip_cliente` no esté en una lista de IPs bloqueadas `ips_bloqueadas = ["10.0.0.99"]`. ¿A qué nivel de profundidad queda el código? ¿Sigue siendo fácil de leer?

---

### `and` en una línea vs anidadas — cuándo cada forma es más legible

```python
# La misma lógica escrita de dos formas distintas

# --- FORMA 1: con 'and' en una sola condición ---
# Ventaja: compacto, una sola expresión booleana
# Desventaja: el mensaje de error es genérico (no sabe qué condición falló)

def verificar_acceso_and(
    cuenta_activa: bool,
    permisos: list[str],
    permiso_requerido: str,
    ip_cliente: str,
    ips_permitidas: list[str],
) -> str:
    """Con 'and' — para cuando no necesitas saber qué condición falló."""
    if cuenta_activa and permiso_requerido in permisos and ip_cliente in ips_permitidas:
        return "ACCESO CONCEDIDO"
    else:
        return "ACCESO DENEGADO"   # no sabe qué condición específica falló


# --- FORMA 2: anidadas — para cuando el mensaje de error es importante ---
# La versión anidada de arriba permite devolver el motivo exacto del rechazo.

# REGLA DE THUMB:
#   - Usar 'and' cuando solo te importa el resultado final (True/False)
#   - Usar anidamiento cuando necesitas saber qué condición específica falló
#   - Usar guard clauses (ver abajo) cuando quieres evitar el anidamiento
#     pero aún necesitas mensajes específicos

print("Con 'and' (genérico):")
print(verificar_acceso_and(True, ["lectura"], "escritura", "10.0.0.1", ["10.0.0.1"]))
print(verificar_acceso_and(True, ["escritura"], "escritura", "10.0.0.1", ["10.0.0.1"]))
```

> **Prueba esto:** ¿Cuál de las dos formas usarías en un sistema de seguridad donde el log de auditoría debe registrar el motivo exacto de cada acceso denegado? ¿Y en una función auxiliar que solo necesita devolver `True`/`False`?

---

### Guard clauses — evitar el anidamiento profundo

La técnica de **guard clause** (también llamada *early return*) invierte las
condiciones: en vez de anidar el camino feliz hacia adentro, se retorna pronto
cuando la condición no se cumple. El resultado es código plano y más legible.

```python
# VERSIÓN CON ANIDAMIENTO PROFUNDO — el camino feliz está enterrado
def procesar_solicitud_anidada(solicitud: dict | None) -> dict:
    if solicitud is not None:
        if "token" in solicitud:
            if len(solicitud["token"]) >= 32:
                if "payload" in solicitud:
                    # El código útil está al fondo del árbol de if
                    datos = solicitud["payload"]
                    return {"estado": "ok", "datos_procesados": len(str(datos))}
                else:
                    return {"estado": "error", "mensaje": "Falta payload"}
            else:
                return {"estado": "error", "mensaje": "Token demasiado corto"}
        else:
            return {"estado": "error", "mensaje": "Falta token"}
    else:
        return {"estado": "error", "mensaje": "Solicitud nula"}


# VERSIÓN CON GUARD CLAUSES — el camino feliz fluye naturalmente al fondo
def procesar_solicitud_guard(solicitud: dict | None) -> dict:
    """Versión limpia con guard clauses — fácil de leer de arriba abajo."""

    # Guard 1: la solicitud no puede ser None
    if solicitud is None:
        return {"estado": "error", "mensaje": "Solicitud nula"}

    # Guard 2: debe tener un token
    if "token" not in solicitud:
        return {"estado": "error", "mensaje": "Falta token"}

    # Guard 3: el token debe tener la longitud mínima de seguridad
    if len(solicitud["token"]) < 32:
        return {"estado": "error", "mensaje": "Token demasiado corto"}

    # Guard 4: debe incluir payload
    if "payload" not in solicitud:
        return {"estado": "error", "mensaje": "Falta payload"}

    # Camino feliz — llegamos aquí solo si todo está correcto
    datos = solicitud["payload"]
    return {"estado": "ok", "datos_procesados": len(str(datos))}


# Probar ambas versiones con los mismos casos
solicitudes_prueba = [
    None,
    {"token": "abc"},                                          # token corto
    {"token": "a" * 32},                                      # falta payload
    {"token": "b" * 32, "payload": {"usuario": 42}},          # correcto
]

print("=== Resultados de guard clauses ===")
for sol in solicitudes_prueba:
    resultado = procesar_solicitud_guard(sol)
    print(f"  Entrada: {str(sol)[:40]!r:>42}  →  {resultado}")
```

> **Prueba esto:** Añade un quinto guard que verifique que `solicitud["payload"]` no sea un diccionario vacío `{}`. El mensaje de error debe ser `"Payload vacío"`. ¿Cuántas líneas añades? Compara con cuántas líneas añadirías en la versión anidada.

---

## Ejemplo aplicado — clasificador de alertas de sistema

Un sistema de monitoreo real recibe métricas de infraestructura cada 30 segundos
y debe determinar el nivel de alerta y la acción a tomar. Usa todos los tipos
de condicionales vistos en esta página.

```python
"""
alertas_sistema.py
Sistema de clasificación de alertas de infraestructura.
Recibe métricas de CPU, memoria y latencia y determina nivel de alerta.
"""
from dataclasses import dataclass, field


# Niveles de alerta ordenados de menor a mayor severidad
NIVELES_ALERTA = ["OK", "AVISO", "ADVERTENCIA", "CRÍTICO", "EMERGENCIA"]


@dataclass
class MetricasSistema:
    """Snapshot de métricas de un nodo en un momento dado."""
    nombre_nodo:    str
    cpu_pct:        float          # Porcentaje de uso de CPU (0-100)
    memoria_pct:    float          # Porcentaje de memoria usada (0-100)
    latencia_ms:    float          # Latencia promedio de red en ms
    disco_pct:      float          # Porcentaje de disco usado (0-100)
    procesos_zombie: int           # Número de procesos en estado zombie
    errores_ultimo_minuto: int     # Errores de aplicación en el último minuto


@dataclass
class ResultadoAlerta:
    """Resultado del análisis de un conjunto de métricas."""
    nivel:      str
    problemas:  list[str] = field(default_factory=list)
    acciones:   list[str] = field(default_factory=list)


def analizar_cpu(cpu_pct: float, problemas: list[str], acciones: list[str]) -> str:
    """Analiza el uso de CPU y devuelve el nivel de alerta para esta métrica."""

    # Guard clause: valor imposible
    if not (0 <= cpu_pct <= 100):
        problemas.append(f"CPU: valor de sensor inválido ({cpu_pct}%)")
        return "EMERGENCIA"

    # Condicionales múltiples para clasificar el nivel
    if cpu_pct >= 95:
        problemas.append(f"CPU al {cpu_pct:.1f}% — sistema al límite")
        acciones.append("Terminar procesos no críticos inmediatamente")
        return "EMERGENCIA"
    elif cpu_pct >= 85:
        problemas.append(f"CPU al {cpu_pct:.1f}% — uso muy elevado")
        acciones.append("Revisar procesos consumidores con 'top' o 'htop'")
        return "CRÍTICO"
    elif cpu_pct >= 70:
        problemas.append(f"CPU al {cpu_pct:.1f}% — uso elevado")
        acciones.append("Monitorear si la tendencia continúa subiendo")
        return "ADVERTENCIA"
    elif cpu_pct >= 50:
        # Solo registrar aviso, sin acción inmediata requerida
        if cpu_pct > 60:
            problemas.append(f"CPU al {cpu_pct:.1f}% — uso moderado-alto")
        return "AVISO"
    else:
        return "OK"


def analizar_memoria(memoria_pct: float, problemas: list[str], acciones: list[str]) -> str:
    """Analiza el uso de memoria con encadenamiento de comparaciones."""

    if not (0 <= memoria_pct <= 100):
        problemas.append(f"Memoria: valor de sensor inválido ({memoria_pct}%)")
        return "EMERGENCIA"

    # Encadenamiento de comparaciones para rangos claros
    if memoria_pct >= 95:
        problemas.append(f"Memoria al {memoria_pct:.1f}% — OOM inminente")
        acciones.append("Reiniciar servicios con fugas de memoria")
        acciones.append("Activar swap de emergencia si disponible")
        return "EMERGENCIA"
    elif 85 <= memoria_pct < 95:
        problemas.append(f"Memoria al {memoria_pct:.1f}% — margen crítico")
        acciones.append("Identificar procesos con mayor consumo de RAM")
        return "CRÍTICO"
    elif 70 <= memoria_pct < 85:
        problemas.append(f"Memoria al {memoria_pct:.1f}% — uso elevado")
        return "ADVERTENCIA"
    else:
        return "OK"


def analizar_latencia(latencia_ms: float, problemas: list[str], acciones: list[str]) -> str:
    """Analiza la latencia de red con guard clause inicial."""

    # Guard: latencia negativa es señal de error de medición
    if latencia_ms < 0:
        problemas.append("Latencia negativa — error en el sensor de red")
        return "EMERGENCIA"

    if latencia_ms > 2000:
        problemas.append(f"Latencia {latencia_ms:.0f}ms — servicio prácticamente inaccesible")
        acciones.append("Verificar estado del switch y rutas de red")
        return "EMERGENCIA"
    elif latencia_ms > 500:
        problemas.append(f"Latencia {latencia_ms:.0f}ms — degradación severa")
        acciones.append("Revisar saturación de ancho de banda")
        return "CRÍTICO"
    elif latencia_ms > 150:
        problemas.append(f"Latencia {latencia_ms:.0f}ms — por encima del umbral")
        return "ADVERTENCIA"
    elif latencia_ms > 50:
        return "AVISO"
    else:
        return "OK"


def clasificar_alertas(metricas: MetricasSistema) -> ResultadoAlerta:
    """
    Función principal de clasificación.
    Combina todos los tipos de condicionales de esta página.
    """
    problemas: list[str] = []
    acciones:  list[str] = []

    # --- Analizar cada métrica individualmente ---
    nivel_cpu     = analizar_cpu(metricas.cpu_pct, problemas, acciones)
    nivel_memoria = analizar_memoria(metricas.memoria_pct, problemas, acciones)
    nivel_latencia = analizar_latencia(metricas.latencia_ms, problemas, acciones)

    # --- Verificaciones adicionales con if simple (efectos colaterales) ---
    if metricas.disco_pct > 90:
        problemas.append(f"Disco al {metricas.disco_pct:.1f}% — riesgo de fallo de escritura")
        acciones.append("Rotar y comprimir logs antiguos")

    if metricas.procesos_zombie > 0:
        # if simple: solo actuar si hay zombies, sin else
        problemas.append(f"{metricas.procesos_zombie} proceso(s) zombie detectado(s)")
        if metricas.procesos_zombie > 5:
            # Anidado justificado: la severidad depende de cuántos zombies hay
            acciones.append("URGENTE: posible fuga en gestión de procesos hijos")

    if metricas.errores_ultimo_minuto > 0:
        mensaje = (
            f"{metricas.errores_ultimo_minuto} errores en el último minuto"
            " — CRÍTICO" if metricas.errores_ultimo_minuto > 50 else ""
        )
        problemas.append(mensaje)

    # --- Determinar el nivel global (el más severo de todas las métricas) ---
    todos_los_niveles = [nivel_cpu, nivel_memoria, nivel_latencia]
    nivel_global = max(todos_los_niveles, key=lambda n: NIVELES_ALERTA.index(n))

    # Acción global según nivel — condicional múltiple
    if nivel_global == "EMERGENCIA":
        acciones.insert(0, "NOTIFICAR AL EQUIPO DE GUARDIA INMEDIATAMENTE")
    elif nivel_global == "CRÍTICO":
        acciones.insert(0, "Abrir incidente P1 en el sistema de tickets")
    elif nivel_global == "ADVERTENCIA":
        acciones.insert(0, "Programar revisión en las próximas 2 horas")
    elif nivel_global == "AVISO":
        acciones.insert(0, "Registrar en el log de tendencias — sin acción inmediata")
    else:
        # Sistema sano — solo registrar el check
        pass

    return ResultadoAlerta(nivel=nivel_global, problemas=problemas, acciones=acciones)


def imprimir_alerta(metricas: MetricasSistema, resultado: ResultadoAlerta) -> None:
    """Formatea e imprime el resultado del análisis de alertas."""
    # Ternario para elegir el separador visual según el nivel
    separador = "=" * 60 if resultado.nivel in ("EMERGENCIA", "CRÍTICO") else "-" * 60

    print(separador)
    print(f"NODO: {metricas.nombre_nodo}  |  NIVEL: {resultado.nivel}")
    print(f"  CPU: {metricas.cpu_pct:.1f}%  MEM: {metricas.memoria_pct:.1f}%  "
          f"LATENCIA: {metricas.latencia_ms:.0f}ms  DISCO: {metricas.disco_pct:.1f}%")

    # if simple — solo imprimir sección de problemas si hay algo
    if resultado.problemas:
        print("  Problemas detectados:")
        for problema in resultado.problemas:
            print(f"    · {problema}")

    if resultado.acciones:
        print("  Acciones requeridas:")
        for accion in resultado.acciones:
            print(f"    → {accion}")

    # None check — solo imprimir si no hay problemas y el sistema está sano
    if not resultado.problemas and resultado.nivel == "OK":
        print("  Sistema operando dentro de parámetros normales")


# ── Datos de prueba ──────────────────────────────────────────────────────────

nodos: list[MetricasSistema] = [
    MetricasSistema("web-prod-01",    45.2, 62.1,   28.0, 55.0, 0,  0),
    MetricasSistema("web-prod-02",    88.7, 71.4,   95.0, 70.0, 0,  12),
    MetricasSistema("db-prod-01",     35.0, 91.5,   12.0, 94.0, 0,  0),
    MetricasSistema("worker-prod-01", 96.1, 88.3, 2500.0, 45.0, 7,  87),
    MetricasSistema("cache-prod-01",  22.0, 45.0,   18.0, 30.0, 0,  0),
]

for nodo in nodos:
    resultado = clasificar_alertas(nodo)
    imprimir_alerta(nodo, resultado)
    print()
```

**Salida esperada (fragmento):**

```
============================================================
NODO: web-prod-02  |  NIVEL: CRÍTICO
  CPU: 88.7%  MEM: 71.4%  LATENCIA: 95.0ms  DISCO: 70.0%
  Problemas detectados:
    · CPU al 88.7% — uso muy elevado
    · 12 errores en el último minuto
  Acciones requeridas:
    → Abrir incidente P1 en el sistema de tickets
    → Revisar procesos consumidores con 'top' o 'htop'
```

---

## Ejercicios propuestos

**1. Semáforo de carga de batería**

Escribe la función `def estado_bateria(pct: int, cargando: bool) -> str` que
use `if/elif/else` con comparaciones encadenadas para clasificar:

- `pct < 10` → `"CRÍTICA"`
- `10 <= pct < 30` → `"BAJA"`
- `30 <= pct < 60` → `"MEDIA"`
- `60 <= pct < 90` → `"BUENA"`
- `pct >= 90` → `"COMPLETA"`

Usa el operador ternario para añadir `" (cargando)"` al final si `cargando` es `True`.
Prueba con al menos: `5`, `29`, `60`, `90`, `100` tanto en modo cargando como sin cargar.

**2. Validador de contraseñas con guard clauses**

Escribe `def validar_contrasena(pwd: str | None) -> dict[str, str | bool]` que
devuelva `{"valida": True/False, "motivo": "..."}`. Usa guard clauses para rechazar:

1. `None` → `"La contraseña no puede ser None"`
2. Longitud < 8 → `"Mínimo 8 caracteres"`
3. Sin mayúsculas (usa `any(c.isupper() for c in pwd)`) → `"Falta al menos una mayúscula"`
4. Sin dígitos (usa `any(c.isdigit() for c in pwd)`) → `"Falta al menos un número"`
5. Si pasa todos los guards → `{"valida": True, "motivo": "Contraseña válida"}`

**3. Clasificador de mensajes de log**

Dado el siguiente formato de log: `"2024-01-15 14:32:01 ERROR auth: token inválido"`,
escribe `def analizar_linea_log(linea: str) -> dict` que extraiga el nivel (`ERROR`,
`WARN`, `INFO`, `DEBUG`) y clasifique la acción a tomar. Usa `if` simple para añadir
la línea a una lista de errores críticos solo si el nivel es `ERROR`. Usa ternario para
formatear el nivel en color de consola (`\033[31mERROR\033[0m` para rojo, etc.).

**4. Sistema de tarifas de taxi**

Escribe `def calcular_tarifa(distancia_km: float, hora: int, es_festivo: bool) -> float`
donde la tarifa base es `2.50€` y el precio por km varía:

- De 6h a 22h en día normal: `1.10€/km`
- De 22h a 6h o en festivo: `1.40€/km`
- Si la distancia supera 50km: aplicar un 10% de descuento sobre el total

Usa `if/else` para el turno, `if` simple para el descuento por distancia, y el
ternario para formatear el resultado como `"X.XX€"`. Prueba con al menos 4 casos.

---

## Resumen de la página 2

- El `if` sin `else` es la forma correcta cuando una acción es opcional o secundaria: logging, añadir a listas, ejecutar efectos colaterales. No hace nada cuando la condición es falsa, y eso es exactamente lo que se quiere.
- El `if/else` garantiza que exactamente una de dos ramas se ejecuta. El operador ternario `a if cond else b` es su forma compacta para asignaciones simples — evitarlo cuando la lógica es compleja o requiere efectos colaterales.
- Al comparar con `None`, usar siempre `is None` o `is not None`, nunca `== None`. El operador `or` para valores por defecto falla silenciosamente cuando el valor legítimo es falsy (`0`, `""`, `[]`).
- `if/elif/elif/else` evalúa condiciones en orden y entra solo en la primera rama verdadera. Python evalúa las condiciones de arriba a abajo — colocar los casos más frecuentes primero es una micro-optimización válida.
- Los valores falsy son: `False`, `None`, `0`, `0.0`, `0j`, `""`, `[]`, `{}`, `set()`, `()`, `b""`. Todo lo demás es truthy. La comprobación `if not lista:` es más idiomática que `if lista == []`.
- El encadenamiento de comparaciones `0 <= x < 256` es matemáticamente natural y más legible que `x >= 0 and x <= 255`. Python lo evalúa correctamente como dos comparaciones con `and`.
- Los condicionales anidados (nivel 2) son legítimos cuando las condiciones son dependientes y se necesitan mensajes de error específicos. Más de 2-3 niveles son señal de refactorización.
- Las **guard clauses** (early return) invierten las condiciones de error para retornar pronto, dejando el camino feliz como flujo lineal al fondo. Producen código más plano, más legible y más fácil de testear que el anidamiento profundo.

---

> **Siguiente página →** Página 3: `match/case` — Structural Pattern Matching (Python 3.10+): literales, guards, tipos, desestructuración de diccionarios, secuencias y dataclasses.
