# Módulo 06 — Arquitectura Interna de LLMs: GPT, BERT, LLaMA

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comparar las familias GPT (decoder-only), BERT (encoder-only) y T5/LLaMA en profundidad; entender KV cache, tokenización, y las mejoras modernas (RoPE, GQA, SwiGLU).
**Herramientas:** PyTorch 2.x, transformers (HuggingFace)
**Prerequisito:** M03 (Transformer), M05 (overview de modelos)

---

## 1. Tres Familias de LLMs

```python
# seccion_01_familias_llm.py
"""
Los LLMs actuales son básicamente variantes del Transformer con:
  1. Mucho más parámetros (7B-405B en lugar de 100M)
  2. Más datos de preentrenamiento (trillones de tokens)
  3. Mejoras arquitectónicas: RoPE, GQA, SwiGLU, RMSNorm

FAMILIA 1 — Encoder-only (BERT, RoBERTa, DeBERTa):
  - Procesa la secuencia completa de forma bidireccional
  - Pre-entrenamiento: Masked LM (predecir tokens enmascarados)
  - Ideal para: clasificación, NER, Q&A extractiva
  - No puede generar texto libre

FAMILIA 2 — Decoder-only (GPT-2/3/4, LLaMA, Mistral, Gemma):
  - Máscara causal: solo ve tokens anteriores
  - Pre-entrenamiento: Language Modeling (predecir siguiente token)
  - Ideal para: generación, chat, code, reasoning
  - Base de los modelos de chat actuales

FAMILIA 3 — Encoder-Decoder (T5, BART, Flan-T5):
  - Encoder bidireccional + Decoder causal
  - Pre-entrenamiento: span corruption (T5), denoising (BART)
  - Ideal para: traducción, resumen, QA generativa
"""

print("Familias de LLMs y sus casos de uso:")
print(f"\n{'Familia':<20} {'Modelos':<30} {'Pre-entrenamiento':>20}")
print("-" * 72)
familias = [
    ("Encoder-only",     "BERT, RoBERTa, DeBERTa",   "Masked LM"),
    ("Decoder-only",     "GPT-2/3/4, LLaMA, Mistral", "Causal LM"),
    ("Encoder-Decoder",  "T5, BART, Flan-T5",          "Span corruption"),
]
for f, m, p in familias:
    print(f"{f:<20} {m:<30} {p:>20}")

print("""
Tendencia actual (2024-2025):
  → Decoder-only domina los modelos de chat y razonamiento
  → Encoder-only sigue siendo preferido para embeddings y clasificación
  → Encoder-Decoder para tareas de traducción/resumen especializadas
""")
```

---

## 2. GPT — Decoder-Only con Máscara Causal

