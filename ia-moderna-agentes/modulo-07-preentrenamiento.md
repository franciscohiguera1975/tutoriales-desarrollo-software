# Módulo 07 — Pre-entrenamiento y Tokenización

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender el proceso de pre-entrenamiento de LLMs (datos, curriculum, scaling laws), implementar BPE desde cero, y entender las estrategias de tokenización modernas.
**Herramientas:** PyTorch 2.x, NumPy, collections
**Prerequisito:** M06 (Arquitectura LLMs)

---

## 1. El Pipeline de Pre-entrenamiento

```python
# seccion_01_pipeline_preentrenamiento.py
"""
Pre-entrenar un LLM requiere:

1. DATOS: billones de tokens de texto de calidad
   - Common Crawl (internet crawl, ~petabytes)
   - Books (BookCorpus, Project Gutenberg)
   - Wikipedia, GitHub, arXiv, StackExchange, etc.
   - Filtrado de calidad (perplexity filtering, dedup)

2. TOKENIZACIÓN: convertir texto → secuencia de enteros

3. ARQUITECTURA: Transformer decoder-only (M06)

4. OBJETIVO: predecir el siguiente token (Causal LM)
   L = -Σ_t log P(x_t | x_1,...,x_{t-1})

5. OPTIMIZACIÓN: AdamW + LR warmup + gradient clipping
   - Batch size: millones de tokens
   - LR: 1e-4 a 3e-4 con warmup + cosine decay

6. INFRAESTRUCTURA: cientos a miles de GPUs/TPUs
   - Data parallelism (batch dividido)
   - Model parallelism (capas divididas)
   - Tensor parallelism (atención y FFN divididos)

SCALING LAWS (Kaplan et al. 2020, Chinchilla 2022):
  L(N, D) ≈ A/N^α + B/D^β + C  (parámetros N, tokens D)

  Chinchilla: para N parámetros, entrenar con ~20N tokens
  → LLaMA-7B: 7B × 20 = 140B tokens mínimo
  → En práctica: 1T-2T tokens (más datos = mejor)
"""
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

# Scaling laws de Kaplan (simplificadas)
def loss_kaplan(N, D, A=406.4, B=410.7, alpha=0.34, beta=0.28, C_min=1.69):
    """
    L(N, D) = A/N^alpha + B/D^beta + C_min
    N: número de parámetros, D: número de tokens de entrenamiento
    """
    return A / N**alpha + B / D**beta + C_min

# Chinchilla: ratio óptimo D/N ≈ 20
N_values = np.logspace(8, 12, 50)  # 100M a 1T parámetros
D_kaplan   = N_values ** 0.74       # Kaplan: más parámetros, menos datos
D_chinchilla = 20 * N_values         # Chinchilla: 20 tokens por parámetro

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].loglog(N_values, D_kaplan, label="Kaplan (2020)", color='blue')
axes[0].loglog(N_values, D_chinchilla, label="Chinchilla (2022)", color='red', linestyle='--')
axes[0].set_xlabel("Parámetros N"); axes[0].set_ylabel("Tokens óptimos D")
axes[0].set_title("Scaling: Tokens óptimos vs Parámetros"); axes[0].legend()

# Pérdida vs tokens para distintos tamaños de modelo
for N, nombre in [(1e8, "100M"), (1e9, "1B"), (1e10, "10B")]:
    D_range = np.logspace(9, 13, 100)
    losses  = [loss_kaplan(N, D) for D in D_range]
    axes[1].semilogx(D_range, losses, label=nombre)
axes[1].set_xlabel("Tokens de entrenamiento D")
axes[1].set_ylabel("Loss")
axes[1].set_title("Loss vs Tokens por tamaño de modelo")
axes[1].legend(); axes[1].set_ylim(1.5, 4)

plt.tight_layout()
plt.savefig("outputs/scaling_laws.png", dpi=150)
print("✓ outputs/scaling_laws.png guardado")

print("\nModelos y sus datos de entrenamiento:")
print(f"{'Modelo':<15} {'Params':>8} {'Tokens':>12} {'Ratio D/N':>10}")
modelos = [
    ("GPT-3",      175,  300,   300/175),
    ("LLaMA-1 7B", 7,    1000,  1000/7),
    ("LLaMA-2 7B", 7,    2000,  2000/7),
    ("LLaMA-3 8B", 8,    15000, 15000/8),
    ("Mistral 7B", 7,    8000,  8000/7),
    ("Gemma 7B",   7,    6000,  6000/7),
]
for nombre, n_b, d_b, ratio in modelos:
    print(f"{nombre:<15} {n_b:>5}B {d_b:>8}B {'tokens'} {ratio:>9.0f}")
```

