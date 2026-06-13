# Módulo 15 — Graph RAG — Conocimiento Estructurado

**Objetivo:** Implementar Graph RAG: extraer entidades y relaciones para construir un grafo de conocimiento, realizar búsquedas estructuradas sobre el grafo, y combinar recuperación vectorial con traversal de grafo para respuestas más precisas.

**Herramientas:** `numpy`, `networkx`, `torch`, `transformers`

**Prerequisito:** M14 (RAG Avanzado)

---

## 1. ¿Por qué Graph RAG?

```python
# grafo_vs_vectorial.py
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# RAG vectorial: recupera chunks similares, pero pierde RELACIONES entre entidades.
#
# Ejemplo:
#   Doc 1: "BERT fue desarrollado por Google en 2018."
#   Doc 2: "El equipo de Google Brain creó GPT-2 junto a OpenAI."
#   Doc 3: "OpenAI lanzó ChatGPT basado en GPT-3.5."
#
# Query: "¿Quién creó BERT y qué relación tiene con ChatGPT?"
#
# RAG vectorial: recupera Doc 1 (BERT) y Doc 3 (ChatGPT) por similitud.
# Pero NO puede razonar que BERT → Google ← Google Brain → GPT-2 → ... → ChatGPT
#
# Graph RAG: construye el grafo:
#   BERT --[desarrollado_por]--> Google
#   Google Brain --[parte_de]--> Google
#   OpenAI --[publicó]--> GPT-2
#   OpenAI --[lanzó]--> ChatGPT
#   ChatGPT --[basado_en]--> GPT-3.5
#
# Y puede hacer traversal: BERT → Google ← Google Brain → (trabaja_con) → OpenAI → ChatGPT

ventajas_graph_rag = {
    "relaciones_multi_hop": "Razonamiento a través de múltiples saltos en el grafo",
    "deduplicacion": "Entidades unificadas (BERT = 'BERT model' = 'Bidirectional BERT')",
    "resumen_por_entidad": "Contexto completo de una entidad a partir de sus aristas",
    "consultas_estructuradas": "WHO-WHAT-HOW queries sobre el grafo",
    "menor_alucinacion": "Las relaciones extraídas están ancladas en el texto fuente",
}
for k, v in ventajas_graph_rag.items():
    print(f"  {k}: {v}")

np.save("outputs/m15_intro.npy", np.array([1]))
print("\noutputs/m15_intro.npy guardado")
```

---

## 2. Extracción de entidades y relaciones (NER + RE)

```python
# extraccion_entidades.py
import numpy as np
import re
import os
from typing import List, Tuple, Dict, Set

os.makedirs("outputs", exist_ok=True)

# --- NER simplificado basado en reglas ---
TIPOS_ENTIDAD = {
    "ORGANIZACION": ["Google", "OpenAI", "Meta", "Microsoft", "Anthropic",
                     "DeepMind", "Hugging Face", "Stanford", "MIT", "Berkeley"],
    "MODELO": ["BERT", "GPT", "GPT-2", "GPT-3", "GPT-4", "ChatGPT", "LLaMA",
               "Llama 2", "Claude", "Gemini", "Mistral", "T5", "RoBERTa",
               "Falcon", "PaLM", "LLaMA 3"],
    "TECNICA": ["Transformer", "Attention", "RLHF", "LoRA", "QLoRA",
                "Fine-tuning", "RAG", "PEFT", "DPO", "SFT", "MLM",
                "Diffusion", "VAE", "GAN"],
    "PERSONA": ["Vaswani", "Devlin", "Radford", "Altman", "LeCun",
                "Bengio", "Hinton", "Karpathy", "Sutskever"],
}

def extraer_entidades(texto: str) -> List[Tuple[str, str]]:
    """
    Extrae entidades con su tipo.
    Retorna lista de (entidad, tipo).
    """
    entidades = []
    for tipo, lista in TIPOS_ENTIDAD.items():
        for entidad in lista:
            # Búsqueda case-insensitive con delimitadores de palabra
            patron = r'\b' + re.escape(entidad) + r'\b'
            if re.search(patron, texto, re.IGNORECASE):
                entidades.append((entidad, tipo))
    return entidades

# --- Extracción de relaciones con patrones ---
PATRONES_RELACION = [
    # (patron_regex, tipo_relacion)
    (r'(\w+(?:\s\w+)*)\s+(?:fue\s+)?(?:creado|desarrollado|publicado|lanzado)\s+por\s+(\w+(?:\s\w+)*)',
     "creado_por"),
    (r'(\w+(?:\s\w+)*)\s+(?:usa|utiliza|emplea|incorpora)\s+(\w+(?:\s\w+)*)',
     "usa"),
    (r'(\w+(?:\s\w+)*)\s+(?:es\s+(?:parte\s+de|un\s+derivado\s+de|basado\s+en))\s+(\w+(?:\s\w+)*)',
     "parte_de"),
    (r'(\w+(?:\s\w+)*)\s+(?:mejoró|superó|reemplazó)\s+(?:a\s+)?(\w+(?:\s\w+)*)',
     "mejora"),
    (r'(\w+(?:\s\w+)*)\s+(?:introdujo|presentó|propuso)\s+(\w+(?:\s\w+)*)',
     "introdujo"),
    (r'(\w+(?:\s\w+)*)\s+y\s+(\w+(?:\s\w+)*)\s+(?:colaboraron|trabajaron)',
     "colabora_con"),
]

def extraer_relaciones(texto: str,
                       entidades_conocidas: Set[str]) -> List[Tuple[str, str, str]]:
    """
    Extrae tripletas (sujeto, relacion, objeto).
    Filtra para que sujeto y objeto sean entidades conocidas.
    """
    relaciones = []
    for patron, tipo_rel in PATRONES_RELACION:
        for match in re.finditer(patron, texto, re.IGNORECASE):
            sujeto = match.group(1).strip()
            objeto = match.group(2).strip()
            # Verificar que son entidades conocidas (NER)
            for ent_s in entidades_conocidas:
                for ent_o in entidades_conocidas:
                    if (ent_s.lower() in sujeto.lower() and
                            ent_o.lower() in objeto.lower() and
                            ent_s != ent_o):
                        relaciones.append((ent_s, tipo_rel, ent_o))
    return relaciones

# --- Demo de extracción ---
corpus = [
    "BERT fue desarrollado por Google en 2018 usando el Transformer.",
    "GPT-2 fue creado por OpenAI y usa la arquitectura Transformer.",
    "LLaMA fue publicado por Meta como un modelo de lenguaje open-source.",
    "Claude es un modelo desarrollado por Anthropic que usa RLHF.",
    "Mistral mejoró a LLaMA en varios benchmarks usando atención deslizante.",
    "RAG usa Attention para recuperar y generar respuestas precisas.",
    "LoRA usa fine-tuning eficiente que mejora el proceso de SFT.",
]

print("=== Extracción de Entidades y Relaciones ===\n")
todas_entidades = {}  # entidad → tipo
todas_relaciones = []

for oracion in corpus:
    ents = extraer_entidades(oracion)
    for ent, tipo in ents:
        todas_entidades[ent] = tipo

    ent_set = set(todas_entidades.keys())
    rels = extraer_relaciones(oracion, ent_set)
    todas_relaciones.extend(rels)

    if ents or rels:
        print(f"Oración: {oracion[:60]}...")
        if ents:
            print(f"  Entidades: {ents}")
        if rels:
            print(f"  Relaciones: {rels}")

print(f"\nTotal entidades únicas: {len(todas_entidades)}")
print(f"Total relaciones extraídas: {len(todas_relaciones)}")

import json
with open("outputs/m15_entidades_relaciones.json", "w", encoding="utf-8") as f:
    json.dump({
        "entidades": todas_entidades,
        "relaciones": todas_relaciones
    }, f, ensure_ascii=False, indent=2)
print("outputs/m15_entidades_relaciones.json guardado")
```

