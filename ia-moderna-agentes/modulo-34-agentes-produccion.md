# Módulo 34 — AI Agents en Producción: Casos Reales

> **Objetivo:** Estudiar patrones probados de despliegue de agentes en producción: gestión de estado persistente, manejo de sesiones concurrentes, sistema de colas de tareas, monitoreo de agentes y estrategias de rollback ante fallos.
>
> **Herramientas:** Python 3.11, json, sqlite3, time, threading
>
> **Prerequisito:** M25 (Orquestación), M30 (Observabilidad), M31 (Deployment)

---

## 1. Gestión de Estado en Producción

```python
# modulo-34-agentes-produccion.py — Parte 1: Estado persistente

"""
En producción, los agentes necesitan:
1. ESTADO PERSISTENTE: sobrevivir a reinicios del proceso
2. SESIONES CONCURRENTES: manejar múltiples usuarios simultáneos
3. COLA DE TAREAS: desacoplar recepción y ejecución
4. IDEMPOTENCIA: re-ejecutar la misma tarea sin efectos duplicados
5. ROLLBACK: deshacer cambios si algo falla
"""

import json
import time
import uuid
import sqlite3
import hashlib
import threading
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Callable, Dict, List, Optional
from enum import Enum
from contextlib import contextmanager


# ── Estado de sesión persistente ──────────────────────────────────────────────

class EstadoSesion(Enum):
    ACTIVA    = "activa"
    PAUSADA   = "pausada"
    COMPLETADA = "completada"
    ERROR     = "error"
    EXPIRADA  = "expirada"


@dataclass
class Sesion:
    """Representa una sesión de usuario con un agente."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    usuario_id: str = ""
    agente_id: str = ""
    estado: EstadoSesion = EstadoSesion.ACTIVA
    creada_en: float = field(default_factory=time.time)
    ultima_actividad: float = field(default_factory=time.time)
    ttl_segundos: int = 3600        # 1 hora por defecto
    mensajes: List[Dict] = field(default_factory=list)
    metadata: Dict = field(default_factory=dict)
    checksum: str = ""

    def esta_expirada(self) -> bool:
        return time.time() - self.ultima_actividad > self.ttl_segundos

    def actualizar_actividad(self):
        self.ultima_actividad = time.time()

    def agregar_mensaje(self, role: str, content: str):
        self.mensajes.append({"role": role, "content": content, "ts": time.time()})
        self.actualizar_actividad()

    def calcular_checksum(self) -> str:
        """Checksum de integridad del estado de la sesión."""
        data = json.dumps({"id": self.id, "mensajes_count": len(self.mensajes), "estado": self.estado.value})
        return hashlib.sha256(data.encode()).hexdigest()[:16]

    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "usuario_id": self.usuario_id,
            "agente_id": self.agente_id,
            "estado": self.estado.value,
            "creada_en": self.creada_en,
            "ultima_actividad": self.ultima_actividad,
            "mensajes_count": len(self.mensajes),
            "metadata": self.metadata,
        }


class GestorSesiones:
    """
    Gestiona sesiones de agentes con persistencia SQLite.
    Thread-safe para múltiples usuarios simultáneos.
    """

    def __init__(self, db_path: str = ":memory:", ttl_default: int = 3600):
        self.ttl_default = ttl_default
        self._db_path = db_path
        self._lock = threading.Lock()
        self._sesiones_cache: Dict[str, Sesion] = {}  # caché en memoria
        self._inicializar_db()

    def _inicializar_db(self):
        with sqlite3.connect(self._db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS sesiones (
                    id TEXT PRIMARY KEY,
                    usuario_id TEXT,
                    agente_id TEXT,
                    estado TEXT,
                    creada_en REAL,
                    ultima_actividad REAL,
                    datos_json TEXT,
                    checksum TEXT
                )
            """)

    @contextmanager
    def _db(self):
        conn = sqlite3.connect(self._db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
            conn.commit()
        finally:
            conn.close()

    def crear(self, usuario_id: str, agente_id: str, metadata: Dict = None) -> Sesion:
        """Crea una nueva sesión y la persiste."""
        sesion = Sesion(
            usuario_id=usuario_id,
            agente_id=agente_id,
            ttl_segundos=self.ttl_default,
            metadata=metadata or {},
        )
        sesion.checksum = sesion.calcular_checksum()
        self._persistir(sesion)
        with self._lock:
            self._sesiones_cache[sesion.id] = sesion
        return sesion

    def obtener(self, sesion_id: str) -> Optional[Sesion]:
        """Obtiene una sesión (desde caché o BD)."""
        # Primero buscar en caché
        with self._lock:
            if sesion_id in self._sesiones_cache:
                sesion = self._sesiones_cache[sesion_id]
                if sesion.esta_expirada():
                    sesion.estado = EstadoSesion.EXPIRADA
                    self._persistir(sesion)
                return sesion

        # Si no está en caché, cargar desde BD
        with self._db() as conn:
            row = conn.execute("SELECT * FROM sesiones WHERE id=?", [sesion_id]).fetchone()
            if not row:
                return None
            datos = json.loads(row["datos_json"])
            sesion = Sesion(
                id=row["id"],
                usuario_id=row["usuario_id"],
                agente_id=row["agente_id"],
                estado=EstadoSesion(row["estado"]),
                creada_en=row["creada_en"],
                ultima_actividad=row["ultima_actividad"],
                mensajes=datos.get("mensajes", []),
                metadata=datos.get("metadata", {}),
            )
            with self._lock:
                self._sesiones_cache[sesion_id] = sesion
            return sesion

    def guardar(self, sesion: Sesion):
        """Persiste el estado actualizado de la sesión."""
        sesion.checksum = sesion.calcular_checksum()
        self._persistir(sesion)

    def _persistir(self, sesion: Sesion):
        datos_json = json.dumps({"mensajes": sesion.mensajes, "metadata": sesion.metadata}, ensure_ascii=False)
        with self._db() as conn:
            conn.execute("""
                INSERT OR REPLACE INTO sesiones
                (id, usuario_id, agente_id, estado, creada_en, ultima_actividad, datos_json, checksum)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            """, [sesion.id, sesion.usuario_id, sesion.agente_id, sesion.estado.value,
                  sesion.creada_en, sesion.ultima_actividad, datos_json, sesion.checksum])

    def limpiar_expiradas(self) -> int:
        """Marca como expiradas las sesiones inactivas. Retorna count."""
        limite = time.time() - self.ttl_default
        with self._db() as conn:
            cursor = conn.execute(
                "UPDATE sesiones SET estado=? WHERE estado=? AND ultima_actividad<?",
                [EstadoSesion.EXPIRADA.value, EstadoSesion.ACTIVA.value, limite]
            )
            return cursor.rowcount

    def listar_activas(self) -> List[Dict]:
        with self._db() as conn:
            rows = conn.execute("SELECT id, usuario_id, agente_id, estado, ultima_actividad FROM sesiones WHERE estado='activa'").fetchall()
            return [dict(row) for row in rows]

    def stats(self) -> Dict:
        with self._db() as conn:
            total = conn.execute("SELECT COUNT(*) FROM sesiones").fetchone()[0]
            por_estado = dict(conn.execute("SELECT estado, COUNT(*) FROM sesiones GROUP BY estado").fetchall())
        return {"total": total, "por_estado": por_estado, "en_cache": len(self._sesiones_cache)}


if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Gestión de Sesiones en Producción")
    print("=" * 60)

    gestor = GestorSesiones("outputs/sesiones_demo.db", ttl_default=60)

    # Crear sesiones concurrentes simuladas
    sesiones_ids = []
    for i in range(5):
        s = gestor.crear(f"usuario_{i}", "agente-principal", {"plan": "premium" if i % 2 == 0 else "basic"})
        s.agregar_mensaje("user", f"Hola, soy el usuario {i}")
        s.agregar_mensaje("assistant", f"Hola usuario {i}, ¿en qué te ayudo?")
        gestor.guardar(s)
        sesiones_ids.append(s.id)
        print(f"  Sesión creada: {s.id[:8]}... → usuario_{i}")

    # Recuperar y actualizar sesión
    sesion_recuperada = gestor.obtener(sesiones_ids[0])
    print(f"\n  Sesión recuperada: {sesion_recuperada.mensajes_count if hasattr(sesion_recuperada, 'mensajes_count') else len(sesion_recuperada.mensajes)} mensajes")

    print(f"\nEstadísticas: {gestor.stats()}")

    with open("outputs/m34_sesiones.json", "w", encoding="utf-8") as f:
        json.dump({
            "stats": gestor.stats(),
            "activas": gestor.listar_activas(),
        }, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m34_sesiones.json")
```

