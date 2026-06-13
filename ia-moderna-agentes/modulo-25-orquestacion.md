# Módulo 25 — Orquestación de Agentes

> **Objetivo:** Diseñar sistemas donde múltiples agentes especializados colaboran bajo la dirección de un orquestador: patrones de delegación, supervisión, paralelismo y gestión de fallos en pipelines agénticos.
>
> **Herramientas:** Python 3.11, dataclasses, threading (simulado)
>
> **Prerequisito:** M19 (CrewAI), M20 (AutoGen)

---

## 1. Patrones de Orquestación

```python
# modulo-25-orquestacion.py — Parte 1: Patrones base

"""
Patrones principales de orquestación multi-agente:

1. Pipeline Secuencial: A → B → C → resultado
   Cada agente recibe el output del anterior.

2. Fan-out / Fan-in: Orquestador → [A, B, C] → merge → resultado
   El orquestador lanza sub-tareas en paralelo y combina resultados.

3. Supervisor-Worker: Supervisor asigna, workers ejecutan, supervisor valida
   El supervisor puede reasignar si un worker falla.

4. Map-Reduce Agéntico: map(tarea, [chunk1, chunk2...]) → reduce(resultados)
   Para tareas sobre colecciones grandes.

5. Recursivo / Divide y Vencerás: Si la tarea es compleja → dividir → resolver → combinar
"""

import json
import time
import uuid
import hashlib
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Tuple
from enum import Enum
from concurrent.futures import ThreadPoolExecutor, as_completed


# ── Primitivas ────────────────────────────────────────────────────────────────

class EstadoTarea(Enum):
    PENDIENTE  = "pendiente"
    EN_CURSO   = "en_curso"
    COMPLETADA = "completada"
    FALLIDA    = "fallida"
    CANCELADA  = "cancelada"


@dataclass
class Tarea:
    """Unidad de trabajo asignable a un agente."""
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    descripcion: str = ""
    input_data: Any = None
    output_data: Any = None
    estado: EstadoTarea = EstadoTarea.PENDIENTE
    agente_asignado: Optional[str] = None
    prioridad: int = 5            # 1=alta, 10=baja
    dependencias: List[str] = field(default_factory=list)  # ids de tareas que deben completarse primero
    intentos: int = 0
    max_intentos: int = 3
    tiempo_inicio: Optional[float] = None
    tiempo_fin: Optional[float] = None
    error: Optional[str] = None

    @property
    def duracion_ms(self) -> Optional[int]:
        if self.tiempo_inicio and self.tiempo_fin:
            return int((self.tiempo_fin - self.tiempo_inicio) * 1000)
        return None

    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "descripcion": self.descripcion,
            "estado": self.estado.value,
            "agente": self.agente_asignado,
            "prioridad": self.prioridad,
            "intentos": self.intentos,
            "duracion_ms": self.duracion_ms,
            "error": self.error,
        }


class LLMSimulado:
    """LLM determinista para demos."""
    def __init__(self, rol: str = "general"):
        self.rol = rol
    def completar(self, prompt: str) -> str:
        prompt_l = prompt.lower()
        if self.rol == "investigador":
            return f"[Investigación] Hallazgos sobre '{prompt[:40]}': contexto histórico, estado del arte, limitaciones actuales."
        if self.rol == "redactor":
            return f"[Redacción] Artículo sobre '{prompt[:40]}': introducción clara, desarrollo con ejemplos, conclusión accionable."
        if self.rol == "critico":
            return f"[Crítica] Revisión de '{prompt[:40]}': ✓ estructura lógica, ✓ argumentos sólidos, ✗ falta ejemplos concretos."
        if self.rol == "sintetizador":
            return f"[Síntesis] Combinando inputs: resumen ejecutivo en 3 puntos clave, recomendaciones finales."
        if self.rol == "planificador":
            # Descompone la tarea en sub-tareas
            return json.dumps({
                "sub_tareas": [
                    {"descripcion": "investigar antecedentes", "prioridad": 1},
                    {"descripcion": "analizar datos disponibles", "prioridad": 2},
                    {"descripcion": "redactar borrador", "prioridad": 3},
                    {"descripcion": "revisar y corregir", "prioridad": 4},
                ]
            })
        return f"[{self.rol}] Procesado: {prompt[:50]}"


@dataclass
class Agente:
    """Agente especializado que puede ejecutar tareas."""
    nombre: str
    rol: str
    llm: LLMSimulado
    capacidades: List[str] = field(default_factory=list)
    disponible: bool = True
    tareas_completadas: int = 0
    tareas_fallidas: int = 0

    def ejecutar(self, tarea: Tarea) -> Tarea:
        """Ejecuta una tarea y actualiza su estado."""
        tarea.agente_asignado = self.nombre
        tarea.estado = EstadoTarea.EN_CURSO
        tarea.tiempo_inicio = time.time()
        tarea.intentos += 1
        self.disponible = False

        try:
            prompt = f"Tarea: {tarea.descripcion}\nInput: {tarea.input_data}"
            resultado = self.llm.completar(prompt)
            tarea.output_data = resultado
            tarea.estado = EstadoTarea.COMPLETADA
            self.tareas_completadas += 1
        except Exception as e:
            tarea.error = str(e)
            tarea.estado = EstadoTarea.FALLIDA
            self.tareas_fallidas += 1
        finally:
            tarea.tiempo_fin = time.time()
            self.disponible = True

        return tarea

    @property
    def stats(self) -> Dict:
        total = self.tareas_completadas + self.tareas_fallidas
        return {
            "completadas": self.tareas_completadas,
            "fallidas": self.tareas_fallidas,
            "tasa_exito": self.tareas_completadas / max(1, total),
        }
```

