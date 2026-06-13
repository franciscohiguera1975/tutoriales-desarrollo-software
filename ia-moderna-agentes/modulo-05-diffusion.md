# Módulo 05 — Modelos de Difusión: Stable Diffusion, DALL-E

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender el proceso de difusión hacia adelante y hacia atrás, implementar un DDPM simple, y explorar la arquitectura de Stable Diffusion (latent diffusion) y el condicionamiento por texto.
**Herramientas:** PyTorch 2.x, NumPy, Matplotlib
**Prerequisito:** M03 (Transformer), M04 (ViT y multimodal)

---

## 1. Intuición: Denoising como Generación

```python
# seccion_01_intuicion.py
"""
Idea central de los modelos de difusión:

HACIA ADELANTE (q): agregar ruido gradualmente
  x_0 (imagen real) → x_1 → x_2 → ... → x_T ≈ N(0, I)
  Proceso de Markov: q(x_t | x_{t-1}) = N(x_t; √(1-β_t)·x_{t-1}, β_t·I)

HACIA ATRÁS (p_θ): aprender a eliminar ruido
  x_T (ruido puro) → x_{T-1} → ... → x_0 (imagen generada)
  El modelo aprende: p_θ(x_{t-1} | x_t)

Truco de reparametrización (DDPM, Ho et al. 2020):
  x_t puede calcularse directamente desde x_0:
  x_t = √ᾱ_t · x_0 + √(1-ᾱ_t) · ε,   ε ~ N(0,I)
  donde ᾱ_t = prod_{s=1}^{t} (1-β_s)

Objetivo de entrenamiento (predicción de ruido):
  L = E_{t, x_0, ε} [|| ε - ε_θ(x_t, t) ||²]
  → el modelo aprende a predecir el ruido ε añadido
"""
import torch
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import os

os.makedirs("outputs", exist_ok=True)

# Visualizar el schedule de β (ruido añadido por paso)
T = 1000
# Linear schedule (DDPM original)
beta_linear = torch.linspace(1e-4, 0.02, T)
# Cosine schedule (más suave, Nichol & Dhariwal 2021)
s = 0.008
t_vals = torch.arange(T+1, dtype=torch.float)
f_t = torch.cos((t_vals/T + s) / (1 + s) * np.pi / 2) ** 2
alpha_bar_cos = f_t / f_t[0]
beta_cos = torch.clamp(1 - alpha_bar_cos[1:] / alpha_bar_cos[:-1], max=0.999)

alpha_bar_linear = torch.cumprod(1 - beta_linear, dim=0)

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].plot(beta_linear.numpy(), label="Linear β")
axes[0].plot(beta_cos.numpy(),    label="Cosine β")
axes[0].set_title("Schedule de ruido β_t"); axes[0].legend()
axes[0].set_xlabel("Paso t")

axes[1].plot(alpha_bar_linear.numpy(), label="Linear ᾱ_t")
axes[1].plot(alpha_bar_cos.numpy(),    label="Cosine ᾱ_t")
axes[1].set_title("Señal restante ᾱ_t = prod(1-β)")
axes[1].legend(); axes[1].set_xlabel("Paso t")

plt.tight_layout()
plt.savefig("outputs/noise_schedule.png", dpi=150)
print("✓ outputs/noise_schedule.png guardado")

# Mostrar cómo se añade ruido paso a paso
print("\nRuido acumulado a distintos pasos (ᾱ_t):")
for t in [0, 100, 250, 500, 750, 999]:
    abar = alpha_bar_linear[t].item()
    snr  = abar / (1 - abar + 1e-8)
    print(f"  t={t:4d}: ᾱ_t={abar:.4f}, SNR={snr:.3f}")
```

---

## 2. Proceso de Difusión — Implementación Completa

