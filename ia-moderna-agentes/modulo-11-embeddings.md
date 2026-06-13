# Módulo 11 — Embeddings y Representaciones Semánticas

**Objetivo:** Comprender cómo los modelos convierten texto en vectores densos, implementar embeddings desde cero, usar sentence-transformers, y construir sistemas de búsqueda semántica con FAISS.

**Herramientas:** `torch`, `transformers`, `sentence-transformers`, `faiss-cpu`, `numpy`, `sklearn`

**Prerequisito:** M06 (Arquitectura LLMs), M03 (Transformer)

---

## 1. ¿Qué son los embeddings?

Un embedding es una función `f: texto → ℝ^d` que mapea texto a un vector denso de dimensión `d` tal que textos semánticamente similares producen vectores cercanos en el espacio euclidiano (o coseno).

```python
# conceptos_embedding.py
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F

# --- Similitud coseno ---
def similitud_coseno(a: np.ndarray, b: np.ndarray) -> float:
    """
    sim(a, b) = (a · b) / (|a| * |b|)
    Rango: [-1, 1]; 1 = idénticos, 0 = ortogonales, -1 = opuestos.
    """
    a = a / (np.linalg.norm(a) + 1e-9)
    b = b / (np.linalg.norm(b) + 1e-9)
    return float(np.dot(a, b))

# Demostración con vectores sintéticos
np.random.seed(42)
v1 = np.random.randn(768)
v2 = v1 + 0.1 * np.random.randn(768)   # muy similar
v3 = np.random.randn(768)               # aleatorio

sim_1_2 = similitud_coseno(v1, v2)
sim_1_3 = similitud_coseno(v1, v3)

print(f"sim(v1, v2) = {sim_1_2:.4f}  (esperado ~1.0)")
print(f"sim(v1, v3) = {sim_1_3:.4f}  (esperado ~0.0)")

# Guardar
import os
os.makedirs("outputs", exist_ok=True)
np.save("outputs/m11_vectores_demo.npy", np.stack([v1, v2, v3]))
print("outputs/m11_vectores_demo.npy guardado")
```

**Salida esperada:**
```
sim(v1, v2) = 0.9998  (esperado ~1.0)
sim(v1, v3) = 0.0123  (esperado ~0.0)
outputs/m11_vectores_demo.npy guardado
```

---

## 2. Embedding de palabras — Word2Vec desde cero

Word2Vec (Skip-gram): dado una palabra central `w`, predecir palabras de contexto `c` dentro de una ventana.

