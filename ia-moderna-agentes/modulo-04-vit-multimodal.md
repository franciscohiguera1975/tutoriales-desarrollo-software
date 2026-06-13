# Módulo 04 — Vision Transformers (ViT) y Modelos Multimodales

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender cómo el Transformer se adapta a imágenes (ViT), implementar un ViT desde cero, y explorar los principales modelos multimodales (CLIP, LLaVA, Flamingo).
**Herramientas:** PyTorch 2.x, torchvision, NumPy, Matplotlib
**Prerequisito:** M03 (Arquitectura Transformer)

---

## 1. El Problema de Aplicar Transformers a Imágenes

```python
# seccion_01_motivacion_vit.py
"""
¿Por qué no aplicar directamente la atención a los píxeles?

Imagen 224×224×3 = 150,528 píxeles
Si cada píxel es un "token": T = 150,528
Complejidad de atención: O(T²) = O(22,600,000,000) → imposible

Solución de ViT (Dosovitskiy et al. 2020):
  → Dividir la imagen en PARCHES (patches) de 16×16 píxeles
  → 224/16 = 14 parches por fila → 14×14 = 196 parches totales
  → Cada parche = 16×16×3 = 768 valores → proyectar a d_model
  → Ahora T = 196 → O(T²) = O(38,416) → manejable
"""
print("Estrategias para Transformers en visión:")
print(f"\n{'Estrategia':<30} {'T (tokens)':>12} {'Complejidad attn':>18}")
print("-" * 62)

configs = [
    ("Píxel a píxel (224×224)",   224*224,  "imposible"),
    ("Parches 32×32",             (224//32)**2, f"{(224//32)**4:,.0f}"),
    ("Parches 16×16 (ViT-Base)",  (224//16)**2, f"{(224//16)**4:,.0f}"),
    ("Parches 8×8",               (224//8)**2,  f"{(224//8)**4:,.0f}"),
]
for nombre, T, comp in configs:
    print(f"{nombre:<30} {T:>12,d} {comp:>18}")

print(f"""
ViT-Base: 14×14 = 196 parches de 16×16 → T=196 (manejable)
Ventajas frente a CNN:
  - Campo receptivo global desde la primera capa
  - Sin inductive bias de localidad ni equivarianza traslacional
  - Escala mejor con grandes datasets (ImageNet-21k, JFT-300M)
""")
```

---

## 2. Patch Embedding — Tokenización de Imágenes

```python
# seccion_02_patch_embedding.py
"""
Patch Embedding: cómo convertir una imagen en secuencia de tokens.

1. Dividir imagen (H, W, C) en N parches de (P, P, C)
   N = (H/P) × (W/P)
2. Aplanar cada parche: P×P×C → d_patch
3. Proyectar linealmente: d_patch → d_model
4. Añadir token [CLS] al inicio (para clasificación global)
5. Sumar positional embeddings aprendibles
"""
import torch
import torch.nn as nn
import os

os.makedirs("outputs", exist_ok=True)

class PatchEmbedding(nn.Module):
    """
    Divide una imagen en parches y los proyecta a d_model.
    Implementación con Conv2d (equivalente pero más eficiente que unfold).
    """
    def __init__(self, img_size=224, patch_size=16, in_channels=3, d_model=768):
        super().__init__()
        self.n_patches = (img_size // patch_size) ** 2
        # Conv2d con kernel=stride=patch_size equivale a "patchify + linear"
        self.proj = nn.Conv2d(in_channels, d_model,
                              kernel_size=patch_size, stride=patch_size)

    def forward(self, x):
        out = self.proj(x)             # (batch, d_model, n_h, n_w)
        out = out.flatten(2)           # (batch, d_model, N)
        return out.transpose(1, 2)     # (batch, N, d_model)

# Demo
B, C, H, W = 4, 3, 224, 224
patch_size, d_model = 16, 768
n_patches = (H // patch_size) ** 2

x_img = torch.randn(B, C, H, W)
patch_embed = PatchEmbedding(H, patch_size, C, d_model)
patches = patch_embed(x_img)

print(f"Imagen original:        {tuple(x_img.shape)}")
print(f"Después de PatchEmbed:  {tuple(patches.shape)}")
print(f"  N = {n_patches} parches de {patch_size}×{patch_size}")
print(f"  d_model = {d_model}")
```

