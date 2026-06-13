# Módulo 31 — Deployment: FastAPI, vLLM, llama.cpp, Ollama

## Encabezado

- **Objetivo**: Simular el deployment de LLMs en producción incluyendo API REST (estilo FastAPI), configuración de servidores de inferencia, load balancing con round-robin, y health checks.
- **Herramientas**: Python stdlib (json, time, math, hashlib, dataclasses, typing, collections, enum, pathlib, random, re)
- **Prerequisito**: Módulos 28–30 (safety, evaluación, observabilidad)

---

## Sección 1: API REST para LLMs con FastAPI (simulado)

En producción, los LLMs se exponen como servicios HTTP compatibles con la especificación OpenAI. Esta sección simula el comportamiento de un servidor FastAPI sin dependencias externas.

```python
import json
import time
import hashlib
import re
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Any
from pathlib import Path
from collections import defaultdict


# ── Schemas de request/response ────────────────────────────────────────────────
@dataclass
class ChatMessage:
    role: str       # "system" | "user" | "assistant"
    content: str

    def to_dict(self):
        return {"role": self.role, "content": self.content}

    @classmethod
    def from_dict(cls, d: dict) -> "ChatMessage":
        return cls(role=d["role"], content=d["content"])


@dataclass
class ChatCompletionRequest:
    model: str
    messages: List[ChatMessage]
    max_tokens: int = 512
    temperature: float = 0.7
    stream: bool = False
    top_p: float = 1.0
    stop: Optional[List[str]] = None

    def validar(self) -> List[str]:
        """Retorna lista de errores de validacion."""
        errores = []
        if not self.model:
            errores.append("'model' es requerido")
        if not self.messages:
            errores.append("'messages' no puede estar vacio")
        for i, msg in enumerate(self.messages):
            if msg.role not in ("system", "user", "assistant"):
                errores.append("Rol invalido en messages[{}]: '{}'".format(i, msg.role))
        if not 0 <= self.temperature <= 2.0:
            errores.append("'temperature' debe estar en [0.0, 2.0]")
        if self.max_tokens < 1 or self.max_tokens > 32768:
            errores.append("'max_tokens' debe estar en [1, 32768]")
        return errores

    @classmethod
    def from_dict(cls, d: dict) -> "ChatCompletionRequest":
        messages = [ChatMessage.from_dict(m) for m in d.get("messages", [])]
        return cls(
            model=d.get("model", ""),
            messages=messages,
            max_tokens=d.get("max_tokens", 512),
            temperature=d.get("temperature", 0.7),
            stream=d.get("stream", False),
            top_p=d.get("top_p", 1.0),
            stop=d.get("stop"),
        )


@dataclass
class ChatCompletionResponse:
    id: str
    model: str
    choices: List[Dict]
    usage: Dict
    created: int = 0

    def __post_init__(self):
        if not self.created:
            self.created = int(time.time())

    def to_dict(self):
        return {
            "id": self.id,
            "object": "chat.completion",
            "created": self.created,
            "model": self.model,
            "choices": self.choices,
            "usage": self.usage,
        }


# ── LLM Simulado ───────────────────────────────────────────────────────────────
class LLMSimulado:
    """Motor de inferencia simulado deterministico."""

    def __init__(self, model_id: str):
        self.model_id = model_id

    def inferir(self, messages: List[ChatMessage], max_tokens: int = 256) -> Dict:
        """Retorna respuesta simulada con conteo de tokens."""
        prompt_text = " ".join(m.content for m in messages)
        h = int(hashlib.md5((prompt_text + self.model_id).encode()).hexdigest(), 16)

        respuestas = [
            "Basandome en la informacion disponible, puedo explicar que " + prompt_text[:30] + "...",
            "Esta es una respuesta generada por " + self.model_id + " para su consulta.",
            "El concepto solicitado se puede entender de la siguiente manera: " + prompt_text[:20],
            "Segun mi entrenamiento, la respuesta a su pregunta es: " + str(h % 10000),
            "Le proporciono una respuesta detallada sobre el tema consultado.",
        ]
        texto_respuesta = respuestas[h % len(respuestas)]

        # Estimar tokens (aprox 1 token = 4 chars)
        prompt_tokens = max(1, len(prompt_text) // 4)
        completion_tokens = max(1, len(texto_respuesta) // 4)

        return {
            "texto": texto_respuesta,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "finish_reason": "stop",
        }


# ── Rate Limiter Simulado ──────────────────────────────────────────────────────
class RateLimiter:
    """
    Rate limiting de ventana deslizante.
    Limita requests por cliente por minuto.
    """

    def __init__(self, max_rpm: int = 60, max_tpm: int = 100000):
        self.max_rpm = max_rpm          # Requests por minuto
        self.max_tpm = max_tpm          # Tokens por minuto
        self._ventana_rpm: Dict[str, List[float]] = defaultdict(list)
        self._tokens_usados: Dict[str, List[tuple]] = defaultdict(list)

    def verificar(self, cliente_id: str, tokens_estimados: int = 100) -> Dict:
        """
        Verifica si el cliente puede hacer un request.
        Retorna: {permitido, rpm_actual, razon}
        """
        ahora = time.time()
        ventana_inicio = ahora - 60.0

        # Limpiar requests antiguos
        self._ventana_rpm[cliente_id] = [
            t for t in self._ventana_rpm[cliente_id] if t > ventana_inicio
        ]
        self._tokens_usados[cliente_id] = [
            (t, n) for t, n in self._tokens_usados[cliente_id] if t > ventana_inicio
        ]

        rpm_actual = len(self._ventana_rpm[cliente_id])
        tpm_actual = sum(n for _, n in self._tokens_usados[cliente_id])

        if rpm_actual >= self.max_rpm:
            return {
                "permitido": False,
                "razon": "Rate limit excedido: {} RPM (max: {})".format(rpm_actual, self.max_rpm),
                "rpm_actual": rpm_actual,
                "retry_after": 60,
            }

        if tpm_actual + tokens_estimados > self.max_tpm:
            return {
                "permitido": False,
                "razon": "Token limit excedido: {} TPM (max: {})".format(tpm_actual, self.max_tpm),
                "tpm_actual": tpm_actual,
                "retry_after": 60,
            }

        # Registrar request
        self._ventana_rpm[cliente_id].append(ahora)
        self._tokens_usados[cliente_id].append((ahora, tokens_estimados))

        return {"permitido": True, "rpm_actual": rpm_actual + 1, "tpm_actual": tpm_actual + tokens_estimados}


# ── LLM API Server ─────────────────────────────────────────────────────────────
class LLMAPIServer:
    """
    Servidor API REST simulado compatible con OpenAI API.
    Endpoints: POST /v1/chat/completions, GET /v1/models, GET /health
    """

    MODELOS_DISPONIBLES = [
        {"id": "llm-7b-instruct", "object": "model", "owned_by": "local"},
        {"id": "llm-13b-chat", "object": "model", "owned_by": "local"},
        {"id": "llm-70b-preview", "object": "model", "owned_by": "local"},
    ]

    def __init__(self, host: str = "0.0.0.0", puerto: int = 8000):
        self.host = host
        self.puerto = puerto
        self.rate_limiter = RateLimiter(max_rpm=60)
        self._motores: Dict[str, LLMSimulado] = {
            m["id"]: LLMSimulado(m["id"]) for m in self.MODELOS_DISPONIBLES
        }
        self._request_log: List[Dict] = []
        self._start_time = time.time()

    def _generar_id(self, prefijo: str = "chatcmpl") -> str:
        return prefijo + "-" + hashlib.md5(str(time.time()).encode()).hexdigest()[:12]

    def get_health(self) -> Dict:
        """GET /health - Estado del servidor."""
        uptime = time.time() - self._start_time
        return {
            "status": "ok",
            "uptime_segundos": round(uptime, 1),
            "modelos_cargados": len(self._motores),
            "total_requests": len(self._request_log),
        }

    def get_models(self) -> Dict:
        """GET /v1/models - Lista de modelos disponibles."""
        return {
            "object": "list",
            "data": self.MODELOS_DISPONIBLES,
        }

    def post_chat_completions(self, request_dict: Dict, cliente_id: str = "anon") -> Dict:
        """POST /v1/chat/completions - Inferencia del modelo."""
        inicio = time.time()

        # Parsear y validar
        try:
            req = ChatCompletionRequest.from_dict(request_dict)
        except Exception as e:
            return {"error": {"code": 400, "message": "Request invalido: " + str(e)}}

        errores = req.validar()
        if errores:
            return {"error": {"code": 422, "message": "Validacion fallida", "detalles": errores}}

        # Rate limiting
        tokens_est = sum(len(m.content) // 4 for m in req.messages) + req.max_tokens
        rl = self.rate_limiter.verificar(cliente_id, tokens_est)
        if not rl["permitido"]:
            return {"error": {"code": 429, "message": rl["razon"], "retry_after": rl.get("retry_after", 60)}}

        # Verificar modelo
        if req.model not in self._motores:
            return {"error": {"code": 404, "message": "Modelo no encontrado: " + req.model}}

        # Inferencia
        motor = self._motores[req.model]
        resultado = motor.inferir(req.messages, req.max_tokens)

        # Simular latencia de inferencia
        time.sleep(0.01)
        latencia = round((time.time() - inicio) * 1000, 2)

        response = ChatCompletionResponse(
            id=self._generar_id(),
            model=req.model,
            choices=[{
                "index": 0,
                "message": {"role": "assistant", "content": resultado["texto"]},
                "finish_reason": resultado["finish_reason"],
            }],
            usage={
                "prompt_tokens": resultado["prompt_tokens"],
                "completion_tokens": resultado["completion_tokens"],
                "total_tokens": resultado["prompt_tokens"] + resultado["completion_tokens"],
            },
        )

        log_entry = {
            "timestamp": inicio,
            "cliente": cliente_id,
            "modelo": req.model,
            "latencia_ms": latencia,
            "tokens": response.usage["total_tokens"],
        }
        self._request_log.append(log_entry)

        return response.to_dict()


# ── Demo Sección 1 ─────────────────────────────────────────────────────────────
servidor = LLMAPIServer()

print("=== LLM API Server ===\n")
print("Health:", servidor.get_health())
print("\nModelos disponibles:")
for m in servidor.get_models()["data"]:
    print("  -", m["id"])

print("\n--- POST /v1/chat/completions ---")
request_valido = {
    "model": "llm-7b-instruct",
    "messages": [
        {"role": "system", "content": "Eres un asistente util."},
        {"role": "user", "content": "Que es el machine learning?"},
    ],
    "max_tokens": 256,
    "temperature": 0.7,
}
resp = servidor.post_chat_completions(request_valido, cliente_id="usuario-demo")
print("ID:", resp.get("id"))
print("Modelo:", resp.get("model"))
print("Tokens:", resp.get("usage"))
print("Respuesta:", resp["choices"][0]["message"]["content"][:80] if "choices" in resp else resp.get("error"))

print("\n--- Request invalido (modelo inexistente) ---")
req_malo = {"model": "modelo-inexistente", "messages": [{"role": "user", "content": "hola"}]}
resp_malo = servidor.post_chat_completions(req_malo)
print("Error:", resp_malo.get("error"))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m31_api_demo.json", "w", encoding="utf-8") as f:
    json.dump({"request": request_valido, "response": resp}, f, ensure_ascii=False, indent=2)
```