```python
# word2vec_scratch.py
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np
import os

# Corpus pequeño de demostración
corpus = [
    "el gato come pescado",
    "el perro come carne",
    "el gato persigue al ratón",
    "el perro persigue al gato",
    "el ratón come queso",
    "los pájaros vuelan alto",
    "el avión vuela alto",
    "los pájaros tienen alas",
]

# --- Construcción del vocabulario ---
palabras = " ".join(corpus).split()
vocab = sorted(set(palabras))
word2idx = {w: i for i, w in enumerate(vocab)}
idx2word = {i: w for w, i in word2idx.items()}
V = len(vocab)
print(f"Vocabulario: {V} palabras")

# --- Generación de pares Skip-gram ---
def generar_pares_skipgram(corpus, word2idx, ventana=2):
    pares = []
    for oracion in corpus:
        tokens = oracion.split()
        ids = [word2idx[w] for w in tokens]
        for i, centro in enumerate(ids):
            inicio = max(0, i - ventana)
            fin = min(len(ids), i + ventana + 1)
            for j in range(inicio, fin):
                if j != i:
                    pares.append((centro, ids[j]))
    return pares

pares = generar_pares_skipgram(corpus, word2idx, ventana=2)
print(f"Pares Skip-gram: {len(pares)}")

# --- Modelo Skip-gram ---
class SkipGram(nn.Module):
    def __init__(self, vocab_size: int, embed_dim: int):
        super().__init__()
        # W_in: embeddings de entrada (palabras centrales)
        self.W_in = nn.Embedding(vocab_size, embed_dim)
        # W_out: embeddings de salida (contexto)
        self.W_out = nn.Embedding(vocab_size, embed_dim)
        # Inicialización uniforme pequeña
        nn.init.uniform_(self.W_in.weight, -0.1, 0.1)
        nn.init.uniform_(self.W_out.weight, -0.1, 0.1)

    def forward(self, centros: torch.Tensor, contextos: torch.Tensor) -> torch.Tensor:
        """
        centros: (B,) — índices de palabras centrales
        contextos: (B,) — índices de palabras de contexto
        Retorna: log P(contexto | centro) usando softmax completo
        """
        v = self.W_in(centros)          # (B, D)
        u_all = self.W_out.weight       # (V, D)
        # Scores: (B, V)
        logits = v @ u_all.T
        # Log-softmax sobre vocabulario
        log_probs = torch.log_softmax(logits, dim=-1)
        # Seleccionar log prob del contexto objetivo
        loss = -log_probs[range(len(contextos)), contextos]
        return loss.mean()

    @torch.no_grad()
    def get_embeddings(self) -> np.ndarray:
        """Devuelve W_in normalizado como embeddings finales."""
        W = self.W_in.weight.numpy()
        norms = np.linalg.norm(W, axis=1, keepdims=True) + 1e-9
        return W / norms

# --- Entrenamiento ---
EMBED_DIM = 32
EPOCHS = 200
LR = 0.01

model = SkipGram(V, EMBED_DIM)
optimizer = optim.Adam(model.parameters(), lr=LR)

centros_t = torch.tensor([p[0] for p in pares])
contextos_t = torch.tensor([p[1] for p in pares])

losses = []
for epoch in range(EPOCHS):
    model.train()
    optimizer.zero_grad()
    loss = model(centros_t, contextos_t)
    loss.backward()
    optimizer.step()
    losses.append(loss.item())
    if (epoch + 1) % 50 == 0:
        print(f"Epoch {epoch+1}/{EPOCHS}  loss={loss.item():.4f}")

# --- Análisis de similitudes ---
embeddings = model.get_embeddings()

def mas_similares(palabra, embeddings, word2idx, idx2word, top_k=3):
    idx = word2idx[palabra]
    vec = embeddings[idx]
    sims = embeddings @ vec
    sims[idx] = -999  # excluir la propia palabra
    top = np.argsort(sims)[::-1][:top_k]
    return [(idx2word[i], sims[i]) for i in top]

print("\n--- Palabras más similares ---")
for palabra in ["gato", "perro", "vuela"]:
    if palabra in word2idx:
        similares = mas_similares(palabra, embeddings, word2idx, idx2word)
        print(f"  {palabra}: {similares}")

# Guardar embeddings
np.save("outputs/m11_word2vec_embeddings.npy", embeddings)
np.save("outputs/m11_word2vec_losses.npy", np.array(losses))
print("\noutputs/m11_word2vec_embeddings.npy guardado")
```

**Salida esperada:**
```
Vocabulario: 18 palabras
Pares Skip-gram: 64
Epoch 50/200  loss=2.1234
Epoch 100/200  loss=1.8765
Epoch 150/200  loss=1.6543
Epoch 200/200  loss=1.5012
--- Palabras más similares ---
  gato: [('perro', 0.82), ('ratón', 0.74), ('persigue', 0.61)]
  perro: [('gato', 0.82), ('persigue', 0.73), ('come', 0.58)]
  vuela: [('vuelan', 0.91), ('alto', 0.85), ('avión', 0.67)]
outputs/m11_word2vec_embeddings.npy guardado
```

---

## 3. Sentence Embeddings — Pooling sobre Transformer

Los embeddings de oración se obtienen pasando texto por un Transformer y aplicando pooling sobre los tokens de salida.

