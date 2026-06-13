# Módulo 30 — Observabilidad: LangSmith, Langfuse, Trazas

## Encabezado

- **Objetivo**: Implementar sistemas de observabilidad para LLMs incluyendo tracing de spans, recolección de métricas de latencia (percentiles p50/p95/p99), alertas configurables y dashboards de monitoreo.
- **Herramientas**: Python stdlib (time, json, math, collections, dataclasses, typing, pathlib, hashlib, contextlib, enum)
- **Prerequisito**: Módulos 16–27 (agentes, orquestación), Módulo 28

---

## Sección 1: Concepto de Tracing en Sistemas LLM

El tracing permite entender el flujo de ejecución de un sistema LLM, identificar cuellos de botella y depurar problemas. Cada operación se modela como un **Span** (unidad de trabajo con inicio/fin), y los spans se organizan en **Traces** (árboles de spans).

```python
import time
import json
import hashlib
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Any
from pathlib import Path


# ── Span ───────────────────────────────────────────────────────────────────────
@dataclass
class Span:
    """
    Unidad de trabajo en el sistema de tracing.
    Representa una operacion con inicio, fin, inputs, outputs y metadatos.
    """
    nombre: str
    span_id: str = ""
    trace_id: str = ""
    parent_id: str = ""
    start_time: float = 0.0
    end_time: float = 0.0
    inputs: Dict = field(default_factory=dict)
    outputs: Dict = field(default_factory=dict)
    errores: List[str] = field(default_factory=list)
    metadata: Dict = field(default_factory=dict)
    hijos: List["Span"] = field(default_factory=list)

    def __post_init__(self):
        if not self.span_id:
            self.span_id = hashlib.md5(
                (self.nombre + str(time.time())).encode()
            ).hexdigest()[:12]
        if not self.start_time:
            self.start_time = time.time()

    def finalizar(self, outputs: Dict = None, error: str = None):
        """Cierra el span registrando tiempo de fin y resultado."""
        self.end_time = time.time()
        if outputs:
            self.outputs = outputs
        if error:
            self.errores.append(error)

    @property
    def duracion_ms(self) -> float:
        if self.end_time and self.start_time:
            return round((self.end_time - self.start_time) * 1000, 2)
        return 0.0

    @property
    def estado(self) -> str:
        if self.errores:
            return "ERROR"
        if self.end_time:
            return "COMPLETADO"
        return "EN_PROGRESO"

    def agregar_hijo(self, span: "Span"):
        span.parent_id = self.span_id
        span.trace_id = self.trace_id
        self.hijos.append(span)

    def to_dict(self) -> Dict:
        return {
            "span_id": self.span_id,
            "nombre": self.nombre,
            "trace_id": self.trace_id,
            "parent_id": self.parent_id,
            "start_time": self.start_time,
            "end_time": self.end_time,
            "duracion_ms": self.duracion_ms,
            "estado": self.estado,
            "inputs": self.inputs,
            "outputs": self.outputs,
            "errores": self.errores,
            "metadata": self.metadata,
            "hijos": [h.to_dict() for h in self.hijos],
        }


# ── Trace ──────────────────────────────────────────────────────────────────────
class Trace:
    """
    Contenedor de spans anidados que forman un arbol.
    Representa una solicitud o pipeline completo.
    """

    def __init__(self, nombre: str, metadata: Dict = None):
        self.trace_id = hashlib.md5(
            (nombre + str(time.time())).encode()
        ).hexdigest()[:16]
        self.nombre = nombre
        self.metadata = metadata or {}
        self.span_raiz: Optional[Span] = None
        self.todos_spans: List[Span] = []
        self.start_time = time.time()
        self.end_time: Optional[float] = None

    def crear_span(self, nombre: str, parent: Optional[Span] = None, inputs: Dict = None) -> Span:
        """Crea un nuevo span, opcionalmente como hijo de otro."""
        span = Span(
            nombre=nombre,
            trace_id=self.trace_id,
            inputs=inputs or {},
        )

        if parent:
            parent.agregar_hijo(span)
        elif self.span_raiz is None:
            self.span_raiz = span

        self.todos_spans.append(span)
        return span

    def finalizar(self):
        self.end_time = time.time()

    @property
    def duracion_total_ms(self) -> float:
        if self.end_time:
            return round((self.end_time - self.start_time) * 1000, 2)
        return 0.0

    @property
    def tiene_errores(self) -> bool:
        return any(s.errores for s in self.todos_spans)

    @property
    def n_spans(self) -> int:
        return len(self.todos_spans)

    def to_dict(self) -> Dict:
        return {
            "trace_id": self.trace_id,
            "nombre": self.nombre,
            "duracion_total_ms": self.duracion_total_ms,
            "n_spans": self.n_spans,
            "tiene_errores": self.tiene_errores,
            "metadata": self.metadata,
            "span_raiz": self.span_raiz.to_dict() if self.span_raiz else None,
        }


# ── Demo Sección 1 ─────────────────────────────────────────────────────────────
print("=== Tracing: Spans y Traces ===\n")

# Simular un pipeline RAG con spans anidados
trace = Trace("pipeline-rag", metadata={"usuario": "demo", "query": "que es un transformer"})

# Span raiz: pipeline completo
span_pipeline = trace.crear_span(
    "pipeline_rag_completo",
    inputs={"query": "que es un transformer"}
)
time.sleep(0.01)  # Simular trabajo

# Span hijo: recuperacion de documentos
span_retrieval = trace.crear_span(
    "recuperar_documentos",
    parent=span_pipeline,
    inputs={"query": "que es un transformer", "top_k": 3}
)
time.sleep(0.02)
span_retrieval.finalizar(outputs={"n_documentos": 3, "scores": [0.92, 0.87, 0.81]})

# Span hijo: llamada LLM
span_llm = trace.crear_span(
    "llamada_llm",
    parent=span_pipeline,
    inputs={"prompt_tokens": 512, "modelo": "llm-simulado"}
)
time.sleep(0.03)
span_llm.finalizar(outputs={"completion_tokens": 128, "finish_reason": "stop"})
span_llm.metadata["costo_usd"] = 0.00064

# Span hijo: postprocesado
span_post = trace.crear_span("postprocesar_respuesta", parent=span_pipeline)
time.sleep(0.005)
span_post.finalizar(outputs={"longitud_respuesta": 256})

# Cerrar span raiz
span_pipeline.finalizar(outputs={"respuesta": "El transformer es..."})
trace.finalizar()

print("Trace ID:", trace.trace_id)
print("Duracion total:", trace.duracion_total_ms, "ms")
print("N spans:", trace.n_spans)
print("Tiene errores:", trace.tiene_errores)
print("\nSpans:")
for s in trace.todos_spans:
    prefijo = "  -> " if s.parent_id else "* "
    print("  {}{}: {} ms [{}]".format(prefijo, s.nombre, s.duracion_ms, s.estado))
```