---

## 3. Construcción del grafo de conocimiento

```python
# grafo_conocimiento.py
import numpy as np
import os
import json
from typing import List, Dict, Set, Tuple, Optional
from collections import defaultdict, deque

os.makedirs("outputs", exist_ok=True)

class GrafoConocimiento:
    """
    Grafo de conocimiento implementado con lista de adyacencia.
    Soporta: entidades con propiedades, relaciones dirigidas con peso,
    traversal BFS/DFS, búsqueda por entidad, comunidades.
    """
    def __init__(self):
        self.entidades: Dict[str, Dict] = {}          # id → {tipo, propiedades, texto_fuente}
        self.aristas: Dict[str, List[Dict]] = defaultdict(list)  # id → [{relacion, target, peso}]
        self.aristas_invertidas: Dict[str, List[Dict]] = defaultdict(list)
        self.textos_fuente: Dict[str, str] = {}       # id → texto del chunk de origen

    def agregar_entidad(self, id_: str, tipo: str,
                        propiedades: dict = None, texto_fuente: str = ""):
        self.entidades[id_] = {
            "tipo": tipo,
            "propiedades": propiedades or {},
            "id": id_
        }
        if texto_fuente:
            self.textos_fuente[id_] = texto_fuente

    def agregar_relacion(self, origen: str, relacion: str, destino: str,
                         peso: float = 1.0, texto_fuente: str = ""):
        """Añade arista dirigida origen → destino."""
        # Auto-crear entidades si no existen
        if origen not in self.entidades:
            self.agregar_entidad(origen, "DESCONOCIDO")
        if destino not in self.entidades:
            self.agregar_entidad(destino, "DESCONOCIDO")

        arista = {"relacion": relacion, "target": destino,
                  "peso": peso, "texto_fuente": texto_fuente}
        self.aristas[origen].append(arista)
        # Arista inversa para traversal bidireccional
        self.aristas_invertidas[destino].append({
            "relacion": f"inv_{relacion}", "target": origen,
            "peso": peso, "texto_fuente": texto_fuente
        })

    def vecinos(self, id_: str, tipos_relacion: List[str] = None,
                bidireccional: bool = False) -> List[Dict]:
        """Retorna vecinos de una entidad."""
        resultado = list(self.aristas.get(id_, []))
        if bidireccional:
            resultado += list(self.aristas_invertidas.get(id_, []))
        if tipos_relacion:
            resultado = [a for a in resultado if a["relacion"] in tipos_relacion]
        return resultado

    def bfs(self, inicio: str, max_profundidad: int = 2,
            tipos_relacion: List[str] = None) -> Dict[str, int]:
        """
        BFS desde la entidad de inicio.
        Retorna {entidad: profundidad} para todos los nodos alcanzados.
        """
        visitados = {inicio: 0}
        cola = deque([(inicio, 0)])

        while cola:
            nodo, prof = cola.popleft()
            if prof >= max_profundidad:
                continue
            for arista in self.vecinos(nodo, tipos_relacion, bidireccional=True):
                vecino = arista["target"]
                if vecino not in visitados:
                    visitados[vecino] = prof + 1
                    cola.append((vecino, prof + 1))
        return visitados

    def subgrafo_entidad(self, id_: str,
                         profundidad: int = 2) -> Tuple[Dict, List]:
        """Extrae subgrafo centrado en una entidad."""
        nodos_en_subgrafo = self.bfs(id_, max_profundidad=profundidad)
        entidades_sub = {n: self.entidades.get(n, {"tipo": "?"})
                         for n in nodos_en_subgrafo}
        aristas_sub = []
        for origen in nodos_en_subgrafo:
            for arista in self.aristas.get(origen, []):
                if arista["target"] in nodos_en_subgrafo:
                    aristas_sub.append((origen, arista["relacion"],
                                        arista["target"]))
        return entidades_sub, aristas_sub

    def paths_entre(self, origen: str, destino: str,
                    max_longitud: int = 4) -> List[List[str]]:
        """Encuentra todos los caminos entre dos entidades."""
        paths = []

        def dfs(nodo_actual, camino, profundidad):
            if nodo_actual == destino:
                paths.append(list(camino))
                return
            if profundidad >= max_longitud:
                return
            for arista in self.vecinos(nodo_actual, bidireccional=True):
                vecino = arista["target"]
                if vecino not in camino:
                    camino.append(vecino)
                    dfs(vecino, camino, profundidad + 1)
                    camino.pop()

        dfs(origen, [origen], 0)
        return paths

    def contexto_entidad(self, id_: str) -> str:
        """
        Genera descripción textual del contexto de una entidad:
        sus relaciones directas y los textos fuente.
        """
        if id_ not in self.entidades:
            return f"Entidad '{id_}' no encontrada en el grafo."

        lineas = [f"=== {id_} ({self.entidades[id_]['tipo']}) ==="]

        if id_ in self.textos_fuente:
            lineas.append(f"Fuente: {self.textos_fuente[id_][:100]}")

        relaciones_salientes = self.aristas.get(id_, [])
        if relaciones_salientes:
            lineas.append(f"\nRelaciones de {id_}:")
            for a in relaciones_salientes[:10]:
                lineas.append(f"  -{a['relacion']}-> {a['target']}")

        relaciones_entrantes = self.aristas_invertidas.get(id_, [])
        if relaciones_entrantes:
            lineas.append(f"\nEntidades que apuntan a {id_}:")
            for a in relaciones_entrantes[:5]:
                lineas.append(f"  <-{a['relacion'].replace('inv_', '')}- {a['target']}")

        return "\n".join(lineas)

    def stats(self) -> Dict:
        return {
            "n_entidades": len(self.entidades),
            "n_aristas": sum(len(v) for v in self.aristas.values()),
            "tipos_entidad": defaultdict(int,
                {k: sum(1 for e in self.entidades.values() if e["tipo"] == k)
                 for k in set(e["tipo"] for e in self.entidades.values())}),
        }

# --- Construir grafo de conocimiento de NLP ---
kg = GrafoConocimiento()

# Entidades
entidades_nlp = [
    ("BERT", "MODELO"), ("GPT-2", "MODELO"), ("GPT-3", "MODELO"),
    ("GPT-4", "MODELO"), ("ChatGPT", "MODELO"), ("LLaMA", "MODELO"),
    ("Claude", "MODELO"), ("Gemini", "MODELO"), ("Mistral", "MODELO"),
    ("T5", "MODELO"), ("RoBERTa", "MODELO"),
    ("Google", "ORGANIZACION"), ("OpenAI", "ORGANIZACION"),
    ("Meta", "ORGANIZACION"), ("Anthropic", "ORGANIZACION"),
    ("DeepMind", "ORGANIZACION"), ("Microsoft", "ORGANIZACION"),
    ("Transformer", "TECNICA"), ("RLHF", "TECNICA"),
    ("LoRA", "TECNICA"), ("MLM", "TECNICA"), ("RAG", "TECNICA"),
]

for id_, tipo in entidades_nlp:
    kg.agregar_entidad(id_, tipo)

# Relaciones
relaciones_nlp = [
    ("BERT", "creado_por", "Google", "BERT fue desarrollado por Google en 2018."),
    ("BERT", "usa", "Transformer", "BERT usa la arquitectura Transformer."),
    ("BERT", "usa", "MLM", "BERT fue preentrenado con MLM."),
    ("GPT-2", "creado_por", "OpenAI", "GPT-2 fue publicado por OpenAI en 2019."),
    ("GPT-2", "usa", "Transformer", "GPT-2 usa decoder-only Transformer."),
    ("GPT-3", "creado_por", "OpenAI", "GPT-3 es el sucesor de GPT-2."),
    ("GPT-4", "creado_por", "OpenAI", "GPT-4 es el modelo más avanzado de OpenAI."),
    ("ChatGPT", "basado_en", "GPT-3", "ChatGPT está basado en GPT-3.5."),
    ("ChatGPT", "basado_en", "RLHF", "ChatGPT usa RLHF para alineación."),
    ("LLaMA", "creado_por", "Meta", "LLaMA fue publicado por Meta AI."),
    ("Claude", "creado_por", "Anthropic", "Claude es el asistente de Anthropic."),
    ("Claude", "usa", "RLHF", "Claude usa Constitutional AI y RLHF."),
    ("Gemini", "creado_por", "Google", "Gemini es el modelo multimodal de Google."),
    ("Gemini", "creado_por", "DeepMind", "Gemini fue desarrollado con DeepMind."),
    ("Mistral", "mejora", "LLaMA", "Mistral 7B supera a LLaMA en varios benchmarks."),
    ("RAG", "usa", "Transformer", "RAG usa Transformer como LLM backbone."),
    ("LoRA", "mejora", "RLHF", "LoRA hace fine-tuning más eficiente que RLHF completo."),
    ("Microsoft", "invirtió_en", "OpenAI", "Microsoft invirtió $10B en OpenAI."),
]

for origen, relacion, destino, fuente in relaciones_nlp:
    kg.agregar_relacion(origen, relacion, destino, texto_fuente=fuente)

stats = kg.stats()
print("=== Grafo de Conocimiento NLP ===")
print(f"Entidades: {stats['n_entidades']}")
print(f"Aristas: {stats['n_aristas']}")
print(f"Tipos: {dict(stats['tipos_entidad'])}")

# Contexto de una entidad
print(f"\n{kg.contexto_entidad('OpenAI')}")

# BFS desde BERT (profundidad 2)
vecinos_bert = kg.bfs("BERT", max_profundidad=2)
print(f"\nVecinos de BERT (depth≤2): {list(vecinos_bert.keys())}")

# Subgrafo
ents_sub, aristas_sub = kg.subgrafo_entidad("OpenAI", profundidad=2)
print(f"\nSubgrafo de OpenAI: {len(ents_sub)} entidades, {len(aristas_sub)} aristas")

# Paths entre BERT y ChatGPT
paths = kg.paths_entre("BERT", "ChatGPT", max_longitud=4)
if paths:
    print(f"\nCaminos BERT → ChatGPT: {len(paths)}")
    for p in paths[:3]:
        print(f"  {' → '.join(p)}")

with open("outputs/m15_grafo_conocimiento.json", "w", encoding="utf-8") as f:
    json.dump({
        "stats": {k: v for k, v in stats.items() if k != "tipos_entidad"},
        "vecinos_bert": list(vecinos_bert.keys()),
        "subgrafo_openai_size": len(ents_sub)
    }, f, indent=2)
print("\noutputs/m15_grafo_conocimiento.json guardado")
```