---

## 2. BPE — Byte Pair Encoding Completo

```python
# seccion_02_bpe_completo.py
"""
BPE (Sennrich et al. 2016): algoritmo estándar para tokenización de LLMs.

Pasos:
  1. Inicializar vocabulario con todos los bytes (0-255)
  2. Representar corpus como secuencia de bytes
  3. Encontrar el par de bytes más frecuente
  4. Fusionar ese par en un nuevo token
  5. Repetir hasta alcanzar vocab_size deseado

GPT-4 usa cl100k_base con 100,277 tokens.
"""
import re
from collections import Counter, defaultdict
import heapq

class BPETokenizer:
    """BPE desde cero — versión educativa completa."""

    def __init__(self, vocab_size=256 + 100):
        self.vocab_size = vocab_size
        self.merges = {}        # {(a, b): nuevo_token_id}
        self.vocab  = {}        # {token_str: id}
        self.idx_to_tok = {}    # {id: bytes}

    def _get_word_freqs(self, corpus):
        """Contar frecuencia de palabras en el corpus."""
        freq = Counter()
        for word in re.findall(r'\S+', corpus):
            # Cada palabra como tupla de bytes (enteros)
            freq[tuple(word.encode("utf-8"))] += 1
        return freq

    def _count_pairs(self, word_freqs):
        """Contar pares de tokens adyacentes ponderados por frecuencia."""
        pairs = Counter()
        for tokens, freq in word_freqs.items():
            for a, b in zip(tokens, tokens[1:]):
                pairs[(a, b)] += freq
        return pairs

    def _merge(self, word_freqs, best_pair, new_id):
        """Aplicar la fusión del mejor par en todo el vocabulario."""
        a, b = best_pair
        new_freqs = {}
        for tokens, freq in word_freqs.items():
            new_tokens = []
            i = 0
            while i < len(tokens):
                if i < len(tokens)-1 and tokens[i]==a and tokens[i+1]==b:
                    new_tokens.append(new_id)
                    i += 2
                else:
                    new_tokens.append(tokens[i])
                    i += 1
            new_freqs[tuple(new_tokens)] = freq
        return new_freqs

    def train(self, corpus):
        """Entrenar el tokenizador BPE en el corpus dado."""
        # Vocabulario base: bytes 0-255
        for i in range(256):
            self.vocab[bytes([i])] = i
            self.idx_to_tok[i]     = bytes([i])

        word_freqs = self._get_word_freqs(corpus)
        next_id    = 256

        n_merges = self.vocab_size - 256
        for merge_i in range(n_merges):
            pairs = self._count_pairs(word_freqs)
            if not pairs:
                break
            best = max(pairs, key=pairs.get)
            new_id = next_id + merge_i

            # Guardar la fusión
            self.merges[best] = new_id
            merged_bytes = self.idx_to_tok[best[0]] + self.idx_to_tok[best[1]]
            self.idx_to_tok[new_id] = merged_bytes
            self.vocab[merged_bytes] = new_id

            word_freqs = self._merge(word_freqs, best, new_id)

            if merge_i < 10:
                print(f"  Merge {merge_i+1}: {self.idx_to_tok[best[0]]} + "
                      f"{self.idx_to_tok[best[1]]} → {merged_bytes} (id={new_id})")

    def encode(self, text):
        """Tokenizar texto usando las fusiones aprendidas."""
        tokens = list(text.encode("utf-8"))
        # Aplicar fusiones en orden
        for (a, b), new_id in self.merges.items():
            i = 0
            while i < len(tokens) - 1:
                if tokens[i] == a and tokens[i+1] == b:
                    tokens[i] = new_id
                    del tokens[i+1]
                else:
                    i += 1
        return tokens

    def decode(self, ids):
        """Reconstruir texto desde ids."""
        return b"".join(self.idx_to_tok[i] for i in ids).decode("utf-8", errors="replace")

# Entrenar en corpus pequeño
corpus = """
la inteligencia artificial es el futuro de la tecnología
el aprendizaje automático permite a las máquinas aprender de los datos
los modelos de lenguaje grande transforman la forma en que procesamos el texto
la inteligencia artificial y el aprendizaje profundo son tecnologías clave
los transformers revolucionaron el procesamiento del lenguaje natural
"""

print("BPE Training:")
tokenizer = BPETokenizer(vocab_size=256 + 50)
tokenizer.train(corpus)

# Probar
texto = "la inteligencia artificial"
ids   = tokenizer.encode(texto)
dec   = tokenizer.decode(ids)
print(f"\nTexto original: '{texto}'")
print(f"IDs:            {ids}")
print(f"Decodificado:   '{dec}'")
print(f"Tokens por char: {len(ids)/len(texto):.2f}")
```