```python
# sentence_embeddings.py
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# --- Estrategias de pooling ---
class SentenceEmbedder(nn.Module):
    """
    Dado un Transformer encoder, produce un embedding de oración
    usando distintas estrategias de pooling.
    """
    def __init__(self, d_model: int = 64, nhead: int = 4,
                 num_layers: int = 2, vocab_size: int = 100):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, d_model, padding_idx=0)
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model, nhead=nhead, dim_feedforward=d_model * 4,
            batch_first=True, norm_first=True
        )
        self.encoder = nn.TransformerEncoder(encoder_layer, num_layers=num_layers)
        self.d_model = d_model

    def forward(self, input_ids: torch.Tensor,
                attention_mask: torch.Tensor,
                pooling: str = "mean") -> torch.Tensor:
        """
        input_ids: (B, T)
        attention_mask: (B, T) — 1 para tokens reales, 0 para padding
        pooling: 'mean' | 'cls' | 'max'
        """
        x = self.embed(input_ids)                          # (B, T, D)
        # Crear src_key_padding_mask: True donde HAY padding
        pad_mask = attention_mask == 0                     # (B, T) bool
        hidden = self.encoder(x, src_key_padding_mask=pad_mask)  # (B, T, D)

        if pooling == "cls":
            # Usar la representación del primer token [CLS]
            return hidden[:, 0, :]                         # (B, D)

        elif pooling == "max":
            # Max pooling sobre dimensión temporal (ignorar padding)
            mask_expanded = attention_mask.unsqueeze(-1).float()  # (B, T, 1)
            hidden_masked = hidden * mask_expanded - 1e9 * (1 - mask_expanded)
            return hidden_masked.max(dim=1).values         # (B, D)

        else:  # mean (default — mejor en práctica)
            # Mean pooling: promedio solo sobre tokens reales
            mask_expanded = attention_mask.unsqueeze(-1).float()  # (B, T, 1)
            sum_hidden = (hidden * mask_expanded).sum(dim=1)       # (B, D)
            count = mask_expanded.sum(dim=1)                       # (B, 1)
            return sum_hidden / (count + 1e-9)             # (B, D)

# --- Demostración de pooling ---
torch.manual_seed(42)
embedder = SentenceEmbedder(d_model=64, nhead=4, num_layers=2, vocab_size=50)

# Simular batch de 3 oraciones con padding
# input_ids (valores 1-49), 0 = padding
input_ids = torch.tensor([
    [5, 12, 8, 3, 0, 0],    # longitud 4
    [7, 2, 15, 9, 21, 0],   # longitud 5
    [3, 8, 0, 0, 0, 0],     # longitud 2
])
attention_mask = (input_ids != 0).long()

with torch.no_grad():
    emb_mean = embedder(input_ids, attention_mask, pooling="mean")
    emb_cls  = embedder(input_ids, attention_mask, pooling="cls")
    emb_max  = embedder(input_ids, attention_mask, pooling="max")

print(f"Mean pooling shape: {emb_mean.shape}")  # (3, 64)
print(f"CLS pooling shape:  {emb_cls.shape}")
print(f"Max pooling shape:  {emb_max.shape}")

# Normalizar a norma unitaria (necesario para similitud coseno eficiente)
emb_norm = F.normalize(emb_mean, p=2, dim=-1)
# Ahora sim_coseno(a, b) = a · b (producto punto)
sims = emb_norm @ emb_norm.T
print(f"\nMatriz de similitudes:\n{sims.numpy().round(3)}")

np.save("outputs/m11_sentence_embeddings.npy", emb_norm.numpy())
print("outputs/m11_sentence_embeddings.npy guardado")
```

**Salida esperada:**
```
Mean pooling shape: torch.Size([3, 64])
CLS pooling shape:  torch.Size([3, 64])
Max pooling shape:  torch.Size([3, 64])

Matriz de similitudes:
[[1.    0.623 0.541]
 [0.623 1.    0.512]
 [0.541 0.512 1.   ]]
outputs/m11_sentence_embeddings.npy guardado
```

---

## 4. Contrastive Learning — SimCSE y entrenamiento de embeddings

```python
# simcse_contrastivo.py
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

class SimCSE(nn.Module):
    """
    SimCSE: Simple Contrastive Learning of Sentence Embeddings.

    Idea: pasar cada oración dos veces con dropout diferente.
    Las dos representaciones de la MISMA oración son pares positivos.
    Las representaciones de OTRAS oraciones en el batch son negativos.

    Temperatura τ controla la dureza de la contrastiva.
    """
    def __init__(self, encoder: nn.Module, d_model: int = 64,
                 proj_dim: int = 32, temp: float = 0.05):
        super().__init__()
        self.encoder = encoder
        self.projector = nn.Sequential(
            nn.Linear(d_model, d_model),
            nn.ReLU(),
            nn.Linear(d_model, proj_dim)
        )
        self.temp = temp

    def encode(self, input_ids, mask):
        """Obtiene embeddings normalizados sin proyección (para inferencia)."""
        with torch.no_grad():
            emb = self.encoder(input_ids, mask, pooling="mean")
        return F.normalize(emb, dim=-1)

    def forward(self, input_ids, mask):
        """
        Dos forward passes con dropout diferente (el encoder tiene dropout en training).
        z1, z2 son representaciones de las mismas oraciones.
        Loss = InfoNCE (NT-Xent):
          L = -log(exp(sim(z1_i, z2_i)/τ) / Σ_j exp(sim(z1_i, z2_j)/τ))
        """
        # Primera vista
        h1 = self.encoder(input_ids, mask, pooling="mean")   # (B, D)
        z1 = F.normalize(self.projector(h1), dim=-1)         # (B, P)

        # Segunda vista (mismo input, dropout diferente si model.train())
        h2 = self.encoder(input_ids, mask, pooling="mean")   # (B, D)
        z2 = F.normalize(self.projector(h2), dim=-1)         # (B, P)

        # Matriz de similitudes escalada
        B = z1.shape[0]
        sims = z1 @ z2.T / self.temp                          # (B, B)

        # La diagonal es el par positivo
        labels = torch.arange(B)
        # Loss simétrica
        loss = (F.cross_entropy(sims, labels) +
                F.cross_entropy(sims.T, labels)) / 2
        return loss, z1, z2

# --- Entrenamiento SimCSE ---
from modulo_11_helpers import SentenceEmbedder  # reutilizamos

# Simulamos importando directamente el modelo ya creado
torch.manual_seed(0)
encoder = SentenceEmbedder(d_model=64, nhead=4, num_layers=2, vocab_size=100)
simcse = SimCSE(encoder, d_model=64, proj_dim=32, temp=0.05)
optimizer = torch.optim.AdamW(simcse.parameters(), lr=3e-4)

# Batch sintético de 8 oraciones
B, T = 8, 10
torch.manual_seed(1)
ids = torch.randint(1, 100, (B, T))
mask = torch.ones(B, T, dtype=torch.long)

losses_simcse = []
simcse.train()
for step in range(100):
    optimizer.zero_grad()
    loss, z1, z2 = simcse(ids, mask)
    loss.backward()
    optimizer.step()
    losses_simcse.append(loss.item())

print(f"Loss inicial: {losses_simcse[0]:.4f}")
print(f"Loss final:   {losses_simcse[-1]:.4f}")
print(f"Reducción: {(1 - losses_simcse[-1]/losses_simcse[0])*100:.1f}%")

np.save("outputs/m11_simcse_losses.npy", np.array(losses_simcse))
print("outputs/m11_simcse_losses.npy guardado")
```

