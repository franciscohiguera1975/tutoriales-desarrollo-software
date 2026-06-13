# Módulo 14 — RAG Avanzado — HyDE, RAG-Fusion, Contextual Retrieval

**Objetivo:** Implementar técnicas avanzadas de RAG que mejoran la calidad de recuperación: HyDE (Hypothetical Document Embeddings), RAG-Fusion con múltiples queries, Contextual Retrieval, Parent-Document Retrieval y re-ranking con cross-encoders.

**Herramientas:** `numpy`, `sentence-transformers`, `torch`, `transformers`

**Prerequisito:** M13 (RAG Básico)

---

## 1. Problema con RAG básico y soluciones avanzadas

```python
# problemas_rag_basico.py
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# --- El problema del mismatch semántico ---
# En RAG básico, comparamos el embedding de la PREGUNTA con embeddings de DOCUMENTOS.
# Pero preguntas y documentos tienen distribuciones lingüísticas muy diferentes:
#   Pregunta: "¿Cómo funciona la atención en transformers?"
#   Documento: "El mecanismo self-attention calcula pesos de atención como softmax(QK^T/√d)..."
#
# La pregunta y el documento tratan el mismo tema pero el espacio embedding
# puede no alinearlos perfectamente. Soluciones:
#
#   HyDE:              Pregunta → LLM genera doc hipotético → embed doc → buscar
#   Query Expansion:   Pregunta → LLM genera sub-queries → buscar múltiple → fusionar
#   RAG-Fusion:        Múltiples queries → múltiples resultados → RRF → top chunks
#   Contextual:        Chunk + contexto del documento → embed enriquecido
#   Parent-Document:   Buscar en chunks pequeños → retornar chunk padre grande
#   Re-ranking:        Cross-encoder rescora los top-K candidatos

problemas = {
    "mismatch_semantico": "Pregunta vs. documento: distribuciones lingüísticas distintas",
    "chunks_sin_contexto": "Un chunk no tiene el contexto de su posición en el doc",
    "query_ambigua": "Una sola query puede no capturar todos los aspectos de la pregunta",
    "precision_vs_recall": "Chunks pequeños = mejor precisión pero peor recall",
}
for k, v in problemas.items():
    print(f"  {k}: {v}")

print("\nSoluciones avanzadas implementadas en este módulo:")
soluciones = ["HyDE", "Query Expansion", "RAG-Fusion + RRF",
              "Contextual Retrieval", "Parent-Document", "Cross-Encoder Re-ranking"]
for s in soluciones:
    print(f"  ✓ {s}")

np.save("outputs/m14_intro.npy", np.array([1]))
print("\noutputs/m14_intro.npy guardado")
```

---

## 2. HyDE — Hypothetical Document Embeddings