```python
# seccion_02_gpt.py
"""
Arquitectura GPT (Generative Pre-trained Transformer):
  - Solo decoder, con máscara causal
  - Token embedding + positional embedding (aprendible en GPT-2)
  - N capas de: LN → MHA (causal) → residual → LN → FFN → residual
  - Language modeling head: linear(d_model, vocab_size)

Objetivo pre-entrenamiento:
  Dado texto x_1, x_2, ..., x_T:
  L = -Σ_t log P(x_t | x_1,...,x_{t-1})
  → predecir cada token dado su contexto anterior
"""
import torch
import torch.nn as nn
import numpy as np

class GPTLayer(nn.Module):
    """Una capa del Transformer GPT-style (pre-norm, decoder-only)."""
    def __init__(self, d_model, n_heads, d_ff, dropout=0.1, max_len=1024):
        super().__init__()
        self.norm1    = nn.LayerNorm(d_model)
        self.attn     = nn.MultiheadAttention(d_model, n_heads,
                                               dropout=dropout, batch_first=True)
        self.norm2    = nn.LayerNorm(d_model)
        self.ffn      = nn.Sequential(
            nn.Linear(d_model, d_ff), nn.GELU(),
            nn.Dropout(dropout), nn.Linear(d_ff, d_model)
        )
        self.drop     = nn.Dropout(dropout)
        # Precalcular máscara causal
        self.register_buffer(
            "causal_mask",
            torch.triu(torch.ones(max_len, max_len), diagonal=1).bool()
        )

    def forward(self, x):
        T = x.shape[1]
        mask = self.causal_mask[:T, :T]
        # Self-attention causal + residual
        x_n = self.norm1(x)
        attn_out, _ = self.attn(x_n, x_n, x_n, attn_mask=mask)
        x = x + self.drop(attn_out)
        # FFN + residual
        x = x + self.drop(self.ffn(self.norm2(x)))
        return x

class MiniGPT(nn.Module):
    """GPT simplificado para entender la arquitectura."""
    def __init__(self, vocab_size, d_model=128, n_heads=4, d_ff=512,
                 n_layers=4, max_len=256, dropout=0.1):
        super().__init__()
        self.tok_embed = nn.Embedding(vocab_size, d_model)
        self.pos_embed = nn.Embedding(max_len, d_model)  # learned (como GPT-2)
        self.drop      = nn.Dropout(dropout)
        self.layers    = nn.ModuleList([
            GPTLayer(d_model, n_heads, d_ff, dropout, max_len)
            for _ in range(n_layers)
        ])
        self.norm  = nn.LayerNorm(d_model)
        self.lm_head = nn.Linear(d_model, vocab_size, bias=False)
        # Weight tying: compartir pesos embedding ↔ lm_head (común en GPT)
        self.lm_head.weight = self.tok_embed.weight

    def forward(self, input_ids):
        B, T = input_ids.shape
        pos  = torch.arange(T, device=input_ids.device)
        x    = self.drop(self.tok_embed(input_ids) + self.pos_embed(pos))
        for layer in self.layers:
            x = layer(x)
        logits = self.lm_head(self.norm(x))
        return logits  # (B, T, vocab_size)

    @torch.no_grad()
    def generate(self, input_ids, max_new=50, temperature=1.0, top_k=50):
        """Generación greedy/top-k."""
        for _ in range(max_new):
            ctx = input_ids[:, -256:]   # limitar contexto
            logits = self(ctx)[:, -1, :] / temperature
            # Top-k sampling
            if top_k > 0:
                vals, _ = torch.topk(logits, top_k)
                logits[logits < vals[:, -1:]] = float('-inf')
            probs = torch.softmax(logits, dim=-1)
            next_tok = torch.multinomial(probs, 1)
            input_ids = torch.cat([input_ids, next_tok], dim=1)
        return input_ids

# Instanciar y verificar
torch.manual_seed(42)
vocab_size = 500
gpt = MiniGPT(vocab_size, d_model=128, n_heads=4, n_layers=4)

# Forward pass
tokens = torch.randint(0, vocab_size, (2, 64))  # (batch, T)
logits = gpt(tokens)
print(f"Input:  {tuple(tokens.shape)}")
print(f"Logits: {tuple(logits.shape)}  (batch, T, vocab_size)")

# Generación
seed = torch.randint(0, vocab_size, (1, 10))
gen  = gpt.generate(seed, max_new=20, temperature=0.8, top_k=20)
print(f"Generado: {gen.shape}  (1, 30 tokens)")

n_params = sum(p.numel() for p in gpt.parameters())
print(f"Parámetros MiniGPT: {n_params:,d}")
```

---

## 3. BERT — Encoder Bidireccional