---

## 3. Sentencia Piece y Tiktoken

```python
# seccion_03_tokenizadores_modernos.py
"""
Tokenizadores modernos en la práctica.

SentencePiece (Google):
  - Usado por T5, LLaMA-1/2, Gemma
  - Trabaja a nivel de caracteres unicode (no bytes)
  - BPE o Unigram LM

Tiktoken (OpenAI):
  - Usado por GPT-3/4, ChatGPT
  - BPE a nivel de bytes con regex para pre-tokenización
  - Más rápido que implementaciones puras de Python

Características importantes:
  1. Manejo de espacios: ▁ (SentencePiece) o Ġ (GPT-2) al inicio de palabra
  2. Tokens especiales: <unk>, <pad>, <s>, </s>, [CLS], [SEP], <|endoftext|>
  3. Continuación de palabras: ▁hello → ["▁hello"], "world" → ["world"] vs "▁world"
"""
import re
from collections import Counter

# Demo: pre-tokenización con regex (como tiktoken)
def pretokenize_gpt4(text):
    """
    GPT-4 usa este patrón regex para pre-tokenizar antes de BPE.
    Separa: contracciones, letras, dígitos, espacios, otros.
    """
    pattern = re.compile(
        r"'(?:[sdmt]|ll|ve|re)|"   # contracciones inglesas
        r" ?[a-zA-Z]+|"             # palabras con espacio opcional
        r" ?[0-9]+|"                # números
        r" ?[^\s\w]+|"              # símbolos
        r"\s+(?!\S)|"               # espacios al final
        r"\s+",                     # otros espacios
        flags=re.UNICODE
    )
    return pattern.findall(text)

textos_prueba = [
    "Hello, world! How are you?",
    "I've got 3 items: apple, banana, cherry.",
    "def fibonacci(n):\n    return n if n<=1 else fib(n-1)+fib(n-2)",
    "La inteligencia artificial es fascinante.",
]

print("Pre-tokenización estilo GPT-4:")
for texto in textos_prueba:
    tokens = pretokenize_gpt4(texto)
    print(f"  '{texto[:50]}...' → {tokens[:8]}...")

# Análisis de eficiencia por idioma
print("\nEficiencia de tokenización (tokens por palabra) por idioma:")
print("(Tokenizadores ingleses son menos eficientes en otros idiomas)\n")

palabras_ejemplo = {
    "inglés":    ["the", "artificial", "intelligence", "is", "transforming"],
    "español":   ["la",  "inteligencia", "artificial", "está", "transformando"],
    "chino":     ["人工", "智能", "正在", "改变", "世界"],
    "árabe":     ["الذكاء", "الاصطناعي", "يغير", "العالم"],
}

for idioma, palabras in palabras_ejemplo.items():
    # Aproximación: chars por palabra vs tokens
    avg_chars = sum(len(w) for w in palabras) / len(palabras)
    # GPT-4 cl100k: ~4 chars/token inglés, menos eficiente otros idiomas
    factor = {"inglés": 4.0, "español": 3.5, "chino": 1.5, "árabe": 2.0}[idioma]
    tokens_aprox = sum(max(1, len(w.encode('utf-8'))//factor) for w in palabras)
    print(f"  {idioma:<10}: ~{tokens_aprox/len(palabras):.1f} tokens/palabra avg")

print("""
Impacto en costo:
  → Inglés: 1 token ≈ 4 caracteres ≈ 0.75 palabras
  → Chino/japonés: 1 token ≈ 1.5 caracteres (ideogramas = múltiples bytes)
  → Código: eficiente (palabras clave frecuentes forman tokens)
  → LLaMA-3 (128K vocab) es mucho más eficiente en no-inglés que LLaMA-2 (32K)
""")
```