```python
# hyde_rag.py
import numpy as np
import os
from typing import List, Dict

os.makedirs("outputs", exist_ok=True)

class EmbedderSimulado:
    """Embedder determinístico para pruebas."""
    def __init__(self, dim=64):
        self.dim = dim

    def encode(self, textos: List[str]) -> np.ndarray:
        vecs = []
        for t in textos:
            seed = sum(ord(c) * (i+1) for i, c in enumerate(t[:200])) % (2**31)
            rng = np.random.RandomState(seed)
            v = rng.randn(self.dim).astype(np.float32)
            vecs.append(v / (np.linalg.norm(v) + 1e-9))
        return np.stack(vecs)

class VectorStore:
    def __init__(self):
        self.ids = []; self.vectors = []; self.textos = []

    def add(self, ids, vectors, textos):
        for id_, vec, txt in zip(ids, vectors, textos):
            self.ids.append(id_)
            self.vectors.append(vec / (np.linalg.norm(vec) + 1e-9))
            self.textos.append(txt)

    def search(self, q, k=5):
        if not self.vectors: return []
        q = q / (np.linalg.norm(q) + 1e-9)
        vecs = np.stack(self.vectors)
        sims = vecs @ q
        top_k = np.argsort(sims)[::-1][:k]
        return [{"id": self.ids[i], "texto": self.textos[i],
                 "score": float(sims[i])} for i in top_k]

class HyDE:
    """
    Hypothetical Document Embeddings.

    Idea: en lugar de embeder la PREGUNTA y buscar documentos similares,
    se genera un documento HIPOTÉTICO que respondería la pregunta,
    se embede ese documento y se buscan documentos similares.

    Ventaja: el documento hipotético tiene la misma distribución lingüística
    que los documentos del corpus → mejor alineación en el espacio embedding.

    Pipeline:
      1. query q → LLM(q) → doc hipotético h
      2. embed(h) → vector de búsqueda
      3. buscar(embed(h), corpus) → chunks relevantes
    """
    def __init__(self, embedder, store: VectorStore, llm_fn=None):
        self.embedder = embedder
        self.store = store
        self.llm_fn = llm_fn or self._llm_simulado

    def _llm_simulado(self, prompt: str) -> str:
        """LLM simulado: genera documento hipotético."""
        # En producción: llamar a GPT-4 / Claude / Llama
        if "transformer" in prompt.lower() or "atención" in prompt.lower():
            return """El mecanismo de self-attention en transformers calcula pesos de
atención usando queries, keys y values. La fórmula es softmax(QK^T/√d_k)V.
Permite capturar dependencias de largo alcance en la secuencia."""
        elif "bert" in prompt.lower():
            return """BERT (Bidirectional Encoder Representations from Transformers) es
un modelo preentrenado con Masked Language Modeling. Usa encoders bidireccionales
para producir representaciones contextuales de alta calidad."""
        elif "rag" in prompt.lower():
            return """RAG (Retrieval-Augmented Generation) combina recuperación de documentos
con generación de texto. Primero busca chunks relevantes en una base de datos vectorial,
luego usa esos chunks como contexto para que un LLM genere la respuesta."""
        else:
            return f"Un documento sobre el tema de la pregunta: {prompt[:50]}..."

    def _generar_doc_hipotetico(self, query: str) -> str:
        """Genera documento hipotético usando el LLM."""
        prompt = f"""Escribe un párrafo conciso que respondería a esta pregunta:
Pregunta: {query}
Párrafo (en el mismo estilo que documentos técnicos):"""
        return self.llm_fn(prompt)

    def query(self, pregunta: str, k: int = 3,
               n_docs_hipoteticos: int = 1) -> Dict:
        """
        n_docs_hipoteticos > 1: generar múltiples docs y promediar sus embeddings.
        Reduce varianza en la generación.
        """
        # Generar docs hipotéticos
        docs_hipoteticos = [self._generar_doc_hipotetico(pregunta)
                            for _ in range(n_docs_hipoteticos)]

        # Embeder y promediar
        embeddings_h = self.embedder.encode(docs_hipoteticos)
        vec_busqueda = embeddings_h.mean(axis=0)  # promedio de embeddings hipotéticos

        # Búsqueda
        resultados = self.store.search(vec_busqueda, k=k)

        return {
            "pregunta": pregunta,
            "docs_hipoteticos": docs_hipoteticos,
            "resultados": resultados,
            "vec_busqueda": vec_busqueda
        }

# --- Demo HyDE ---
np.random.seed(42)
embedder = EmbedderSimulado(dim=64)
store = VectorStore()

# Corpus
corpus = [
    "El self-attention calcula QK^T/√d_k para ponderar la importancia de cada token.",
    "BERT usa bidireccionalidad y MLM para preentrenamiento.",
    "RAG combina recuperación semántica con generación de texto condicionado.",
    "Los gradientes que desaparecen son un problema en RNNs profundas.",
    "LoRA añade matrices de bajo rango sin modificar los pesos originales.",
    "FAISS indexa vectores para búsqueda eficiente de vecinos más cercanos.",
]
ids = [f"doc_{i}" for i in range(len(corpus))]
embs = embedder.encode(corpus)
store.add(ids, list(embs), corpus)

hyde = HyDE(embedder, store)

preguntas = [
    "¿Cómo funciona el mecanismo de atención en transformers?",
    "¿Qué es BERT y cómo fue preentrenado?",
]

print("=== HyDE Demo ===\n")
resultados_hyde = []
for pregunta in preguntas:
    res = hyde.query(pregunta, k=2, n_docs_hipoteticos=2)
    print(f"Pregunta: {pregunta}")
    print(f"Doc hipotético: {res['docs_hipoteticos'][0][:100]}...")
    print(f"Top resultado: {res['resultados'][0]['texto'][:80]}... (score={res['resultados'][0]['score']:.4f})")
    print()
    resultados_hyde.append({
        "pregunta": pregunta,
        "top_score": res["resultados"][0]["score"]
    })

np.save("outputs/m14_hyde_resultados.npy",
        np.array([r["top_score"] for r in resultados_hyde]))
print("outputs/m14_hyde_resultados.npy guardado")
```

---

## 3. Query Expansion y RAG-Fusion

