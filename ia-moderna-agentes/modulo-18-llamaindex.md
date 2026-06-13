# Módulo 18 — LlamaIndex — Indexación y Consulta

**Objetivo:** Entender la arquitectura de LlamaIndex, implementar pipelines de indexación y consulta sobre documentos, y construir un motor RAG estructurado con nodos, índices y query engines desde sus conceptos fundamentales.

**Herramientas:** `llama-index`, `llama-index-core`, `numpy`, `sentence-transformers`

**Prerequisito:** M17 (LangChain)

---

## 1. Arquitectura de LlamaIndex

```python
# llamaindex_arquitectura.py
import os

os.makedirs("outputs", exist_ok=True)

# LlamaIndex organiza el RAG en torno a estas abstracciones:
#
# Document → Node(s) → Index → QueryEngine → Response
#
# Document: unidad de datos raw (texto, PDF, web, etc.)
# Node: chunk de Document con metadata (texto, doc_id, inicio, fin)
# Index: estructura de búsqueda sobre Nodes (VectorStore, Tree, Keyword, List)
# QueryEngine: combina retrieval + síntesis de respuesta
# Response: respuesta con source_nodes y score

ABSTRACCIONES = {
    "Document": "Datos raw: texto, PDF, JSON, código, web",
    "Node (TextNode)": "Chunk con texto + metadata + embeddings",
    "NodeParser": "Divide Document en Nodes (SentenceSplitter, etc.)",
    "Index": "Estructura de búsqueda: VectorStoreIndex, TreeIndex, KeywordTableIndex",
    "Retriever": "Extrae Nodes relevantes del Index dado una query",
    "ResponseSynthesizer": "Sintetiza respuesta final con los Nodes recuperados",
    "QueryEngine": "Combina Retriever + ResponseSynthesizer",
    "ChatEngine": "QueryEngine con memoria de conversación",
}

print("=== Abstracciones LlamaIndex ===")
for k, v in ABSTRACCIONES.items():
    print(f"  {k}: {v}")

MODOS_QUERY = {
    "compact": "Concatena todos los Nodes en un único prompt (default)",
    "refine": "Genera respuesta con Node 1, luego refina con Node 2, ...",
    "tree_summarize": "Síntesis jerárquica en árbol (mejor para documentos largos)",
    "no_text": "Solo retorna source_nodes sin generar respuesta",
    "accumulate": "Responde por separado para cada Node, luego acumula",
}

print("\n=== Modos de Query ===")
for k, v in MODOS_QUERY.items():
    print(f"  {k}: {v}")

with open("outputs/m18_intro.txt", "w") as f:
    f.write("M18: LlamaIndex\n")
print("\noutputs/m18_intro.txt guardado")
```

---

## 2. Document y NodeParser

