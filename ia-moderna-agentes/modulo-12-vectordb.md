# Módulo 12 — Bases de Datos Vectoriales — Chroma, Pinecone, pgvector

**Objetivo:** Entender la arquitectura de bases de datos vectoriales, implementar operaciones CRUD sobre colecciones de embeddings, y comparar Chroma, Pinecone y pgvector para distintos casos de uso.

**Herramientas:** `chromadb`, `pinecone-client`, `psycopg2`, `numpy`, `sentence-transformers`

**Prerequisito:** M11 (Embeddings)

---

## 1. Arquitectura de bases de datos vectoriales

Una base de datos vectorial extiende una BD tradicional con:
- **Almacenamiento** de vectores densos de alta dimensión
- **Índices ANN** (Approximate Nearest Neighbor): HNSW, IVF, PQ
- **Filtrado híbrido**: búsqueda vectorial + filtros por metadatos
- **CRUD** de vectores con persistencia

```python
# arquitectura_vectordb.py
import numpy as np
import os
from typing import Dict, List, Any

os.makedirs("outputs", exist_ok=True)

# --- Implementación simplificada de una VectorDB ---
class SimpleVectorDB:
    """
    Base de datos vectorial mínima para entender los conceptos.
    Soporta: insert, search, delete, update, filter por metadatos.
    """
    def __init__(self, dim: int, metric: str = "cosine"):
        self.dim = dim
        self.metric = metric
        self.vectors: Dict[str, np.ndarray] = {}
        self.metadata: Dict[str, Dict] = {}
        self._id_counter = 0

    def _auto_id(self) -> str:
        self._id_counter += 1
        return f"id_{self._id_counter}"

    def insert(self, vector: np.ndarray, metadata: dict = None, doc_id: str = None) -> str:
        """Insertar un vector con metadatos opcionales."""
        assert vector.shape == (self.dim,), f"Dim esperada {self.dim}, got {vector.shape}"
        doc_id = doc_id or self._auto_id()
        # Normalizar si métrica es coseno
        if self.metric == "cosine":
            vector = vector / (np.linalg.norm(vector) + 1e-9)
        self.vectors[doc_id] = vector.astype(np.float32)
        self.metadata[doc_id] = metadata or {}
        return doc_id

    def insert_batch(self, vectors: np.ndarray, metadatas: list = None,
                     doc_ids: list = None) -> List[str]:
        """Inserción por lotes eficiente."""
        N = vectors.shape[0]
        metadatas = metadatas or [{} for _ in range(N)]
        doc_ids = doc_ids or [None] * N
        return [self.insert(vectors[i], metadatas[i], doc_ids[i]) for i in range(N)]

    def search(self, query: np.ndarray, k: int = 5,
               where: dict = None) -> List[Dict]:
        """
        Buscar los k vectores más similares.
        where: filtro de metadatos, ej. {"categoria": "ciencia"}
        """
        if self.metric == "cosine":
            query = query / (np.linalg.norm(query) + 1e-9)

        # Filtrar por metadatos si se especifica
        ids = list(self.vectors.keys())
        if where:
            ids = [i for i in ids
                   if all(self.metadata[i].get(k_) == v
                          for k_, v in where.items())]

        if not ids:
            return []

        # Calcular similitudes
        vecs = np.stack([self.vectors[i] for i in ids])
        if self.metric == "cosine":
            sims = vecs @ query           # producto punto entre vectores normalizados
        else:  # euclidean
            sims = -np.linalg.norm(vecs - query, axis=1)

        # Top-k
        top_idx = np.argsort(sims)[::-1][:k]
        return [
            {
                "id": ids[i],
                "score": float(sims[i]),
                "metadata": self.metadata[ids[i]]
            }
            for i in top_idx
        ]

    def delete(self, doc_id: str):
        self.vectors.pop(doc_id, None)
        self.metadata.pop(doc_id, None)

    def update(self, doc_id: str, vector: np.ndarray = None, metadata: dict = None):
        if vector is not None:
            if self.metric == "cosine":
                vector = vector / (np.linalg.norm(vector) + 1e-9)
            self.vectors[doc_id] = vector.astype(np.float32)
        if metadata is not None:
            self.metadata[doc_id].update(metadata)

    def __len__(self):
        return len(self.vectors)

    def stats(self):
        return {"n_docs": len(self), "dim": self.dim, "metric": self.metric}

# --- Demostración ---
np.random.seed(42)
DIM = 64
db = SimpleVectorDB(dim=DIM, metric="cosine")

# Insertar documentos con metadatos
categorias = ["ciencia", "deportes", "tecnologia"]
docs = [
    ("Artículo sobre física cuántica", "ciencia"),
    ("Química orgánica y reacciones", "ciencia"),
    ("El fútbol y sus reglas", "deportes"),
    ("Tenis: estrategias ganadoras", "deportes"),
    ("Machine learning con Python", "tecnologia"),
    ("Redes neuronales profundas", "tecnologia"),
    ("Baloncesto en los Juegos Olímpicos", "deportes"),
]

vectores = np.random.randn(len(docs), DIM).astype(np.float32)
ids_insertados = db.insert_batch(
    vectores,
    metadatas=[{"texto": t, "categoria": c} for t, c in docs]
)
print(f"DB: {db.stats()}")

# Búsqueda sin filtro
query = np.random.randn(DIM).astype(np.float32)
resultados = db.search(query, k=3)
print(f"\nTop-3 sin filtro:")
for r in resultados:
    print(f"  {r['id']}: score={r['score']:.4f} | {r['metadata']['texto'][:40]}")

# Búsqueda con filtro por categoría
resultados_ciencia = db.search(query, k=3, where={"categoria": "ciencia"})
print(f"\nTop-3 de 'ciencia':")
for r in resultados_ciencia:
    print(f"  {r['id']}: score={r['score']:.4f} | {r['metadata']['texto'][:40]}")

# CRUD
db.delete(ids_insertados[0])
print(f"\nTras delete: {db.stats()}")

nuevo_vec = np.random.randn(DIM).astype(np.float32)
db.update(ids_insertados[1], metadata={"categoria": "ciencia_avanzada"})
print(f"Metadata actualizada: {db.metadata[ids_insertados[1]]}")

np.save("outputs/m12_vectordb_demo.npy", vectores)
print("\noutputs/m12_vectordb_demo.npy guardado")
```