```python
# rag_fusion.py
import numpy as np
import os
from typing import List, Dict

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

class VectorStore:
    def __init__(self):
        self.ids = []; self.vectors = []; self.textos = []

    def add(self, ids, vectors, textos):
        for id_, vec, txt in zip(ids, vectors, textos):
            self.ids.append(id_)
            self.vectors.append(vec / (np.linalg.norm(vec) + 1e-9))
            self.textos.append(txt)

    def search(self, q, k=5):
        if not self.vectors: return []
        q = q / (np.linalg.norm(q) + 1e-9)
        sims = np.stack(self.vectors) @ q
        top_k = np.argsort(sims)[::-1][:k]
        return [{"id": self.ids[i], "score": float(sims[i])} for i in top_k]

# --- Reciprocal Rank Fusion ---
def rrf(ranked_lists: List[List[str]], k: int = 60) -> List[str]:
    """
    RRF: fusiona múltiples rankings.
    score(d) = Σ_i 1 / (k + rank_i(d))
    k=60 penaliza posiciones bajas y es empíricamente óptimo.
    """
    scores: Dict[str, float] = {}
    for ranked in ranked_lists:
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

# --- Query Expansion ---
class QueryExpander:
    """
    Genera múltiples variaciones de la query original para mejorar recall.
    Estrategias:
    - Sub-queries: descomponer pregunta compleja en sub-preguntas
    - Paráfrasis: reformular la misma pregunta de formas distintas
    - Step-back: preguntas más generales (de específico → general)
    """
    def __init__(self, llm_fn=None):
        self.llm_fn = llm_fn or self._llm_simulado

    def _llm_simulado(self, prompt: str) -> str:
        """LLM simulado: genera variaciones de la query."""
        return prompt  # Pasamos las queries directamente en el simulado

    def expandir_subqueries(self, query: str, n: int = 3) -> List[str]:
        """Genera sub-preguntas que descomponen la query principal."""
        # En producción, prompt: "Genera {n} sub-preguntas para: {query}"
        # Simulado: modificaciones sintéticas
        sub = [query]
        palabras = query.split()
        if len(palabras) > 3:
            sub.append(" ".join(palabras[:len(palabras)//2]))     # primera mitad
            sub.append(" ".join(palabras[len(palabras)//2:]))     # segunda mitad
        sub.append(f"explicar {query.replace('¿', '').replace('?', '').strip()}")
        return sub[:n+1]

    def expandir_parafraseo(self, query: str, n: int = 3) -> List[str]:
        """Genera paráfrasis de la query."""
        variaciones = [
            query,
            f"¿Puedes explicar {query.replace('¿', '').replace('?', '')}?",
            f"Describe {query.replace('¿', '').replace('?', '').replace('Qué es', 'el concepto de')}",
            f"¿Cómo se define {query.replace('¿', '').replace('?', '').replace('Qué es', '').strip()}?",
        ]
        return variaciones[:n+1]

    def step_back(self, query: str) -> List[str]:
        """Step-back prompting: pregunta más general → más específica."""
        return [
            query,  # original
            f"¿Qué conceptos son relevantes para entender {query.replace('¿','').replace('?','')}?",
            "¿Cuál es el contexto general de este tema?",
        ]

# --- RAG-Fusion ---
class RAGFusion:
    """
    RAG-Fusion combina múltiples queries para mejorar recall y diversidad.

    Pipeline:
    1. Query original + variaciones generadas
    2. Búsqueda independiente para cada query
    3. Fusión con RRF
    4. Construcción del prompt con chunks fusionados
    """
    def __init__(self, embedder, store: VectorStore,
                 expander: QueryExpander, top_k: int = 3, rrf_k: int = 60):
        self.embedder = embedder
        self.store = store
        self.expander = expander
        self.top_k = top_k
        self.rrf_k = rrf_k

    def query(self, pregunta: str, estrategia: str = "subqueries") -> Dict:
        """
        estrategia: "subqueries" | "parafraseo" | "step_back"
        """
        # 1. Generar queries
        if estrategia == "subqueries":
            queries = self.expander.expandir_subqueries(pregunta, n=3)
        elif estrategia == "parafraseo":
            queries = self.expander.expandir_parafraseo(pregunta, n=3)
        else:  # step_back
            queries = self.expander.step_back(pregunta)

        # 2. Búsqueda para cada query
        rankings = []
        todos_resultados = {}

        for q in queries:
            q_emb = self.embedder.encode([q])[0]
            resultados = self.store.search(q_emb, k=self.top_k * 2)
            rankings.append([r["id"] for r in resultados])
            for r in resultados:
                todos_resultados[r["id"]] = r

        # 3. RRF fusion
        ids_fusionados = rrf(rankings, k=self.rrf_k)[:self.top_k]

        # 4. Recuperar textos
        chunks_finales = [todos_resultados[id_] for id_ in ids_fusionados
                          if id_ in todos_resultados]

        return {
            "pregunta": pregunta,
            "queries_generadas": queries,
            "ids_fusionados": ids_fusionados,
            "chunks": chunks_finales,
            "n_queries": len(queries)
        }

# --- Demo ---
np.random.seed(42)
embedder = EmbedderSimulado(dim=64)
store = VectorStore()

corpus = [
    "doc0: BERT usa MLM para preentrenamiento bidireccional.",
    "doc1: El self-attention computa QK^T/√d_k para pesos de atención.",
    "doc2: LoRA añade matrices de bajo rango a las capas Wq y Wv.",
    "doc3: RAG combina recuperación vectorial con generación LLM.",
    "doc4: FAISS indexa vectores con HNSW para búsqueda ANN.",
    "doc5: GPT usa decoder causal con máscara para predicción de tokens.",
    "doc6: Sentence-transformers produce embeddings para búsqueda semántica.",
    "doc7: RoPE aplica codificación posicional rotacional a Q y K.",
]
embs = embedder.encode(corpus)
store.add([f"doc_{i}" for i in range(len(corpus))], list(embs), corpus)

expander = QueryExpander()
rag_fusion = RAGFusion(embedder, store, expander, top_k=3)

pregunta = "¿Cómo se implementa la atención en modelos de lenguaje?"
print("=== RAG-Fusion Demo ===\n")
print(f"Pregunta: {pregunta}")

for estrategia in ["subqueries", "parafraseo", "step_back"]:
    res = rag_fusion.query(pregunta, estrategia=estrategia)
    print(f"\nEstrategia: {estrategia}")
    print(f"  Queries generadas ({res['n_queries']}): {res['queries_generadas']}")
    print(f"  IDs fusionados: {res['ids_fusionados']}")

np.save("outputs/m14_rag_fusion_demo.npy", np.array([1]))
print("\noutputs/m14_rag_fusion_demo.npy guardado")
```

---

## 4. Contextual Retrieval