---

## 3. ViT Completo

```python
# seccion_03_vit_completo.py
"""
Vision Transformer (ViT) completo para clasificación de imágenes.
Arquitectura:
  Imagen → PatchEmbed → [CLS] + PE → TransformerEncoder → CLS → Linear → clase
"""
import torch
import torch.nn as nn

class ViT(nn.Module):
    def __init__(
        self,
        img_size=32,       # CIFAR-10: 32×32
        patch_size=4,
        in_channels=3,
        n_classes=10,
        d_model=128,
        n_heads=4,
        d_ff=512,
        n_layers=4,
        dropout=0.1,
    ):
        super().__init__()
        self.n_patches = (img_size // patch_size) ** 2
        d_patch = patch_size * patch_size * in_channels

        self.patch_embed = nn.Unfold(kernel_size=patch_size, stride=patch_size)
        self.proj        = nn.Linear(d_patch, d_model)

        # Token [CLS] aprendible
        self.cls_token = nn.Parameter(torch.zeros(1, 1, d_model))
        nn.init.trunc_normal_(self.cls_token, std=0.02)

        # Positional embeddings aprendibles (N+1: CLS + parches)
        self.pos_embed = nn.Parameter(torch.zeros(1, self.n_patches + 1, d_model))
        nn.init.trunc_normal_(self.pos_embed, std=0.02)

        enc_layer = nn.TransformerEncoderLayer(
            d_model, n_heads, d_ff, dropout, batch_first=True, norm_first=True
        )
        self.transformer = nn.TransformerEncoder(
            enc_layer, n_layers, norm=nn.LayerNorm(d_model)
        )
        self.head    = nn.Linear(d_model, n_classes)
        self.dropout = nn.Dropout(dropout)

    def forward(self, x):
        B = x.shape[0]
        # Patchify → embed
        patches = self.patch_embed(x).transpose(1, 2)   # (B, N, d_patch)
        tokens  = self.proj(patches)                     # (B, N, d_model)
        # Prepend CLS
        cls    = self.cls_token.expand(B, -1, -1)
        tokens = torch.cat([cls, tokens], dim=1)         # (B, N+1, d_model)
        tokens = self.dropout(tokens + self.pos_embed)
        # Transformer → usar CLS para clasificar
        out = self.transformer(tokens)
        return self.head(out[:, 0])

# Verificar shapes
torch.manual_seed(42)
vit = ViT(img_size=32, patch_size=4, n_classes=10, d_model=128, n_heads=4, n_layers=4)
x   = torch.randn(8, 3, 32, 32)
out = vit(x)
print(f"Input:  {tuple(x.shape)}")
print(f"Output: {tuple(out.shape)}  (batch, n_classes)")
n_params = sum(p.numel() for p in vit.parameters())
print(f"Parámetros ViT-Tiny: {n_params:,d}")
```

---

## 4. Entrenamiento en Dataset Sintético

