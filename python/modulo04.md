# Tutorial Python — Página 4
## Módulo 1 · Entorno y Fundamentos
### Bucles — `for`, `while`, `break`, `continue` y `else`

Los bucles son el mecanismo que permite ejecutar código repetidamente. Python ofrece dos constructos principales (`for` y `while`) más modificadores de flujo (`break`, `continue`, `else`) que lo distinguen de otros lenguajes.

---

## 1. El bucle `for` — iterar sobre secuencias

### `for elemento in lista:` — la forma idiomática

En Python, `for` no usa un contador interno como en C o Java. Itera directamente sobre cualquier objeto **iterable**: listas, strings, archivos, rangos, generadores, etc.

```python
# Lista de microservicios desplegados en producción
servicios = ["auth-service", "api-gateway", "user-service", "billing-service", "notification-service"]

# Iterar directamente sobre los elementos — no se necesita índice manual
for servicio in servicios:
    print(f"Verificando estado de: {servicio}")

# Iterar sobre un string carácter a carácter
protocolo = "HTTP/1.1"
for caracter in protocolo:
    print(caracter, end=" ")  # Imprime: H T T P / 1 . 1
print()  # Salto de línea final

# Iterar sobre las líneas de un archivo de configuración (simulado)
config_lineas = [
    "host=db.prod.internal",
    "port=5432",
    "max_connections=100",
    "ssl=true",
]

for linea in config_lineas:
    # Ignorar líneas vacías o comentarios
    if not linea or linea.startswith("#"):
        continue
    clave, valor = linea.split("=")
    print(f"  Configuración detectada: {clave.strip()} = {valor.strip()}")
```

> **Prueba esto:** Modifica la lista `servicios` para incluir un servicio con guión bajo (como `"cache_service"`). ¿Qué pasa si además añades una cadena vacía `""` a la lista? ¿Cómo la filtrarías dentro del bucle?

### `range(inicio, fin, paso)` — los 3 argumentos

`range()` genera una secuencia de enteros de forma eficiente (no crea la lista completa en memoria).

```python
# range(fin) — desde 0 hasta fin-1
print("Intentos de conexión:")
for intento in range(5):
    print(f"  Intento #{intento + 1}")

# range(inicio, fin) — inicio incluido, fin excluido
print("\nPuertos en rango de escaneo:")
for puerto in range(8000, 8010):
    print(f"  Escaneando puerto {puerto}")

# range(inicio, fin, paso) — paso positivo o negativo
print("\nCuenta regresiva para reinicio del servicio:")
for segundos in range(10, 0, -1):
    print(f"  Reiniciando en {segundos}s...")
print("  Reiniciando ahora.")

# Caso real: dividir un archivo grande en chunks de 1000 registros
total_registros = 7_500
chunk_size = 1_000

print(f"\nPlan de procesamiento por chunks ({total_registros} registros):")
for inicio in range(0, total_registros, chunk_size):
    fin = min(inicio + chunk_size, total_registros)
    print(f"  Procesando registros [{inicio}:{fin}] — {fin - inicio} registros")

# Paso personalizado: verificar cada N servidores en una granja
servidores_totales = 50
paso_verificacion = 5
print(f"\nVerificación de muestra (cada {paso_verificacion} servidores):")
for idx in range(0, servidores_totales, paso_verificacion):
    print(f"  Verificando servidor #{idx}")
```

> **Prueba esto:** Usa `range()` para generar los números pares entre 2 y 20 (inclusive). Luego modifica el paso para obtener solo los múltiplos de 3 entre 3 y 30.

---

## 2. `enumerate()` y `zip()`

### `enumerate(iterable, start=N)` — índice + valor

`enumerate()` devuelve pares `(índice, valor)` sin necesidad de mantener un contador manual.

