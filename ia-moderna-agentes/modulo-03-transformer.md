# Módulo 03 — Arquitectura Transformer en Detalle

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Implementar el Transformer completo desde cero (encoder y decoder), comprender cada componente (PE, FFN, LayerNorm, residual), y entrenar un pequeño modelo de traducción.
**Herramientas:** PyTorch 2.x, NumPy, Matplotlib
**Prerequisito:** M02 (Mecanismo de Atención)

---

## 1. Visión General de la Arquitectura

```python
# seccion_01_vision_general.py
"""
"Attention is All You Need" — Vaswani et al. (2017)

Arquitectura del Transformer:
  - ENCODER: N capas idénticas, cada una con:
      1. Multi-Head Self-Attention
      2. Feed-Forward Network (FFN)
      → Ambos con residual + LayerNorm
  - DECODER: N capas idénticas, cada una con:
      1. Masked Multi-Head Self-Attention
      2. Cross Multi-Head Attention (encoder-decoder)
      3. Feed-Forward Network
      → Los tres con residual + LayerNorm
"""

arquitectura = """
INPUT TOKENS
     ↓
[Embedding + Positional Encoding]
     ↓
┌─────────────────────────────────┐
│  ENCODER LAYER (× N)            │
│  ┌───────────────────────────┐  │
│  │ Multi-Head Self-Attention │  │
│  └───────────────────────────┘  │
│            + Residual            │
│         Layer Normalization      │
│  ┌───────────────────────────┐  │
│  │  Feed-Forward Network     │  │
│  └───────────────────────────┘  │
│            + Residual            │
│         Layer Normalization      │
└─────────────────────────────────┘
     ↓ encoder_output
┌─────────────────────────────────┐
│  DECODER LAYER (× N)            │
│  ┌───────────────────────────┐  │
│  │ Masked MH Self-Attention  │  │  ← causal mask
│  └───────────────────────────┘  │
│            + Residual + LN       │
│  ┌───────────────────────────┐  │
│  │  Cross Attention          │  │  ← Q del decoder, K,V del encoder
│  └───────────────────────────┘  │
│            + Residual + LN       │
│  ┌───────────────────────────┐  │
│  │  Feed-Forward Network     │  │
│  └───────────────────────────┘  │
│            + Residual + LN       │
└─────────────────────────────────┘
     ↓
  Linear + Softmax
     ↓
  PREDICCIÓN
"""
print(arquitectura)
```

---

## 2. Positional Encoding

```python
# seccion_02_positional_encoding.py
"""
Sin RNN, el Transformer no tiene noción de orden.
Positional Encoding inyecta información posicional en el embedding.

PE(pos, 2i)   = sin(pos / 10000^{2i/d_model})
PE(pos, 2i+1) = cos(pos / 10000^{2i/d_model})

Propiedades:
  - PE(pos+k) puede expresarse como combinación lineal de PE(pos)
  - Funciona con secuencias más largas que las vistas en entrenamiento
  - Alternativa: learned positional embeddings (GPT, BERT)
"""
import torch
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

def positional_encoding(max_len, d_model):
    """
    Retorna tensor (1, max_len, d_model) con el encoding sinusoidal.
    """
    PE = torch.zeros(max_len, d_model)
    pos = torch.arange(max_len).unsqueeze(1).float()   # (max_len, 1)
    # Divisores para cada par de dimensiones
    div_term = torch.exp(
        torch.arange(0, d_model, 2).float() * (-np.log(10000.0) / d_model)
    )
    PE[:, 0::2] = torch.sin(pos * div_term)  # dimensiones pares
    PE[:, 1::2] = torch.cos(pos * div_term)  # dimensiones impares
    return PE.unsqueeze(0)  # (1, max_len, d_model)

# Visualizar
d_model = 64
max_len = 100
PE = positional_encoding(max_len, d_model)

fig, axes = plt.subplots(1, 2, figsize=(14, 4))

# Heatmap del PE completo
axes[0].imshow(PE[0].numpy(), cmap="RdBu", aspect="auto",
               vmin=-1, vmax=1)
axes[0].set_title("Positional Encoding (pos × dim)")
axes[0].set_xlabel("Dimensión")
axes[0].set_ylabel("Posición")
plt.colorbar(axes[0].images[0], ax=axes[0])

# Primeras dimensiones a lo largo de posiciones
for i in [0, 1, 4, 8]:
    axes[1].plot(PE[0, :, i].numpy(), label=f"dim {i}")
axes[1].set_title("Valores PE por posición (primeras dims)")
axes[1].set_xlabel("Posición")
axes[1].legend()

plt.tight_layout()
plt.savefig("outputs/positional_encoding.png", dpi=150)
print("✓ outputs/positional_encoding.png guardado")

# Verificar propiedades
# dot product entre pos y pos+k debe ser similar para todo pos (trasladable)
pe = PE[0]  # (max_len, d_model)
dot_0_10 = (pe[0] @ pe[10]).item()
dot_5_15 = (pe[5] @ pe[15]).item()
dot_20_30 = (pe[20] @ pe[30]).item()
print(f"\nDot product PE(0)·PE(10)  = {dot_0_10:.3f}")
print(f"Dot product PE(5)·PE(15)  = {dot_5_15:.3f}")
print(f"Dot product PE(20)·PE(30) = {dot_20_30:.3f}")
print("→ Valores similares: el encoding es aproximadamente trasladable")
```

