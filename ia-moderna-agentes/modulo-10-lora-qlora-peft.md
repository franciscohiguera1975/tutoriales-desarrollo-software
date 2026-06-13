# Módulo 10 — Fine-tuning Eficiente: LoRA, QLoRA, PEFT

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Dominar las técnicas de Parameter-Efficient Fine-Tuning (PEFT): LoRA, QLoRA, Adapters, prefix tuning y prompt tuning; implementarlas desde cero y con la librería peft de HuggingFace.
**Herramientas:** PyTorch 2.x, peft (HuggingFace), bitsandbytes
**Prerequisito:** M09 (Fine-tuning SFT/RLHF)

---

## 1. Motivación: Fine-tuning de LLMs Grandes es Imposible (sin PEFT)

```python
# seccion_01_motivacion.py
"""
Fine-tuning completo de LLMs grandes:
  LLaMA-7B: 7B parámetros × 4 bytes (fp32) = 28 GB solo para el modelo
             + gradientes: 28 GB
             + estados del optimizador (Adam): 56 GB
             Total: ~112 GB de VRAM → necesitas 4× A100 80GB

PEFT (Parameter-Efficient Fine-Tuning):
  Solo actualizar una pequeña fracción de parámetros (~0.1-5%)
  Mantener el resto del modelo congelado

Beneficios:
  - Memoria: LLaMA-7B con LoRA rank=16 → ~60 MB de parámetros entrenables
  - Velocidad: menos cómputo al hacer backward
  - Generalización: menos overfitting (menos parámetros)
  - Modularidad: guardar solo los adaptadores, no todo el modelo
"""

print("Comparativa de recursos para fine-tuning LLaMA-7B:")
print(f"\n{'Método':<20} {'Params entrenables':>20} {'VRAM aprox':>12}")
print("-" * 55)
metodos = [
    ("Full fine-tuning",  "7,000M (100%)",    ">80 GB"),
    ("LoRA (rank=64)",    "~120M (1.7%)",      "~16 GB"),
    ("LoRA (rank=16)",    "~30M (0.4%)",       "~12 GB"),
    ("QLoRA (4-bit)",     "~30M (0.4%)",       "~6 GB"),
    ("Prefix tuning",     "~10M (0.1%)",       "~10 GB"),
    ("Prompt tuning",     "~0.3M (0.004%)",    "~10 GB"),
]
for m, p, v in metodos:
    print(f"{m:<20} {p:>20} {v:>12}")

print("""
Regla práctica 2024:
  GPU 8 GB  → QLoRA con LLaMA-7B
  GPU 16 GB → LoRA con LLaMA-7B o QLoRA con LLaMA-13B
  GPU 24 GB → LoRA con LLaMA-13B o QLoRA con LLaMA-34B
  GPU 80 GB → LoRA con LLaMA-70B o full fine-tuning LLaMA-7B
""")
```

---

## 2. LoRA — Low-Rank Adaptation