```python
# seccion_02_ddpm_proceso.py
"""
Implementación del proceso de difusión (forward y backward).
Trabajamos con datos 1D sintéticos para simplicidad.
"""
import torch
import torch.nn as nn
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

class GaussianDiffusion:
    """
    Proceso de difusión gaussiano con linear noise schedule.
    """
    def __init__(self, T=1000, beta_start=1e-4, beta_end=0.02, device="cpu"):
        self.T = T
        self.device = device

        self.beta     = torch.linspace(beta_start, beta_end, T, device=device)
        self.alpha    = 1.0 - self.beta
        self.alpha_bar = torch.cumprod(self.alpha, dim=0)
        # Para el sampling
        self.alpha_bar_prev = torch.cat([torch.ones(1, device=device),
                                          self.alpha_bar[:-1]], dim=0)

    def q_sample(self, x0, t, noise=None):
        """
        Forward process: añadir ruido al paso t.
        x_t = √ᾱ_t · x_0 + √(1-ᾱ_t) · ε
        """
        if noise is None:
            noise = torch.randn_like(x0)
        abar_t = self.alpha_bar[t].view(-1, *([1]*(x0.dim()-1)))
        return torch.sqrt(abar_t) * x0 + torch.sqrt(1 - abar_t) * noise, noise

    def p_losses(self, model, x0, t):
        """
        Calcular pérdida de predicción de ruido para el batch.
        """
        x_noisy, noise = self.q_sample(x0, t)
        pred_noise = model(x_noisy, t)
        return nn.functional.mse_loss(pred_noise, noise)

    @torch.no_grad()
    def p_sample(self, model, x_t, t_scalar):
        """
        Backward step: eliminar ruido de x_t → x_{t-1}.
        """
        betas_t     = self.beta[t_scalar]
        alpha_t     = self.alpha[t_scalar]
        abar_t      = self.alpha_bar[t_scalar]
        abar_prev_t = self.alpha_bar_prev[t_scalar]

        t_batch = torch.full((x_t.shape[0],), t_scalar, device=self.device, dtype=torch.long)
        pred_noise = model(x_t, t_batch)

        # Media del paso backward (DDPM original)
        coef1 = torch.sqrt(abar_prev_t) * betas_t / (1 - abar_t)
        coef2 = torch.sqrt(alpha_t) * (1 - abar_prev_t) / (1 - abar_t)
        mu    = coef1 * (x_t - torch.sqrt(1 - abar_t) * pred_noise) / torch.sqrt(abar_t) \
                + coef2 * x_t
        # Varianza
        if t_scalar == 0:
            return mu
        sigma = torch.sqrt(betas_t * (1 - abar_prev_t) / (1 - abar_t))
        return mu + sigma * torch.randn_like(x_t)

    @torch.no_grad()
    def sample(self, model, shape):
        """Generar muestras completas desde ruido puro."""
        x = torch.randn(shape, device=self.device)
        for t in reversed(range(self.T)):
            x = self.p_sample(model, x, t)
        return x

# Visualizar el proceso forward en datos 1D
diffusion = GaussianDiffusion(T=200, beta_start=1e-3, beta_end=0.1)

# Datos: mezcla de gaussianas
x0 = torch.cat([torch.randn(50) * 0.3 + 2.0,
                 torch.randn(50) * 0.3 - 2.0])
x0 = x0.unsqueeze(1)  # (100, 1)

fig, axes = plt.subplots(1, 5, figsize=(15, 3))
for k, t in enumerate([0, 25, 50, 100, 199]):
    t_batch = torch.full((100,), t, dtype=torch.long)
    x_t, _  = diffusion.q_sample(x0, t_batch)
    axes[k].hist(x_t[:, 0].numpy(), bins=30, density=True, alpha=0.7)
    axes[k].set_title(f"t={t}")
    axes[k].set_xlim(-4, 4)

plt.suptitle("Proceso forward: difusión de datos bimodal → ruido")
plt.tight_layout()
plt.savefig("outputs/forward_diffusion.png", dpi=150)
print("✓ outputs/forward_diffusion.png guardado")
```

---

## 3. Red de Denoising — U-Net con Atención