---

## Sección 2: Implementación de Tracer

Un tracer completo usa el patrón context manager para abrir y cerrar spans automáticamente, capturando latencia, tokens y costos.

```python
import time
import json
import hashlib
from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Generator
from pathlib import Path


@dataclass
class Span:
    nombre: str
    span_id: str = ""
    trace_id: str = ""
    parent_id: str = ""
    start_time: float = 0.0
    end_time: float = 0.0
    inputs: Dict = field(default_factory=dict)
    outputs: Dict = field(default_factory=dict)
    errores: List[str] = field(default_factory=list)
    metadata: Dict = field(default_factory=dict)

    def __post_init__(self):
        if not self.span_id:
            self.span_id = hashlib.md5((self.nombre+str(time.time())).encode()).hexdigest()[:12]
        if not self.start_time:
            self.start_time = time.time()

    @property
    def duracion_ms(self):
        return round((self.end_time - self.start_time)*1000, 2) if self.end_time else 0.0

    @property
    def estado(self):
        return "ERROR" if self.errores else ("COMPLETADO" if self.end_time else "EN_PROGRESO")

    def to_dict(self):
        return {
            "span_id": self.span_id, "nombre": self.nombre, "trace_id": self.trace_id,
            "parent_id": self.parent_id, "duracion_ms": self.duracion_ms, "estado": self.estado,
            "inputs": self.inputs, "outputs": self.outputs, "errores": self.errores,
            "metadata": self.metadata,
        }


# ── Modelo de costos ───────────────────────────────────────────────────────────
COSTO_POR_1K_TOKENS = 0.001  # $0.001 por 1000 tokens (simulado)

def calcular_costo(tokens_entrada: int, tokens_salida: int) -> float:
    """Calcula costo simulado en USD."""
    total_tokens = tokens_entrada + tokens_salida
    return round(total_tokens / 1000 * COSTO_POR_1K_TOKENS, 6)


# ── LLMTracer ─────────────────────────────────────────────────────────────────
class LLMTracer:
    """
    Tracer para sistemas LLM con context manager para manejo automatico de spans.
    Captura latencia, tokens y costos.
    """

    def __init__(self, nombre_servicio: str = "llm-service"):
        self.nombre_servicio = nombre_servicio
        self.trazas: List[Dict] = []
        self._span_stack: List[Span] = []
        self._trace_id_actual: str = ""
        self._spans_actuales: List[Span] = []

    def nueva_traza(self, nombre: str, metadata: Dict = None) -> str:
        """Inicia una nueva traza y retorna su ID."""
        self._trace_id_actual = hashlib.md5(
            (nombre + str(time.time())).encode()
        ).hexdigest()[:16]
        self._span_stack = []
        self._spans_actuales = []
        return self._trace_id_actual

    @contextmanager
    def span(self, nombre: str, inputs: Dict = None) -> Generator[Span, None, None]:
        """
        Context manager que abre un span al entrar y lo cierra al salir.
        Maneja automáticamente jerarquia de spans.
        """
        parent_id = self._span_stack[-1].span_id if self._span_stack else ""
        s = Span(
            nombre=nombre,
            trace_id=self._trace_id_actual,
            parent_id=parent_id,
            inputs=inputs or {},
        )
        self._span_stack.append(s)
        self._spans_actuales.append(s)

        try:
            yield s
        except Exception as e:
            s.errores.append(str(e))
            raise
        finally:
            s.end_time = time.time()
            self._span_stack.pop()

    def finalizar_traza(self, metadata: Dict = None):
        """Cierra la traza actual y la almacena."""
        traza = {
            "trace_id": self._trace_id_actual,
            "servicio": self.nombre_servicio,
            "timestamp": time.time(),
            "metadata": metadata or {},
            "spans": [s.to_dict() for s in self._spans_actuales],
            "duracion_total_ms": sum(
                s.duracion_ms for s in self._spans_actuales if not s.parent_id
            ),
            "tiene_errores": any(s.errores for s in self._spans_actuales),
            "n_spans": len(self._spans_actuales),
        }

        # Agregar metricas de tokens y costos
        tokens_entrada = sum(
            s.inputs.get("tokens", 0) for s in self._spans_actuales
        )
        tokens_salida = sum(
            s.outputs.get("tokens", 0) for s in self._spans_actuales
        )
        traza["tokens_entrada"] = tokens_entrada
        traza["tokens_salida"] = tokens_salida
        traza["costo_usd"] = calcular_costo(tokens_entrada, tokens_salida)

        self.trazas.append(traza)
        return traza

    def obtener_metricas_resumen(self) -> Dict:
        """Retorna resumen de todas las trazas registradas."""
        if not self.trazas:
            return {"error": "Sin trazas registradas"}

        latencias = [t["duracion_total_ms"] for t in self.trazas if t["duracion_total_ms"] > 0]
        errores = sum(1 for t in self.trazas if t["tiene_errores"])
        costo_total = sum(t.get("costo_usd", 0) for t in self.trazas)

        latencias_sorted = sorted(latencias)
        n = len(latencias_sorted)

        def percentil(data, p):
            if not data:
                return 0.0
            idx = int(p / 100 * len(data))
            return data[min(idx, len(data)-1)]

        return {
            "total_trazas": len(self.trazas),
            "trazas_con_error": errores,
            "tasa_error": round(errores / len(self.trazas), 3),
            "latencia_p50_ms": percentil(latencias_sorted, 50),
            "latencia_p95_ms": percentil(latencias_sorted, 95),
            "latencia_p99_ms": percentil(latencias_sorted, 99),
            "costo_total_usd": round(costo_total, 6),
            "costo_promedio_usd": round(costo_total / len(self.trazas), 6),
        }


# ── Demo Sección 2 ─────────────────────────────────────────────────────────────
print("=== LLMTracer con Context Manager ===\n")

tracer = LLMTracer(nombre_servicio="rag-service-demo")

# Simular múltiples trazas
for i in range(5):
    tid = tracer.nueva_traza("consulta_rag_" + str(i))

    with tracer.span("pipeline_completo", inputs={"query": "pregunta_" + str(i)}) as span_root:
        with tracer.span("embedder", inputs={"tokens": 50 + i*10}) as span_emb:
            time.sleep(0.005 + i*0.002)
            span_emb.outputs = {"embedding_dim": 768, "tokens": 50 + i*10}

        with tracer.span("retriever") as span_ret:
            time.sleep(0.010 + i*0.003)
            span_ret.outputs = {"n_docs": 3}

        with tracer.span("llm_call", inputs={"tokens": 400 + i*20}) as span_llm:
            time.sleep(0.020 + i*0.005)
            span_llm.outputs = {"tokens": 100 + i*5, "finish_reason": "stop"}
            span_llm.metadata["modelo"] = "llm-simulado"

        span_root.outputs = {"respuesta_generada": True}

    traza = tracer.finalizar_traza(metadata={"iteracion": i})

print("Trazas registradas:", len(tracer.trazas))
metricas = tracer.obtener_metricas_resumen()
print("\nResumen de metricas:")
for k, v in metricas.items():
    print("  {}: {}".format(k, v))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m30_tracer.json", "w", encoding="utf-8") as f:
    json.dump({"metricas": metricas, "n_trazas": len(tracer.trazas)}, f, indent=2)
print("\nGuardado en outputs/m30_tracer.json")
```