```python
# seccion_02_lora.py
"""
LoRA (Hu et al. 2021): Low-Rank Adaptation of Large Language Models

Idea: los updates de fine-tuning tienen rango intrínseco bajo.
  W' = W + ΔW = W + BA
  donde B ∈ R^{d×r}, A ∈ R^{r×k}, rank r << min(d,k)

LoRA solo entrena A y B (inicializados A~N(0,σ²), B=0):
  - Al inicio: ΔW = BA = 0 → el modelo es idéntico al base
  - Después: ΔW tiene rank r

Scaling: W' = W + α/r · BA
  - α: escala (típicamente igual a r)
  - Normaliza el learning rate respecto al rank

¿Dónde aplicar LoRA?
  - Tipicamente: Q, V de la atención (mínimo)
  - Mejor: Q, K, V, O y también FFN
  - LLaMA: también up/gate/down de SwiGLU FFN
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class LoRALayer(nn.Module):
    """
    Implementación de LoRA sobre una capa lineal.
    W' x = (W + α/r · BA) x = Wx + α/r · B(Ax)
    """
    def __init__(self, in_features, out_features, rank=4, alpha=None, dropout=0.0):
        super().__init__()
        self.rank     = rank
        self.alpha    = alpha or rank   # escala = alpha/rank
        self.scaling  = self.alpha / rank

        # Pesos originales (congelados durante fine-tuning)
        self.W = nn.Linear(in_features, out_features, bias=False)
        self.W.weight.requires_grad_(False)

        # Adaptadores LoRA: A (in→rank) y B (rank→out)
        self.lora_A = nn.Linear(in_features, rank, bias=False)
        self.lora_B = nn.Linear(rank, out_features, bias=False)
        self.dropout = nn.Dropout(dropout) if dropout > 0 else nn.Identity()

        # Inicialización: A ~ N(0, σ²), B = 0
        nn.init.kaiming_uniform_(self.lora_A.weight, a=math.sqrt(5))
        nn.init.zeros_(self.lora_B.weight)

    def forward(self, x):
        # Camino base (congelado) + adaptador LoRA (entrenable)
        return self.W(x) + self.scaling * self.lora_B(self.dropout(self.lora_A(x)))

    def merge_weights(self):
        """
        Fusionar LoRA en el modelo base para inferencia sin overhead.
        W' = W + α/r · BA
        """
        delta_W = self.scaling * self.lora_B.weight @ self.lora_A.weight
        self.W.weight.data += delta_W
        self.W.weight.requires_grad_(True)
        # Eliminar adaptadores después de fusionar
        del self.lora_A, self.lora_B
        self.forward = self.W.forward  # reemplazar forward

# Análisis de parámetros según rank
print("Parámetros LoRA según rank (capa 768→768):")
print(f"{'rank':>6} {'Params LoRA':>12} {'% de W':>10}")
d = 768
W_params = d * d
for r in [1, 4, 8, 16, 32, 64]:
    lora_params = 2 * d * r  # A + B
    print(f"{r:>6} {lora_params:>12,d} {lora_params/W_params:>9.2%}")

# Demostración
torch.manual_seed(42)
d_in, d_out, rank = 64, 64, 4

lora = LoRALayer(d_in, d_out, rank=rank)
x    = torch.randn(4, 10, d_in)

# Al inicio: output = W(x) (BA = 0)
out_init = lora(x)
out_base = lora.W(x)
print(f"\nAl inicio (B=0): |out_lora - out_base| = {(out_init - out_base).abs().max():.8f}")

# Después de entrenar: B ya no es cero
nn.init.normal_(lora.lora_B.weight, std=0.01)
out_trained = lora(x)
print(f"Después de entrenar: |out_lora - out_base| = {(out_trained - out_base).abs().max():.6f}")

params_totales    = sum(p.numel() for p in lora.parameters())
params_entrenables = sum(p.numel() for p in lora.parameters() if p.requires_grad)
print(f"\nParámetros totales: {params_totales:,d}")
print(f"Parámetros entrenables (LoRA): {params_entrenables:,d} ({params_entrenables/params_totales:.1%})")
```

---

## 3. LoRA en un Transformer Completo