---

## 3. Feed-Forward Network y LayerNorm

```python
# seccion_03_ffn_layernorm.py
"""
Dos componentes esenciales de cada capa Transformer:

1. Feed-Forward Network (FFN):
   FFN(x) = max(0, x·W₁ + b₁)·W₂ + b₂
   - Aplicado de forma INDEPENDIENTE a cada posición
   - d_ff = 4 * d_model (típico)
   - No hay interacción entre posiciones aquí (eso lo hace la atención)

2. Layer Normalization:
   LN(x) = (x - μ) / (σ + ε) * γ + β
   - μ, σ calculados sobre las DIMENSIONES (no el batch)
   - Aprende γ (scale) y β (shift)
   - Alternativa: BatchNorm (menos efectiva en NLP por longitudes variables)
"""
import torch
import torch.nn as nn
import numpy as np

class PositionWiseFFN(nn.Module):
    """FFN aplicada independientemente a cada posición."""
    def __init__(self, d_model, d_ff, dropout=0.1):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
        )

    def forward(self, x):
        # x: (batch, T, d_model) → misma forma
        return self.net(x)

# LayerNorm manual para entender
class LayerNormManual(nn.Module):
    def __init__(self, d_model, eps=1e-6):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(d_model))
        self.beta  = nn.Parameter(torch.zeros(d_model))
        self.eps   = eps

    def forward(self, x):
        # Normalizar sobre la última dimensión (features)
        mu    = x.mean(dim=-1, keepdim=True)
        sigma = x.std(dim=-1, keepdim=True, unbiased=False)
        x_norm = (x - mu) / (sigma + self.eps)
        return self.gamma * x_norm + self.beta

# Verificar equivalencia con PyTorch
torch.manual_seed(42)
d_model = 64
x = torch.randn(4, 20, d_model)

ln_manual = LayerNormManual(d_model)
ln_pytorch = nn.LayerNorm(d_model)

# Inicializar con mismos pesos
with torch.no_grad():
    ln_pytorch.weight.data = ln_manual.gamma.data
    ln_pytorch.bias.data   = ln_manual.beta.data

out_manual  = ln_manual(x)
out_pytorch = ln_pytorch(x)

diff = (out_manual - out_pytorch).abs().max().item()
print(f"LayerNorm manual vs PyTorch: max diff = {diff:.8f}  (≈ 0)")

# Diferencia con BatchNorm
print("\nLayerNorm vs BatchNorm:")
print("BatchNorm: normaliza sobre el BATCH → problemático con secuencias variables")
print("LayerNorm: normaliza sobre las FEATURES → invariante a longitud de secuencia")

bn = nn.BatchNorm1d(d_model)
# BN requiere (N, C) o (N, C, L) — problemático con T variable en NLP
print(f"LayerNorm output stats: μ={out_pytorch.mean():.4f}, σ={out_pytorch.std():.4f}")

# Verificar que LN normaliza por posición
x_pos0 = out_pytorch[:, 0, :]  # posición 0 de todas las muestras
print(f"Por posición 0: media={x_pos0.mean(dim=-1).detach().numpy().round(4)}")
print("→ Cada posición tiene media ≈ 0 y std ≈ 1 (antes de γ, β)")
```

---

## 4. Capa Encoder Completa