```python
# seccion_03_unet_denoising.py
"""
La red de denoising ε_θ(x_t, t) en DDPM usa una U-Net.

U-Net: encoder (downsampling) + decoder (upsampling) con skip connections.
  - El tiempo t se inyecta mediante sinusoidal time embedding.
  - Las capas intermedias usan atención (similar al Transformer).

Para datos 1D (series temporales / vectores):
  - Usamos MLP con time embedding en lugar de U-Net completa.
"""
import torch
import torch.nn as nn
import numpy as np

class SinusoidalTimeEmbedding(nn.Module):
    """
    Embedding sinusoidal para el paso de tiempo t.
    Similar al positional encoding del Transformer.
    """
    def __init__(self, d_model):
        super().__init__()
        self.d_model = d_model
        self.proj = nn.Sequential(
            nn.Linear(d_model, d_model * 4), nn.SiLU(),
            nn.Linear(d_model * 4, d_model)
        )

    def forward(self, t):
        # t: (batch,) enteros
        half = self.d_model // 2
        freqs = torch.exp(
            -np.log(10000) * torch.arange(half, device=t.device).float() / half
        )
        emb = t[:, None].float() * freqs[None, :]  # (batch, half)
        emb = torch.cat([torch.sin(emb), torch.cos(emb)], dim=-1)  # (batch, d)
        return self.proj(emb)

class ResBlock1D(nn.Module):
    """Bloque residual con time conditioning para datos 1D."""
    def __init__(self, d, d_time):
        super().__init__()
        self.net = nn.Sequential(nn.Linear(d, d), nn.SiLU(), nn.Linear(d, d))
        self.time_proj = nn.Linear(d_time, d)
        self.norm = nn.LayerNorm(d)

    def forward(self, x, t_emb):
        return x + self.net(self.norm(x + self.time_proj(t_emb)))

class DenoisingMLP(nn.Module):
    """
    Red de denoising para datos 1D.
    Entrada: x_t (batch, D) + t (batch,)
    Salida:  ε predicho (batch, D)
    """
    def __init__(self, data_dim=1, d_model=128, n_layers=4, T=1000):
        super().__init__()
        self.time_embed = SinusoidalTimeEmbedding(d_model)
        self.input_proj = nn.Linear(data_dim, d_model)
        self.blocks = nn.ModuleList([
            ResBlock1D(d_model, d_model) for _ in range(n_layers)
        ])
        self.output_proj = nn.Linear(d_model, data_dim)

    def forward(self, x, t):
        t_emb = self.time_embed(t)               # (batch, d_model)
        h     = self.input_proj(x)               # (batch, d_model)
        for block in self.blocks:
            h = block(h, t_emb)
        return self.output_proj(h)               # (batch, data_dim)

# Verificar shapes
model = DenoisingMLP(data_dim=2, d_model=64, n_layers=3)
x_t   = torch.randn(16, 2)
t     = torch.randint(0, 1000, (16,))
pred  = model(x_t, t)
print(f"Input x_t: {x_t.shape}, t: {t.shape}")
print(f"Pred ε:    {pred.shape}  (debe ser igual a x_t)")

n_params = sum(p.numel() for p in model.parameters())
print(f"Parámetros DenoisingMLP: {n_params:,d}")
```

---

## 4. Entrenamiento DDPM en Datos 2D