```python
# seccion_03_lora_transformer.py
"""
Aplicar LoRA a las capas de atención de un Transformer.
Reemplazar las proyecciones Q, K, V, O con versiones LoRA.
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class LoRALinear(nn.Linear):
    """Extensión de nn.Linear con adaptador LoRA."""
    def __init__(self, in_f, out_f, rank=4, alpha=None, **kwargs):
        super().__init__(in_f, out_f, **kwargs)
        self.rank    = rank
        self.scaling = (alpha or rank) / rank
        # Congelar peso base
        self.weight.requires_grad_(False)
        if self.bias is not None:
            self.bias.requires_grad_(False)
        # Adaptadores
        self.lora_A = nn.Parameter(torch.empty(rank, in_f))
        self.lora_B = nn.Parameter(torch.zeros(out_f, rank))
        nn.init.kaiming_uniform_(self.lora_A, a=math.sqrt(5))

    def forward(self, x):
        return F.linear(x, self.weight, self.bias) + \
               self.scaling * F.linear(F.linear(x, self.lora_A), self.lora_B)

def aplicar_lora(model, rank=4, alpha=None, target_modules=None):
    """
    Reemplaza capas lineales del modelo con versiones LoRA.
    target_modules: lista de sufijos de nombres de módulos a modificar.
    """
    if target_modules is None:
        target_modules = ["q_proj", "v_proj"]  # mínimo recomendado

    for name, module in model.named_modules():
        if any(name.endswith(t) for t in target_modules):
            # Obtener la capa padre y el nombre del atributo
            *parent_path, attr = name.split(".")
            parent = model
            for p in parent_path:
                parent = getattr(parent, p)
            # Reemplazar con LoRA
            old = getattr(parent, attr)
            new = LoRALinear(old.in_features, old.out_features,
                             rank=rank, alpha=alpha,
                             bias=old.bias is not None)
            new.weight.data = old.weight.data.clone()
            if old.bias is not None:
                new.bias.data = old.bias.data.clone()
            setattr(parent, attr, new)

    return model

# Verificar en un modelo pequeño
class SimpleTransformer(nn.Module):
    def __init__(self, d=64, n_heads=4):
        super().__init__()
        self.q_proj = nn.Linear(d, d, bias=False)
        self.k_proj = nn.Linear(d, d, bias=False)
        self.v_proj = nn.Linear(d, d, bias=False)
        self.o_proj = nn.Linear(d, d, bias=False)
        self.ffn    = nn.Sequential(nn.Linear(d, d*4), nn.GELU(), nn.Linear(d*4, d))

    def forward(self, x):
        q = self.q_proj(x); k = self.k_proj(x); v = self.v_proj(x)
        att = F.softmax(q @ k.transpose(-2,-1) / 8, -1) @ v
        return self.ffn(self.o_proj(att) + x)

model = SimpleTransformer()
p_antes = sum(p.numel() for p in model.parameters() if p.requires_grad)

model_lora = aplicar_lora(model, rank=4, target_modules=["q_proj", "v_proj"])
p_despues  = sum(p.numel() for p in model_lora.parameters() if p.requires_grad)

print(f"Parámetros entrenables antes de LoRA: {p_antes:,d}")
print(f"Parámetros entrenables con LoRA:      {p_despues:,d}")
print(f"Reducción: {1 - p_despues/p_antes:.1%}")

# Verificar forward
x = torch.randn(2, 10, 64)
out = model_lora(x)
print(f"Output shape: {out.shape}")
```

---

## 4. QLoRA — Quantization + LoRA