```python
# llamaindex_nodes.py
import os
import re
import hashlib
from typing import List, Dict, Optional, Any
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

@dataclass
class Document:
    """
    Equivalente a llama_index.core.Document.
    Unidad básica de datos de entrada.
    """
    text: str
    doc_id: str = ""
    metadata: Dict = field(default_factory=dict)

    def __post_init__(self):
        if not self.doc_id:
            # Generar ID único basado en el contenido
            self.doc_id = hashlib.md5(self.text[:200].encode()).hexdigest()[:12]

@dataclass
class TextNode:
    """
    Equivalente a llama_index.core.schema.TextNode.
    Chunk de texto con metadata y embedding.
    """
    text: str
    node_id: str = ""
    doc_id: str = ""
    start_char_idx: int = 0
    end_char_idx: int = 0
    metadata: Dict = field(default_factory=dict)
    embedding: Optional[List[float]] = None
    relationships: Dict = field(default_factory=dict)  # prev/next/parent

    def __post_init__(self):
        if not self.node_id:
            self.node_id = hashlib.md5(
                (self.doc_id + str(self.start_char_idx)).encode()
            ).hexdigest()[:12]

    def get_content(self) -> str:
        return self.text

    def get_metadata_str(self) -> str:
        return "\n".join([f"{k}: {v}" for k, v in self.metadata.items()])

class SentenceSplitter:
    """
    Equivalente a llama_index.core.node_parser.SentenceSplitter.
    Divide documentos en nodos respetando límites de oraciones.
    """
    def __init__(self, chunk_size: int = 512, chunk_overlap: int = 50,
                 paragraph_separator: str = "\n\n"):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.paragraph_separator = paragraph_separator

    def _split_into_sentences(self, text: str) -> List[str]:
        """Divide texto en oraciones usando puntuación."""
        oraciones = re.split(r'(?<=[.!?])\s+(?=[A-ZÁÉÍÓÚ¿]|\d)', text)
        return [o.strip() for o in oraciones if o.strip()]

    def get_nodes_from_documents(self, documents: List[Document]) -> List[TextNode]:
        """
        Divide cada Document en TextNodes.
        Mantiene relaciones prev/next entre nodos del mismo documento.
        """
        all_nodes = []

        for doc in documents:
            oraciones = self._split_into_sentences(doc.text)
            nodos_doc = []
            acumulado = ""
            inicio_char = 0

            for oracion in oraciones:
                candidato = (acumulado + " " + oracion).strip() if acumulado else oracion

                if len(candidato) > self.chunk_size and acumulado:
                    # Guardar chunk actual
                    nodo = TextNode(
                        text=acumulado,
                        doc_id=doc.doc_id,
                        start_char_idx=inicio_char,
                        end_char_idx=inicio_char + len(acumulado),
                        metadata={**doc.metadata, "doc_id": doc.doc_id}
                    )
                    nodos_doc.append(nodo)
                    # Overlap: empezar con el final del chunk anterior
                    palabras_prev = acumulado.split()
                    n_overlap = min(len(palabras_prev),
                                    self.chunk_overlap // 6)  # ~6 chars por palabra
                    acumulado = " ".join(palabras_prev[-n_overlap:]) + " " + oracion
                    inicio_char = inicio_char + len(acumulado)
                else:
                    acumulado = candidato

            # Último chunk
            if acumulado.strip():
                nodo = TextNode(
                    text=acumulado.strip(),
                    doc_id=doc.doc_id,
                    start_char_idx=inicio_char,
                    end_char_idx=inicio_char + len(acumulado),
                    metadata={**doc.metadata, "doc_id": doc.doc_id}
                )
                nodos_doc.append(nodo)

            # Añadir relaciones prev/next
            for i, nodo in enumerate(nodos_doc):
                if i > 0:
                    nodo.relationships["PREVIOUS"] = nodos_doc[i-1].node_id
                if i < len(nodos_doc) - 1:
                    nodo.relationships["NEXT"] = nodos_doc[i+1].node_id

            all_nodes.extend(nodos_doc)

        return all_nodes

class SimpleNodeParser:
    """Parser más simple: división por párrafos."""
    def __init__(self, chunk_size: int = 300):
        self.chunk_size = chunk_size

    def get_nodes_from_documents(self, documents: List[Document]) -> List[TextNode]:
        nodes = []
        for doc in documents:
            parrafos = [p.strip() for p in doc.text.split("\n\n") if p.strip()]
            for i, parrafo in enumerate(parrafos):
                nodes.append(TextNode(
                    text=parrafo,
                    doc_id=doc.doc_id,
                    start_char_idx=doc.text.find(parrafo),
                    end_char_idx=doc.text.find(parrafo) + len(parrafo),
                    metadata={**doc.metadata, "chunk_idx": i}
                ))
        return nodes

# --- Demo ---
documentos = [
    Document(
        text="""Los transformers son la arquitectura dominante en NLP.
Fueron introducidos en 2017 con el paper "Attention is All You Need".
El mecanismo de self-attention permite procesar secuencias en paralelo.

BERT usa encoders bidireccionales y fue preentrenado con MLM.
GPT usa decoders causales para generación de texto.
LLaMA es un modelo open-source de Meta AI.""",
        metadata={"fuente": "guia_nlp.txt", "autor": "AI Team"}
    ),
    Document(
        text="""RAG (Retrieval-Augmented Generation) combina búsqueda con generación.
El pipeline incluye: ingesta, chunking, embedding, indexado y query.
LlamaIndex es un framework especializado en construir pipelines RAG.

La búsqueda semántica con embeddings mejora la recuperación de contexto.
El re-ranking con cross-encoders refina los resultados de recuperación.""",
        metadata={"fuente": "guia_rag.txt"}
    )
]

# Parsear con SentenceSplitter
splitter = SentenceSplitter(chunk_size=150, chunk_overlap=30)
nodos = splitter.get_nodes_from_documents(documentos)

print(f"=== NodeParser Demo ===")
print(f"Documentos: {len(documentos)}")
print(f"Nodos generados: {len(nodos)}")

for i, nodo in enumerate(nodos):
    relaciones = list(nodo.relationships.keys())
    print(f"\nNodo {i}: {nodo.text[:60]}...")
    print(f"  doc_id={nodo.doc_id} | chars={nodo.start_char_idx}-{nodo.end_char_idx}")
    print(f"  relaciones={relaciones}")

import json
with open("outputs/m18_nodos_demo.json", "w", encoding="utf-8") as f:
    json.dump([{
        "node_id": n.node_id,
        "text": n.text[:100],
        "doc_id": n.doc_id,
        "n_relaciones": len(n.relationships)
    } for n in nodos], f, ensure_ascii=False, indent=2)
print("\noutputs/m18_nodos_demo.json guardado")
```

---

## 3. VectorStoreIndex y QueryEngine