```python
# Tabla numerada de servicios con su estado
servicios_estado = [
    ("auth-service", "activo"),
    ("api-gateway", "activo"),
    ("user-service", "degradado"),
    ("billing-service", "activo"),
    ("notification-service", "detenido"),
]

print("=" * 50)
print(f"{'#':<4} {'Servicio':<25} {'Estado':<12}")
print("=" * 50)

# start=1 para que la numeración comience en 1 en lugar de 0
for numero, (servicio, estado) in enumerate(servicios_estado, start=1):
    # Marcar visualmente los servicios problemáticos
    marca = "(*)" if estado != "activo" else "   "
    print(f"{numero:<4} {servicio:<25} {estado:<12} {marca}")

print("=" * 50)
print(f"Total: {len(servicios_estado)} servicios | Problemáticos marcados con (*)")

# enumerate es útil para actualizar elementos en una lista por índice
errores_log = ["timeout en db", "conexión rechazada", "memoria insuficiente"]
for idx, error in enumerate(errores_log):
    errores_log[idx] = f"[ERROR-{idx + 1:03d}] {error.upper()}"

print("\nErrores formateados:")
for error in errores_log:
    print(f"  {error}")
```

> **Prueba esto:** Cambia el `start=1` a `start=100` para que la numeración comience en 100. Luego añade una condición para que solo muestre los servicios cuyo estado sea `"activo"`.

### `zip()` para iterar en paralelo — y `zip_longest()` para listas desiguales

```python
from itertools import zip_longest

# Combinar tres listas relacionadas en un solo bucle
hosts = ["web-01", "web-02", "db-01", "cache-01"]
ips   = ["10.0.1.10", "10.0.1.11", "10.0.2.10", "10.0.3.10"]
roles = ["frontend", "frontend", "base_de_datos", "caché"]

print("Inventario de infraestructura:")
print(f"{'Host':<12} {'IP':<16} {'Rol':<15}")
print("-" * 43)

for host, ip, rol in zip(hosts, ips, roles):
    print(f"{host:<12} {ip:<16} {rol:<15}")

# zip_longest: cuando las listas tienen distinta longitud
# El valor fillvalue rellena los espacios faltantes
versiones_actuales   = ["3.1.0", "2.4.5", "1.0.0"]
versiones_requeridas = ["3.2.0", "2.4.5", "1.1.0", "4.0.0"]  # Una versión extra

print("\nAudit de versiones:")
for actual, requerida in zip_longest(versiones_actuales, versiones_requeridas, fillvalue="N/A"):
    estado = "OK" if actual == requerida else "ACTUALIZAR"
    print(f"  Actual: {actual:<10} Requerida: {requerida:<10} → {estado}")

# zip también sirve para crear dicts desde dos listas
claves = ["timeout", "retries", "ssl", "pool_size"]
valores = [30, 3, True, 10]
config = dict(zip(claves, valores))
print(f"\nConfig generada con zip: {config}")
```

> **Prueba esto:** Añade una quinta IP a la lista `ips` sin añadir el host ni el rol correspondiente. Observa qué pasa con `zip()`. Luego reemplázalo con `zip_longest()` usando `fillvalue="PENDIENTE"`.

---

## 3. `for` con diccionarios

Los diccionarios ofrecen tres vistas iterables: `.items()`, `.keys()` y `.values()`.

```python
# Configuración de una conexión a base de datos
config_db = {
    "host": "postgres.prod.internal",
    "port": 5432,
    "database": "app_production",
    "user": "app_user",
    "password": "***",
    "ssl_mode": "require",
    "connect_timeout": 10,
    "pool_size": 20,
}

# .items() — la forma más común: clave y valor juntos
print("Configuración activa de base de datos:")
for clave, valor in config_db.items():
    # Ocultar valores sensibles en el output
    if "password" in clave or "secret" in clave:
        valor_mostrado = "[REDACTADO]"
    else:
        valor_mostrado = valor
    print(f"  {clave:<20} = {valor_mostrado}")

# .keys() — solo las claves (útil para verificar presencia)
campos_requeridos = {"host", "port", "database", "user"}
claves_presentes = set(config_db.keys())

faltantes = campos_requeridos - claves_presentes
if faltantes:
    print(f"\nFaltan campos requeridos: {faltantes}")
else:
    print("\nTodos los campos requeridos están presentes.")

# .values() — solo los valores (útil para validaciones)
print("\nValidación de tipos en config:")
for clave, valor in config_db.items():
    tipo = type(valor).__name__
    print(f"  {clave:<20}: {tipo}")

# Iterar y construir un nuevo dict filtrado (sin la password)
config_publica = {k: v for k, v in config_db.items() if "password" not in k}
print(f"\nConfig pública (sin secrets): {len(config_publica)} campos")
```