```python
# seccion_04_qlora.py
"""
QLoRA (Dettmers et al. 2023):
  = Quantización NF4 del modelo base + LoRA entrenable

Pasos:
  1. Cargar modelo en NF4 (4 bits): 75% menos memoria que fp32
  2. Aplicar LoRA a las capas de atención (en fp16/bf16)
  3. Entrenar solo los adaptadores LoRA

Innovaciones técnicas de QLoRA:
  1. NF4 (Normal Float 4): quantización óptima para pesos gaussianos
     Los pesos LLM siguen N(0,σ) → NF4 tiene bins adaptativos
  2. Double Quantization: quantizar las constantes de quantización
     Ahorra ~0.37 bits adicionales por parámetro
  3. Paged Optimizers: spill de optimizer state a CPU RAM si se llena GPU

Ahorro de memoria:
  LLaMA-7B full precision:  28 GB
  LLaMA-7B int8:            8 GB
  LLaMA-7B nf4 (QLoRA):    ~4 GB  ← cabe en GPU de consumo
"""
import torch
import torch.nn as nn
import numpy as np

# Quantización simplificada para ilustrar NF4
def quantizar_nf4(weight_tensor):
    """
    Simulación de quantización NF4 (Normal Float 4).
    NF4 tiene 16 niveles (4 bits) distribuidos según N(0,1).
    """
    # Niveles NF4 (precomputados según la distribución normal)
    nf4_levels = torch.tensor([
        -1.0, -0.6961, -0.5251, -0.3949, -0.2844, -0.1848,
         -0.0910, 0.0, 0.0796, 0.1609, 0.2461, 0.3379,
         0.4407, 0.5626, 0.7230, 1.0
    ])

    # Normalizar al rango [-1, 1]
    absmax    = weight_tensor.abs().max()
    w_norm    = weight_tensor / (absmax + 1e-8)

    # Quantizar: encontrar el nivel más cercano
    dists     = (w_norm.unsqueeze(-1) - nf4_levels).abs()
    q_indices = dists.argmin(dim=-1)   # (shape)  int de 0 a 15 → 4 bits

    # Dequantizar
    w_deq     = nf4_levels[q_indices] * absmax

    return q_indices, w_deq, absmax

# Análisis del error de quantización
torch.manual_seed(42)
d = 128
W = torch.randn(d, d)   # pesos simulados de LLM

_, W_nf4, _ = quantizar_nf4(W)

# Comparar con cuantización uniforme (int4 simple)
W_min, W_max = W.min(), W.max()
W_int4 = torch.round((W - W_min) / (W_max - W_min) * 15) / 15 * (W_max - W_min) + W_min

error_nf4  = (W - W_nf4).pow(2).mean().sqrt().item()
error_int4 = (W - W_int4).pow(2).mean().sqrt().item()

print("Comparativa de quantización a 4 bits:")
print(f"  Error RMS Int4 uniforme: {error_int4:.6f}")
print(f"  Error RMS NF4:           {error_nf4:.6f}")
print(f"  Mejora NF4:              {(1 - error_nf4/error_int4):.1%} menos error")

# Ahorro de memoria
def memoria_modelo(n_params, dtype):
    bytes_por_param = {"float32": 4, "float16": 2, "bfloat16": 2, "int8": 1, "nf4": 0.5}
    return n_params * bytes_por_param.get(dtype, 4) / 1e9

n_params = 7e9  # LLaMA-7B
print(f"\nMemoria LLaMA-7B según tipo:")
for dt in ["float32", "float16", "int8", "nf4"]:
    print(f"  {dt:<12}: {memoria_modelo(n_params, dt):.2f} GB")

# Código QLoRA con bitsandbytes + peft
codigo_qlora = '''
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, TaskType

# Configuración NF4 (QLoRA)
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",          # Normal Float 4
    bnb_4bit_compute_dtype=torch.bfloat16,  # Cómputo en bf16
    bnb_4bit_use_double_quant=True,     # Double quantization
)

# Cargar modelo cuantizado
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Meta-Llama-3-8B",
    quantization_config=bnb_config,
    device_map="auto",
)

# Configurar LoRA
lora_config = LoraConfig(
    r=16,                    # rank
    lora_alpha=32,           # escala = alpha/r = 2.0
    target_modules=[         # capas donde aplicar LoRA
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",   # FFN
    ],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 41,943,040 || all params: 8,030,261,248 || trainable%: 0.52%
'''
print("\nCódigo QLoRA con bitsandbytes + peft:")
print(codigo_qlora)
```

---

## 5. Adapters, Prefix Tuning y Prompt Tuning