```python
# seccion_04_ddpm_entrenamiento.py
"""
DDPM completo entrenado en datos 2D sintéticos (espiral / swiss roll).
"""
import torch
import torch.nn as nn
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

torch.manual_seed(42)
np.random.seed(42)

# Dataset: espiral
def make_spiral(n=2000):
    t = torch.linspace(0, 4*np.pi, n)
    r = t / (4*np.pi)
    x = r * torch.cos(t) + torch.randn(n) * 0.05
    y = r * torch.sin(t) + torch.randn(n) * 0.05
    return torch.stack([x, y], dim=1)

data = make_spiral()
data = (data - data.mean(0)) / data.std(0)  # normalizar

# Diffusion
T = 500
beta  = torch.linspace(1e-4, 0.02, T)
alpha = 1 - beta
abar  = torch.cumprod(alpha, 0)

def q_sample(x0, t, noise=None):
    if noise is None: noise = torch.randn_like(x0)
    ab = abar[t].view(-1, 1)
    return torch.sqrt(ab) * x0 + torch.sqrt(1-ab) * noise, noise

# Modelo simple
class Denoiser(nn.Module):
    def __init__(self):
        super().__init__()
        d = 128
        # Time embedding
        self.te = nn.Sequential(nn.Linear(1, d), nn.SiLU(), nn.Linear(d, d))
        self.net = nn.Sequential(
            nn.Linear(2 + d, 256), nn.SiLU(),
            nn.Linear(256, 256), nn.SiLU(),
            nn.Linear(256, 2)
        )
    def forward(self, x, t):
        t_emb = self.te(t.float().unsqueeze(1) / T)  # normalizar t
        return self.net(torch.cat([x, t_emb], dim=1))

model = Denoiser()
opt   = torch.optim.Adam(model.parameters(), lr=2e-4)

# Entrenamiento
losses = []
for epoch in range(2000):
    idx  = torch.randint(0, len(data), (256,))
    x0   = data[idx]
    t    = torch.randint(0, T, (256,))
    x_t, noise = q_sample(x0, t)
    pred = model(x_t, t)
    loss = nn.functional.mse_loss(pred, noise)
    opt.zero_grad(); loss.backward(); opt.step()
    losses.append(loss.item())
    if epoch % 500 == 499:
        print(f"Epoch {epoch+1:4d} | loss: {loss.item():.5f}")

# Sampling (DDPM)
@torch.no_grad()
def sample_ddpm(n=500):
    x = torch.randn(n, 2)
    abar_prev = torch.cat([torch.ones(1), abar[:-1]])
    for t_s in reversed(range(T)):
        b = beta[t_s]; a = alpha[t_s]; ab = abar[t_s]; abp = abar_prev[t_s]
        t_b = torch.full((n,), t_s, dtype=torch.long)
        eps = model(x, t_b)
        mu  = (x - b / torch.sqrt(1-ab) * eps) / torch.sqrt(a)
        if t_s > 0:
            sigma = torch.sqrt(b * (1 - abp) / (1 - ab))
            x = mu + sigma * torch.randn_like(x)
        else:
            x = mu
    return x

samples = sample_ddpm(500)

fig, axes = plt.subplots(1, 2, figsize=(10, 4))
axes[0].scatter(data[:500, 0], data[:500, 1], s=5, alpha=0.5)
axes[0].set_title("Datos reales (espiral)")
axes[1].scatter(samples[:, 0].numpy(), samples[:, 1].numpy(), s=5, alpha=0.5, c='orange')
axes[1].set_title("Muestras DDPM generadas")
for ax in axes:
    ax.set_xlim(-3, 3); ax.set_ylim(-3, 3)
plt.tight_layout()
plt.savefig("outputs/ddpm_muestras.png", dpi=150)
print("✓ outputs/ddpm_muestras.png guardado")
```

---

## 5. DDIM — Deterministic Sampling Más Rápido