---

## 4. Datos de Pre-entrenamiento y Filtrado

```python
# seccion_04_datos_filtrado.py
"""
Calidad de datos = calidad del modelo.
El filtrado y la curación son críticos.

Fuentes principales:
  Common Crawl: ~70% de los datos de la mayoría de LLMs
  Books:        ~8%  (razonamiento, coherencia larga)
  GitHub:       ~5%  (código, estructura lógica)
  Wikipedia:    ~3%  (conocimiento factual)
  arXiv:        ~2%  (razonamiento científico)
  StackExchange: ~2% (Q&A técnico)

Técnicas de filtrado:
  1. Deduplicación: eliminar duplicados (Min-Hash, suffix array)
  2. Quality filter: perplexity baja (GPT-2 score < umbral)
  3. Content filter: eliminar contenido dañino/spam/adulto
  4. Language detection: fastText para detectar idioma
  5. Heurísticas: ratio alfanumérico, longitud media de palabras, etc.
"""
import re
import math
from collections import Counter

def filtrar_calidad(texto, min_words=20, max_words=100000,
                    min_alpha_ratio=0.6, min_avg_word_len=3.0):
    """
    Filtros heurísticos de calidad de texto.
    Basado en los filtros usados en The Pile, RedPajama, etc.
    """
    palabras = texto.split()
    n_words  = len(palabras)
    filtros_aplicados = []

    # 1. Longitud mínima/máxima
    if n_words < min_words:
        return False, f"demasiado corto ({n_words} palabras)"
    if n_words > max_words:
        return False, f"demasiado largo ({n_words} palabras)"

    # 2. Ratio alfanumérico
    n_alpha = sum(c.isalpha() for c in texto)
    n_total = len(texto)
    if n_total == 0 or n_alpha / n_total < min_alpha_ratio:
        return False, f"bajo ratio alfa ({n_alpha/max(n_total,1):.2f})"

    # 3. Longitud media de palabras
    avg_len = sum(len(w) for w in palabras) / n_words
    if avg_len < min_avg_word_len:
        return False, f"palabras demasiado cortas (avg={avg_len:.1f})"

    # 4. Detección de spam/repetición
    unique_ratio = len(set(palabras)) / n_words
    if unique_ratio < 0.2:
        return False, f"alta repetición ({unique_ratio:.2f} unique)"

    return True, "ok"

# Perplexity filter con modelo de n-grams
class NGramLM:
    """Modelo de lenguaje n-gram para filtrar texto de baja calidad."""
    def __init__(self, n=2):
        self.n = n
        self.counts   = Counter()
        self.contexts = Counter()

    def train(self, corpus):
        words = corpus.lower().split()
        for i in range(len(words) - self.n + 1):
            ctx = tuple(words[i:i+self.n-1])
            tok = words[i+self.n-1]
            self.contexts[ctx] += 1
            self.counts[(ctx, tok)] += 1

    def perplexity(self, text, smoothing=1e-6):
        words = text.lower().split()
        if len(words) < self.n:
            return float('inf')
        log_prob = 0
        n_grams  = 0
        vocab_size = len(self.counts)
        for i in range(len(words) - self.n + 1):
            ctx = tuple(words[i:i+self.n-1])
            tok = words[i+self.n-1]
            c_ctx = self.contexts.get(ctx, 0)
            c_ctok = self.counts.get((ctx, tok), 0)
            p = (c_ctok + smoothing) / (c_ctx + smoothing * (vocab_size + 1))
            log_prob += math.log(p)
            n_grams  += 1
        return math.exp(-log_prob / n_grams) if n_grams > 0 else float('inf')

# Entrenar en corpus de referencia
corpus_ref = """
la inteligencia artificial es el estudio de sistemas computacionales que realizan
tareas que normalmente requieren inteligencia humana el aprendizaje automático permite
a las computadoras aprender de la experiencia sin ser programadas explícitamente
los modelos de lenguaje grande han revolucionado el procesamiento del lenguaje natural
"""

lm = NGramLM(n=2)
lm.train(corpus_ref)

textos_test = [
    ("Texto coherente en español sobre IA",
     "los modelos de lenguaje grande aprenden patrones del texto de entrenamiento"),
    ("Texto de spam/repetición",
     "compra ahora compra ahora compra ahora descuento descuento descuento oferta"),
    ("Texto demasiado corto", "hola mundo"),
    ("Texto con símbolos",
     "!!! >>> *** @@@ ### precio mejor oferta %%% descuento !!! compra ya >>>"),
]

print("Filtrado de calidad:")
for nombre, texto in textos_test:
    ok, razon = filtrar_calidad(texto)
    ppx = lm.perplexity(texto)
    print(f"  {nombre}:")
    print(f"    Filtro heurístico: {'✓ ok' if ok else '✗ ' + razon}")
    print(f"    Perplexidad:       {ppx:.1f}")
```