> **Prueba esto:** Añade una clave `"api_secret": "abc123"` al diccionario. Verifica que el filtro de redacción también la oculta. ¿Qué condición necesitas cambiar en el `if`?

---

## 4. El bucle `while`

### `while condición:` — cuándo preferirlo sobre `for`

Usa `while` cuando no sabes de antemano cuántas iteraciones necesitas: esperar una condición externa, reintentos, polling, lectura de streams.

```python
import time
import random

# Sistema de reintentos con backoff exponencial
# (patrón estándar para llamadas a APIs y servicios externos)

def simular_llamada_api(endpoint: str) -> bool:
    """Simula una llamada a API que falla aleatoriamente."""
    # 40% de probabilidad de éxito en cada intento
    return random.random() > 0.6

MAX_INTENTOS = 5
ESPERA_INICIAL = 1  # segundos
FACTOR_BACKOFF = 2  # duplicar la espera en cada fallo

endpoint = "/api/v2/usuarios"
intento = 0
espera = ESPERA_INICIAL
exito = False

print(f"Iniciando llamada a {endpoint} con backoff exponencial...")

while intento < MAX_INTENTOS and not exito:
    intento += 1
    print(f"  Intento {intento}/{MAX_INTENTOS}...", end=" ")

    exito = simular_llamada_api(endpoint)

    if exito:
        print("OK")
    else:
        print(f"FALLO — esperando {espera}s antes del próximo intento")
        if intento < MAX_INTENTOS:
            # En código real: time.sleep(espera)
            print(f"  [simulando sleep de {espera}s]")
            espera *= FACTOR_BACKOFF  # backoff exponencial: 1 → 2 → 4 → 8

if exito:
    print(f"\nConexión establecida en el intento {intento}.")
else:
    print(f"\nNo se pudo conectar tras {MAX_INTENTOS} intentos. Enviando alerta.")
```

> **Prueba esto:** Cambia el `FACTOR_BACKOFF` a 1.5 para un backoff menos agresivo. ¿Cuánto tiempo total de espera acumularía en el peor caso con 5 intentos? Calcula la suma manualmente.

### `while True:` con `break` — el patrón "loop hasta que se cumpla condición"

```python
# Leer y validar entrada hasta recibir un valor válido
# (simulado con una lista de entradas en lugar de input() real)

entradas_simuladas = ["", "abc", "-5", "256", "42"]  # La válida es la última
indice_entrada = 0

def leer_entrada_simulada() -> str:
    """Simula input() del usuario para propósitos del tutorial."""
    global indice_entrada
    valor = entradas_simuladas[indice_entrada % len(entradas_simuladas)]
    indice_entrada += 1
    print(f"  [usuario escribe]: '{valor}'")
    return valor

PUERTO_MIN = 1024
PUERTO_MAX = 65535

print("Sistema de configuración de puerto — ingrese un puerto válido:")

while True:
    entrada = leer_entrada_simulada()

    # Validar que no esté vacío
    if not entrada.strip():
        print("  Error: el campo no puede estar vacío.")
        continue

    # Validar que sea numérico
    if not entrada.isdigit():
        print("  Error: debe ingresar un número entero.")
        continue

    puerto = int(entrada)

    # Validar rango
    if not (PUERTO_MIN <= puerto <= PUERTO_MAX):
        print(f"  Error: el puerto debe estar entre {PUERTO_MIN} y {PUERTO_MAX}.")
        continue

    # Si llegamos aquí, el valor es válido
    print(f"  Puerto {puerto} aceptado.")
    break

print(f"\nConfiguración guardada: puerto = {puerto}")
```

> **Prueba esto:** Modifica la lista `entradas_simuladas` para que la primera entrada válida sea `"8080"`. Añade también una validación que rechace puertos bien conocidos (menores a 1024) con un mensaje específico.

---

## 5. `break` — salir del bucle

