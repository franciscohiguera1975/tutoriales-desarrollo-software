# Módulo 13 — Pipeline RAG Básico

**Objetivo:** Implementar un pipeline completo de Retrieval-Augmented Generation (RAG) desde cero: ingesta de documentos, indexado de embeddings, recuperación por similitud, y generación aumentada con contexto recuperado.

**Herramientas:** `numpy`, `torch`, `transformers`, `chromadb`, `sentence-transformers`, `openai`/`anthropic`

**Prerequisito:** M11 (Embeddings), M12 (VectorDBs)

---

## 1. Arquitectura RAG

RAG combina recuperación de información con generación de texto:

```
Pregunta → Embeder query → Buscar en VectorDB → Recuperar K chunks
         → Construir prompt[contexto + pregunta] → LLM → Respuesta
```

**Ventajas sobre fine-tuning:**
- Sin necesidad de re-entrenar cuando cambian los datos
- Fuentes citables y verificables
- Actualización en tiempo real (solo reindexar)
- Menor coste computacional

```python
# rag_arquitectura.py
import numpy as np
import os
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

@dataclass
class Documento:
    """Unidad básica de información en RAG."""
    id: str
    contenido: str
    metadata: Dict = field(default_factory=dict)
    embedding: Optional[np.ndarray] = None

@dataclass
class Chunk:
    """Fragmento de documento después del chunking."""
    id: str
    contenido: str
    doc_id: str
    inicio: int    # posición de carácter en el doc original
    fin: int
    metadata: Dict = field(default_factory=dict)
    embedding: Optional[np.ndarray] = None

@dataclass
class ResultadoRAG:
    """Resultado completo del pipeline RAG."""
    pregunta: str
    chunks_recuperados: List[Chunk]
    scores_recuperacion: List[float]
    prompt_construido: str
    respuesta: str
    tokens_contexto: int

print("Arquitectura RAG definida")
print(f"Componentes: Documento, Chunk, ResultadoRAG")
print(f"Pipeline: Ingest → Chunk → Embed → Index → Retrieve → Generate")

# Guardar diagrama conceptual
diagrama = """
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  INGESTA     │ --> │  CHUNKING    │ --> │   EMBEDDING      │
│  Cargar docs │     │  Dividir     │     │   Calcular vecs  │
└──────────────┘     └──────────────┘     └──────────────────┘
                                                    │
                                                    v
                                          ┌──────────────────┐
                                          │   VECTOR DB      │
                                          │   Indexar chunks │
                                          └──────────────────┘
                                                    │
┌──────────────┐     ┌──────────────┐              │
│  GENERACIÓN  │ <-- │  PROMPT      │ <-- Retrieve │
│  LLM responde│     │  + Contexto  │     Top-K    │
└──────────────┘     └──────────────┘              │
                                          ┌──────────────────┐
                                          │   QUERY EMBED    │
                                          │   Pregunta → vec │
                                          └──────────────────┘
"""
with open("outputs/m13_rag_diagrama.txt", "w") as f:
    f.write(diagrama)
print("outputs/m13_rag_diagrama.txt guardado")
```

---

## 2. Chunking — Estrategias de fragmentación

El chunking divide documentos largos en fragmentos que caben en el contexto del LLM.