```python
# seccion_04_vit_entrenamiento.py
"""
Entrenamiento de ViT pequeño en dataset sintético de 4 clases:
  - Clase 0: cuadrado claro en el centro
  - Clase 1: imagen oscura con ruido
  - Clase 2: gradiente horizontal
  - Clase 3: gradiente vertical
"""
import torch
import torch.nn as nn
from torch.utils.data import TensorDataset, DataLoader

torch.manual_seed(42)

def generar_dataset(n_por_clase=200, img_size=32):
    X, y = [], []
    for clase in range(4):
        for _ in range(n_por_clase):
            img = torch.zeros(3, img_size, img_size)
            if clase == 0:
                img[:, 8:24, 8:24] = 1.0
            elif clase == 1:
                img = torch.rand(3, img_size, img_size) * 0.3
            elif clase == 2:
                grad = torch.linspace(0, 1, img_size).view(1, 1, -1).expand(3, img_size, -1)
                img = grad.clone()
            else:
                grad = torch.linspace(0, 1, img_size).view(1, -1, 1).expand(3, -1, img_size)
                img = grad.clone()
            img = (img + torch.randn_like(img) * 0.1).clamp(0, 1)
            X.append(img); y.append(clase)
    return torch.stack(X), torch.tensor(y)

X, y = generar_dataset()
n = int(0.8 * len(X))
train_dl = DataLoader(TensorDataset(X[:n], y[:n]), batch_size=64, shuffle=True)
val_dl   = DataLoader(TensorDataset(X[n:], y[n:]), batch_size=64)

class ViTSmall(nn.Module):
    def __init__(self):
        super().__init__()
        ps, d = 8, 64
        N = (32 // ps) ** 2
        self.unfold = nn.Unfold(ps, stride=ps)
        self.proj   = nn.Linear(ps*ps*3, d)
        self.cls    = nn.Parameter(torch.zeros(1, 1, d))
        self.pe     = nn.Parameter(torch.randn(1, N+1, d) * 0.02)
        layer = nn.TransformerEncoderLayer(d, 4, 256, 0.1,
                                           batch_first=True, norm_first=True)
        self.enc  = nn.TransformerEncoder(layer, 3, nn.LayerNorm(d))
        self.head = nn.Linear(d, 4)

    def forward(self, x):
        B = x.shape[0]
        t = self.proj(self.unfold(x).transpose(1, 2))
        t = torch.cat([self.cls.expand(B,-1,-1), t], 1) + self.pe
        return self.head(self.enc(t)[:, 0])

model = ViTSmall()
opt   = torch.optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.05)
crit  = nn.CrossEntropyLoss()
sch   = torch.optim.lr_scheduler.CosineAnnealingLR(opt, 30)

for epoch in range(30):
    model.train()
    total = 0
    for xb, yb in train_dl:
        loss = crit(model(xb), yb)
        opt.zero_grad(); loss.backward(); opt.step()
        total += loss.item()
    sch.step()
    if epoch % 10 == 9:
        model.eval()
        with torch.no_grad():
            corr = sum((model(xb).argmax(1) == yb).sum().item() for xb, yb in val_dl)
        print(f"Epoch {epoch+1:3d} | loss: {total/len(train_dl):.4f} | val acc: {corr/len(X[n:]):.1%}")

torch.save(model.state_dict(), "outputs/vit_sintetico.pt")
print("✓ outputs/vit_sintetico.pt guardado")
```

---

## 5. CLIP — Contrastive Language-Image Pre-training