```python
# llamaindex_index.py
import os
import json
import numpy as np
from typing import List, Dict, Optional, Any
from dataclasses import dataclass, field
import hashlib
import re

os.makedirs("outputs", exist_ok=True)

# Reutilizar TextNode y Document del apartado anterior
@dataclass
class TextNode:
    text: str
    node_id: str = ""
    doc_id: str = ""
    metadata: Dict = field(default_factory=dict)
    embedding: Optional[np.ndarray] = None

    def __post_init__(self):
        if not self.node_id:
            self.node_id = hashlib.md5(self.text[:100].encode()).hexdigest()[:10]

class EmbedderSimulado:
    def __init__(self, dim=64):
        self.dim = dim

    def encode(self, textos: List[str]) -> np.ndarray:
        vecs = []
        for t in textos:
            seed = sum(ord(c)*(i+1) for i,c in enumerate(t[:200])) % (2**31)
            rng = np.random.RandomState(seed)
            v = rng.randn(self.dim).astype(np.float32)
            vecs.append(v / (np.linalg.norm(v) + 1e-9))
        return np.stack(vecs)

class VectorStoreIndex:
    """
    Equivalente a llama_index.core.VectorStoreIndex.
    Indexa TextNodes con embeddings para búsqueda semántica.

    Operaciones principales:
    - from_documents(): construye índice desde Documents
    - as_retriever(): crea un Retriever desde el índice
    - as_query_engine(): crea un QueryEngine completo
    - insert(): añade nuevos nodos al índice
    - delete(): elimina nodos por doc_id
    """
    def __init__(self, nodes: List[TextNode], embedder):
        self.embedder = embedder
        self.nodes: Dict[str, TextNode] = {}
        self.vectors: Dict[str, np.ndarray] = {}

        # Calcular embeddings para todos los nodos
        textos = [n.text for n in nodes]
        embs = embedder.encode(textos) if textos else np.array([])

        for node, emb in zip(nodes, embs):
            node.embedding = emb
            self.nodes[node.node_id] = node
            self.vectors[node.node_id] = emb

    @classmethod
    def from_documents(cls, documents, node_parser=None, embedder=None):
        """Construye índice desde una lista de Documents."""
        if node_parser is None:
            # SimpleNodeParser por defecto
            nodes = []
            for doc in documents:
                parrafos = [p.strip() for p in doc.text.split("\n") if p.strip()]
                for i, p in enumerate(parrafos):
                    nodes.append(TextNode(
                        text=p, doc_id=doc.doc_id,
                        metadata={**doc.metadata, "chunk_idx": i}
                    ))
        else:
            nodes = node_parser.get_nodes_from_documents(documents)
        return cls(nodes, embedder or EmbedderSimulado())

    def insert(self, node: TextNode):
        """Añade un nodo nuevo al índice."""
        emb = self.embedder.encode([node.text])[0]
        node.embedding = emb
        self.nodes[node.node_id] = node
        self.vectors[node.node_id] = emb

    def delete(self, doc_id: str):
        """Elimina todos los nodos de un documento."""
        to_del = [nid for nid, n in self.nodes.items() if n.doc_id == doc_id]
        for nid in to_del:
            del self.nodes[nid]
            del self.vectors[nid]
        return len(to_del)

    def as_retriever(self, similarity_top_k: int = 3) -> 'VectorIndexRetriever':
        return VectorIndexRetriever(self, similarity_top_k)

    def as_query_engine(self, similarity_top_k: int = 3,
                         response_mode: str = "compact") -> 'RetrieverQueryEngine':
        retriever = self.as_retriever(similarity_top_k)
        return RetrieverQueryEngine(retriever, response_mode=response_mode)

    def stats(self) -> Dict:
        return {"n_nodes": len(self.nodes),
                "n_docs": len(set(n.doc_id for n in self.nodes.values()))}

class NodeWithScore:
    """Nodo con score de relevancia."""
    def __init__(self, node: TextNode, score: float):
        self.node = node
        self.score = score

    def __repr__(self):
        return f"NodeWithScore(score={self.score:.4f}, text='{self.node.text[:40]}...')"

class VectorIndexRetriever:
    """
    Recuperador basado en similitud coseno.
    Equivalente a VectorIndexRetriever de LlamaIndex.
    """
    def __init__(self, index: VectorStoreIndex, similarity_top_k: int = 3):
        self.index = index
        self.similarity_top_k = similarity_top_k

    def retrieve(self, query: str) -> List[NodeWithScore]:
        """Recupera los nodos más similares a la query."""
        q_emb = self.index.embedder.encode([query])[0]
        q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)

        ids = list(self.index.vectors.keys())
        if not ids:
            return []

        vecs = np.stack([self.index.vectors[id_] for id_ in ids])
        sims = vecs @ q_norm
        top_k = np.argsort(sims)[::-1][:self.similarity_top_k]

        return [
            NodeWithScore(
                node=self.index.nodes[ids[i]],
                score=float(sims[i])
            )
            for i in top_k
        ]

class Response:
    """Equivalente a llama_index.core.Response."""
    def __init__(self, response: str, source_nodes: List[NodeWithScore] = None):
        self.response = response
        self.source_nodes = source_nodes or []

    def __str__(self):
        return self.response

    def get_formatted_sources(self) -> str:
        return "\n".join([
            f"[{i+1}] (score={n.score:.4f}): {n.node.text[:80]}..."
            for i, n in enumerate(self.source_nodes)
        ])

class ResponseSynthesizer:
    """Sintetiza respuesta final a partir de los nodos recuperados."""
    def __init__(self, llm_fn, mode: str = "compact"):
        self.llm_fn = llm_fn
        self.mode = mode

    def synthesize(self, query: str, nodes: List[NodeWithScore]) -> Response:
        if not nodes:
            return Response("No encontré información relevante.", [])

        if self.mode == "compact":
            # Concatenar todos los nodos en un único prompt
            contexto = "\n\n".join([f"[{i+1}] {n.node.text}"
                                     for i, n in enumerate(nodes)])
            prompt = f"Contexto:\n{contexto}\n\nPregunta: {query}\nRespuesta:"
            respuesta = self.llm_fn(prompt)

        elif self.mode == "refine":
            # Generar respuesta inicial con el primer nodo, luego refinar
            respuesta = self.llm_fn(
                f"Contexto: {nodes[0].node.text}\nPregunta: {query}\nRespuesta:"
            )
            for nodo in nodes[1:]:
                respuesta = self.llm_fn(
                    f"Respuesta existente: {respuesta}\n"
                    f"Contexto adicional: {nodo.node.text}\n"
                    f"Refina la respuesta si el nuevo contexto es relevante:"
                )

        elif self.mode == "tree_summarize":
            # Síntesis jerárquica
            resps_por_nodo = [
                self.llm_fn(f"Contexto: {n.node.text}\nPregunta: {query}\nRespuesta breve:")
                for n in nodes
            ]
            todas_resps = "\n".join(resps_por_nodo)
            respuesta = self.llm_fn(
                f"Sintetiza en una respuesta final:\n{todas_resps}\nRespuesta:"
            )
        else:
            respuesta = self.llm_fn(
                f"Contexto: {nodes[0].node.text}\nPregunta: {query}\nRespuesta:"
            )

        return Response(respuesta, nodes)

class RetrieverQueryEngine:
    """
    Equivalente a RetrieverQueryEngine de LlamaIndex.
    Combina retriever + synthesizer.
    """
    def __init__(self, retriever: VectorIndexRetriever,
                 llm_fn=None, response_mode: str = "compact"):
        self.retriever = retriever
        self.llm_fn = llm_fn or (lambda p: f"[LLM] {p[-60:]}")
        self.synthesizer = ResponseSynthesizer(self.llm_fn, mode=response_mode)

    def query(self, question: str) -> Response:
        nodes = self.retriever.retrieve(question)
        return self.synthesizer.synthesize(question, nodes)

# --- Demo ---
from dataclasses import dataclass as _dc

@_dc
class Document:
    text: str
    doc_id: str = ""
    metadata: Dict = field(default_factory=dict)
    def __post_init__(self):
        if not self.doc_id:
            self.doc_id = hashlib.md5(self.text[:100].encode()).hexdigest()[:10]

documentos = [
    Document("El Transformer usa atención multi-cabeza.\nBERT usa encoders bidireccionales.\nGPT usa decoders causales.",
             metadata={"tema": "LLMs"}),
    Document("RAG combina recuperación vectorial con generación LLM.\nLlamaIndex es un framework para RAG estructurado.\nLa búsqueda semántica usa embeddings.",
             metadata={"tema": "RAG"}),
    Document("LoRA añade matrices de bajo rango para fine-tuning eficiente.\nQLoRA combina cuantización con LoRA.\nPEFT engloba técnicas eficientes de fine-tuning.",
             metadata={"tema": "PEFT"}),
]

embedder = EmbedderSimulado(dim=64)
indice = VectorStoreIndex.from_documents(documentos, embedder=embedder)
print(f"Índice: {indice.stats()}")

# Query engine
def llm_compact(prompt):
    lineas = prompt.split("\n")
    for linea in lineas:
        if "Transformer" in linea or "BERT" in linea:
            return "El Transformer y BERT son arquitecturas de deep learning para NLP."
    return f"[LLM Compacto] {prompt[-80:]}"

qe = indice.as_query_engine(similarity_top_k=2, response_mode="compact")
qe.llm_fn = llm_compact
qe.synthesizer = ResponseSynthesizer(llm_compact, mode="compact")

preguntas = [
    "¿Qué es BERT y qué arquitectura usa?",
    "¿Cómo funciona LoRA?",
    "¿Para qué sirve LlamaIndex?",
]

print("\n=== QueryEngine Demo ===")
resultados = []
for pregunta in preguntas:
    resp = qe.query(pregunta)
    print(f"\nQ: {pregunta}")
    print(f"A: {resp.response}")
    print(f"Fuentes: {len(resp.source_nodes)} nodos")
    resultados.append({
        "pregunta": pregunta,
        "n_fuentes": len(resp.source_nodes),
        "scores": [n.score for n in resp.source_nodes]
    })

# Insert y delete
nuevo_nodo = TextNode(text="Claude 3 es el modelo más capaz de Anthropic.", doc_id="nuevo_doc")
indice.insert(nuevo_nodo)
print(f"\nTras insert: {indice.stats()}")
n_eliminados = indice.delete("nuevo_doc")
print(f"Eliminados {n_eliminados} nodos. Tras delete: {indice.stats()}")

with open("outputs/m18_index_demo.json", "w", encoding="utf-8") as f:
    json.dump(resultados, f, ensure_ascii=False, indent=2)
print("\noutputs/m18_index_demo.json guardado")
```