---

## 2. Chroma — base de datos vectorial embebida

Chroma es una base de datos vectorial en proceso (embebida), ideal para prototipado y aplicaciones locales.

```python
# chroma_demo.py
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

try:
    import chromadb
    from chromadb.config import Settings

    # --- Inicialización ---
    # Cliente en memoria (para pruebas)
    client_mem = chromadb.Client()

    # Cliente persistente (guarda en disco)
    client_persist = chromadb.PersistentClient(path="outputs/chroma_db")

    # --- Crear colección ---
    # distance_function: "cosine" | "l2" | "ip" (inner product)
    coleccion = client_mem.create_collection(
        name="documentos",
        metadata={"hnsw:space": "cosine"}
    )

    # --- Insertar documentos ---
    # Opción 1: con embeddings precalculados
    np.random.seed(42)
    N, D = 10, 128
    embeddings = np.random.randn(N, D).tolist()  # Chroma requiere lista de listas

    docs = [f"Documento número {i} sobre tema {i%3}" for i in range(N)]
    ids = [f"doc_{i}" for i in range(N)]
    metadatas = [{"tema": f"tema_{i%3}", "indice": i} for i in range(N)]

    coleccion.add(
        embeddings=embeddings,
        documents=docs,
        metadatas=metadatas,
        ids=ids
    )
    print(f"Colección: {coleccion.count()} documentos")

    # --- Búsqueda semántica ---
    query_emb = np.random.randn(1, D).tolist()
    resultados = coleccion.query(
        query_embeddings=query_emb,
        n_results=3,
        where={"tema": "tema_1"},       # filtro por metadatos
        include=["documents", "distances", "metadatas"]
    )

    print("\nTop-3 (tema_1):")
    for i in range(len(resultados["ids"][0])):
        doc_id   = resultados["ids"][0][i]
        dist     = resultados["distances"][0][i]
        doc_text = resultados["documents"][0][i]
        print(f"  {doc_id}: dist={dist:.4f} | {doc_text}")

    # --- Get por ID ---
    resultado_get = coleccion.get(ids=["doc_0", "doc_1"])
    print(f"\nGet doc_0: {resultado_get['documents'][0]}")

    # --- Update ---
    coleccion.update(
        ids=["doc_0"],
        metadatas=[{"tema": "tema_especial", "indice": 0}]
    )
    actualizado = coleccion.get(ids=["doc_0"])
    print(f"Metadata actualizada: {actualizado['metadatas'][0]}")

    # --- Delete ---
    coleccion.delete(ids=["doc_9"])
    print(f"\nTras delete: {coleccion.count()} documentos")

    # --- Colección con función de embedding integrada ---
    # Chroma puede integrar sentence-transformers directamente
    try:
        from chromadb.utils import embedding_functions
        ef = embedding_functions.SentenceTransformerEmbeddingFunction(
            model_name="all-MiniLM-L6-v2"
        )
        col_st = client_mem.create_collection(
            name="documentos_st",
            embedding_function=ef,
            metadata={"hnsw:space": "cosine"}
        )
        # Insertar solo texto — Chroma calcula embeddings automáticamente
        col_st.add(
            documents=["El gato come pescado", "El perro corre en el parque"],
            ids=["d1", "d2"]
        )
        res = col_st.query(query_texts=["animales domésticos"], n_results=2)
        print(f"\nBúsqueda por texto: {res['ids'][0]}")
    except Exception:
        print("[sentence-transformers no disponible para embedding_function]")

    print("\nChroma demo completado")

except ImportError:
    print("[chromadb no instalado] pip install chromadb")
    print("Simulando con SimpleVectorDB...")
```

