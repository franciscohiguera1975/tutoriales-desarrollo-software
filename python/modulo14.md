# Tutorial Python — Página 14
## Módulo 6 · Concurrencia y Asincronía
### threading, multiprocessing y asyncio

---

## El GIL — Global Interpreter Lock

El GIL es un mutex que protege el intérprete CPython: solo un hilo
puede ejecutar bytecode Python a la vez.

```
Implicación práctica
─────────────────────────────────────────────────────────────────────
Tareas I/O-bound   → threading funciona bien — el GIL se libera
                     mientras espera red, disco, etc.
Tareas CPU-bound   → threading NO ayuda — el GIL impide ejecución
                     paralela real. Usa multiprocessing o concurrent.futures
Workarounds        → multiprocessing (procesos separados, cada uno
                     con su propio GIL), C extensions (NumPy, etc.)
Python 3.13+       → experimental "no-GIL" build (PEP 703)
```

---

## threading.Thread — I/O bound

```python
import threading
import time
import urllib.request

# threading.Thread crea un hilo del SO
# Ideal para: llamadas HTTP, lectura de archivos, esperas de red

def descargar_url(url: str, resultados: list[str], indice: int) -> None:
    """Descarga una URL y guarda el resultado en la lista compartida."""
    # Simula una descarga — el GIL se libera durante la espera de red
    with urllib.request.urlopen(url, timeout=5) as r:
        contenido = r.read(200)  # primeros 200 bytes
    resultados[indice] = f"{url}: {len(contenido)} bytes"

urls = [
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
]

resultados: list[str] = [""] * len(urls)
hilos: list[threading.Thread] = []

inicio = time.perf_counter()

# Crear e iniciar un hilo por URL
for i, url in enumerate(urls):
    hilo = threading.Thread(
        target=descargar_url,
        args=(url, resultados, i),
        name=f"descarga-{i}",   # nombre descriptivo para depuración
        daemon=True,             # el hilo muere si el proceso principal termina
    )
    hilos.append(hilo)
    hilo.start()

# Esperar a que todos terminen
for hilo in hilos:
    hilo.join(timeout=10)  # timeout de seguridad

fin = time.perf_counter()
print(f"Tiempo: {fin - inicio:.2f}s — esperado ~1s, no ~3s")
for r in resultados:
    print(r)
```

---

## Lock, Event y Semaphore

```python
import threading
import time
import random

# ── Lock — exclusión mutua ────────────────────────────────────────
# Evita race conditions cuando múltiples hilos modifican datos compartidos

contador_global = 0
lock = threading.Lock()

def incrementar_con_lock(n: int) -> None:
    global contador_global
    for _ in range(n):
        with lock:              # adquiere y libera automáticamente
            contador_global += 1

# ── Event — comunicación entre hilos ─────────────────────────────
# Un hilo espera hasta que otro señaliza el evento

evento_listo = threading.Event()

def productor() -> None:
    print("Productor: preparando datos...")
    time.sleep(2)
    evento_listo.set()          # señaliza — desbloquea todos los wait()
    print("Productor: datos listos")

def consumidor(nombre: str) -> None:
    print(f"{nombre}: esperando datos...")
    evento_listo.wait()         # bloquea hasta set()
    print(f"{nombre}: procesando datos")

# ── Semaphore — limitar concurrencia ─────────────────────────────
# Permite máximo N hilos simultáneos en una sección crítica

sem = threading.Semaphore(3)    # máximo 3 conexiones simultáneas

def conectar_a_db(id_hilo: int) -> None:
    with sem:                   # espera si ya hay 3 dentro
        print(f"Hilo {id_hilo}: conexión abierta")
        time.sleep(random.uniform(0.5, 1.5))
        print(f"Hilo {id_hilo}: conexión cerrada")

# Demostración
hilos_db = [threading.Thread(target=conectar_a_db, args=(i,))
            for i in range(8)]
for h in hilos_db:
    h.start()
for h in hilos_db:
    h.join()
```

---

## ThreadPoolExecutor — la forma moderna

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import httpx
import time

# ThreadPoolExecutor gestiona un pool de hilos reutilizables
# Evita el overhead de crear/destruir hilos para cada tarea