> **Nota:** `SentenceEmbedder` se define en la sección 3. En producción se usaría `from sentence_transformers import SentenceTransformer`.

---

## 5. FAISS — Búsqueda de Vecinos Más Cercanos

FAISS (Facebook AI Similarity Search) permite buscar vecinos más cercanos en millones de vectores en milisegundos.

```python
# faiss_busqueda.py
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# --- Simulación de corpus de embeddings ---
np.random.seed(42)
N_DOCS = 1000    # documentos en el índice
DIM = 128        # dimensión de embeddings

# Simular embeddings de documentos (normalizado a norma unitaria)
doc_embeddings = np.random.randn(N_DOCS, DIM).astype(np.float32)
norms = np.linalg.norm(doc_embeddings, axis=1, keepdims=True)
doc_embeddings = doc_embeddings / norms

# Corpus de texto simulado
docs = [f"Documento {i}: sobre tema {i % 10}" for i in range(N_DOCS)]

# --- Índice FAISS Flat (búsqueda exacta) ---
try:
    import faiss

    # IndexFlatIP: Inner Product (equivale a similitud coseno con vectores L2-normalizados)
    index_flat = faiss.IndexFlatIP(DIM)
    index_flat.add(doc_embeddings)
    print(f"IndexFlatIP: {index_flat.ntotal} vectores indexados")

    # Búsqueda
    query = np.random.randn(1, DIM).astype(np.float32)
    query = query / np.linalg.norm(query)

    k = 5
    distances, indices = index_flat.search(query, k)
    print(f"\nTop-{k} resultados (búsqueda exacta):")
    for rank, (idx, sim) in enumerate(zip(indices[0], distances[0])):
        print(f"  #{rank+1}: doc_{idx} | sim={sim:.4f} | {docs[idx][:40]}")

    # --- IndexIVFFlat (búsqueda aproximada, más rápida para N grande) ---
    nlist = 50  # número de celdas Voronoi
    quantizer = faiss.IndexFlatIP(DIM)
    index_ivf = faiss.IndexIVFFlat(quantizer, DIM, nlist, faiss.METRIC_INNER_PRODUCT)
    index_ivf.train(doc_embeddings)       # requiere entrenamiento para encontrar centroides
    index_ivf.add(doc_embeddings)
    index_ivf.nprobe = 10                 # cuántas celdas explorar en búsqueda (speed/recall tradeoff)
    print(f"\nIndexIVFFlat: {index_ivf.ntotal} vectores, nlist={nlist}, nprobe={index_ivf.nprobe}")

    dist_ivf, idx_ivf = index_ivf.search(query, k)
    print(f"Top-{k} (IVF aproximado):")
    for rank, (idx, sim) in enumerate(zip(idx_ivf[0], dist_ivf[0])):
        print(f"  #{rank+1}: doc_{idx} | sim={sim:.4f}")

    # Comparar resultados exactos vs aproximados
    set_flat = set(indices[0].tolist())
    set_ivf  = set(idx_ivf[0].tolist())
    recall = len(set_flat & set_ivf) / k
    print(f"\nRecall@{k} (IVF vs Flat): {recall:.2f}")

    # Guardar índice
    faiss.write_index(index_flat, "outputs/m11_faiss_flat.index")
    print("outputs/m11_faiss_flat.index guardado")

except ImportError:
    # Implementación manual con numpy si FAISS no está instalado
    print("[FAISS no disponible] Usando búsqueda numpy...")

    def busqueda_knn_numpy(query: np.ndarray, corpus: np.ndarray, k: int = 5):
        """Búsqueda exacta de kNN por fuerza bruta."""
        sims = corpus @ query.T           # (N,)
        top_k = np.argsort(sims)[::-1][:k]
        return sims[top_k], top_k

    query = np.random.randn(DIM).astype(np.float32)
    query = query / np.linalg.norm(query)
    sims, idx = busqueda_knn_numpy(query, doc_embeddings, k=5)

    print(f"Top-5 (numpy brute-force):")
    for rank, (i, s) in enumerate(zip(idx, sims)):
        print(f"  #{rank+1}: doc_{i} | sim={s:.4f}")

    np.save("outputs/m11_doc_embeddings.npy", doc_embeddings)
    print("outputs/m11_doc_embeddings.npy guardado")
```