```python
# seccion_05_otras_peft.py
"""
Otras técnicas PEFT: Adapters, Prefix Tuning, Prompt Tuning.

ADAPTERS (Houlsby et al. 2019):
  Insertar pequeñas redes (down → nonlinear → up) después de FFN y atención.
  down: d_model → r (cuello de botella)
  up:   r → d_model
  Residual: x + Adapter(x)

PREFIX TUNING (Li & Liang 2021):
  Prepend K "tokens virtuales" (vectores aprendibles) al K y V de cada capa.
  El modelo procesa [prefix_K; K] y [prefix_V; V] en la atención.
  Solo se aprenden los embeddings del prefijo.

PROMPT TUNING (Lester et al. 2021):
  Versión más simple: solo prepend tokens virtuales al embedding de entrada.
  Eficiente para modelos muy grandes (T5-11B: 0.001% parámetros).
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

# Adapter Layer
class AdapterLayer(nn.Module):
    """
    Adapter: down-project → activación → up-project, con residual.
    """
    def __init__(self, d_model, adapter_dim, dropout=0.1):
        super().__init__()
        self.down = nn.Linear(d_model, adapter_dim)
        self.up   = nn.Linear(adapter_dim, d_model)
        self.act  = nn.GELU()
        self.drop = nn.Dropout(dropout)
        # Inicializar up en cero para estabilidad inicial
        nn.init.zeros_(self.up.weight)
        nn.init.zeros_(self.up.bias)

    def forward(self, x):
        return x + self.drop(self.up(self.act(self.down(x))))

# Prefix Tuning
class PrefixTuning(nn.Module):
    """
    Prefix: K vectores aprendibles prepended a K y V de cada capa de atención.
    """
    def __init__(self, n_heads, d_k, prefix_len, n_layers):
        super().__init__()
        # Prefijos para K y V de cada capa
        self.prefix_k = nn.Parameter(torch.randn(n_layers, prefix_len, n_heads, d_k) * 0.01)
        self.prefix_v = nn.Parameter(torch.randn(n_layers, prefix_len, n_heads, d_k) * 0.01)

    def get_prefix(self, layer_idx, batch_size):
        """Expande el prefijo al batch size actual."""
        pk = self.prefix_k[layer_idx].unsqueeze(0).expand(batch_size, -1, -1, -1)
        pv = self.prefix_v[layer_idx].unsqueeze(0).expand(batch_size, -1, -1, -1)
        return pk, pv

# Prompt Tuning
class PromptTuning(nn.Module):
    """
    Prompt soft: tokens virtuales en el espacio de embedding.
    """
    def __init__(self, prompt_len, d_model, backbone_embed):
        super().__init__()
        # Soft prompts: vectores aprendibles en el espacio de embedding
        self.soft_prompt = nn.Parameter(torch.randn(prompt_len, d_model) * 0.01)
        self.backbone_embed = backbone_embed  # embedding del modelo base (congelado)

    def forward(self, input_ids):
        B, T = input_ids.shape
        tok_emb    = self.backbone_embed(input_ids)            # (B, T, d)
        soft_emb   = self.soft_prompt.unsqueeze(0).expand(B, -1, -1)  # (B, P, d)
        return torch.cat([soft_emb, tok_emb], dim=1)           # (B, P+T, d)

# Comparativa de parámetros entrenables
d, n_layers, r_adapter, r_lora = 768, 12, 32, 16
prefix_len, prompt_len          = 10, 20

params_full    = n_layers * (4 * d**2 + 2 * d * 4*d)      # atención + FFN
params_adapter = n_layers * 2 * 2 * (d * r_adapter)       # down + up en 2 bloques
params_lora    = n_layers * 4 * 2 * (d * r_lora)          # Q, K, V, O: A + B
params_prefix  = n_layers * 2 * prefix_len * d            # K y V por capa
params_prompt  = prompt_len * d                            # solo el prompt

print("Parámetros entrenables (modelo 768-dim, 12 capas):")
print(f"{'Método':<20} {'Params':>12} {'% del full':>12}")
full = params_full
for nombre, p in [("Full fine-tuning", full), ("Adapters (r=32)", params_adapter),
                   ("LoRA (rank=16)", params_lora), ("Prefix (P=10)", params_prefix),
                   ("Prompt (P=20)", params_prompt)]:
    print(f"{nombre:<20} {p:>12,d} {p/full:>11.2%}")
```

---

## 6. Uso Práctico con PEFT (HuggingFace)