```python
# seccion_05_clip.py
"""
CLIP (Radford et al. 2021, OpenAI):
  - Encoder de imagen + encoder de texto entrenados juntos
  - Maximizar similitud entre pares (imagen, texto) correctos
  - Minimizar similitud entre pares incorrectos (contrastivo)
  - Resultado: espacio de embeddings compartido imagen-texto

Pérdida:
  Batch de N pares → matriz S[i,j] = cos(img_i, txt_j) × exp(τ)
  loss = CrossEntropy(S, diag) + CrossEntropy(S.T, diag)
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

class CLIPSimplificado(nn.Module):
    def __init__(self, d_img=128, d_txt=128, d_embed=64):
        super().__init__()
        self.img_encoder = nn.Sequential(
            nn.Flatten(), nn.Linear(3*32*32, 256), nn.ReLU(), nn.Linear(256, d_img)
        )
        self.txt_encoder = nn.Sequential(
            nn.Linear(50, 128), nn.ReLU(), nn.Linear(128, d_txt)
        )
        self.img_proj  = nn.Linear(d_img, d_embed, bias=False)
        self.txt_proj  = nn.Linear(d_txt, d_embed, bias=False)
        # Temperatura aprendible (CLIP usa log(1/0.07) ≈ 2.66 al inicio)
        self.logit_scale = nn.Parameter(torch.tensor(np.log(1/0.07)))

    def encode_image(self, img):
        return F.normalize(self.img_proj(self.img_encoder(img)), dim=-1)

    def encode_text(self, txt):
        return F.normalize(self.txt_proj(self.txt_encoder(txt)), dim=-1)

    def forward(self, imgs, txts):
        img_emb = self.encode_image(imgs)
        txt_emb = self.encode_text(txts)
        scale   = self.logit_scale.exp().clamp(max=100)
        logits  = scale * img_emb @ txt_emb.T  # (B, B)
        return logits, img_emb, txt_emb

def perdida_contrastiva(logits):
    N      = logits.shape[0]
    labels = torch.arange(N)
    return (F.cross_entropy(logits, labels) + F.cross_entropy(logits.T, labels)) / 2

torch.manual_seed(42)
model = CLIPSimplificado()
opt   = torch.optim.AdamW(model.parameters(), lr=3e-4)

for step in range(200):
    imgs = torch.randn(32, 3, 32, 32)
    txts = torch.randn(32, 50)
    logits, _, _ = model(imgs, txts)
    loss = perdida_contrastiva(logits)
    opt.zero_grad(); loss.backward(); opt.step()
    if step % 50 == 49:
        print(f"Step {step+1:3d} | loss: {loss.item():.4f} | temp: {model.logit_scale.exp().item():.2f}")

print("\nZero-Shot Classification con CLIP (concepto):")
print("  1. Embed las descripciones de cada clase (texto)")
print("  2. Embed la imagen query")
print("  3. Clase = argmax(similitud coseno imagen, embeddings de clase)")
print("  → Sin ejemplos de entrenamiento por clase")
```

---

## 6. Arquitecturas Multimodales: LLaVA y Flamingo

```python
# seccion_06_multimodal.py
"""
Dos arquitecturas de referencia para modelos multimodales.

Flamingo (DeepMind, 2022):
  ViT → Perceiver Resampler (N → M tokens fijos) → cross-attn con LLM congelado

LLaVA (2023):
  ViT (CLIP) → proyección MLP → concatenar con tokens de texto → LLM
  Entrenamiento: fase 1 (solo proyección), fase 2 (proyección + LLM)
"""
import torch
import torch.nn as nn

class LLaVAProyeccion(nn.Module):
    """Proyección MLP de visual tokens al espacio del LLM (LLaVA-1.5)."""
    def __init__(self, d_vision=1024, d_llm=4096):
        super().__init__()
        self.proj = nn.Sequential(
            nn.Linear(d_vision, d_llm), nn.GELU(), nn.Linear(d_llm, d_llm)
        )

    def forward(self, visual_feats):
        return self.proj(visual_feats)   # (batch, N_patches, d_llm)

d_vision, d_llm, N = 1024, 2048, 256
proj = LLaVAProyeccion(d_vision, d_llm)
visual_feats = torch.randn(2, N, d_vision)
visual_tokens_llm = proj(visual_feats)
text_tokens = torch.randn(2, 50, d_llm)
combined = torch.cat([visual_tokens_llm, text_tokens], dim=1)

print(f"Visual tokens (espacio LLM): {visual_tokens_llm.shape}")
print(f"Text tokens:                 {text_tokens.shape}")
print(f"Combined → LLM:              {combined.shape}")
print(f"  = {N} visual + 50 texto = {N+50} tokens totales")

print("""
Flamingo (simplificado):
  ViT output: (batch, N, d_vis)
  Perceiver Resampler: (batch, N, d_vis) → (batch, M, d_model)  M << N
  Gated Cross-Attention intercalada en cada capa del LLM congelado
  → permite adaptar el LLM a visión sin re-entrenarlo completamente

Comparativa LLaVA vs Flamingo:
  LLaVA:    simple, pocas fases de entrenamiento, excelente rendimiento
  Flamingo: más sofisticado, LLM completamente congelado, mejor few-shot
""")
```