def obtener_estado(url: str) -> dict[str, object]:
    """Retorna estado HTTP y tiempo de respuesta."""
    inicio = time.perf_counter()
    try:
        respuesta = httpx.get(url, timeout=5)
        return {
            "url":    url,
            "status": respuesta.status_code,
            "tiempo": round(time.perf_counter() - inicio, 3),
            "error":  None,
        }
    except Exception as e:
        return {"url": url, "status": None, "tiempo": None, "error": str(e)}

urls = [
    "https://httpbin.org/status/200",
    "https://httpbin.org/status/404",
    "https://httpbin.org/status/500",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/json",
]

# max_workers=None usa min(32, os.cpu_count() + 4) por defecto
with ThreadPoolExecutor(max_workers=5) as executor:
    # submit() devuelve un Future inmediatamente
    futuros = {executor.submit(obtener_estado, url): url for url in urls}

    # as_completed() itera en orden de finalización — no de envío
    for futuro in as_completed(futuros):
        resultado = futuro.result()     # lanza excepción si hubo error
        print(f"{resultado['status']} | {resultado['tiempo']}s | {resultado['url']}")

# map() — más simple cuando no necesitas manejar errores individuales
with ThreadPoolExecutor(max_workers=5) as executor:
    resultados = list(executor.map(obtener_estado, urls, timeout=10))
```

---

## multiprocessing — CPU bound

```python
import multiprocessing as mp
import math
import time

# multiprocessing crea procesos separados — cada uno con su propio GIL
# Ideal para: cálculos numéricos, procesamiento de imágenes, ML

def factorizar(n: int) -> list[int]:
    """Factorización por división de prueba — intensivo en CPU."""
    factores = []
    d = 2
    while d * d <= n:
        while n % d == 0:
            factores.append(d)
            n //= d
        d += 1
    if n > 1:
        factores.append(n)
    return factores

def tarea_cpu(numero: int) -> tuple[int, list[int]]:
    """Wrapper que retorna el número junto con sus factores."""
    return numero, factorizar(numero)

numeros_grandes = [
    999_999_937,   # primo
    2**31 - 1,     # primo de Mersenne
    999_999_893,
    2**29 - 1,
]

# Secuencial
inicio = time.perf_counter()
resultados_seq = [tarea_cpu(n) for n in numeros_grandes]
print(f"Secuencial: {time.perf_counter() - inicio:.2f}s")

# Paralelo con Pool
# IMPORTANTE: este bloque DEBE estar bajo if __name__ == "__main__"
if __name__ == "__main__":
    inicio = time.perf_counter()
    # processes=None usa os.cpu_count()
    with mp.Pool(processes=4) as pool:
        resultados_par = pool.map(tarea_cpu, numeros_grandes)
    print(f"Paralelo:   {time.perf_counter() - inicio:.2f}s")

    for numero, factores in resultados_par:
        print(f"{numero} → {factores}")
```

---

## Queue y Pipe entre procesos

```python
import multiprocessing as mp
import time
import random

# Queue — comunicación segura entre múltiples productores/consumidores
# Pipe  — canal bidireccional entre exactamente 2 procesos

def trabajador(cola_entrada: mp.Queue, cola_salida: mp.Queue,
               id_worker: int) -> None:
    """Consume tareas de cola_entrada y pone resultados en cola_salida."""
    while True:
        tarea = cola_entrada.get()
        if tarea is None:       # señal de terminación (poison pill)
            break
        numero, exponente = tarea
        resultado = numero ** exponente
        time.sleep(random.uniform(0.01, 0.05))   # simula trabajo
        cola_salida.put((id_worker, numero, exponente, resultado))

if __name__ == "__main__":
    NUM_WORKERS = 4
    cola_entrada = mp.Queue()
    cola_salida  = mp.Queue()

    # Iniciar workers
    procesos = [
        mp.Process(target=trabajador, args=(cola_entrada, cola_salida, i))
        for i in range(NUM_WORKERS)
    ]
    for p in procesos:
        p.start()

    # Enviar tareas
    tareas = [(random.randint(2, 100), random.randint(2, 10))
              for _ in range(20)]
    for t in tareas:
        cola_entrada.put(t)

    # Enviar señales de terminación (una por worker)
    for _ in range(NUM_WORKERS):
        cola_entrada.put(None)

    # Recoger resultados
    for _ in range(len(tareas)):
        worker_id, base, exp, res = cola_salida.get()
        print(f"Worker {worker_id}: {base}^{exp} = {res}")

    for p in procesos:
        p.join()