---

## 3. HNSW — el índice detrás de las VectorDBs

HNSW (Hierarchical Navigable Small World) es el algoritmo de indexado ANN más usado. Aquí lo analizamos conceptualmente.

```python
# hnsw_conceptual.py
import numpy as np
import os
from typing import Set, List, Tuple

os.makedirs("outputs", exist_ok=True)

# --- Grafo NSW simplificado (1 capa) ---
class NSWGraph:
    """
    Navigable Small World Graph: versión 1-capa sin jerarquía.
    Complejidad de búsqueda: O(log N) en promedio.
    """
    def __init__(self, dim: int, M: int = 16):
        """
        M: número máximo de vecinos por nodo (controla densidad del grafo).
        Mayor M → mejor recall, más memoria y tiempo de inserción.
        """
        self.dim = dim
        self.M = M
        self.vectors: List[np.ndarray] = []
        self.graph: List[Set[int]] = []  # lista de adyacencia

    def _dist(self, a: np.ndarray, b: np.ndarray) -> float:
        """Distancia euclidiana (negativa para max-heap)."""
        return float(np.linalg.norm(a - b))

    def _greedy_search(self, query: np.ndarray,
                       entry_point: int, ef: int = 20) -> List[int]:
        """
        Búsqueda greedy en el grafo NSW.
        ef: tamaño del conjunto de candidatos a explorar.
        """
        if not self.vectors:
            return []

        visited = {entry_point}
        # Candidatos: (distancia, índice)
        candidates = [(self._dist(query, self.vectors[entry_point]), entry_point)]
        resultado = list(candidates)

        while candidates:
            # Explorar el candidato más cercano
            dist_c, c = min(candidates)
            candidates.remove((dist_c, c))

            # Si el más lejano en resultado < candidato más cercano → parar
            if resultado and max(d for d, _ in resultado) < dist_c:
                break

            for vecino in self.graph[c]:
                if vecino not in visited:
                    visited.add(vecino)
                    d_v = self._dist(query, self.vectors[vecino])
                    candidates.append((d_v, vecino))
                    resultado.append((d_v, vecino))

        resultado.sort(key=lambda x: x[0])
        return [idx for _, idx in resultado[:ef]]

    def insert(self, vector: np.ndarray):
        """Insertar un nuevo vector y conectarlo al grafo."""
        idx = len(self.vectors)
        self.vectors.append(vector.copy())
        self.graph.append(set())

        if idx == 0:
            return idx  # primer nodo, sin conexiones

        # Encontrar M vecinos más cercanos usando búsqueda greedy
        entry = 0
        candidatos = self._greedy_search(vector, entry, ef=max(self.M, 10))
        # Conectar con los M más cercanos
        vecinos = sorted(candidatos, key=lambda n: self._dist(vector, self.vectors[n]))
        for v in vecinos[:self.M]:
            self.graph[idx].add(v)
            if len(self.graph[v]) < self.M:
                self.graph[v].add(idx)   # conexión bidireccional
        return idx

    def search(self, query: np.ndarray, k: int = 5) -> List[Tuple[float, int]]:
        """Retorna los k vectores más cercanos."""
        if not self.vectors:
            return []
        entry = 0
        candidatos = self._greedy_search(query, entry, ef=max(k * 4, 50))
        with_dists = [(self._dist(query, self.vectors[i]), i) for i in candidatos]
        with_dists.sort()
        return with_dists[:k]

# --- Benchmark: NSW vs brute force ---
np.random.seed(42)
DIM, N = 32, 500
data = np.random.randn(N, DIM).astype(np.float32)

# Construir grafo
print("Construyendo NSW graph...")
grafo = NSWGraph(dim=DIM, M=8)
for i, vec in enumerate(data):
    grafo.insert(vec)

query = np.random.randn(DIM).astype(np.float32)

# Búsqueda NSW
res_nsw = grafo.search(query, k=5)
idx_nsw = [i for _, i in res_nsw]

# Brute force (ground truth)
dists_bf = np.linalg.norm(data - query, axis=1)
idx_bf = np.argsort(dists_bf)[:5].tolist()

# Recall@5
recall = len(set(idx_nsw) & set(idx_bf)) / 5
print(f"\nNSW vs Brute Force:")
print(f"NSW top-5: {idx_nsw}")
print(f"BF  top-5: {idx_bf}")
print(f"Recall@5: {recall:.2f}")

# Grado promedio del grafo
grados = [len(v) for v in grafo.graph]
print(f"Grado promedio: {np.mean(grados):.2f} (M={grafo.M})")

resultados_nsw = {
    "recall_at_5": recall,
    "avg_degree": float(np.mean(grados)),
    "n_nodes": N
}
np.save("outputs/m12_hnsw_benchmark.npy", resultados_nsw)
print("outputs/m12_hnsw_benchmark.npy guardado")
```