---

## 7. ViT vs CNN — Análisis Comparativo

```python
# seccion_07_vit_vs_cnn.py
"""
Comparativa cuantitativa y conceptual: ViT vs CNN.
"""
import torch
import torch.nn as nn

class SmallCNN(nn.Module):
    def __init__(self, n_classes=10):
        super().__init__()
        self.net = nn.Sequential(
            nn.Conv2d(3, 32, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),
            nn.Conv2d(64, 128, 3, padding=1), nn.ReLU(),
            nn.AdaptiveAvgPool2d(1), nn.Flatten(),
            nn.Linear(128, n_classes),
        )
    def forward(self, x): return self.net(x)

class SmallViT(nn.Module):
    def __init__(self, n_classes=10, d=128, n_heads=4, n_layers=4):
        super().__init__()
        ps = 4; N = (32//ps)**2
        self.unfold = nn.Unfold(ps, stride=ps)
        self.proj   = nn.Linear(ps*ps*3, d)
        self.cls    = nn.Parameter(torch.randn(1, 1, d) * 0.02)
        self.pe     = nn.Parameter(torch.randn(1, N+1, d) * 0.02)
        layer = nn.TransformerEncoderLayer(d, n_heads, d*4, 0.1,
                                           batch_first=True, norm_first=True)
        self.enc  = nn.TransformerEncoder(layer, n_layers)
        self.head = nn.Linear(d, n_classes)

    def forward(self, x):
        B = x.shape[0]
        t = self.proj(self.unfold(x).transpose(1, 2))
        t = torch.cat([self.cls.expand(B,-1,-1), t], 1) + self.pe
        return self.head(self.enc(t)[:, 0])

cnn = SmallCNN(); vit = SmallViT()
x   = torch.randn(8, 3, 32, 32)

print(f"{'Modelo':<12} {'Params':>10} {'Output'}")
print(f"{'SmallCNN':<12} {sum(p.numel() for p in cnn.parameters()):>10,d} {tuple(cnn(x).shape)}")
print(f"{'SmallViT':<12} {sum(p.numel() for p in vit.parameters()):>10,d} {tuple(vit(x).shape)}")

print("""
Comparativa ViT vs CNN:
─────────────────────────────────────────────────────
               CNN                 ViT
─────────────────────────────────────────────────────
Inductive bias  Localidad, equiv.  Ninguno
Datos pequeños  ✓ Mejor            ✗ Necesita preentren.
Datos grandes   Bueno              ✓ Excelente
Campo receptivo Local → global     Global desde capa 1
Transferencia   Buena              ✓ Excelente
─────────────────────────────────────────────────────
""")
```

---

## Checkpoint M04