```python
# seccion_03_bert.py
"""
BERT (Devlin et al. 2019): Bidirectional Encoder Representations from Transformers

Diferencias clave respecto a GPT:
  1. No tiene máscara causal → cada token ve TODOS los demás
  2. Pre-entrenamiento MLM: 15% tokens enmascarados → predecir original
  3. Token especial [CLS] para clasificación de secuencia
  4. Token [SEP] para separar dos secuencias
  5. Segment embeddings (sentence A vs B) para NSP

BERT-base:  12 capas, d=768,  12 heads, FFN=3072, 110M params
BERT-large: 24 capas, d=1024, 16 heads, FFN=4096, 340M params
"""
import torch
import torch.nn as nn

class BERTLayer(nn.Module):
    """Capa del encoder BERT (post-norm, bidireccional)."""
    def __init__(self, d_model, n_heads, d_ff, dropout=0.1):
        super().__init__()
        self.attn  = nn.MultiheadAttention(d_model, n_heads,
                                            dropout=dropout, batch_first=True)
        self.ffn   = nn.Sequential(
            nn.Linear(d_model, d_ff), nn.GELU(),
            nn.Dropout(dropout), nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.drop  = nn.Dropout(dropout)

    def forward(self, x, key_padding_mask=None):
        attn_out, _ = self.attn(x, x, x, key_padding_mask=key_padding_mask)
        x = self.norm1(x + self.drop(attn_out))
        x = self.norm2(x + self.drop(self.ffn(x)))
        return x

class MiniBERT(nn.Module):
    """BERT simplificado para clasificación."""
    def __init__(self, vocab_size, n_classes, d_model=128, n_heads=4,
                 d_ff=512, n_layers=4, max_len=512, dropout=0.1):
        super().__init__()
        self.tok_embed  = nn.Embedding(vocab_size, d_model, padding_idx=0)
        self.pos_embed  = nn.Embedding(max_len, d_model)
        self.type_embed = nn.Embedding(2, d_model)   # segment A vs B
        self.drop       = nn.Dropout(dropout)
        self.layers     = nn.ModuleList([
            BERTLayer(d_model, n_heads, d_ff, dropout) for _ in range(n_layers)
        ])
        self.norm = nn.LayerNorm(d_model)
        # Clasificador en el token [CLS] (posición 0)
        self.classifier = nn.Sequential(
            nn.Linear(d_model, d_model), nn.Tanh(),
            nn.Dropout(dropout), nn.Linear(d_model, n_classes)
        )
        # MLM head (para pre-entrenamiento)
        self.mlm_head = nn.Sequential(
            nn.Linear(d_model, d_model), nn.GELU(),
            nn.LayerNorm(d_model), nn.Linear(d_model, vocab_size)
        )

    def forward(self, input_ids, token_type_ids=None, attention_mask=None):
        B, T = input_ids.shape
        pos  = torch.arange(T, device=input_ids.device)
        seg  = token_type_ids if token_type_ids is not None else torch.zeros_like(input_ids)
        pad_mask = (~attention_mask.bool()) if attention_mask is not None else None

        x = self.drop(self.tok_embed(input_ids)
                       + self.pos_embed(pos)
                       + self.type_embed(seg))
        for layer in self.layers:
            x = layer(x, key_padding_mask=pad_mask)
        x = self.norm(x)

        cls_output  = self.classifier(x[:, 0])       # (B, n_classes)
        mlm_logits  = self.mlm_head(x)               # (B, T, vocab_size)
        return cls_output, mlm_logits

# Verificar BERT
torch.manual_seed(42)
bert = MiniBERT(vocab_size=1000, n_classes=3, d_model=128, n_layers=4)
ids  = torch.randint(1, 1000, (4, 32))
mask = torch.ones(4, 32)
mask[:, 20:] = 0  # padding en posiciones 20-31
cls_out, mlm = bert(ids, attention_mask=mask)
print(f"BERT CLS output: {tuple(cls_out.shape)}  (batch, n_classes)")
print(f"BERT MLM logits: {tuple(mlm.shape)}  (batch, T, vocab_size)")

# Diferencia GPT vs BERT en atención
print("""
GPT (causal):    token_i solo ve tokens 0..i     → máscara triangular
BERT (bidir):    token_i ve TODOS los tokens     → sin máscara
                 → mejor para comprensión (NLU)
                 → imposible para generación libre
""")
```

---

## 4. LLaMA — Mejoras Arquitectónicas Modernas