---

## 4. Tipos de índices: Tree y Keyword

```python
# llamaindex_tipos_indices.py
import os
import json
import numpy as np
from typing import List, Dict
from collections import defaultdict

os.makedirs("outputs", exist_ok=True)

# --- KeywordTableIndex ---
class KeywordTableIndex:
    """
    Índice basado en palabras clave.
    Mapea keyword → lista de nodos que la contienen.
    Ventaja: muy rápido para búsquedas léxicas exactas.
    Desventaja: no captura similitud semántica.
    """
    def __init__(self, nodes: List):
        self.nodes: Dict[str, any] = {n.node_id: n for n in nodes}
        self.keyword_map: Dict[str, List[str]] = defaultdict(list)  # keyword → [node_ids]
        self._stop_words = {"el", "la", "los", "las", "de", "del", "en", "un",
                             "una", "y", "o", "a", "con", "por", "que", "se"}
        self._build_index()

    def _extraer_keywords(self, texto: str) -> List[str]:
        """Extrae palabras relevantes (excluye stopwords)."""
        import re
        palabras = re.findall(r'\b[a-záéíóúA-ZÁÉÍÓÚ]{4,}\b', texto.lower())
        return [p for p in palabras if p not in self._stop_words]

    def _build_index(self):
        for node in self.nodes.values():
            keywords = self._extraer_keywords(node.text)
            for kw in set(keywords):
                self.keyword_map[kw].append(node.node_id)

    def retrieve(self, query: str, top_k: int = 3) -> List:
        keywords_query = self._extraer_keywords(query)
        # Contar cuántas keywords de la query aparecen en cada nodo
        scores: Dict[str, float] = defaultdict(float)
        for kw in keywords_query:
            for node_id in self.keyword_map.get(kw, []):
                scores[node_id] += 1.0
        # Normalizar por número de keywords de la query
        n_kws = max(len(keywords_query), 1)
        for node_id in scores:
            scores[node_id] /= n_kws

        top_k_nodes = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
        return [{"node_id": nid, "score": score,
                 "texto": self.nodes[nid].text[:80]} for nid, score in top_k_nodes]

# --- TreeIndex (simplificado) ---
class TreeIndex:
    """
    Índice en árbol: organiza nodos jerárquicamente.
    La recuperación es por traversal del árbol desde la raíz.

    Estructura:
    - Hoja: TextNode original
    - Nodo interno: resumen de sus hijos
    - Raíz: resumen del documento completo

    Ventaja: maneja bien documentos muy largos.
    Desventaja: requiere múltiples llamadas al LLM para construir.
    """
    def __init__(self, nodes: List, llm_fn=None, num_children: int = 2):
        self.leaf_nodes = nodes
        self.llm_fn = llm_fn or (lambda x: f"[Resumen] {x[:60]}")
        self.num_children = num_children
        self.tree = self._build_tree(nodes)

    def _summarize(self, textos: List[str]) -> str:
        combined = "\n".join(textos[:3])
        return self.llm_fn(f"Resume brevemente:\n{combined}")

    def _build_tree(self, nodos: List) -> Dict:
        """Construye árbol bottom-up agrupando nodos."""
        if len(nodos) <= self.num_children:
            resumen = self._summarize([n.text for n in nodos])
            return {"type": "root", "summary": resumen, "children": nodos}

        # Agrupar en chunks de num_children
        grupos = [nodos[i:i + self.num_children]
                  for i in range(0, len(nodos), self.num_children)]
        nodos_internos = []
        for grupo in grupos:
            resumen = self._summarize([n.text for n in grupo])
            nodos_internos.append({
                "type": "internal",
                "summary": resumen,
                "children": grupo
            })

        # Recursivamente hasta la raíz
        return self._build_tree_from_internos(nodos_internos)

    def _build_tree_from_internos(self, internos: List[Dict]) -> Dict:
        if len(internos) == 1:
            return internos[0]
        resumen_raiz = self._summarize([n["summary"] for n in internos])
        return {"type": "root", "summary": resumen_raiz, "children": internos}

    def retrieve_top_down(self, query: str) -> List:
        """Traversal top-down: elige el hijo más relevante en cada nivel."""
        results = []
        cola = [self.tree]
        while cola:
            nodo = cola.pop(0)
            if isinstance(nodo, dict):
                # Es un nodo interno: añadir hijos a la cola
                cola.extend(nodo.get("children", []))
            else:
                # Es un nodo hoja
                results.append(nodo)
        return results[:3]

# --- Demo ---
import hashlib
from dataclasses import dataclass, field

@dataclass
class SimpleNode:
    text: str
    node_id: str = ""
    def __post_init__(self):
        if not self.node_id:
            self.node_id = hashlib.md5(self.text[:50].encode()).hexdigest()[:8]

nodos = [
    SimpleNode("BERT usa encoders bidireccionales con MLM para preentrenamiento."),
    SimpleNode("GPT usa decoders causales para generación autoregresiva de texto."),
    SimpleNode("LLaMA de Meta AI fue publicado como modelo open-source en 2023."),
    SimpleNode("RAG combina recuperación vectorial con generación de texto."),
    SimpleNode("LoRA añade matrices de bajo rango para fine-tuning eficiente."),
    SimpleNode("LlamaIndex facilita la construcción de pipelines RAG estructurados."),
]

print("=== KeywordTableIndex ===")
kw_idx = KeywordTableIndex(nodos)
print(f"Keywords indexadas: {len(kw_idx.keyword_map)}")
query = "preentrenamiento de modelos de lenguaje"
resultados_kw = kw_idx.retrieve(query, top_k=3)
print(f"\nQuery: '{query}'")
for r in resultados_kw:
    print(f"  score={r['score']:.3f}: {r['texto']}")

print("\n=== TreeIndex ===")
tree_idx = TreeIndex(nodos, num_children=3)
print(f"Árbol construido. Resumen raíz: {tree_idx.tree['summary'][:80]}")
nodos_recuperados = tree_idx.retrieve_top_down(query)
print(f"Nodos recuperados: {len(nodos_recuperados)}")

resultados_indices = {
    "keyword_n_keywords": len(kw_idx.keyword_map),
    "keyword_top3_scores": [r["score"] for r in resultados_kw],
    "tree_root_summary": tree_idx.tree["summary"][:80],
}
with open("outputs/m18_indices_demo.json", "w", encoding="utf-8") as f:
    json.dump(resultados_indices, f, ensure_ascii=False, indent=2)
print("\noutputs/m18_indices_demo.json guardado")
```