---

## 5. Curriculum de Entrenamiento

```python
# seccion_05_curriculum.py
"""
El orden y la mezcla de datos importan.

CURRICULUM TÍPICO para LLMs:
  Fase 1 — Pre-entrenamiento general:
    - Billones de tokens de web, libros, código, etc.
    - Batch size: incrementa gradualmente (1M → 4M tokens)
    - LR: warmup → peak → cosine decay

  Fase 2 — Domain adaptation (opcional):
    - Continuar pre-entrenando en dominio específico
    - Ejemplo: código (StarCoder), medicina, derecho

  Fase 3 — Supervised Fine-Tuning (SFT):
    - Miles a millones de ejemplos instruction-response
    - Enseñar al modelo a seguir instrucciones

  Fase 4 — Alignment (RLHF/DPO):
    - Comparaciones humanas → preferencias
    - PPO (RLHF) o DPO (más simple)

Data mixing (LLaMA-3 ejemplo aproximado):
  50% web (Common Crawl filtrado)
  25% código
  15% matemáticas y razonamiento
  10% curado de alta calidad (libros, Wikipedia, arXiv)

Annealing: al final del entrenamiento, aumentar el ratio de datos de alta calidad
"""
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# Visualizar LR schedule típico de LLMs
def lr_schedule(step, total_steps=100000, warmup=2000,
                lr_max=3e-4, lr_min=3e-5):
    if step < warmup:
        return lr_max * step / warmup
    elif step < total_steps:
        progress = (step - warmup) / (total_steps - warmup)
        return lr_min + 0.5 * (lr_max - lr_min) * (1 + np.cos(np.pi * progress))
    else:
        return lr_min

steps = np.arange(0, 100001, 100)
lrs   = [lr_schedule(s) for s in steps]

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].plot(steps, lrs)
axes[0].axvline(2000, color='r', linestyle='--', alpha=0.5, label='End warmup')
axes[0].set_title("LR Schedule: Warmup + Cosine Decay")
axes[0].set_xlabel("Training steps"); axes[0].set_ylabel("Learning Rate")
axes[0].legend()

# Data mixing pie chart
categorias = ["Web (Common Crawl)", "Código", "Matemáticas", "Alta calidad curada"]
proporciones = [0.50, 0.25, 0.15, 0.10]
axes[1].pie(proporciones, labels=categorias, autopct='%1.0f%%',
            colors=['#4CAF50', '#2196F3', '#FF9800', '#9C27B0'])
axes[1].set_title("Data Mixing típico LLM moderno")

plt.tight_layout()
plt.savefig("outputs/curriculum_entrenamiento.png", dpi=150)
print("✓ outputs/curriculum_entrenamiento.png guardado")

print("\nBatch size schedule (warmup de batch):")
print("  GPT-3 comienza con batch pequeño, lo incrementa")
print("  Batch grande → gradientes más estables, mejor uso de GPU")
print("  2^k tokens es típico: 256K → 1M → 4M tokens/batch")
```