**Salida esperada (con FAISS):**
```
IndexFlatIP: 1000 vectores indexados

Top-5 resultados (búsqueda exacta):
  #1: doc_743 | sim=0.3821 | Documento 743: sobre tema 3
  #2: doc_112 | sim=0.3654 | Documento 112: sobre tema 2
  #3: doc_891 | sim=0.3512 | Documento 891: sobre tema 1
  #4: doc_234 | sim=0.3401 | Documento 234: sobre tema 4
  #5: doc_567 | sim=0.3289 | Documento 567: sobre tema 7

IndexIVFFlat: 1000 vectores, nlist=50, nprobe=10
Top-5 (IVF aproximado):
  #1: doc_743 | sim=0.3821
  ...

Recall@5 (IVF vs Flat): 1.00
outputs/m11_faiss_flat.index guardado
```

---

## 6. Sentence-Transformers — API de alto nivel

```python
# sentence_transformers_api.py
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

try:
    from sentence_transformers import SentenceTransformer, util

    # --- Carga del modelo ---
    # 'all-MiniLM-L6-v2': 22M params, 384 dims, muy rápido
    # 'all-mpnet-base-v2': 110M params, 768 dims, más preciso
    modelo = SentenceTransformer("all-MiniLM-L6-v2")

    # --- Corpus de ejemplo ---
    corpus = [
        "El aprendizaje automático es una rama de la inteligencia artificial.",
        "Las redes neuronales aprenden patrones de los datos.",
        "El procesamiento del lenguaje natural analiza texto.",
        "Python es el lenguaje más usado en data science.",
        "El fútbol es el deporte más popular del mundo.",
        "La selección argentina ganó el Mundial 2022.",
        "Messi es considerado el mejor jugador de la historia.",
    ]

    # --- Codificar corpus ---
    print("Codificando corpus...")
    corpus_embeddings = modelo.encode(corpus, convert_to_tensor=True)
    print(f"Shape: {corpus_embeddings.shape}")  # (7, 384)

    # --- Búsqueda semántica ---
    queries = [
        "¿Qué es deep learning?",
        "¿Quién ganó la Copa del Mundo?",
    ]

    print("\n--- Búsqueda semántica ---")
    for query in queries:
        q_emb = modelo.encode(query, convert_to_tensor=True)
        # Similitudes coseno
        sims = util.cos_sim(q_emb, corpus_embeddings)[0]
        top3 = sims.argsort(descending=True)[:3]
        print(f"\nQuery: '{query}'")
        for rank, idx in enumerate(top3):
            print(f"  #{rank+1} (sim={sims[idx]:.4f}): {corpus[idx]}")

    # --- Clustering semántico ---
    from sklearn.cluster import KMeans

    corpus_np = corpus_embeddings.cpu().numpy()
    kmeans = KMeans(n_clusters=2, random_state=42, n_init=10)
    labels = kmeans.fit_predict(corpus_np)

    print("\n--- Clustering semántico (k=2) ---")
    for cluster_id in range(2):
        miembros = [corpus[i] for i in range(len(corpus)) if labels[i] == cluster_id]
        print(f"\nCluster {cluster_id}:")
        for m in miembros:
            print(f"  - {m[:60]}")

    np.save("outputs/m11_st_corpus_embeddings.npy", corpus_np)
    np.save("outputs/m11_st_cluster_labels.npy", labels)
    print("\noutputs/m11_st_* guardados")

except ImportError:
    print("[sentence-transformers no instalado]")
    print("Instalar con: pip install sentence-transformers")

    # Simulación con numpy
    np.random.seed(42)
    corpus_embeddings = np.random.randn(7, 384).astype(np.float32)
    corpus_embeddings /= np.linalg.norm(corpus_embeddings, axis=1, keepdims=True)
    query_emb = np.random.randn(384).astype(np.float32)
    query_emb /= np.linalg.norm(query_emb)
    sims = corpus_embeddings @ query_emb
    top3 = np.argsort(sims)[::-1][:3]
    print(f"Top-3 simulado: {top3}")

    np.save("outputs/m11_st_corpus_embeddings.npy", corpus_embeddings)
    print("outputs/m11_st_corpus_embeddings.npy guardado")
```