```python
# chunking_estrategias.py
import numpy as np
import re
from typing import List
import os

os.makedirs("outputs", exist_ok=True)

# --- Texto de ejemplo ---
TEXTO_LARGO = """
Los transformers revolucionaron el procesamiento de lenguaje natural en 2017 con el paper
"Attention is All You Need". A diferencia de las RNNs, los transformers procesan toda la
secuencia en paralelo usando el mecanismo de atención.

El mecanismo de self-attention permite que cada token "atienda" a todos los demás tokens
de la secuencia. Esto captura dependencias de largo alcance que las RNNs perdían por el
problema de los gradientes que desaparecen o explotan.

BERT fue el primer modelo que aprovechó plenamente la arquitectura bidireccional del
transformer para preentrenamiento. Usando Masked Language Modeling, BERT aprendió
representaciones contextuales muy ricas que mejoraron el estado del arte en decenas
de benchmarks de NLP.

GPT adoptó un enfoque diferente: solo el decoder del transformer, con un objetivo de
predicción del siguiente token. Esta simplicidad resultó ser increíblemente escalable:
GPT-2, GPT-3, y GPT-4 demostraron que escalar el modelo y los datos produce capacidades
emergentes sorprendentes.

LLaMA y sus derivados (Mistral, Llama 2, Llama 3) democratizaron los LLMs de alta calidad
permitiendo ejecutarlos en hardware de consumo. Con técnicas como LoRA y QLoRA, es posible
hacer fine-tuning de modelos de 7B a 70B parámetros en GPUs de consumidor.
""".strip()

# --- Estrategia 1: Fixed-size chunking ---
def chunk_fixed_size(text: str, chunk_size: int = 200,
                     overlap: int = 50) -> List[str]:
    """
    Divide el texto en chunks de tamaño fijo con solapamiento.
    overlap: número de caracteres que se solapan entre chunks consecutivos.
    Ventaja: simple, predecible.
    Desventaja: puede cortar en medio de oraciones.
    """
    chunks = []
    start = 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks

# --- Estrategia 2: Sentence-based chunking ---
def chunk_sentences(text: str, max_sentences: int = 3,
                    overlap_sentences: int = 1) -> List[str]:
    """
    Divide en grupos de oraciones completas con solapamiento.
    Ventaja: preserva unidades semánticas completas.
    """
    # Dividir por puntos seguidos de mayúscula (simplificado)
    oraciones = re.split(r'(?<=[.!?])\s+(?=[A-ZÁÉÍÓÚ])', text)
    oraciones = [o.strip() for o in oraciones if o.strip()]

    chunks = []
    i = 0
    while i < len(oraciones):
        grupo = oraciones[i:i + max_sentences]
        chunks.append(" ".join(grupo))
        i += max_sentences - overlap_sentences
    return chunks

# --- Estrategia 3: Semantic chunking ---
def chunk_semantico(text: str, paragraphs_per_chunk: int = 1) -> List[str]:
    """
    Divide por párrafos (doble newline).
    Ventaja: respeta límites temáticos naturales del documento.
    """
    parrafos = [p.strip() for p in re.split(r'\n\n+', text) if p.strip()]
    chunks = []
    for i in range(0, len(parrafos), paragraphs_per_chunk):
        grupo = parrafos[i:i + paragraphs_per_chunk]
        chunks.append("\n\n".join(grupo))
    return chunks

# --- Estrategia 4: Recursive chunking (estilo LangChain) ---
def chunk_recursivo(text: str, chunk_size: int = 300,
                    overlap: int = 50, separadores: List[str] = None) -> List[str]:
    """
    Divide recursivamente usando separadores en orden de preferencia:
    \\n\\n → \\n → '. ' → ' ' → carácter a carácter
    """
    if separadores is None:
        separadores = ["\n\n", "\n", ". ", " ", ""]

    def _split(texto, seps):
        if len(texto) <= chunk_size:
            return [texto] if texto.strip() else []
        if not seps:
            # División por caracteres con overlap
            result = []
            start = 0
            while start < len(texto):
                result.append(texto[start:start + chunk_size])
                start += chunk_size - overlap
            return result

        sep = seps[0]
        partes = texto.split(sep)
        chunks_resultado = []
        actual = ""

        for parte in partes:
            candidato = (actual + sep + parte).strip() if actual else parte
            if len(candidato) <= chunk_size:
                actual = candidato
            else:
                if actual:
                    chunks_resultado.append(actual)
                if len(parte) > chunk_size:
                    chunks_resultado.extend(_split(parte, seps[1:]))
                    actual = ""
                else:
                    actual = parte

        if actual:
            chunks_resultado.append(actual)
        return chunks_resultado

    return _split(text, separadores)

# --- Comparación ---
print("=== Comparación de estrategias de chunking ===\n")

estrategias = {
    "Fixed (200, overlap=50)": chunk_fixed_size(TEXTO_LARGO, 200, 50),
    "Sentences (3, overlap=1)": chunk_sentences(TEXTO_LARGO, 3, 1),
    "Semántico (párrafos)": chunk_semantico(TEXTO_LARGO, 1),
    "Recursivo (300, overlap=50)": chunk_recursivo(TEXTO_LARGO, 300, 50),
}

for nombre, chunks in estrategias.items():
    longitudes = [len(c) for c in chunks]
    print(f"{nombre}:")
    print(f"  Número de chunks: {len(chunks)}")
    print(f"  Long. promedio: {np.mean(longitudes):.0f} chars")
    print(f"  Long. mín/máx: {min(longitudes)}/{max(longitudes)}")
    print(f"  Primer chunk: {chunks[0][:80]}...")
    print()

# Guardar chunks de referencia
chunks_ref = chunk_recursivo(TEXTO_LARGO, 300, 50)
import json
with open("outputs/m13_chunks_ref.json", "w", encoding="utf-8") as f:
    json.dump(chunks_ref, f, ensure_ascii=False, indent=2)
print(f"outputs/m13_chunks_ref.json guardado ({len(chunks_ref)} chunks)")
```

---

## 3. Pipeline RAG completo