---

## Sección 3: Métricas y Alertas

Los sistemas de producción necesitan monitoreo continuo con alertas que notifiquen cuando métricas clave se degradan.

```python
import math
import time
import json
from dataclasses import dataclass, field
from typing import List, Dict, Callable, Optional
from collections import defaultdict, deque
from pathlib import Path


# ── MetricsCollector ───────────────────────────────────────────────────────────
class MetricsCollector:
    """
    Recopila métricas de latencia, tokens y errores.
    Calcula percentiles p50, p95, p99.
    """

    def __init__(self, ventana_max: int = 1000):
        self.ventana_max = ventana_max
        self._latencias: deque = deque(maxlen=ventana_max)
        self._errores: deque = deque(maxlen=ventana_max)
        self._tokens: deque = deque(maxlen=ventana_max)
        self._costos: deque = deque(maxlen=ventana_max)
        self.total_requests: int = 0
        self.total_errores: int = 0

    def registrar(self, latencia_ms: float, tokens: int = 0, error: bool = False, costo: float = 0.0):
        """Registra una observacion."""
        self._latencias.append(latencia_ms)
        self._errores.append(1 if error else 0)
        self._tokens.append(tokens)
        self._costos.append(costo)
        self.total_requests += 1
        if error:
            self.total_errores += 1

    @staticmethod
    def _percentil(datos: List[float], p: int) -> float:
        """Calcula percentil p de una lista de datos."""
        if not datos:
            return 0.0
        datos_sorted = sorted(datos)
        idx = math.ceil(p / 100 * len(datos_sorted)) - 1
        return round(datos_sorted[max(0, idx)], 2)

    def estadisticas_latencia(self) -> Dict:
        """Retorna estadisticas de latencia: p50, p95, p99, min, max, promedio."""
        datos = list(self._latencias)
        if not datos:
            return {}
        return {
            "p50_ms": self._percentil(datos, 50),
            "p95_ms": self._percentil(datos, 95),
            "p99_ms": self._percentil(datos, 99),
            "min_ms": round(min(datos), 2),
            "max_ms": round(max(datos), 2),
            "promedio_ms": round(sum(datos) / len(datos), 2),
            "n_observaciones": len(datos),
        }

    def tasa_error(self) -> float:
        """Tasa de error en la ventana actual."""
        errores = list(self._errores)
        if not errores:
            return 0.0
        return round(sum(errores) / len(errores), 4)

    def throughput_tokens(self) -> float:
        """Promedio de tokens por request."""
        tokens = list(self._tokens)
        if not tokens:
            return 0.0
        return round(sum(tokens) / len(tokens), 2)

    def resumen(self) -> Dict:
        return {
            "latencias": self.estadisticas_latencia(),
            "tasa_error": self.tasa_error(),
            "tokens_promedio": self.throughput_tokens(),
            "total_requests": self.total_requests,
            "total_errores": self.total_errores,
            "costo_total_usd": round(sum(self._costos), 6),
        }


# ── AlertManager ───────────────────────────────────────────────────────────────
@dataclass
class ReglaAlerta:
    nombre: str
    metrica: str           # "p95_latencia", "tasa_error", "costo_hora"
    umbral: float
    operador: str          # ">", "<", ">="
    severidad: str         # "WARNING", "CRITICAL"
    mensaje_template: str  # Template del mensaje (sin llaves)
    cooldown_segundos: int = 60
    _ultimo_disparo: float = field(default=0.0, repr=False)


class AlertManager:
    """
    Gestor de alertas con reglas configurables.
    Evalua métricas contra umbrales y dispara alertas.
    """

    def __init__(self):
        self.reglas: List[ReglaAlerta] = []
        self.alertas_disparadas: List[Dict] = []
        self._registrar_reglas_default()

    def _registrar_reglas_default(self):
        self.reglas = [
            ReglaAlerta(
                nombre="latencia_p95_alta",
                metrica="p95_ms",
                umbral=500.0,
                operador=">",
                severidad="WARNING",
                mensaje_template="Latencia P95 supera 500ms",
                cooldown_segundos=30,
            ),
            ReglaAlerta(
                nombre="latencia_p99_critica",
                metrica="p99_ms",
                umbral=2000.0,
                operador=">",
                severidad="CRITICAL",
                mensaje_template="Latencia P99 supera 2000ms - degradacion severa",
                cooldown_segundos=10,
            ),
            ReglaAlerta(
                nombre="tasa_error_alta",
                metrica="tasa_error",
                umbral=0.05,
                operador=">",
                severidad="WARNING",
                mensaje_template="Tasa de error supera 5%",
                cooldown_segundos=60,
            ),
            ReglaAlerta(
                nombre="tasa_error_critica",
                metrica="tasa_error",
                umbral=0.20,
                operador=">",
                severidad="CRITICAL",
                mensaje_template="Tasa de error supera 20% - sistema degradado",
                cooldown_segundos=10,
            ),
        ]

    def agregar_regla(self, regla: ReglaAlerta):
        self.reglas.append(regla)

    def _evaluar_condicion(self, valor: float, umbral: float, operador: str) -> bool:
        if operador == ">":
            return valor > umbral
        if operador == ">=":
            return valor >= umbral
        if operador == "<":
            return valor < umbral
        if operador == "<=":
            return valor <= umbral
        return False

    def evaluar(self, metricas: Dict) -> List[Dict]:
        """Evalua métricas contra reglas y retorna alertas activas."""
        alertas_nuevas = []
        ahora = time.time()

        # Aplanar metricas anidadas
        metricas_planas = {}
        for k, v in metricas.items():
            if isinstance(v, dict):
                for k2, v2 in v.items():
                    metricas_planas[k2] = v2
            else:
                metricas_planas[k] = v

        for regla in self.reglas:
            valor = metricas_planas.get(regla.metrica)
            if valor is None:
                continue

            if not self._evaluar_condicion(valor, regla.umbral, regla.operador):
                continue

            # Verificar cooldown
            if ahora - regla._ultimo_disparo < regla.cooldown_segundos:
                continue

            regla._ultimo_disparo = ahora
            alerta = {
                "nombre": regla.nombre,
                "severidad": regla.severidad,
                "mensaje": regla.mensaje_template,
                "metrica": regla.metrica,
                "valor_actual": valor,
                "umbral": regla.umbral,
                "timestamp": ahora,
            }
            alertas_nuevas.append(alerta)
            self.alertas_disparadas.append(alerta)

        return alertas_nuevas


# ── Demo Sección 3 ─────────────────────────────────────────────────────────────
print("=== MetricsCollector y AlertManager ===\n")

collector = MetricsCollector(ventana_max=100)
alert_mgr = AlertManager()

# Simular requests con distintas latencias
import random
random.seed(42)

latencias_simuladas = (
    [random.uniform(50, 200) for _ in range(80)] +   # normales
    [random.uniform(600, 1000) for _ in range(15)] +  # lentas
    [random.uniform(2500, 4000) for _ in range(5)]    # criticas
)

for i, lat in enumerate(latencias_simuladas):
    es_error = lat > 2500  # Error si latencia muy alta
    collector.registrar(
        latencia_ms=lat,
        tokens=random.randint(200, 800),
        error=es_error,
        costo=lat * 0.000001,
    )

resumen = collector.resumen()
print("Estadisticas de latencia:")
for k, v in resumen["latencias"].items():
    print("  {}: {}".format(k, v))
print("Tasa de error:", resumen["tasa_error"])
print("Tokens promedio:", resumen["tokens_promedio"])

print("\nEvaluando alertas...")
alertas = alert_mgr.evaluar(resumen)
if alertas:
    for a in alertas:
        print("  [{}] {} (valor={}, umbral={})".format(
            a["severidad"], a["mensaje"], round(a["valor_actual"], 2), a["umbral"]
        ))
else:
    print("  Sin alertas activas")

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m30_metricas.json", "w", encoding="utf-8") as f:
    json.dump({"resumen": resumen, "alertas": alertas}, f, indent=2)
print("\nGuardado en outputs/m30_metricas.json")
```