---

## 2. Orquestador con Pipeline Secuencial

```python
# Parte 2: Pipeline Secuencial

class PipelineSecuencial:
    """
    Ejecuta una secuencia de agentes en orden.
    El output de cada agente es el input del siguiente.
    """

    def __init__(self, nombre: str):
        self.nombre = nombre
        self._pasos: List[Tuple[str, Agente]] = []  # (descripcion, agente)
        self._historial: List[Tarea] = []

    def agregar_paso(self, descripcion: str, agente: Agente) -> "PipelineSecuencial":
        self._pasos.append((descripcion, agente))
        return self

    def ejecutar(self, input_inicial: Any, contexto: str = "") -> Dict:
        """
        Ejecuta todos los pasos en secuencia.
        Retorna el resultado final y el historial de ejecución.
        """
        print(f"\n[Pipeline: {self.nombre}]")
        data_actual = input_inicial
        inicio = time.time()

        for i, (desc, agente) in enumerate(self._pasos):
            tarea = Tarea(
                descripcion=f"Paso {i+1}: {desc}",
                input_data=data_actual,
                prioridad=i + 1,
            )
            print(f"  Paso {i+1}/{len(self._pasos)}: {agente.nombre} ({agente.rol})")
            tarea = agente.ejecutar(tarea)
            self._historial.append(tarea)

            if tarea.estado == EstadoTarea.FALLIDA:
                print(f"  ✗ Fallo en paso {i+1}: {tarea.error}")
                return {
                    "exito": False,
                    "fallo_en_paso": i + 1,
                    "error": tarea.error,
                    "historial": [t.to_dict() for t in self._historial],
                }

            data_actual = tarea.output_data
            print(f"  ✓ Completado en {tarea.duracion_ms}ms")

        duracion_total = time.time() - inicio
        print(f"  Pipeline completado en {int(duracion_total*1000)}ms")

        return {
            "exito": True,
            "resultado": data_actual,
            "pasos": len(self._pasos),
            "duracion_ms": int(duracion_total * 1000),
            "historial": [t.to_dict() for t in self._historial],
        }


# ── Demo 1 ────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Pipeline Secuencial")
    print("=" * 60)

    investigador = Agente("Inv-01", "investigador", LLMSimulado("investigador"), ["investigar", "resumir"])
    redactor = Agente("Red-01", "redactor", LLMSimulado("redactor"), ["redactar", "editar"])
    critico = Agente("Cri-01", "critico", LLMSimulado("critico"), ["revisar", "criticar"])

    pipeline = (
        PipelineSecuencial("Generación de Artículo")
        .agregar_paso("Investigar el tema", investigador)
        .agregar_paso("Redactar el artículo", redactor)
        .agregar_paso("Revisar y criticar", critico)
    )

    resultado = pipeline.ejecutar(
        input_inicial="Impacto de los LLMs en la educación superior",
        contexto="Artículo para revista académica",
    )

    print(f"\nResultado final (últimos 200 chars):\n{resultado['resultado'][-200:]}")

    with open("outputs/m25_pipeline.json", "w", encoding="utf-8") as f:
        json.dump(resultado, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m25_pipeline.json")
```