`break` termina inmediatamente el bucle más cercano (no los externos). Se usa cuando ya se encontró lo que se buscaba o cuando se detecta una condición de error crítica.

```python
# Buscar el primer servidor disponible en una lista de backends
# (patrón de "circuit breaker" / balanceador simple)

servidores_backend = [
    {"host": "backend-01", "disponible": False, "carga": 95},
    {"host": "backend-02", "disponible": False, "carga": 87},
    {"host": "backend-03", "disponible": True,  "carga": 42},
    {"host": "backend-04", "disponible": True,  "carga": 31},
    {"host": "backend-05", "disponible": True,  "carga": 18},
]

servidor_seleccionado = None

print("Buscando servidor disponible (primer disponible):")
for servidor in servidores_backend:
    print(f"  Verificando {servidor['host']}...", end=" ")

    if not servidor["disponible"]:
        print("no disponible, continuando...")
        continue  # Saltar sin romper el bucle

    if servidor["carga"] > 80:
        print(f"sobrecargado ({servidor['carga']}%), continuando...")
        continue

    # Encontramos un servidor válido
    servidor_seleccionado = servidor
    print(f"SELECCIONADO (carga: {servidor['carga']}%)")
    break  # No necesitamos seguir buscando

if servidor_seleccionado:
    print(f"\nDirigiendo tráfico a: {servidor_seleccionado['host']}")
else:
    print("\nNo hay servidores disponibles. Devolviendo error 503.")
```

> **Prueba esto:** Cambia `"disponible": True` a `"disponible": False` en el servidor backend-03 y backend-04 para que solo quede disponible backend-05. ¿El `break` aún funciona correctamente? ¿Qué servidor se selecciona?

---

## 6. `continue` — saltar iteración

`continue` salta el resto del cuerpo del bucle para la iteración actual y avanza a la siguiente. A diferencia de `break`, no termina el bucle.

```python
# Procesar paquetes de red, saltando los corruptos o inválidos

paquetes_red = [
    {"id": 1,  "datos": "GET /api/usuarios",    "checksum": "valid",   "tamanio": 256},
    {"id": 2,  "datos": None,                   "checksum": "invalid", "tamanio": 0},
    {"id": 3,  "datos": "POST /api/pedidos",     "checksum": "valid",   "tamanio": 512},
    {"id": 4,  "datos": "DELETE /api/sesion",    "checksum": "invalid", "tamanio": 128},
    {"id": 5,  "datos": "",                      "checksum": "valid",   "tamanio": 0},
    {"id": 6,  "datos": "GET /api/productos",    "checksum": "valid",   "tamanio": 384},
    {"id": 7,  "datos": "PUT /api/usuario/42",   "checksum": "valid",   "tamanio": 320},
]

procesados = 0
descartados = 0

print("Procesando cola de paquetes de red:\n")

for paquete in paquetes_red:
    pid = paquete["id"]

    # Descartar paquetes con checksum inválido
    if paquete["checksum"] != "valid":
        print(f"  [PKT-{pid:03d}] DESCARTADO — checksum inválido")
        descartados += 1
        continue  # Saltar al siguiente paquete

    # Descartar paquetes sin datos
    if not paquete["datos"]:
        print(f"  [PKT-{pid:03d}] DESCARTADO — payload vacío")
        descartados += 1
        continue

    # Descartar paquetes demasiado pequeños (menos de 10 bytes)
    if paquete["tamanio"] < 10:
        print(f"  [PKT-{pid:03d}] DESCARTADO — tamaño insuficiente ({paquete['tamanio']} bytes)")
        descartados += 1
        continue

    # Si llegamos aquí, el paquete es válido — procesarlo
    metodo = paquete["datos"].split()[0]
    ruta = paquete["datos"].split()[1]
    print(f"  [PKT-{pid:03d}] OK — {metodo} {ruta} ({paquete['tamanio']} bytes)")
    procesados += 1

print(f"\nResumen: {procesados} procesados, {descartados} descartados de {len(paquetes_red)} totales")
```

> **Prueba esto:** Añade un nuevo tipo de validación con `continue`: descartar paquetes cuyo campo `"datos"` contenga la cadena `"DELETE"` (simula una política de firewall que bloquea eliminaciones). ¿Cuántos paquetes pasan ahora?