```

---

## ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import os

def procesar_lote(datos: list[int]) -> dict[str, object]:
    """Procesa un lote de datos — ejecuta en un proceso separado."""
    pid = os.getpid()
    suma = sum(n ** 2 for n in datos)
    return {"pid": pid, "lote_size": len(datos), "resultado": suma}

if __name__ == "__main__":
    # Dividir datos en lotes para distribuir entre procesos
    datos_totales = list(range(1_000_000))
    num_workers   = os.cpu_count() or 4
    tamanio_lote  = len(datos_totales) // num_workers
    lotes = [
        datos_totales[i:i + tamanio_lote]
        for i in range(0, len(datos_totales), tamanio_lote)
    ]

    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        futuros = [executor.submit(procesar_lote, lote) for lote in lotes]
        for futuro in as_completed(futuros):
            resultado = futuro.result()
            print(f"PID {resultado['pid']}: lote={resultado['lote_size']}, "
                  f"suma={resultado['resultado']}")
```

---

## Comparativa: threading vs multiprocessing vs asyncio

```
Criterio              threading           multiprocessing     asyncio
────────────────────  ──────────────────  ──────────────────  ─────────────────
Tipo de tarea         I/O-bound           CPU-bound           I/O-bound
Paralelismo real      No (GIL)            Sí (procesos sep.)  No (1 hilo)
Overhead              Bajo                Alto (fork/spawn)   Muy bajo
Memoria compartida    Sí (con locks)      No (serialización)  Sí (coroutines)
Escalabilidad         Cientos de hilos    Decenas de proc.    Miles de tasks
Complejidad           Media               Alta                Media-Alta
Caso ideal            Llamadas HTTP       Numpy, PIL, ML      Servidores web
```

---

## asyncio — el event loop y coroutines

```python
import asyncio
import time

# async def define una coroutine — función que puede pausarse y reanudarse
# await cede el control al event loop mientras espera

async def tarea_lenta(nombre: str, segundos: float) -> str:
    """Simula una operación I/O asíncrona."""
    print(f"[{nombre}] iniciando — espera {segundos}s")
    await asyncio.sleep(segundos)    # libera el event loop durante la espera
    print(f"[{nombre}] completada")
    return f"resultado de {nombre}"

async def main_secuencial() -> None:
    """Las coroutines se ejecutan una tras otra — tiempo total = suma."""
    inicio = time.perf_counter()
    r1 = await tarea_lenta("A", 1.0)
    r2 = await tarea_lenta("B", 1.0)
    r3 = await tarea_lenta("C", 1.0)
    print(f"Total secuencial: {time.perf_counter() - inicio:.2f}s")
    print(r1, r2, r3)

async def main_concurrente() -> None:
    """gather() ejecuta las coroutines concurrentemente — tiempo = máximo."""
    inicio = time.perf_counter()
    r1, r2, r3 = await asyncio.gather(
        tarea_lenta("A", 1.0),
        tarea_lenta("B", 1.0),
        tarea_lenta("C", 1.0),
    )
    print(f"Total concurrente: {time.perf_counter() - inicio:.2f}s")
    print(r1, r2, r3)

# asyncio.run() crea y cierra el event loop — punto de entrada principal
asyncio.run(main_secuencial())   # ~3.0s
asyncio.run(main_concurrente())  # ~1.0s
```

---

## asyncio.gather, create_task y wait