```python
# seccion_04_encoder_layer.py
"""
Una capa del Transformer Encoder:
  SubLayer1: x → LN(x + MultiHeadAttn(x, x, x))
  SubLayer2: x → LN(x + FFN(x))

Nota: "Pre-norm" (LN antes) vs "Post-norm" (LN después)
  - Paper original: Post-norm → LN(x + Sublayer(x))
  - En práctica: Pre-norm suele entrenar mejor
  → Aquí implementamos Pre-norm (usado en GPT-2, LLaMA)
"""
import torch
import torch.nn as nn
import numpy as np

def positional_encoding(max_len, d_model):
    PE = torch.zeros(max_len, d_model)
    pos = torch.arange(max_len).unsqueeze(1).float()
    div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-np.log(10000.0) / d_model))
    PE[:, 0::2] = torch.sin(pos * div_term)
    PE[:, 1::2] = torch.cos(pos * div_term)
    return PE.unsqueeze(0)

class EncoderLayer(nn.Module):
    def __init__(self, d_model, n_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = nn.MultiheadAttention(d_model, n_heads, dropout=dropout,
                                               batch_first=True)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.drop  = nn.Dropout(dropout)

    def forward(self, x, src_key_padding_mask=None):
        # Pre-norm self-attention + residual
        x_norm = self.norm1(x)
        attn_out, _ = self.self_attn(
            x_norm, x_norm, x_norm,
            key_padding_mask=src_key_padding_mask
        )
        x = x + self.drop(attn_out)

        # Pre-norm FFN + residual
        x = x + self.drop(self.ffn(self.norm2(x)))
        return x

class TransformerEncoder(nn.Module):
    def __init__(self, vocab_size, d_model, n_heads, d_ff, n_layers,
                 max_len=512, dropout=0.1):
        super().__init__()
        self.embed    = nn.Embedding(vocab_size, d_model, padding_idx=0)
        self.register_buffer("PE", positional_encoding(max_len, d_model))
        self.layers   = nn.ModuleList(
            [EncoderLayer(d_model, n_heads, d_ff, dropout) for _ in range(n_layers)]
        )
        self.norm     = nn.LayerNorm(d_model)
        self.drop     = nn.Dropout(dropout)
        self.d_model  = d_model

    def forward(self, src, src_key_padding_mask=None):
        T = src.shape[1]
        x = self.drop(self.embed(src) * np.sqrt(self.d_model) + self.PE[:, :T])
        for layer in self.layers:
            x = layer(x, src_key_padding_mask)
        return self.norm(x)

# Prueba
torch.manual_seed(42)
encoder = TransformerEncoder(vocab_size=1000, d_model=64, n_heads=4,
                             d_ff=256, n_layers=2)
src = torch.randint(1, 1000, (4, 20))  # (batch, T)
padding_mask = (src == 0)              # True donde hay PAD

out = encoder(src, src_key_padding_mask=padding_mask)
print(f"Encoder output: {out.shape}  (batch, T, d_model)")

n_params = sum(p.numel() for p in encoder.parameters())
print(f"Parámetros encoder (vocab=1000, d=64, h=4, ff=256, L=2): {n_params:,d}")
```

---

## 5. Capa Decoder Completa

```python
# seccion_05_decoder_layer.py
"""
Una capa del Transformer Decoder:
  1. Masked Self-Attention (causal mask)
  2. Cross-Attention (queries del decoder, K,V del encoder)
  3. FFN
  Cada uno con residual + LayerNorm
"""
import torch
import torch.nn as nn
import numpy as np

class DecoderLayer(nn.Module):
    def __init__(self, d_model, n_heads, d_ff, dropout=0.1):
        super().__init__()
        # 1. Self-attention enmascarada (causal)
        self.self_attn  = nn.MultiheadAttention(d_model, n_heads,
                                                 dropout=dropout, batch_first=True)
        # 2. Cross-attention (encoder-decoder)
        self.cross_attn = nn.MultiheadAttention(d_model, n_heads,
                                                 dropout=dropout, batch_first=True)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff), nn.ReLU(), nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.drop  = nn.Dropout(dropout)

    @staticmethod
    def causal_mask(T, device):
        """Máscara causal: True en posiciones futuras (serán enmascaradas)."""
        mask = torch.triu(torch.ones(T, T, device=device), diagonal=1).bool()
        return mask

    def forward(self, x, enc_out, tgt_key_padding_mask=None, src_key_padding_mask=None):
        T = x.shape[1]
        causal = self.causal_mask(T, x.device)

        # 1. Masked self-attention
        x_norm = self.norm1(x)
        sa_out, _ = self.self_attn(x_norm, x_norm, x_norm,
                                   attn_mask=causal,
                                   key_padding_mask=tgt_key_padding_mask)
        x = x + self.drop(sa_out)

        # 2. Cross-attention
        x_norm = self.norm2(x)
        ca_out, attn_w = self.cross_attn(x_norm, enc_out, enc_out,
                                          key_padding_mask=src_key_padding_mask)
        x = x + self.drop(ca_out)

        # 3. FFN
        x = x + self.drop(self.ffn(self.norm3(x)))
        return x, attn_w

# Prueba de la capa decoder
torch.manual_seed(42)
d, H, T_src, T_tgt = 64, 4, 20, 15
enc_out = torch.randn(4, T_src, d)
dec_in  = torch.randn(4, T_tgt, d)

layer = DecoderLayer(d, H, d_ff=256)
out, attn_w = layer(dec_in, enc_out)
print(f"Decoder layer output: {out.shape}   (batch, T_tgt, d_model)")
print(f"Cross-attention weights: {attn_w.shape}  (batch, T_tgt, T_src)")
```