---

## 3. Fan-out / Fan-in (Paralelismo)

```python
# Parte 3: Fan-out/Fan-in

class FanOutFanIn:
    """
    Orquestador fan-out/fan-in:
    1. Divide la tarea en sub-tareas
    2. Las ejecuta en paralelo (simulado con threads)
    3. Combina los resultados con el agente sintetizador
    """

    def __init__(self, sintetizador: Agente, max_workers: int = 4):
        self._sintetizador = sintetizador
        self._max_workers = max_workers
        self._workers: List[Agente] = []
        self._historial: List[Tarea] = []

    def agregar_worker(self, agente: Agente) -> "FanOutFanIn":
        self._workers.append(agente)
        return self

    def _distribuir_tarea(self, tarea_principal: str, sub_tareas: List[str]) -> List[Tarea]:
        """Distribuye sub-tareas entre los workers disponibles."""
        tareas = []
        for i, desc in enumerate(sub_tareas):
            worker = self._workers[i % len(self._workers)]
            tarea = Tarea(
                descripcion=f"[Sub-tarea] {desc}",
                input_data=f"{tarea_principal}: {desc}",
                agente_asignado=worker.nombre,
            )
            tareas.append((tarea, worker))
        return tareas

    def ejecutar(self, tarea_principal: str, sub_tareas: List[str]) -> Dict:
        """Ejecuta sub-tareas en paralelo y sintetiza el resultado."""
        print(f"\n[Fan-out/Fan-in]")
        print(f"  Tarea: {tarea_principal}")
        print(f"  Sub-tareas: {len(sub_tareas)}, Workers: {len(self._workers)}")

        inicio = time.time()
        distribuidas = self._distribuir_tarea(tarea_principal, sub_tareas)

        # Ejecutar en paralelo (simulado con ThreadPoolExecutor)
        resultados_parciales: List[str] = []
        tareas_ejecutadas: List[Tarea] = []

        def ejecutar_tarea(par):
            tarea, worker = par
            return worker.ejecutar(tarea)

        with ThreadPoolExecutor(max_workers=self._max_workers) as executor:
            futures = {executor.submit(ejecutar_tarea, par): par for par in distribuidas}
            for future in as_completed(futures):
                tarea_resultado = future.result()
                tareas_ejecutadas.append(tarea_resultado)
                self._historial.append(tarea_resultado)
                estado = "✓" if tarea_resultado.estado == EstadoTarea.COMPLETADA else "✗"
                print(f"  {estado} [{tarea_resultado.agente_asignado}] {tarea_resultado.descripcion[:50]}")
                if tarea_resultado.output_data:
                    resultados_parciales.append(str(tarea_resultado.output_data))

        # Fan-in: sintetizar resultados
        print(f"  → Sintetizando {len(resultados_parciales)} resultados parciales...")
        tarea_sintesis = Tarea(
            descripcion="Sintetizar resultados",
            input_data="\n\n".join(resultados_parciales),
        )
        tarea_sintesis = self._sintetizador.ejecutar(tarea_sintesis)
        self._historial.append(tarea_sintesis)

        duracion_total = time.time() - inicio
        completadas = sum(1 for t in tareas_ejecutadas if t.estado == EstadoTarea.COMPLETADA)

        return {
            "exito": tarea_sintesis.estado == EstadoTarea.COMPLETADA,
            "resultado_final": tarea_sintesis.output_data,
            "sub_tareas_completadas": completadas,
            "sub_tareas_total": len(sub_tareas),
            "duracion_ms": int(duracion_total * 1000),
            "historial": [t.to_dict() for t in self._historial],
        }


def demo_fan_out():
    print("\n" + "=" * 60)
    print("DEMO 2 — Fan-out / Fan-in")
    print("=" * 60)

    workers = [
        Agente(f"Worker-{i}", "investigador", LLMSimulado("investigador"))
        for i in range(3)
    ]
    sintetizador = Agente("Sintetizador", "sintetizador", LLMSimulado("sintetizador"))

    orq = FanOutFanIn(sintetizador, max_workers=3)
    for w in workers:
        orq.agregar_worker(w)

    resultado = orq.ejecutar(
        tarea_principal="Análisis de tendencias en IA 2024",
        sub_tareas=[
            "Tendencias en LLMs y modelos de lenguaje",
            "Avances en visión por computadora",
            "Desarrollo de agentes autónomos",
            "IA en producción y MLOps",
            "Regulación y ética de la IA",
        ],
    )

    print(f"\nResultado final: {resultado['resultado_final'][:200]}")
    print(f"Sub-tareas: {resultado['sub_tareas_completadas']}/{resultado['sub_tareas_total']}")

    with open("outputs/m25_fan_out.json", "w", encoding="utf-8") as f:
        json.dump(resultado, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m25_fan_out.json")
    return resultado


if __name__ == "__main__":
    demo_fan_out()
```