---

## 6. Mini Pre-entrenamiento de Carácter

```python
# seccion_06_pretrain_caracter.py
"""
Pre-entrenamiento de un GPT mínimo en texto a nivel de carácter.
Basado en el famoso minGPT de Karpathy.
Tarea: modelado de lenguaje en un corpus pequeño.
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

torch.manual_seed(42)

# Corpus: fragmento de texto
corpus = """La inteligencia artificial es una rama de la informática
que busca crear sistemas capaces de realizar tareas que normalmente
requieren inteligencia humana. El aprendizaje automático, una subdisciplina
de la IA, permite a los sistemas aprender de los datos sin ser programados
explícitamente. Los modelos de lenguaje transforman la forma en que
interactuamos con las computadoras.""" * 3

# Vocabulario de caracteres
chars = sorted(set(corpus))
vocab_size = len(chars)
c2i = {c: i for i, c in enumerate(chars)}
i2c = {i: c for c, i in c2i.items()}
data = torch.tensor([c2i[c] for c in corpus], dtype=torch.long)

print(f"Corpus: {len(corpus)} caracteres, vocabulario: {vocab_size} chars")

# Modelo GPT mínimo
class CharGPT(nn.Module):
    def __init__(self, vocab, d=64, n_heads=4, n_layers=2, ctx=64):
        super().__init__()
        self.ctx = ctx
        self.emb = nn.Embedding(vocab, d)
        self.pe  = nn.Embedding(ctx, d)
        layers   = []
        for _ in range(n_layers):
            layers.extend([
                nn.LayerNorm(d),
                nn.MultiheadAttention(d, n_heads, batch_first=True, dropout=0.1),
                nn.LayerNorm(d),
                nn.Sequential(nn.Linear(d, d*4), nn.GELU(), nn.Linear(d*4, d)),
            ])
        self.blocks = nn.ModuleList(layers)
        self.norm   = nn.LayerNorm(d)
        self.head   = nn.Linear(d, vocab, bias=False)
        self.head.weight = self.emb.weight  # weight tying
        self.register_buffer("causal",
            torch.triu(torch.ones(ctx, ctx), diagonal=1).bool())

    def forward(self, x):
        B, T = x.shape
        tok  = self.emb(x) + self.pe(torch.arange(T, device=x.device))
        for i in range(0, len(self.blocks), 4):
            ln1, attn, ln2, ffn = self.blocks[i:i+4]
            x2 = ln1(tok)
            a, _ = attn(x2, x2, x2, attn_mask=self.causal[:T, :T])
            tok  = tok + a
            tok  = tok + ffn(ln2(tok))
        return self.head(self.norm(tok))

model = CharGPT(vocab_size)
opt   = torch.optim.AdamW(model.parameters(), lr=3e-3, weight_decay=0.01)

def get_batch(batch_size=32, ctx=64):
    idx = torch.randint(len(data)-ctx-1, (batch_size,))
    X   = torch.stack([data[i:i+ctx]   for i in idx])
    Y   = torch.stack([data[i+1:i+ctx+1] for i in idx])
    return X, Y

# Entrenamiento
model.train()
for step in range(1000):
    X, Y = get_batch()
    logits = model(X)
    loss   = F.cross_entropy(logits.view(-1, vocab_size), Y.view(-1))
    opt.zero_grad(); loss.backward(); opt.step()
    if step % 200 == 199:
        perp = torch.exp(loss).item()
        print(f"  Step {step+1:4d} | loss: {loss.item():.4f} | perplexity: {perp:.2f}")

# Generación
@torch.no_grad()
def generar(seed="La", n=100, temp=0.8):
    model.eval()
    ctx = torch.tensor([c2i.get(c, 0) for c in seed]).unsqueeze(0)
    result = seed
    for _ in range(n):
        logits = model(ctx[:, -64:])[:, -1, :] / temp
        tok    = torch.multinomial(F.softmax(logits, -1), 1)
        ctx    = torch.cat([ctx, tok], 1)
        result += i2c[tok.item()]
    return result

print("\nTexto generado:")
print(generar("La inteligencia ", n=150))

torch.save(model.state_dict(), "outputs/char_gpt.pt")
print("\n✓ outputs/char_gpt.pt guardado")
```