---

## Sección 2: Configuración de Servidor de Inferencia

La configuración correcta del servidor de inferencia es crítica para balancear rendimiento, calidad y recursos disponibles.

```python
import json
import math
from dataclasses import dataclass, field
from typing import Dict, Optional, List
from pathlib import Path


@dataclass
class InferenceConfig:
    """
    Configuracion completa de un servidor de inferencia LLM.
    Compatible con parametros de vLLM, llama.cpp y Ollama.
    """
    # Modelo
    model_path: str = "modelos/llm-7b-q4.gguf"
    model_id: str = "llm-7b-instruct"

    # Generacion
    max_tokens: int = 2048
    temperature: float = 0.7
    top_p: float = 0.95
    top_k: int = 40
    repetition_penalty: float = 1.1

    # Contexto
    context_window: int = 4096      # Tamano de contexto en tokens

    # Hardware (GPU)
    num_gpu_layers: int = 35        # Capas en GPU (0 = CPU only)
    gpu_memory_utilization: float = 0.9  # Fraccion de VRAM a usar
    tensor_parallel_size: int = 1   # Numero de GPUs para tensor parallelism

    # Cuantizacion
    quantization: str = "q4_0"      # "fp16", "q8_0", "q4_0", "q4_k_m"
    dtype: str = "float16"

    # Servidor
    host: str = "0.0.0.0"
    port: int = 8000
    max_concurrent_requests: int = 8
    request_timeout: int = 120

    # Batching
    max_batch_size: int = 32
    continuous_batching: bool = True

    def to_dict(self) -> Dict:
        return {
            "model": {"path": self.model_path, "id": self.model_id},
            "generacion": {
                "max_tokens": self.max_tokens, "temperature": self.temperature,
                "top_p": self.top_p, "top_k": self.top_k,
                "repetition_penalty": self.repetition_penalty,
            },
            "contexto": {"context_window": self.context_window},
            "hardware": {
                "num_gpu_layers": self.num_gpu_layers,
                "gpu_memory_utilization": self.gpu_memory_utilization,
                "tensor_parallel_size": self.tensor_parallel_size,
            },
            "cuantizacion": {"quantization": self.quantization, "dtype": self.dtype},
            "servidor": {
                "host": self.host, "port": self.port,
                "max_concurrent": self.max_concurrent_requests,
            },
        }


# ── Estimacion de recursos ─────────────────────────────────────────────────────
BITS_POR_CUANTIZACION = {
    "fp32": 32, "float32": 32,
    "fp16": 16, "float16": 16, "bfloat16": 16,
    "q8_0": 8,
    "q6_k": 6,
    "q5_0": 5, "q5_k_m": 5,
    "q4_0": 4, "q4_k_m": 4, "q4_1": 4,
    "q3_k_m": 3,
    "q2_k": 2,
}

PARAMETROS_POR_TAMANIO = {
    "7b": 7_000_000_000,
    "13b": 13_000_000_000,
    "34b": 34_000_000_000,
    "70b": 70_000_000_000,
    "mixtral": 46_700_000_000,  # Mixtral 8x7B activos: ~13B, total: ~47B
}


def calcular_requisitos(config: InferenceConfig) -> Dict:
    """
    Estima VRAM y RAM necesaria para servir un modelo.
    Formula: parametros * bits / 8 / 1e9 GB para pesos
    + overhead de KV cache = 2 * capas * heads * dim * context * batch * bytes
    """
    # Extraer tamanio del modelo desde el path o model_id
    nombre = config.model_path.lower() + config.model_id.lower()
    n_params = 7_000_000_000  # default 7B
    for key, val in PARAMETROS_POR_TAMANIO.items():
        if key in nombre:
            n_params = val
            break

    bits = BITS_POR_CUANTIZACION.get(config.quantization, 16)

    # VRAM para pesos del modelo
    vram_pesos_gb = round(n_params * bits / 8 / 1e9, 2)

    # Overhead KV cache (estimacion simplificada)
    # KV cache ~ 2 * n_layers * context_window * hidden_dim * bytes_per_element
    n_layers = max(1, int(math.log2(n_params / 1e6) * 3))  # aprox
    hidden_dim = 4096 if n_params < 15e9 else 8192
    bytes_kv = 2 if config.dtype in ("float16", "bfloat16") else 4
    kv_cache_gb = round(
        2 * n_layers * config.context_window * hidden_dim * bytes_kv * config.max_batch_size / 1e9,
        2
    )

    # Overhead del framework
    overhead_gb = round(vram_pesos_gb * 0.1, 2)

    vram_total_gb = round(vram_pesos_gb + kv_cache_gb + overhead_gb, 2)
    ram_total_gb = round(vram_pesos_gb * 1.5, 2)  # RAM = 1.5x VRAM para swapping

    # Recomendacion de GPU
    if vram_total_gb <= 8:
        gpu_recomendada = "RTX 3080 (10GB) o similar"
    elif vram_total_gb <= 16:
        gpu_recomendada = "RTX 4090 (24GB) o A10 (24GB)"
    elif vram_total_gb <= 40:
        gpu_recomendada = "A100 40GB o A40"
    elif vram_total_gb <= 80:
        gpu_recomendada = "A100 80GB o H100"
    else:
        gpu_recomendada = "Multi-GPU: 2+ A100 80GB"

    return {
        "modelo": {
            "parametros_b": round(n_params / 1e9, 1),
            "bits_cuantizacion": bits,
            "cuantizacion": config.quantization,
        },
        "vram": {
            "pesos_gb": vram_pesos_gb,
            "kv_cache_gb": kv_cache_gb,
            "overhead_gb": overhead_gb,
            "total_gb": vram_total_gb,
        },
        "ram_recomendada_gb": ram_total_gb,
        "gpu_recomendada": gpu_recomendada,
        "puede_ejecutar_cpu": vram_total_gb <= ram_total_gb,
        "context_window": config.context_window,
        "batch_size": config.max_batch_size,
    }


# ── Presets de configuracion ───────────────────────────────────────────────────
def config_desarrollo() -> InferenceConfig:
    return InferenceConfig(
        model_path="modelos/llm-7b-q4.gguf",
        model_id="llm-7b-instruct",
        context_window=2048,
        num_gpu_layers=0,       # CPU only para dev
        quantization="q4_0",
        max_concurrent_requests=2,
        max_batch_size=1,
    )

def config_produccion_gpu() -> InferenceConfig:
    return InferenceConfig(
        model_path="modelos/llm-13b-fp16.bin",
        model_id="llm-13b-chat",
        context_window=8192,
        num_gpu_layers=40,
        gpu_memory_utilization=0.85,
        quantization="fp16",
        max_concurrent_requests=32,
        max_batch_size=16,
        continuous_batching=True,
    )


# ── Demo Sección 2 ─────────────────────────────────────────────────────────────
print("=== Configuracion de Servidor de Inferencia ===\n")

configs = [
    ("Desarrollo CPU", config_desarrollo()),
    ("Produccion GPU", config_produccion_gpu()),
]

for nombre, config in configs:
    requisitos = calcular_requisitos(config)
    print("Config: " + nombre)
    print("  Modelo:", requisitos["modelo"])
    print("  VRAM total:", requisitos["vram"]["total_gb"], "GB")
    print("  RAM recomendada:", requisitos["ram_recomendada_gb"], "GB")
    print("  GPU recomendada:", requisitos["gpu_recomendada"])
    print()

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m31_inference_config.json", "w", encoding="utf-8") as f:
    json.dump({
        "configs": [
            {"nombre": n, "requisitos": calcular_requisitos(c), "config": c.to_dict()}
            for n, c in configs
        ]
    }, f, ensure_ascii=False, indent=2)
print("Guardado en outputs/m31_inference_config.json")
```