---

## 4. Graph RAG — búsqueda híbrida vectorial + grafo

```python
# graph_rag.py
import numpy as np
import os
from typing import List, Dict, Set
from collections import defaultdict, deque

os.makedirs("outputs", exist_ok=True)

class EmbedderSimulado:
    def __init__(self, dim=64):
        self.dim = dim
    def encode(self, textos):
        vecs = []
        for t in textos:
            seed = sum(ord(c)*(i+1) for i,c in enumerate(t[:200])) % (2**31)
            rng = np.random.RandomState(seed)
            v = rng.randn(self.dim).astype(np.float32)
            vecs.append(v / (np.linalg.norm(v) + 1e-9))
        return np.stack(vecs)

class GraphRAG:
    """
    Graph RAG: combina búsqueda vectorial con traversal de grafo.

    Pipeline:
    1. Búsqueda vectorial → Top-K chunks relevantes
    2. Extraer entidades de los chunks recuperados
    3. Expandir con vecinos del grafo (depth=2)
    4. Combinar contexto vectorial + contexto del grafo
    5. LLM genera respuesta con contexto enriquecido

    Ventaja sobre RAG básico:
    - Captura relaciones entre entidades
    - Permite responder preguntas multi-hop
    - Agrega información distribuida en múltiples chunks
    """
    def __init__(self, embedder, grafo, llm_fn=None):
        self.embedder = embedder
        self.grafo = grafo
        self.llm_fn = llm_fn or self._llm_sim
        self.chunks: List[Dict] = []
        self.chunk_vectors: List[np.ndarray] = []
        self.chunk_entidades: Dict[str, List[str]] = {}  # chunk_id → entidades

    def _llm_sim(self, prompt): return f"[LLM] {prompt[:100]}..."

    def indexar_chunks(self, chunks: List[Dict], entidades_conocidas: Set[str]):
        """
        chunks: lista de {"id": str, "texto": str}
        Extrae entidades de cada chunk para enriquecer el grafo.
        """
        import re
        for chunk in chunks:
            emb = self.embedder.encode([chunk["texto"]])[0]
            self.chunks.append(chunk)
            self.chunk_vectors.append(emb / (np.linalg.norm(emb) + 1e-9))
            # Extraer entidades presentes en este chunk
            ents_en_chunk = []
            for ent in entidades_conocidas:
                if re.search(r'\b' + re.escape(ent) + r'\b',
                             chunk["texto"], re.IGNORECASE):
                    ents_en_chunk.append(ent)
            self.chunk_entidades[chunk["id"]] = ents_en_chunk

    def _buscar_vectorial(self, query: str, k: int = 5) -> List[Dict]:
        """Búsqueda semántica estándar."""
        q_emb = self.embedder.encode([query])[0]
        q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)
        sims = np.stack(self.chunk_vectors) @ q_norm
        top_k = np.argsort(sims)[::-1][:k]
        return [{"score": float(sims[i]), **self.chunks[i]} for i in top_k]

    def _expandir_con_grafo(self, entidades: List[str],
                             profundidad: int = 2) -> str:
        """
        Expande el contexto usando el grafo de conocimiento.
        BFS desde las entidades encontradas → contexto adicional.
        """
        contexto_grafo = []
        visitados_globales = set()

        for ent in entidades:
            if ent not in self.grafo.entidades:
                continue
            # BFS para encontrar entidades relacionadas
            vecinos = self.grafo.bfs(ent, max_profundidad=profundidad)
            for vecino, dist in vecinos.items():
                if vecino == ent or vecino in visitados_globales:
                    continue
                visitados_globales.add(vecino)
                # Obtener las relaciones que conectan ent con vecino
                for arista in self.grafo.aristas.get(ent, []):
                    if arista["target"] == vecino and arista.get("texto_fuente"):
                        contexto_grafo.append(
                            f"[Grafo] {arista['texto_fuente']}"
                        )

        return "\n".join(contexto_grafo[:10])  # máximo 10 relaciones

    def _extraer_entidades_de_chunks(self,
                                      chunks: List[Dict]) -> List[str]:
        """Extrae entidades de los chunks recuperados."""
        entidades = []
        for chunk in chunks:
            chunk_id = chunk.get("id", "")
            if chunk_id in self.chunk_entidades:
                entidades.extend(self.chunk_entidades[chunk_id])
        return list(set(entidades))

    def query(self, pregunta: str, k: int = 3,
               profundidad_grafo: int = 2) -> Dict:
        """Graph RAG query completo."""
        # 1. Búsqueda vectorial
        chunks_vectoriales = self._buscar_vectorial(pregunta, k=k)

        # 2. Extraer entidades de chunks
        entidades = self._extraer_entidades_de_chunks(chunks_vectoriales)

        # 3. Expandir con grafo
        contexto_grafo = self._expandir_con_grafo(entidades, profundidad_grafo)

        # 4. Construir prompt enriquecido
        contexto_vectorial = "\n".join(
            [f"[Chunk {i+1}] {c['texto']}"
             for i, c in enumerate(chunks_vectoriales)]
        )

        prompt = f"""CONTEXTO DE DOCUMENTOS:
{contexto_vectorial}

CONTEXTO DEL GRAFO DE CONOCIMIENTO:
{contexto_grafo if contexto_grafo else '[Sin relaciones de grafo encontradas]'}

PREGUNTA: {pregunta}
RESPUESTA (usa ambos contextos):"""

        respuesta = self.llm_fn(prompt)

        return {
            "pregunta": pregunta,
            "chunks_vectoriales": chunks_vectoriales,
            "entidades_encontradas": entidades,
            "contexto_grafo": contexto_grafo,
            "prompt_len": len(prompt),
            "respuesta": respuesta
        }

# --- Demo: GrafoConocimiento mínimo inline ---
class GrafoMin:
    def __init__(self):
        self.entidades = {}
        self.aristas = defaultdict(list)
        self.aristas_inv = defaultdict(list)

    def add_ent(self, id_, tipo):
        self.entidades[id_] = {"tipo": tipo}

    def add_rel(self, o, r, d, fuente=""):
        if o not in self.entidades: self.add_ent(o, "?")
        if d not in self.entidades: self.add_ent(d, "?")
        self.aristas[o].append({"relacion": r, "target": d, "texto_fuente": fuente})

    def bfs(self, inicio, max_profundidad=2):
        visitados = {inicio: 0}
        cola = deque([(inicio, 0)])
        while cola:
            nodo, prof = cola.popleft()
            if prof >= max_profundidad: continue
            for a in self.aristas.get(nodo, []):
                v = a["target"]
                if v not in visitados:
                    visitados[v] = prof + 1
                    cola.append((v, prof + 1))
        return visitados

grafo = GrafoMin()
entidades_data = [
    ("BERT", "MODELO"), ("Google", "ORGANIZACION"), ("Transformer", "TECNICA"),
    ("MLM", "TECNICA"), ("OpenAI", "ORGANIZACION"), ("GPT-2", "MODELO"),
    ("ChatGPT", "MODELO"), ("RLHF", "TECNICA"), ("LLaMA", "MODELO"),
    ("Meta", "ORGANIZACION"), ("Mistral", "MODELO"),
]
for id_, tipo in entidades_data:
    grafo.add_ent(id_, tipo)

relaciones_data = [
    ("BERT", "creado_por", "Google", "BERT fue desarrollado por Google."),
    ("BERT", "usa", "Transformer", "BERT usa arquitectura Transformer."),
    ("BERT", "usa", "MLM", "BERT preentrenado con MLM."),
    ("GPT-2", "creado_por", "OpenAI", "GPT-2 creado por OpenAI."),
    ("ChatGPT", "basado_en", "RLHF", "ChatGPT usa RLHF."),
    ("LLaMA", "creado_por", "Meta", "LLaMA de Meta AI."),
    ("Mistral", "mejora", "LLaMA", "Mistral supera a LLaMA."),
]
for o, r, d, f in relaciones_data:
    grafo.add_rel(o, r, d, f)

np.random.seed(42)
embedder = EmbedderSimulado(dim=64)
graph_rag = GraphRAG(embedder, grafo)

chunks = [
    {"id": "c0", "texto": "BERT fue desarrollado por Google usando Transformer y MLM."},
    {"id": "c1", "texto": "GPT-2 y ChatGPT son modelos de OpenAI basados en RLHF."},
    {"id": "c2", "texto": "LLaMA de Meta fue mejorado por Mistral en eficiencia."},
    {"id": "c3", "texto": "El Transformer usa atención multi-cabeza en paralelo."},
]
entidades_conocidas = set(grafo.entidades.keys())
graph_rag.indexar_chunks(chunks, entidades_conocidas)

print("=== Graph RAG Demo ===\n")
for pregunta in ["¿Quién creó BERT y qué técnicas usa?",
                 "¿Qué modelos mejoran a LLaMA?"]:
    res = graph_rag.query(pregunta, k=2, profundidad_grafo=2)
    print(f"Pregunta: {pregunta}")
    print(f"Entidades encontradas: {res['entidades_encontradas']}")
    print(f"Contexto grafo: {res['contexto_grafo'][:100] or '[vacío]'}")
    print(f"Prompt length: {res['prompt_len']} chars")
    print()

np.save("outputs/m15_graph_rag_demo.npy", np.array([1]))
print("outputs/m15_graph_rag_demo.npy guardado")
```