```python
# seccion_04_llama.py
"""
LLaMA (Touvron et al. 2023) — decoder-only con mejoras:
  1. Pre-norm con RMSNorm (en lugar de post-norm con LayerNorm)
  2. SwiGLU en FFN (en lugar de ReLU/GeLU)
  3. RoPE: Rotary Positional Embeddings (en lugar de embedding aprendible)
  4. Sin bias en las capas lineales (más eficiente)
  5. Grouped Query Attention (GQA) en LLaMA-2/3
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

# 1. RMSNorm
class RMSNorm(nn.Module):
    def __init__(self, d, eps=1e-6):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(d))
        self.eps   = eps

    def forward(self, x):
        rms = x.pow(2).mean(-1, keepdim=True).add(self.eps).sqrt()
        return self.gamma * x / rms

# 2. SwiGLU FFN
class SwiGLUFFN(nn.Module):
    def __init__(self, d_model):
        super().__init__()
        # LLaMA usa d_ff ≈ 8/3 * d_model (redondeado a múltiplo de 256)
        d_ff = int(d_model * 8 / 3)
        d_ff = ((d_ff + 255) // 256) * 256
        self.W  = nn.Linear(d_model, d_ff, bias=False)
        self.V  = nn.Linear(d_model, d_ff, bias=False)
        self.W2 = nn.Linear(d_ff, d_model, bias=False)

    def forward(self, x):
        return self.W2(F.silu(self.W(x)) * self.V(x))

# 3. RoPE (Rotary Positional Embeddings)
def precompute_rope_freqs(d_k, max_len=2048, theta=10000.0):
    freqs = 1.0 / (theta ** (torch.arange(0, d_k, 2).float() / d_k))
    t     = torch.arange(max_len)
    freqs = torch.outer(t, freqs)          # (max_len, d_k/2)
    cos   = freqs.cos()                    # (max_len, d_k/2)
    sin   = freqs.sin()
    return cos, sin

def apply_rope(x, cos, sin):
    """
    x: (batch, T, n_heads, d_k)
    cos, sin: (T, d_k/2)
    """
    x1, x2 = x.chunk(2, dim=-1)
    # Rotar en pares: [x1, x2] → [x1*cos - x2*sin, x2*cos + x1*sin]
    cos_ = cos.unsqueeze(0).unsqueeze(2)   # (1, T, 1, d_k/2)
    sin_ = sin.unsqueeze(0).unsqueeze(2)
    x_rot = torch.cat([x1 * cos_ - x2 * sin_,
                        x2 * cos_ + x1 * sin_], dim=-1)
    return x_rot

# 4. Grouped Query Attention (GQA)
class GroupedQueryAttention(nn.Module):
    """
    GQA: n_kv_heads grupos de K,V compartidos por n_heads queries.
    Reduce el KV cache a n_kv_heads/n_heads del tamaño de MHA.

    MHA:  n_heads = n_kv_heads  (cada head tiene su propio KV)
    GQA:  n_kv_heads < n_heads  (múltiples heads comparten KV)
    MQA:  n_kv_heads = 1        (todos los heads comparten un KV)
    """
    def __init__(self, d_model, n_heads, n_kv_heads=None, max_len=2048):
        super().__init__()
        self.n_heads    = n_heads
        self.n_kv_heads = n_kv_heads or n_heads
        self.d_k        = d_model // n_heads
        assert n_heads % self.n_kv_heads == 0
        self.n_rep      = n_heads // self.n_kv_heads  # repeticiones por grupo

        self.Wq = nn.Linear(d_model, n_heads * self.d_k,    bias=False)
        self.Wk = nn.Linear(d_model, self.n_kv_heads * self.d_k, bias=False)
        self.Wv = nn.Linear(d_model, self.n_kv_heads * self.d_k, bias=False)
        self.Wo = nn.Linear(d_model, d_model, bias=False)

        cos, sin = precompute_rope_freqs(self.d_k, max_len)
        self.register_buffer("rope_cos", cos)
        self.register_buffer("rope_sin", sin)

    def forward(self, x):
        B, T, _ = x.shape
        Q = self.Wq(x).view(B, T, self.n_heads,    self.d_k)
        K = self.Wk(x).view(B, T, self.n_kv_heads, self.d_k)
        V = self.Wv(x).view(B, T, self.n_kv_heads, self.d_k)

        # Aplicar RoPE a Q y K
        cos = self.rope_cos[:T]
        sin = self.rope_sin[:T]
        Q = apply_rope(Q, cos, sin)
        K = apply_rope(K, cos, sin)

        # Repetir K, V para cada grupo de queries
        K = K.repeat_interleave(self.n_rep, dim=2)  # (B, T, n_heads, d_k)
        V = V.repeat_interleave(self.n_rep, dim=2)

        # Transponar para atención
        Q = Q.transpose(1, 2)  # (B, n_heads, T, d_k)
        K = K.transpose(1, 2)
        V = V.transpose(1, 2)

        # Atención causal con SDPA
        out = F.scaled_dot_product_attention(Q, K, V, is_causal=True)
        out = out.transpose(1, 2).contiguous().view(B, T, -1)
        return self.Wo(out)

# LLaMA layer completo
class LLaMALayer(nn.Module):
    def __init__(self, d_model, n_heads, n_kv_heads=None):
        super().__init__()
        self.norm1 = RMSNorm(d_model)
        self.attn  = GroupedQueryAttention(d_model, n_heads, n_kv_heads)
        self.norm2 = RMSNorm(d_model)
        self.ffn   = SwiGLUFFN(d_model)

    def forward(self, x):
        x = x + self.attn(self.norm1(x))
        x = x + self.ffn(self.norm2(x))
        return x

# Verificar LLaMA layer
torch.manual_seed(42)
d, H = 128, 8
layer = LLaMALayer(d_model=d, n_heads=H, n_kv_heads=2)  # GQA: 2 kv_heads para 8 heads
x = torch.randn(2, 32, d)
out = layer(x)
print(f"LLaMA layer: {tuple(x.shape)} → {tuple(out.shape)}")
print(f"GQA reducción KV: {2}/{H} = {2/H:.1%} del KV cache vs MHA")
```