---

## Sección 3: Load Balancing y Escalado

El load balancing distribuye requests entre múltiples instancias para maximizar throughput y disponibilidad.

```python
import time
import json
import hashlib
import random
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from collections import defaultdict, deque
from pathlib import Path


@dataclass
class Instancia:
    """Representa una instancia del servidor LLM."""
    id: str
    host: str
    puerto: int
    modelo: str
    activa: bool = True
    requests_en_curso: int = 0
    max_concurrencia: int = 8
    total_requests: int = 0
    total_tokens: int = 0
    errores: int = 0
    latencias: List[float] = field(default_factory=list)

    @property
    def disponible(self) -> bool:
        return self.activa and self.requests_en_curso < self.max_concurrencia

    @property
    def latencia_promedio(self) -> float:
        if not self.latencias:
            return 0.0
        return round(sum(self.latencias[-20:]) / len(self.latencias[-20:]), 2)

    @property
    def tasa_error(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return round(self.errores / self.total_requests, 4)

    def info(self) -> Dict:
        return {
            "id": self.id,
            "host": self.host + ":" + str(self.puerto),
            "modelo": self.modelo,
            "activa": self.activa,
            "disponible": self.disponible,
            "requests_en_curso": self.requests_en_curso,
            "total_requests": self.total_requests,
            "latencia_promedio_ms": self.latencia_promedio,
            "tasa_error": self.tasa_error,
        }


class LoadBalancer:
    """
    Load balancer con estrategias: round-robin, least-connections, weighted.
    Simula distribucion de requests entre N instancias.
    """

    ESTRATEGIAS = ("round_robin", "least_connections", "weighted_round_robin")

    def __init__(self, estrategia: str = "round_robin"):
        assert estrategia in self.ESTRATEGIAS, "Estrategia invalida: " + estrategia
        self.estrategia = estrategia
        self.instancias: List[Instancia] = []
        self._rr_index: int = 0
        self._request_queue: deque = deque()
        self.metricas_globales = {
            "total_requests": 0,
            "requests_exitosos": 0,
            "requests_fallidos": 0,
            "total_tokens": 0,
        }

    def registrar_instancia(self, instancia: Instancia):
        self.instancias.append(instancia)

    def _instancias_disponibles(self) -> List[Instancia]:
        return [i for i in self.instancias if i.disponible]

    def seleccionar_instancia(self) -> Optional[Instancia]:
        """Selecciona instancia segun la estrategia configurada."""
        disponibles = self._instancias_disponibles()
        if not disponibles:
            return None

        if self.estrategia == "round_robin":
            # Cyclo simple entre instancias disponibles
            idx = self._rr_index % len(disponibles)
            self._rr_index += 1
            return disponibles[idx]

        elif self.estrategia == "least_connections":
            # Instancia con menos requests en curso
            return min(disponibles, key=lambda i: i.requests_en_curso)

        elif self.estrategia == "weighted_round_robin":
            # Ponderado por capacidad (max_concurrencia)
            pesos = [i.max_concurrencia for i in disponibles]
            total_peso = sum(pesos)
            r = random.random() * total_peso
            acumulado = 0
            for inst, peso in zip(disponibles, pesos):
                acumulado += peso
                if r <= acumulado:
                    return inst
            return disponibles[-1]

        return disponibles[0]

    def procesar_request(self, request: Dict) -> Dict:
        """Simula el procesamiento de un request en una instancia."""
        instancia = self.seleccionar_instancia()
        self.metricas_globales["total_requests"] += 1

        if instancia is None:
            self.metricas_globales["requests_fallidos"] += 1
            return {
                "exito": False,
                "error": "Sin instancias disponibles - todos los workers ocupados",
                "instancia": None,
            }

        # Simular procesamiento
        instancia.requests_en_curso += 1
        inicio = time.time()
        latencia = 50 + random.uniform(0, 200)  # 50-250ms
        time.sleep(latencia / 1000 * 0.01)  # Comprimido para el demo

        es_error = random.random() < 0.03  # 3% error rate
        tokens = random.randint(100, 500)

        instancia.requests_en_curso -= 1
        instancia.total_requests += 1
        instancia.latencias.append(latencia)
        if es_error:
            instancia.errores += 1
            self.metricas_globales["requests_fallidos"] += 1
        else:
            instancia.total_tokens += tokens
            self.metricas_globales["requests_exitosos"] += 1
            self.metricas_globales["total_tokens"] += tokens

        return {
            "exito": not es_error,
            "instancia_id": instancia.id,
            "latencia_ms": round(latencia, 2),
            "tokens": tokens if not es_error else 0,
            "error": "Timeout" if es_error else None,
        }

    def throughput_tokens_por_seg(self) -> float:
        """Estima throughput total en tokens/segundo."""
        latencias_todas = []
        for inst in self.instancias:
            latencias_todas.extend(inst.latencias[-10:])
        if not latencias_todas:
            return 0.0
        latencia_prom_ms = sum(latencias_todas) / len(latencias_todas)
        tps = (self.metricas_globales["total_tokens"] / latencia_prom_ms * 1000
               if latencia_prom_ms > 0 else 0)
        return round(tps, 2)

    def estado(self) -> Dict:
        return {
            "estrategia": self.estrategia,
            "instancias_totales": len(self.instancias),
            "instancias_activas": sum(1 for i in self.instancias if i.activa),
            "instancias_disponibles": len(self._instancias_disponibles()),
            "metricas_globales": self.metricas_globales,
            "throughput_tps": self.throughput_tokens_por_seg(),
            "instancias": [i.info() for i in self.instancias],
        }


# ── Demo Sección 3 ─────────────────────────────────────────────────────────────
print("=== Load Balancer ===\n")

lb = LoadBalancer(estrategia="round_robin")
random.seed(42)

# Registrar 3 instancias
for i in range(3):
    lb.registrar_instancia(Instancia(
        id="worker-" + str(i),
        host="10.0.0." + str(i+1),
        puerto=8000 + i,
        modelo="llm-7b-instruct",
        max_concurrencia=8,
    ))

# Simular 30 requests
print("Procesando 30 requests...")
for req_num in range(30):
    resultado = lb.procesar_request({"query": "pregunta_" + str(req_num)})

estado = lb.estado()
print("\nEstado del balanceador:")
print("  Estrategia:", estado["estrategia"])
print("  Total requests:", estado["metricas_globales"]["total_requests"])
print("  Exitosos:", estado["metricas_globales"]["requests_exitosos"])
print("  Fallidos:", estado["metricas_globales"]["requests_fallidos"])
print("\nDistribucion por worker:")
for inst in estado["instancias"]:
    print("  {}: {} requests, {:.1f}ms prom".format(
        inst["id"], inst["total_requests"], inst["latencia_promedio_ms"]
    ))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m31_load_balancer.json", "w", encoding="utf-8") as f:
    json.dump(estado, f, ensure_ascii=False, indent=2)
print("\nGuardado en outputs/m31_load_balancer.json")
```