---

## Checkpoint M07

```python
# checkpoint_m07.py
"""
Tests de verificación para Módulo 07 — Pre-entrenamiento y Tokenización.
Ejecutar con: python checkpoint_m07.py
"""
import torch
import torch.nn.functional as F
from collections import Counter
import re
import math

def test_1_bpe_fusiona_par_frecuente():
    """BPE fusiona el par más frecuente correctamente."""
    # Par más frecuente en "aaab" representado como bytes
    vocab = Counter({(97, 97, 97, 98): 1,   # "aaab"
                     (97, 97): 3})            # "aa" extra
    # Par más frecuente: (97, 97)
    pairs = Counter()
    for tokens, freq in vocab.items():
        for a, b in zip(tokens, tokens[1:]):
            pairs[(a, b)] += freq
    best = max(pairs, key=pairs.get)
    assert best == (97, 97), f"Par más frecuente: {best}"
    print(f"✓ Test 1: BPE identifica par más frecuente: {chr(best[0])}{chr(best[1])}")

def test_2_encode_decode_roundtrip():
    """Encode → decode produce el texto original."""
    texto = "hello world"
    # Tokenizar a nivel de bytes (BPE base)
    ids   = list(texto.encode("utf-8"))
    dec   = bytes(ids).decode("utf-8")
    assert dec == texto
    print(f"✓ Test 2: Encode→Decode roundtrip: '{dec}'")

def test_3_causal_lm_loss():
    """Pérdida de Language Modeling: predicción del siguiente token."""
    vocab = 50
    B, T  = 2, 10
    logits  = torch.randn(B, T, vocab)
    targets = torch.randint(0, vocab, (B, T))
    # En LM: predecir tokens 1..T dado 0..T-1
    # logits[:, t, :] predice targets[:, t]
    loss = F.cross_entropy(logits.view(-1, vocab), targets.view(-1))
    assert 0 < loss.item() < 10, f"Loss fuera de rango: {loss.item()}"
    print(f"✓ Test 3: Causal LM loss en rango válido: {loss.item():.4f}")

def test_4_perplexity_mejor_con_mas_datos():
    """Más entrenamiento → menor perplexidad."""
    import torch.nn as nn

    V, d = 20, 32
    model = nn.Sequential(nn.Embedding(V, d),
                           nn.Linear(d, V))
    opt   = torch.optim.Adam(model.parameters(), lr=1e-2)

    def paso():
        x = torch.randint(0, V, (16, 8))
        y = torch.randint(0, V, (16, 8))
        out = model(x)
        if hasattr(out, 'view'):
            pass
        out = out.view(-1, V)
        loss = F.cross_entropy(out, y.view(-1))
        return loss

    ppx_ini = torch.exp(paso()).item()
    for _ in range(200):
        loss = paso(); opt.zero_grad(); loss.backward(); opt.step()
    ppx_fin = torch.exp(paso()).item()

    assert ppx_fin < ppx_ini, f"Perplexidad no decreció: {ppx_ini:.2f} → {ppx_fin:.2f}"
    print(f"✓ Test 4: Perplexidad decreció: {ppx_ini:.2f} → {ppx_fin:.2f}")

def test_5_scaling_law_mas_params_menos_loss():
    """Más parámetros → menor pérdida de validación (Scaling Laws)."""
    def loss_approx(N, D=1e10, A=406.4, alpha=0.34, B=410.7, beta=0.28, C=1.69):
        return A/N**alpha + B/D**beta + C

    loss_100M = loss_approx(1e8)
    loss_1B   = loss_approx(1e9)
    loss_10B  = loss_approx(1e10)

    assert loss_100M > loss_1B > loss_10B, "Scaling law: más params → menos loss"
    print(f"✓ Test 5: Scaling law: L(100M)={loss_100M:.3f} > L(1B)={loss_1B:.3f} > L(10B)={loss_10B:.3f}")

def test_6_filtro_calidad():
    """El filtro rechaza texto de baja calidad."""
    def filtrar(texto, min_words=5, min_alpha=0.5):
        words = texto.split()
        if len(words) < min_words: return False
        n_alpha = sum(c.isalpha() for c in texto)
        if n_alpha / max(len(texto), 1) < min_alpha: return False
        return True

    assert filtrar("hola mundo esto es una frase")     == True
    assert filtrar("!!!@@@###")                         == False  # sin alfa
    assert filtrar("ok")                               == False  # muy corto
    assert filtrar("a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5")  == True   # mezcla ok
    print("✓ Test 6: Filtro de calidad funciona correctamente")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M07 — Pre-entrenamiento y Tokenización")
    print("=" * 50)
    test_1_bpe_fusiona_par_frecuente()
    test_2_encode_decode_roundtrip()
    test_3_causal_lm_loss()
    test_4_perplexity_mejor_con_mas_datos()
    test_5_scaling_law_mas_params_menos_loss()
    test_6_filtro_calidad()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M07 completado")
```

---

## Resumen del Módulo

| Concepto | Detalle |
|---|---|
| **Objetivo LM** | `L = -Σ log P(x_t | x_{<t})` — predecir siguiente token |
| **BPE** | Fusionar pares más frecuentes; vocabulario 32K-128K tokens |
| **Scaling Laws** | L ∝ N^{-α} + D^{-β}; Chinchilla: D ≈ 20N tokens óptimos |
| **Filtrado** | Dedup + ratio alfanumérico + perplexidad + longitud |
| **Data mixing** | ~50% web, 25% código, 15% math, 10% curado |
| **LR schedule** | Warmup lineal → peak → cosine decay hasta `lr_min` |
| **Batch schedule** | Batch pequeño → batch grande (más estabilidad) |
| **Weight tying** | `lm_head.weight = embed.weight` — ahorra parámetros |

**Próximo módulo:** M08 — Prompt Engineering: Zero-shot, Few-shot, CoT, ToT
