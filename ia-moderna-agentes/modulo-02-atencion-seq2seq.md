# Módulo 02 — Mecanismo de Atención: de seq2seq a Transformer

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender el mecanismo de atención (Bahdanau y Luong), implementar atención de producto punto, multi-head attention, y entender el camino desde seq2seq hacia la arquitectura Transformer.
**Herramientas:** PyTorch 2.x, NumPy, Matplotlib
**Prerequisito:** M01 (RNN/LSTM/GRU), álgebra lineal (matrices, producto punto)

---

## 1. Motivación: el Cuello de Botella de seq2seq

```python
# seccion_01_motivacion_atencion.py
"""
El problema del bottleneck en encoder-decoder sin atención.
Para secuencias largas, un solo vector de contexto no alcanza.
"""
import torch
import torch.nn as nn

# En seq2seq clásico sin atención:
# - El encoder lee T tokens y comprime TODO en h_T (vector fijo)
# - El decoder genera toda la salida solo desde ese vector
# - Para T=100 tokens, se pierde información de los primeros tokens

# Intuición de Bahdanau (2015): "¿Por qué no dejar que el decoder
# 'mire' todos los estados del encoder, ponderados según relevancia?"

# Analogía: cuando traducimos "The cat sat on the mat",
# para generar "el", miramos principalmente "The"
# para generar "gato", miramos principalmente "cat"
# → atención selectiva y dinámica

print("Sin atención:")
print("  Encoder: [h_1, h_2, ..., h_T] → h_T (cuello de botella)")
print("  Decoder: genera todo desde h_T\n")

print("Con atención (Bahdanau):")
print("  Encoder: produce [h_1, h_2, ..., h_T]  (todos los estados)")
print("  Decoder: en cada paso t, calcula pesos α_{t,i} para cada h_i")
print("  Contexto: c_t = Σ_i α_{t,i} · h_i  (suma ponderada dinámica)")
print("  Genera: ŷ_t = f(s_{t-1}, c_t, ŷ_{t-1})")
print()
print("Ventajas:")
print("  ✓ No hay cuello de botella: el decoder accede a TODOS los estados")
print("  ✓ Los pesos α permiten interpretabilidad (qué miraba el modelo)")
print("  ✓ Maneja secuencias largas mucho mejor")
```

---

## 2. Atención de Bahdanau (Aditiva)

```python
# seccion_02_bahdanau.py
"""
Bahdanau attention (2015): "Neural Machine Translation by Jointly
Learning to Align and Translate"

score(s_{t-1}, h_i) = v^T · tanh(W_s · s_{t-1} + W_h · h_i)
α_{t,i} = softmax(score)
c_t = Σ_i α_{t,i} · h_i
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

class BahdanauAttention(nn.Module):
    """
    Atención aditiva de Bahdanau.
    s: estado del decoder  (batch, dec_hidden)
    H: estados del encoder (batch, T_src, enc_hidden)
    """
    def __init__(self, enc_hidden, dec_hidden, attn_dim=64):
        super().__init__()
        self.W_s = nn.Linear(dec_hidden, attn_dim, bias=False)
        self.W_h = nn.Linear(enc_hidden, attn_dim, bias=False)
        self.v   = nn.Linear(attn_dim, 1, bias=False)

    def forward(self, s, H):
        # s: (batch, dec_hidden) → (batch, 1, attn_dim)
        # H: (batch, T, enc_hidden) → (batch, T, attn_dim)
        energy = self.v(torch.tanh(
            self.W_s(s).unsqueeze(1) + self.W_h(H)
        ))  # (batch, T, 1)

        weights = F.softmax(energy, dim=1)     # α: (batch, T, 1)
        context = (weights * H).sum(dim=1)     # c: (batch, enc_hidden)
        return context, weights.squeeze(2)     # (batch, enc_hidden), (batch, T)

# Ejemplo de uso
torch.manual_seed(42)
batch, T_src, enc_h, dec_h = 4, 10, 32, 32

attn = BahdanauAttention(enc_h, dec_h, attn_dim=16)
H = torch.randn(batch, T_src, enc_h)  # estados encoder
s = torch.randn(batch, dec_h)         # estado decoder

context, weights = attn(s, H)
print(f"H (encoder states): {H.shape}")
print(f"s (decoder state):  {s.shape}")
print(f"context c_t:        {context.shape}")
print(f"weights α:          {weights.shape}")
print(f"Suma de pesos (debe ser 1): {weights.sum(dim=1)}")
```