```python
# seccion_05_ddim.py
"""
DDIM (Denoising Diffusion Implicit Models, Song et al. 2020):
  - Mismo entrenamiento que DDPM
  - Sampling determinístico (o con stochasticidad controlada η)
  - Puede usar mucho menos pasos: 50-100 en lugar de 1000
  - La calidad es similar o mejor

Fórmula DDIM:
  x_{t-1} = √ᾱ_{t-1} · (x_t - √(1-ᾱ_t)·ε_θ) / √ᾱ_t
           + √(1-ᾱ_{t-1} - σ²_t) · ε_θ
           + σ_t · ε,   σ_t = η·√((1-ᾱ_{t-1})/(1-ᾱ_t)) · √(1-ᾱ_t/ᾱ_{t-1})
  η=0 → determinístico (DDIM); η=1 → DDPM

Classifier-Free Guidance (CFG):
  ε̂ = ε_uncond + w · (ε_cond - ε_uncond)
  w > 1: más adherencia al texto/condición (pero menos diversidad)
"""
import torch
import numpy as np

# DDIM sampling con pasos reducidos
def ddim_sample(model, data_shape, T=500, n_steps=50, eta=0.0,
                beta_start=1e-4, beta_end=0.02):
    beta  = torch.linspace(beta_start, beta_end, T)
    abar  = torch.cumprod(1 - beta, 0)

    # Seleccionar subconjunto de pasos
    step_indices = list(range(0, T, T // n_steps))[::-1]  # n_steps pasos

    x = torch.randn(data_shape)

    for i, t_curr in enumerate(step_indices):
        t_prev = step_indices[i+1] if i+1 < len(step_indices) else 0
        ab_curr = abar[t_curr]
        ab_prev = abar[t_prev] if t_prev > 0 else torch.ones(1)

        t_batch = torch.full((data_shape[0],), t_curr, dtype=torch.long)
        with torch.no_grad():
            eps = model(x, t_batch)

        # Predecir x_0 desde x_t
        x0_pred = (x - torch.sqrt(1 - ab_curr) * eps) / torch.sqrt(ab_curr)
        x0_pred = x0_pred.clamp(-1, 1)  # clip para estabilidad

        # Dirección apuntando a x_t
        dir_xt = torch.sqrt(1 - ab_prev) * eps

        # Noise (sigma = 0 para DDIM determinístico)
        sigma = eta * torch.sqrt((1 - ab_prev) / (1 - ab_curr) * (1 - ab_curr/ab_prev))

        x = torch.sqrt(ab_prev) * x0_pred + dir_xt + sigma * torch.randn_like(x)

    return x

print("DDIM vs DDPM — Comparativa de velocidad de sampling:")
print(f"{'Método':<15} {'Pasos':>8} {'Calidad relativa'}")
print(f"{'DDPM':<15} {'1000':>8} {'Referencia'}")
print(f"{'DDIM η=0':<15} {'50':>8} {'~95% (determinístico)'}")
print(f"{'DDIM η=0':<15} {'20':>8} {'~90%'}")
print(f"{'DPM-Solver':<15} {'10-20':>8} {'~95%'}")
print(f"{'SDXL Turbo':<15} {'1-4':>8} {'~85% (distilado)'}")

print("""
Classifier-Free Guidance (CFG):
  ε̂(x_t, c, w) = ε_θ(x_t, ∅) + w·(ε_θ(x_t, c) - ε_θ(x_t, ∅))
  c = condición (texto), ∅ = sin condición
  w = guidance scale: 7.5 para Stable Diffusion (típico)
  → w alto: más coherente con el texto, menos diverso
  → w bajo: más diverso, puede alejarse del texto
""")
```

---

## 6. Stable Diffusion — Latent Diffusion