---

## 4. pgvector — embeddings en PostgreSQL

pgvector extiende PostgreSQL con tipos `vector` y operadores de similitud.

```python
# pgvector_demo.py
"""
Requiere: PostgreSQL con extensión pgvector instalada.
  CREATE EXTENSION vector;
"""
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# Esquema SQL de referencia (ejecutar en psql):
SCHEMA_SQL = """
-- Crear extensión
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabla de documentos con embedding
CREATE TABLE documentos (
    id          SERIAL PRIMARY KEY,
    contenido   TEXT NOT NULL,
    categoria   VARCHAR(50),
    embedding   VECTOR(768),       -- dimensión fija
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Índice HNSW para búsqueda ANN (recomendado para N > 100K)
CREATE INDEX ON documentos USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Alternativa: índice IVFFlat (menor memoria)
-- CREATE INDEX ON documentos USING ivfflat (embedding vector_cosine_ops)
--     WITH (lists = 100);

-- Insertar documento
INSERT INTO documentos (contenido, categoria, embedding)
VALUES ($1, $2, $3::vector);

-- Búsqueda por similitud coseno (los más similares)
SELECT id, contenido, categoria,
       1 - (embedding <=> $1::vector) AS similitud
FROM documentos
ORDER BY embedding <=> $1::vector
LIMIT 5;

-- Búsqueda con filtro por categoría
SELECT id, contenido,
       1 - (embedding <=> $1::vector) AS similitud
FROM documentos
WHERE categoria = 'ciencia'
ORDER BY embedding <=> $1::vector
LIMIT 5;

-- Operadores de distancia pgvector:
--   <=>  coseno (1 - similitud)
--   <->  euclidiana
--   <#>  producto interno negativo

-- Estadísticas de uso de índice
EXPLAIN ANALYZE
SELECT id FROM documentos
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;
"""

print("=== Schema pgvector ===")
print(SCHEMA_SQL[:500] + "...\n")

# Código Python con psycopg2
PYTHON_CODE = '''
import psycopg2
import numpy as np
from psycopg2.extras import execute_batch

def conectar():
    return psycopg2.connect(
        host="localhost", port=5432,
        dbname="vectordb", user="postgres", password="password"
    )

def insertar_documentos(conn, docs: list, embeddings: np.ndarray):
    """
    docs: lista de (contenido, categoria)
    embeddings: np.ndarray (N, D)
    """
    with conn.cursor() as cur:
        datos = [
            (texto, cat, embedding.tolist())
            for (texto, cat), embedding in zip(docs, embeddings)
        ]
        execute_batch(
            cur,
            "INSERT INTO documentos (contenido, categoria, embedding) "
            "VALUES (%s, %s, %s::vector)",
            datos,
            page_size=100
        )
    conn.commit()

def buscar_similares(conn, query_emb: np.ndarray, k: int = 5,
                     categoria: str = None) -> list:
    with conn.cursor() as cur:
        q_str = str(query_emb.tolist())
        if categoria:
            cur.execute(
                "SELECT id, contenido, 1 - (embedding <=> %s::vector) "
                "FROM documentos WHERE categoria = %s "
                "ORDER BY embedding <=> %s::vector LIMIT %s",
                (q_str, categoria, q_str, k)
            )
        else:
            cur.execute(
                "SELECT id, contenido, 1 - (embedding <=> %s::vector) "
                "FROM documentos "
                "ORDER BY embedding <=> %s::vector LIMIT %s",
                (q_str, q_str, k)
            )
        return cur.fetchall()
'''

print("=== Código Python con psycopg2 ===")
print(PYTHON_CODE[:400] + "...\n")

# Guardar schema como referencia
with open("outputs/m12_pgvector_schema.sql", "w") as f:
    f.write(SCHEMA_SQL)
print("outputs/m12_pgvector_schema.sql guardado")
```

---

## 5. Comparativa de VectorDBs