---

## 3. Atención de Luong (Producto Punto Escalado)

```python
# seccion_03_luong_scaled.py
"""
Luong attention (2015): formas general, bilineal y producto punto.
El producto punto escalado es la base del Transformer.

score(q, k) = q · k / sqrt(d_k)

Dividir por sqrt(d_k) estabiliza los gradientes cuando d_k es grande:
- Si d_k=512, el producto puede ser ~22 → softmax muy "peaked" → gradientes ~0
- Dividir por sqrt(512)≈22.6 normaliza el rango
"""
import torch
import torch.nn.functional as F
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

def atencion_producto_punto_escalado(Q, K, V, mask=None):
    """
    Q: queries  (batch, heads, T_q, d_k)
    K: keys     (batch, heads, T_k, d_k)
    V: values   (batch, heads, T_k, d_v)
    mask: opcional (batch, 1, 1, T_k) → -inf donde hay padding
    """
    d_k = Q.shape[-1]
    # Scores: (batch, heads, T_q, T_k)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / np.sqrt(d_k)

    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))

    weights = F.softmax(scores, dim=-1)  # α
    output  = torch.matmul(weights, V)   # (batch, heads, T_q, d_v)
    return output, weights

# Demostración: por qué dividir por sqrt(d_k)
print("Efecto de la escala en softmax:")
for d_k in [1, 16, 64, 512]:
    q = torch.randn(1, d_k)
    k = torch.randn(10, d_k)   # 10 posibles claves
    dots = (q @ k.T).squeeze()
    # Sin escala
    w_sin = F.softmax(dots, dim=0)
    # Con escala
    w_con = F.softmax(dots / np.sqrt(d_k), dim=0)
    print(f"  d_k={d_k:4d}: max(w) sin={w_sin.max():.3f}  con={w_con.max():.3f}  "
          f"entropia sin={-(w_sin*w_sin.log()).sum():.3f}  "
          f"con={-(w_con*w_con.log()).sum():.3f}")

print("""
→ Sin escala, con d_k grande: softmax colapsa en 1 token (baja entropía)
→ Con escala: distribución más suave (alta entropía) → gradientes mejores
""")

# Visualizar atención en una frase de ejemplo
np.random.seed(42)
torch.manual_seed(42)

palabras_src = ["the", "cat", "sat", "on", "mat"]
palabras_tgt = ["el", "gato", "se", "sentó", "en", "la", "estera"]
T_s, T_t = len(palabras_src), len(palabras_tgt)
d_k = 8

Q = torch.randn(1, 1, T_t, d_k)  # queries del decoder
K = torch.randn(1, 1, T_s, d_k)  # keys del encoder
V = torch.randn(1, 1, T_s, d_k)

_, weights = atencion_producto_punto_escalado(Q, K, V)
W = weights[0, 0].detach().numpy()  # (T_t, T_s)

fig, ax = plt.subplots(figsize=(7, 5))
im = ax.imshow(W, cmap="Blues", aspect="auto")
ax.set_xticks(range(T_s)); ax.set_xticklabels(palabras_src)
ax.set_yticks(range(T_t)); ax.set_yticklabels(palabras_tgt)
ax.set_xlabel("Encoder (fuente)")
ax.set_ylabel("Decoder (objetivo)")
ax.set_title("Pesos de atención (producto punto escalado)")
plt.colorbar(im)
plt.tight_layout()
plt.savefig("outputs/atencion_visualizada.png", dpi=150)
print("✓ outputs/atencion_visualizada.png guardado")
```

---

## 4. Multi-Head Attention