---

## 5. Consultas estructuradas en el grafo

```python
# consultas_grafo.py
import numpy as np
import os
from collections import defaultdict, deque
from typing import List, Dict, Optional

os.makedirs("outputs", exist_ok=True)

class GrafoMin:
    def __init__(self):
        self.entidades = {}
        self.aristas = defaultdict(list)

    def add_ent(self, id_, tipo, props=None):
        self.entidades[id_] = {"tipo": tipo, "props": props or {}}

    def add_rel(self, o, r, d, fuente="", peso=1.0):
        if o not in self.entidades: self.add_ent(o, "?")
        if d not in self.entidades: self.add_ent(d, "?")
        self.aristas[o].append({"relacion": r, "target": d,
                                 "fuente": fuente, "peso": peso})

    def bfs(self, inicio, max_depth=2, tipos_rel=None):
        visitados = {inicio: 0}
        cola = deque([(inicio, 0)])
        while cola:
            nodo, prof = cola.popleft()
            if prof >= max_depth: continue
            for a in self.aristas.get(nodo, []):
                if tipos_rel and a["relacion"] not in tipos_rel: continue
                v = a["target"]
                if v not in visitados:
                    visitados[v] = prof + 1
                    cola.append((v, prof + 1))
        return visitados

def construir_grafo_nlp():
    g = GrafoMin()
    entidades = [
        ("BERT", "MODELO", {"anio": 2018, "params": "340M"}),
        ("GPT-2", "MODELO", {"anio": 2019, "params": "1.5B"}),
        ("GPT-3", "MODELO", {"anio": 2020, "params": "175B"}),
        ("GPT-4", "MODELO", {"anio": 2023, "params": "~1T"}),
        ("ChatGPT", "MODELO", {"anio": 2022}),
        ("LLaMA", "MODELO", {"anio": 2023, "params": "65B"}),
        ("LLaMA 3", "MODELO", {"anio": 2024}),
        ("Mistral", "MODELO", {"anio": 2023, "params": "7B"}),
        ("Claude", "MODELO", {"anio": 2023}),
        ("Gemini", "MODELO", {"anio": 2023}),
        ("Google", "ORGANIZACION"), ("OpenAI", "ORGANIZACION"),
        ("Meta", "ORGANIZACION"), ("Anthropic", "ORGANIZACION"),
        ("Microsoft", "ORGANIZACION"), ("DeepMind", "ORGANIZACION"),
        ("Transformer", "TECNICA"), ("RLHF", "TECNICA"),
        ("LoRA", "TECNICA"), ("MLM", "TECNICA"), ("RAG", "TECNICA"),
    ]
    for ent in entidades:
        g.add_ent(*ent) if len(ent) == 3 else g.add_ent(ent[0], ent[1])

    relaciones = [
        ("BERT", "creado_por", "Google"), ("BERT", "usa", "Transformer"),
        ("BERT", "usa", "MLM"), ("GPT-2", "creado_por", "OpenAI"),
        ("GPT-2", "usa", "Transformer"), ("GPT-3", "creado_por", "OpenAI"),
        ("GPT-4", "creado_por", "OpenAI"), ("ChatGPT", "basado_en", "GPT-3"),
        ("ChatGPT", "usa", "RLHF"), ("LLaMA", "creado_por", "Meta"),
        ("LLaMA 3", "creado_por", "Meta"), ("LLaMA 3", "mejora", "LLaMA"),
        ("Mistral", "mejora", "LLaMA"), ("Claude", "creado_por", "Anthropic"),
        ("Claude", "usa", "RLHF"), ("Gemini", "creado_por", "Google"),
        ("Gemini", "creado_por", "DeepMind"), ("Microsoft", "invirtió_en", "OpenAI"),
        ("LoRA", "mejora", "RLHF"), ("RAG", "usa", "Transformer"),
    ]
    for o, r, d in relaciones:
        g.add_rel(o, r, d)
    return g

# --- Consultas tipo SPARQL simplificadas ---
class QueryEngine:
    """Motor de consultas sobre el grafo de conocimiento."""
    def __init__(self, grafo: GrafoMin):
        self.g = grafo

    def quien_creo(self, modelo: str) -> List[str]:
        """¿Quién creó X?"""
        return [a["target"] for a in self.g.aristas.get(modelo, [])
                if a["relacion"] == "creado_por"]

    def que_crea(self, organizacion: str) -> List[str]:
        """¿Qué modelos creó X?"""
        resultado = []
        for ent in self.g.entidades:
            for a in self.g.aristas.get(ent, []):
                if a["relacion"] == "creado_por" and a["target"] == organizacion:
                    resultado.append(ent)
        return resultado

    def que_usa(self, entidad: str) -> List[str]:
        """¿Qué técnicas usa X?"""
        return [a["target"] for a in self.g.aristas.get(entidad, [])
                if a["relacion"] == "usa"]

    def que_mejora(self, modelo: str) -> List[str]:
        """¿Qué modelos mejoran a X?"""
        mejores = []
        for ent in self.g.entidades:
            for a in self.g.aristas.get(ent, []):
                if a["relacion"] == "mejora" and a["target"] == modelo:
                    mejores.append(ent)
        return mejores

    def camino_entre(self, origen: str, destino: str,
                      max_len: int = 4) -> Optional[List[str]]:
        """BFS para encontrar el camino más corto."""
        if origen == destino: return [origen]
        visitados = {origen: [origen]}
        cola = deque([origen])
        while cola:
            nodo = cola.popleft()
            for a in self.g.aristas.get(nodo, []):
                v = a["target"]
                if v not in visitados:
                    camino = visitados[nodo] + [v]
                    if len(camino) > max_len: continue
                    if v == destino: return camino
                    visitados[v] = camino
                    cola.append(v)
        return None

    def context_para_query(self, pregunta: str,
                            profundidad: int = 2) -> str:
        """
        Genera contexto del grafo relevante para una pregunta.
        Estrategia: detectar entidades conocidas en la pregunta → BFS → contexto.
        """
        palabras = set(pregunta.split())
        entidades_en_pregunta = [e for e in self.g.entidades
                                   if e in pregunta or
                                   any(p.lower() == e.lower() for p in palabras)]

        lineas = []
        for ent in entidades_en_pregunta[:3]:
            vecinos = self.g.bfs(ent, max_depth=profundidad)
            for v, d in vecinos.items():
                if v == ent: continue
                # Buscar la relación
                for a in self.g.aristas.get(ent, []):
                    if a["target"] == v:
                        lineas.append(f"{ent} --[{a['relacion']}]--> {v}")
        return "\n".join(lineas) if lineas else "No se encontraron entidades en el grafo."

# --- Demo ---
g = construir_grafo_nlp()
qe = QueryEngine(g)

print("=== Consultas sobre el grafo ===\n")

preguntas_demo = [
    ("¿Quién creó BERT?", lambda: qe.quien_creo("BERT")),
    ("¿Qué modelos creó OpenAI?", lambda: qe.que_crea("OpenAI")),
    ("¿Qué técnicas usa Claude?", lambda: qe.que_usa("Claude")),
    ("¿Qué modelos mejoran a LLaMA?", lambda: qe.que_mejora("LLaMA")),
    ("Camino BERT → ChatGPT", lambda: qe.camino_entre("BERT", "ChatGPT")),
]

resultados = {}
for pregunta, fn in preguntas_demo:
    r = fn()
    print(f"{pregunta}: {r}")
    resultados[pregunta] = r

# Contexto para Graph RAG
pregunta_rag = "¿Qué relación tiene BERT con OpenAI?"
ctx = qe.context_para_query(pregunta_rag)
print(f"\nContexto de grafo para '{pregunta_rag}':\n{ctx}")

import json
with open("outputs/m15_consultas_grafo.json", "w", encoding="utf-8") as f:
    json.dump({k: (v if isinstance(v, list) else str(v))
               for k, v in resultados.items()}, f, ensure_ascii=False, indent=2)
print("\noutputs/m15_consultas_grafo.json guardado")
```