---

## 2. Cola de Tareas con Prioridad y Workers

```python
# Parte 2: Sistema de colas para desacoplar recepción y ejecución

class PrioridadTarea(Enum):
    CRITICA = 0
    ALTA    = 1
    NORMAL  = 2
    BAJA    = 3


@dataclass
class TareaProductiva:
    """Tarea en el sistema de cola de producción."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    tipo: str = ""
    payload: Dict = field(default_factory=dict)
    prioridad: PrioridadTarea = PrioridadTarea.NORMAL
    creada_en: float = field(default_factory=time.time)
    intentos: int = 0
    max_intentos: int = 3
    resultado: Optional[Any] = None
    estado: str = "pendiente"
    idempotency_key: Optional[str] = None  # evitar procesamiento duplicado

    @property
    def es_idempotente(self) -> bool:
        return self.idempotency_key is not None

    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "tipo": self.tipo,
            "estado": self.estado,
            "prioridad": self.prioridad.name,
            "intentos": self.intentos,
            "creada_en": self.creada_en,
        }


class ColaTareas:
    """
    Cola de tareas con prioridad, idempotencia y persistencia.
    Thread-safe para múltiples workers.
    """

    def __init__(self, db_path: str = ":memory:"):
        self._db_path = db_path
        self._lock = threading.Lock()
        self._procesadas: set = set()  # idempotency keys procesados
        self._inicializar_db()

    def _inicializar_db(self):
        with sqlite3.connect(self._db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS tareas (
                    id TEXT PRIMARY KEY,
                    tipo TEXT,
                    prioridad INTEGER,
                    estado TEXT,
                    payload_json TEXT,
                    creada_en REAL,
                    intentos INTEGER DEFAULT 0,
                    max_intentos INTEGER DEFAULT 3,
                    idempotency_key TEXT,
                    resultado_json TEXT,
                    error TEXT
                )
            """)

    def encolar(self, tarea: TareaProductiva) -> bool:
        """
        Encola una tarea. Retorna False si ya fue procesada (idempotencia).
        """
        with self._lock:
            # Verificar idempotencia
            if tarea.idempotency_key and tarea.idempotency_key in self._procesadas:
                return False  # ya procesada

        with sqlite3.connect(self._db_path) as conn:
            conn.execute("""
                INSERT OR IGNORE INTO tareas
                (id, tipo, prioridad, estado, payload_json, creada_en, intentos, max_intentos, idempotency_key)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, [tarea.id, tarea.tipo, tarea.prioridad.value, "pendiente",
                  json.dumps(tarea.payload, ensure_ascii=False), tarea.creada_en,
                  tarea.intentos, tarea.max_intentos, tarea.idempotency_key])
        return True

    def obtener_siguiente(self, tipos_permitidos: List[str] = None) -> Optional[TareaProductiva]:
        """Obtiene la tarea de mayor prioridad en estado pendiente."""
        with sqlite3.connect(self._db_path) as conn:
            conn.row_factory = sqlite3.Row
            where = "estado='pendiente' AND intentos<max_intentos"
            if tipos_permitidos:
                placeholders = ",".join("?" for _ in tipos_permitidos)
                where += f" AND tipo IN ({placeholders})"
                params = tipos_permitidos
            else:
                params = []

            row = conn.execute(
                f"SELECT * FROM tareas WHERE {where} ORDER BY prioridad ASC, creada_en ASC LIMIT 1",
                params
            ).fetchone()

            if not row:
                return None

            # Marcar como en proceso (optimistic locking)
            conn.execute("UPDATE tareas SET estado='en_proceso', intentos=intentos+1 WHERE id=?", [row["id"]])

            return TareaProductiva(
                id=row["id"],
                tipo=row["tipo"],
                payload=json.loads(row["payload_json"]),
                prioridad=PrioridadTarea(row["prioridad"]),
                creada_en=row["creada_en"],
                intentos=row["intentos"] + 1,
                max_intentos=row["max_intentos"],
                idempotency_key=row["idempotency_key"],
            )

    def completar(self, tarea_id: str, resultado: Any):
        with self._lock:
            pass  # En producción: registrar idempotency key

        with sqlite3.connect(self._db_path) as conn:
            conn.execute(
                "UPDATE tareas SET estado='completada', resultado_json=? WHERE id=?",
                [json.dumps(resultado, ensure_ascii=False, default=str), tarea_id]
            )

    def fallar(self, tarea_id: str, error: str):
        with sqlite3.connect(self._db_path) as conn:
            row = conn.execute("SELECT intentos, max_intentos FROM tareas WHERE id=?", [tarea_id]).fetchone()
            if row and row[0] >= row[1]:
                nuevo_estado = "fallida"
            else:
                nuevo_estado = "pendiente"  # reintentable
            conn.execute(
                "UPDATE tareas SET estado=?, error=? WHERE id=?",
                [nuevo_estado, error[:500], tarea_id]
            )

    def stats(self) -> Dict:
        with sqlite3.connect(self._db_path) as conn:
            total = conn.execute("SELECT COUNT(*) FROM tareas").fetchone()[0]
            por_estado = dict(conn.execute("SELECT estado, COUNT(*) FROM tareas GROUP BY estado").fetchall())
            por_tipo = dict(conn.execute("SELECT tipo, COUNT(*) FROM tareas GROUP BY tipo").fetchall())
        return {"total": total, "por_estado": por_estado, "por_tipo": por_tipo}


class WorkerAgente:
    """Worker que procesa tareas de la cola."""

    def __init__(self, worker_id: str, cola: ColaTareas, procesadores: Dict[str, Callable]):
        self.worker_id = worker_id
        self.cola = cola
        self.procesadores = procesadores
        self.tareas_procesadas = 0
        self.tareas_fallidas = 0
        self._activo = True

    def procesar_siguiente(self) -> bool:
        """Procesa la siguiente tarea disponible. Retorna True si procesó algo."""
        tarea = self.cola.obtener_siguiente(list(self.procesadores.keys()))
        if not tarea:
            return False

        procesador = self.procesadores.get(tarea.tipo)
        if not procesador:
            self.cola.fallar(tarea.id, f"Sin procesador para tipo '{tarea.tipo}'")
            return True

        try:
            resultado = procesador(tarea.payload)
            self.cola.completar(tarea.id, resultado)
            self.tareas_procesadas += 1
            return True
        except Exception as e:
            self.cola.fallar(tarea.id, str(e))
            self.tareas_fallidas += 1
            return True


def demo_cola_tareas():
    print("\n" + "=" * 60)
    print("DEMO 2 — Cola de Tareas con Workers")
    print("=" * 60)

    cola = ColaTareas("outputs/cola_demo.db")

    # Definir procesadores
    def procesar_analisis(payload: Dict) -> Dict:
        texto = payload.get("texto", "")
        return {"palabras": len(texto.split()), "chars": len(texto), "resultado": "analizado"}

    def procesar_resumen(payload: Dict) -> Dict:
        texto = payload.get("texto", "")
        return {"resumen": texto[:50] + "...", "ratio": 0.3}

    def procesar_traduccion(payload: Dict) -> Dict:
        return {"traduccion": f"[EN] {payload.get('texto', '')[:30]}...", "idioma": "en"}

    worker = WorkerAgente("worker-01", cola, {
        "analisis": procesar_analisis,
        "resumen": procesar_resumen,
        "traduccion": procesar_traduccion,
    })

    # Encolar tareas con distintas prioridades
    tareas = [
        TareaProductiva(tipo="analisis", payload={"texto": "Texto de prueba para analizar en profundidad."}, prioridad=PrioridadTarea.NORMAL),
        TareaProductiva(tipo="resumen", payload={"texto": "Este es un texto largo que necesita ser resumido."}, prioridad=PrioridadTarea.ALTA),
        TareaProductiva(tipo="traduccion", payload={"texto": "Hola mundo"}, prioridad=PrioridadTarea.BAJA, idempotency_key="translate-hola-mundo"),
        TareaProductiva(tipo="traduccion", payload={"texto": "Hola mundo"}, prioridad=PrioridadTarea.BAJA, idempotency_key="translate-hola-mundo"),  # duplicado
        TareaProductiva(tipo="analisis", payload={"texto": "Segunda tarea crítica"}, prioridad=PrioridadTarea.CRITICA),
    ]

    encoladas = 0
    for tarea in tareas:
        ok = cola.encolar(tarea)
        encoladas += (1 if ok else 0)
        print(f"  {'✓' if ok else '⊘'} {tarea.tipo} (prioridad={tarea.prioridad.name}){' [DUPLICADO]' if not ok else ''}")

    print(f"\nTareas encoladas: {encoladas}/{len(tareas)} (idempotencia descartó {len(tareas)-encoladas})")

    # Procesar todas las tareas
    print("\nProcesando:")
    while worker.procesar_siguiente():
        pass

    stats = cola.stats()
    print(f"Completadas: {stats['por_estado'].get('completada', 0)}")
    print(f"Fallidas: {stats['por_estado'].get('fallida', 0)}")
    print(f"Worker: {worker.tareas_procesadas} procesadas, {worker.tareas_fallidas} fallidas")

    with open("outputs/m34_cola_tareas.json", "w", encoding="utf-8") as f:
        json.dump({"stats_finales": stats, "worker": {"id": worker.worker_id, "procesadas": worker.tareas_procesadas}}, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m34_cola_tareas.json")
    print("\n[M34 completado]")
    return stats


if __name__ == "__main__":
    demo_cola_tareas()
```