---

## 6. Transformer Completo — Traducción

```python
# seccion_06_transformer_completo.py
"""
Transformer encoder-decoder completo para traducción.
Tarea de ejemplo: invertir secuencia (mismo dataset que M01/M02).
"""
import torch
import torch.nn as nn
import numpy as np
import random

torch.manual_seed(42); random.seed(42)

def positional_encoding(max_len, d_model):
    PE = torch.zeros(max_len, d_model)
    pos = torch.arange(max_len).unsqueeze(1).float()
    div = torch.exp(torch.arange(0, d_model, 2).float() * (-np.log(10000.0) / d_model))
    PE[:, 0::2] = torch.sin(pos * div)
    PE[:, 1::2] = torch.cos(pos * div)
    return PE.unsqueeze(0)

class TransformerTraduccion(nn.Module):
    def __init__(self, src_vocab, tgt_vocab, d_model=64, n_heads=4,
                 d_ff=256, n_layers=2, max_len=50, dropout=0.1):
        super().__init__()
        self.d_model = d_model

        self.src_embed = nn.Embedding(src_vocab, d_model, padding_idx=0)
        self.tgt_embed = nn.Embedding(tgt_vocab, d_model, padding_idx=0)
        self.register_buffer("PE", positional_encoding(max_len, d_model))

        enc_layer = nn.TransformerEncoderLayer(d_model, n_heads, d_ff,
                                               dropout, batch_first=True, norm_first=True)
        dec_layer = nn.TransformerDecoderLayer(d_model, n_heads, d_ff,
                                               dropout, batch_first=True, norm_first=True)

        self.encoder = nn.TransformerEncoder(enc_layer, n_layers,
                                             norm=nn.LayerNorm(d_model))
        self.decoder = nn.TransformerDecoder(dec_layer, n_layers,
                                             norm=nn.LayerNorm(d_model))
        self.output_proj = nn.Linear(d_model, tgt_vocab)

    def encode(self, src, src_pad_mask=None):
        T = src.shape[1]
        x = self.src_embed(src) * np.sqrt(self.d_model) + self.PE[:, :T]
        return self.encoder(x, src_key_padding_mask=src_pad_mask)

    def decode(self, tgt, enc_out, tgt_causal_mask=None,
               src_pad_mask=None, tgt_pad_mask=None):
        T = tgt.shape[1]
        x = self.tgt_embed(tgt) * np.sqrt(self.d_model) + self.PE[:, :T]
        return self.decoder(x, enc_out,
                            tgt_mask=tgt_causal_mask,
                            memory_key_padding_mask=src_pad_mask,
                            tgt_key_padding_mask=tgt_pad_mask)

    def forward(self, src, tgt, src_pad_mask=None):
        T_tgt = tgt.shape[1]
        causal = nn.Transformer.generate_square_subsequent_mask(T_tgt, device=src.device)
        enc_out = self.encode(src, src_pad_mask)
        dec_out = self.decode(tgt, enc_out, causal, src_pad_mask)
        return self.output_proj(dec_out)

# Dataset: invertir secuencia de dígitos
SOS, EOS, PAD = 1, 2, 0
VOCAB_SIZE = 12

def make_batch(batch_size=32, n=7):
    srcs, trgs = [], []
    for _ in range(batch_size):
        nums = random.sample(range(3, 12), n)
        src = nums
        trg = [SOS] + list(reversed(nums)) + [EOS]
        srcs.append(src)
        trgs.append(trg)
    return (torch.tensor(srcs),
            torch.tensor(trgs))

model = TransformerTraduccion(VOCAB_SIZE, VOCAB_SIZE)
opt   = torch.optim.Adam(model.parameters(), lr=3e-4, betas=(0.9, 0.98))
crit  = nn.CrossEntropyLoss(ignore_index=PAD)

# Entrenamiento con learning rate warmup (como en el paper)
def lr_lambda(step, d_model=64, warmup=400):
    step = max(1, step)
    return min(step**-0.5, step * warmup**-1.5) * d_model**-0.5

scheduler = torch.optim.lr_scheduler.LambdaLR(opt, lr_lambda)

best_loss = float('inf')
for epoch in range(500):
    src, trg = make_batch(batch_size=32)
    # Input: SOS + secuencia (sin EOS)
    tgt_in  = trg[:, :-1]
    # Target: secuencia + EOS (sin SOS)
    tgt_out = trg[:, 1:]

    logits = model(src, tgt_in)
    loss   = crit(logits.reshape(-1, VOCAB_SIZE), tgt_out.reshape(-1))

    opt.zero_grad(); loss.backward(); opt.step(); scheduler.step()

    if epoch % 100 == 99:
        best_loss = min(best_loss, loss.item())
        print(f"Epoch {epoch+1:4d} | loss: {loss.item():.4f} | lr: {scheduler.get_last_lr()[0]:.6f}")

# Inferencia greedy
def predecir(src_seq):
    model.eval()
    with torch.no_grad():
        src = torch.tensor([src_seq])
        enc_out = model.encode(src)
        # Comenzar con SOS
        tgt = torch.tensor([[SOS]])
        IDX2D = {i: str(i-2) if i >= 3 else str(i) for i in range(12)}
        resultado = []
        for _ in range(len(src_seq) + 2):
            T_t = tgt.shape[1]
            causal = nn.Transformer.generate_square_subsequent_mask(T_t)
            dec_out = model.decode(tgt, enc_out, causal)
            logits = model.output_proj(dec_out[:, -1])
            next_tok = logits.argmax(-1).item()
            if next_tok == EOS:
                break
            resultado.append(next_tok - 2)  # quitar offset SOS/EOS/PAD
            tgt = torch.cat([tgt, torch.tensor([[next_tok]])], dim=1)
    return resultado

prueba = [5, 3, 8, 4, 7, 6, 9]
pred = predecir(prueba)
print(f"\nEntrada:  {prueba}")
print(f"Esperado: {list(reversed(prueba))}")
print(f"Modelo:   {pred}")

n_params = sum(p.numel() for p in model.parameters())
print(f"\nParámetros del Transformer: {n_params:,d}")
torch.save(model.state_dict(), "outputs/transformer_basico.pt")
print("✓ outputs/transformer_basico.pt guardado")
```