---

## 5. Sub-question Query Engine

```python
# llamaindex_subquery.py
import os
import json
import numpy as np
from typing import List, Dict
import re

os.makedirs("outputs", exist_ok=True)

class SubQuestionQueryEngine:
    """
    Sub-question Query Engine de LlamaIndex.

    Para queries complejas:
    1. Descompone la query en sub-preguntas
    2. Enruta cada sub-pregunta al query engine más adecuado
    3. Sintetiza todas las sub-respuestas en una respuesta final

    Ideal para: preguntas multi-documento, comparaciones, análisis
    """
    def __init__(self, query_engines: Dict[str, any], llm_fn=None):
        self.query_engines = query_engines
        self.llm_fn = llm_fn or (lambda p: f"[LLM] {p[-60:]}")

    def _generar_subqueries(self, query: str) -> List[Dict]:
        """
        Genera sub-preguntas y las asigna a query engines.
        En producción: usar LLM con structured output.
        """
        # Simulación: reglas basadas en palabras clave
        subqueries = []
        engines_disponibles = list(self.query_engines.keys())

        # Detectar si es una comparación
        if any(w in query.lower() for w in ["diferencia", "compara", "versus", "mejor"]):
            for engine_name in engines_disponibles[:2]:
                subqueries.append({
                    "query": f"¿Qué características tiene {engine_name}?",
                    "engine": engine_name
                })
        elif any(w in query.lower() for w in ["resume", "explica", "qué es"]):
            # Todos los engines para un resumen completo
            for engine_name in engines_disponibles:
                subqueries.append({
                    "query": query,
                    "engine": engine_name
                })
        else:
            # Query directa al engine más relevante
            for engine_name in engines_disponibles[:1]:
                subqueries.append({
                    "query": query,
                    "engine": engine_name
                })
        return subqueries

    def _sintetizar(self, query: str, sub_respuestas: List[Dict]) -> str:
        """Sintetiza sub-respuestas en respuesta final."""
        partes = [f"Fuente '{r['engine']}': {r['respuesta']}"
                  for r in sub_respuestas]
        combined = "\n\n".join(partes)
        prompt = f"Sintetiza estas respuestas en una respuesta final para: '{query}'\n\n{combined}\n\nRespuesta:"
        return self.llm_fn(prompt)

    def query(self, query: str) -> Dict:
        """Ejecuta el pipeline de sub-queries."""
        subqueries = self._generar_subqueries(query)

        sub_respuestas = []
        for sq in subqueries:
            engine = self.query_engines.get(sq["engine"])
            if engine:
                resp = engine(sq["query"])
                sub_respuestas.append({
                    "engine": sq["engine"],
                    "query": sq["query"],
                    "respuesta": resp[:100]
                })

        respuesta_final = self._sintetizar(query, sub_respuestas)

        return {
            "query": query,
            "sub_queries": subqueries,
            "sub_respuestas": sub_respuestas,
            "respuesta_final": respuesta_final
        }

# --- Demo ---
# Simular múltiples "índices" (query engines) por documento
def engine_llms(q):
    if "bert" in q.lower(): return "BERT usa MLM bidireccional, es de Google (2018)."
    if "gpt" in q.lower(): return "GPT usa decoder causal, es de OpenAI."
    return f"[LLMs engine] {q[:40]}"

def engine_rag(q):
    if "rag" in q.lower(): return "RAG combina retrieval vectorial con generación LLM."
    if "llama" in q.lower(): return "LlamaIndex es un framework RAG con nodos y query engines."
    return f"[RAG engine] {q[:40]}"

def llm_sintetizador(prompt):
    return "[Síntesis] " + prompt.split("Respuesta:")[-1].strip()[:80]

sqe = SubQuestionQueryEngine(
    query_engines={"llms_index": engine_llms, "rag_index": engine_rag},
    llm_fn=llm_sintetizador
)

preguntas = [
    "¿Qué diferencia hay entre BERT y GPT?",
    "¿Qué es LlamaIndex y qué es RAG?",
]

print("=== SubQuestion QueryEngine Demo ===\n")
resultados_sq = []
for pregunta in preguntas:
    resultado = sqe.query(pregunta)
    print(f"Query: {pregunta}")
    print(f"Sub-queries ({len(resultado['sub_queries'])}):")
    for sq in resultado["sub_queries"]:
        print(f"  [{sq['engine']}]: {sq['query'][:50]}")
    print(f"Respuesta final: {resultado['respuesta_final'][:80]}")
    print()
    resultados_sq.append({
        "query": pregunta,
        "n_subqueries": len(resultado["sub_queries"])
    })

with open("outputs/m18_subquery_demo.json", "w", encoding="utf-8") as f:
    json.dump(resultados_sq, f, ensure_ascii=False, indent=2)
print("outputs/m18_subquery_demo.json guardado")
```