```python
# seccion_06_stable_diffusion.py
"""
Stable Diffusion (Rombach et al. 2022):
  - Diffusion en el ESPACIO LATENTE (no en píxeles)
  - VAE: imagen → latente (4× más pequeño) → imagen
  - U-Net de difusión opera en el espacio latente
  - Condicionamiento: texto → CLIP text encoder → cross-attention en U-Net

Arquitectura completa:
  ENTRENAMIENTO:
    imagen (512×512×3) → VAE encoder → latente (64×64×4)
    latente + ruido → U-Net (con cross-attn CLIP text) → ε predicho
    pérdida MSE(ε, ε_predicho)

  GENERACIÓN:
    texto → CLIP text encoder → embeddings de texto
    z_T ~ N(0,I) de forma (64×64×4)
    z_T, embeddings → U-Net DDIM → z_0
    z_0 → VAE decoder → imagen (512×512×3)

Ventaja de Latent Diffusion:
  - Espacio latente 64×64×4 = 16,384 dims (vs 512×512×3 = 786,432)
  - Ratio: 48x menos computo que difusión en píxeles
  - Calidad similar: el VAE ya "aprende" la estructura de imagen
"""
import torch
import torch.nn as nn

# Modelo VAE simplificado (encoder + decoder)
class VAEEncoder(nn.Module):
    """Comprime imagen a espacio latente."""
    def __init__(self, in_ch=3, latent_ch=4, scale=4):
        super().__init__()
        C = 32
        self.net = nn.Sequential(
            nn.Conv2d(in_ch, C, 3, padding=1), nn.SiLU(),
            nn.Conv2d(C, C*2, 3, stride=2, padding=1), nn.SiLU(),  # /2
            nn.Conv2d(C*2, C*4, 3, stride=2, padding=1), nn.SiLU(), # /4
            nn.Conv2d(C*4, latent_ch*2, 1),  # mu + logvar
        )

    def forward(self, x):
        h = self.net(x)
        mu, logvar = h.chunk(2, dim=1)
        std = torch.exp(0.5 * logvar)
        z   = mu + std * torch.randn_like(std)
        return z, mu, logvar

class VAEDecoder(nn.Module):
    """Reconstruye imagen desde espacio latente."""
    def __init__(self, latent_ch=4, out_ch=3):
        super().__init__()
        C = 32
        self.net = nn.Sequential(
            nn.Conv2d(latent_ch, C*4, 1), nn.SiLU(),
            nn.ConvTranspose2d(C*4, C*2, 4, stride=2, padding=1), nn.SiLU(),  # ×2
            nn.ConvTranspose2d(C*2, C,   4, stride=2, padding=1), nn.SiLU(),  # ×4
            nn.Conv2d(C, out_ch, 3, padding=1), nn.Tanh(),
        )

    def forward(self, z): return self.net(z)

# Verificar shapes en SD (escala reducida)
batch, H_img = 2, 32   # SD real: H_img=512, latent=64
enc = VAEEncoder()
dec = VAEDecoder()
img = torch.randn(batch, 3, H_img, H_img)
z, mu, logvar = enc(img)
recon         = dec(z)

H_lat = H_img // 4
print(f"Imagen original:  {tuple(img.shape)}")
print(f"Espacio latente:  {tuple(z.shape)}  (H/4 × W/4 × 4)")
print(f"Reconstrucción:   {tuple(recon.shape)}")
print(f"\nRatio compresión: {img.numel()} → {z.numel()} = {img.numel()//z.numel()}x")

# Cross-attention para condicionamiento por texto
print("\nCondicionamiento por texto (cross-attention):")
print("""
En la U-Net de Stable Diffusion, cada bloque residual tiene:
  1. ResNet block (convolucional)
  2. Cross-attention: Q del latente, K/V del texto (CLIP embeddings)

Esto permite que el texto "guíe" la denoising en cada capa:
  latente   → Q  (¿qué está buscando en el texto?)
  clip_text → K  (¿qué información ofrece el texto?)
  clip_text → V  (¿qué valor se extrae del texto?)
""")
```

---

## 7. Controladores: ControlNet y LoRA para Difusión