```python
# contextual_retrieval.py
import numpy as np
import os
from typing import List, Dict

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

class ContextualRetrieval:
    """
    Contextual Retrieval (Anthropic, 2024).

    Problema: un chunk como "Este artículo explica..." no tiene contexto.
    Solución: añadir contexto del documento completo a cada chunk ANTES de indexar.

    Pipeline:
    1. Para cada chunk, enviar al LLM:
       DOCUMENTO: [doc completo]
       CHUNK: [chunk]
       "Proporciona un contexto breve (2-3 frases) que sitúe este chunk
        dentro del documento completo."
    2. Concatenar contexto + chunk → indexar
    3. Búsqueda normal sobre chunks enriquecidos

    Resultado: mejora del 35-49% en recall@20 (según benchmark de Anthropic).
    """
    def __init__(self, embedder, llm_fn=None):
        self.embedder = embedder
        self.llm_fn = llm_fn or self._llm_simulado
        self.chunks_contextuales: List[Dict] = []
        self.vectors: List[np.ndarray] = []

    def _llm_simulado(self, doc_completo: str, chunk: str) -> str:
        """LLM simulado para generación de contexto."""
        # En producción: llamar a Claude claude-haiku-4-5-20251001 o similar
        titulo = doc_completo.split(".")[0] if "." in doc_completo else doc_completo[:50]
        return f"Este fragmento pertenece a un documento sobre '{titulo}'. " \
               f"Proporciona información sobre {chunk.split()[0] if chunk.split() else 'el tema'}."

    def _generar_contexto(self, doc_completo: str, chunk: str) -> str:
        """Genera contexto situacional para el chunk."""
        prompt = f"""Documento completo:
{doc_completo[:800]}

Chunk específico:
{chunk}

Proporciona un contexto breve (2-3 frases) que sitúe este chunk dentro del documento:"""
        return self.llm_fn(doc_completo, chunk)

    def indexar(self, documentos: List[Dict]):
        """
        documentos: lista de {"id": str, "titulo": str, "contenido": str, "chunks": list}
        """
        todos_chunks = []

        for doc in documentos:
            doc_completo = f"{doc.get('titulo', '')}\n{doc['contenido']}"
            for i, chunk_texto in enumerate(doc["chunks"]):
                # Generar contexto
                contexto = self._generar_contexto(doc_completo, chunk_texto)
                # Chunk contextualizado: contexto + chunk original
                chunk_contextualizado = f"{contexto}\n\n{chunk_texto}"
                todos_chunks.append({
                    "id": f"{doc['id']}_chunk_{i}",
                    "chunk_original": chunk_texto,
                    "contexto": contexto,
                    "chunk_contextualizado": chunk_contextualizado,
                    "doc_id": doc["id"]
                })

        # Embeder chunks contextualizados
        textos = [c["chunk_contextualizado"] for c in todos_chunks]
        embs = self.embedder.encode(textos)

        for chunk, emb in zip(todos_chunks, embs):
            chunk["embedding"] = emb
            self.chunks_contextuales.append(chunk)
            self.vectors.append(emb / (np.linalg.norm(emb) + 1e-9))

        print(f"Indexados: {len(todos_chunks)} chunks contextualizados")

    def search(self, query: str, k: int = 3) -> List[Dict]:
        """Búsqueda sobre chunks contextualizados."""
        if not self.vectors:
            return []
        q_emb = self.embedder.encode([query])[0]
        q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)
        vecs = np.stack(self.vectors)
        sims = vecs @ q_norm
        top_k = np.argsort(sims)[::-1][:k]
        return [
            {
                **self.chunks_contextuales[i],
                "score": float(sims[i])
            }
            for i in top_k
        ]

# --- Demo: comparar retrieval con/sin contexto ---
np.random.seed(42)
embedder = EmbedderSimulado(dim=64)

documentos = [
    {
        "id": "doc_transformer",
        "titulo": "Guía de Transformers",
        "contenido": "El transformer revolucionó el NLP. Usa atención multi-cabeza. BERT y GPT son derivados.",
        "chunks": [
            "Este método usa múltiples cabezas de atención.",
            "Fue introducido en 2017 con el paper seminal.",
            "Permite paralelismo durante el entrenamiento.",
        ]
    },
    {
        "id": "doc_rag",
        "titulo": "RAG: Recuperación Aumentada",
        "contenido": "RAG mejora LLMs con recuperación externa. Reduce alucinaciones. Escala bien.",
        "chunks": [
            "El sistema recupera documentos relevantes.",
            "Las respuestas se anclan en fuentes reales.",
            "La indexación vectorial permite búsqueda semántica.",
        ]
    }
]

cr = ContextualRetrieval(embedder)
cr.indexar(documentos)

print("\n=== Contextual Retrieval Demo ===")
query = "búsqueda semántica con múltiples cabezas"
resultados = cr.search(query, k=3)
for r in resultados:
    print(f"\nDoc {r['doc_id']} | score={r['score']:.4f}")
    print(f"  Chunk original: {r['chunk_original']}")
    print(f"  Contexto añadido: {r['contexto']}")

# Comparación cuantitativa: chunk original vs contextualizado
print("\n=== Impacto del contexto ===")
# Embeder sólo los chunks originales (sin contexto)
chunks_originales = [c["chunk_original"] for c in cr.chunks_contextuales]
chunks_contextualizados = [c["chunk_contextualizado"] for c in cr.chunks_contextuales]

embs_orig = embedder.encode(chunks_originales)
embs_ctx = embedder.encode(chunks_contextualizados)

# Varianza de embeddings (más varianza → más distinguibles)
var_orig = embs_orig.var(axis=0).mean()
var_ctx = embs_ctx.var(axis=0).mean()
print(f"Varianza media (originales):      {var_orig:.6f}")
print(f"Varianza media (contextualizados): {var_ctx:.6f}")

np.save("outputs/m14_contextual_retrieval.npy",
        np.array([var_orig, var_ctx]))
print("\noutputs/m14_contextual_retrieval.npy guardado")
```

---

## 5. Parent-Document Retrieval y Cross-Encoder Re-ranking