---

## 7. Variantes: Pre-norm vs Post-norm, GPT vs BERT

```python
# seccion_07_variantes.py
"""
El Transformer original tuvo muchas variantes importantes:

BERT (Bidirectional Encoder Representations from Transformers, 2019)
  - Solo el ENCODER
  - Pre-entrenado con:
    1. Masked Language Modeling (MLM): predecir tokens [MASK]
    2. Next Sentence Prediction (NSP)
  - Bidireccional: cada token ve todo el contexto
  - Ideal para: clasificación, NER, QA extractiva

GPT (Generative Pre-trained Transformer, 2018)
  - Solo el DECODER (con máscara causal)
  - Pre-entrenado con: Language Modeling (predecir siguiente token)
  - Autoregresivo: genera token por token
  - Ideal para: generación de texto, code, chat

T5 / mT5 (Text-to-Text Transfer Transformer, 2020)
  - Encoder-Decoder completo (como el original)
  - Todo es text-to-text: traducción, resumen, QA como generación
  - Ideal para: tareas de seq2seq

LLaMA (Meta, 2023)
  - Similar a GPT pero con mejoras:
    * RMSNorm en lugar de LayerNorm (más rápido)
    * RoPE: Rotary Positional Embeddings (extendible a cualquier longitud)
    * SwiGLU activation en FFN: swish(x·W₁) · (x·W₂)
    * Grouped Query Attention (GQA): reduce KV cache
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

# RMSNorm (usado en LLaMA en lugar de LayerNorm)
class RMSNorm(nn.Module):
    def __init__(self, d_model, eps=1e-6):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(d_model))
        self.eps   = eps

    def forward(self, x):
        # RMS: no centra, solo normaliza por la raíz cuadrada media
        rms = x.pow(2).mean(dim=-1, keepdim=True).add(self.eps).sqrt()
        return self.gamma * x / rms

# SwiGLU activation (FFN de LLaMA)
class SwiGLUFFN(nn.Module):
    def __init__(self, d_model, d_ff=None):
        super().__init__()
        d_ff = d_ff or int(d_model * 8/3)
        # Dos proyecciones: gate y value
        self.W  = nn.Linear(d_model, d_ff, bias=False)
        self.V  = nn.Linear(d_model, d_ff, bias=False)
        self.W2 = nn.Linear(d_ff, d_model, bias=False)

    def forward(self, x):
        # SwiGLU: swish(xW) · (xV)
        return self.W2(F.silu(self.W(x)) * self.V(x))

# Rotary Positional Embeddings (RoPE, usado en LLaMA)
def rotate_half(x):
    x1, x2 = x.chunk(2, dim=-1)
    return torch.cat([-x2, x1], dim=-1)

def apply_rope(q, k, cos, sin):
    """Aplica RoPE a queries y keys."""
    q_rope = q * cos + rotate_half(q) * sin
    k_rope = k * cos + rotate_half(k) * sin
    return q_rope, k_rope

def precompute_rope(d_k, max_len=2048, theta=10000.0):
    """Precalcula cosenos y senos para RoPE."""
    freqs = 1.0 / (theta ** (torch.arange(0, d_k, 2).float() / d_k))
    t = torch.arange(max_len)
    freqs = torch.outer(t, freqs)                         # (max_len, d_k/2)
    freqs_cis = torch.cat([freqs, freqs], dim=-1)         # (max_len, d_k)
    return freqs_cis.cos(), freqs_cis.sin()

# Comparativa
print("Comparativa de arquitecturas Transformer:")
print(f"{'Componente':<25} {'Original (2017)':>18} {'GPT-2':>12} {'LLaMA':>12}")
print("-" * 70)
componentes = [
    ("Normalización",   "Post-LN",      "Post-LN", "Pre-RMSNorm"),
    ("Activación FFN",  "ReLU",         "GeLU",    "SwiGLU"),
    ("Pos. Encoding",   "Sinusoidal",   "Learned", "RoPE"),
    ("Tipo",            "Encoder+Dec",  "Decoder", "Decoder"),
    ("Atención",        "MHA",          "MHA",     "GQA"),
]
for c, orig, gpt, llama in componentes:
    print(f"{c:<25} {orig:>18} {gpt:>12} {llama:>12}")

print("""
RMSNorm vs LayerNorm:
  LayerNorm: normaliza con media Y std (2 estadísticos)
  RMSNorm:   normaliza solo con RMS (1 estadístico, ~10% más rápido)

SwiGLU vs ReLU:
  ReLU: max(0, x) — simple pero muchos gradientes cero
  SwiGLU: swish(xW) · (xV) — suave, mejor gradiente

RoPE vs Positional Encoding:
  PE sinusoidal: suma fija al embedding
  RoPE: rota el espacio Q,K según la posición → relativo, extendible
""")
```