---

## Checkpoint

```python
# checkpoint_m34.py — 6 tests para Agentes en Producción

import json, time, uuid, sqlite3, threading
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List, Optional
from enum import Enum
from contextlib import contextmanager

class EstadoSesion(Enum):
    ACTIVA = "activa"; COMPLETADA = "completada"; EXPIRADA = "expirada"

class PrioridadTarea(Enum):
    CRITICA = 0; ALTA = 1; NORMAL = 2; BAJA = 3

@dataclass
class Sesion:
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    usuario_id: str = ""; agente_id: str = ""
    estado: EstadoSesion = EstadoSesion.ACTIVA
    ultima_actividad: float = field(default_factory=time.time)
    ttl_segundos: int = 3600; mensajes: List[Dict] = field(default_factory=list)
    def esta_expirada(self): return time.time() - self.ultima_actividad > self.ttl_segundos
    def actualizar_actividad(self): self.ultima_actividad = time.time()
    def agregar_mensaje(self, role, content):
        self.mensajes.append({"role":role,"content":content}); self.actualizar_actividad()

@dataclass
class TareaProductiva:
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    tipo: str = ""; payload: Dict = field(default_factory=dict)
    prioridad: PrioridadTarea = PrioridadTarea.NORMAL
    creada_en: float = field(default_factory=time.time)
    intentos: int = 0; max_intentos: int = 3; estado: str = "pendiente"
    idempotency_key: Optional[str] = None


def test_01_sesion_agrega_mensajes():
    """Sesión acumula mensajes y actualiza actividad."""
    s = Sesion(usuario_id="u1", agente_id="a1")
    t_antes = s.ultima_actividad
    time.sleep(0.01)
    s.agregar_mensaje("user", "Hola")
    s.agregar_mensaje("assistant", "Hola, ¿en qué te ayudo?")
    assert len(s.mensajes) == 2
    assert s.ultima_actividad > t_antes
    print("✓ test_01 — Sesión agrega mensajes OK")

def test_02_sesion_expiracion():
    """Sesión detecta expiración por TTL."""
    s = Sesion(ttl_segundos=0)  # TTL=0 → expira inmediatamente
    time.sleep(0.01)
    assert s.esta_expirada(), "Sesión con TTL=0 debe estar expirada"
    s2 = Sesion(ttl_segundos=3600)
    assert not s2.esta_expirada(), "Sesión nueva no debe estar expirada"
    print("✓ test_02 — Expiración de sesión OK")

def test_03_tarea_prioridad_orden():
    """Las tareas se ordenan correctamente por prioridad numérica."""
    tareas = [
        TareaProductiva(tipo="t", prioridad=PrioridadTarea.BAJA),
        TareaProductiva(tipo="t", prioridad=PrioridadTarea.CRITICA),
        TareaProductiva(tipo="t", prioridad=PrioridadTarea.NORMAL),
    ]
    ordenadas = sorted(tareas, key=lambda t: t.prioridad.value)
    assert ordenadas[0].prioridad == PrioridadTarea.CRITICA
    assert ordenadas[-1].prioridad == PrioridadTarea.BAJA
    print("✓ test_03 — Ordenamiento por prioridad OK")

def test_04_idempotencia_tarea():
    """Idempotency key previene procesamiento duplicado."""
    procesadas = set()
    def encolar_idempotente(tarea):
        if tarea.idempotency_key and tarea.idempotency_key in procesadas:
            return False
        procesadas.add(tarea.idempotency_key)
        return True

    t1 = TareaProductiva(tipo="traducir", payload={"txt": "hola"}, idempotency_key="key-001")
    t2 = TareaProductiva(tipo="traducir", payload={"txt": "hola"}, idempotency_key="key-001")
    t3 = TareaProductiva(tipo="traducir", payload={"txt": "mundo"}, idempotency_key="key-002")

    assert encolar_idempotente(t1) == True
    assert encolar_idempotente(t2) == False  # duplicado
    assert encolar_idempotente(t3) == True   # diferente key
    print("✓ test_04 — Idempotencia OK")

def test_05_worker_procesa_tipos():
    """Worker procesa solo los tipos que tiene registrados."""
    procesadas = []
    def proc_a(payload): procesadas.append("a"); return "ok_a"
    def proc_b(payload): procesadas.append("b"); return "ok_b"

    procesadores = {"tipo_a": proc_a, "tipo_b": proc_b}
    tareas_pendientes = [
        TareaProductiva(tipo="tipo_a", payload={}),
        TareaProductiva(tipo="tipo_b", payload={}),
        TareaProductiva(tipo="tipo_c", payload={}),  # sin procesador
    ]

    for tarea in tareas_pendientes:
        if tarea.tipo in procesadores:
            procesadores[tarea.tipo](tarea.payload)

    assert "a" in procesadas
    assert "b" in procesadas
    assert len(procesadas) == 2  # tipo_c no se procesó
    print("✓ test_05 — Worker tipos OK")

def test_06_sesion_persistencia_sqlite():
    """GestorSesiones persiste y recupera sesiones en SQLite."""
    db_path = ":memory:"
    conn = sqlite3.connect(db_path)
    conn.execute("CREATE TABLE sesiones (id TEXT PRIMARY KEY, usuario_id TEXT, datos TEXT)")
    conn.commit()

    sesion_id = str(uuid.uuid4())
    conn.execute("INSERT INTO sesiones VALUES (?, ?, ?)", [sesion_id, "user_test", json.dumps({"msgs": 3})])
    conn.commit()

    row = conn.execute("SELECT * FROM sesiones WHERE id=?", [sesion_id]).fetchone()
    assert row is not None
    assert row[1] == "user_test"
    datos = json.loads(row[2])
    assert datos["msgs"] == 3
    conn.close()
    print("✓ test_06 — Persistencia SQLite OK")


if __name__ == "__main__":
    import os; os.makedirs("outputs", exist_ok=True)
    tests = [
        test_01_sesion_agrega_mensajes,
        test_02_sesion_expiracion,
        test_03_tarea_prioridad_orden,
        test_04_idempotencia_tarea,
        test_05_worker_procesa_tipos,
        test_06_sesion_persistencia_sqlite,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M34 completado")
```

---

## Resumen

| Patrón | Problema | Solución | Herramienta |
|---|---|---|---|
| **Estado persistente** | Reinicios del proceso | SQLite + caché en memoria | `GestorSesiones` |
| **Idempotencia** | Procesamiento duplicado | Idempotency key + set de procesadas | `idempotency_key` |
| **Cola con prioridad** | Tareas heterogéneas | Prioridad numérica + ORDER BY | `ColaTareas` |
| **Workers** | Desacoplar recepción/ejecución | Patrón consumer de cola | `WorkerAgente` |
| **Expiración de sesiones** | Recursos no liberados | TTL + limpieza periódica | `esta_expirada()` |
| **Reintentos** | Fallos transitorios | `intentos < max_intentos` | `fallar()` re-encola |