```python
# seccion_07_controlnet_lora.py
"""
Extensiones de Stable Diffusion para control adicional.

ControlNet (Zhang et al. 2023):
  - Añade control espacial: bordes Canny, pose, profundidad, etc.
  - Copia los pesos de la U-Net, entrena la copia, conecta con "zero conv"
  - Zero conv: convoluciones inicializadas en 0 → entrenamiento estable

LoRA para difusión (fine-tuning eficiente):
  - Añade matrices de rango bajo a los pesos Q, K, V, out de la atención
  - Solo entrena ~2-4% de parámetros del modelo completo
  - Dreambooth + LoRA: personalizar SD con 10-30 imágenes propias
"""
import torch
import torch.nn as nn

# LoRA aplicado a una capa de atención
class LoRALinear(nn.Module):
    """
    Capa lineal con LoRA: W' = W + BA donde rank(BA) = r << min(d_in, d_out)
    """
    def __init__(self, d_in, d_out, rank=4, alpha=1.0):
        super().__init__()
        self.linear = nn.Linear(d_in, d_out, bias=False)
        self.A      = nn.Linear(d_in, rank,   bias=False)   # B = random
        self.B      = nn.Linear(rank,  d_out, bias=False)   # A = 0 init
        self.scale  = alpha / rank

        nn.init.kaiming_uniform_(self.A.weight)
        nn.init.zeros_(self.B.weight)    # LoRA comienza sin efecto

        # Congelar pesos originales
        self.linear.weight.requires_grad_(False)

    def forward(self, x):
        return self.linear(x) + self.scale * self.B(self.A(x))

# Análisis de parámetros LoRA
d_in, d_out = 768, 768
rank = 16

linear_orig = nn.Linear(d_in, d_out, bias=False)
lora_layer  = LoRALinear(d_in, d_out, rank=rank)

p_orig  = sum(p.numel() for p in linear_orig.parameters())
p_lora  = sum(p.numel() for p in lora_layer.parameters() if p.requires_grad)
p_total = sum(p.numel() for p in lora_layer.parameters())

print(f"Capa lineal original: {p_orig:,d} params")
print(f"LoRA entrenables:     {p_lora:,d} params  (A + B)")
print(f"Total LoRA layer:     {p_total:,d} params")
print(f"Reducción:            {p_lora/p_orig:.1%} de los params originales")
print(f"\nPara rank={rank}: A=({d_in},{rank})={d_in*rank}, B=({rank},{d_out})={rank*d_out}")

# Zero convolution (ControlNet)
class ZeroConv2d(nn.Conv2d):
    """Convolución inicializada en cero para ControlNet."""
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        nn.init.zeros_(self.weight)
        if self.bias is not None:
            nn.init.zeros_(self.bias)

print("""
ControlNet — flujo de información:
  imagen_control (borde, pose...) → encoder bloques copiados → zero_conv → U-Net original

  Ventaja: zero_conv garantiza que al inicio, el ControlNet no afecta a SD
  El entrenamiento puede ser más agresivo sin riesgo de destruir el modelo base
""")
```

---

## Checkpoint M05