```python
import asyncio

async def fetch_dato(id: int) -> dict[str, int]:
    await asyncio.sleep(0.1 * id)   # simula latencia variable
    return {"id": id, "valor": id * 10}

async def ejemplo_gather() -> None:
    # gather() ejecuta todas las coroutines y retorna lista de resultados
    # return_exceptions=True evita que un error cancele las demás
    resultados = await asyncio.gather(
        fetch_dato(1),
        fetch_dato(2),
        fetch_dato(3),
        return_exceptions=True,      # errores se incluyen en la lista
    )
    for r in resultados:
        if isinstance(r, Exception):
            print(f"Error: {r}")
        else:
            print(r)

async def ejemplo_tasks() -> None:
    # create_task() programa una coroutine sin esperar de inmediato
    # Permite hacer otra cosa mientras la task corre en segundo plano

    tarea1 = asyncio.create_task(fetch_dato(1), name="fetch-1")
    tarea2 = asyncio.create_task(fetch_dato(2), name="fetch-2")
    tarea3 = asyncio.create_task(fetch_dato(3), name="fetch-3")

    # Hacer otras cosas aquí mientras las tareas corren...
    await asyncio.sleep(0)    # cede el control para que empiecen

    # Esperar resultados cuando se necesiten
    print(await tarea1)
    print(await tarea2)
    print(await tarea3)

async def ejemplo_wait() -> None:
    # wait() ofrece control fino: FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED
    tasks = [asyncio.create_task(fetch_dato(i)) for i in range(5)]

    # Procesar conforme se van completando (como as_completed)
    while tasks:
        completadas, tasks = await asyncio.wait(
            tasks,
            return_when=asyncio.FIRST_COMPLETED,
        )
        for tarea in completadas:
            print(f"Completada: {tarea.result()}")

asyncio.run(ejemplo_gather())
asyncio.run(ejemplo_tasks())
asyncio.run(ejemplo_wait())
```

---

## asyncio.Queue

```python
import asyncio
import random

# asyncio.Queue — comunicación segura entre coroutines del mismo event loop

async def productor(cola: asyncio.Queue[int], num_items: int,
                    id_productor: int) -> None:
    for i in range(num_items):
        item = random.randint(1, 100)
        await cola.put(item)                       # bloquea si cola llena
        print(f"Productor {id_productor}: puso {item}")
        await asyncio.sleep(random.uniform(0.01, 0.1))

async def consumidor(cola: asyncio.Queue[int], id_consumidor: int) -> None:
    while True:
        try:
            item = await asyncio.wait_for(
                cola.get(), timeout=2.0            # timeout de 2s
            )
            print(f"Consumidor {id_consumidor}: procesó {item}")
            await asyncio.sleep(random.uniform(0.05, 0.15))
            cola.task_done()                       # señaliza que el item fue procesado
        except asyncio.TimeoutError:
            print(f"Consumidor {id_consumidor}: sin trabajo, saliendo")
            break

async def main_queue() -> None:
    # maxsize=0 → cola ilimitada; maxsize=N → cola acotada
    cola: asyncio.Queue[int] = asyncio.Queue(maxsize=10)

    # Iniciar 2 productores y 3 consumidores concurrentemente
    await asyncio.gather(
        productor(cola, num_items=5, id_productor=1),
        productor(cola, num_items=5, id_productor=2),
        consumidor(cola, id_consumidor=1),
        consumidor(cola, id_consumidor=2),
        consumidor(cola, id_consumidor=3),
    )

    # join() espera hasta que todos los items sean procesados
    await cola.join()
    print("Todos los items procesados")

asyncio.run(main_queue())
```

---

## asyncio.timeout y asyncio.wait_for

```python
import asyncio

async def operacion_lenta() -> str:
    await asyncio.sleep(5)
    return "resultado"

async def ejemplo_timeouts() -> None:
    # asyncio.timeout() — context manager (Python 3.11+)
    try:
        async with asyncio.timeout(2.0):         # 2 segundos máximo
            resultado = await operacion_lenta()
            print(resultado)
    except TimeoutError:
        print("Timeout con asyncio.timeout()")

    # asyncio.wait_for() — wrapper equivalente, compatible con 3.9+
    try:
        resultado = await asyncio.wait_for(
            operacion_lenta(),
            timeout=2.0,
        )
    except asyncio.TimeoutError:
        print("Timeout con asyncio.wait_for()")

    # timeout_at() con deadline absoluto (Python 3.11+)
    deadline = asyncio.get_event_loop().time() + 2.0
    try:
        async with asyncio.timeout_at(deadline):
            resultado = await operacion_lenta()
    except TimeoutError:
        print("Timeout con asyncio.timeout_at()")

asyncio.run(ejemplo_timeouts())
```

---