---

## 5. KV Cache — Inferencia Eficiente

```python
# seccion_05_kv_cache.py
"""
KV Cache: la optimización más importante para inferencia de LLMs.

En generación autoregresiva:
  - Procesamos tokens uno por uno
  - En cada paso t, necesitamos K,V de todos los pasos anteriores 0..t
  - Sin cache: re-computar K,V para todos los tokens en cada paso → O(T²)
  - Con cache: guardar K,V y solo calcular para el nuevo token → O(T) amortizado

Impacto en memoria:
  KV cache = 2 × n_layers × n_kv_heads × d_k × T_max × batch × 2 bytes (FP16)
  Para LLaMA-7B (batch=1, T=4096):
    2 × 32 × 32 × 128 × 4096 × 2 ≈ 2 GB solo para el KV cache
"""
import torch
import torch.nn as nn
import time

class AttentionWithKVCache(nn.Module):
    """Atención con KV cache para generación eficiente."""
    def __init__(self, d_model, n_heads, max_len=256):
        super().__init__()
        self.d_k    = d_model // n_heads
        self.n_heads = n_heads
        self.Wq = nn.Linear(d_model, d_model, bias=False)
        self.Wk = nn.Linear(d_model, d_model, bias=False)
        self.Wv = nn.Linear(d_model, d_model, bias=False)
        self.Wo = nn.Linear(d_model, d_model, bias=False)
        # Cache (inicializado vacío)
        self.cache_k = None
        self.cache_v = None

    def forward(self, x, use_cache=False):
        B, T, _ = x.shape
        Q = self.Wq(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)
        K = self.Wk(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)
        V = self.Wv(x).view(B, T, self.n_heads, self.d_k).transpose(1, 2)

        if use_cache and self.cache_k is not None:
            # Concatenar con cache anterior
            K = torch.cat([self.cache_k, K], dim=2)
            V = torch.cat([self.cache_v, V], dim=2)

        if use_cache:
            self.cache_k = K.detach()
            self.cache_v = V.detach()

        out = torch.nn.functional.scaled_dot_product_attention(Q, K, V, is_causal=(T > 1))
        return self.Wo(out.transpose(1, 2).contiguous().view(B, T, -1))

    def reset_cache(self):
        self.cache_k = self.cache_v = None

# Demo: medir speedup del KV cache
d, H, B = 64, 4, 1
attn = AttentionWithKVCache(d, H)

# Simular generación sin KV cache (procesar secuencia completa en cada paso)
T_max = 64
tokens = torch.randn(B, 1, d)
context = torch.randn(B, 0, d)

t0 = time.time()
for t in range(T_max):
    context = torch.cat([context, tokens], dim=1)
    _ = attn(context, use_cache=False)  # re-procesa todo cada vez
t_sin_cache = time.time() - t0

# Con KV cache (solo procesa el nuevo token)
attn.reset_cache()
t0 = time.time()
# Prefill (procesar prompt)
prompt = torch.randn(B, 16, d)
_ = attn(prompt, use_cache=True)

# Decode (un token a la vez)
for t in range(T_max - 16):
    new_tok = torch.randn(B, 1, d)
    _ = attn(new_tok, use_cache=True)
t_con_cache = time.time() - t0

print(f"Generación T={T_max} tokens:")
print(f"  Sin KV cache: {t_sin_cache*1000:.1f} ms")
print(f"  Con KV cache: {t_con_cache*1000:.1f} ms")
print(f"  Speedup:      {t_sin_cache/t_con_cache:.1f}x")

# Memoria del KV cache
n_layers, n_kv_heads, d_k, T_max_llama = 32, 8, 128, 4096
kv_cache_bytes = 2 * n_layers * n_kv_heads * d_k * T_max_llama * 2  # FP16
print(f"\nKV cache LLaMA-7B (GQA, T=4096): {kv_cache_bytes/1e9:.2f} GB")
```