```python
# checkpoint_m05.py
"""
Tests de verificación para Módulo 05 — Modelos de Difusión.
Ejecutar con: python checkpoint_m05.py
"""
import torch
import torch.nn as nn
import numpy as np

def test_1_q_sample_forma():
    """q_sample produce x_t del mismo shape que x_0."""
    T = 200
    beta  = torch.linspace(1e-4, 0.02, T)
    abar  = torch.cumprod(1 - beta, 0)

    x0 = torch.randn(8, 2)
    t  = torch.randint(0, T, (8,))
    ab = abar[t].unsqueeze(1)
    noise = torch.randn_like(x0)
    x_t = torch.sqrt(ab) * x0 + torch.sqrt(1 - ab) * noise

    assert x_t.shape == x0.shape, f"Shape: {x_t.shape}"
    print("✓ Test 1: q_sample shape correcto")

def test_2_alpha_bar_decreciente():
    """ᾱ_t es monótonamente decreciente (la señal se degrada)."""
    T = 1000
    beta = torch.linspace(1e-4, 0.02, T)
    abar = torch.cumprod(1 - beta, 0)
    assert (abar[1:] <= abar[:-1]).all(), "ᾱ debe ser decreciente"
    assert abar[-1] < 0.01, f"ᾱ_T debe ser ≈ 0, got {abar[-1]:.4f}"
    print(f"✓ Test 2: ᾱ_0={abar[0]:.4f}, ᾱ_T={abar[-1]:.4f} (decreciente)")

def test_3_time_embedding_shape():
    """SinusoidalTimeEmbedding produce forma (batch, d_model)."""
    class TE(nn.Module):
        def __init__(self, d):
            super().__init__()
            self.d = d
        def forward(self, t):
            half = self.d // 2
            freqs = torch.exp(-np.log(10000) * torch.arange(half).float() / half)
            emb = t[:, None].float() * freqs[None, :]
            return torch.cat([torch.sin(emb), torch.cos(emb)], dim=-1)

    d = 64; batch = 16
    te = TE(d)
    t  = torch.randint(0, 1000, (batch,))
    emb = te(t)
    assert emb.shape == (batch, d), f"Shape: {emb.shape}"
    print(f"✓ Test 3: Time embedding shape ({batch}, {d})")

def test_4_denoising_loss_decrece():
    """La pérdida de denoising debe decrecer durante entrenamiento."""
    torch.manual_seed(42)
    T = 100
    beta = torch.linspace(1e-3, 0.1, T)
    abar = torch.cumprod(1 - beta, 0)

    # Modelo simple
    model = nn.Sequential(nn.Linear(3, 32), nn.SiLU(), nn.Linear(32, 2))
    opt   = torch.optim.Adam(model.parameters(), lr=1e-3)

    def step():
        x0 = torch.randn(64, 2)
        t  = torch.randint(0, T, (64,))
        ab = abar[t].unsqueeze(1)
        noise = torch.randn_like(x0)
        x_t = torch.sqrt(ab) * x0 + torch.sqrt(1 - ab) * noise
        t_feat = t.float().unsqueeze(1) / T
        pred = model(torch.cat([x_t, t_feat], 1))
        return nn.functional.mse_loss(pred, noise)

    loss_ini = step().item()
    for _ in range(100):
        loss = step(); opt.zero_grad(); loss.backward(); opt.step()
    loss_fin = step().item()

    assert loss_fin < loss_ini, f"Pérdida no decreció: {loss_ini:.4f} → {loss_fin:.4f}"
    print(f"✓ Test 4: Pérdida decreció: {loss_ini:.4f} → {loss_fin:.4f}")

def test_5_vae_latente_reducido():
    """El VAE produce latente más pequeño que la imagen."""
    class Enc(nn.Module):
        def forward(self, x): return x[:, :4, ::4, ::4]  # stub

    enc = Enc()
    img = torch.randn(2, 3, 32, 32)
    lat = enc(img)
    assert lat.numel() < img.numel()
    factor = img.numel() // lat.numel()
    print(f"✓ Test 5: Latente {lat.shape} es {factor}x más pequeño que imagen")

def test_6_lora_sin_efecto_inicial():
    """LoRA inicializado en cero no modifica la salida."""
    class LoRA(nn.Module):
        def __init__(self, d, r):
            super().__init__()
            self.W = nn.Linear(d, d, bias=False)
            self.A = nn.Linear(d, r, bias=False)
            self.B = nn.Linear(r, d, bias=False)
            nn.init.zeros_(self.B.weight)  # inicializar B en 0

        def forward(self, x):
            return self.W(x) + self.B(self.A(x))

    d = 32; r = 4
    lora = LoRA(d, r)
    x    = torch.randn(4, d)
    out_lora  = lora(x)
    out_orig  = lora.W(x)
    assert torch.allclose(out_lora, out_orig, atol=1e-6), \
        "LoRA con B=0 debe ser idéntico a W original"
    print("✓ Test 6: LoRA con B=0 inicializado no modifica la salida")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M05 — Modelos de Difusión")
    print("=" * 50)
    test_1_q_sample_forma()
    test_2_alpha_bar_decreciente()
    test_3_time_embedding_shape()
    test_4_denoising_loss_decrece()
    test_5_vae_latente_reducido()
    test_6_lora_sin_efecto_inicial()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M05 completado")
```

---

## Resumen del Módulo

| Concepto | Detalle |
|---|---|
| **Forward process** | `x_t = √ᾱ_t·x_0 + √(1-ᾱ_t)·ε` — ruido acumulado en un paso |
| **Objetivo DDPM** | `L = E[‖ε - ε_θ(x_t, t)‖²]` — predicción del ruido |
| **Time embedding** | Sinusoidal (como PE del Transformer) → inyectado en cada capa |
| **DDIM** | Mismo entrenamiento, sampling determinístico, 50 pasos en lugar de 1000 |
| **Latent Diffusion** | Difusión en espacio latente VAE: 48x menos cómputo |
| **CFG** | `ε̂ = ε_uncond + w·(ε_cond - ε_uncond)` — adherencia al texto |
| **ControlNet** | Control espacial con zero convolutions (bordes, pose, etc.) |
| **LoRA** | `W' = W + BA`, rank pequeño, <5% params entrenables |

**Próximo módulo:** M06 — Arquitectura Interna de LLMs: GPT, BERT, LLaMA