## Ejemplo aplicado — Crawler de URLs concurrente

```python
"""
crawler_async.py — Crawler de URLs con asyncio + httpx
Características:
  - Limite de concurrencia con asyncio.Semaphore
  - Reintentos con backoff exponencial
  - Timeout configurable por request
  - Reporte de resultados en tiempo real
"""

import asyncio
import time
from dataclasses import dataclass, field
from typing import Literal

import httpx  # pip install httpx

# ── Modelos de datos ──────────────────────────────────────────────

@dataclass
class ResultadoCrawl:
    url:     str
    status:  int | None
    tiempo:  float | None
    error:   str | None
    reintentos: int = 0

    @property
    def exitoso(self) -> bool:
        return self.status is not None and 200 <= self.status < 400

# ── Función de crawl con reintentos ──────────────────────────────

async def crawl_url(
    client:   httpx.AsyncClient,
    url:      str,
    semaforo: asyncio.Semaphore,
    timeout:  float = 10.0,
    max_reintentos: int = 3,
) -> ResultadoCrawl:
    """Descarga una URL con límite de concurrencia y reintentos."""
    reintentos = 0

    async with semaforo:     # limita a N requests simultáneos
        while reintentos <= max_reintentos:
            inicio = time.perf_counter()
            try:
                respuesta = await client.get(url, timeout=timeout)
                return ResultadoCrawl(
                    url=url,
                    status=respuesta.status_code,
                    tiempo=round(time.perf_counter() - inicio, 3),
                    error=None,
                    reintentos=reintentos,
                )
            except httpx.TimeoutException:
                error = f"Timeout después de {timeout}s"
            except httpx.RequestError as e:
                error = f"Error de red: {type(e).__name__}"
            except Exception as e:
                error = f"Error inesperado: {e}"

            reintentos += 1
            if reintentos <= max_reintentos:
                # backoff exponencial: 1s, 2s, 4s
                espera = 2 ** (reintentos - 1)
                print(f"  Reintento {reintentos}/{max_reintentos} para {url} "
                      f"(espera {espera}s): {error}")
                await asyncio.sleep(espera)

        return ResultadoCrawl(
            url=url,
            status=None,
            tiempo=None,
            error=error,
            reintentos=reintentos - 1,
        )

# ── Función principal del crawler ─────────────────────────────────

async def crawl_batch(
    urls:               list[str],
    max_concurrentes:   int   = 10,
    timeout_por_url:    float = 10.0,
    max_reintentos:     int   = 3,
) -> list[ResultadoCrawl]:
    """Crawlea una lista de URLs con concurrencia limitada."""
    semaforo = asyncio.Semaphore(max_concurrentes)

    # Un solo cliente httpx para toda la sesión — reutiliza conexiones HTTP
    async with httpx.AsyncClient(
        follow_redirects=True,
        headers={"User-Agent": "Python-Crawler/1.0"},
    ) as client:
        # Crear todas las tareas de inmediato
        tareas = [
            asyncio.create_task(
                crawl_url(client, url, semaforo, timeout_por_url, max_reintentos),
                name=f"crawl-{i}",
            )
            for i, url in enumerate(urls)
        ]

        resultados: list[ResultadoCrawl] = []

        # Procesar conforme se completan (orden de finalización)
        for futuro in asyncio.as_completed(tareas):
            resultado = await futuro
            resultados.append(resultado)
            icono = "✓" if resultado.exitoso else "✗"
            print(
                f"{icono} [{resultado.status or 'ERR'}] "
                f"{resultado.url[:60]:<60} "
                f"{resultado.tiempo or 0:.3f}s "
                f"(reintentos: {resultado.reintentos})"
            )

    return resultados

# ── Reporte final ─────────────────────────────────────────────────

def imprimir_reporte(resultados: list[ResultadoCrawl], tiempo_total: float) -> None:
    exitosos  = [r for r in resultados if r.exitoso]
    fallidos  = [r for r in resultados if not r.exitoso]
    tiempos   = [r.tiempo for r in exitosos if r.tiempo is not None]

    print("\n" + "=" * 70)
    print(f"REPORTE FINAL")
    print("=" * 70)
    print(f"Total URLs:      {len(resultados)}")
    print(f"Exitosas:        {len(exitosos)}")
    print(f"Fallidas:        {len(fallidos)}")
    print(f"Tiempo total:    {tiempo_total:.2f}s")

    if tiempos:
        print(f"Tiempo promedio: {sum(tiempos)/len(tiempos):.3f}s")
        print(f"Tiempo máximo:   {max(tiempos):.3f}s")
        print(f"Tiempo mínimo:   {min(tiempos):.3f}s")

    if fallidos:
        print("\nURLs fallidas:")
        for r in fallidos:
            print(f"  {r.url}: {r.error}")

# ── Punto de entrada ──────────────────────────────────────────────

async def main() -> None:
    urls_a_crawlear = [
        "https://httpbin.org/status/200",
        "https://httpbin.org/status/201",
        "https://httpbin.org/status/404",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/json",
        "https://httpbin.org/headers",
        "https://httpbin.org/ip",
        "https://httpbin.org/uuid",
        "https://httpbin.org/status/500",
    ] * 2   # 20 URLs en total

    print(f"Crawleando {len(urls_a_crawlear)} URLs con max 5 concurrentes...")
    inicio = time.perf_counter()

    resultados = await crawl_batch(
        urls=urls_a_crawlear,
        max_concurrentes=5,
        timeout_por_url=8.0,
        max_reintentos=2,
    )

    tiempo_total = time.perf_counter() - inicio
    imprimir_reporte(resultados, tiempo_total)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Ejercicios propuestos

1. **Pipeline de procesamiento asíncrono** — Implementa un pipeline de 3
   etapas usando `asyncio.Queue`: el primer stage lee líneas de un archivo,
   el segundo las transforma (convierte a mayúsculas y elimina espacios),
   y el tercero las escribe en otro archivo. Usa `asyncio.gather()` para
   correr los 3 stages concurrentemente y `None` como señal de terminación.

2. **ThreadPoolExecutor para hashing** — Tienes 1000 contraseñas que necesitas
   hashear con bcrypt (operación CPU-bound de ~0.5s cada una). Usa
   `ProcessPoolExecutor` para distribuir el trabajo entre todos los núcleos
   disponibles. Compara el tiempo con el enfoque secuencial y reporta el
   speedup obtenido.

3. **Rate limiter con Semaphore** — Crea un decorador `@rate_limit(calls, period)`
   que use `asyncio.Semaphore` y un timestamp para garantizar que una función
   async no sea llamada más de `calls` veces en `period` segundos. Pruébalo
   con una función que simula una llamada a una API con límite de 5 req/s.

4. **Worker pool con reinicio automático** — Implementa un `WorkerPool` usando
   `multiprocessing.Process` que mantiene siempre N workers activos: si un
   worker muere (lanza una excepción), el pool lo detecta y lanza uno nuevo.
   Usa `mp.Queue` para distribuir tareas y una `mp.Event` para señalizar el
   apagado limpio del pool.

---

## Resumen de la página 11

- El GIL limita CPython a un hilo de Python a la vez — `threading` funciona bien para I/O-bound, `multiprocessing` para CPU-bound.
- `threading.Lock` protege secciones críticas, `threading.Event` sincroniza hilos, `threading.Semaphore` limita concurrencia.
- `ThreadPoolExecutor` y `ProcessPoolExecutor` son la API moderna de alto nivel — preferir sobre `Thread`/`Process` directos.
- `multiprocessing.Queue` y `Pipe` permiten comunicación entre procesos; los datos se serializan con pickle automáticamente.
- `async def` define una coroutine; `await` cede el control al event loop sin bloquear el hilo.
- `asyncio.gather()` ejecuta coroutines concurrentemente y espera todos los resultados; `return_exceptions=True` evita fallos en cascada.
- `asyncio.create_task()` programa una coroutine sin esperar — permite trabajo en paralelo dentro del mismo event loop.
- `asyncio.Semaphore` es la herramienta clave para limitar concurrencia en crawlers y clientes HTTP — evita saturar servidores.

---

> **Siguiente página →** Página 15: HTTP con httpx y FastAPI — fundamentos.
> Consumir APIs con httpx (sync y async), construir APIs con FastAPI,
> Pydantic, path/query params, routers y dependency injection básica.
