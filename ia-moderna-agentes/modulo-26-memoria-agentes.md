# Módulo 26 — Memoria a Largo Plazo en Agentes

> **Objetivo:** Implementar los cuatro tipos de memoria en agentes de IA (episódica, semántica, procedimental y de trabajo), con persistencia en disco y estrategias de recuperación relevante para conversaciones largas.
>
> **Herramientas:** Python 3.11, sqlite3, json, pathlib
>
> **Prerequisito:** M16 (Agentes ReAct), M11 (Embeddings), M13 (RAG Básico)

---

## 1. Los Cuatro Tipos de Memoria Agéntica

```python
# modulo-26-memoria-agentes.py — Parte 1: Taxonomía de memoria

"""
Tipos de memoria en agentes de IA (analogía cognitiva):

┌─────────────────────────────────────────────────────────────────┐
│  MEMORIA DE TRABAJO (Working Memory)                            │
│  Corto plazo, contexto actual, ~últimas N interacciones         │
│  Implementación: deque(maxlen=N)                                │
├─────────────────────────────────────────────────────────────────┤
│  MEMORIA EPISÓDICA (Episodic Memory)                            │
│  Experiencias pasadas con timestamp, búsqueda por similitud     │
│  Implementación: lista de episodios con embeddings              │
├─────────────────────────────────────────────────────────────────┤
│  MEMORIA SEMÁNTICA (Semantic Memory)                            │
│  Hechos, entidades, relaciones (knowledge base)                 │
│  Implementación: diccionario de entidades + grafo               │
├─────────────────────────────────────────────────────────────────┤
│  MEMORIA PROCEDIMENTAL (Procedural Memory)                      │
│  Habilidades, procedimientos, cómo hacer cosas                  │
│  Implementación: registro de herramientas + workflows           │
└─────────────────────────────────────────────────────────────────┘
"""

import json
import time
import uuid
import hashlib
import sqlite3
import math
from collections import deque
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Callable, Dict, List, Optional, Tuple
from enum import Enum


# ── Embedder simulado ─────────────────────────────────────────────────────────

class EmbedderSimulado:
    """Embeddings deterministas basados en hash para demos offline."""

    def __init__(self, dim: int = 64):
        self.dim = dim

    def encode(self, texto: str) -> List[float]:
        seed = int(hashlib.md5(texto.encode()).hexdigest(), 16)
        vec = []
        for i in range(self.dim):
            seed = (seed * 1664525 + 1013904223) & 0xFFFFFFFF
            vec.append((seed / 0xFFFFFFFF) * 2 - 1)
        norm = math.sqrt(sum(x**2 for x in vec))
        return [x / norm for x in vec] if norm > 0 else vec

    def similitud(self, a: List[float], b: List[float]) -> float:
        return sum(x*y for x,y in zip(a, b))


# ── 1. Memoria de Trabajo ─────────────────────────────────────────────────────

@dataclass
class Mensaje:
    role: str        # "user" | "assistant" | "system"
    content: str
    timestamp: float = field(default_factory=time.time)
    metadata: Dict = field(default_factory=dict)

    def to_dict(self) -> Dict:
        return {"role": self.role, "content": self.content, "timestamp": self.timestamp}


class MemoriaDeTrabajo:
    """
    Memoria de trabajo: ventana deslizante de los últimos N mensajes.
    Se usa para construir el contexto inmediato del LLM.
    """

    def __init__(self, max_mensajes: int = 20, max_tokens: int = 4096):
        self.max_mensajes = max_mensajes
        self.max_tokens = max_tokens
        self._mensajes: deque = deque(maxlen=max_mensajes)
        self._tokens_actuales = 0

    def agregar(self, role: str, content: str, metadata: Dict = None):
        msg = Mensaje(role=role, content=content, metadata=metadata or {})
        tokens_aprox = len(content) // 4  # ~4 chars por token

        # Si excede tokens, eliminar mensajes viejos (no el system)
        while self._tokens_actuales + tokens_aprox > self.max_tokens and len(self._mensajes) > 1:
            msg_viejo = self._mensajes.popleft()
            if msg_viejo.role != "system":
                self._tokens_actuales -= len(msg_viejo.content) // 4

        self._mensajes.append(msg)
        self._tokens_actuales += tokens_aprox

    def obtener_contexto(self) -> List[Dict]:
        """Retorna los mensajes en formato para el LLM."""
        return [m.to_dict() for m in self._mensajes]

    def obtener_recientes(self, n: int = 5) -> List[Mensaje]:
        return list(self._mensajes)[-n:]

    def limpiar(self):
        self._mensajes.clear()
        self._tokens_actuales = 0

    def __len__(self) -> int:
        return len(self._mensajes)

    @property
    def resumen_stats(self) -> Dict:
        return {
            "mensajes": len(self._mensajes),
            "max_mensajes": self.max_mensajes,
            "tokens_aprox": self._tokens_actuales,
            "max_tokens": self.max_tokens,
        }


# ── 2. Memoria Episódica ──────────────────────────────────────────────────────

@dataclass
class Episodio:
    """Un episodio es una experiencia pasada del agente."""
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    tipo: str = "conversacion"    # conversacion | tarea | error | insight
    contenido: str = ""
    contexto: Dict = field(default_factory=dict)  # metadatos del episodio
    timestamp: float = field(default_factory=time.time)
    embedding: List[float] = field(default_factory=list)
    importancia: float = 1.0      # 0.0 - 1.0, afecta prioridad de recuperación
    accesos: int = 0              # cuántas veces fue recuperado

    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "tipo": self.tipo,
            "contenido": self.contenido[:200],
            "timestamp": self.timestamp,
            "importancia": self.importancia,
            "accesos": self.accesos,
        }


class MemoriaEpisodica:
    """
    Almacena y recupera experiencias pasadas del agente.
    La recuperación usa similitud semántica + recencia + importancia.
    """

    def __init__(self, embedder: EmbedderSimulado, max_episodios: int = 1000):
        self.embedder = embedder
        self.max_episodios = max_episodios
        self._episodios: List[Episodio] = []

    def guardar(self, contenido: str, tipo: str = "conversacion", contexto: Dict = None, importancia: float = 1.0) -> Episodio:
        """Guarda un nuevo episodio con su embedding."""
        episodio = Episodio(
            tipo=tipo,
            contenido=contenido,
            contexto=contexto or {},
            importancia=importancia,
            embedding=self.embedder.encode(contenido),
        )
        self._episodios.append(episodio)

        # Si se supera el límite, eliminar los menos importantes y más antiguos
        if len(self._episodios) > self.max_episodios:
            self._episodios.sort(key=lambda e: e.importancia * 0.5 + (e.timestamp / time.time()) * 0.5, reverse=True)
            self._episodios = self._episodios[:self.max_episodios]

        return episodio

    def recuperar(self, query: str, k: int = 5, tipo: Optional[str] = None, min_importancia: float = 0.0) -> List[Tuple[Episodio, float]]:
        """
        Recupera los K episodios más relevantes para el query.
        Score = similitud_semántica * 0.6 + recencia * 0.2 + importancia * 0.2
        """
        if not self._episodios:
            return []

        query_emb = self.embedder.encode(query)
        ahora = time.time()
        candidatos = self._episodios

        if tipo:
            candidatos = [e for e in candidatos if e.tipo == tipo]
        if min_importancia > 0:
            candidatos = [e for e in candidatos if e.importancia >= min_importancia]

        scores = []
        for episodio in candidatos:
            sim = self.embedder.similitud(query_emb, episodio.embedding)
            # Recencia: cuán reciente es (0-1, donde 1=ahora)
            edad_dias = (ahora - episodio.timestamp) / 86400
            recencia = math.exp(-0.1 * edad_dias)  # decaimiento exponencial
            # Score combinado
            score = sim * 0.6 + recencia * 0.2 + episodio.importancia * 0.2
            scores.append((episodio, score))

        scores.sort(key=lambda x: x[1], reverse=True)
        resultado = scores[:k]

        # Actualizar accesos
        for episodio, _ in resultado:
            episodio.accesos += 1

        return resultado

    def consolidar(self):
        """
        Consolida episodios similares en uno solo (memoria comprimida).
        Útil para reducir memoria con muchos episodios redundantes.
        """
        if len(self._episodios) < 10:
            return

        consolidados = 0
        i = 0
        while i < len(self._episodios):
            episodio_actual = self._episodios[i]
            similares = []
            for j in range(i+1, len(self._episodios)):
                sim = self.embedder.similitud(episodio_actual.embedding, self._episodios[j].embedding)
                if sim > 0.95:  # muy similares
                    similares.append(j)

            if similares:
                # Combinar importancia y marcar duplicados para eliminación
                for j in sorted(similares, reverse=True):
                    episodio_actual.importancia = max(episodio_actual.importancia, self._episodios[j].importancia)
                    episodio_actual.accesos += self._episodios[j].accesos
                    self._episodios.pop(j)
                consolidados += len(similares)
            i += 1

        return consolidados

    def __len__(self) -> int:
        return len(self._episodios)


# ── 3. Memoria Semántica ──────────────────────────────────────────────────────

@dataclass
class Entidad:
    """Entidad en la base de conocimiento del agente."""
    nombre: str
    tipo: str               # persona, lugar, concepto, organización, etc.
    descripcion: str = ""
    atributos: Dict = field(default_factory=dict)
    relaciones: Dict[str, List[str]] = field(default_factory=dict)  # tipo → [entidades]
    fuentes: List[str] = field(default_factory=list)
    confianza: float = 1.0  # 0.0 - 1.0
    ultima_actualizacion: float = field(default_factory=time.time)

    def agregar_relacion(self, tipo: str, entidad_destino: str):
        if tipo not in self.relaciones:
            self.relaciones[tipo] = []
        if entidad_destino not in self.relaciones[tipo]:
            self.relaciones[tipo].append(entidad_destino)

    def to_dict(self) -> Dict:
        return {
            "nombre": self.nombre,
            "tipo": self.tipo,
            "descripcion": self.descripcion,
            "atributos": self.atributos,
            "relaciones": self.relaciones,
            "confianza": self.confianza,
        }


class MemoriaSemantica:
    """
    Base de conocimiento del agente: entidades, hechos y relaciones.
    Se actualiza cuando el agente aprende nuevas informaciones.
    """

    def __init__(self):
        self._entidades: Dict[str, Entidad] = {}
        self._hechos: List[Dict] = []   # {sujeto, predicado, objeto, confianza, timestamp}
        self._actualizaciones: int = 0

    def aprender_entidad(self, nombre: str, tipo: str, descripcion: str = "", atributos: Dict = None, fuente: str = "") -> Entidad:
        """Aprende o actualiza una entidad."""
        if nombre in self._entidades:
            # Actualizar entidad existente
            entidad = self._entidades[nombre]
            if descripcion:
                entidad.descripcion = descripcion
            if atributos:
                entidad.atributos.update(atributos)
            if fuente:
                entidad.fuentes.append(fuente)
            entidad.ultima_actualizacion = time.time()
        else:
            entidad = Entidad(
                nombre=nombre,
                tipo=tipo,
                descripcion=descripcion,
                atributos=atributos or {},
                fuentes=[fuente] if fuente else [],
            )
            self._entidades[nombre] = entidad

        self._actualizaciones += 1
        return entidad

    def aprender_relacion(self, sujeto: str, predicado: str, objeto: str, confianza: float = 1.0, fuente: str = ""):
        """Aprende una relación entre dos entidades."""
        # Asegurar que las entidades existen
        if sujeto not in self._entidades:
            self._entidades[sujeto] = Entidad(nombre=sujeto, tipo="desconocido")
        if objeto not in self._entidades:
            self._entidades[objeto] = Entidad(nombre=objeto, tipo="desconocido")

        # Agregar relación
        self._entidades[sujeto].agregar_relacion(predicado, objeto)

        # Guardar hecho
        self._hechos.append({
            "sujeto": sujeto,
            "predicado": predicado,
            "objeto": objeto,
            "confianza": confianza,
            "fuente": fuente,
            "timestamp": time.time(),
        })
        self._actualizaciones += 1

    def consultar(self, nombre: str) -> Optional[Entidad]:
        return self._entidades.get(nombre)

    def buscar(self, query: str, tipo: Optional[str] = None) -> List[Entidad]:
        """Búsqueda por nombre o descripción (substring)."""
        resultados = []
        for entidad in self._entidades.values():
            if tipo and entidad.tipo != tipo:
                continue
            if (query.lower() in entidad.nombre.lower() or
                query.lower() in entidad.descripcion.lower() or
                any(query.lower() in str(v).lower() for v in entidad.atributos.values())):
                resultados.append(entidad)
        return resultados

    def hechos_sobre(self, sujeto: str) -> List[Dict]:
        """Retorna todos los hechos conocidos sobre una entidad."""
        return [h for h in self._hechos if h["sujeto"] == sujeto]

    @property
    def stats(self) -> Dict:
        tipos = {}
        for e in self._entidades.values():
            tipos[e.tipo] = tipos.get(e.tipo, 0) + 1
        return {
            "entidades": len(self._entidades),
            "hechos": len(self._hechos),
            "tipos": tipos,
            "actualizaciones": self._actualizaciones,
        }


# ── 4. Memoria Procedimental ──────────────────────────────────────────────────

@dataclass
class Procedimiento:
    """Habilidad o workflow que el agente ha aprendido a ejecutar."""
    nombre: str
    descripcion: str
    pasos: List[str]
    herramientas_requeridas: List[str] = field(default_factory=list)
    condicion_aplicacion: str = ""   # cuándo usar este procedimiento
    tasa_exito: float = 1.0
    veces_usado: int = 0
    ultima_uso: Optional[float] = None

    def to_dict(self) -> Dict:
        return {
            "nombre": self.nombre,
            "descripcion": self.descripcion,
            "pasos": self.pasos,
            "herramientas": self.herramientas_requeridas,
            "tasa_exito": self.tasa_exito,
            "veces_usado": self.veces_usado,
        }


class MemoriaProcedimental:
    """
    Registro de habilidades y procedimientos del agente.
    Aprende de los éxitos y fracasos para mejorar futuras ejecuciones.
    """

    def __init__(self):
        self._procedimientos: Dict[str, Procedimiento] = {}

    def registrar(self, proc: Procedimiento):
        self._procedimientos[proc.nombre] = proc

    def encontrar(self, tarea: str) -> List[Procedimiento]:
        """Encuentra procedimientos aplicables a una tarea."""
        resultados = []
        for proc in self._procedimientos.values():
            if (tarea.lower() in proc.nombre.lower() or
                tarea.lower() in proc.descripcion.lower() or
                (proc.condicion_aplicacion and tarea.lower() in proc.condicion_aplicacion.lower())):
                resultados.append(proc)
        # Ordenar por tasa de éxito y veces usado
        return sorted(resultados, key=lambda p: p.tasa_exito * 0.7 + min(p.veces_usado / 10, 1) * 0.3, reverse=True)

    def registrar_resultado(self, nombre: str, exito: bool):
        """Actualiza la tasa de éxito de un procedimiento."""
        if nombre in self._procedimientos:
            proc = self._procedimientos[nombre]
            # Media móvil exponencial para la tasa de éxito
            alpha = 0.1
            proc.tasa_exito = (1 - alpha) * proc.tasa_exito + alpha * (1.0 if exito else 0.0)
            proc.veces_usado += 1
            proc.ultima_uso = time.time()


# ── Sistema de Memoria Unificado ──────────────────────────────────────────────

class SistemaMemoria:
    """
    Integra los cuatro tipos de memoria en una interfaz unificada.
    El agente usa esta clase para todas sus operaciones de memoria.
    """

    def __init__(self, nombre_agente: str, embedder: EmbedderSimulado = None):
        self.nombre_agente = nombre_agente
        embedder = embedder or EmbedderSimulado(64)
        self.trabajo = MemoriaDeTrabajo(max_mensajes=20, max_tokens=8192)
        self.episodica = MemoriaEpisodica(embedder, max_episodios=500)
        self.semantica = MemoriaSemantica()
        self.procedimental = MemoriaProcedimental()
        self._sesiones: int = 0

    def nueva_sesion(self):
        """Inicia una nueva sesión (limpia memoria de trabajo)."""
        # Guardar resumen de la sesión anterior en memoria episódica
        if len(self.trabajo) > 0:
            msgs = self.trabajo.obtener_recientes(10)
            resumen = f"Sesión {self._sesiones}: {len(msgs)} mensajes. Último: {msgs[-1].content[:100]}"
            self.episodica.guardar(resumen, tipo="sesion", importancia=0.7)

        self.trabajo.limpiar()
        self._sesiones += 1

    def aprender_de_interaccion(self, usuario: str, respuesta: str, importantes: List[str] = None):
        """
        Aprende de una interacción usuario-agente:
        1. Guarda en memoria de trabajo (contexto inmediato)
        2. Guarda episodio si es importante
        3. Extrae entidades relevantes para memoria semántica
        """
        # 1. Memoria de trabajo
        self.trabajo.agregar("user", usuario)
        self.trabajo.agregar("assistant", respuesta)

        # 2. Episodio si hay contenido importante
        if importantes:
            for item in importantes:
                self.episodica.guardar(item, tipo="insight", importancia=0.9)

        # 3. Memoria semántica: extraer entidades mencionadas
        self._extraer_y_aprender(usuario + " " + respuesta)

    def _extraer_y_aprender(self, texto: str):
        """Extrae entidades simples del texto (heurístico)."""
        import re
        # Palabras capitalizadas que podrían ser nombres propios
        palabras_cap = re.findall(r'\b[A-ZÁÉÍÓÚÑ][a-záéíóúñ]{3,}\b', texto)
        for palabra in set(palabras_cap):
            if len(palabra) > 4:  # ignorar palabras muy cortas
                if palabra not in self.semantica._entidades:
                    self.semantica.aprender_entidad(
                        nombre=palabra,
                        tipo="mencionado",
                        descripcion=f"Mencionado en conversación",
                    )

    def contexto_completo(self, query: str) -> str:
        """
        Construye el contexto completo para el LLM combinando:
        - Memoria de trabajo (contexto inmediato)
        - Episodios relevantes (experiencias pasadas)
        - Entidades conocidas (conocimiento semántico)
        """
        partes = []

        # Episodios relevantes
        episodios = self.episodica.recuperar(query, k=3)
        if episodios:
            partes.append("## Experiencias relevantes")
            for ep, score in episodios:
                partes.append(f"- [{ep.tipo}] {ep.contenido[:100]} (relevancia: {score:.2f})")

        # Entidades conocidas relacionadas
        entidades = self.semantica.buscar(query.split()[0] if query else "")
        if entidades:
            partes.append("\n## Conocimiento relacionado")
            for ent in entidades[:3]:
                partes.append(f"- {ent.nombre} ({ent.tipo}): {ent.descripcion[:80]}")

        # Procedimientos aplicables
        procs = self.procedimental.encontrar(query)
        if procs:
            partes.append("\n## Procedimientos disponibles")
            for proc in procs[:2]:
                partes.append(f"- {proc.nombre}: {proc.descripcion[:60]} (éxito: {proc.tasa_exito:.0%})")

        return "\n".join(partes) if partes else "Sin contexto previo relevante."

    @property
    def stats(self) -> Dict:
        return {
            "agente": self.nombre_agente,
            "sesiones": self._sesiones,
            "memoria_trabajo": self.trabajo.resumen_stats,
            "episodios": len(self.episodica),
            "entidades": self.semantica.stats["entidades"],
            "hechos": self.semantica.stats["hechos"],
            "procedimientos": len(self.procedimental._procedimientos),
        }


# ── Demo completo ─────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO — Sistema de Memoria Multi-nivel")
    print("=" * 60)

    embedder = EmbedderSimulado(64)
    memoria = SistemaMemoria("Claude-Demo", embedder)

    # Poblar memoria semántica
    memoria.semantica.aprender_entidad("Python", "lenguaje", "Lenguaje de programación de alto nivel", {"paradigma": "multi-paradigma"})
    memoria.semantica.aprender_entidad("OpenAI", "organización", "Empresa de IA que desarrolla GPT", {"fundada": "2015"})
    memoria.semantica.aprender_entidad("Transformer", "arquitectura", "Arquitectura base para LLMs modernos")
    memoria.semantica.aprender_relacion("OpenAI", "desarrolló", "GPT-4", confianza=1.0)
    memoria.semantica.aprender_relacion("Transformer", "es_base_de", "GPT-4", confianza=1.0)

    # Registrar procedimientos
    memoria.procedimental.registrar(Procedimiento(
        nombre="debug_python",
        descripcion="Depurar código Python con errores",
        pasos=["1. Leer el traceback", "2. Identificar la línea de error", "3. Verificar tipos", "4. Añadir prints de debug"],
        herramientas_requeridas=["Bash", "Read"],
        condicion_aplicacion="cuando hay error o excepción en python",
    ))
    memoria.procedimental.registrar(Procedimiento(
        nombre="analizar_datos",
        descripcion="Analizar un dataset con pandas",
        pasos=["1. Cargar datos", "2. Explorar shape y tipos", "3. Verificar nulos", "4. Estadísticas descriptivas"],
        herramientas_requeridas=["Bash"],
        condicion_aplicacion="cuando se necesita explorar o analizar datos",
    ))

    # Simular sesiones
    interacciones = [
        ("¿Cómo puedo usar Python para analizar datos?", "Puedes usar pandas y numpy. Python es ideal para análisis de datos.", ["Python es excelente para análisis de datos con pandas"]),
        ("Tengo un error AttributeError en mi código", "Vamos a depurar: primero lee el traceback completo.", ["AttributeError ocurre cuando se accede a atributo inexistente"]),
        ("¿Qué es OpenAI?", "OpenAI es una organización que desarrolló GPT-4 y otros modelos de IA.", None),
    ]

    for usuario, respuesta, importantes in interacciones:
        memoria.aprender_de_interaccion(usuario, respuesta, importantes)

    # Guardar episodios explícitos
    memoria.episodica.guardar("El usuario prefiere explicaciones con código de ejemplo", tipo="preferencia", importancia=0.9)
    memoria.episodica.guardar("Proyecto actual: sistema de análisis de ventas con ML", tipo="contexto", importancia=0.8)

    # Consultar contexto para nueva consulta
    print("\n--- Contexto recuperado para 'analizar datos python' ---")
    ctx = memoria.contexto_completo("analizar datos python")
    print(ctx)

    print("\n--- Stats del sistema ---")
    stats = memoria.stats
    print(json.dumps(stats, ensure_ascii=False, indent=2))

    with open("outputs/m26_memoria.json", "w", encoding="utf-8") as f:
        json.dump({"stats": stats, "contexto_ejemplo": ctx}, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m26_memoria.json")
    print("\n[M26 completado]")
```