```python
# rag_pipeline.py
import numpy as np
import json
import os
from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass, field

os.makedirs("outputs", exist_ok=True)

# --- Embedder simulado (en producción: sentence-transformers) ---
class EmbedderSimulado:
    """Embedder determinístico para pruebas sin GPU."""
    def __init__(self, dim: int = 64, seed: int = 42):
        self.dim = dim
        np.random.seed(seed)
        # Vocabulario de referencia para embeddings reproducibles
        self._rng = np.random.RandomState(seed)

    def _texto_a_vec(self, texto: str) -> np.ndarray:
        """Hash determinístico del texto → vector."""
        seed_val = sum(ord(c) * (i + 1) for i, c in enumerate(texto[:200]))
        rng = np.random.RandomState(seed_val % (2**31))
        vec = rng.randn(self.dim).astype(np.float32)
        return vec / (np.linalg.norm(vec) + 1e-9)

    def encode(self, textos: List[str]) -> np.ndarray:
        return np.stack([self._texto_a_vec(t) for t in textos])

# --- VectorStore ---
class VectorStore:
    """Almacén de vectores con búsqueda por similitud coseno."""
    def __init__(self):
        self.ids: List[str] = []
        self.vectors: List[np.ndarray] = []
        self.metadata: List[Dict] = []
        self.textos: List[str] = []

    def add(self, ids, vectors, textos, metadatas=None):
        metadatas = metadatas or [{} for _ in ids]
        for i, (id_, vec, txt, meta) in enumerate(zip(ids, vectors, textos, metadatas)):
            self.ids.append(id_)
            self.vectors.append(vec / (np.linalg.norm(vec) + 1e-9))
            self.textos.append(txt)
            self.metadata.append(meta)

    def search(self, query_vec: np.ndarray, k: int = 5) -> List[Dict]:
        if not self.vectors:
            return []
        q = query_vec / (np.linalg.norm(query_vec) + 1e-9)
        vecs = np.stack(self.vectors)
        sims = vecs @ q
        top_k = np.argsort(sims)[::-1][:k]
        return [
            {
                "id": self.ids[i],
                "texto": self.textos[i],
                "score": float(sims[i]),
                "metadata": self.metadata[i]
            }
            for i in top_k
        ]

    def __len__(self):
        return len(self.ids)

# --- Pipeline RAG completo ---
class RAGPipeline:
    """
    Pipeline RAG: Ingest → Chunk → Embed → Store → Retrieve → Generate
    """
    def __init__(self, embedder, llm_fn=None,
                 chunk_size: int = 300, chunk_overlap: int = 50,
                 top_k: int = 3, max_context_chars: int = 1500):
        self.embedder = embedder
        self.llm_fn = llm_fn or self._llm_simulado
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.top_k = top_k
        self.max_context_chars = max_context_chars
        self.store = VectorStore()
        self.chunks_index: Dict[str, Dict] = {}

    # --- INGESTA ---
    def _chunking(self, texto: str, doc_id: str) -> List[Dict]:
        """Chunking recursivo."""
        separadores = ["\n\n", "\n", ". ", " "]
        resultado = []
        partes = texto.split("\n\n")
        acumulado = ""

        for parte in partes:
            candidato = (acumulado + "\n\n" + parte).strip() if acumulado else parte
            if len(candidato) <= self.chunk_size:
                acumulado = candidato
            else:
                if acumulado:
                    resultado.append(acumulado)
                acumulado = parte

        if acumulado:
            resultado.append(acumulado)

        chunks = []
        for i, contenido in enumerate(resultado):
            chunk_id = f"{doc_id}_chunk_{i}"
            chunks.append({
                "id": chunk_id,
                "contenido": contenido,
                "doc_id": doc_id,
                "chunk_idx": i
            })
        return chunks

    def agregar_documentos(self, documentos: List[Dict]):
        """
        documentos: lista de {"id": str, "contenido": str, "metadata": dict}
        """
        todos_chunks = []
        for doc in documentos:
            chunks = self._chunking(doc["contenido"], doc["id"])
            for chunk in chunks:
                chunk["metadata"] = {**doc.get("metadata", {}),
                                      "doc_id": doc["id"]}
            todos_chunks.extend(chunks)

        if not todos_chunks:
            return

        # Calcular embeddings en batch
        textos = [c["contenido"] for c in todos_chunks]
        embeddings = self.embedder.encode(textos)

        # Añadir al store
        self.store.add(
            ids=[c["id"] for c in todos_chunks],
            vectors=list(embeddings),
            textos=textos,
            metadatas=[c["metadata"] for c in todos_chunks]
        )

        for chunk in todos_chunks:
            self.chunks_index[chunk["id"]] = chunk

        print(f"  Indexados: {len(todos_chunks)} chunks de {len(documentos)} docs")

    # --- RETRIEVAL ---
    def recuperar(self, pregunta: str) -> Tuple[List[Dict], np.ndarray]:
        """Embeder query y recuperar top-K chunks."""
        q_emb = self.embedder.encode([pregunta])[0]
        resultados = self.store.search(q_emb, k=self.top_k)
        return resultados, q_emb

    # --- CONSTRUCCIÓN DE PROMPT ---
    def construir_prompt(self, pregunta: str,
                         chunks_recuperados: List[Dict]) -> str:
        """
        Prompt RAG estándar con contexto numerado.
        """
        contexto_parts = []
        chars_total = 0

        for i, chunk in enumerate(chunks_recuperados, start=1):
            texto = chunk["texto"]
            if chars_total + len(texto) > self.max_context_chars:
                break
            contexto_parts.append(f"[Fuente {i}]\n{texto}")
            chars_total += len(texto)

        contexto = "\n\n".join(contexto_parts)

        prompt = f"""Eres un asistente que responde preguntas basándose SOLO en el contexto proporcionado.
Si la información no está en el contexto, di "No tengo información suficiente".

CONTEXTO:
{contexto}

PREGUNTA: {pregunta}

RESPUESTA:"""
        return prompt

    # --- LLM SIMULADO ---
    def _llm_simulado(self, prompt: str) -> str:
        """LLM de prueba: extrae la primera oración relevante del contexto."""
        # Buscar el contexto entre las marcas
        if "CONTEXTO:" in prompt and "PREGUNTA:" in prompt:
            ctx_start = prompt.index("CONTEXTO:") + 9
            ctx_end = prompt.index("PREGUNTA:")
            contexto = prompt[ctx_start:ctx_end].strip()
            # Retornar primera oración del contexto (simulación)
            primera = contexto.split(".")[0] if "." in contexto else contexto[:100]
            return f"[Respuesta simulada] {primera.strip()}."
        return "[Respuesta simulada] Sin contexto disponible."

    # --- QUERY ---
    def query(self, pregunta: str) -> Dict:
        """Pipeline completo: pregunta → respuesta."""
        # Recuperar
        chunks, q_emb = self.recuperar(pregunta)

        # Construir prompt
        prompt = self.construir_prompt(pregunta, chunks)

        # Generar
        respuesta = self.llm_fn(prompt)

        return {
            "pregunta": pregunta,
            "chunks_recuperados": chunks,
            "prompt": prompt,
            "respuesta": respuesta,
            "n_chunks": len(chunks),
            "score_promedio": float(np.mean([c["score"] for c in chunks])) if chunks else 0.0
        }

# --- Demo ---
embedder = EmbedderSimulado(dim=64)
rag = RAGPipeline(embedder, top_k=3, chunk_size=250)

# Corpus de ejemplo
documentos = [
    {
        "id": "doc_transformers",
        "contenido": """Los transformers son la arquitectura dominante en NLP desde 2017.
El mecanismo de atención permite procesar secuencias en paralelo.
BERT usa bidireccionalidad y MLM para preentrenamiento.
GPT usa un decoder causal para generación de texto.
LLaMA democratizó los LLMs de alta calidad.""",
        "metadata": {"fuente": "guia_nlp.txt", "tema": "LLMs"}
    },
    {
        "id": "doc_rag",
        "contenido": """RAG combina recuperación de información con generación de texto.
El pipeline incluye: ingesta, chunking, embedding, indexado y generación.
La búsqueda semántica encuentra chunks relevantes para cada pregunta.
El LLM genera respuestas basadas en el contexto recuperado.
RAG reduce alucinaciones al anclar las respuestas en documentos reales.""",
        "metadata": {"fuente": "guia_rag.txt", "tema": "RAG"}
    },
    {
        "id": "doc_embeddings",
        "contenido": """Los embeddings son representaciones vectoriales densas del texto.
Sentence-transformers produce embeddings de alta calidad para búsqueda semántica.
FAISS indexa millones de vectores para búsqueda eficiente.
La similitud coseno mide qué tan parecidos son dos textos semánticamente.""",
        "metadata": {"fuente": "guia_embeddings.txt", "tema": "Embeddings"}
    }
]

print("Indexando documentos...")
rag.agregar_documentos(documentos)
print(f"Store: {len(rag.store)} chunks indexados")

# Queries de prueba
preguntas = [
    "¿Qué es BERT y cómo se preentrenó?",
    "¿Cómo funciona el pipeline RAG?",
    "¿Qué es la similitud coseno?",
]

print("\n=== Queries RAG ===")
resultados_queries = []
for pregunta in preguntas:
    resultado = rag.query(pregunta)
    print(f"\nPregunta: {pregunta}")
    print(f"Top chunk (score={resultado['chunks_recuperados'][0]['score']:.4f}):")
    print(f"  {resultado['chunks_recuperados'][0]['texto'][:100]}...")
    print(f"Respuesta: {resultado['respuesta']}")
    resultados_queries.append({
        "pregunta": pregunta,
        "score_promedio": resultado["score_promedio"],
        "n_chunks": resultado["n_chunks"]
    })

with open("outputs/m13_rag_resultados.json", "w", encoding="utf-8") as f:
    json.dump(resultados_queries, f, ensure_ascii=False, indent=2)
print("\noutputs/m13_rag_resultados.json guardado")
```