---

## 7. `else` en bucles

El bloque `else` de un bucle se ejecuta **únicamente** si el bucle terminó de forma natural (sin `break`). Es una característica exclusiva de Python y muy útil para búsquedas.

### El `else` confirma que NO se encontró el elemento

```python
# Buscar un certificado TLS específico en el almacén
certificados = [
    {"dominio": "api.empresa.com",    "valido": True,  "dias_restantes": 45},
    {"dominio": "app.empresa.com",    "valido": True,  "dias_restantes": 12},
    {"dominio": "admin.empresa.com",  "valido": False, "dias_restantes": -5},
    {"dominio": "internal.empresa.com","valido": True,  "dias_restantes": 90},
]

dominio_buscado = "pagos.empresa.com"  # No existe en el almacén

print(f"Buscando certificado para: {dominio_buscado}\n")

for cert in certificados:
    if cert["dominio"] == dominio_buscado:
        print(f"  Encontrado: {cert['dominio']} — {cert['dias_restantes']} días restantes")
        break
else:
    # Este bloque SOLO se ejecuta si el for terminó sin break
    print(f"  ALERTA: No se encontró certificado para '{dominio_buscado}'")
    print(f"  Acción requerida: generar certificado o verificar configuración DNS")
```

> **Prueba esto:** Cambia `dominio_buscado` a `"api.empresa.com"` (que sí existe). Confirma que el bloque `else` ya no se ejecuta. Luego cámbialo de nuevo a uno inexistente.

### Verificar que TODOS los servidores tienen TLS configurado

```python
# Verificar cumplimiento de política de seguridad TLS
servidores_tls = [
    {"nombre": "web-prod-01", "tls": True,  "version": "TLS1.3"},
    {"nombre": "web-prod-02", "tls": True,  "version": "TLS1.3"},
    {"nombre": "api-prod-01", "tls": True,  "version": "TLS1.2"},
    {"nombre": "legacy-01",   "tls": False, "version": None},    # Falla aquí
    {"nombre": "api-prod-02", "tls": True,  "version": "TLS1.3"},
]

print("Auditoría de seguridad TLS:\n")

for servidor in servidores_tls:
    print(f"  Verificando {servidor['nombre']}...", end=" ")
    if not servidor["tls"]:
        print(f"FALLO — TLS no configurado")
        primer_fallo = servidor["nombre"]
        break  # Detener en el primer fallo encontrado
    print(f"OK ({servidor['version']})")
else:
    # Solo llega aquí si TODOS pasaron la verificación
    print("\nAuditoría completada: todos los servidores tienen TLS configurado.")
    primer_fallo = None

if primer_fallo:
    print(f"\nAuditoría FALLIDA en: {primer_fallo}")
    print("Acción requerida: configurar TLS antes del próximo ciclo de despliegue.")
```

> **Prueba esto:** Elimina el servidor `"legacy-01"` de la lista. ¿Qué mensaje aparece ahora? Luego añade un servidor con `"tls": False` al final de la lista para ver cómo el `else` vuelve a no ejecutarse.

---

## 8. Bucles anidados y etiquetas de ruptura

Python **no tiene etiquetas** para `break` (como `break label` en Java). El patrón estándar es usar una función o una variable bandera.