---

## 6. Checkpoint — Pruebas unitarias

```python
# checkpoint_m18.py
"""
Checkpoint M18: LlamaIndex — Indexación y Consulta
Ejecutar con: python checkpoint_m18.py
"""
import numpy as np
import json
import os
import hashlib
import re
from typing import List, Dict, Optional
from dataclasses import dataclass, field
from collections import defaultdict

os.makedirs("outputs", exist_ok=True)

@dataclass
class SimpleNode:
    text: str
    node_id: str = ""
    doc_id: str = ""
    metadata: Dict = field(default_factory=dict)
    def __post_init__(self):
        if not self.node_id:
            self.node_id = hashlib.md5(self.text[:50].encode()).hexdigest()[:8]

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

# Tests
def test_nodeparser_genera_nodos():
    """T1: el NodeParser divide documentos en nodos"""
    @dataclass
    class Doc:
        text: str
        doc_id: str = "d1"
        metadata: dict = field(default_factory=dict)

    texto = "Primer parrafo sobre transformers.\n\nSegundo parrafo sobre BERT.\n\nTercer parrafo sobre GPT."
    doc = Doc(text=texto)

    # SimpleNodeParser por parrafos
    parrafos = [p.strip() for p in texto.split("\n\n") if p.strip()]
    nodos = [SimpleNode(text=p, doc_id="d1") for p in parrafos]

    assert len(nodos) == 3, f"Deben ser 3 nodos, got {len(nodos)}"
    assert nodos[0].text.startswith("Primer"), "El primer nodo debe ser el primer párrafo"
    assert nodos[2].text.startswith("Tercer"), "El tercer nodo debe ser el tercer párrafo"
    print(f"T1 PASS: NodeParser genera {len(nodos)} nodos")

def test_vector_index_retrieval():
    """T2: VectorStoreIndex recupera el nodo más similar"""
    np.random.seed(42)
    embedder = EmbedderSimulado(dim=64)

    nodos = [
        SimpleNode("BERT usa encoders bidireccionales.", doc_id="d1"),
        SimpleNode("GPT usa decoder causal para generacion.", doc_id="d2"),
        SimpleNode("LoRA anade matrices de bajo rango.", doc_id="d3"),
        SimpleNode("RAG combina recuperacion con generacion LLM.", doc_id="d4"),
    ]

    # Construir índice
    textos = [n.text for n in nodos]
    embs = embedder.encode(textos)
    index = {n.node_id: (n, emb) for n, emb in zip(nodos, embs)}

    # Retrieval
    query = "BERT usa encoders bidireccionales."
    q_emb = embedder.encode([query])[0]
    q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)

    scores = {}
    for nid, (node, emb) in index.items():
        emb_norm = emb / (np.linalg.norm(emb) + 1e-9)
        scores[nid] = float(emb_norm @ q_norm)

    top_nid = max(scores, key=scores.get)
    top_node = index[top_nid][0]

    assert top_node.text == "BERT usa encoders bidireccionales.", \
        f"El nodo mas similar a si mismo debe recuperarse primero, got: {top_node.text}"
    assert scores[top_nid] > 0.99, f"Score esperado > 0.99, got {scores[top_nid]}"
    print(f"T2 PASS: nodo mas similar recuperado (score={scores[top_nid]:.4f})")

def test_keyword_index_score():
    """T3: KeywordTableIndex da mayor score a nodos con más keywords de la query"""
    nodos = [
        SimpleNode("BERT transformer bidireccional MLM Google preentrenamiento"),
        SimpleNode("GPT transformer decoder causal generacion autoregresiva"),
        SimpleNode("futbol deporte popular mundial jugadores"),
    ]

    stop_words = {"el","la","los","las","de","del","en","un","una","y","o","a","con"}
    keyword_map = defaultdict(list)
    for nodo in nodos:
        palabras = re.findall(r'\b[a-z]{4,}\b', nodo.text.lower())
        for p in set(palabras):
            if p not in stop_words:
                keyword_map[p].append(nodo.node_id)

    # Query
    query = "transformer bidireccional preentrenamiento"
    kws_q = [p for p in re.findall(r'\b[a-z]{4,}\b', query.lower())
              if p not in stop_words]

    scores = defaultdict(float)
    for kw in kws_q:
        for nid in keyword_map.get(kw, []):
            scores[nid] += 1.0 / len(kws_q)

    top_nid = max(scores, key=scores.get) if scores else None
    # El nodo sobre BERT (más keywords en común) debe ser el top
    nodo_bert = nodos[0].node_id
    assert top_nid == nodo_bert, \
        f"BERT debe ser el top result, got: {top_nid}"
    print(f"T3 PASS: keyword index da mayor score a BERT ({scores[top_nid]:.3f})")

def test_response_tiene_source_nodes():
    """T4: la respuesta incluye source_nodes con scores"""
    class MockResponse:
        def __init__(self, response, source_nodes):
            self.response = response
            self.source_nodes = source_nodes

    class MockNodeWithScore:
        def __init__(self, text, score):
            self.node = type("Node", (), {"text": text})()
            self.score = score

    sources = [
        MockNodeWithScore("Texto relevante 1", 0.95),
        MockNodeWithScore("Texto relevante 2", 0.87),
    ]
    resp = MockResponse("Respuesta generada", sources)

    assert len(resp.source_nodes) == 2, "Debe tener 2 source nodes"
    assert resp.source_nodes[0].score > resp.source_nodes[1].score, \
        "Nodos deben estar ordenados por score descendente"
    assert resp.response == "Respuesta generada", "Respuesta incorrecta"
    print(f"T4 PASS: respuesta con {len(resp.source_nodes)} source nodes")

def test_insert_delete_index():
    """T5: insertar y eliminar nodos del índice"""
    embedder = EmbedderSimulado(dim=64)
    index = {}  # node_id → (node, emb)

    # Insertar
    nodos_iniciales = [SimpleNode(f"Documento {i}", doc_id=f"doc_{i}") for i in range(5)]
    for n in nodos_iniciales:
        emb = embedder.encode([n.text])[0]
        index[n.node_id] = (n, emb)

    assert len(index) == 5, f"Deben ser 5 nodos, got {len(index)}"

    # Insertar nuevo
    nuevo = SimpleNode("Nuevo documento", doc_id="doc_nuevo")
    emb_nuevo = embedder.encode([nuevo.text])[0]
    index[nuevo.node_id] = (nuevo, emb_nuevo)
    assert len(index) == 6, f"Tras insert: deben ser 6, got {len(index)}"

    # Eliminar por doc_id
    to_del = [nid for nid, (n, _) in index.items() if n.doc_id == "doc_nuevo"]
    for nid in to_del:
        del index[nid]
    assert len(index) == 5, f"Tras delete: deben ser 5, got {len(index)}"
    print(f"T5 PASS: insert y delete correctos ({len(index)} nodos finales)")

def test_compact_mode_concatena_nodos():
    """T6: el modo compact concatena todos los nodos en un único prompt"""
    nodos_texto = [
        "BERT usa encoders bidireccionales.",
        "GPT usa decoders causales.",
    ]
    query = "¿Qué modelos existen?"

    # Modo compact: concatenar
    contexto = "\n\n".join([f"[{i+1}] {t}" for i, t in enumerate(nodos_texto)])
    prompt = f"Contexto:\n{contexto}\n\nPregunta: {query}\nRespuesta:"

    assert "[1]" in prompt, "Debe tener referencia [1]"
    assert "[2]" in prompt, "Debe tener referencia [2]"
    assert "BERT" in prompt, "Debe contener contenido del primer nodo"
    assert "GPT" in prompt, "Debe contener contenido del segundo nodo"
    assert query in prompt, "Debe contener la query"
    print(f"T6 PASS: modo compact genera prompt de {len(prompt)} chars con {len(nodos_texto)} nodos")

if __name__ == "__main__":
    tests = [
        test_nodeparser_genera_nodos,
        test_vector_index_retrieval,
        test_keyword_index_score,
        test_response_tiene_source_nodes,
        test_insert_delete_index,
        test_compact_mode_concatena_nodos,
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
        print("M18 COMPLETADO")
```

---

## Resumen

| Componente | LlamaIndex | Descripción |
|---|---|---|
| `Document` | `llama_index.core.Document` | Dato raw de entrada |
| `TextNode` | `llama_index.core.schema.TextNode` | Chunk con embedding |
| `SentenceSplitter` | `llama_index.core.node_parser` | Parser de nodos |
| `VectorStoreIndex` | `llama_index.core.VectorStoreIndex` | Índice vectorial principal |
| `KeywordTableIndex` | `llama_index.core.KeywordTableIndex` | Índice léxico |
| `TreeIndex` | `llama_index.core.TreeIndex` | Índice jerárquico |
| `RetrieverQueryEngine` | `llama_index.core.query_engine` | Motor de consulta |
| `SubQuestionQueryEngine` | `llama_index.core.query_engine` | Multi-documento |

**Pipeline LlamaIndex típico:**
```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# Cargar
docs = SimpleDirectoryReader("./data").load_data()

# Indexar
index = VectorStoreIndex.from_documents(docs)

# Query
query_engine = index.as_query_engine(similarity_top_k=3)
response = query_engine.query("¿Qué es el Transformer?")
print(response)
print(response.source_nodes)
```