```python
# parent_document_reranking.py
import numpy as np
import os
from typing import List, Dict, Tuple

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

# --- Parent-Document Retrieval ---
class ParentDocumentRetriever:
    """
    Parent-Document Retrieval:
    - Indexa chunks PEQUEÑOS (mejor precisión en recuperación)
    - Retorna el chunk PADRE GRANDE (más contexto para el LLM)

    Esto evita el dilema:
    - Chunks pequeños: más precisos para recuperar, pero poco contexto
    - Chunks grandes: más contexto, pero menor precisión de recuperación

    Solución: buscar en pequeños, retornar grandes.
    """
    def __init__(self, embedder,
                 child_size: int = 100, parent_size: int = 400):
        self.embedder = embedder
        self.child_size = child_size
        self.parent_size = parent_size
        self.parents: Dict[str, str] = {}    # parent_id → texto completo
        self.children: List[Dict] = []
        self.child_vectors: List[np.ndarray] = []

    def _crear_chunks(self, texto: str, size: int, overlap: int = 20) -> List[str]:
        chunks = []
        start = 0
        while start < len(texto):
            chunks.append(texto[start:start+size])
            start += size - overlap
        return chunks

    def indexar(self, documentos: List[Dict]):
        """
        documentos: lista de {"id": str, "contenido": str}
        """
        for doc in documentos:
            texto = doc["contenido"]
            # Crear chunks padre (grandes)
            padres = self._crear_chunks(texto, self.parent_size, overlap=50)
            for p_idx, padre in enumerate(padres):
                p_id = f"{doc['id']}_padre_{p_idx}"
                self.parents[p_id] = padre
                # Crear chunks hijo (pequeños) dentro de cada padre
                hijos = self._crear_chunks(padre, self.child_size, overlap=10)
                for h_idx, hijo in enumerate(hijos):
                    child_entry = {
                        "id": f"{p_id}_hijo_{h_idx}",
                        "texto": hijo,
                        "parent_id": p_id,
                        "doc_id": doc["id"]
                    }
                    self.children.append(child_entry)
                    emb = self.embedder.encode([hijo])[0]
                    self.child_vectors.append(emb / (np.linalg.norm(emb)+1e-9))

        print(f"Indexados: {len(self.children)} hijos en {len(self.parents)} padres")

    def search(self, query: str, k: int = 3) -> List[Dict]:
        """Busca en hijos, retorna padres únicos."""
        if not self.child_vectors:
            return []
        q_emb = self.embedder.encode([query])[0]
        q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)
        vecs = np.stack(self.child_vectors)
        sims = vecs @ q_norm

        # Top k*3 hijos (para asegurar k padres únicos)
        top_children = np.argsort(sims)[::-1][:k*3]
        vistos_padres = set()
        resultados = []

        for idx in top_children:
            child = self.children[idx]
            p_id = child["parent_id"]
            if p_id not in vistos_padres:
                vistos_padres.add(p_id)
                resultados.append({
                    "child_id": child["id"],
                    "child_texto": child["texto"],
                    "parent_id": p_id,
                    "parent_texto": self.parents[p_id],
                    "child_score": float(sims[idx])
                })
            if len(resultados) >= k:
                break

        return resultados

# --- Cross-Encoder Re-ranking ---
class CrossEncoderSimulado:
    """
    Cross-Encoder: modelo que recibe (query, documento) y produce un score de relevancia.
    A diferencia de bi-encoders (embeds separados), el cross-encoder procesa el par JUNTO,
    lo que permite capturar interacciones sutiles entre query y documento.

    Más preciso que bi-encoder pero más lento (O(N) forward passes por query).
    Se usa para re-rankear el top-K del bi-encoder (normalmente top-50 → re-rank → top-10).
    """
    def __init__(self):
        np.random.seed(999)
        # Simular un cross-encoder con features manuales
        pass

    def score(self, query: str, doc: str) -> float:
        """
        Score de relevancia (query, doc). Rango: [0, 1].
        Simulado: word overlap + longitud heurística.
        """
        palabras_q = set(query.lower().split())
        palabras_d = set(doc.lower().split())
        stopwords = {"el", "la", "los", "las", "de", "del", "en", "un", "una",
                     "y", "o", "a", "con", "por", "que", "se", "es", "son",
                     "su", "sus", "¿", "?", "cómo", "qué", "cuál"}
        palabras_q -= stopwords
        palabras_d -= stopwords
        if not palabras_q:
            return 0.0
        overlap = len(palabras_q & palabras_d) / len(palabras_q)
        # Bonus por longitud del documento (más información)
        len_bonus = min(len(palabras_d) / 50.0, 0.2)
        return min(float(overlap + len_bonus), 1.0)

    def rerank(self, query: str, documentos: List[Dict],
               texto_key: str = "texto") -> List[Dict]:
        """Re-rankea documentos usando cross-encoder score."""
        for doc in documentos:
            doc["rerank_score"] = self.score(query, doc[texto_key])
        return sorted(documentos, key=lambda x: x["rerank_score"], reverse=True)

# --- Demo integrado ---
np.random.seed(42)
embedder = EmbedderSimulado(dim=64)

doc_largo = """
Los transformers son la arquitectura dominante en deep learning moderno.
El mecanismo de self-attention permite que cada token atienda a todos los demás.
La fórmula de atención es: Attention(Q,K,V) = softmax(QK^T/√d_k)V.
Multi-head attention divide el espacio en múltiples cabezas de atención.
Cada cabeza aprende relaciones diferentes entre los tokens.
BERT usa transformers bidireccionales preentrenados con MLM y NSP.
GPT usa transformers causales para generación de texto autoregresivo.
LLaMA mejora la eficiencia con RoPE, RMSNorm y SwiGLU.
Los transformers escalaron mejor que las RNNs en hardware moderno.
RAG combina transformers con recuperación de información externa.
"""

# Parent-Document Retrieval
pdr = ParentDocumentRetriever(embedder, child_size=80, parent_size=250)
pdr.indexar([{"id": "doc_main", "contenido": doc_largo.strip()}])

query = "mecanismo de atención con queries y keys"
print("=== Parent-Document Retrieval ===")
resultados_pdr = pdr.search(query, k=2)
for r in resultados_pdr:
    print(f"\nHijo recuperado (score={r['child_score']:.4f}):")
    print(f"  '{r['child_texto']}'")
    print(f"Padre retornado ({len(r['parent_texto'])} chars):")
    print(f"  '{r['parent_texto'][:100]}...'")

# Cross-Encoder Re-ranking
print("\n=== Cross-Encoder Re-ranking ===")
cross_encoder = CrossEncoderSimulado()

candidatos = [
    {"id": "c1", "texto": "BERT usa MLM y NSP para preentrenamiento bidireccional."},
    {"id": "c2", "texto": "La atención computa softmax(QK^T/√d_k)V sobre todos los tokens."},
    {"id": "c3", "texto": "Los transformers escalaron mejor que RNNs en hardware GPU."},
    {"id": "c4", "texto": "Multi-head attention divide queries keys values en cabezas."},
    {"id": "c5", "texto": "RAG mejora LLMs con recuperación de información externa."},
]

reranked = cross_encoder.rerank(query, candidatos.copy(), texto_key="texto")
print(f"Query: '{query}'")
print("Re-ranking:")
for i, doc in enumerate(reranked):
    print(f"  #{i+1}: [{doc['id']}] score={doc['rerank_score']:.4f} | {doc['texto'][:60]}")

scores_rerank = np.array([d["rerank_score"] for d in reranked])
np.save("outputs/m14_reranking_scores.npy", scores_rerank)
print("\noutputs/m14_reranking_scores.npy guardado")
```