---

## 4. Supervisor con Manejo de Fallos y Reasignación

```python
# Parte 4: Supervisor-Worker con recuperación de fallos

class ColaProioridad:
    """Cola de tareas ordenada por prioridad (min-heap simulado)."""

    def __init__(self):
        self._tareas: List[Tarea] = []

    def encolar(self, tarea: Tarea):
        self._tareas.append(tarea)
        self._tareas.sort(key=lambda t: t.prioridad)

    def desencolar(self) -> Optional[Tarea]:
        if not self._tareas:
            return None
        return self._tareas.pop(0)

    def __len__(self) -> int:
        return len(self._tareas)

    @property
    def pendientes(self) -> List[Tarea]:
        return [t for t in self._tareas if t.estado == EstadoTarea.PENDIENTE]


class Supervisor:
    """
    Orquestador tipo supervisor-worker:
    - Mantiene una cola de tareas
    - Asigna tareas a workers disponibles
    - Monitorea el progreso
    - Reasigna tareas fallidas a otros workers
    - Detecta workers no responsivos (timeout)
    """

    def __init__(self, nombre: str, max_reintentos: int = 2):
        self.nombre = nombre
        self.max_reintentos = max_reintentos
        self._cola = ColaProioridad()
        self._workers: Dict[str, Agente] = {}
        self._tareas_completadas: List[Tarea] = []
        self._tareas_fallidas: List[Tarea] = []
        self._log: List[Dict] = []

    def registrar_worker(self, agente: Agente):
        self._workers[agente.nombre] = agente

    def encolar_tarea(self, tarea: Tarea):
        self._cola.encolar(tarea)
        self._log.append({"accion": "encolar", "tarea_id": tarea.id, "desc": tarea.descripcion[:50]})

    def _seleccionar_worker(self, tarea: Tarea) -> Optional[Agente]:
        """Selecciona el worker más adecuado (disponible + capacidad)."""
        candidatos = [
            w for w in self._workers.values()
            if w.disponible and (
                not tarea.descripcion or  # cualquier worker
                not w.capacidades or      # worker sin restricción
                any(cap in tarea.descripcion.lower() for cap in w.capacidades)
            )
        ]
        if not candidatos:
            return None
        # Preferir el worker con más éxitos
        return max(candidatos, key=lambda w: w.tareas_completadas)

    def ejecutar_todo(self, timeout_total: float = 60.0) -> Dict:
        """Ejecuta todas las tareas en cola hasta que se vacíe o se agote el tiempo."""
        inicio = time.time()
        print(f"\n[Supervisor: {self.nombre}]")
        print(f"  Workers: {list(self._workers.keys())}")
        print(f"  Tareas en cola: {len(self._cola)}")

        while len(self._cola) > 0:
            if time.time() - inicio > timeout_total:
                print("  ⚠ Timeout alcanzado")
                break

            tarea = self._cola.desencolar()
            if not tarea:
                break

            # Verificar dependencias
            deps_completas = all(
                any(t.id == dep_id and t.estado == EstadoTarea.COMPLETADA
                    for t in self._tareas_completadas)
                for dep_id in tarea.dependencias
            )
            if not deps_completas:
                # Re-encolar al final con menor prioridad
                tarea.prioridad += 1
                self._cola.encolar(tarea)
                continue

            # Asignar worker
            worker = self._seleccionar_worker(tarea)
            if not worker:
                print(f"  ⏳ Sin workers disponibles para: {tarea.descripcion[:40]}")
                self._cola.encolar(tarea)
                time.sleep(0.01)
                continue

            # Ejecutar
            print(f"  → [{worker.nombre}] ejecuta: {tarea.descripcion[:45]}")
            tarea = worker.ejecutar(tarea)
            self._log.append({
                "accion": "ejecutar",
                "tarea_id": tarea.id,
                "worker": worker.nombre,
                "estado": tarea.estado.value,
                "duracion_ms": tarea.duracion_ms,
            })

            if tarea.estado == EstadoTarea.COMPLETADA:
                self._tareas_completadas.append(tarea)
                print(f"  ✓ [{worker.nombre}] {tarea.descripcion[:40]} ({tarea.duracion_ms}ms)")
            else:
                # Fallo: reintentar si quedan intentos
                if tarea.intentos < self.max_reintentos:
                    print(f"  ↻ Reintento {tarea.intentos}/{self.max_reintentos}: {tarea.descripcion[:40]}")
                    tarea.estado = EstadoTarea.PENDIENTE
                    self._cola.encolar(tarea)
                else:
                    self._tareas_fallidas.append(tarea)
                    print(f"  ✗ Abandono tras {tarea.intentos} intentos: {tarea.descripcion[:40]}")

        duracion = time.time() - inicio
        return {
            "completadas": len(self._tareas_completadas),
            "fallidas": len(self._tareas_fallidas),
            "duracion_ms": int(duracion * 1000),
            "resultados": [
                {"id": t.id, "desc": t.descripcion, "output": str(t.output_data)[:100]}
                for t in self._tareas_completadas
            ],
            "log": self._log,
        }


def demo_supervisor():
    print("\n" + "=" * 60)
    print("DEMO 3 — Supervisor-Worker con Gestión de Fallos")
    print("=" * 60)

    supervisor = Supervisor("Coordinador", max_reintentos=2)

    # Workers con diferentes especialidades
    supervisor.registrar_worker(Agente("W-Inv", "investigador", LLMSimulado("investigador"), ["investigar", "analizar"]))
    supervisor.registrar_worker(Agente("W-Red", "redactor", LLMSimulado("redactor"), ["redactar", "escribir"]))
    supervisor.registrar_worker(Agente("W-Gen", "general", LLMSimulado(), []))  # worker genérico

    # Encolar tareas con diferentes prioridades y dependencias
    t1 = Tarea("t1", "investigar el tema principal", prioridad=1)
    t2 = Tarea("t2", "analizar competencia", prioridad=2)
    t3 = Tarea("t3", "redactar sección introducción", prioridad=3, dependencias=["t1"])
    t4 = Tarea("t4", "redactar sección análisis", prioridad=3, dependencias=["t1", "t2"])
    t5 = Tarea("t5", "revisión final", prioridad=4, dependencias=["t3", "t4"])

    for tarea in [t1, t2, t3, t4, t5]:
        supervisor.encolar_tarea(tarea)

    resultado = supervisor.ejecutar_todo(timeout_total=10.0)

    print(f"\nCompletadas: {resultado['completadas']}, Fallidas: {resultado['fallidas']}")
    print(f"Duración total: {resultado['duracion_ms']}ms")

    with open("outputs/m25_supervisor.json", "w", encoding="utf-8") as f:
        json.dump(resultado, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m25_supervisor.json")
    return resultado


if __name__ == "__main__":
    demo_supervisor()
    print("\n[M25 completado] Todos los outputs guardados en outputs/")
```