---

## 6. Checkpoint — Pruebas unitarias

```python
# checkpoint_m15.py
"""
Checkpoint M15: Graph RAG
Ejecutar con: python checkpoint_m15.py
"""
import numpy as np
import os
import json
from typing import List, Dict
from collections import defaultdict, deque

os.makedirs("outputs", exist_ok=True)

# ── Grafo mínimo ─────────────────────────────────────────────────────────────
class GrafoMin:
    def __init__(self):
        self.entidades = {}
        self.aristas = defaultdict(list)

    def add_ent(self, id_, tipo):
        self.entidades[id_] = {"tipo": tipo}

    def add_rel(self, o, r, d, fuente=""):
        if o not in self.entidades: self.add_ent(o, "?")
        if d not in self.entidades: self.add_ent(d, "?")
        self.aristas[o].append({"relacion": r, "target": d, "fuente": fuente})

    def bfs(self, inicio, max_depth=2):
        visitados = {inicio: 0}
        cola = deque([(inicio, 0)])
        while cola:
            nodo, prof = cola.popleft()
            if prof >= max_depth: continue
            for a in self.aristas.get(nodo, []):
                v = a["target"]
                if v not in visitados:
                    visitados[v] = prof + 1
                    cola.append((v, prof + 1))
        return visitados

    def camino_mas_corto(self, origen, destino, max_len=5):
        if origen == destino: return [origen]
        visitados = {origen: [origen]}
        cola = deque([origen])
        while cola:
            nodo = cola.popleft()
            for a in self.aristas.get(nodo, []):
                v = a["target"]
                if v not in visitados:
                    camino = visitados[nodo] + [v]
                    if len(camino) > max_len: continue
                    if v == destino: return camino
                    visitados[v] = camino
                    cola.append(v)
        return None

# ── Tests ─────────────────────────────────────────────────────────────────────
def test_agregar_entidad_y_arista():
    """T1: agregar entidades y aristas correctamente"""
    g = GrafoMin()
    g.add_ent("BERT", "MODELO")
    g.add_ent("Google", "ORGANIZACION")
    g.add_rel("BERT", "creado_por", "Google", "BERT fue hecho por Google.")

    assert "BERT" in g.entidades, "BERT debe estar en entidades"
    assert "Google" in g.entidades, "Google debe estar en entidades"
    assert len(g.aristas["BERT"]) == 1, "BERT debe tener 1 arista"
    assert g.aristas["BERT"][0]["relacion"] == "creado_por", "Relación incorrecta"
    assert g.aristas["BERT"][0]["target"] == "Google", "Target incorrecto"
    print(f"T1 PASS: entidades={len(g.entidades)}, aristas_bert={len(g.aristas['BERT'])}")

def test_bfs_profundidad_correcta():
    """T2: BFS respeta la profundidad máxima"""
    g = GrafoMin()
    # Cadena: A → B → C → D
    g.add_rel("A", "conecta", "B")
    g.add_rel("B", "conecta", "C")
    g.add_rel("C", "conecta", "D")

    vecinos_d1 = g.bfs("A", max_depth=1)
    vecinos_d2 = g.bfs("A", max_depth=2)
    vecinos_d3 = g.bfs("A", max_depth=3)

    assert "B" in vecinos_d1, "B debe estar en depth=1"
    assert "C" not in vecinos_d1, "C no debe estar en depth=1"
    assert "C" in vecinos_d2, "C debe estar en depth=2"
    assert "D" not in vecinos_d2, "D no debe estar en depth=2"
    assert "D" in vecinos_d3, "D debe estar en depth=3"
    print(f"T2 PASS: depth=1:{list(vecinos_d1.keys())}, depth=2:{list(vecinos_d2.keys())}")

def test_camino_mas_corto():
    """T3: encuentra el camino más corto entre dos entidades"""
    g = GrafoMin()
    g.add_rel("BERT", "creado_por", "Google")
    g.add_rel("Google", "compite_con", "OpenAI")
    g.add_rel("OpenAI", "creó", "ChatGPT")

    camino = g.camino_mas_corto("BERT", "ChatGPT")
    assert camino is not None, "Debe existir un camino"
    assert camino[0] == "BERT", f"Camino debe iniciar en BERT"
    assert camino[-1] == "ChatGPT", f"Camino debe terminar en ChatGPT"
    assert len(camino) == 4, f"Camino debe tener 4 nodos, got {len(camino)}"
    print(f"T3 PASS: camino={camino}")

def test_grafo_ner_extrae_entidades():
    """T4: el NER extrae entidades conocidas del texto"""
    import re
    ENTIDADES_CONOCIDAS = {"BERT", "Google", "OpenAI", "GPT-2", "Transformer"}

    texto = "BERT fue desarrollado por Google y usa el Transformer."

    encontradas = []
    for ent in ENTIDADES_CONOCIDAS:
        if re.search(r'\b' + re.escape(ent) + r'\b', texto, re.IGNORECASE):
            encontradas.append(ent)

    assert "BERT" in encontradas, "BERT debe encontrarse"
    assert "Google" in encontradas, "Google debe encontrarse"
    assert "Transformer" in encontradas, "Transformer debe encontrarse"
    assert "OpenAI" not in encontradas, "OpenAI no está en el texto"
    print(f"T4 PASS: encontradas={sorted(encontradas)}")

def test_context_grafo_enriquece_prompt():
    """T5: el contexto del grafo añade información no presente en chunks"""
    g = GrafoMin()
    g.add_rel("BERT", "creado_por", "Google",
              fuente="BERT fue desarrollado por Google en 2018.")
    g.add_rel("BERT", "usa", "Transformer",
              fuente="BERT usa la arquitectura Transformer.")

    chunk_texto = "BERT es un modelo de lenguaje preentrenado."  # sin mencionar Google

    # Generar contexto de grafo
    lineas_grafo = []
    for a in g.aristas.get("BERT", []):
        if a["fuente"]:
            lineas_grafo.append(a["fuente"])
    contexto_grafo = "\n".join(lineas_grafo)

    prompt_sin_grafo = f"Contexto: {chunk_texto}\nPregunta: ¿Quién creó BERT?"
    prompt_con_grafo = f"Contexto: {chunk_texto}\nGrafo: {contexto_grafo}\nPregunta: ¿Quién creó BERT?"

    assert "Google" in contexto_grafo, "El contexto del grafo debe mencionar Google"
    assert "Google" not in chunk_texto, "El chunk no menciona Google (control)"
    assert len(prompt_con_grafo) > len(prompt_sin_grafo), "Prompt con grafo debe ser más largo"
    print(f"T5 PASS: grafo añade 'Google' al contexto ({len(contexto_grafo)} chars extra)")

def test_subgrafo_contiene_nodos_correctos():
    """T6: el subgrafo extraído contiene todos los nodos alcanzables"""
    g = GrafoMin()
    # Estrella con centro OpenAI
    g.add_rel("OpenAI", "creó", "GPT-2")
    g.add_rel("OpenAI", "creó", "GPT-3")
    g.add_rel("OpenAI", "creó", "ChatGPT")
    g.add_rel("Microsoft", "invirtió_en", "OpenAI")
    g.add_rel("GPT-3", "precede_a", "ChatGPT")

    # BFS depth=1 desde OpenAI
    vecinos_d1 = g.bfs("OpenAI", max_depth=1)
    # Debe incluir GPT-2, GPT-3, ChatGPT (salientes)
    # NO debe incluir Microsoft (la arista va Microsoft→OpenAI, no OpenAI→Microsoft)
    assert "GPT-2" in vecinos_d1, "GPT-2 debe estar en profundidad 1"
    assert "GPT-3" in vecinos_d1, "GPT-3 debe estar en profundidad 1"
    assert "ChatGPT" in vecinos_d1, "ChatGPT debe estar en profundidad 1"
    assert "Microsoft" not in vecinos_d1, "Microsoft no debe estar (arista entrante)"

    # BFS depth=2 desde OpenAI incluye vecinos de GPT-3
    vecinos_d2 = g.bfs("OpenAI", max_depth=2)
    assert "ChatGPT" in vecinos_d2, "ChatGPT debe aparecer en depth=2 via GPT-3→ChatGPT"
    print(f"T6 PASS: subgrafo depth=1: {list(vecinos_d1.keys())}")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_agregar_entidad_y_arista,
        test_bfs_profundidad_correcta,
        test_camino_mas_corto,
        test_grafo_ner_extrae_entidades,
        test_context_grafo_enriquece_prompt,
        test_subgrafo_contiene_nodos_correctos,
    ]

    passed = 0
    for test in tests:
        try:
            test()
            passed += 1
        except AssertionError as e:
            print(f"FAIL: {test.__name__}: {e}")
        except Exception as e:
            print(f"ERROR: {test.__name__}: {e}")

    print(f"\n{'='*50}")
    print(f"Resultado: {passed}/{len(tests)} tests pasaron")
    if passed == len(tests):
        print("M15 COMPLETADO")
```

---

## Resumen

| Componente | Tecnología | Función |
|---|---|---|
| NER | spaCy / regex / LLM | Extraer entidades del texto |
| RE | LLM / patrones | Extraer relaciones (tripletas) |
| Grafo | NetworkX / propio | Almacenar entidades y relaciones |
| Búsqueda vectorial | FAISS / Chroma | Recuperar chunks similares |
| Graph traversal | BFS / DFS | Expandir contexto multi-hop |
| Construcción de prompt | Template | Combinar contexto vectorial + grafo |

**Pipeline Graph RAG completo:**
```
Documentos → NER + RE → Grafo de Conocimiento
                     ↕
Query → Búsqueda vectorial → Top-K chunks
     → Extraer entidades de chunks
     → BFS en grafo (depth=2)
     → Contexto vectorial + grafo
     → LLM → Respuesta con razonamiento multi-hop
```

**Casos de uso de Graph RAG:**
- Preguntas multi-hop: "¿A través de qué empresa X está relacionado con Y?"
- Resumir entidades: "Todo lo que sabemos sobre BERT"
- Detección de contradicciones en el grafo
- Completado de relaciones faltantes (link prediction)