```python
# comparativa_vectordb.py
import numpy as np
import time
import os

os.makedirs("outputs", exist_ok=True)

# --- Benchmark con implementación interna ---
from typing import List, Dict, Any
import json

# Reutilizar SimpleVectorDB del apartado 1
class SimpleVectorDB:
    def __init__(self, dim, metric="cosine"):
        self.dim = dim
        self.metric = metric
        self.vectors = {}
        self.metadata = {}
        self._counter = 0

    def insert(self, vec, meta=None, doc_id=None):
        if self.metric == "cosine":
            vec = vec / (np.linalg.norm(vec) + 1e-9)
        doc_id = doc_id or f"id_{self._counter}"
        self._counter += 1
        self.vectors[doc_id] = vec.astype(np.float32)
        self.metadata[doc_id] = meta or {}
        return doc_id

    def search(self, q, k=5, where=None):
        if self.metric == "cosine":
            q = q / (np.linalg.norm(q) + 1e-9)
        ids = [i for i in self.vectors
               if not where or all(self.metadata[i].get(k_) == v
                                   for k_, v in where.items())]
        if not ids:
            return []
        vecs = np.stack([self.vectors[i] for i in ids])
        sims = vecs @ q
        top = np.argsort(sims)[::-1][:k]
        return [{"id": ids[i], "score": float(sims[i])} for i in top]

    def __len__(self):
        return len(self.vectors)

# Benchmark de inserción y búsqueda
def benchmark(name: str, n_docs: int, dim: int, n_queries: int = 100):
    np.random.seed(42)
    db = SimpleVectorDB(dim=dim, metric="cosine")

    # Datos
    vecs = np.random.randn(n_docs, dim).astype(np.float32)
    queries = np.random.randn(n_queries, dim).astype(np.float32)

    # Inserción
    t0 = time.perf_counter()
    for i, v in enumerate(vecs):
        db.insert(v, {"cat": str(i % 5)})
    t_insert = time.perf_counter() - t0

    # Búsqueda sin filtro
    t0 = time.perf_counter()
    for q in queries:
        db.search(q, k=10)
    t_search = time.perf_counter() - t0

    # Búsqueda con filtro
    t0 = time.perf_counter()
    for q in queries:
        db.search(q, k=5, where={"cat": "2"})
    t_filtered = time.perf_counter() - t0

    return {
        "sistema": name,
        "n_docs": n_docs,
        "dim": dim,
        "insert_total_s": round(t_insert, 4),
        "insert_per_doc_ms": round(t_insert / n_docs * 1000, 3),
        "search_total_s": round(t_search, 4),
        "search_per_query_ms": round(t_search / n_queries * 1000, 3),
        "filtered_search_ms": round(t_filtered / n_queries * 1000, 3),
    }

configs = [
    (500, 64),
    (1000, 128),
    (2000, 256),
]

resultados = []
for n, d in configs:
    res = benchmark(f"SimpleVectorDB ({n} docs, dim={d})", n, d)
    resultados.append(res)
    print(f"\n{res['sistema']}:")
    print(f"  Insert:         {res['insert_per_doc_ms']:.3f} ms/doc")
    print(f"  Search:         {res['search_per_query_ms']:.3f} ms/query")
    print(f"  Filtered:       {res['filtered_search_ms']:.3f} ms/query")

# Tabla comparativa con características
TABLA_COMPARATIVA = """
| Feature            | Chroma          | Pinecone        | pgvector        | Weaviate        |
|--------------------|-----------------|-----------------|-----------------|-----------------|
| Tipo               | Embebido/Cloud  | Cloud managed   | PostgreSQL ext  | Cloud/Self-host |
| Índice ANN         | HNSW            | HNSW            | HNSW / IVFFlat  | HNSW            |
| Filtro metadatos   | Sí              | Sí              | SQL completo    | Sí              |
| Escalabilidad      | Millones        | Mil millones    | Millones*       | Millones        |
| SQL joins          | No              | No              | Sí (PostgreSQL) | No              |
| Setup              | pip install     | API key         | PostgreSQL      | Docker          |
| Mejor para         | Prototipado     | Producción      | Datos mixtos    | Multimodal      |
| Persistencia       | Disco local     | Cloud           | PostgreSQL      | Cloud/Disco     |
"""
print(TABLA_COMPARATIVA)

with open("outputs/m12_comparativa.json", "w") as f:
    json.dump(resultados, f, indent=2)
print("outputs/m12_comparativa.json guardado")
```

---

## 6. Búsqueda Híbrida — Vectorial + BM25

La búsqueda híbrida combina similitud semántica (embeddings) con relevancia léxica (BM25).