```python
# checkpoint_m04.py
"""
Tests de verificación para Módulo 04 — ViT y Modelos Multimodales.
Ejecutar con: python checkpoint_m04.py
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

def test_1_patch_embedding_shape():
    """PatchEmbedding produce (batch, N, d_model)."""
    B, C, H = 4, 3, 32
    ps, d   = 4, 64
    N_esperado = (H // ps) ** 2

    unfold = nn.Unfold(kernel_size=ps, stride=ps)
    proj   = nn.Linear(ps**2 * C, d)
    x      = torch.randn(B, C, H, H)
    tokens = proj(unfold(x).transpose(1, 2))

    assert tokens.shape == (B, N_esperado, d)
    print(f"✓ Test 1: PatchEmbedding → ({B}, {N_esperado}, {d})")

def test_2_cls_token_prepend():
    """CLS token se añade correctamente al inicio."""
    B, N, d  = 4, 64, 128
    tokens   = torch.randn(B, N, d)
    cls      = torch.zeros(1, 1, d).expand(B, -1, -1)
    combined = torch.cat([cls, tokens], dim=1)
    assert combined.shape == (B, N+1, d)
    assert torch.allclose(combined[:, 0, :], torch.zeros(B, d))
    print("✓ Test 2: CLS token prepend correcto")

def test_3_vit_output_shape():
    """ViT produce (batch, n_classes)."""
    ps, d = 4, 64; N = (32//ps)**2
    unfold = nn.Unfold(ps, stride=ps)
    proj   = nn.Linear(ps*ps*3, d)
    cls    = nn.Parameter(torch.zeros(1, 1, d))
    pe     = nn.Parameter(torch.zeros(1, N+1, d))
    layer  = nn.TransformerEncoderLayer(d, 4, 256, batch_first=True)
    enc    = nn.TransformerEncoder(layer, 2)
    head   = nn.Linear(d, 10)

    x = torch.randn(4, 3, 32, 32)
    t = proj(unfold(x).transpose(1, 2))
    t = torch.cat([cls.expand(4,-1,-1), t], 1) + pe
    out = head(enc(t)[:, 0])
    assert out.shape == (4, 10)
    print("✓ Test 3: ViT output shape (4, 10)")

def test_4_clip_embeddings_normalizados():
    """Los embeddings CLIP están L2-normalizados."""
    emb      = torch.randn(8, 64)
    emb_norm = F.normalize(emb, dim=-1)
    normas   = emb_norm.norm(dim=-1)
    assert torch.allclose(normas, torch.ones(8), atol=1e-5)
    print("✓ Test 4: Embeddings CLIP L2-normalizados")

def test_5_contrastive_loss_diagonal():
    """Pérdida contrastiva mínima cuando la diagonal es máxima."""
    N = 4
    labels      = torch.arange(N)
    logits_bien = torch.eye(N) * 10
    logits_mal  = torch.ones(N, N)
    loss_bien   = F.cross_entropy(logits_bien, labels)
    loss_mal    = F.cross_entropy(logits_mal,  labels)
    assert loss_bien < loss_mal
    print(f"✓ Test 5: Contrastiva: bien={loss_bien:.3f} < mal={loss_mal:.3f}")

def test_6_llava_proyeccion_shape():
    """Proyección LLaVA preserva dimensión de secuencia."""
    d_vis, d_llm, N = 512, 1024, 256
    proj = nn.Sequential(nn.Linear(d_vis, d_llm), nn.GELU(), nn.Linear(d_llm, d_llm))
    x    = torch.randn(2, N, d_vis)
    out  = proj(x)
    assert out.shape == (2, N, d_llm)
    print(f"✓ Test 6: Proyección LLaVA → (2, {N}, {d_llm})")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M04 — ViT y Modelos Multimodales")
    print("=" * 50)
    test_1_patch_embedding_shape()
    test_2_cls_token_prepend()
    test_3_vit_output_shape()
    test_4_clip_embeddings_normalizados()
    test_5_contrastive_loss_diagonal()
    test_6_llava_proyeccion_shape()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M04 completado")
```

---

## Resumen del Módulo

| Concepto | Detalle |
|---|---|
| **Patch Embedding** | Imagen → N parches de P×P → proyección a d_model |
| **CLS token** | Token aprendible al inicio → clasificación global |
| **PE aprendibles** | `pos_embed = nn.Parameter(...)` (no sinusoidal) |
| **ViT vs CNN** | ViT: sin inductive bias, escala con datos; CNN: eficiente en datos pequeños |
| **CLIP** | Espacio embedding compartido imagen-texto, pérdida contrastiva simétrica |
| **LLaVA** | ViT (CLIP) → proyección MLP → tokens del LLM concatenados |
| **Flamingo** | Perceiver Resampler → cross-attention intercalada en LLM congelado |
| **Zero-shot** | Con CLIP: embed texto de clases → comparar con embed imagen |

**Próximo módulo:** M05 — Modelos de Difusión: Stable Diffusion, DALL-E