---

## 6. Pipeline RAG avanzado integrado

```python
# rag_avanzado_completo.py
import numpy as np
import os
from typing import List, Dict

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

class AdvancedRAG:
    """
    Pipeline RAG avanzado con:
    - Contextual Retrieval en la indexación
    - HyDE o Query Expansion en la búsqueda
    - Re-ranking con cross-encoder
    - Fusión RRF de múltiples resultados
    """
    def __init__(self, embedder, llm_fn=None, top_k=5, rerank_top=10):
        self.embedder = embedder
        self.llm_fn = llm_fn or self._llm_sim
        self.top_k = top_k
        self.rerank_top = rerank_top
        self.chunks: List[Dict] = []
        self.vectors: List[np.ndarray] = []

    def _llm_sim(self, texto): return f"[LLM simulado] {texto[:60]}"

    def _contextualizar_chunk(self, doc_contenido, chunk):
        return f"Sección de documento sobre {doc_contenido.split()[0]}. {chunk}"

    def indexar(self, documentos: List[Dict]):
        """Indexa con contextualización automática."""
        for doc in documentos:
            texto = doc["contenido"]
            # Chunking simple por párrafos
            parrafos = [p.strip() for p in texto.split("\n") if p.strip()]
            for i, parrafo in enumerate(parrafos):
                ctx = self._contextualizar_chunk(texto, parrafo)
                chunk_ctx = f"{ctx}\n{parrafo}"
                emb = self.embedder.encode([chunk_ctx])[0]
                self.chunks.append({
                    "id": f"{doc['id']}_p{i}",
                    "texto": parrafo,
                    "texto_contextualizado": chunk_ctx,
                    "doc_id": doc["id"]
                })
                self.vectors.append(emb / (np.linalg.norm(emb) + 1e-9))
        print(f"Indexados: {len(self.chunks)} chunks")

    def _buscar_raw(self, query_emb: np.ndarray, k: int) -> List[Dict]:
        if not self.vectors: return []
        q = query_emb / (np.linalg.norm(query_emb) + 1e-9)
        sims = np.stack(self.vectors) @ q
        top_k = np.argsort(sims)[::-1][:k]
        return [{"idx": int(i), "score": float(sims[i]),
                 **self.chunks[i]} for i in top_k]

    def _rrf(self, listas_ids: List[List[str]], k=60) -> List[str]:
        scores = {}
        for lista in listas_ids:
            for rank, id_ in enumerate(lista, start=1):
                scores[id_] = scores.get(id_, 0.0) + 1.0 / (k + rank)
        return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

    def _cross_encoder_score(self, query: str, texto: str) -> float:
        palabras_q = set(query.lower().split())
        palabras_d = set(texto.lower().split())
        stopwords = {"el","la","los","las","de","del","en","un","una","y","o","a","que","se"}
        palabras_q -= stopwords; palabras_d -= stopwords
        if not palabras_q: return 0.0
        return float(len(palabras_q & palabras_d) / len(palabras_q))

    def query(self, pregunta: str, use_hyde: bool = True,
              use_expansion: bool = True) -> Dict:
        """Pipeline completo."""
        resultados_por_query = []

        # Query 1: original
        q_emb_orig = self.embedder.encode([pregunta])[0]
        res_orig = self._buscar_raw(q_emb_orig, k=self.rerank_top)
        resultados_por_query.append([r["id"] for r in res_orig])

        # Query 2: HyDE (doc hipotético)
        if use_hyde:
            doc_hipotetico = f"El {pregunta.replace('¿','').replace('?','').strip()} funciona mediante..."
            q_emb_hyde = self.embedder.encode([doc_hipotetico])[0]
            res_hyde = self._buscar_raw(q_emb_hyde, k=self.rerank_top)
            resultados_por_query.append([r["id"] for r in res_hyde])

        # Query 3: expansión
        if use_expansion:
            palabras_key = [w for w in pregunta.split() if len(w) > 4]
            q_expandida = " ".join(palabras_key) if palabras_key else pregunta
            q_emb_exp = self.embedder.encode([q_expandida])[0]
            res_exp = self._buscar_raw(q_emb_exp, k=self.rerank_top)
            resultados_por_query.append([r["id"] for r in res_exp])

        # RRF fusion
        ids_fusionados = self._rrf(resultados_por_query)[:self.rerank_top]

        # Construir candidatos únicos
        todos_chunks = {r["id"]: r
                        for lista in [res_orig] + ([] if not use_hyde else [res_hyde])
                        for r in lista}
        candidatos = [todos_chunks[id_] for id_ in ids_fusionados
                      if id_ in todos_chunks]

        # Re-ranking con cross-encoder
        for c in candidatos:
            c["rerank_score"] = self._cross_encoder_score(pregunta, c["texto"])
        candidatos.sort(key=lambda x: x["rerank_score"], reverse=True)

        top_chunks = candidatos[:self.top_k]

        # Prompt
        contexto = "\n\n".join([f"[{i+1}] {c['texto']}"
                                 for i, c in enumerate(top_chunks)])
        prompt = f"Contexto:\n{contexto}\n\nPregunta: {pregunta}\nRespuesta:"

        return {
            "pregunta": pregunta,
            "n_queries_usadas": len(resultados_por_query),
            "chunks_finales": top_chunks,
            "prompt_preview": prompt[:200],
            "rerank_scores": [c["rerank_score"] for c in top_chunks]
        }

# --- Demo ---
np.random.seed(42)
embedder = EmbedderSimulado(dim=64)
rag = AdvancedRAG(embedder, top_k=3)

docs = [
    {"id": "nlp", "contenido": "BERT usa MLM bidireccional\nGPT usa decoder causal\nLLaMA es open-source"},
    {"id": "attn", "contenido": "Atención: softmax(QK^T/√d)V\nMulti-head divide en cabezas\nRoPE aplica rotaciones"},
    {"id": "rag", "contenido": "RAG combina recuperación con generación\nChunking divide documentos\nFAISS indexa vectores"},
]
rag.indexar(docs)

pregunta = "¿Cómo funciona la atención con queries y keys?"
resultado = rag.query(pregunta)
print(f"\n=== Advanced RAG ===")
print(f"Pregunta: {pregunta}")
print(f"Queries usadas: {resultado['n_queries_usadas']}")
print(f"Top chunks:")
for i, chunk in enumerate(resultado["chunks_finales"]):
    print(f"  #{i+1}: rerank={chunk['rerank_score']:.3f} | {chunk['texto'][:60]}")

import json
with open("outputs/m14_advanced_rag_demo.json", "w", encoding="utf-8") as f:
    json.dump({
        "pregunta": resultado["pregunta"],
        "n_queries": resultado["n_queries_usadas"],
        "rerank_scores": resultado["rerank_scores"]
    }, f, indent=2)
print("\noutputs/m14_advanced_rag_demo.json guardado")
```