---

## 4. RAG con sentence-transformers y Chroma

```python
# rag_sentence_transformers.py
import os, json
import numpy as np

os.makedirs("outputs", exist_ok=True)

try:
    from sentence_transformers import SentenceTransformer
    import chromadb

    # --- Configuración ---
    modelo_embed = SentenceTransformer("all-MiniLM-L6-v2")
    client = chromadb.Client()
    coleccion = client.create_collection(
        "rag_basico",
        metadata={"hnsw:space": "cosine"}
    )

    # --- Corpus ---
    documentos = {
        "doc1": "Los transformers usan mecanismo de atención para procesar secuencias en paralelo.",
        "doc2": "BERT es preentrenado con Masked Language Modeling de forma bidireccional.",
        "doc3": "GPT genera texto usando un decoder causal que predice el siguiente token.",
        "doc4": "RAG combina recuperación de documentos relevantes con generación de texto.",
        "doc5": "Los embeddings de sentence-transformers permiten búsqueda semántica eficiente.",
        "doc6": "FAISS indexa millones de vectores para búsqueda de vecinos más cercanos.",
        "doc7": "El fine-tuning con LoRA añade matrices de bajo rango a las capas del modelo.",
        "doc8": "La cuantización reduce la precisión numérica para reducir memoria y aumentar velocidad.",
    }

    # --- Indexar ---
    textos = list(documentos.values())
    ids = list(documentos.keys())
    embeddings = modelo_embed.encode(textos)

    coleccion.add(
        embeddings=embeddings.tolist(),
        documents=textos,
        ids=ids,
        metadatas=[{"doc_id": i} for i in ids]
    )
    print(f"Indexados: {coleccion.count()} documentos")

    # --- RAG Query ---
    def rag_query_chroma(pregunta: str, k: int = 3) -> Dict:
        q_emb = modelo_embed.encode([pregunta])
        resultados = coleccion.query(
            query_embeddings=q_emb.tolist(),
            n_results=k,
            include=["documents", "distances"]
        )

        chunks = resultados["documents"][0]
        scores = [1 - d for d in resultados["distances"][0]]  # distancia coseno → similitud

        contexto = "\n".join([f"[{i+1}] {c}" for i, c in enumerate(chunks)])
        prompt = f"Contexto:\n{contexto}\n\nPregunta: {pregunta}\nRespuesta:"

        return {
            "pregunta": pregunta,
            "chunks": chunks,
            "scores": scores,
            "prompt_preview": prompt[:200] + "..."
        }

    preguntas = ["¿Cómo funciona RAG?", "¿Qué es BERT?", "¿Para qué sirve LoRA?"]
    print("\n=== RAG con sentence-transformers + Chroma ===")
    resultados_st = []
    for p in preguntas:
        res = rag_query_chroma(p, k=2)
        print(f"\n'{p}'")
        for i, (chunk, score) in enumerate(zip(res["chunks"], res["scores"])):
            print(f"  [{i+1}] score={score:.4f}: {chunk[:70]}...")
        resultados_st.append({"pregunta": p, "scores": res["scores"]})

    with open("outputs/m13_rag_st_chroma.json", "w", encoding="utf-8") as f:
        json.dump(resultados_st, f, ensure_ascii=False, indent=2)
    print("\noutputs/m13_rag_st_chroma.json guardado")

except ImportError as e:
    print(f"[Dependencia no disponible: {e}]")
    print("pip install sentence-transformers chromadb")
    # Guardar placeholder
    with open("outputs/m13_rag_st_chroma.json", "w") as f:
        json.dump([{"nota": "sentence-transformers no disponible"}], f)
```