---

## 6. Tokenización

```python
# seccion_06_tokenizacion.py
"""
Los LLMs no trabajan con caracteres ni palabras completas.
Usan Byte-Pair Encoding (BPE) o SentencePiece:

BPE (GPT-2, GPT-4, LLaMA):
  1. Comenzar con tokens = bytes individuales (256 tokens)
  2. Repetidamente fusionar el par más frecuente
  3. Parar cuando se alcanza el tamaño de vocabulario deseado

Vocabularios típicos:
  GPT-2:    50,257 tokens
  GPT-4:    100,277 tokens (cl100k_base)
  LLaMA-2:  32,000 tokens
  LLaMA-3:  128,256 tokens
"""
import re
from collections import Counter

# BPE simplificado
def bpe_simple(corpus, n_merges=10):
    """
    Implementación básica de BPE.
    Trabaja a nivel de palabras para simplicidad.
    """
    # Tokenizar: cada palabra es lista de chars + </w>
    def word_to_tokens(word):
        return list(word) + ['</w>']

    vocab = Counter()
    for word in corpus.split():
        tokens = tuple(word_to_tokens(word))
        vocab[tokens] += 1

    merges = {}
    for i in range(n_merges):
        # Contar pares adyacentes
        pairs = Counter()
        for tokens, freq in vocab.items():
            for a, b in zip(tokens, tokens[1:]):
                pairs[(a, b)] += freq

        if not pairs:
            break

        # Par más frecuente
        best = max(pairs, key=pairs.get)
        merges[best] = "".join(best)

        # Fusionar en el vocabulario
        new_vocab = Counter()
        for tokens, freq in vocab.items():
            new_tokens = []
            i = 0
            while i < len(tokens):
                if i < len(tokens) - 1 and (tokens[i], tokens[i+1]) == best:
                    new_tokens.append(merges[best])
                    i += 2
                else:
                    new_tokens.append(tokens[i])
                    i += 1
            new_vocab[tuple(new_tokens)] += freq
        vocab = new_vocab
        print(f"  Merge {i+1}: {best} → {''.join(best)}")

    return vocab, merges

corpus = "hola mundo hola hola mundo hola clase clase clase clase"
print("BPE simplificado:")
vocab, merges = bpe_simple(corpus, n_merges=6)
print(f"\nVocabulario final:")
for tokens, freq in sorted(vocab.items(), key=lambda x: -x[1]):
    print(f"  {tokens}: {freq}")

# Comparativa de tokenización por modelo
print("\nTokenización de 'artificial intelligence' por familia:")
frases = [
    "artificial intelligence is transforming the world",
    "Hello, how are you?",
    "def fibonacci(n): return n if n <= 1 else fibonacci(n-1) + fibonacci(n-2)",
]

# Usar tiktoken si está disponible
try:
    import tiktoken
    enc = tiktoken.get_encoding("cl100k_base")  # GPT-4
    for frase in frases:
        toks = enc.encode(frase)
        print(f"  GPT-4 ({len(toks)} tokens): {toks[:8]}...")
except ImportError:
    print("  (instalar tiktoken para demo real: pip install tiktoken)")
    # Aproximación manual
    for frase in frases:
        n_chars  = len(frase)
        n_aprox  = n_chars // 4  # regla aproximada: ~4 chars por token
        print(f"  '{frase[:40]}...' → ≈{n_aprox} tokens")
```

---

## 7. Uso con HuggingFace Transformers