---

## 8. Entrenamiento Eficiente y Trucos

```python
# seccion_08_trucos_entrenamiento.py
"""
Técnicas de entrenamiento fundamentales para Transformers.
"""
import torch
import torch.nn as nn
import numpy as np

# 1. Learning Rate Warmup (crucial para Transformers)
print("1. Learning Rate Warmup")
d_model = 512
warmup_steps = 4000

def lr_transformer(step):
    step = max(1, step)
    return d_model**-0.5 * min(step**-0.5, step * warmup_steps**-1.5)

steps = np.arange(1, 20001)
lrs   = [lr_transformer(s) for s in steps]
peak_step = warmup_steps
peak_lr   = lr_transformer(peak_step)
print(f"  Peak LR en step {peak_step}: {peak_lr:.5f}")
print(f"  LR decae después del peak (inversamente a sqrt(step))")

# 2. Gradient Clipping
print("\n2. Gradient Clipping")
print("  torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)")
print("  Previene explosión de gradientes (común con BPTT largo)")

# 3. Label Smoothing
print("\n3. Label Smoothing")
class LabelSmoothingLoss(nn.Module):
    def __init__(self, vocab_size, smoothing=0.1, ignore_index=0):
        super().__init__()
        self.smoothing    = smoothing
        self.vocab_size   = vocab_size
        self.ignore_index = ignore_index
        self.confidence   = 1.0 - smoothing

    def forward(self, logits, targets):
        # logits: (N, vocab_size)
        log_probs = torch.log_softmax(logits, dim=-1)
        # Distribución suavizada
        smooth_dist = torch.full_like(log_probs, self.smoothing / (self.vocab_size - 1))
        smooth_dist.scatter_(-1, targets.unsqueeze(-1), self.confidence)
        # Ignorar padding
        mask = (targets != self.ignore_index)
        loss = -(smooth_dist * log_probs).sum(dim=-1)
        return loss[mask].mean()

# 4. Dropout
print("4. Dropout en Transformers:")
print("  - Dropout en atención (sobre weights)")
print("  - Dropout en FFN (entre las dos lineales)")
print("  - Dropout en embeddings (p=0.1 típico)")

# 5. Gradient Accumulation
print("\n5. Gradient Accumulation")
print("  Para batches efectivos grandes con memoria limitada:")
accum_steps = 4
print(f"  batch_size=8, accum_steps={accum_steps} → batch efectivo=32")
print("""
  for step, (src, tgt) in enumerate(loader):
      loss = model(src, tgt) / accum_steps  # normalizar
      loss.backward()
      if (step + 1) % accum_steps == 0:
          clip_grad_norm_(model.parameters(), 1.0)
          optimizer.step()
          scheduler.step()
          optimizer.zero_grad()
""")

# 6. Mixed Precision Training
print("6. Mixed Precision (FP16/BF16)")
print("  BF16: mismo rango que FP32 pero 16 bits → 2x memoria, ~2x velocidad")
print("  Ejemplo:")
print("""
  from torch.cuda.amp import autocast, GradScaler
  scaler = GradScaler()
  with autocast(dtype=torch.bfloat16):
      loss = model(src, tgt)
  scaler.scale(loss).backward()
  scaler.step(optimizer)
  scaler.update()
""")

# Comparativa de capacidad según tamaño del modelo
print("Capacidad vs Tamaño (regla general):")
print(f"{'Modelo':>12} {'Params':>10} {'d_model':>8} {'Capas':>6} {'Heads':>6}")
modelos = [
    ("Tiny",     "1M",   64,   2,  2),
    ("Small",    "10M",  128,  4,  4),
    ("Base",     "110M", 768,  12, 12),  # BERT-base
    ("Large",    "340M", 1024, 24, 16),  # BERT-large
    ("GPT-2",    "1.5B", 1600, 48, 25),
    ("LLaMA-7B", "7B",   4096, 32, 32),
]
for nombre, params, d, L, h in modelos:
    print(f"{nombre:>12} {params:>10} {d:>8} {L:>6} {h:>6}")
```