---

## Checkpoint

```python
# checkpoint_m25.py — 6 tests para Orquestación

import time, uuid, json
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional, Tuple
from enum import Enum

class EstadoTarea(Enum):
    PENDIENTE = "pendiente"; EN_CURSO = "en_curso"; COMPLETADA = "completada"; FALLIDA = "fallida"

@dataclass
class Tarea:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    descripcion: str = ""; input_data: Any = None; output_data: Any = None
    estado: EstadoTarea = EstadoTarea.PENDIENTE; agente_asignado: Optional[str] = None
    prioridad: int = 5; dependencias: List[str] = field(default_factory=list)
    intentos: int = 0; max_intentos: int = 3
    tiempo_inicio: Optional[float] = None; tiempo_fin: Optional[float] = None; error: Optional[str] = None
    @property
    def duracion_ms(self): return int((self.tiempo_fin - self.tiempo_inicio)*1000) if self.tiempo_inicio and self.tiempo_fin else None
    def to_dict(self): return {"id":self.id,"estado":self.estado.value,"agente":self.agente_asignado,"duracion_ms":self.duracion_ms}

class LLMSimulado:
    def __init__(self, rol="general"): self.rol = rol
    def completar(self, prompt): return f"[{self.rol}] {prompt[:50]}"

@dataclass
class Agente:
    nombre: str; rol: str; llm: LLMSimulado; capacidades: List[str] = field(default_factory=list)
    disponible: bool = True; tareas_completadas: int = 0; tareas_fallidas: int = 0
    def ejecutar(self, tarea):
        tarea.agente_asignado = self.nombre; tarea.estado = EstadoTarea.EN_CURSO
        tarea.tiempo_inicio = time.time(); tarea.intentos += 1; self.disponible = False
        try:
            tarea.output_data = self.llm.completar(f"{tarea.descripcion}: {tarea.input_data}")
            tarea.estado = EstadoTarea.COMPLETADA; self.tareas_completadas += 1
        except Exception as e:
            tarea.error = str(e); tarea.estado = EstadoTarea.FALLIDA; self.tareas_fallidas += 1
        finally: tarea.tiempo_fin = time.time(); self.disponible = True
        return tarea

class ColaProioridad:
    def __init__(self): self._tareas = []
    def encolar(self, t): self._tareas.append(t); self._tareas.sort(key=lambda x: x.prioridad)
    def desencolar(self): return self._tareas.pop(0) if self._tareas else None
    def __len__(self): return len(self._tareas)

class PipelineSecuencial:
    def __init__(self, nombre): self.nombre = nombre; self._pasos = []; self._historial = []
    def agregar_paso(self, desc, agente): self._pasos.append((desc, agente)); return self
    def ejecutar(self, input_inicial):
        data = input_inicial
        for desc, agente in self._pasos:
            t = Tarea(descripcion=desc, input_data=data)
            t = agente.ejecutar(t); self._historial.append(t)
            if t.estado == EstadoTarea.FALLIDA: return {"exito": False, "error": t.error}
            data = t.output_data
        return {"exito": True, "resultado": data, "historial": [t.to_dict() for t in self._historial]}


def test_01_tarea_duracion_ms():
    """Tarea registra duración correctamente."""
    t = Tarea(descripcion="test")
    t.tiempo_inicio = 1000.0; t.tiempo_fin = 1000.5
    assert t.duracion_ms == 500
    t2 = Tarea()
    assert t2.duracion_ms is None
    print("✓ test_01 — Tarea duracion_ms OK")

def test_02_agente_ejecuta_tarea():
    """Agente ejecuta tarea y la marca como completada."""
    agente = Agente("TestAgent", "general", LLMSimulado())
    tarea = Tarea(descripcion="tarea de prueba", input_data="dato")
    resultado = agente.ejecutar(tarea)
    assert resultado.estado == EstadoTarea.COMPLETADA
    assert resultado.agente_asignado == "TestAgent"
    assert resultado.output_data is not None
    assert resultado.intentos == 1
    assert agente.tareas_completadas == 1
    print("✓ test_02 — Agente ejecuta tarea OK")

def test_03_pipeline_secuencial():
    """Pipeline secuencial ejecuta pasos en orden."""
    a1 = Agente("A1", "r1", LLMSimulado("investigador"))
    a2 = Agente("A2", "r2", LLMSimulado("redactor"))
    pipeline = PipelineSecuencial("Test").agregar_paso("investigar", a1).agregar_paso("redactar", a2)
    resultado = pipeline.ejecutar("tema de prueba")
    assert resultado["exito"] == True
    assert len(resultado["historial"]) == 2
    assert resultado["resultado"] is not None
    print("✓ test_03 — Pipeline secuencial OK")

def test_04_cola_prioridad_orden():
    """ColaProioridad entrega tareas en orden de prioridad."""
    cola = ColaProioridad()
    cola.encolar(Tarea(prioridad=5, descripcion="baja"))
    cola.encolar(Tarea(prioridad=1, descripcion="alta"))
    cola.encolar(Tarea(prioridad=3, descripcion="media"))
    primera = cola.desencolar()
    assert primera.prioridad == 1
    segunda = cola.desencolar()
    assert segunda.prioridad == 3
    print("✓ test_04 — Cola de prioridad OK")

def test_05_dependencias_en_tarea():
    """Las dependencias se registran y son accesibles."""
    t_padre = Tarea(id="p1", descripcion="tarea padre")
    t_hijo = Tarea(descripcion="tarea hijo", dependencias=["p1"])
    assert "p1" in t_hijo.dependencias
    assert len(t_hijo.dependencias) == 1
    print("✓ test_05 — Dependencias de tareas OK")

def test_06_agente_disponibilidad():
    """El agente actualiza su estado disponible durante la ejecución."""
    estados = []
    class AgenteMonitoreado(Agente):
        def ejecutar(self, tarea):
            estados.append(self.disponible)
            return super().ejecutar(tarea)
    agente = AgenteMonitoreado("M", "g", LLMSimulado())
    assert agente.disponible == True
    agente.ejecutar(Tarea())
    assert agente.disponible == True  # disponible al terminar
    print("✓ test_06 — Disponibilidad del agente OK")


if __name__ == "__main__":
    import os; os.makedirs("outputs", exist_ok=True)
    tests = [test_01_tarea_duracion_ms, test_02_agente_ejecuta_tarea, test_03_pipeline_secuencial,
             test_04_cola_prioridad_orden, test_05_dependencias_en_tarea, test_06_agente_disponibilidad]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M25 completado")
```

---

## Resumen

| Patrón | Cuándo usarlo | Ventaja | Limitación |
|---|---|---|---|
| **Pipeline** | Pasos obligatorios en orden | Simple, trazable | Un fallo detiene todo |
| **Fan-out/Fan-in** | Tareas independientes en paralelo | Velocidad (N×) | Más complejo de debuggear |
| **Supervisor-Worker** | Tareas heterogéneas con prioridades | Resiliente, flexible | Overhead de coordinación |
| **Map-Reduce** | Grandes colecciones de datos | Escalable | Requiere función de reducción |

```python
# Composición de patrones:
pipeline_con_fan_out = (
    PipelineSecuencial("Análisis completo")
    .agregar_paso("recopilar datos", data_agent)       # secuencial
    .agregar_paso("analizar (fan-out)", fan_out_agent) # interno: paralelismo
    .agregar_paso("sintetizar", synthesis_agent)       # secuencial
)
```