```python
# seccion_04_multihead.py
"""
Multi-Head Attention (Vaswani et al. 2017):
En lugar de una sola atención con d_model dimensiones,
proyectamos Q, K, V a h subespacios (heads) de d_k = d_model/h.

Cada head puede "especializar" en diferentes tipos de relaciones:
- Head 1: relaciones sintácticas (sujeto-verbo)
- Head 2: relaciones semánticas (co-referencia)
- Head 3: posición relativa, etc.
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, n_heads):
        super().__init__()
        assert d_model % n_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads

        # Proyecciones lineales (una para cada Q, K, V y salida)
        self.W_Q = nn.Linear(d_model, d_model, bias=False)
        self.W_K = nn.Linear(d_model, d_model, bias=False)
        self.W_V = nn.Linear(d_model, d_model, bias=False)
        self.W_O = nn.Linear(d_model, d_model, bias=False)

    def split_heads(self, x):
        """(batch, T, d_model) → (batch, n_heads, T, d_k)"""
        batch, T, _ = x.shape
        x = x.view(batch, T, self.n_heads, self.d_k)
        return x.transpose(1, 2)  # (batch, heads, T, d_k)

    def forward(self, query, key, value, mask=None):
        Q = self.split_heads(self.W_Q(query))
        K = self.split_heads(self.W_K(key))
        V = self.split_heads(self.W_V(value))

        # Atención escalada por head
        scores = torch.matmul(Q, K.transpose(-2, -1)) / np.sqrt(self.d_k)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        weights = F.softmax(scores, dim=-1)
        attn_out = torch.matmul(weights, V)  # (batch, heads, T_q, d_k)

        # Concatenar heads: (batch, T_q, d_model)
        batch, heads, T_q, d_k = attn_out.shape
        attn_out = attn_out.transpose(1, 2).contiguous().view(batch, T_q, heads * d_k)

        return self.W_O(attn_out), weights

# Demostración
torch.manual_seed(42)
d_model, n_heads = 64, 8
batch, T = 4, 20

mha = MultiHeadAttention(d_model, n_heads)
x = torch.randn(batch, T, d_model)

# Self-attention: Q = K = V = x
out, w = mha(x, x, x)
print(f"Input:  {x.shape}")
print(f"Output: {out.shape}  (mismo shape que input)")
print(f"Pesos:  {w.shape}   (batch, heads, T_q, T_k)")
print(f"Suma de pesos por posición: {w.sum(-1)[0, 0]}")  # debe ser 1.0

# Parámetros de MHA
n_params = sum(p.numel() for p in mha.parameters())
print(f"\nParámetros MHA(d={d_model}, h={n_heads}): {n_params:,d}")
print(f"  = 4 × d_model² = 4 × {d_model}² = {4 * d_model**2}")
```

---

## 5. Tipos de Self-Attention

```python
# seccion_05_tipos_atencion.py
"""
En el Transformer hay 3 tipos de atención, diferenciados por la máscara:

1. Encoder self-attention:     cada token ve TODOS los tokens (sin máscara)
2. Decoder masked self-attn:   cada token solo ve tokens ANTERIORES (causal)
3. Cross-attention:            decoder queries miran encoder keys/values
"""
import torch
import torch.nn.functional as F
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

T = 8  # longitud de secuencia

# 1. Atención sin máscara (encoder)
scores_enc = torch.randn(T, T)
w_enc = F.softmax(scores_enc, dim=-1)

# 2. Máscara causal (decoder)
mask_causal = torch.tril(torch.ones(T, T)).bool()
scores_dec = scores_enc.clone()
scores_dec[~mask_causal] = float('-inf')
w_dec = F.softmax(scores_dec, dim=-1)

# 3. Cross-attention (decoder → encoder): sin máscara especial
# (las máscaras vienen del padding del encoder si lo hay)
w_cross = F.softmax(torch.randn(T, T), dim=-1)

fig, axes = plt.subplots(1, 3, figsize=(15, 4))
titles = [
    "Encoder Self-Attention\n(cada token ve todos)",
    "Decoder Masked Self-Attention\n(solo tokens anteriores)",
    "Cross-Attention\n(decoder mira encoder)"
]
for ax, w, title in zip(axes, [w_enc, w_dec, w_cross], titles):
    im = ax.imshow(w.detach().numpy(), cmap="Blues", vmin=0, vmax=1)
    ax.set_title(title, fontsize=10)
    ax.set_xlabel("Key (posición)")
    ax.set_ylabel("Query (posición)")
    plt.colorbar(im, ax=ax)

plt.tight_layout()
plt.savefig("outputs/tipos_atencion.png", dpi=150)
print("✓ outputs/tipos_atencion.png guardado")

# Mostrar la máscara causal
print("\nMáscara causal (1=puede ver, 0=no puede):")
print(mask_causal.int().numpy())
print("\nEfecto: token en posición t solo puede atender posiciones 0..t")
print("→ garantiza autoregresión: predecir el siguiente sin ver el futuro")
```