```python
# busqueda_hibrida.py
import numpy as np
import math
from typing import List, Dict
import os

os.makedirs("outputs", exist_ok=True)

# --- BM25 ---
class BM25:
    """
    BM25 (Best Match 25): modelo probabilístico de recuperación de información.
    score(q, d) = Σ_t IDF(t) * (tf(t,d) * (k1+1)) / (tf(t,d) + k1*(1-b+b*|d|/avgdl))
    """
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        self.k1 = k1
        self.b = b
        self.corpus: List[List[str]] = []
        self.idf: Dict[str, float] = {}
        self.avgdl = 0.0

    def fit(self, corpus: List[str]):
        """Tokenización simple y cálculo de IDF."""
        tokenized = [doc.lower().split() for doc in corpus]
        self.corpus = tokenized
        N = len(tokenized)
        self.avgdl = np.mean([len(d) for d in tokenized])

        # IDF: log((N - df + 0.5) / (df + 0.5) + 1)
        df: Dict[str, int] = {}
        for doc in tokenized:
            for term in set(doc):
                df[term] = df.get(term, 0) + 1

        self.idf = {
            term: math.log((N - freq + 0.5) / (freq + 0.5) + 1)
            for term, freq in df.items()
        }

    def score(self, query: str, doc_idx: int) -> float:
        """Score BM25 para query vs documento."""
        tokens_q = query.lower().split()
        doc = self.corpus[doc_idx]
        dl = len(doc)

        score = 0.0
        for term in tokens_q:
            tf = doc.count(term)
            idf = self.idf.get(term, 0.0)
            numerator = tf * (self.k1 + 1)
            denominator = tf + self.k1 * (1 - self.b + self.b * dl / self.avgdl)
            score += idf * numerator / (denominator + 1e-9)
        return score

    def search(self, query: str, k: int = 5) -> List[Dict]:
        scores = [self.score(query, i) for i in range(len(self.corpus))]
        top_k = np.argsort(scores)[::-1][:k]
        return [{"idx": int(i), "bm25_score": scores[i]} for i in top_k]

# --- Búsqueda híbrida con Reciprocal Rank Fusion ---
def reciprocal_rank_fusion(ranked_lists: List[List[int]], k: int = 60) -> List[int]:
    """
    RRF: combina múltiples rankings sin necesidad de normalización de scores.
    score(d) = Σ 1 / (k + rank(d))
    k=60 es el valor estándar por defecto.
    """
    scores: Dict[int, float] = {}
    for ranked in ranked_lists:
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

def busqueda_hibrida(query_text: str, query_emb: np.ndarray,
                     corpus: List[str], corpus_emb: np.ndarray,
                     bm25: BM25, k: int = 5, alpha: float = 0.5) -> List[Dict]:
    """
    Combina BM25 y similitud coseno usando normalización min-max.
    alpha: peso del score vectorial (1-alpha = BM25)
    """
    N = len(corpus)

    # Scores BM25 (léxico)
    bm25_scores = np.array([bm25.score(query_text, i) for i in range(N)])

    # Scores vectoriales (semántico)
    q_norm = query_emb / (np.linalg.norm(query_emb) + 1e-9)
    emb_norm = corpus_emb / (np.linalg.norm(corpus_emb, axis=1, keepdims=True) + 1e-9)
    vec_scores = emb_norm @ q_norm

    # Normalizar ambos scores a [0, 1]
    def min_max_norm(x):
        r = x.max() - x.min()
        return (x - x.min()) / (r + 1e-9)

    bm25_norm = min_max_norm(bm25_scores)
    vec_norm = min_max_norm(vec_scores)

    # Score combinado
    hybrid_scores = alpha * vec_norm + (1 - alpha) * bm25_norm

    top_k = np.argsort(hybrid_scores)[::-1][:k]
    return [
        {
            "idx": int(i),
            "texto": corpus[i][:60],
            "bm25_norm": float(bm25_norm[i]),
            "vec_norm": float(vec_norm[i]),
            "hybrid": float(hybrid_scores[i])
        }
        for i in top_k
    ]

# --- Demo ---
corpus = [
    "Los transformers revolutionaron el procesamiento de lenguaje natural",
    "BERT es un modelo bidireccional preentrenado",
    "GPT usa arquitectura decoder-only para generación de texto",
    "El fútbol es el deporte más popular del mundo",
    "Messi ganó el Mundial con Argentina en Qatar 2022",
    "Las redes neuronales recurrentes procesaban secuencias antes de los transformers",
    "Attention is all you need: el paper original del transformer",
    "PyTorch y TensorFlow son los frameworks de deep learning más usados",
]

np.random.seed(42)
DIM = 64
corpus_emb = np.random.randn(len(corpus), DIM).astype(np.float32)

bm25 = BM25()
bm25.fit(corpus)

query_text = "transformers para procesamiento de lenguaje"
query_emb = np.random.randn(DIM).astype(np.float32)

resultados = busqueda_hibrida(query_text, query_emb, corpus, corpus_emb, bm25, k=3)
print("Búsqueda híbrida (alpha=0.5):")
for r in resultados:
    print(f"  [{r['idx']}] hybrid={r['hybrid']:.4f} | bm25={r['bm25_norm']:.3f} | vec={r['vec_norm']:.3f}")
    print(f"       {r['texto']}")

# RRF
bm25_rank = [r["idx"] for r in bm25.search(query_text, k=len(corpus))]
vec_sims = corpus_emb @ (query_emb / np.linalg.norm(query_emb))
vec_rank = np.argsort(vec_sims)[::-1].tolist()
fused = reciprocal_rank_fusion([bm25_rank, vec_rank])

print(f"\nRRF top-3: {fused[:3]}")
for idx in fused[:3]:
    print(f"  [{idx}] {corpus[idx][:60]}")

np.save("outputs/m12_hybrid_search_demo.npy",
        np.array([r["hybrid"] for r in resultados]))
print("\noutputs/m12_hybrid_search_demo.npy guardado")
```