---

## Sección 4: Dashboard de Observabilidad Simulado

El dashboard consolida toda la información de observabilidad en un reporte ejecutivo.

```python
import time
import json
import math
import hashlib
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from collections import deque, defaultdict
from pathlib import Path


def _percentil(datos, p):
    if not datos:
        return 0.0
    s = sorted(datos)
    idx = math.ceil(p / 100 * len(s)) - 1
    return round(s[max(0, idx)], 2)


def generar_reporte(trazas: List[Dict]) -> Dict:
    """
    Genera un JSON con resumen ejecutivo de observabilidad.
    Incluye: latencias, errores, costos, spans mas lentos, tendencias.
    """
    if not trazas:
        return {"error": "Sin trazas disponibles"}

    latencias = [t.get("duracion_total_ms", 0) for t in trazas if t.get("duracion_total_ms", 0) > 0]
    errores = [t for t in trazas if t.get("tiene_errores", False)]
    costos = [t.get("costo_usd", 0) for t in trazas]
    tokens_entrada = [t.get("tokens_entrada", 0) for t in trazas]
    tokens_salida = [t.get("tokens_salida", 0) for t in trazas]

    # Estadisticas de latencia
    stats_latencia = {
        "p50_ms": _percentil(latencias, 50),
        "p95_ms": _percentil(latencias, 95),
        "p99_ms": _percentil(latencias, 99),
        "promedio_ms": round(sum(latencias)/len(latencias), 2) if latencias else 0,
        "max_ms": round(max(latencias), 2) if latencias else 0,
    }

    # Distribucion de estados
    estados = defaultdict(int)
    for t in trazas:
        estados["error" if t.get("tiene_errores") else "ok"] += 1

    # Spans mas lentos
    todos_spans = []
    for t in trazas:
        for s in t.get("spans", []):
            todos_spans.append({
                "nombre": s.get("nombre", ""),
                "duracion_ms": s.get("duracion_ms", 0),
                "trace_id": s.get("trace_id", ""),
            })
    spans_lentos = sorted(todos_spans, key=lambda x: -x["duracion_ms"])[:5]

    # Tendencia: primera mitad vs segunda mitad de latencias
    mid = len(latencias) // 2
    if mid > 0:
        tendencia_latencia = round(
            (sum(latencias[mid:]) / len(latencias[mid:])) - (sum(latencias[:mid]) / len(latencias[:mid])),
            2
        ) if latencias[mid:] and latencias[:mid] else 0.0
    else:
        tendencia_latencia = 0.0

    reporte = {
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "periodo_analisis": {
            "n_trazas": len(trazas),
            "primera_traza": trazas[0].get("timestamp", 0),
            "ultima_traza": trazas[-1].get("timestamp", 0),
        },
        "resumen_ejecutivo": {
            "total_requests": len(trazas),
            "tasa_exito": round(estados["ok"] / len(trazas), 4),
            "tasa_error": round(estados["error"] / len(trazas), 4),
            "costo_total_usd": round(sum(costos), 6),
            "costo_promedio_usd": round(sum(costos)/len(costos), 6) if costos else 0,
            "tokens_totales": sum(tokens_entrada) + sum(tokens_salida),
        },
        "latencias": stats_latencia,
        "tendencias": {
            "delta_latencia_ms": tendencia_latencia,
            "tendencia": "EMPEORANDO" if tendencia_latencia > 50 else (
                "MEJORANDO" if tendencia_latencia < -50 else "ESTABLE"
            ),
        },
        "top_5_spans_lentos": spans_lentos,
        "alertas_activas": [],
        "recomendaciones": [],
    }

    # Generar alertas automaticas
    if stats_latencia["p99_ms"] > 2000:
        reporte["alertas_activas"].append({
            "nivel": "CRITICAL",
            "mensaje": "P99 latencia supera 2000ms - revisar modelo o infraestructura",
        })
    if reporte["resumen_ejecutivo"]["tasa_error"] > 0.05:
        reporte["alertas_activas"].append({
            "nivel": "WARNING",
            "mensaje": "Tasa de error supera 5%",
        })

    # Recomendaciones
    if stats_latencia["p95_ms"] > 500:
        reporte["recomendaciones"].append("Considerar caching de respuestas frecuentes")
    if reporte["resumen_ejecutivo"]["costo_total_usd"] > 0.01:
        reporte["recomendaciones"].append("Revisar longitud de prompts para reducir costos")
    if not reporte["recomendaciones"]:
        reporte["recomendaciones"].append("Sistema operando dentro de parametros normales")

    return reporte


# ── Generar datos simulados ────────────────────────────────────────────────────
import random
random.seed(123)

def simular_trazas(n: int) -> List[Dict]:
    """Genera N trazas simuladas para el dashboard."""
    trazas = []
    base_time = time.time() - n * 2

    for i in range(n):
        latencia = random.uniform(50, 300) + (i * 0.5)  # Tendencia creciente leve
        tiene_error = random.random() < 0.08  # 8% de error
        tokens_in = random.randint(200, 600)
        tokens_out = random.randint(50, 200)

        spans = [
            {"nombre": "embedder", "duracion_ms": latencia * 0.1, "trace_id": "t" + str(i)},
            {"nombre": "retriever", "duracion_ms": latencia * 0.2, "trace_id": "t" + str(i)},
            {"nombre": "llm_call", "duracion_ms": latencia * 0.6, "trace_id": "t" + str(i)},
            {"nombre": "postprocess", "duracion_ms": latencia * 0.1, "trace_id": "t" + str(i)},
        ]

        trazas.append({
            "trace_id": "t" + str(i),
            "timestamp": base_time + i * 2,
            "duracion_total_ms": latencia,
            "tiene_errores": tiene_error,
            "tokens_entrada": tokens_in,
            "tokens_salida": tokens_out,
            "costo_usd": (tokens_in + tokens_out) / 1000 * 0.001,
            "spans": spans,
        })

    return trazas


trazas = simular_trazas(50)
reporte = generar_reporte(trazas)

print("=== Dashboard de Observabilidad ===\n")
print("Total requests:", reporte["resumen_ejecutivo"]["total_requests"])
print("Tasa exito:", reporte["resumen_ejecutivo"]["tasa_exito"])
print("Tasa error:", reporte["resumen_ejecutivo"]["tasa_error"])
print("\nLatencias:")
for k, v in reporte["latencias"].items():
    print("  {}: {}ms".format(k, v))
print("\nTendencia:", reporte["tendencias"]["tendencia"])
print("Costo total:", reporte["resumen_ejecutivo"]["costo_total_usd"], "USD")

if reporte["alertas_activas"]:
    print("\nAlertas activas:")
    for a in reporte["alertas_activas"]:
        print("  [{}] {}".format(a["nivel"], a["mensaje"]))

print("\nRecomendaciones:")
for r in reporte["recomendaciones"]:
    print("  - " + r)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m30_dashboard.json", "w", encoding="utf-8") as f:
    json.dump(reporte, f, ensure_ascii=False, indent=2)
print("\nDashboard guardado en outputs/m30_dashboard.json")
```