---

## Sección 4: Health Checks y Monitoreo

Los health checks permiten detectar automáticamente instancias degradadas y excluirlas del pool de servidores.

```python
import time
import json
import random
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Callable
from pathlib import Path


@dataclass
class ComponenteHealth:
    nombre: str
    estado: str      # "OK" | "WARNING" | "CRITICAL" | "UNKNOWN"
    mensaje: str
    valor: Optional[float] = None
    umbral_warning: Optional[float] = None
    umbral_critical: Optional[float] = None


class HealthChecker:
    """
    Sistema de health checks para servidor LLM.
    Verifica: modelo cargado, memoria disponible, latencia promedio.
    """

    def __init__(self, config: Dict = None):
        self.config = config or {
            "latencia_warning_ms": 500,
            "latencia_critical_ms": 2000,
            "memoria_warning_gb": 2.0,
            "memoria_critical_gb": 0.5,
            "error_rate_warning": 0.05,
            "error_rate_critical": 0.15,
        }
        self._historial: List[Dict] = []

    def check_modelo_cargado(self, modelo_id: str, modelos_activos: List[str]) -> ComponenteHealth:
        """Verifica si el modelo esta cargado y listo."""
        cargado = modelo_id in modelos_activos
        return ComponenteHealth(
            nombre="modelo_cargado",
            estado="OK" if cargado else "CRITICAL",
            mensaje="Modelo '{}' {} en memoria".format(
                modelo_id, "disponible" if cargado else "NO disponible"
            ),
        )

    def check_memoria(self, memoria_libre_gb: float) -> ComponenteHealth:
        """Verifica memoria VRAM/RAM disponible."""
        if memoria_libre_gb >= self.config["memoria_warning_gb"]:
            estado = "OK"
        elif memoria_libre_gb >= self.config["memoria_critical_gb"]:
            estado = "WARNING"
        else:
            estado = "CRITICAL"

        return ComponenteHealth(
            nombre="memoria_disponible",
            estado=estado,
            mensaje="{:.1f} GB libres".format(memoria_libre_gb),
            valor=memoria_libre_gb,
            umbral_warning=self.config["memoria_warning_gb"],
            umbral_critical=self.config["memoria_critical_gb"],
        )

    def check_latencia(self, latencias_ms: List[float]) -> ComponenteHealth:
        """Verifica latencia promedio de inferencia."""
        if not latencias_ms:
            return ComponenteHealth(nombre="latencia", estado="UNKNOWN", mensaje="Sin datos")

        p95 = sorted(latencias_ms)[int(0.95 * len(latencias_ms))]

        if p95 < self.config["latencia_warning_ms"]:
            estado = "OK"
        elif p95 < self.config["latencia_critical_ms"]:
            estado = "WARNING"
        else:
            estado = "CRITICAL"

        return ComponenteHealth(
            nombre="latencia_p95",
            estado=estado,
            mensaje="P95: {:.0f}ms".format(p95),
            valor=p95,
            umbral_warning=self.config["latencia_warning_ms"],
            umbral_critical=self.config["latencia_critical_ms"],
        )

    def check_tasa_error(self, total: int, errores: int) -> ComponenteHealth:
        """Verifica tasa de error reciente."""
        if total == 0:
            return ComponenteHealth(nombre="tasa_error", estado="UNKNOWN", mensaje="Sin requests")

        tasa = errores / total

        if tasa < self.config["error_rate_warning"]:
            estado = "OK"
        elif tasa < self.config["error_rate_critical"]:
            estado = "WARNING"
        else:
            estado = "CRITICAL"

        return ComponenteHealth(
            nombre="tasa_error",
            estado=estado,
            mensaje="{:.1%} ({}/{})".format(tasa, errores, total),
            valor=tasa,
        )

    def status_check(
        self,
        modelo_id: str,
        modelos_activos: List[str],
        memoria_libre_gb: float,
        latencias_ms: List[float],
        total_requests: int,
        errores: int,
    ) -> Dict:
        """
        Ejecuta todos los health checks y retorna estado consolidado.
        """
        checks = [
            self.check_modelo_cargado(modelo_id, modelos_activos),
            self.check_memoria(memoria_libre_gb),
            self.check_latencia(latencias_ms),
            self.check_tasa_error(total_requests, errores),
        ]

        # Estado global: el peor de los checks
        prioridad = {"CRITICAL": 3, "WARNING": 2, "OK": 1, "UNKNOWN": 0}
        estado_global = max(checks, key=lambda c: prioridad[c.estado]).estado

        resultado = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "estado_global": estado_global,
            "estado_texto": {
                "OK": "Servicio operando correctamente",
                "WARNING": "Servicio operando con advertencias",
                "CRITICAL": "Servicio degradado - requiere atencion",
                "UNKNOWN": "Estado desconocido",
            }.get(estado_global, ""),
            "checks": [
                {
                    "nombre": c.nombre,
                    "estado": c.estado,
                    "mensaje": c.mensaje,
                    "valor": c.valor,
                }
                for c in checks
            ],
            "resumen": {
                "total_checks": len(checks),
                "ok": sum(1 for c in checks if c.estado == "OK"),
                "warnings": sum(1 for c in checks if c.estado == "WARNING"),
                "critical": sum(1 for c in checks if c.estado == "CRITICAL"),
            },
        }

        self._historial.append(resultado)
        return resultado

    def historial_reciente(self, n: int = 10) -> List[Dict]:
        return self._historial[-n:]


# ── Demo Sección 4 ─────────────────────────────────────────────────────────────
print("=== Health Checker ===\n")
random.seed(99)

checker = HealthChecker()

# Escenario 1: Sistema saludable
latencias_ok = [random.uniform(50, 300) for _ in range(100)]
status_ok = checker.status_check(
    modelo_id="llm-7b-instruct",
    modelos_activos=["llm-7b-instruct", "llm-13b-chat"],
    memoria_libre_gb=8.5,
    latencias_ms=latencias_ok,
    total_requests=500,
    errores=10,
)
print("Escenario OK:")
print("  Estado global:", status_ok["estado_global"])
for check in status_ok["checks"]:
    print("  {}: {} - {}".format(check["nombre"], check["estado"], check["mensaje"]))

# Escenario 2: Sistema degradado
latencias_malas = [random.uniform(500, 3000) for _ in range(100)]
status_malo = checker.status_check(
    modelo_id="llm-70b-preview",
    modelos_activos=["llm-7b-instruct"],  # Modelo no cargado
    memoria_libre_gb=0.3,
    latencias_ms=latencias_malas,
    total_requests=200,
    errores=50,
)
print("\nEscenario DEGRADADO:")
print("  Estado global:", status_malo["estado_global"])
for check in status_malo["checks"]:
    print("  {}: {} - {}".format(check["nombre"], check["estado"], check["mensaje"]))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m31_health_check.json", "w", encoding="utf-8") as f:
    json.dump({"escenario_ok": status_ok, "escenario_degradado": status_malo}, f, ensure_ascii=False, indent=2)
print("\nGuardado en outputs/m31_health_check.json")
```