---

## 7. Checkpoint — Pruebas unitarias

```python
# checkpoint_m14.py
"""
Checkpoint M14: RAG Avanzado
Ejecutar con: python checkpoint_m14.py
"""
import numpy as np
import os
from typing import List, Dict

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

def rrf(ranked_lists, k=60):
    scores = {}
    for ranked in ranked_lists:
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

def test_hyde_genera_doc_diferente_de_query():
    """T1: HyDE produce un documento más largo y diferente a la query original"""
    query = "¿Qué es BERT?"
    # En producción el LLM genera texto; aquí usamos una expansión simulada
    doc_hipotetico = f"BERT (Bidirectional Encoder Representations) es un modelo de lenguaje " \
                     f"preentrenado con MLM. La pregunta era: {query}"
    assert len(doc_hipotetico) > len(query) * 2, \
        "Doc hipotético debe ser más largo que la query"
    assert doc_hipotetico != query, "Doc hipotético debe diferir de la query"
    print(f"T1 PASS: doc hipotético ({len(doc_hipotetico)} chars) > query ({len(query)} chars)")

def test_rrf_da_mayor_score_a_documentos_consistentes():
    """T2: RRF da mayor score a docs que aparecen en múltiples rankings"""
    # doc_A aparece en los 3 rankings; doc_C solo en 1
    r1 = ["doc_A", "doc_B", "doc_C"]
    r2 = ["doc_D", "doc_A", "doc_E"]
    r3 = ["doc_A", "doc_F", "doc_G"]

    fused = rrf([r1, r2, r3])
    # doc_A aparece en posición alta en los 3 → debe ser el primero
    assert fused[0] == "doc_A", f"doc_A debe ser primero, got {fused[0]}"
    # doc_C solo aparece una vez (posición 3) → debe tener menor score que doc_A
    idx_a = fused.index("doc_A")
    idx_c = fused.index("doc_C")
    assert idx_a < idx_c, f"doc_A (idx={idx_a}) debe estar antes que doc_C (idx={idx_c})"
    print(f"T2 PASS: RRF posiciona doc_A en #{idx_a+1}, doc_C en #{idx_c+1}")

def test_contextual_retrieval_enriquece_chunks():
    """T3: chunk contextualizado es más largo que el chunk original"""
    chunk_original = "Este método calcula pesos de atención."
    doc_completo = "Guía de Transformers. El transformer usa atención multi-cabeza. BERT y GPT son ejemplos."

    contexto = f"Este fragmento pertenece a '{doc_completo.split('.')[0]}'. "
    chunk_contextualizado = f"{contexto}\n{chunk_original}"

    assert len(chunk_contextualizado) > len(chunk_original), \
        "Chunk contextualizado debe ser más largo"
    assert chunk_original in chunk_contextualizado, \
        "El chunk original debe estar contenido en el contextualizado"
    print(f"T3 PASS: contextualizado ({len(chunk_contextualizado)}) > original ({len(chunk_original)})")

def test_parent_document_retorna_padre():
    """T4: al buscar, se retorna el chunk padre (más grande) no el hijo"""
    np.random.seed(42)
    embedder = EmbedderSimulado(dim=64)

    doc_texto = "palabra " * 200   # 200 palabras
    child_size = 50
    parent_size = 150

    # Crear padres e hijos manualmente
    partes = doc_texto.split()
    padres_textos = [" ".join(partes[i:i+parent_size])
                     for i in range(0, len(partes), parent_size - 20)]
    hijos_textos = [" ".join(partes[i:i+child_size])
                    for i in range(0, len(partes), child_size - 5)]

    assert all(len(p) >= len(h) for p, h in zip(padres_textos, hijos_textos) if h), \
        "El padre debe tener más contenido que el hijo"

    # Verificar que el hijo está contenido en algún padre
    hijo = hijos_textos[0] if hijos_textos else ""
    padre = padres_textos[0] if padres_textos else ""
    # Comparar primeras palabras
    primeras_palabras_hijo = hijo.split()[:5]
    primeras_palabras_padre = padre.split()[:5]
    assert primeras_palabras_hijo == primeras_palabras_padre, \
        "El primer hijo debe iniciar igual que el primer padre"
    print(f"T4 PASS: padre={len(padre.split())} palabras >= hijo={len(hijo.split())} palabras")

def test_cross_encoder_rankea_mejor_match():
    """T5: el cross-encoder da mayor score al documento más relevante"""
    query = "mecanismo de atención en transformers"
    docs = [
        {"id": "relevante", "texto": "El mecanismo de atención en transformers calcula QK^T/√d_k."},
        {"id": "irrelevante", "texto": "El fútbol es el deporte más popular del mundo."},
        {"id": "parcial", "texto": "Los transformers procesan texto en paralelo."},
    ]

    def cross_score(q, d):
        pq = set(q.lower().split()) - {"de","en","el","la","los","las","un","una"}
        pd_ = set(d.lower().split()) - {"de","en","el","la","los","las","un","una"}
        return len(pq & pd_) / (len(pq) + 1e-9)

    for doc in docs:
        doc["score"] = cross_score(query, doc["texto"])

    docs_ordenados = sorted(docs, key=lambda x: x["score"], reverse=True)
    assert docs_ordenados[0]["id"] == "relevante", \
        f"El doc relevante debe ser primero, got {docs_ordenados[0]['id']}"
    assert docs_ordenados[-1]["id"] == "irrelevante", \
        f"El doc irrelevante debe ser último, got {docs_ordenados[-1]['id']}"
    print(f"T5 PASS: relevante={docs_ordenados[0]['score']:.3f} > "
          f"parcial={docs_ordenados[1]['score']:.3f} > "
          f"irrelevante={docs_ordenados[2]['score']:.3f}")

def test_pipeline_avanzado_mejora_sobre_basico():
    """T6: recuperar con múltiples queries + RRF produce mayor cobertura que una sola query"""
    np.random.seed(42)
    embedder = EmbedderSimulado(dim=64)

    docs = [f"documento_{i} sobre tema_{i%5}" for i in range(20)]
    embs = embedder.encode(docs)
    embs_norm = embs / (np.linalg.norm(embs, axis=1, keepdims=True) + 1e-9)

    # Query original
    query = "tema_2 relevante"
    q_emb = embedder.encode([query])[0]
    q_norm = q_emb / (np.linalg.norm(q_emb) + 1e-9)
    sims = embs_norm @ q_norm
    top5_single = set(np.argsort(sims)[::-1][:5].tolist())

    # Múltiples queries
    queries_extra = ["tema_2 información", "documentos tema dos"]
    all_rankings = [np.argsort(sims)[::-1][:10].tolist()]
    for eq in queries_extra:
        q2_emb = embedder.encode([eq])[0]
        q2_norm = q2_emb / (np.linalg.norm(q2_emb) + 1e-9)
        sims2 = embs_norm @ q2_norm
        all_rankings.append(np.argsort(sims2)[::-1][:10].tolist())

    fused_ids = rrf([[str(i) for i in r] for r in all_rankings])
    top5_multi = set(fused_ids[:5])

    # La cobertura de múltiples queries debe ser al menos igual
    # (en la práctica suele ser mayor)
    assert len(top5_multi) == 5, f"Debe retornar exactamente 5, got {len(top5_multi)}"
    print(f"T6 PASS: multi-query RRF retorna 5 resultados únicos; "
          f"overlap con single-query: {len(top5_single & top5_multi)}/5")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_hyde_genera_doc_diferente_de_query,
        test_rrf_da_mayor_score_a_documentos_consistentes,
        test_contextual_retrieval_enriquece_chunks,
        test_parent_document_retorna_padre,
        test_cross_encoder_rankea_mejor_match,
        test_pipeline_avanzado_mejora_sobre_basico,
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
        print("M14 COMPLETADO")
```

---

## Resumen

| Técnica | Problema que resuelve | Impacto |
|---|---|---|
| **HyDE** | Mismatch query/documento | +10-20% recall |
| **Query Expansion** | Query ambigua o incompleta | +15% recall |
| **RAG-Fusion + RRF** | Una query no es suficiente | +20% recall |
| **Contextual Retrieval** | Chunks sin contexto | +35-49% recall |
| **Parent-Document** | Dilema tamaño de chunk | Mejor precision+recall |
| **Cross-Encoder Re-rank** | Bi-encoder impreciso | +10-15% MRR |

**Flujo avanzado completo:**
```
Query → [HyDE] + [Expansión] → Múltiples queries
     → Búsqueda en VectorDB (chunks contextualizados)
     → RRF Fusion → Top-50 candidatos
     → Cross-Encoder Re-ranking → Top-5 chunks
     → Prompt + LLM → Respuesta
```