```python
# seccion_06_peft_practica.py
"""
La librería `peft` de HuggingFace simplifica la aplicación de PEFT.
Soporta: LoRA, AdaLoRA, IA3, Prompt Tuning, Prefix Tuning, LoftQ, etc.
"""

# Código de referencia completo para LoRA con peft
codigo_lora_peft = '''
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import (
    LoraConfig, get_peft_model, TaskType,
    PeftModel, PeftConfig   # para cargar modelo guardado
)
import torch

# 1. Cargar modelo base
model_id  = "meta-llama/Meta-Llama-3-8B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model     = AutoModelForCausalLM.from_pretrained(
    model_id, torch_dtype=torch.bfloat16
)

# 2. Configurar LoRA
config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
    inference_mode=False,
)

# 3. Aplicar LoRA
model = get_peft_model(model, config)
model.print_trainable_parameters()

# 4. Entrenar (con tu trainer favorito)
# ... (ver M09 para SFT con TRL)

# 5. Guardar solo los adaptadores (pequeños!)
model.save_pretrained("./lora_adapters")   # guarda solo ~60 MB
tokenizer.save_pretrained("./lora_adapters")

# 6. Cargar para inferencia
config_loaded = PeftConfig.from_pretrained("./lora_adapters")
base_model    = AutoModelForCausalLM.from_pretrained(
    config_loaded.base_model_name_or_path, torch_dtype=torch.bfloat16
)
model_loaded = PeftModel.from_pretrained(base_model, "./lora_adapters")
model_loaded.eval()

# 7. (Opcional) Fusionar para inferencia sin overhead
model_merged = model_loaded.merge_and_unload()
# Ahora es un modelo estándar, sin overhead de LoRA
'''

print("Flujo completo LoRA con peft:")
print(codigo_lora_peft)

# Demo: entrenar LoRA en tarea clasificación de sentimiento
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class LoRAMLP(nn.Module):
    """MLP con LoRA para demo."""
    def __init__(self, d_in=64, d_hidden=128, d_out=3, rank=4):
        super().__init__()
        # Capa base congelada
        self.linear1 = nn.Linear(d_in, d_hidden)
        self.linear2 = nn.Linear(d_hidden, d_out)
        self.linear1.weight.requires_grad_(False)
        self.linear2.weight.requires_grad_(False)

        # Adaptadores LoRA para linear1
        self.A = nn.Parameter(torch.empty(rank, d_in))
        self.B = nn.Parameter(torch.zeros(d_hidden, rank))
        self.scaling = 1.0 / rank
        nn.init.kaiming_uniform_(self.A, a=math.sqrt(5))

    def forward(self, x):
        h = F.relu(self.linear1(x) + self.scaling * F.linear(F.linear(x, self.A), self.B))
        return self.linear2(h)

torch.manual_seed(42)
model = LoRAMLP(d_in=16, d_hidden=32, d_out=3, rank=2)
opt   = torch.optim.AdamW([p for p in model.parameters() if p.requires_grad], lr=1e-2)

# Dataset sintético: 3 clases separables
X = torch.cat([
    torch.randn(50, 16) + torch.tensor([1., 0.]*8),
    torch.randn(50, 16) + torch.tensor([0., 1.]*8),
    torch.randn(50, 16) + torch.tensor([-1., 0.]*8),
])
y = torch.tensor([0]*50 + [1]*50 + [2]*50)

for epoch in range(200):
    logits = model(X)
    loss   = F.cross_entropy(logits, y)
    opt.zero_grad(); loss.backward(); opt.step()

with torch.no_grad():
    acc = (model(X).argmax(1) == y).float().mean()
print(f"\nLoRA MLP demo — Accuracy: {acc:.1%}")

p_ent = sum(p.numel() for p in model.parameters() if p.requires_grad)
p_tot = sum(p.numel() for p in model.parameters())
print(f"Params entrenables: {p_ent:,d} / {p_tot:,d} ({p_ent/p_tot:.1%})")
```

---

## Checkpoint M10