---

## Checkpoint

```python
# checkpoint_m26.py — 6 tests para Memoria de Agentes

import time, math, hashlib
from collections import deque
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Tuple
from enum import Enum


class EmbedderSimulado:
    def __init__(self, dim=32):
        self.dim = dim
    def encode(self, texto):
        seed = int(hashlib.md5(texto.encode()).hexdigest(), 16)
        vec = []
        for i in range(self.dim):
            seed = (seed * 1664525 + 1013904223) & 0xFFFFFFFF
            vec.append((seed / 0xFFFFFFFF) * 2 - 1)
        norm = math.sqrt(sum(x**2 for x in vec))
        return [x/norm for x in vec] if norm > 0 else vec
    def similitud(self, a, b): return sum(x*y for x,y in zip(a,b))

@dataclass
class Mensaje:
    role: str; content: str; timestamp: float = field(default_factory=time.time)
    def to_dict(self): return {"role":self.role,"content":self.content,"timestamp":self.timestamp}

class MemoriaDeTrabajo:
    def __init__(self, max_mensajes=20, max_tokens=4096):
        self.max_mensajes = max_mensajes; self.max_tokens = max_tokens
        self._mensajes = deque(maxlen=max_mensajes); self._tokens_actuales = 0
    def agregar(self, role, content):
        self._mensajes.append(Mensaje(role=role, content=content))
        self._tokens_actuales += len(content)//4
    def obtener_contexto(self): return [m.to_dict() for m in self._mensajes]
    def limpiar(self): self._mensajes.clear(); self._tokens_actuales = 0
    def __len__(self): return len(self._mensajes)

@dataclass
class Episodio:
    id: str = field(default_factory=lambda: __import__('uuid').uuid4().hex[:8])
    tipo: str = "general"; contenido: str = ""
    timestamp: float = field(default_factory=time.time)
    embedding: List[float] = field(default_factory=list)
    importancia: float = 1.0; accesos: int = 0

class MemoriaEpisodica:
    def __init__(self, embedder, max_episodios=100):
        self.embedder = embedder; self.max_episodios = max_episodios; self._episodios = []
    def guardar(self, contenido, tipo="general", importancia=1.0):
        ep = Episodio(tipo=tipo, contenido=contenido, importancia=importancia, embedding=self.embedder.encode(contenido))
        self._episodios.append(ep); return ep
    def recuperar(self, query, k=5):
        if not self._episodios: return []
        q_emb = self.embedder.encode(query)
        scored = [(ep, self.embedder.similitud(q_emb, ep.embedding)) for ep in self._episodios]
        scored.sort(key=lambda x: x[1], reverse=True)
        for ep,_ in scored[:k]: ep.accesos += 1
        return scored[:k]
    def __len__(self): return len(self._episodios)

@dataclass
class Entidad:
    nombre: str; tipo: str; descripcion: str = ""; atributos: Dict = field(default_factory=dict)
    relaciones: Dict = field(default_factory=dict); confianza: float = 1.0
    def agregar_relacion(self, tipo, destino):
        if tipo not in self.relaciones: self.relaciones[tipo] = []
        if destino not in self.relaciones[tipo]: self.relaciones[tipo].append(destino)

class MemoriaSemantica:
    def __init__(self): self._entidades = {}; self._hechos = []
    def aprender_entidad(self, nombre, tipo, descripcion="", atributos=None):
        if nombre in self._entidades: self._entidades[nombre].descripcion = descripcion or self._entidades[nombre].descripcion
        else: self._entidades[nombre] = Entidad(nombre=nombre, tipo=tipo, descripcion=descripcion, atributos=atributos or {})
        return self._entidades[nombre]
    def aprender_relacion(self, sujeto, predicado, objeto, confianza=1.0):
        for n in [sujeto, objeto]:
            if n not in self._entidades: self._entidades[n] = Entidad(nombre=n, tipo="desconocido")
        self._entidades[sujeto].agregar_relacion(predicado, objeto)
        self._hechos.append({"sujeto":sujeto,"predicado":predicado,"objeto":objeto,"confianza":confianza})
    def consultar(self, nombre): return self._entidades.get(nombre)


def test_01_memoria_trabajo_ventana_deslizante():
    """MemoriaDeTrabajo respeta el límite de mensajes."""
    mt = MemoriaDeTrabajo(max_mensajes=5)
    for i in range(8):
        mt.agregar("user", f"Mensaje {i}")
    assert len(mt) == 5, "Debe mantener solo los últimos 5 mensajes"
    ctx = mt.obtener_contexto()
    assert ctx[-1]["content"] == "Mensaje 7"  # el más reciente
    print("✓ test_01 — Ventana deslizante OK")

def test_02_memoria_trabajo_limpiar():
    """limpiar() resetea la memoria de trabajo."""
    mt = MemoriaDeTrabajo()
    mt.agregar("user", "hola"); mt.agregar("assistant", "hola tú")
    assert len(mt) == 2
    mt.limpiar()
    assert len(mt) == 0
    print("✓ test_02 — Limpiar memoria de trabajo OK")

def test_03_memoria_episodica_guardar_recuperar():
    """MemoriaEpisodica guarda y recupera episodios relevantes."""
    emb = EmbedderSimulado(32)
    me = MemoriaEpisodica(emb)
    me.guardar("Python es un lenguaje de programación", "conocimiento", importancia=0.9)
    me.guardar("El usuario prefiere ejemplos de código", "preferencia", importancia=0.8)
    me.guardar("Reunión con el equipo de producto", "evento", importancia=0.5)
    assert len(me) == 3
    resultados = me.recuperar("programación Python", k=2)
    assert len(resultados) == 2
    assert resultados[0][0].tipo in ["conocimiento", "preferencia"]
    print("✓ test_03 — Memoria episódica OK")

def test_04_episodio_accesos_incrementan():
    """Los accesos de un episodio se incrementan al recuperarlo."""
    emb = EmbedderSimulado(32)
    me = MemoriaEpisodica(emb)
    ep = me.guardar("Dato importante", importancia=1.0)
    assert ep.accesos == 0
    me.recuperar("dato importante", k=1)
    assert ep.accesos == 1
    me.recuperar("dato importante", k=1)
    assert ep.accesos == 2
    print("✓ test_04 — Accesos de episodios OK")

def test_05_memoria_semantica_entidades_relaciones():
    """MemoriaSemantica aprende entidades y relaciones."""
    ms = MemoriaSemantica()
    ms.aprender_entidad("GPT-4", "modelo", "LLM de OpenAI", {"params": "1.8T"})
    ms.aprender_entidad("OpenAI", "org", "Empresa de IA")
    ms.aprender_relacion("OpenAI", "desarrolló", "GPT-4", confianza=1.0)
    ent = ms.consultar("GPT-4")
    assert ent is not None
    assert ent.tipo == "modelo"
    assert ent.atributos.get("params") == "1.8T"
    org = ms.consultar("OpenAI")
    assert "GPT-4" in org.relaciones.get("desarrolló", [])
    assert len(ms._hechos) == 1
    print("✓ test_05 — Memoria semántica OK")

def test_06_sistema_memoria_integrado():
    """SistemaMemoria integra todos los tipos correctamente."""
    emb = EmbedderSimulado(32)
    trabajo = MemoriaDeTrabajo(max_mensajes=10)
    episodica = MemoriaEpisodica(emb)
    semantica = MemoriaSemantica()

    # Simular ciclo completo
    trabajo.agregar("user", "¿Qué es Python?")
    trabajo.agregar("assistant", "Python es un lenguaje de alto nivel.")
    episodica.guardar("Python: lenguaje de alto nivel, tipado dinámico", importancia=0.9)
    semantica.aprender_entidad("Python", "lenguaje", "Lenguaje de programación")
    semantica.aprender_relacion("Python", "tipo", "lenguaje_scripting")

    assert len(trabajo) == 2
    assert len(episodica) == 1
    assert semantica.consultar("Python") is not None
    ep_relevantes = episodica.recuperar("Python programación", k=1)
    assert len(ep_relevantes) > 0
    print("✓ test_06 — Sistema memoria integrado OK")


if __name__ == "__main__":
    tests = [
        test_01_memoria_trabajo_ventana_deslizante,
        test_02_memoria_trabajo_limpiar,
        test_03_memoria_episodica_guardar_recuperar,
        test_04_episodio_accesos_incrementan,
        test_05_memoria_semantica_entidades_relaciones,
        test_06_sistema_memoria_integrado,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M26 completado")
```