---

## Checkpoint — 6 Tests de Verificación

```python
import math
import time
import json
import hashlib
from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from collections import deque
from pathlib import Path


# ── Clases reducidas para checkpoint ──────────────────────────────────────────

@dataclass
class Span:
    nombre: str
    span_id: str = ""
    trace_id: str = ""
    parent_id: str = ""
    start_time: float = 0.0
    end_time: float = 0.0
    inputs: Dict = field(default_factory=dict)
    outputs: Dict = field(default_factory=dict)
    errores: List[str] = field(default_factory=list)
    metadata: Dict = field(default_factory=dict)

    def __post_init__(self):
        if not self.span_id:
            self.span_id = hashlib.md5((self.nombre+str(time.time())).encode()).hexdigest()[:12]
        if not self.start_time:
            self.start_time = time.time()

    @property
    def duracion_ms(self):
        return round((self.end_time - self.start_time)*1000, 2) if self.end_time else 0.0

    @property
    def estado(self):
        return "ERROR" if self.errores else ("COMPLETADO" if self.end_time else "EN_PROGRESO")


class LLMTracer:
    def __init__(self, servicio="test"):
        self.servicio = servicio
        self.trazas = []
        self._stack = []
        self._trace_id = ""
        self._spans = []

    def nueva_traza(self, nombre):
        self._trace_id = hashlib.md5((nombre+str(time.time())).encode()).hexdigest()[:16]
        self._stack = []
        self._spans = []
        return self._trace_id

    @contextmanager
    def span(self, nombre, inputs=None):
        parent_id = self._stack[-1].span_id if self._stack else ""
        s = Span(nombre=nombre, trace_id=self._trace_id, parent_id=parent_id, inputs=inputs or {})
        self._stack.append(s)
        self._spans.append(s)
        try:
            yield s
        except Exception as e:
            s.errores.append(str(e))
            raise
        finally:
            s.end_time = time.time()
            self._stack.pop()

    def finalizar_traza(self):
        traza = {
            "trace_id": self._trace_id,
            "spans": [{"nombre": s.nombre, "duracion_ms": s.duracion_ms, "estado": s.estado,
                       "parent_id": s.parent_id} for s in self._spans],
            "n_spans": len(self._spans),
            "tiene_errores": any(s.errores for s in self._spans),
        }
        self.trazas.append(traza)
        return traza


def _percentil(datos, p):
    if not datos: return 0.0
    s = sorted(datos)
    idx = math.ceil(p/100*len(s)) - 1
    return round(s[max(0, idx)], 2)


class MetricsCollector:
    def __init__(self, ventana=100):
        self._latencias = deque(maxlen=ventana)
        self._errores = deque(maxlen=ventana)
        self.total = 0
    def registrar(self, latencia_ms, error=False, tokens=0, costo=0.0):
        self._latencias.append(latencia_ms)
        self._errores.append(1 if error else 0)
        self.total += 1
    def estadisticas(self):
        datos = list(self._latencias)
        errores = list(self._errores)
        return {
            "p50_ms": _percentil(datos, 50),
            "p95_ms": _percentil(datos, 95),
            "p99_ms": _percentil(datos, 99),
            "tasa_error": round(sum(errores)/len(errores), 4) if errores else 0.0,
            "n": len(datos),
        }


@dataclass
class ReglaAlerta:
    nombre: str
    metrica: str
    umbral: float
    operador: str
    severidad: str

class AlertManager:
    def __init__(self):
        self.alertas = []
    def evaluar(self, metricas, reglas):
        activas = []
        for r in reglas:
            val = metricas.get(r.metrica, 0)
            dispara = (r.operador == ">" and val > r.umbral) or \
                      (r.operador == "<" and val < r.umbral)
            if dispara:
                activas.append({"nombre": r.nombre, "severidad": r.severidad, "valor": val})
        self.alertas.extend(activas)
        return activas

def generar_reporte(trazas):
    if not trazas: return {"error": "sin datos"}
    lats = [t.get("duracion_total_ms", 0) for t in trazas]
    errores = sum(1 for t in trazas if t.get("tiene_errores", False))
    return {
        "total": len(trazas),
        "tasa_error": round(errores/len(trazas), 4),
        "p50_ms": _percentil(lats, 50),
        "p95_ms": _percentil(lats, 95),
        "p99_ms": _percentil(lats, 99),
    }


# ── TESTS ──────────────────────────────────────────────────────────────────────
tests_resultados = []

def run_test(nombre, fn):
    try:
        fn()
        tests_resultados.append((nombre, "PASS", None))
        print("[PASS] " + nombre)
    except AssertionError as e:
        tests_resultados.append((nombre, "FAIL", str(e)))
        print("[FAIL] " + nombre + " -> " + str(e))
    except Exception as e:
        tests_resultados.append((nombre, "ERROR", str(e)))
        print("[ERROR] " + nombre + " -> " + str(e))


def test_1_span_duracion():
    """Span debe capturar duracion correctamente."""
    s = Span(nombre="test-span")
    time.sleep(0.05)
    s.end_time = time.time()
    assert s.duracion_ms >= 40, "Duracion debe ser >= 40ms, got: " + str(s.duracion_ms)
    assert s.estado == "COMPLETADO", "Estado debe ser COMPLETADO, got: " + s.estado


def test_2_tracer_context_manager():
    """Context manager debe abrir y cerrar span correctamente."""
    tracer = LLMTracer(servicio="test")
    tracer.nueva_traza("traza-test")

    with tracer.span("operacion-principal", inputs={"x": 1}) as s:
        time.sleep(0.02)
        with tracer.span("sub-operacion") as s2:
            time.sleep(0.01)
            s2.outputs = {"resultado": 42}

    traza = tracer.finalizar_traza()
    assert traza["n_spans"] == 2, "Deben ser 2 spans, got: " + str(traza["n_spans"])
    assert not traza["tiene_errores"], "No debe haber errores"

    span_raiz = [sp for sp in traza["spans"] if not sp["parent_id"]]
    assert len(span_raiz) == 1, "Debe haber exactamente 1 span raiz"
    assert span_raiz[0]["duracion_ms"] >= 25, "Raiz debe durar >= 25ms"


def test_3_metricas_percentiles():
    """MetricsCollector debe calcular percentiles correctamente."""
    mc = MetricsCollector(ventana=100)
    # 100 observaciones: 90 de latencia 100ms, 9 de 500ms, 1 de 5000ms
    for _ in range(90):
        mc.registrar(100.0)
    for _ in range(9):
        mc.registrar(500.0)
    mc.registrar(5000.0)

    stats = mc.estadisticas()
    assert stats["p50_ms"] == 100.0, "P50 debe ser 100ms, got: " + str(stats["p50_ms"])
    assert stats["p95_ms"] >= 400.0, "P95 debe ser >= 400ms, got: " + str(stats["p95_ms"])
    assert stats["p99_ms"] >= 500.0, "P99 debe ser >= 500ms, got: " + str(stats["p99_ms"])


def test_4_metricas_tasa_error():
    """Tasa de error debe calcularse correctamente."""
    mc = MetricsCollector(ventana=100)
    for _ in range(80):
        mc.registrar(100.0, error=False)
    for _ in range(20):
        mc.registrar(100.0, error=True)

    stats = mc.estadisticas()
    assert abs(stats["tasa_error"] - 0.2) < 0.01, "Tasa error debe ser ~0.2, got: " + str(stats["tasa_error"])


def test_5_alertas_configurables():
    """AlertManager debe disparar alertas cuando se superan umbrales."""
    am = AlertManager()
    reglas = [
        ReglaAlerta("latencia_alta", "p95_ms", 500.0, ">", "WARNING"),
        ReglaAlerta("error_critico", "tasa_error", 0.1, ">", "CRITICAL"),
    ]

    metricas_normales = {"p95_ms": 200.0, "tasa_error": 0.02}
    alertas_normales = am.evaluar(metricas_normales, reglas)
    assert len(alertas_normales) == 0, "No debe haber alertas en condiciones normales"

    metricas_altas = {"p95_ms": 700.0, "tasa_error": 0.15}
    alertas_altas = am.evaluar(metricas_altas, reglas)
    assert len(alertas_altas) == 2, "Deben dispararse 2 alertas, got: " + str(len(alertas_altas))

    criticas = [a for a in alertas_altas if a["severidad"] == "CRITICAL"]
    assert len(criticas) == 1, "Debe haber 1 alerta CRITICAL"


def test_6_reporte_dashboard():
    """generar_reporte debe producir JSON con metricas correctas."""
    trazas_test = [
        {"trace_id": "t1", "duracion_total_ms": 100.0, "tiene_errores": False, "spans": []},
        {"trace_id": "t2", "duracion_total_ms": 200.0, "tiene_errores": False, "spans": []},
        {"trace_id": "t3", "duracion_total_ms": 5000.0, "tiene_errores": True, "spans": []},
        {"trace_id": "t4", "duracion_total_ms": 150.0, "tiene_errores": False, "spans": []},
    ]
    reporte = generar_reporte(trazas_test)

    assert reporte["total"] == 4, "Total debe ser 4, got: " + str(reporte["total"])
    assert abs(reporte["tasa_error"] - 0.25) < 0.01, "Tasa error debe ser 0.25, got: " + str(reporte["tasa_error"])
    assert reporte["p99_ms"] >= 1000.0, "P99 debe ser >= 1000ms por la traza lenta"
    assert reporte["p50_ms"] < 300.0, "P50 debe ser < 300ms"

    # Verificar que se puede serializar a JSON
    json_str = json.dumps(reporte)
    assert len(json_str) > 10, "JSON no debe estar vacio"


# ── Ejecutar todos los tests ───────────────────────────────────────────────────
print("\n" + "="*60)
print("CHECKPOINT M30 — Observabilidad")
print("="*60)

run_test("T1: Span captura duracion correctamente", test_1_span_duracion)
run_test("T2: Tracer context manager jerarquia spans", test_2_tracer_context_manager)
run_test("T3: Percentiles p50/p95/p99 correctos", test_3_metricas_percentiles)
run_test("T4: Tasa de error correcta en ventana", test_4_metricas_tasa_error)
run_test("T5: Alertas configurables con umbrales", test_5_alertas_configurables)
run_test("T6: Reporte dashboard genera JSON valido", test_6_reporte_dashboard)

total = len(tests_resultados)
pasados = sum(1 for _, s, _ in tests_resultados if s == "PASS")
print("\n" + "="*60)
print("Resultado: {}/{} tests pasados".format(pasados, total))
if pasados == total:
    print("TODOS LOS TESTS PASARON - Modulo 30 completado")
else:
    for nombre, estado, msg in tests_resultados:
        if estado != "PASS":
            print("  FALLO: " + nombre + " -> " + str(msg))
print("="*60)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m30_checkpoint.json", "w", encoding="utf-8") as f:
    json.dump({
        "modulo": "M30",
        "total": total,
        "pasados": pasados,
        "tests": [{"nombre": n, "estado": s, "mensaje": m} for n, s, m in tests_resultados],
    }, f, ensure_ascii=False, indent=2)
```