```python
# checkpoint_m10.py
"""
Tests de verificación para Módulo 10 — LoRA, QLoRA, PEFT.
Ejecutar con: python checkpoint_m10.py
"""
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

def test_1_lora_sin_efecto_inicial():
    """LoRA con B=0 no modifica la salida del modelo base."""
    d, r = 32, 4
    W = nn.Linear(d, d, bias=False)
    A = nn.Parameter(torch.randn(r, d))
    B = nn.Parameter(torch.zeros(d, r))  # inicializar en 0

    x   = torch.randn(4, d)
    out_base = W(x)
    out_lora = W(x) + F.linear(F.linear(x, A), B)
    assert torch.allclose(out_base, out_lora, atol=1e-6)
    print("✓ Test 1: LoRA con B=0 no modifica la salida")

def test_2_lora_params_reduccion():
    """LoRA rank=4 tiene menos parámetros que el peso original."""
    d, r = 64, 4
    W_params   = d * d          # 4096
    LoRA_params = 2 * d * r     # A + B = 512
    assert LoRA_params < W_params
    print(f"✓ Test 2: LoRA rank={r}: {LoRA_params} params vs W: {W_params} ({LoRA_params/W_params:.1%})")

def test_3_lora_gradiente_solo_ab():
    """Solo los parámetros A y B de LoRA tienen gradiente."""
    d, r = 32, 4

    class LoRALayer(nn.Module):
        def __init__(self):
            super().__init__()
            self.W = nn.Linear(d, d, bias=False)
            self.W.weight.requires_grad_(False)
            self.A = nn.Parameter(torch.randn(r, d))
            self.B = nn.Parameter(torch.zeros(d, r))
        def forward(self, x):
            return self.W(x) + F.linear(F.linear(x, self.A), self.B)

    layer = LoRALayer()
    x     = torch.randn(4, d)
    loss  = layer(x).sum()
    loss.backward()

    assert layer.W.weight.grad is None, "W no debe tener gradiente"
    assert layer.A.grad is not None,    "A debe tener gradiente"
    assert layer.B.grad is not None,    "B debe tener gradiente"
    print("✓ Test 3: Solo A y B tienen gradiente, W está congelado")

def test_4_adapter_residual():
    """Adapter con up=0 es idéntico a identity (conexión residual)."""
    d, r = 32, 4
    class Adapter(nn.Module):
        def __init__(self):
            super().__init__()
            self.down = nn.Linear(d, r)
            self.up   = nn.Linear(r, d)
            nn.init.zeros_(self.up.weight)
            nn.init.zeros_(self.up.bias)
        def forward(self, x):
            return x + self.up(torch.relu(self.down(x)))

    adapter = Adapter()
    x       = torch.randn(4, d)
    out     = adapter(x)
    assert torch.allclose(x, out, atol=1e-6), "Adapter con up=0 debe ser identity"
    print("✓ Test 4: Adapter con up=0 es identity (residual correcto)")

def test_5_qlora_quantizacion_reduce_memoria():
    """Quantización a 4 bits reduce la memoria por 4x vs float32."""
    n_params = 1_000_000
    mem_fp32 = n_params * 4    # bytes
    mem_nf4  = n_params * 0.5  # bytes (4 bits)
    ratio    = mem_fp32 / mem_nf4
    assert ratio == 8.0  # 8x reducción
    print(f"✓ Test 5: NF4 reduce memoria {ratio:.0f}x vs fp32 ({mem_nf4/1e6:.1f} MB vs {mem_fp32/1e6:.1f} MB por 1M params)")

def test_6_merge_lora():
    """merge_weights produce el mismo resultado que la aplicación de LoRA."""
    d, r = 16, 2
    W = nn.Linear(d, d, bias=False)
    A = nn.Parameter(torch.randn(r, d))
    B = nn.Parameter(torch.randn(d, r))
    scaling = 1.0

    x = torch.randn(4, d)
    # Con LoRA separado
    out_lora = W(x) + scaling * F.linear(F.linear(x, A), B)
    # Merge: W_new = W + BA
    W_merged = W.weight.data + scaling * B @ A
    out_merged = F.linear(x, W_merged)
    assert torch.allclose(out_lora, out_merged, atol=1e-5)
    print("✓ Test 6: Merge de LoRA produce resultado equivalente")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M10 — LoRA, QLoRA, PEFT")
    print("=" * 50)
    test_1_lora_sin_efecto_inicial()
    test_2_lora_params_reduccion()
    test_3_lora_gradiente_solo_ab()
    test_4_adapter_residual()
    test_5_qlora_quantizacion_reduce_memoria()
    test_6_merge_lora()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M10 completado")
```

---

## Resumen del Módulo

| Técnica | Params entrenables | Innovación | Ideal para |
|---|---|---|---|
| **LoRA** | 0.1-5% | Matrices rango bajo en Q,K,V,O | General purpose |
| **QLoRA** | 0.1-5% | LoRA + NF4 quantización | GPUs de consumo (<24GB) |
| **Adapters** | 0.5-3% | Módulos bottleneck entre capas | Multitarea |
| **Prefix Tuning** | 0.1% | Tokens virtuales en K,V | Pocas muestras |
| **Prompt Tuning** | <0.01% | Tokens virtuales en input | Modelos muy grandes |

**Regla práctica:**
- Recurso limitado (≤8 GB): QLoRA rank=8 con target Q,V
- Recursos medios (16 GB): LoRA rank=16 con target Q,K,V,O,FFN
- Máxima calidad: full fine-tuning o LoRA rank=64+

**Próximo módulo:** M11 — Embeddings y Representaciones Semánticas