---

## Resumen

| Tipo de Memoria | Duración | Capacidad | Recuperación | Implementación |
|---|---|---|---|---|
| **Trabajo** | Sesión actual | ~20 msgs / 8K tokens | Secuencial (FIFO) | `deque(maxlen=N)` |
| **Episódica** | Permanente | 100–10K episodios | Similitud semántica + recencia | Lista + embeddings |
| **Semántica** | Permanente | Ilimitada | Búsqueda por nombre/tipo | Diccionario de entidades |
| **Procedimental** | Permanente | Decenas de habilidades | Por condición de aplicación | Diccionario de procedimientos |

```python
# Flujo de uso en un agente real:
def procesar_consulta(self, query: str) -> str:
    # 1. Construir contexto con memoria larga
    ctx_pasado = self.memoria.contexto_completo(query)

    # 2. Contexto inmediato (memoria de trabajo)
    ctx_inmediato = self.memoria.trabajo.obtener_contexto()

    # 3. Llamar al LLM con contexto enriquecido
    respuesta = self.llm.completar(
        system=f"Contexto relevante:\n{ctx_pasado}",
        messages=ctx_inmediato + [{"role": "user", "content": query}]
    )

    # 4. Aprender de la interacción
    self.memoria.aprender_de_interaccion(query, respuesta)
    return respuesta
```