```python
# Escanear una matriz de datacenters × racks buscando el primer servidor disponible
# Patrón con variable bandera (flag)

datacenters = ["DC-Norte", "DC-Sur", "DC-Este"]
racks_por_dc = {
    "DC-Norte": ["rack-A1", "rack-A2", "rack-A3"],
    "DC-Sur":   ["rack-B1", "rack-B2"],
    "DC-Este":  ["rack-C1", "rack-C2", "rack-C3", "rack-C4"],
}

# Simulación: disponibilidad de servidores por rack
disponibilidad = {
    "DC-Norte/rack-A1": False,
    "DC-Norte/rack-A2": False,
    "DC-Norte/rack-A3": False,
    "DC-Sur/rack-B1":   False,
    "DC-Sur/rack-B2":   True,   # ← Este está disponible
    "DC-Este/rack-C1":  True,
}

# Patrón 1: variable bandera para salir de bucles anidados
servidor_encontrado = None
encontrado = False  # bandera de control

for dc in datacenters:
    if encontrado:
        break  # Salir del bucle externo cuando la bandera esté activa
    for rack in racks_por_dc[dc]:
        clave = f"{dc}/{rack}"
        disponible = disponibilidad.get(clave, False)
        print(f"  Verificando {clave}...", end=" ")
        if disponible:
            print("DISPONIBLE")
            servidor_encontrado = clave
            encontrado = True
            break  # Salir del bucle interno
        else:
            print("ocupado")

if servidor_encontrado:
    print(f"\nServidor asignado en: {servidor_encontrado}")

# Patrón 2: encapsular en función y usar return (más limpio)
def encontrar_servidor(datacenters, racks_por_dc, disponibilidad):
    """Retorna la primera ubicación disponible o None si no hay ninguna."""
    for dc in datacenters:
        for rack in racks_por_dc[dc]:
            clave = f"{dc}/{rack}"
            if disponibilidad.get(clave, False):
                return clave  # return sale de todos los bucles automáticamente
    return None  # No se encontró ninguno

resultado = encontrar_servidor(datacenters, racks_por_dc, disponibilidad)
print(f"\n[Función] Resultado de búsqueda: {resultado}")
```

> **Prueba esto:** Cambia todos los valores de `disponibilidad` a `False`. ¿Qué devuelve la función `encontrar_servidor`? ¿Cómo cambiarías el código para lanzar una excepción en lugar de retornar `None`?

---

## 9. Comprensiones de lista — vista previa

Las comprensiones ofrecen una sintaxis compacta equivalente a un `for` con (opcionalmente) un `if`.

```python
# Comparación: for tradicional vs comprensión de lista

# --- Ejemplo 1: filtrar IPs privadas ---
todas_ips = ["10.0.0.1", "8.8.8.8", "192.168.1.1", "1.1.1.1", "172.16.0.5", "203.0.113.1"]

# Forma tradicional con for
ips_privadas_for = []
for ip in todas_ips:
    if ip.startswith(("10.", "192.168.", "172.")):
        ips_privadas_for.append(ip)

# Comprensión equivalente — más concisa y legible
ips_privadas_comp = [
    ip for ip in todas_ips
    if ip.startswith(("10.", "192.168.", "172."))
]

print(f"IPs privadas (for):  {ips_privadas_for}")
print(f"IPs privadas (comp): {ips_privadas_comp}")

# --- Ejemplo 2: transformar y filtrar registros de log ---
logs_raw = ["ERROR: timeout", "INFO: inicio", "ERROR: conexión rechazada", "DEBUG: ping", "ERROR: SSL"]

# For tradicional
errores_for = []
for log in logs_raw:
    if log.startswith("ERROR"):
        errores_for.append(log.replace("ERROR: ", "").upper())

# Comprensión
errores_comp = [
    log.replace("ERROR: ", "").upper()
    for log in logs_raw
    if log.startswith("ERROR")
]

print(f"\nErrores (for):  {errores_for}")
print(f"Errores (comp): {errores_comp}")

# --- Ejemplo 3: generar configuraciones para múltiples entornos ---
entornos = ["dev", "staging", "prod"]
servicios_base = ["api", "worker", "scheduler"]

# For anidado tradicional
configs_for = []
for entorno in entornos:
    for servicio in servicios_base:
        configs_for.append(f"{servicio}-{entorno}")

# Comprensión anidada
configs_comp = [
    f"{servicio}-{entorno}"
    for entorno in entornos
    for servicio in servicios_base
]

print(f"\nConfigs generadas: {len(configs_comp)}")
for config in configs_comp:
    print(f"  {config}")
```

> **Prueba esto:** Escribe una comprensión que genere una lista de los cuadrados de los números del 1 al 10, pero solo incluyendo los que son divisibles por 4. El resultado esperado es `[4, 16, 36, 64, 100]` — espera, ¿es correcto? Verifica cuáles cuadrados entre 1 y 100 son divisibles por 4.

---

## Ejemplo aplicado — procesador de eventos de log con estadísticas