---

## 6. Seq2Seq con Atención Completa

```python
# seccion_06_seq2seq_atencion.py
"""
Seq2Seq completo con atención de Bahdanau en PyTorch.
Tarea: ordenar secuencia de números (1..9) de forma ascendente.
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import random

torch.manual_seed(42)
random.seed(42)

# Vocabulario
SOS, EOS, PAD = 1, 2, 0
vocab_size = 12  # 0-9 + SOS + EOS

class EncoderAttn(nn.Module):
    def __init__(self, vocab_size, embed, hidden):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed, padding_idx=PAD)
        self.gru   = nn.GRU(embed, hidden, batch_first=True, bidirectional=True)
        self.fc    = nn.Linear(2*hidden, hidden)  # unificar bidireccional

    def forward(self, x):
        emb = self.embed(x)
        # H: (batch, T, 2*hidden),  hn: (2, batch, hidden)
        H, hn = self.gru(emb)
        # Combinar forward + backward del último estado
        s0 = torch.tanh(self.fc(torch.cat([hn[-2], hn[-1]], dim=1)))
        return H, s0   # H=(batch,T,2H), s0=(batch,H)

class DecoderAttn(nn.Module):
    def __init__(self, vocab_size, embed, hidden):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed, padding_idx=PAD)
        self.attn  = nn.Linear(hidden + 2*hidden, hidden)
        self.attn_v= nn.Linear(hidden, 1, bias=False)
        self.gru   = nn.GRU(embed + 2*hidden, hidden, batch_first=True)
        self.out   = nn.Linear(hidden + 2*hidden + embed, vocab_size)

    def attend(self, s, H):
        """s:(batch,hidden), H:(batch,T,2hidden) → context:(batch,2hidden)"""
        T = H.shape[1]
        s_exp = s.unsqueeze(1).expand(-1, T, -1)       # (batch,T,hidden)
        energy = torch.tanh(self.attn(torch.cat([s_exp, H], dim=2)))
        weights = F.softmax(self.attn_v(energy), dim=1) # (batch,T,1)
        context = (weights * H).sum(dim=1)               # (batch,2hidden)
        return context, weights.squeeze(2)

    def forward(self, token, s, H):
        emb = self.embed(token.unsqueeze(1))             # (batch,1,embed)
        context, w = self.attend(s, H)
        gru_in = torch.cat([emb, context.unsqueeze(1)], dim=2)
        out, s_new = self.gru(gru_in, s.unsqueeze(0))
        s_new = s_new.squeeze(0)
        # Predicción final con todas las fuentes de información
        logits = self.out(torch.cat([out.squeeze(1), context, emb.squeeze(1)], dim=1))
        return logits, s_new, w

# Modelo completo
enc = EncoderAttn(vocab_size, embed=16, hidden=32)
dec = DecoderAttn(vocab_size, embed=16, hidden=32)
opt = torch.optim.Adam(list(enc.parameters()) + list(dec.parameters()), lr=5e-3)
crit = nn.CrossEntropyLoss(ignore_index=PAD)

def make_sort_batch(batch_size=32, n=5):
    """Tarea: dado [3,1,4,1,5] → ordenar [1,1,3,4,5]"""
    srcs, trgs = [], []
    for _ in range(batch_size):
        nums = [random.randint(3, 11) for _ in range(n)]  # offset por SOS/EOS/PAD
        src  = sorted(nums, key=lambda x: random.random())  # orden aleatorio
        trg  = [SOS] + sorted(nums) + [EOS]
        srcs.append(src)
        trgs.append(trg)
    return (torch.tensor(srcs),
            torch.tensor([t[1:] for t in trgs]))  # objetivo sin SOS

# Entrenamiento
for epoch in range(300):
    src, trg = make_sort_batch(batch_size=32)
    H, s = enc(src)

    loss = 0
    token = torch.full((src.shape[0],), SOS, dtype=torch.long)
    tf = 0.7 if epoch < 150 else 0.3  # reducir teacher forcing

    for t in range(trg.shape[1]):
        logits, s, _ = dec(token, s, H)
        loss += crit(logits, trg[:, t])
        token = trg[:, t] if random.random() < tf else logits.argmax(1)

    opt.zero_grad(); loss.backward(); opt.step()
    if epoch % 100 == 99:
        print(f"Epoch {epoch+1} | loss: {loss.item():.3f}")

# Inferencia
def ordenar(nums):
    enc.eval(); dec.eval()
    with torch.no_grad():
        src = torch.tensor([nums])
        H, s = enc(src)
        token = torch.tensor([SOS])
        resultado = []
        for _ in range(len(nums)+2):
            logits, s, _ = dec(token, s, H)
            pred = logits.argmax(1).item()
            if pred == EOS:
                break
            resultado.append(pred)
            token = torch.tensor([pred])
    return resultado

prueba = [5, 3, 8, 4, 7]
print(f"\nEntrada:  {prueba}")
print(f"Esperado: {sorted(prueba)}")
print(f"Modelo:   {ordenar(prueba)}")
```