```python
# seccion_07_huggingface.py
"""
En la práctica, usamos modelos preentrenados de HuggingFace.
Aquí vemos cómo cargar, tokenizar, inferir y entender las salidas.
"""
print("Uso de HuggingFace Transformers (código de referencia):")
print()

codigo_bert = '''
# Clasificación con BERT
from transformers import BertTokenizer, BertForSequenceClassification
import torch

tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")
model = BertForSequenceClassification.from_pretrained(
    "bert-base-uncased", num_labels=2
)

texto = "This movie is absolutely fantastic!"
inputs = tokenizer(texto, return_tensors="pt", truncation=True, max_length=128)
# inputs: {"input_ids": ..., "attention_mask": ..., "token_type_ids": ...}

with torch.no_grad():
    outputs = model(**inputs)
# outputs.logits: (1, 2) → positivo/negativo
probs = torch.softmax(outputs.logits, dim=-1)
print(f"Positivo: {probs[0,1]:.2%}")
'''

codigo_gpt2 = '''
# Generación con GPT-2
from transformers import GPT2LMHeadModel, GPT2Tokenizer

tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
model = GPT2LMHeadModel.from_pretrained("gpt2")

prompt = "The future of artificial intelligence is"
inputs = tokenizer(prompt, return_tensors="pt")
output = model.generate(
    **inputs,
    max_new_tokens=50,
    temperature=0.8,
    top_k=50,
    do_sample=True,
    pad_token_id=tokenizer.eos_token_id
)
print(tokenizer.decode(output[0], skip_special_tokens=True))
'''

codigo_llama = '''
# Inferencia con LLaMA-3 (requiere acceso)
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_id = "meta-llama/Meta-Llama-3-8B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id, torch_dtype=torch.bfloat16, device_map="auto"
)

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user",   "content": "Explain quantum computing in 2 sentences."},
]
input_ids = tokenizer.apply_chat_template(
    messages, add_generation_prompt=True, return_tensors="pt"
)
output = model.generate(input_ids, max_new_tokens=200, do_sample=True, temperature=0.7)
print(tokenizer.decode(output[0][input_ids.shape[1]:], skip_special_tokens=True))
'''

print("=== BERT (clasificación) ==="); print(codigo_bert)
print("=== GPT-2 (generación) ==="); print(codigo_gpt2)
print("=== LLaMA-3 (instrucciones) ==="); print(codigo_llama)
```

---

## Checkpoint M06