```python
# Sistema completo de procesamiento de eventos de log
# Usa for, while, break, continue y acumula estadísticas

from collections import defaultdict

# Dataset de eventos simulados (en producción vendría de un archivo o stream)
eventos_log = [
    {"ts": "2024-01-15T10:00:01", "nivel": "INFO",    "servicio": "auth",         "msg": "Usuario autenticado"},
    {"ts": "2024-01-15T10:00:02", "nivel": "ERROR",   "servicio": "db",           "msg": "Timeout en consulta"},
    {"ts": "2024-01-15T10:00:03", "nivel": "ERROR",   "servicio": "db",           "msg": "Conexión rechazada"},
    {"ts": "2024-01-15T10:00:04", "nivel": "WARN",    "servicio": "api-gateway",  "msg": "Rate limit cerca del límite"},
    {"ts": "2024-01-15T10:00:05", "nivel": "ERROR",   "servicio": "db",           "msg": "Timeout en consulta"},
    {"ts": "2024-01-15T10:00:06", "nivel": "INFO",    "servicio": "auth",         "msg": "Token renovado"},
    {"ts": "2024-01-15T10:00:07", "nivel": "DEBUG",   "servicio": "cache",        "msg": "Cache hit"},
    {"ts": "2024-01-15T10:00:08", "nivel": "ERROR",   "servicio": "db",           "msg": "Conexión rechazada"},
    {"ts": "2024-01-15T10:00:09", "nivel": "ERROR",   "servicio": "payment",      "msg": "API externa no responde"},
    {"ts": "2024-01-15T10:00:10", "nivel": "CRITICAL","servicio": "db",           "msg": "Base de datos inaccesible"},
    {"ts": "2024-01-15T10:00:11", "nivel": "INFO",    "servicio": "health-check", "msg": "Ping OK"},
    {"ts": "2024-01-15T10:00:12", "nivel": "WARN",    "servicio": "memory",       "msg": "Uso de memoria al 85%"},
]

# Configuración del procesador
NIVELES_CRITICOS = {"ERROR", "CRITICAL"}
UMBRAL_RAFAGA_ERRORES = 3   # Alertar si hay N errores consecutivos
SERVICIO_IGNORADO = "health-check"  # No procesar pings de health-check

# Estadísticas acumuladas
stats = defaultdict(int)          # Conteo por nivel
errores_por_servicio = defaultdict(int)
mensajes_procesados = []

# Variables para detección de ráfaga de errores
errores_consecutivos = 0
max_errores_consecutivos = 0
rafaga_detectada = False

print("=" * 60)
print("PROCESADOR DE EVENTOS DE LOG")
print("=" * 60)

# Bucle principal de procesamiento
for evento in eventos_log:
    nivel = evento["nivel"]
    servicio = evento["servicio"]
    mensaje = evento["msg"]
    timestamp = evento["ts"]

    # Saltar eventos de servicios ignorados
    if servicio == SERVICIO_IGNORADO:
        stats["ignorados"] += 1
        continue  # No contar en estadísticas principales

    # Saltar eventos DEBUG en este procesador (solo en producción)
    if nivel == "DEBUG":
        stats["debug_omitidos"] += 1
        continue

    # Procesar el evento
    stats[nivel] += 1
    stats["total"] += 1
    mensajes_procesados.append(f"[{timestamp}] {nivel} | {servicio}: {mensaje}")

    # Acumular errores por servicio
    if nivel in NIVELES_CRITICOS:
        errores_por_servicio[servicio] += 1
        errores_consecutivos += 1
        max_errores_consecutivos = max(max_errores_consecutivos, errores_consecutivos)
    else:
        errores_consecutivos = 0  # Resetear contador de ráfaga

# Detectar ráfaga de errores con while sobre las estadísticas
# (ejemplo de while para procesamiento post-análisis)
umbral_alerta = 2
servicio_indice = 0
servicios_problema = list(errores_por_servicio.items())

print("\nAnálisis de servicios con errores:")
while servicio_indice < len(servicios_problema):
    servicio, total_errores = servicios_problema[servicio_indice]

    if total_errores >= umbral_alerta:
        print(f"  ALERTA: {servicio} — {total_errores} errores detectados")
        if total_errores >= UMBRAL_RAFAGA_ERRORES:
            print(f"  CRITICO: Posible ráfaga de errores en {servicio}")
            rafaga_detectada = True
    servicio_indice += 1

# Mostrar eventos procesados
print("\nEventos procesados:")
for i, msg in enumerate(mensajes_procesados, start=1):
    print(f"  {i:02d}. {msg}")

# Buscar el primer CRITICAL para alerta inmediata
print("\nBúsqueda de primer evento CRITICAL:")
for evento in eventos_log:
    if evento["nivel"] == "CRITICAL":
        print(f"  ENCONTRADO: {evento['ts']} — {evento['servicio']}: {evento['msg']}")
        break
else:
    print("  No se encontraron eventos CRITICAL en este lote.")

# Resumen final de estadísticas
print("\n" + "=" * 60)
print("RESUMEN DE ESTADÍSTICAS")
print("=" * 60)
for nivel in ["CRITICAL", "ERROR", "WARN", "INFO", "DEBUG", "ignorados", "debug_omitidos", "total"]:
    if nivel in stats:
        print(f"  {nivel:<20}: {stats[nivel]}")

print(f"\n  Ráfaga máxima detectada : {max_errores_consecutivos} errores consecutivos")
print(f"  Estado de alerta        : {'ACTIVA' if rafaga_detectada else 'Normal'}")
```