---

## Checkpoint — 6 Tests de Verificación

```python
import json
import time
import hashlib
import random
import math
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from collections import defaultdict, deque
from pathlib import Path


# ── Clases compactas para el checkpoint ───────────────────────────────────────

@dataclass
class ChatMessage:
    role: str
    content: str

@dataclass
class ChatCompletionRequest:
    model: str
    messages: List[ChatMessage]
    max_tokens: int = 512
    temperature: float = 0.7

    def validar(self):
        errores = []
        if not self.model: errores.append("model requerido")
        if not self.messages: errores.append("messages vacio")
        for i, m in enumerate(self.messages):
            if m.role not in ("system", "user", "assistant"):
                errores.append("rol invalido en messages[" + str(i) + "]")
        if not (0 <= self.temperature <= 2.0): errores.append("temperature invalida")
        if not (1 <= self.max_tokens <= 32768): errores.append("max_tokens invalido")
        return errores

class RateLimiter:
    def __init__(self, max_rpm=60):
        self.max_rpm = max_rpm
        self._ventana = defaultdict(list)
    def verificar(self, cliente, tokens=100):
        ahora = time.time()
        self._ventana[cliente] = [t for t in self._ventana[cliente] if t > ahora - 60]
        rpm = len(self._ventana[cliente])
        if rpm >= self.max_rpm:
            return {"permitido": False, "razon": "rate limit", "rpm_actual": rpm}
        self._ventana[cliente].append(ahora)
        return {"permitido": True, "rpm_actual": rpm + 1}

@dataclass
class InferenceConfig:
    model_path: str = "model.gguf"
    model_id: str = "llm-7b"
    quantization: str = "q4_0"
    context_window: int = 4096
    max_batch_size: int = 8

BITS_POR_CUANTIZACION = {"fp32": 32, "fp16": 16, "q8_0": 8, "q4_0": 4, "q2_k": 2}

def calcular_requisitos(config):
    bits = BITS_POR_CUANTIZACION.get(config.quantization, 16)
    n_params = 7_000_000_000
    for key in ["70b", "34b", "13b", "7b"]:
        if key in config.model_id.lower():
            n_params = {"70b": 70e9, "34b": 34e9, "13b": 13e9, "7b": 7e9}[key]
            break
    vram_gb = round(n_params * bits / 8 / 1e9, 2)
    return {"vram_total_gb": vram_gb, "bits": bits, "parametros_b": round(n_params/1e9, 1)}

@dataclass
class Instancia:
    id: str
    activa: bool = True
    requests_en_curso: int = 0
    max_concurrencia: int = 8
    total_requests: int = 0
    @property
    def disponible(self): return self.activa and self.requests_en_curso < self.max_concurrencia

class LoadBalancer:
    def __init__(self, estrategia="round_robin"):
        self.estrategia = estrategia
        self.instancias = []
        self._rr_idx = 0
        self.total = 0
    def registrar(self, inst): self.instancias.append(inst)
    def seleccionar(self):
        disp = [i for i in self.instancias if i.disponible]
        if not disp: return None
        if self.estrategia == "round_robin":
            inst = disp[self._rr_idx % len(disp)]
            self._rr_idx += 1
            return inst
        elif self.estrategia == "least_connections":
            return min(disp, key=lambda i: i.requests_en_curso)
        return disp[0]
    def procesar(self, req):
        self.total += 1
        inst = self.seleccionar()
        if not inst: return {"exito": False, "error": "sin instancias"}
        inst.total_requests += 1
        return {"exito": True, "instancia": inst.id}

class HealthChecker:
    def __init__(self):
        self.cfg = {"lat_warn": 500, "lat_crit": 2000, "mem_warn": 2.0, "mem_crit": 0.5}
    def check_modelo(self, modelo, activos):
        return {"nombre": "modelo", "estado": "OK" if modelo in activos else "CRITICAL"}
    def check_memoria(self, libre_gb):
        est = "OK" if libre_gb >= self.cfg["mem_warn"] else ("WARNING" if libre_gb >= self.cfg["mem_crit"] else "CRITICAL")
        return {"nombre": "memoria", "estado": est, "valor": libre_gb}
    def check_latencia(self, lats):
        if not lats: return {"nombre": "latencia", "estado": "UNKNOWN"}
        p95 = sorted(lats)[int(0.95*len(lats))]
        est = "OK" if p95 < self.cfg["lat_warn"] else ("WARNING" if p95 < self.cfg["lat_crit"] else "CRITICAL")
        return {"nombre": "latencia_p95", "estado": est, "valor": p95}
    def status_check(self, modelo, activos, memoria, lats, total, errores):
        checks = [self.check_modelo(modelo, activos), self.check_memoria(memoria), self.check_latencia(lats)]
        prioridad = {"CRITICAL": 3, "WARNING": 2, "OK": 1, "UNKNOWN": 0}
        global_est = max(checks, key=lambda c: prioridad[c["estado"]])["estado"]
        return {"estado_global": global_est, "checks": checks}


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


def test_1_api_schema_valido():
    """Request valido no debe tener errores de validacion."""
    req = ChatCompletionRequest(
        model="llm-7b-instruct",
        messages=[ChatMessage("user", "Hola"), ChatMessage("assistant", "Hola!")],
        max_tokens=256,
        temperature=0.7,
    )
    errores = req.validar()
    assert len(errores) == 0, "Request valido no debe tener errores: " + str(errores)


def test_2_api_schema_invalido():
    """Request con temperatura invalida y sin modelo debe fallar."""
    req = ChatCompletionRequest(
        model="",
        messages=[ChatMessage("user", "test")],
        temperature=5.0,
        max_tokens=256,
    )
    errores = req.validar()
    assert len(errores) >= 2, "Debe haber al menos 2 errores, got: " + str(errores)
    tiene_model_error = any("model" in e.lower() for e in errores)
    tiene_temp_error = any("temperature" in e.lower() for e in errores)
    assert tiene_model_error, "Debe detectar error de model vacio"
    assert tiene_temp_error, "Debe detectar error de temperature invalida"


def test_3_rate_limiter():
    """Rate limiter debe rechazar cuando se supera el maximo."""
    rl = RateLimiter(max_rpm=3)
    # Primeros 3 requests permitidos
    for _ in range(3):
        res = rl.verificar("cliente-test")
        assert res["permitido"], "Los primeros 3 requests deben ser permitidos"
    # El 4to debe ser rechazado
    res4 = rl.verificar("cliente-test")
    assert not res4["permitido"], "El 4to request debe ser rechazado"
    assert "rate limit" in res4["razon"].lower(), "Razon debe mencionar rate limit"


def test_4_load_balancer_round_robin():
    """Round-robin debe distribuir requests equitativamente."""
    lb = LoadBalancer(estrategia="round_robin")
    for i in range(3):
        lb.registrar(Instancia(id="worker-" + str(i), max_concurrencia=10))

    for _ in range(9):
        lb.procesar({})

    conteos = {inst.id: inst.total_requests for inst in lb.instancias}
    for worker_id, cnt in conteos.items():
        assert cnt == 3, "Cada worker debe tener 3 requests en round-robin, got: " + str(conteos)


def test_5_health_check_critico():
    """Sistema degradado debe reportar estado CRITICAL."""
    checker = HealthChecker()
    status = checker.status_check(
        modelo="llm-70b",
        activos=["llm-7b"],        # Modelo no cargado
        memoria=0.1,               # Muy poca memoria
        lats=[3000.0] * 20,        # Latencia muy alta
        total=100,
        errores=30,
    )
    assert status["estado_global"] == "CRITICAL", \
        "Estado degradado debe ser CRITICAL, got: " + status["estado_global"]

    estados = {c["nombre"]: c["estado"] for c in status["checks"]}
    assert estados.get("modelo") == "CRITICAL", "Modelo no cargado debe ser CRITICAL"
    assert estados.get("memoria") == "CRITICAL", "Memoria escasa debe ser CRITICAL"


def test_6_config_vram_cuantizacion():
    """Cuantizacion menor debe requerir menos VRAM."""
    config_fp16 = InferenceConfig(model_id="llm-7b", quantization="fp16")
    config_q4 = InferenceConfig(model_id="llm-7b", quantization="q4_0")

    req_fp16 = calcular_requisitos(config_fp16)
    req_q4 = calcular_requisitos(config_q4)

    assert req_fp16["bits"] == 16, "FP16 debe ser 16 bits, got: " + str(req_fp16["bits"])
    assert req_q4["bits"] == 4, "Q4_0 debe ser 4 bits, got: " + str(req_q4["bits"])
    assert req_fp16["vram_total_gb"] > req_q4["vram_total_gb"], \
        "FP16 debe requerir mas VRAM que Q4, got FP16={} Q4={}".format(
            req_fp16["vram_total_gb"], req_q4["vram_total_gb"]
        )
    assert req_fp16["vram_total_gb"] / req_q4["vram_total_gb"] >= 3, \
        "FP16 debe ser al menos 4x mas VRAM que Q4"


# ── Ejecutar todos los tests ───────────────────────────────────────────────────
print("\n" + "="*60)
print("CHECKPOINT M31 — Deployment")
print("="*60)

run_test("T1: Schema API request valido", test_1_api_schema_valido)
run_test("T2: Schema API request invalido detectado", test_2_api_schema_invalido)
run_test("T3: Rate limiter rechaza exceso de RPM", test_3_rate_limiter)
run_test("T4: Load balancer round-robin distribuye equitativamente", test_4_load_balancer_round_robin)
run_test("T5: Health check detecta sistema critico", test_5_health_check_critico)
run_test("T6: Cuantizacion menor usa menos VRAM", test_6_config_vram_cuantizacion)

total = len(tests_resultados)
pasados = sum(1 for _, s, _ in tests_resultados if s == "PASS")
print("\n" + "="*60)
print("Resultado: {}/{} tests pasados".format(pasados, total))
if pasados == total:
    print("TODOS LOS TESTS PASARON - Modulo 31 completado")
else:
    for nombre, estado, msg in tests_resultados:
        if estado != "PASS":
            print("  FALLO: " + nombre + " -> " + str(msg))
print("="*60)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m31_checkpoint.json", "w", encoding="utf-8") as f:
    json.dump({
        "modulo": "M31",
        "total": total,
        "pasados": pasados,
        "tests": [{"nombre": n, "estado": s, "mensaje": m} for n, s, m in tests_resultados],
    }, f, ensure_ascii=False, indent=2)
```