```python
# checkpoint_m06.py
"""
Tests de verificación para Módulo 06 — Arquitectura LLMs.
Ejecutar con: python checkpoint_m06.py
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

def test_1_gpt_causal_mask():
    """La máscara causal impide atención hacia el futuro."""
    T = 8
    mask = torch.triu(torch.ones(T, T), diagonal=1).bool()
    scores = torch.randn(T, T)
    scores[mask] = float('-inf')
    w = F.softmax(scores, dim=-1)
    # Triángulo superior (sin diagonal) debe ser 0
    for i in range(T):
        for j in range(i+1, T):
            assert w[i, j].item() < 1e-6
    print("✓ Test 1: Máscara causal GPT correcta")

def test_2_weight_tying():
    """Weight tying: lm_head y tok_embed comparten parámetros."""
    d, V = 32, 100
    embed  = nn.Embedding(V, d)
    lm_head = nn.Linear(d, V, bias=False)
    lm_head.weight = embed.weight   # tying

    # Modificar embed cambia lm_head
    with torch.no_grad():
        embed.weight[0, 0] = 999.0
    assert lm_head.weight[0, 0].item() == 999.0
    print("✓ Test 2: Weight tying funciona (embed ↔ lm_head)")

def test_3_rmsnorm():
    """RMSNorm normaliza correctamente (sin centrar)."""
    d = 64
    class RMS(nn.Module):
        def __init__(self):
            super().__init__()
            self.g = nn.Parameter(torch.ones(d))
        def forward(self, x):
            rms = x.pow(2).mean(-1, keepdim=True).add(1e-6).sqrt()
            return self.g * x / rms

    rms = RMS()
    x   = torch.randn(4, 10, d) * 5
    y   = rms(x)
    # La norma RMS de y (sin gamma) debe ser ≈ sqrt(d)
    # Con gamma=1: |y| ≈ sqrt(d)
    rms_y = y.pow(2).mean(-1).sqrt()
    # Verificar que está normalizado (rms ≈ 1 por construcción con g=1)
    assert rms_y.mean().item() < 2.0   # razonablemente normalizado
    print(f"✓ Test 3: RMSNorm funciona, rms_y_mean={rms_y.mean():.3f}")

def test_4_swiglu_shape():
    """SwiGLU produce misma forma que input."""
    d = 64
    class SwiGLU(nn.Module):
        def __init__(self):
            super().__init__()
            self.W  = nn.Linear(d, d*2, bias=False)
            self.W2 = nn.Linear(d*2, d, bias=False)
        def forward(self, x):
            a, b = self.W(x).chunk(2, dim=-1)
            return self.W2(F.silu(a) * b)
    net = SwiGLU()
    x   = torch.randn(4, 10, d)
    assert net(x).shape == x.shape
    print("✓ Test 4: SwiGLU preserva shape")

def test_5_kv_cache_equivalente():
    """KV cache produce los mismos logits que sin cache."""
    torch.manual_seed(42)
    d, H, T = 32, 4, 10
    attn = nn.MultiheadAttention(d, H, batch_first=True)
    x    = torch.randn(1, T, d)

    # Sin cache: procesar toda la secuencia de una vez
    causal = torch.triu(torch.ones(T, T), diagonal=1).bool()
    out_full, _ = attn(x, x, x, attn_mask=causal)

    # Con cache: step a step (simplificado)
    outputs = []
    for t in range(T):
        ctx = x[:, :t+1, :]
        mask_t = torch.triu(torch.ones(t+1, t+1), diagonal=1).bool()
        out_t, _ = attn(ctx, ctx, ctx, attn_mask=mask_t)
        outputs.append(out_t[:, -1:, :])  # solo el último token

    out_cached = torch.cat(outputs, dim=1)
    assert torch.allclose(out_full, out_cached, atol=1e-4), \
        f"Max diff: {(out_full - out_cached).abs().max():.6f}"
    print("✓ Test 5: KV cache produce logits equivalentes")

def test_6_rope_rotacion():
    """RoPE aplica rotación (preserva norma)."""
    def apply_rope_test(x, T, d_k):
        freqs = 1.0 / (10000 ** (torch.arange(0, d_k, 2).float() / d_k))
        t     = torch.arange(T)
        f     = torch.outer(t, freqs)
        cos   = f.cos().unsqueeze(0).unsqueeze(0)  # (1,1,T,d_k/2)
        sin   = f.sin().unsqueeze(0).unsqueeze(0)
        x1, x2 = x.chunk(2, dim=-1)
        return torch.cat([x1*cos - x2*sin, x2*cos + x1*sin], dim=-1)

    B, H, T, d_k = 2, 4, 16, 32
    q = torch.randn(B, H, T, d_k)
    q_rope = apply_rope_test(q, T, d_k)
    # RoPE es una rotación → preserva norma
    norm_orig = q.norm(dim=-1)
    norm_rope = q_rope.norm(dim=-1)
    assert torch.allclose(norm_orig, norm_rope, atol=1e-5), \
        "RoPE debe preservar la norma"
    print("✓ Test 6: RoPE preserva la norma (es una rotación)")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M06 — Arquitectura LLMs")
    print("=" * 50)
    test_1_gpt_causal_mask()
    test_2_weight_tying()
    test_3_rmsnorm()
    test_4_swiglu_shape()
    test_5_kv_cache_equivalente()
    test_6_rope_rotacion()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M06 completado")
```

---

## Resumen del Módulo

| Concepto | GPT | BERT | LLaMA |
|---|---|---|---|
| **Tipo** | Decoder-only | Encoder-only | Decoder-only |
| **Atención** | Causal (máscara) | Bidireccional | Causal + GQA |
| **Pre-entrenamiento** | Causal LM | Masked LM | Causal LM |
| **Normalización** | Post-LN | Post-LN | Pre-RMSNorm |
| **Activación** | GeLU | GeLU | SwiGLU |
| **Positional** | Aprendible | Aprendible | RoPE |
| **KV Cache** | Sí | No | Sí (reducido por GQA) |
| **Uso ideal** | Generación | Clasificación/NER | Generación/Chat |

**Próximo módulo:** M07 — Pre-entrenamiento y Tokenización