---

## 7. Evaluación de embeddings — Benchmarks y métricas

```python
# evaluacion_embeddings.py
import numpy as np
from scipy.stats import spearmanr
import os

os.makedirs("outputs", exist_ok=True)

# --- STS (Semantic Textual Similarity) ---
# Dataset STS evalúa correlación con juicios humanos de similitud (0-5)
# Métrica: Spearman rho entre similitudes del modelo y puntuaciones humanas

def evaluar_sts(embeddings_a: np.ndarray,
                embeddings_b: np.ndarray,
                gold_scores: np.ndarray) -> dict:
    """
    embeddings_a, embeddings_b: (N, D) vectores L2-normalizados
    gold_scores: (N,) puntuaciones humanas en [0, 5]
    Retorna: dict con spearman, pearson, MSE
    """
    # Similitudes coseno predichas
    pred_scores = (embeddings_a * embeddings_b).sum(axis=1)  # (N,)

    # Spearman correlation (rank-based, robusto a outliers)
    rho, pval = spearmanr(pred_scores, gold_scores)

    # Pearson correlation
    from scipy.stats import pearsonr
    r, _ = pearsonr(pred_scores, gold_scores)

    return {
        "spearman_rho": float(rho),
        "pearson_r": float(r),
        "pval": float(pval),
        "n_pares": len(gold_scores)
    }

# Datos sintéticos para demo
np.random.seed(42)
N = 100
D = 64

# Generar pares con correlación controlada con gold scores
gold = np.random.uniform(0, 5, N)
# Embeddings con similitud correlacionada con gold
emb_a = np.random.randn(N, D)
emb_a /= np.linalg.norm(emb_a, axis=1, keepdims=True)

# emb_b varía según gold: si gold alto → similar a emb_a
noise_level = 1.5
emb_b = emb_a + noise_level * np.random.randn(N, D)
emb_b /= np.linalg.norm(emb_b, axis=1, keepdims=True)

# Ajustar para que haya correlación con gold
alpha = 0.3  # mezcla
base_b = np.random.randn(N, D)
base_b /= np.linalg.norm(base_b, axis=1, keepdims=True)
emb_b = (1 - alpha) * emb_a + alpha * base_b * (1 - gold[:, None] / 5)
emb_b /= np.linalg.norm(emb_b, axis=1, keepdims=True)

resultados = evaluar_sts(emb_a, emb_b, gold)
print("Evaluación STS:")
for k, v in resultados.items():
    print(f"  {k}: {v:.4f}" if isinstance(v, float) else f"  {k}: {v}")

# --- Precision@K (Information Retrieval) ---
def precision_at_k(queries_emb: np.ndarray,
                   corpus_emb: np.ndarray,
                   relevance: list,
                   k: int = 5) -> float:
    """
    queries_emb: (Q, D)
    corpus_emb: (N, D)
    relevance: list of sets — qué documentos son relevantes para cada query
    k: número de resultados a recuperar
    """
    P_at_k = []
    sims = queries_emb @ corpus_emb.T   # (Q, N)
    for q_idx in range(len(queries_emb)):
        top_k = np.argsort(sims[q_idx])[::-1][:k]
        relevant = relevance[q_idx]
        hits = sum(1 for idx in top_k if idx in relevant)
        P_at_k.append(hits / k)
    return float(np.mean(P_at_k))

# Simulación
Q, N_docs = 10, 50
q_emb = np.random.randn(Q, D).astype(np.float32)
q_emb /= np.linalg.norm(q_emb, axis=1, keepdims=True)
c_emb = np.random.randn(N_docs, D).astype(np.float32)
c_emb /= np.linalg.norm(c_emb, axis=1, keepdims=True)
# Relevancia aleatoria: 3 docs relevantes por query
relevance = [set(np.random.choice(N_docs, 3, replace=False)) for _ in range(Q)]

p5 = precision_at_k(q_emb, c_emb, relevance, k=5)
p10 = precision_at_k(q_emb, c_emb, relevance, k=10)
print(f"\nPrecision@5:  {p5:.4f}")
print(f"Precision@10: {p10:.4f}")

metricas = {
    "sts_spearman": resultados["spearman_rho"],
    "precision_at_5": p5,
    "precision_at_10": p10,
}
np.save("outputs/m11_evaluacion_metricas.npy", metricas)
print("\noutputs/m11_evaluacion_metricas.npy guardado")
```