---

## 7. Atención Lineal y Eficiente

```python
# seccion_07_atencion_eficiente.py
"""
Comparativa de complejidad computacional y memoria:

- Atención estándar: O(T²·d) tiempo, O(T²) memoria → limitante para T grande
- FlashAttention, Linformer, Performer: soluciones para T largo

Este módulo muestra el problema de escala y las estrategias conceptuales.
"""
import torch
import numpy as np

# Complejidad en función de T
print("Memoria y tiempo de la atención estándar")
print(f"{'T':>8} {'Matriz attn (MB)':>18} {'FLOPs (G)':>12}")
d = 512
for T in [128, 512, 1024, 2048, 4096, 8192]:
    mem_bytes = T * T * 4  # float32
    mem_mb    = mem_bytes / 1e6
    flops_g   = 2 * T * T * d / 1e9  # QK^T + AV
    print(f"{T:>8,d} {mem_mb:>18.1f} {flops_g:>12.2f}")

print("""
→ Para T=8192 (documentos largos): 268 MB solo para la matriz de atención
→ GPT-4 (32K tokens): ~4 GB por capa solo en atención

Estrategias para atención eficiente:
──────────────────────────────────────
FlashAttention (Dao et al. 2022):
  - Mismo resultado que atención estándar
  - Fusión de kernels: no materializa la matriz T×T
  - 2-4x más rápido, O(T) memoria
  - Disponible: torch.nn.functional.scaled_dot_product_attention (PyTorch 2.0+)

Linformer (Wang et al. 2020):
  - Proyecta K, V de T→k (k<<T)
  - O(T·k) en lugar de O(T²)
  - trade-off: aproximación (no exacta)

Longformer (Beltagy et al. 2020):
  - Atención local (ventana deslizante) + atención global (CLS token)
  - O(T·w) donde w es el tamaño de ventana

Performer (Choromanski et al. 2021):
  - Aproxima softmax(QK^T) con Random Feature maps
  - O(T·d·r) donde r es el rango de la aproximación
""")

# PyTorch 2.0+ incluye FlashAttention vía scaled_dot_product_attention
print("Demo: FlashAttention con PyTorch 2.0+")
T, d, B, H = 512, 64, 2, 4

Q = torch.randn(B, H, T, d)
K = torch.randn(B, H, T, d)
V = torch.randn(B, H, T, d)

# Implementación manual (materializa T×T)
scores = torch.matmul(Q, K.transpose(-2, -1)) / np.sqrt(d)
w_manual = torch.softmax(scores, dim=-1)
out_manual = torch.matmul(w_manual, V)

# PyTorch SDPA (puede usar FlashAttention si está disponible)
out_sdpa = torch.nn.functional.scaled_dot_product_attention(Q, K, V)

# Verificar equivalencia
diff = (out_manual - out_sdpa).abs().max().item()
print(f"Diferencia max entre manual y SDPA: {diff:.6f}  (≈ 0 → equivalente)")
print(f"torch.backends.cuda.flash_sdp_enabled(): {torch.backends.cuda.flash_sdp_enabled()}")
```