> **Prueba esto:** Añade 2 eventos más de nivel `"ERROR"` del servicio `"db"` al final de `eventos_log`. ¿Cambia el valor de `max_errores_consecutivos`? ¿Por qué depende de la posición en la lista?

---

## Ejercicios propuestos

**Ejercicio 1 — Contador de palabras en logs**
Dado un string de texto con múltiples líneas de log, usa un `for` y un diccionario para contar cuántas veces aparece cada nivel (`INFO`, `ERROR`, `WARN`, etc.). Muestra el resultado ordenado por frecuencia descendente.

**Ejercicio 2 — Validador de rangos de puertos**
Escribe una función que reciba una lista de números de puerto y use `for` con `continue` y `break` para:
- Saltar puertos reservados (< 1024)
- Detenerse si encuentra un puerto mayor a 65535
- Retornar la lista de puertos válidos procesados

**Ejercicio 3 — Búsqueda con `else`**
Dado un diccionario de usuarios con sus roles, escribe un bucle que busque si existe al menos un usuario con rol `"admin"`. Usa el bloque `else` para mostrar un mensaje de advertencia si no se encuentra ninguno.

**Ejercicio 4 — Pipeline de transformación**
Usando bucles anidados, procesa una lista de archivos CSV simulados (lista de listas de strings). Para cada archivo: salta las líneas de encabezado (`continue`), detente si encuentras una línea con `"FIN"` (`break`), y acumula el total de registros procesados.

---

## Resumen de la página 4

- **`for elemento in iterable`** es la forma idiomática en Python; itera sobre cualquier objeto iterable sin necesidad de índice manual.
- **`range(inicio, fin, paso)`** genera secuencias de enteros eficientemente; los tres argumentos permiten rangos personalizados incluyendo cuentas regresivas.
- **`enumerate(iterable, start=N)`** proporciona índice y valor simultáneamente, eliminando la necesidad de contadores manuales.
- **`zip()`** combina múltiples iterables en paralelo; `zip_longest()` maneja listas de distinta longitud con un valor de relleno.
- **`while condición:`** es ideal cuando el número de iteraciones depende de un estado externo; el patrón de backoff exponencial es su caso de uso más común en sistemas.
- **`break`** termina el bucle inmediatamente; útil en búsquedas donde ya no se necesitan más iteraciones.
- **`continue`** salta la iteración actual y avanza a la siguiente; útil para filtrar elementos inválidos manteniendo el flujo del bucle.
- **El bloque `else` en bucles** se ejecuta solo si el bucle terminó sin `break`; es exclusivo de Python y permite confirmar que una búsqueda no encontró resultados.

---

> **Siguiente página →** Página 5: Funciones — definición, parámetros nombrados, type hints y docstrings.