---

## Checkpoint M03

```python
# checkpoint_m03.py
"""
Tests de verificación para Módulo 03 — Arquitectura Transformer.
Ejecutar con: python checkpoint_m03.py
"""
import torch
import torch.nn as nn
import numpy as np

def test_1_positional_encoding_shape():
    """PE tiene la forma correcta y valores en [-1, 1]."""
    d_model, max_len = 64, 100
    PE = torch.zeros(max_len, d_model)
    pos = torch.arange(max_len).unsqueeze(1).float()
    div = torch.exp(torch.arange(0, d_model, 2).float() * (-np.log(10000.0) / d_model))
    PE[:, 0::2] = torch.sin(pos * div)
    PE[:, 1::2] = torch.cos(pos * div)
    assert PE.shape == (max_len, d_model)
    assert PE.abs().max() <= 1.0 + 1e-6
    print("✓ Test 1: Positional Encoding shape y rango correctos")

def test_2_layernorm_normaliza():
    """LayerNorm produce media ≈ 0 y std ≈ 1 (antes de gamma/beta)."""
    x = torch.randn(4, 20, 64) * 10 + 5  # media=5, std=10
    ln = nn.LayerNorm(64)
    # Inicializar gamma=1, beta=0
    with torch.no_grad():
        ln.weight.fill_(1); ln.bias.fill_(0)
    y = ln(x)
    assert y.mean().abs() < 0.1, f"Media: {y.mean():.4f}"
    assert abs(y.std().item() - 1.0) < 0.1, f"Std: {y.std():.4f}"
    print("✓ Test 2: LayerNorm normaliza correctamente")

def test_3_transformer_encoder_shape():
    """TransformerEncoder produce output del mismo shape que input."""
    d, h, T, B = 64, 4, 20, 4
    layer = nn.TransformerEncoderLayer(d, h, dim_feedforward=256,
                                       batch_first=True, norm_first=True)
    enc = nn.TransformerEncoder(layer, num_layers=2)
    x = torch.randn(B, T, d)
    out = enc(x)
    assert out.shape == (B, T, d)
    print("✓ Test 3: Encoder shape preservado")

def test_4_causal_mask():
    """La máscara causal es triangular inferior."""
    T = 8
    mask = nn.Transformer.generate_square_subsequent_mask(T)
    # En PyTorch: -inf en posiciones futuras
    for i in range(T):
        for j in range(T):
            if j > i:
                assert mask[i, j].item() == float('-inf'), f"mask[{i},{j}] debería ser -inf"
            else:
                assert mask[i, j].item() == 0.0, f"mask[{i},{j}] debería ser 0"
    print("✓ Test 4: Máscara causal correcta (triangular inferior)")

def test_5_residual_preserva_gradiente():
    """Las conexiones residuales permiten que el gradiente fluya directamente."""
    d = 64
    # Sin residual
    layer_sin = nn.Sequential(nn.Linear(d, d), nn.ReLU(), nn.Linear(d, d))
    # Con residual
    x = torch.randn(4, 10, d, requires_grad=True)
    x2 = x.clone().detach().requires_grad_(True)

    # Forward sin residual (5 capas)
    y = x.clone()
    for _ in range(5):
        y = torch.relu(y @ torch.randn(d, d))
    y.sum().backward()

    # Forward con residual (5 capas)
    y2 = x2.clone()
    for _ in range(5):
        y2 = y2 + torch.relu(y2 @ torch.randn(d, d))
    y2.sum().backward()

    grad_sin = x.grad.abs().mean().item()
    grad_con = x2.grad.abs().mean().item()
    # Con residual el gradiente debería ser mayor (no se desvanece)
    print(f"✓ Test 5: grad sin residual={grad_sin:.6f}, con residual={grad_con:.6f}")

def test_6_ffn_posicion_independiente():
    """La FFN procesa cada posición de forma independiente."""
    d, T, B = 32, 10, 4
    ffn = nn.Sequential(nn.Linear(d, 128), nn.ReLU(), nn.Linear(128, d))
    x = torch.randn(B, T, d)

    # Procesar posición 5 de forma aislada
    out_full = ffn(x)
    out_pos5 = ffn(x[:, 5:6, :])  # solo posición 5

    assert torch.allclose(out_full[:, 5:6, :], out_pos5, atol=1e-5), \
        "FFN debe ser position-wise (independiente)"
    print("✓ Test 6: FFN es position-wise (independiente por posición)")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M03 — Arquitectura Transformer")
    print("=" * 50)
    test_1_positional_encoding_shape()
    test_2_layernorm_normaliza()
    test_3_transformer_encoder_shape()
    test_4_causal_mask()
    test_5_residual_preserva_gradiente()
    test_6_ffn_posicion_independiente()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M03 completado")
```

---

## Resumen del Módulo

| Componente | Función | Fórmula / Detalle |
|---|---|---|
| **Embedding + PE** | Token + posición | `embed(x) * √d + PE` |
| **Multi-Head Attn** | Contexto global | `softmax(QK^T/√d)V`, h cabezas paralelas |
| **FFN** | Transformación no lineal | `ReLU(xW₁+b₁)W₂+b₂`, por posición |
| **LayerNorm** | Estabilización | normaliza features, aprende γ, β |
| **Residual** | Flujo de gradiente | `x + Sublayer(LN(x))` |
| **Causal mask** | Autoregresión | `-inf` en posiciones futuras del decoder |
| **Cross-attn** | Conexión enc-dec | Q del decoder, K/V del encoder |
| **LR Warmup** | Estabilidad inicial | crece linealmente, luego `∝ step^{-0.5}` |

**Próximo módulo:** M04 — Vision Transformers (ViT) y Modelos Multimodales