---

## 8. Del Mecanismo de Atención al Transformer

```python
# seccion_08_camino_transformer.py
"""
Hoja de ruta: cómo cada componente de atención lleva al Transformer.
"""

evolucion = """
EVOLUCIÓN DEL MECANISMO DE ATENCIÓN
=====================================

1. RNN seq2seq sin atención (2014)
   ├── Encoder: h_T = tanh(W·h_{T-1} + W·x_T)
   └── Decoder: usa solo h_T  ← cuello de botella

2. Bahdanau Attention (2015)
   ├── Encoder produce [h_1, ..., h_T]
   ├── Decoder calcula score = v·tanh(W_s·s + W_h·h_i)
   └── context c_t = Σ α_i·h_i  ← dinámico

3. Luong Attention (2015)
   └── score = q·k / √d_k  ← más simple, más rápido

4. Self-Attention (2016, varios)
   └── Q = K = V = misma secuencia  ← cada token mira todos los demás

5. Multi-Head Attention (2017)
   └── H cabezas paralelas, d_model/H dims cada una  ← múltiples perspectivas

6. Transformer: "Attention is All You Need" (Vaswani et al. 2017)
   └── Elimina la RNN por completo
       ├── Encoder: Stack de capas (MHA + FFN + LayerNorm + Residual)
       ├── Decoder: Stack de capas (Masked MHA + Cross-MHA + FFN + ...)
       └── Positional Encoding: sin RNN, necesitamos inyectar posición

COMPONENTES NUEVOS DEL TRANSFORMER (próximo módulo):
  - Positional Encoding: PE(pos, 2i) = sin(pos/10000^{2i/d_model})
  - Feed-Forward: FFN(x) = max(0, xW₁+b₁)W₂+b₂  (por posición, independiente)
  - Layer Normalization: LN(x) = (x-μ)/σ * γ + β
  - Residual connections: x + Sublayer(LN(x))
"""
print(evolucion)

# Resumen de complejidades
print("COMPLEJIDADES (T=longitud secuencia, d=dimensión, k=kernel size):")
print(f"{'Capa':<30} {'Complejidad/capa':>20} {'Min camino'}")
print("-" * 65)
capas = [
    ("RNN",                   "O(T·d²)",              "O(T)"),
    ("LSTM",                  "O(T·d²)",              "O(T)"),
    ("Convolución (k)",       "O(k·T·d²)",            "O(T/k)"),
    ("Self-Attention",        "O(T²·d)",              "O(1)"),
    ("Self-Attn restringida", "O(r·T·d)",             "O(T/r)"),
]
for nombre, comp, camino in capas:
    print(f"{nombre:<30} {comp:>20} {camino}")

print("""
"Min camino" = pasos mínimos para propagar señal de posición 1 a T
→ Self-Attention: O(1) — cada par de tokens está a UN paso de atención
→ RNN: O(T) — debe pasar por cada token secuencialmente
""")
```

---

## Checkpoint M02