---

## 7. Checkpoint — Pruebas unitarias

```python
# checkpoint_m12.py
"""
Checkpoint M12: Bases de Datos Vectoriales
Ejecutar con: python checkpoint_m12.py
"""
import numpy as np
import time
import sys
import os

sys.path.insert(0, os.path.dirname(__file__))
os.makedirs("outputs", exist_ok=True)

# ── Clases locales ────────────────────────────────────────────────────────────
class SimpleVectorDB:
    def __init__(self, dim, metric="cosine"):
        self.dim = dim
        self.metric = metric
        self.vectors = {}
        self.metadata = {}
        self._counter = 0

    def insert(self, vec, meta=None, doc_id=None):
        if self.metric == "cosine":
            vec = vec / (np.linalg.norm(vec) + 1e-9)
        doc_id = doc_id or f"id_{self._counter}"
        self._counter += 1
        self.vectors[doc_id] = vec.astype(np.float32)
        self.metadata[doc_id] = meta or {}
        return doc_id

    def search(self, q, k=5, where=None):
        if self.metric == "cosine":
            q = q / (np.linalg.norm(q) + 1e-9)
        ids = [i for i in self.vectors
               if not where or all(self.metadata[i].get(k_) == v
                                   for k_, v in where.items())]
        if not ids:
            return []
        vecs = np.stack([self.vectors[i] for i in ids])
        sims = vecs @ q
        top = np.argsort(sims)[::-1][:k]
        return [{"id": ids[i], "score": float(sims[i]), "metadata": self.metadata[ids[i]]} for i in top]

    def delete(self, doc_id):
        self.vectors.pop(doc_id, None)
        self.metadata.pop(doc_id, None)

    def __len__(self):
        return len(self.vectors)

import math

class BM25:
    def __init__(self, k1=1.5, b=0.75):
        self.k1 = k1; self.b = b
        self.corpus = []; self.idf = {}; self.avgdl = 0.0

    def fit(self, corpus):
        tokenized = [d.lower().split() for d in corpus]
        self.corpus = tokenized
        N = len(tokenized)
        self.avgdl = np.mean([len(d) for d in tokenized])
        df = {}
        for doc in tokenized:
            for t in set(doc):
                df[t] = df.get(t, 0) + 1
        self.idf = {t: math.log((N - f + 0.5) / (f + 0.5) + 1) for t, f in df.items()}

    def score(self, query, doc_idx):
        tokens_q = query.lower().split()
        doc = self.corpus[doc_idx]; dl = len(doc)
        s = 0.0
        for term in tokens_q:
            tf = doc.count(term); idf = self.idf.get(term, 0.0)
            s += idf * tf * (self.k1 + 1) / (tf + self.k1 * (1 - self.b + self.b * dl / (self.avgdl + 1e-9)) + 1e-9)
        return s

# ── Tests ─────────────────────────────────────────────────────────────────────
def test_insert_and_search():
    """T1: insertar y recuperar vectores correctamente"""
    np.random.seed(0)
    db = SimpleVectorDB(dim=32, metric="cosine")
    vecs = np.random.randn(10, 32).astype(np.float32)
    for i, v in enumerate(vecs):
        db.insert(v, meta={"idx": i})
    assert len(db) == 10, "Debe haber 10 vectores"

    # El vector más similar a sí mismo debe ser recuperado primero
    q = vecs[5].copy()
    resultados = db.search(q, k=3)
    assert resultados[0]["score"] > 0.99, \
        f"Similitud consigo mismo debe ser ~1, got {resultados[0]['score']}"
    print(f"T1 PASS: insert+search (top score={resultados[0]['score']:.4f})")

def test_filtro_metadatos():
    """T2: la búsqueda con filtro de metadatos solo retorna docs del filtro"""
    np.random.seed(1)
    db = SimpleVectorDB(dim=32, metric="cosine")
    for i in range(20):
        v = np.random.randn(32).astype(np.float32)
        db.insert(v, meta={"cat": "A" if i < 10 else "B"})

    q = np.random.randn(32).astype(np.float32)
    resultados = db.search(q, k=5, where={"cat": "A"})
    assert len(resultados) == 5, f"Esperados 5, got {len(resultados)}"
    for r in resultados:
        assert r["metadata"]["cat"] == "A", "Todos los resultados deben ser cat=A"
    print(f"T2 PASS: filtro metadatos correcto ({len(resultados)} resultados cat=A)")

def test_delete_reduce_count():
    """T3: delete reduce el número de documentos"""
    np.random.seed(2)
    db = SimpleVectorDB(dim=16, metric="cosine")
    ids = [db.insert(np.random.randn(16).astype(np.float32), doc_id=f"doc_{i}")
           for i in range(5)]
    assert len(db) == 5
    db.delete(ids[2])
    assert len(db) == 4, f"Tras delete, esperados 4, got {len(db)}"
    # El documento eliminado no debe aparecer en búsqueda
    q = np.random.randn(16).astype(np.float32)
    res = db.search(q, k=5)
    res_ids = [r["id"] for r in res]
    assert ids[2] not in res_ids, "Documento eliminado no debe aparecer"
    print(f"T3 PASS: delete correcto ({len(db)} docs restantes)")

def test_bm25_score_positivo_para_termino_exacto():
    """T4: BM25 da score > 0 cuando query contiene términos del documento"""
    corpus = [
        "machine learning deep neural networks",
        "el fútbol es un deporte popular",
        "python programming language syntax",
    ]
    bm25 = BM25()
    bm25.fit(corpus)

    # El doc 0 debe tener score > 0 para query sobre ML
    s0 = bm25.score("machine learning", 0)
    s1 = bm25.score("machine learning", 1)
    assert s0 > 0, f"Score para doc relevante debe ser > 0, got {s0}"
    assert s0 > s1, f"Doc relevante ({s0:.4f}) debe tener mayor score que irrelevante ({s1:.4f})"
    print(f"T4 PASS: BM25 score doc_relevante={s0:.4f} > doc_irrelevante={s1:.4f}")

def test_busqueda_hibrida_combina_scores():
    """T5: la búsqueda híbrida produce scores en [0,1] y rankea por score combinado"""
    import math

    np.random.seed(42)
    N, D = 8, 32
    corpus = [f"documento sobre tema {i % 4}" for i in range(N)]
    corpus_emb = np.random.randn(N, D).astype(np.float32)

    bm25 = BM25(); bm25.fit(corpus)
    query_text = "documento tema"
    query_emb = np.random.randn(D).astype(np.float32)

    # Calcular scores
    bm25_scores = np.array([bm25.score(query_text, i) for i in range(N)])
    q_norm = query_emb / (np.linalg.norm(query_emb) + 1e-9)
    emb_norm = corpus_emb / (np.linalg.norm(corpus_emb, axis=1, keepdims=True) + 1e-9)
    vec_scores = emb_norm @ q_norm

    def min_max_norm(x):
        r = x.max() - x.min()
        return (x - x.min()) / (r + 1e-9)

    alpha = 0.5
    hybrid = alpha * min_max_norm(vec_scores) + (1 - alpha) * min_max_norm(bm25_scores)

    assert hybrid.min() >= 0.0 and hybrid.max() <= 1.0, \
        f"Scores deben estar en [0,1], got [{hybrid.min():.4f}, {hybrid.max():.4f}]"
    assert np.all(np.diff(np.argsort(hybrid)[::-1]) != 0), \
        "Los índices ordenados deben ser únicos"
    print(f"T5 PASS: hybrid scores en [{hybrid.min():.4f}, {hybrid.max():.4f}]")

def test_cosine_norm_implica_producto_punto():
    """T6: con vectores L2-normalizados, similitud coseno = producto punto"""
    np.random.seed(3)
    D = 64
    a = np.random.randn(D).astype(np.float32)
    b = np.random.randn(D).astype(np.float32)

    a_n = a / np.linalg.norm(a)
    b_n = b / np.linalg.norm(b)

    # Similitud coseno exacta
    coseno = np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
    # Producto punto entre normalizados
    dot_norm = np.dot(a_n, b_n)

    assert abs(coseno - dot_norm) < 1e-5, \
        f"Coseno ({coseno:.6f}) ≠ dot_norm ({dot_norm:.6f})"
    print(f"T6 PASS: cos={coseno:.6f} == dot_norm={dot_norm:.6f}")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_insert_and_search,
        test_filtro_metadatos,
        test_delete_reduce_count,
        test_bm25_score_positivo_para_termino_exacto,
        test_busqueda_hibrida_combina_scores,
        test_cosine_norm_implica_producto_punto,
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
        print("M12 COMPLETADO")
    else:
        print("Revisar implementaciones")
```

---

## Resumen

| Base de datos | Índice | Persistencia | Filtros | Mejor para |
|---|---|---|---|---|
| Chroma | HNSW | Local/Cloud | Metadatos | Prototipos, apps locales |
| Pinecone | HNSW | Cloud | Metadata | Producción escalable |
| pgvector | HNSW/IVF | PostgreSQL | SQL completo | Datos relacionales + vectores |
| Weaviate | HNSW | Cloud/Self | GraphQL | Multimodal, grafo |
| Qdrant | HNSW | Cloud/Self | JSON | Producción, filtros complejos |

**Algoritmos ANN:**
- `IndexFlatIP`: exacto, O(N), para N < 100K
- `HNSW`: grafo jerárquico, O(log N) búsqueda, alto recall
- `IVFFlat`: Voronoi cells, configurable speed/recall con `nprobe`
- `IVFPQ`: IVF + Product Quantization, mínima memoria, N > 10M