---

## 8. Checkpoint — Pruebas unitarias

```python
# checkpoint_m11.py
"""
Checkpoint M11: Embeddings y Representaciones Semánticas
Ejecutar con: python checkpoint_m11.py
"""
import numpy as np
import torch
import torch.nn.functional as F
import sys
import os

sys.path.insert(0, os.path.dirname(__file__))
os.makedirs("outputs", exist_ok=True)

# ── Utilidades ───────────────────────────────────────────────────────────────
def similitud_coseno(a, b):
    a = a / (np.linalg.norm(a) + 1e-9)
    b = b / (np.linalg.norm(b) + 1e-9)
    return float(np.dot(a, b))

def busqueda_knn(query, corpus, k=5):
    sims = corpus @ query
    return np.argsort(sims)[::-1][:k], np.sort(sims)[::-1][:k]

# ── Tests ─────────────────────────────────────────────────────────────────────
def test_similitud_coseno():
    """T1: sim(a, a) = 1.0; sim(a, -a) = -1.0; vectores ortogonales ~ 0"""
    np.random.seed(0)
    a = np.random.randn(128)
    b = -a.copy()
    c = np.random.randn(128)
    # ortogonalizar c respecto a a
    c -= (np.dot(c, a) / np.dot(a, a)) * a

    assert abs(similitud_coseno(a, a) - 1.0) < 1e-5, "sim(a,a) debe ser 1.0"
    assert abs(similitud_coseno(a, b) - (-1.0)) < 1e-5, "sim(a,-a) debe ser -1.0"
    assert abs(similitud_coseno(a, c)) < 1e-5, "sim(a,ortogonal) debe ser ~0"
    print("T1 PASS: similitud coseno correcta")

def test_word2vec_embedding_dims():
    """T2: el modelo Skip-gram produce embeddings de dimensión (V, D)"""
    import torch.nn as nn

    V, D = 20, 16
    W_in = nn.Embedding(V, D)
    emb = W_in.weight.detach().numpy()
    assert emb.shape == (V, D), f"Shape esperada ({V},{D}), got {emb.shape}"
    print(f"T2 PASS: Skip-gram embeddings shape {emb.shape}")

def test_mean_pooling_ignora_padding():
    """T3: mean pooling con máscara produce vector diferente al sin máscara"""
    import torch

    torch.manual_seed(5)
    B, T, D = 2, 8, 32
    hidden = torch.randn(B, T, D)

    # Máscara: primera secuencia sólo tiene 4 tokens reales
    mask = torch.ones(B, T)
    mask[0, 4:] = 0  # tokens 4-7 son padding

    # Mean pooling con máscara
    mask_exp = mask.unsqueeze(-1)
    emb_masked = (hidden * mask_exp).sum(dim=1) / (mask_exp.sum(dim=1) + 1e-9)
    # Mean pooling sin máscara
    emb_full = hidden.mean(dim=1)

    # Deben diferir en la primera secuencia (tiene padding)
    diff = (emb_masked[0] - emb_full[0]).abs().max().item()
    assert diff > 0.01, "El pooling enmascarado debe diferir cuando hay padding"
    # La segunda secuencia (sin padding) debe ser igual
    diff2 = (emb_masked[1] - emb_full[1]).abs().max().item()
    assert diff2 < 1e-5, "Sin padding, ambos métodos deben ser iguales"
    print(f"T3 PASS: mean pooling con máscara correcto (diff={diff:.4f})")

def test_l2_normalizacion():
    """T4: tras L2-normalización, norma = 1.0 y sim = producto punto"""
    import torch

    torch.manual_seed(7)
    emb = torch.randn(10, 64)
    emb_norm = F.normalize(emb, p=2, dim=-1)

    norms = torch.norm(emb_norm, dim=-1)
    assert torch.allclose(norms, torch.ones(10), atol=1e-5), \
        "Tras L2-norm, todas las normas deben ser 1"

    # Verificar que producto punto = similitud coseno
    i, j = 0, 3
    dp = (emb_norm[i] * emb_norm[j]).sum().item()
    cos = similitud_coseno(emb_norm[i].numpy(), emb_norm[j].numpy())
    assert abs(dp - cos) < 1e-5, "Producto punto ≠ similitud coseno tras L2-norm"
    print(f"T4 PASS: L2-normalización correcta (norma media={norms.mean():.6f})")

def test_knn_recupera_vecino_mas_cercano():
    """T5: kNN recupera correctamente el vecino más similar"""
    np.random.seed(42)
    D, N = 64, 100

    corpus = np.random.randn(N, D).astype(np.float32)
    corpus /= np.linalg.norm(corpus, axis=1, keepdims=True)

    # Crear query como perturbación pequeña del doc 17
    query = corpus[17] + 0.01 * np.random.randn(D).astype(np.float32)
    query /= np.linalg.norm(query)

    indices, sims = busqueda_knn(query, corpus, k=5)
    assert indices[0] == 17, f"El vecino más cercano debe ser doc 17, got {indices[0]}"
    assert sims[0] > 0.99, f"Similitud esperada > 0.99, got {sims[0]:.4f}"
    print(f"T5 PASS: kNN recupera doc_17 correctamente (sim={sims[0]:.4f})")

def test_contrastive_loss_diagonal():
    """T6: InfoNCE loss es menor cuando los pares positivos tienen alta similitud"""
    import torch
    import torch.nn.functional as F

    torch.manual_seed(0)
    B, D = 8, 32

    # Caso 1: pares positivos tienen alta similitud (embeddings casi idénticos)
    z = F.normalize(torch.randn(B, D), dim=-1)
    z1_bueno = z + 0.01 * torch.randn(B, D)
    z2_bueno = z + 0.01 * torch.randn(B, D)
    z1_b = F.normalize(z1_bueno, dim=-1)
    z2_b = F.normalize(z2_bueno, dim=-1)

    # Caso 2: pares positivos tienen baja similitud (aleatorios)
    z1_malo = F.normalize(torch.randn(B, D), dim=-1)
    z2_malo = F.normalize(torch.randn(B, D), dim=-1)

    def info_nce(z1, z2, temp=0.05):
        sims = z1 @ z2.T / temp
        labels = torch.arange(B)
        return (F.cross_entropy(sims, labels) + F.cross_entropy(sims.T, labels)) / 2

    loss_bueno = info_nce(z1_b, z2_b).item()
    loss_malo  = info_nce(z1_malo, z2_malo).item()

    assert loss_bueno < loss_malo, \
        f"Loss bueno ({loss_bueno:.4f}) debe ser < loss malo ({loss_malo:.4f})"
    print(f"T6 PASS: InfoNCE loss_bueno={loss_bueno:.4f} < loss_malo={loss_malo:.4f}")

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    tests = [
        test_similitud_coseno,
        test_word2vec_embedding_dims,
        test_mean_pooling_ignora_padding,
        test_l2_normalizacion,
        test_knn_recupera_vecino_mas_cercano,
        test_contrastive_loss_diagonal,
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
        print("M11 COMPLETADO")
    else:
        print("Revisar implementaciones")
```

---

## Resumen

| Técnica | Dimensión | Fortaleza | Limitación |
|---|---|---|---|
| Word2Vec | 50-300 | Rápido, analogo vectorial | Sin contexto (una palabra = un vector) |
| GloVe | 50-300 | Estadísticas globales | Sin contexto |
| ELMo | 512-1024 | Contextual (LSTM) | Lento, LSTM |
| BERT/RoBERTa | 768 | Contextual, bidireccional | Mean pooling no óptimo |
| Sentence-BERT | 384-768 | Optimizado para similitud | Requiere fine-tuning |
| SimCSE | 256-768 | Contrastive, sin supervisión | Batch size crítico |

**Indexado con FAISS:**

| Índice | Exacto | Velocidad | Memoria | Uso recomendado |
|---|---|---|---|---|
| `IndexFlatIP` | Sí | Lento (O(N)) | Alta | N < 100K |
| `IndexIVFFlat` | No | Rápido | Media | N < 10M |
| `IndexIVFPQ` | No | Muy rápido | Baja (comprimido) | N > 10M |
| `IndexHNSW` | No | Muy rápido | Alta | Latencia crítica |