```python
# checkpoint_m02.py
"""
Tests de verificación para Módulo 02 — Mecanismo de Atención.
Ejecutar con: python checkpoint_m02.py
"""
import torch
import torch.nn.functional as F
import numpy as np

def test_1_pesos_suman_uno():
    """Los pesos de atención suman 1 por fila."""
    T = 12
    scores = torch.randn(4, 8, T, T)  # (batch, heads, T_q, T_k)
    weights = F.softmax(scores, dim=-1)
    sumas = weights.sum(dim=-1)
    assert torch.allclose(sumas, torch.ones_like(sumas), atol=1e-5)
    print("✓ Test 1: Pesos de atención suman 1")

def test_2_escala_sqrt_dk():
    """Dividir por sqrt(d_k) reduce la varianza del producto punto."""
    torch.manual_seed(42)
    d_k = 64
    q = torch.randn(100, d_k)
    k = torch.randn(100, d_k)
    dots = (q @ k.T).flatten()
    dots_scaled = dots / np.sqrt(d_k)
    assert dots_scaled.var() < dots.var(), "La varianza debe reducirse"
    print(f"✓ Test 2: var(sin escala)={dots.var():.2f}, var(con escala)={dots_scaled.var():.2f}")

def test_3_mascara_causal():
    """La máscara causal impide atender posiciones futuras."""
    T = 6
    scores = torch.randn(T, T)
    mask = torch.tril(torch.ones(T, T)).bool()
    scores_masked = scores.masked_fill(~mask, float('-inf'))
    w = F.softmax(scores_masked, dim=-1)
    # Posiciones futuras deben tener peso 0
    for i in range(T):
        for j in range(i+1, T):
            assert w[i, j].item() < 1e-6, f"w[{i},{j}]={w[i,j]:.6f} debe ser ~0"
    print("✓ Test 3: Máscara causal: posiciones futuras tienen peso 0")

def test_4_multihead_shape():
    """MultiHeadAttention produce output del mismo shape que el input."""
    import torch.nn as nn

    class MHA(nn.Module):
        def __init__(self, d, h):
            super().__init__()
            self.mha = nn.MultiheadAttention(d, h, batch_first=True)
        def forward(self, x):
            out, _ = self.mha(x, x, x)
            return out

    d_model, n_heads = 64, 8
    batch, T = 4, 20
    mha = MHA(d_model, n_heads)
    x = torch.randn(batch, T, d_model)
    out = mha(x)
    assert out.shape == x.shape, f"Output shape {out.shape} != input shape {x.shape}"
    print("✓ Test 4: MultiHeadAttention preserva shape")

def test_5_context_es_suma_ponderada():
    """El vector de contexto es realmente la suma ponderada de los values."""
    torch.manual_seed(1)
    T, d = 5, 8
    V = torch.randn(T, d)
    weights = F.softmax(torch.randn(T), dim=0)  # pesos que suman 1
    # Suma ponderada manual
    context_manual = (weights.unsqueeze(1) * V).sum(0)
    # Usando matmul
    context_matmul = weights @ V
    assert torch.allclose(context_manual, context_matmul, atol=1e-5)
    print("✓ Test 5: Contexto = suma ponderada correcta")

def test_6_sdpa_equivale_manual():
    """scaled_dot_product_attention produce mismo resultado que implementación manual."""
    torch.manual_seed(42)
    B, H, T, d_k = 2, 4, 16, 32
    Q = torch.randn(B, H, T, d_k)
    K = torch.randn(B, H, T, d_k)
    V = torch.randn(B, H, T, d_k)

    # Manual
    scores = torch.matmul(Q, K.transpose(-2, -1)) / np.sqrt(d_k)
    w = F.softmax(scores, dim=-1)
    out_manual = torch.matmul(w, V)

    # PyTorch SDPA
    out_sdpa = F.scaled_dot_product_attention(Q, K, V)

    assert torch.allclose(out_manual, out_sdpa, atol=1e-4), \
        f"Max diff: {(out_manual - out_sdpa).abs().max():.6f}"
    print("✓ Test 6: SDPA equivale a implementación manual")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M02 — Mecanismo de Atención")
    print("=" * 50)
    test_1_pesos_suman_uno()
    test_2_escala_sqrt_dk()
    test_3_mascara_causal()
    test_4_multihead_shape()
    test_5_context_es_suma_ponderada()
    test_6_sdpa_equivale_manual()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M02 completado")
```

---

## Resumen del Módulo

| Concepto | Fórmula | Clave |
|---|---|---|
| **Score Bahdanau** | `v·tanh(W_s·s + W_h·h)` | aditiva, paramétrica |
| **Score Luong** | `q·k / √d_k` | producto punto escalado |
| **Pesos** | `α = softmax(scores)` | suman 1, interpretables |
| **Contexto** | `c = Σ α_i · v_i` | suma ponderada de values |
| **Multi-Head** | h subatenciones paralelas | múltiples representaciones |
| **Causal mask** | `-inf` en posiciones futuras | garantiza autoregresión |
| **Complejidad** | O(T²·d) tiempo, O(T²) memoria | limitante para T largo |

**Próximo módulo:** M03 — Arquitectura Transformer en Detalle