---

## 5. Evaluación del pipeline RAG

```python
# rag_evaluacion.py
import numpy as np
import json
import os

os.makedirs("outputs", exist_ok=True)

# --- Métricas de evaluación RAG ---

def faithfulness(respuesta: str, contexto: List[str]) -> float:
    """
    Faithfulness: fracción de afirmaciones de la respuesta respaldadas por el contexto.
    Versión simplificada: overlap de palabras clave entre respuesta y contexto.
    En producción: usar LLM como juez.
    """
    palabras_resp = set(respuesta.lower().split())
    palabras_ctx = set(" ".join(contexto).lower().split())
    # Excluir stopwords básicas
    stopwords = {"el", "la", "los", "las", "de", "del", "en", "un", "una",
                 "y", "o", "a", "con", "por", "que", "se", "es", "son"}
    palabras_resp -= stopwords
    palabras_ctx -= stopwords
    if not palabras_resp:
        return 0.0
    overlap = len(palabras_resp & palabras_ctx) / len(palabras_resp)
    return float(overlap)

def answer_relevance(pregunta: str, respuesta: str) -> float:
    """
    Answer Relevance: qué tan relevante es la respuesta para la pregunta.
    Versión simplificada: overlap de términos de la pregunta en la respuesta.
    """
    palabras_q = set(pregunta.lower().split())
    palabras_r = set(respuesta.lower().split())
    stopwords = {"qué", "es", "cómo", "por", "qué", "cuál", "cuando", "un", "una"}
    palabras_q -= stopwords
    if not palabras_q:
        return 0.0
    return float(len(palabras_q & palabras_r) / len(palabras_q))

def context_precision(chunks_recuperados: List[str],
                      chunks_relevantes: List[str]) -> float:
    """
    Context Precision: fracción de chunks recuperados que son relevantes.
    Requiere ground truth de qué chunks son relevantes.
    """
    if not chunks_recuperados:
        return 0.0
    relevantes_set = set(chunks_relevantes)
    hits = sum(1 for c in chunks_recuperados if c in relevantes_set)
    return hits / len(chunks_recuperados)

def context_recall(chunks_relevantes: List[str],
                   chunks_recuperados: List[str]) -> float:
    """
    Context Recall: fracción de chunks relevantes que fueron recuperados.
    """
    if not chunks_relevantes:
        return 1.0
    recuperados_set = set(chunks_recuperados)
    hits = sum(1 for c in chunks_relevantes if c in recuperados_set)
    return hits / len(chunks_relevantes)

def ragas_score(faithfulness_s: float, answer_relevance_s: float,
                context_precision_s: float, context_recall_s: float) -> float:
    """
    RAGAS score: media armónica de las 4 métricas.
    Penaliza fuertemente si cualquier métrica es baja.
    """
    scores = [faithfulness_s, answer_relevance_s,
              context_precision_s, context_recall_s]
    if any(s <= 0 for s in scores):
        return 0.0
    return 4.0 / sum(1.0 / s for s in scores)

# --- Evaluación de ejemplo ---
from typing import List

ejemplos = [
    {
        "pregunta": "¿Qué es BERT?",
        "respuesta": "BERT es un modelo de lenguaje bidireccional preentrenado con Masked Language Modeling.",
        "contexto_recuperado": [
            "BERT es preentrenado con Masked Language Modeling de forma bidireccional.",
            "Los transformers usan mecanismo de atención para procesar secuencias."
        ],
        "contexto_relevante": [
            "BERT es preentrenado con Masked Language Modeling de forma bidireccional."
        ]
    },
    {
        "pregunta": "¿Cómo funciona RAG?",
        "respuesta": "RAG combina recuperación de documentos con generación de texto usando un LLM.",
        "contexto_recuperado": [
            "RAG combina recuperación de documentos relevantes con generación de texto.",
            "Los embeddings de sentence-transformers permiten búsqueda semántica."
        ],
        "contexto_relevante": [
            "RAG combina recuperación de documentos relevantes con generación de texto."
        ]
    },
    {
        "pregunta": "¿Para qué sirve la cuantización?",
        "respuesta": "La cuantización reduce la memoria y aumenta la velocidad de inferencia.",
        "contexto_recuperado": [
            "FAISS indexa millones de vectores para búsqueda eficiente.",   # irrelevante
            "La cuantización reduce la precisión numérica para reducir memoria."
        ],
        "contexto_relevante": [
            "La cuantización reduce la precisión numérica para reducir memoria y aumentar velocidad."
        ]
    }
]

print("=== Evaluación RAGAS ===\n")
metricas_globales = []

for ej in ejemplos:
    f = faithfulness(ej["respuesta"], ej["contexto_recuperado"])
    ar = answer_relevance(ej["pregunta"], ej["respuesta"])
    cp = context_precision(ej["contexto_recuperado"], ej["contexto_relevante"])
    cr = context_recall(ej["contexto_relevante"], ej["contexto_recuperado"])
    ragas = ragas_score(f, ar, cp, cr)

    print(f"Pregunta: {ej['pregunta']}")
    print(f"  Faithfulness:       {f:.3f}")
    print(f"  Answer Relevance:   {ar:.3f}")
    print(f"  Context Precision:  {cp:.3f}")
    print(f"  Context Recall:     {cr:.3f}")
    print(f"  RAGAS Score:        {ragas:.3f}")
    print()
    metricas_globales.append({"pregunta": ej["pregunta"], "ragas": ragas,
                               "faithfulness": f, "answer_relevance": ar})

# Media global
ragas_medio = np.mean([m["ragas"] for m in metricas_globales])
print(f"RAGAS medio: {ragas_medio:.3f}")

with open("outputs/m13_evaluacion_ragas.json", "w", encoding="utf-8") as f:
    json.dump(metricas_globales, f, ensure_ascii=False, indent=2)
print("outputs/m13_evaluacion_ragas.json guardado")
```

---

## 6. Checkpoint — Pruebas unitarias

```python
# checkpoint_m13.py
"""
Checkpoint M13: Pipeline RAG Básico
Ejecutar con: python checkpoint_m13.py
"""
import numpy as np
import json
import os
import re
from typing import List, Dict

sys_import = __import__("sys")
sys_import.path.insert(0, os.path.dirname(__file__))
os.makedirs("outputs", exist_ok=True)

# ── Implementaciones locales ─────────────────────────────────────────────────
class EmbedderSimulado:
    def __init__(self, dim=64, seed=42):
        self.dim = dim

    def _hash_vec(self, texto):
        seed_val = sum(ord(c) * (i + 1) for i, c in enumerate(texto[:200])) % (2**31)
        rng = np.random.RandomState(seed_val)
        v = rng.randn(self.dim).astype(np.float32)
        return v / (np.linalg.norm(v) + 1e-9)

    def encode(self, textos):
        return np.stack([self._hash_vec(t) for t in textos])

class VectorStore:
    def __init__(self):
        self.ids = []; self.vectors = []; self.textos = []; self.metadata = []

    def add(self, ids, vectors, textos, metadatas=None):
        metadatas = metadatas or [{} for _ in ids]
        for id_, vec, txt, meta in zip(ids, vectors, textos, metadatas):
            self.ids.append(id_); self.vectors.append(vec / (np.linalg.norm(vec)+1e-9))
            self.textos.append(txt); self.metadata.append(meta)

    def search(self, q, k=5):
        if not self.vectors: return []
        q = q / (np.linalg.norm(q) + 1e-9)
        vecs = np.stack(self.vectors)
        sims = vecs @ q
        top_k = np.argsort(sims)[::-1][:k]
        return [{"id": self.ids[i], "texto": self.textos[i],
                 "score": float(sims[i]), "metadata": self.metadata[i]} for i in top_k]

    def __len__(self): return len(self.ids)

def chunk_fixed(text, size=200, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        chunks.append(text[start:start+size])
        start += size - overlap
    return chunks

def faithfulness(respuesta, contexto):
    palabras_resp = set(respuesta.lower().split())
    palabras_ctx = set(" ".join(contexto).lower().split())
    stopwords = {"el","la","los","las","de","del","en","un","una","y","o","a","con","por","que","se","es"}
    palabras_resp -= stopwords; palabras_ctx -= stopwords
    if not palabras_resp: return 0.0
    return float(len(palabras_resp & palabras_ctx) / len(palabras_resp))

# ── Tests ─────────────────────────────────────────────────────────────────────
def test_chunking_cobre_texto_completo():
    """T1: los chunks cubren todo el texto original"""
    texto = "a " * 500  # 1000 chars
    chunks = chunk_fixed(texto, size=200, overlap=50)
    # El último chunk debe incluir el final del texto
    contenido_total = set()
    for chunk in chunks:
        contenido_total.update(chunk.split())
    assert "a" in contenido_total, "Los chunks deben cubrir el texto"
    # Verificar que la longitud cubierta sea suficiente
    total_cubierto = sum(len(c) for c in chunks)
    assert total_cubierto >= len(texto), f"Chunks cubren {total_cubierto} < {len(texto)}"
    print(f"T1 PASS: chunking cubre {len(chunks)} chunks, total={total_cubierto}")

def test_overlap_genera_redundancia():
    """T2: con overlap > 0, texto duplicado entre chunks consecutivos"""
    texto = "a" * 400
    sin_overlap = chunk_fixed(texto, size=100, overlap=0)
    con_overlap = chunk_fixed(texto, size=100, overlap=40)
    # Con overlap debe haber más chunks
    assert len(con_overlap) > len(sin_overlap), \
        "Con overlap debe generarse más chunks"
    print(f"T2 PASS: sin_overlap={len(sin_overlap)} chunks, con_overlap={len(con_overlap)} chunks")

def test_vector_store_recupera_similar():
    """T3: VectorStore recupera el chunk más similar a la query"""
    np.random.seed(42)
    embedder = EmbedderSimulado(dim=64)
    store = VectorStore()

    chunks = ["gato siamés azul persa", "perro labrador golden", "pez tropical",
              "ave canario loro", "reptil iguana tortuga"]
    embs = embedder.encode(chunks)
    store.add(ids=[f"c{i}" for i in range(len(chunks))],
              vectors=list(embs), textos=chunks)

    # Query igual al chunk 0 debe recuperarlo primero
    q_emb = embedder.encode(["gato siamés azul persa"])[0]
    resultados = store.search(q_emb, k=3)
    assert resultados[0]["id"] == "c0", \
        f"Esperado c0 primero, got {resultados[0]['id']}"
    assert resultados[0]["score"] > 0.99, \
        f"Score esperado >0.99, got {resultados[0]['score']}"
    print(f"T3 PASS: chunk más similar recuperado (score={resultados[0]['score']:.4f})")

def test_prompt_contiene_contexto_y_pregunta():
    """T4: el prompt RAG incluye el contexto recuperado y la pregunta"""
    contexto_chunks = [
        {"texto": "BERT es un modelo bidireccional.", "score": 0.9},
        {"texto": "GPT usa decoder causal.", "score": 0.8}
    ]
    pregunta = "¿Qué diferencia hay entre BERT y GPT?"

    # Construir prompt
    contexto_str = "\n".join([f"[{i+1}] {c['texto']}"
                               for i, c in enumerate(contexto_chunks)])
    prompt = f"CONTEXTO:\n{contexto_str}\n\nPREGUNTA: {pregunta}\n\nRESPUESTA:"

    assert "BERT" in prompt, "Prompt debe contener contexto"
    assert pregunta in prompt, "Prompt debe contener la pregunta"
    assert "CONTEXTO" in prompt, "Prompt debe tener sección CONTEXTO"
    assert "PREGUNTA" in prompt, "Prompt debe tener sección PREGUNTA"
    print(f"T4 PASS: prompt contiene contexto y pregunta ({len(prompt)} chars)")

def test_faithfulness_alta_cuando_respuesta_en_contexto():
    """T5: faithfulness alta cuando la respuesta usa términos del contexto"""
    contexto = ["BERT es un modelo bidireccional preentrenado con MLM."]
    respuesta_buena = "BERT es un modelo bidireccional preentrenado."
    respuesta_mala  = "Los coches eléctricos son el futuro del transporte."

    f_buena = faithfulness(respuesta_buena, contexto)
    f_mala  = faithfulness(respuesta_mala, contexto)

    assert f_buena > f_mala, \
        f"Faithfulness buena ({f_buena:.3f}) debe ser > mala ({f_mala:.3f})"
    assert f_buena > 0.5, f"Faithfulness buena debe ser > 0.5, got {f_buena}"
    print(f"T5 PASS: faithfulness buena={f_buena:.3f} > mala={f_mala:.3f}")

def test_rag_pipeline_end_to_end():
    """T6: el pipeline completo retorna resultado con estructura correcta"""
    embedder = EmbedderSimulado(dim=64)
    store = VectorStore()

    # Indexar
    docs = ["El transformer usa atención. Es una arquitectura poderosa.",
            "BERT es bidireccional. GPT es causal.",
            "RAG recupera documentos relevantes."]
    embs = embedder.encode(docs)
    store.add(ids=[f"d{i}" for i in range(3)], vectors=list(embs), textos=docs)

    # Query
    pregunta = "¿Qué es RAG?"
    q_emb = embedder.encode([pregunta])[0]
    chunks = store.search(q_emb, k=2)

    # Verificar estructura
    assert len(chunks) == 2, f"Esperados 2 chunks, got {len(chunks)}"
    assert all("id" in c and "texto" in c and "score" in c for c in chunks), \
        "Cada chunk debe tener id, texto, score"
    assert chunks[0]["score"] >= chunks[1]["score"], \
        "Chunks deben estar ordenados por score descendente"
    print(f"T6 PASS: pipeline E2E ok ({len(chunks)} chunks, top score={chunks[0]['score']:.4f})")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_chunking_cobre_texto_completo,
        test_overlap_genera_redundancia,
        test_vector_store_recupera_similar,
        test_prompt_contiene_contexto_y_pregunta,
        test_faithfulness_alta_cuando_respuesta_en_contexto,
        test_rag_pipeline_end_to_end,
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
        print("M13 COMPLETADO")
```

---

## Resumen

| Componente | Tecnología | Alternativas |
|---|---|---|
| Chunking | Fixed-size / Recursive / Sentence | LangChain TextSplitter |
| Embedding | sentence-transformers | OpenAI ada-002, Cohere |
| VectorStore | Chroma | Pinecone, pgvector, FAISS |
| LLM | GPT-4, Claude, Llama | Mistral, Gemini |
| Evaluación | RAGAS | TruLens, UpTrain |

**Métricas RAGAS:**
- `Faithfulness`: ¿La respuesta está respaldada por el contexto?
- `Answer Relevance`: ¿La respuesta responde la pregunta?
- `Context Precision`: ¿Los chunks recuperados son relevantes?
- `Context Recall`: ¿Se recuperaron todos los chunks relevantes?
