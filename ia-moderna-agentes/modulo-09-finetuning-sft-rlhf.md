# Módulo 09 — Fine-tuning Completo: SFT y RLHF

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender el pipeline completo de alignment de LLMs: Supervised Fine-Tuning (SFT), Reward Model y RLHF (PPO), y DPO como alternativa más simple.
**Herramientas:** PyTorch 2.x, transformers, trl (HuggingFace)
**Prerequisito:** M06 (Arquitectura LLMs), M07 (Pre-entrenamiento)

---

## 1. El Pipeline de Alignment

```python
# seccion_01_pipeline_alignment.py
"""
Los LLMs preentrenados saben mucho pero no "ayudan" por defecto.
El alignment los convierte en asistentes útiles y seguros.

PIPELINE COMPLETO (InstructGPT, ChatGPT, Claude):

Fase 1 — Pre-entrenamiento (M07):
  Corpus masivo → Causal LM → Base model
  El modelo sabe completar texto, no seguir instrucciones

Fase 2 — SFT (Supervised Fine-Tuning):
  Demostraciones humanas de alta calidad
  Formato: (prompt_usuario, respuesta_ideal)
  El modelo aprende A SEGUIR INSTRUCCIONES

Fase 3 — Reward Model (RM):
  Comparaciones humanas: dado (prompt, respuesta_A, respuesta_B)
  Humano elige cuál es mejor
  RM aprende a predecir preferencias humanas

Fase 4 — RLHF con PPO:
  Usar el RM como función de recompensa
  Optimizar el SFT con RL para maximizar recompensa
  + KL penalty para no alejarse demasiado del SFT

Fase 4' — DPO (alternativa a RLHF):
  Entrenamiento directo con pares de preferencia
  Sin RM separado, sin RL → mucho más simple
"""

print("Pipeline de alignment de LLMs:")
etapas = [
    ("1. Pre-entrenamiento",  "Billones de tokens",       "Modelo base"),
    ("2. SFT",                "~100K instrucciones",      "Modelo instruction-tuned"),
    ("3. RM Training",        "~50K comparaciones",       "Reward Model"),
    ("4. RLHF (PPO)",        "RM feedback",              "Modelo aligned (RLHF)"),
    ("4'. DPO",              "Pares de preferencia",     "Modelo aligned (DPO)"),
]
print(f"\n{'Etapa':<25} {'Datos':<25} {'Resultado'}")
print("-" * 70)
for etapa, datos, resultado in etapas:
    print(f"{etapa:<25} {datos:<25} {resultado}")
```

---

## 2. Supervised Fine-Tuning (SFT)

```python
# seccion_02_sft.py
"""
SFT: fine-tuning estándar en pares (instrucción, respuesta).
Los datos de SFT son fundamentales: "RLHF about the data" (Karpathy).

Formato de datos:
  OpenAI: {"prompt": ..., "completion": ...}
  Alpaca: {"instruction": ..., "input": ..., "output": ...}
  ChatML: [{"role": "user", "content": ...}, {"role": "assistant", "content": ...}]

Loss: solo en los tokens de la RESPUESTA (no en el prompt)
  → Máscara de loss: -100 en tokens del prompt
  → Modelo aprende a generar la respuesta correcta dado el contexto
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

# Dataset SFT sintético
DATOS_SFT = [
    {
        "instruccion": "Traduce al inglés: 'La inteligencia artificial es fascinante'",
        "respuesta":   "Artificial intelligence is fascinating."
    },
    {
        "instruccion": "¿Cuál es la capital de Francia?",
        "respuesta":   "La capital de Francia es París."
    },
    {
        "instruccion": "Escribe un haiku sobre la primavera.",
        "respuesta":   "Flores rosadas\ndespertan con la brisa\nvida que renace."
    },
    {
        "instruccion": "Explica qué es una función en Python con un ejemplo.",
        "respuesta":   "Una función en Python es un bloque de código reutilizable.\nEjemplo:\n```python\ndef sumar(a, b):\n    return a + b\nresultado = sumar(3, 4)  # 7\n```"
    },
]

def formatear_prompt_sft(instruccion: str, respuesta: str = None) -> str:
    """
    Formato ChatML (usado por muchos modelos modernos).
    En entrenamiento: incluir respuesta.
    En inferencia: solo el prompt.
    """
    prompt = f"<|im_start|>user\n{instruccion}<|im_end|>\n<|im_start|>assistant\n"
    if respuesta:
        prompt += f"{respuesta}<|im_end|>"
    return prompt

print("Ejemplos de formato SFT (ChatML):")
for d in DATOS_SFT[:2]:
    print("\n" + "="*50)
    print(formatear_prompt_sft(d["instruccion"], d["respuesta"]))

# Loss de SFT: solo en los tokens de la respuesta
def sft_loss(logits, labels, prompt_len):
    """
    logits: (batch, T, vocab)
    labels: (batch, T) - tokens objetivo
    prompt_len: longitud del prompt (tokens con loss enmascarado)
    """
    # Enmascarar tokens del prompt (-100 es ignorado por CrossEntropy)
    labels_masked = labels.clone()
    labels_masked[:, :prompt_len] = -100

    # Shift: logits[t] predice labels[t+1]
    shift_logits = logits[:, :-1, :].contiguous()
    shift_labels = labels_masked[:, 1:].contiguous()

    loss = F.cross_entropy(
        shift_logits.view(-1, shift_logits.size(-1)),
        shift_labels.view(-1),
        ignore_index=-100
    )
    return loss

# Demo del loss
vocab, T, d = 100, 20, 32
prompt_len  = 8  # primeros 8 tokens son el prompt
logits      = torch.randn(2, T, vocab)
labels      = torch.randint(0, vocab, (2, T))
loss        = sft_loss(logits, labels, prompt_len)
print(f"\nSFT Loss (solo en respuesta, T={T}, prompt_len={prompt_len}): {loss.item():.4f}")

# Código de referencia para SFT con transformers
codigo_sft_hf = '''
# Fine-tuning con HuggingFace Transformers + TRL
from transformers import AutoModelForCausalLM, AutoTokenizer
from trl import SFTTrainer, SFTConfig
from datasets import Dataset

model_id = "meta-llama/Meta-Llama-3-8B"
model     = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype="bfloat16")
tokenizer = AutoTokenizer.from_pretrained(model_id)

# Datos en formato ChatML
dataset = Dataset.from_list([
    {"messages": [
        {"role": "user",      "content": "¿Qué es Python?"},
        {"role": "assistant", "content": "Python es un lenguaje de programación..."},
    ]},
    # ... más ejemplos
])

config = SFTConfig(
    output_dir="./sft_output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,       # batch efectivo = 32
    learning_rate=2e-5,
    lr_scheduler_type="cosine",
    warmup_ratio=0.05,
    bf16=True,                           # bfloat16 si hay GPU
    logging_steps=10,
    save_steps=500,
    max_seq_length=2048,
)

trainer = SFTTrainer(
    model=model,
    args=config,
    train_dataset=dataset,
    processing_class=tokenizer,
)
trainer.train()
'''
print("\nCódigo SFT con TRL:")
print(codigo_sft_hf)
```

---

## 3. Reward Model

```python
# seccion_03_reward_model.py
"""
El Reward Model (RM) aprende a puntuar respuestas como lo haría un humano.

Entrenamiento:
  Datos: (prompt, respuesta_elegida, respuesta_rechazada)
  El RM aprende: score(elegida) > score(rechazada)
  Pérdida Bradley-Terry:
    L = -log(σ(score(elegida) - score(rechazada)))

Arquitectura: LLM con una capa linear al final → scalar score
  (en lugar de vocab_size → 1)
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

class RewardModel(nn.Module):
    """
    RM simplificado: backbone LLM + regression head.
    En práctica: LLaMA/GPT-2/etc. con la última capa reemplazada por Linear(d, 1).
    """
    def __init__(self, d_model=128, n_heads=4, n_layers=2, vocab_size=500):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, d_model)
        self.pe    = nn.Embedding(256, d_model)
        layer      = nn.TransformerEncoderLayer(d_model, n_heads, d_model*4,
                                                batch_first=True, norm_first=True)
        self.backbone = nn.TransformerEncoder(layer, n_layers)
        # Regression head: predice un scalar de recompensa
        self.value_head = nn.Linear(d_model, 1)

    def forward(self, input_ids):
        B, T = input_ids.shape
        x = self.embed(input_ids) + self.pe(torch.arange(T, device=input_ids.device))
        # Usar máscara causal (igual que el LLM base)
        x = self.backbone(x)
        # Tomar el último token como representación de la secuencia completa
        score = self.value_head(x[:, -1, :]).squeeze(-1)  # (batch,)
        return score

def perdida_bradley_terry(score_elegida, score_rechazada):
    """
    Pérdida de preferencia Bradley-Terry.
    Maximiza P(elegida > rechazada).
    """
    return -F.logsigmoid(score_elegida - score_rechazada).mean()

# Dataset de preferencias simulado
PREFERENCIAS = [
    {
        "prompt":    "¿Cuál es la capital de España?",
        "elegida":   "La capital de España es Madrid, ciudad conocida por el Prado y el Real Madrid.",
        "rechazada": "España tiene capital.",
    },
    {
        "prompt":    "Explica qué es recursión en programación.",
        "elegida":   "La recursión es cuando una función se llama a sí misma. Ejemplo: factorial(n) = n × factorial(n-1).",
        "rechazada": "Es complicado.",
    },
]

# Tokenización simplificada (carácter-nivel)
def tokenizar(texto, max_len=64):
    ids = [ord(c) % 500 for c in texto[:max_len]]
    ids += [0] * (max_len - len(ids))
    return torch.tensor(ids)

# Entrenamiento del RM
torch.manual_seed(42)
rm    = RewardModel()
opt_rm = torch.optim.Adam(rm.parameters(), lr=1e-3)

for epoch in range(100):
    total_loss = 0
    for pref in PREFERENCIAS:
        ids_e = tokenizar(pref["prompt"] + " " + pref["elegida"]).unsqueeze(0)
        ids_r = tokenizar(pref["prompt"] + " " + pref["rechazada"]).unsqueeze(0)
        score_e = rm(ids_e)
        score_r = rm(ids_r)
        loss = perdida_bradley_terry(score_e, score_r)
        opt_rm.zero_grad(); loss.backward(); opt_rm.step()
        total_loss += loss.item()
    if epoch % 25 == 24:
        print(f"Epoch {epoch+1:3d} | RM loss: {total_loss:.4f}")

# Verificar que el RM puntúa mejor las respuestas elegidas
print("\nVerificación del RM:")
for pref in PREFERENCIAS:
    with torch.no_grad():
        s_e = rm(tokenizar(pref["prompt"] + " " + pref["elegida"]).unsqueeze(0)).item()
        s_r = rm(tokenizar(pref["prompt"] + " " + pref["rechazada"]).unsqueeze(0)).item()
    mejor = "✓" if s_e > s_r else "✗"
    print(f"  {mejor} Elegida={s_e:.3f}, Rechazada={s_r:.3f}: '{pref['prompt'][:40]}'")
```

---

## 4. RLHF con PPO (Conceptual)

```python
# seccion_04_rlhf_ppo.py
"""
RLHF con PPO (Proximal Policy Optimization):

El LLM fine-tuneado (SFT) actúa como la "política" π_θ.
Genera respuestas, recibe recompensa del RM, actualiza parámetros.

Función objetivo PPO:
  L_PPO = E[min(r_t·A_t, clip(r_t, 1-ε, 1+ε)·A_t)]
  - r_t = π_θ(a_t|s_t) / π_ref(a_t|s_t)  (ratio de probabilidades)
  - A_t: ventaja (reward - baseline)
  - Clipping: limita el cambio para estabilidad

KL penalty (crucial para evitar "reward hacking"):
  L_total = L_PPO - β·KL(π_θ || π_ref)
  - π_ref: modelo SFT congelado (referencia)
  - β: coeficiente de penalización KL
  - Evita que el modelo genere texto extraño que confunde al RM

En práctica (código de referencia con trl):
"""

codigo_ppo_trl = '''
from transformers import AutoModelForCausalLM, AutoTokenizer
from trl import PPOTrainer, PPOConfig, AutoModelForCausalLMWithValueHead
import torch

# Modelo SFT como punto de partida
model = AutoModelForCausalLMWithValueHead.from_pretrained("path/to/sft_model")
ref_model = AutoModelForCausalLMWithValueHead.from_pretrained("path/to/sft_model")
reward_model = ... # cargar RM entrenado
tokenizer = AutoTokenizer.from_pretrained("path/to/sft_model")

config = PPOConfig(
    model_name="mi_modelo",
    learning_rate=1.41e-5,
    log_with="wandb",
    batch_size=256,
    mini_batch_size=64,
    gradient_accumulation_steps=1,
    optimize_cuda_cache=True,
    early_stopping=False,
    target_kl=6.0,         # KL máxima permitida
    kl_penalty="kl",       # penalización KL
    init_kl_coef=0.2,      # β inicial
    adap_kl_ctrl=True,     # ajuste adaptativo de β
    ppo_epochs=4,
    ratio_threshold=10.0,  # descartar si ratio muy alto
)

ppo_trainer = PPOTrainer(config, model, ref_model, tokenizer)

# Loop de RLHF
for batch in dataloader:
    prompts = batch["prompt"]
    # 1. Generar respuestas con el modelo actual
    responses = ppo_trainer.generate(prompts, max_new_tokens=200,
                                      do_sample=True, temperature=0.7)
    # 2. Calcular recompensa con el RM
    rewards = reward_model(prompts, responses)
    # 3. Actualizar con PPO
    stats = ppo_trainer.step(prompts, responses, rewards)
'''
print("RLHF con PPO (código referencia):")
print(codigo_ppo_trl)

# Comparativa RLHF vs DPO
print("\nComparativa RLHF vs DPO:")
print(f"{'Aspecto':<25} {'RLHF (PPO)':>20} {'DPO':>20}")
print("-" * 67)
aspectos = [
    ("RM separado",          "Sí",         "No"),
    ("Generación online",    "Sí",         "No"),
    ("Complejidad",          "Alta",       "Baja"),
    ("Estabilidad",          "Variable",   "Más estable"),
    ("Memoria GPU",          "Muy alta",   "Moderada"),
    ("Calidad resultado",    "Alta",       "Alta"),
    ("Pasos de entrenamiento", "3",        "2 (SFT + DPO)"),
]
for asp, rlhf, dpo in aspectos:
    print(f"{asp:<25} {rlhf:>20} {dpo:>20}")
```

---

## 5. DPO — Direct Preference Optimization

```python
# seccion_05_dpo.py
"""
DPO (Rafailov et al. 2023): alternativa simple y efectiva a RLHF.

Insight clave: se puede derivar la solución óptima del RL analíticamente:
  π*(y|x) ∝ π_ref(y|x) · exp(r(x,y)/β)

Esto permite derivar la pérdida DPO que entrena directamente con preferencias:
  L_DPO = -E[log σ(β·log(π_θ(y_w|x)/π_ref(y_w|x))
                 - β·log(π_θ(y_l|x)/π_ref(y_l|x)))]

  y_w = respuesta elegida (winner)
  y_l = respuesta rechazada (loser)
  π_ref = modelo SFT de referencia (congelado)
  β = temperatura (KL strength, típico 0.1-0.5)

Ventajas:
  - No necesita RM separado
  - No usa RL (sin PPO, sin on-policy generation)
  - Mucho más simple de implementar y estable
  - Competitivo o mejor que PPO en muchos benchmarks
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

def dpo_loss(log_probs_policy_w, log_probs_policy_l,
             log_probs_ref_w, log_probs_ref_l, beta=0.1):
    """
    Pérdida DPO.

    log_probs_policy_w: log P_θ(y_w | x) - suma sobre tokens de la respuesta ganadora
    log_probs_policy_l: log P_θ(y_l | x) - suma sobre tokens de la respuesta perdedora
    log_probs_ref_w:    log P_ref(y_w | x)
    log_probs_ref_l:    log P_ref(y_l | x)
    beta:               temperatura KL
    """
    # Ratio log-probabilidades
    log_ratio_w = log_probs_policy_w - log_probs_ref_w  # log(π_θ/π_ref) para ganadora
    log_ratio_l = log_probs_policy_l - log_probs_ref_l  # log(π_θ/π_ref) para perdedora

    # Pérdida DPO
    loss = -F.logsigmoid(beta * (log_ratio_w - log_ratio_l))
    return loss.mean()

def calcular_log_probs(modelo, input_ids, response_mask):
    """
    Calcula log P(respuesta | prompt) sumando sobre los tokens de la respuesta.

    input_ids:      (batch, T)  — prompt + respuesta concatenados
    response_mask:  (batch, T)  — 1 en tokens de respuesta, 0 en prompt
    """
    with torch.no_grad() if modelo.training is False else torch.enable_grad():
        logits = modelo(input_ids)                    # (batch, T, vocab)
    log_probs = F.log_softmax(logits, dim=-1)         # (batch, T, vocab)

    # Seleccionar log prob del token correcto (shift)
    labels = input_ids[:, 1:]                         # (batch, T-1)
    lp     = log_probs[:, :-1, :]                     # (batch, T-1, vocab)
    token_lp = lp.gather(-1, labels.unsqueeze(-1)).squeeze(-1)  # (batch, T-1)

    # Sumar solo sobre tokens de la respuesta
    mask = response_mask[:, 1:]                       # (batch, T-1)
    return (token_lp * mask).sum(-1)                  # (batch,)

# Modelo minimal para demo
class MiniLM(nn.Module):
    def __init__(self, vocab=100, d=32):
        super().__init__()
        self.emb  = nn.Embedding(vocab, d)
        self.gru  = nn.GRU(d, d, batch_first=True)
        self.head = nn.Linear(d, vocab)

    def forward(self, x):
        out, _ = self.gru(self.emb(x))
        return self.head(out)

torch.manual_seed(42)
vocab = 100
policy_model = MiniLM(vocab)
ref_model    = MiniLM(vocab)
# Copiar pesos: ref_model es el SFT congelado
ref_model.load_state_dict(policy_model.state_dict())
for p in ref_model.parameters():
    p.requires_grad_(False)

opt = torch.optim.Adam(policy_model.parameters(), lr=1e-3)

# Entrenamiento DPO simulado
B, T = 4, 16
for step in range(100):
    # Datos de preferencia simulados
    ids_w  = torch.randint(0, vocab, (B, T))   # secuencias ganadoras
    ids_l  = torch.randint(0, vocab, (B, T))   # secuencias perdedoras
    mask_w = torch.cat([torch.zeros(B, T//2), torch.ones(B, T//2)], dim=1)  # respuesta en 2da mitad
    mask_l = mask_w.clone()

    # Log probs política
    lp_w_pol = calcular_log_probs(policy_model, ids_w, mask_w)
    lp_l_pol = calcular_log_probs(policy_model, ids_l, mask_l)
    # Log probs referencia
    lp_w_ref = calcular_log_probs(ref_model, ids_w, mask_w)
    lp_l_ref = calcular_log_probs(ref_model, ids_l, mask_l)

    loss = dpo_loss(lp_w_pol, lp_l_pol, lp_w_ref, lp_l_ref, beta=0.1)
    opt.zero_grad(); loss.backward(); opt.step()

    if step % 25 == 24:
        print(f"Step {step+1:3d} | DPO loss: {loss.item():.4f}")

# Código de referencia DPO con TRL
codigo_dpo_trl = '''
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer
from datasets import Dataset

model     = AutoModelForCausalLM.from_pretrained("path/to/sft_model")
ref_model = AutoModelForCausalLM.from_pretrained("path/to/sft_model")
tokenizer = AutoTokenizer.from_pretrained("path/to/sft_model")

# Dataset de preferencias
dataset = Dataset.from_dict({
    "prompt":   ["¿Cuál es la capital de España?", ...],
    "chosen":   ["La capital de España es Madrid.", ...],
    "rejected": ["No sé.", ...],
})

config = DPOConfig(
    beta=0.1,                          # fuerza de KL
    learning_rate=5e-7,
    num_train_epochs=3,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    bf16=True,
    output_dir="./dpo_output",
)
trainer = DPOTrainer(
    model=model,
    ref_model=ref_model,
    args=config,
    train_dataset=dataset,
    processing_class=tokenizer,
)
trainer.train()
'''
print("\nDPO con TRL:")
print(codigo_dpo_trl)
```

---

## 6. Creación de Datos SFT de Calidad

```python
# seccion_06_datos_sft.py
"""
"La calidad de los datos > cantidad" — lección de Alpaca, LIMA, etc.

LIMA (Zhou et al. 2023): 1000 ejemplos cuidadosamente seleccionados
  superan a modelos con 50K ejemplos de menor calidad.

Fuentes de datos SFT:
  1. Demostraciones humanas: caras pero de alta calidad
  2. Destilación de modelos: GPT-4 genera respuestas (self-instruct)
  3. Self-play / Constitutional AI (Anthropic): el modelo se critica a sí mismo
  4. Synthetic data: generar instrucciones programáticamente

Principios de datos de calidad (LIMA, Dolly, Orca):
  - Diversidad: muchos tipos de tareas, formatos, dominios
  - Calidad: respuestas precisas, completas, bien formateadas
  - Consistencia: tono y estilo consistentes
  - Sin sesgos: representación balanceada
"""

def generar_instrucciones_sinteticas(semillas, transformaciones):
    """
    Self-Instruct (Wang et al. 2023): generar instrucciones automáticamente
    a partir de un conjunto semilla.
    """
    instrucciones = list(semillas)
    for seed in semillas:
        for transform in transformaciones:
            nueva = transform(seed)
            if nueva not in instrucciones:
                instrucciones.append(nueva)
    return instrucciones

# Transformaciones de instrucciones
transformaciones = [
    lambda x: f"Explica brevemente: {x}",
    lambda x: f"Proporciona un ejemplo de: {x}",
    lambda x: f"¿Cuáles son los ventajas y desventajas de: {x}?",
    lambda x: f"Compara {x} con su alternativa principal",
]

semillas = [
    "el aprendizaje supervisado",
    "las redes neuronales recurrentes",
    "el algoritmo de atención",
]

instrucciones = generar_instrucciones_sinteticas(semillas, transformaciones)
print(f"Generadas {len(instrucciones)} instrucciones a partir de {len(semillas)} semillas:")
for i in instrucciones[:8]:
    print(f"  - {i}")

# Proceso Constitutional AI (Anthropic)
def constitutional_ai_revision(respuesta_inicial: str, principios: list) -> str:
    """
    CAI: el modelo critica su propia respuesta usando principios,
    luego genera una versión mejorada.
    """
    critica_prompt = f"""Revisa la siguiente respuesta según estos principios:

Respuesta inicial:
{respuesta_inicial}

Principios a verificar:
{chr(10).join(f'{i+1}. {p}' for i, p in enumerate(principios))}

¿Viola algún principio? ¿Cómo se puede mejorar?"""

    revision_prompt = f"""Basándote en la crítica anterior, reescribe la respuesta
para que cumpla todos los principios, manteniendo la información útil."""

    # En producción: llamar al LLM dos veces
    return "[Respuesta revisada según principios constitucionales]"

principios_cai = [
    "No proporcionar información dañina o peligrosa",
    "Ser honesto sobre las limitaciones del conocimiento",
    "Respetar la privacidad y los datos personales",
    "Promover pensamiento crítico independiente",
]

ejemplo_respuesta = "Aquí tienes cómo hacer X..."
revision = constitutional_ai_revision(ejemplo_respuesta, principios_cai)
print(f"\nConstitutional AI:")
print(f"  Principios aplicados: {len(principios_cai)}")
print(f"  Resultado: {revision}")
```

---

## Checkpoint M09

```python
# checkpoint_m09.py
"""
Tests de verificación para Módulo 09 — Fine-tuning SFT y RLHF.
Ejecutar con: python checkpoint_m09.py
"""
import torch
import torch.nn.functional as F

def test_1_sft_loss_ignora_prompt():
    """SFT loss ignora los tokens del prompt (-100)."""
    vocab, T = 50, 10
    logits  = torch.randn(2, T, vocab)
    labels  = torch.randint(0, vocab, (2, T))
    labels[:, :5] = -100  # primeros 5 tokens son el prompt

    loss = F.cross_entropy(logits[:, :-1].reshape(-1, vocab),
                           labels[:, 1:].reshape(-1), ignore_index=-100)
    assert not torch.isnan(loss) and not torch.isinf(loss)
    print(f"✓ Test 1: SFT loss con máscara: {loss.item():.4f}")

def test_2_reward_model_shape():
    """Reward Model produce un scalar por muestra."""
    import torch.nn as nn
    rm = nn.Sequential(nn.Linear(16, 32), nn.ReLU(), nn.Linear(32, 1))
    x  = torch.randn(4, 16)
    scores = rm(x).squeeze(-1)
    assert scores.shape == (4,), f"Shape: {scores.shape}"
    print(f"✓ Test 2: RM produce scores de shape (4,): {scores.detach().numpy().round(3)}")

def test_3_bradley_terry_loss():
    """Bradley-Terry loss es menor cuando score(ganadora) > score(perdedora)."""
    s_w_bueno = torch.tensor([2.0, 1.5])   # ganadora puntúa más
    s_l_bueno = torch.tensor([0.5, 0.3])
    s_w_malo  = torch.tensor([0.3, 0.5])   # perdedora puntúa más (mal RM)
    s_l_malo  = torch.tensor([2.0, 1.5])

    loss_bueno = -F.logsigmoid(s_w_bueno - s_l_bueno).mean()
    loss_malo  = -F.logsigmoid(s_w_malo  - s_l_malo).mean()
    assert loss_bueno < loss_malo
    print(f"✓ Test 3: BT loss: bueno={loss_bueno:.4f} < malo={loss_malo:.4f}")

def test_4_dpo_loss_formula():
    """DPO loss disminuye cuando el ratio de ganadora > ratio de perdedora."""
    beta = 0.1
    log_ratio_w_bueno = torch.tensor(0.5)   # política favorece ganadora
    log_ratio_l_bueno = torch.tensor(-0.3)
    log_ratio_w_malo  = torch.tensor(-0.5)  # política favorece perdedora
    log_ratio_l_malo  = torch.tensor(0.3)

    loss_bueno = -F.logsigmoid(beta * (log_ratio_w_bueno - log_ratio_l_bueno))
    loss_malo  = -F.logsigmoid(beta * (log_ratio_w_malo  - log_ratio_l_malo))
    assert loss_bueno < loss_malo
    print(f"✓ Test 4: DPO loss: bueno={loss_bueno:.4f} < malo={loss_malo:.4f}")

def test_5_kl_penalty_limita_divergencia():
    """La penalización KL restringe que el modelo se aleje demasiado del referencia."""
    # KL(P||Q) = Σ P(x) log(P(x)/Q(x))
    P = torch.tensor([0.6, 0.3, 0.1])  # política
    Q = torch.tensor([0.5, 0.3, 0.2])  # referencia
    kl = (P * torch.log(P / Q)).sum()
    assert kl >= 0, "KL debe ser >= 0"
    # Con β alto, la penalización domina → modelo se queda cerca del ref
    beta_alto = 1.0
    beta_bajo  = 0.01
    reward    = 2.0
    objetivo_alto = reward - beta_alto * kl
    objetivo_bajo  = reward - beta_bajo * kl
    # Con β alto el objetivo es más penalizado (le "importa más" quedarse cerca)
    assert objetivo_alto < objetivo_bajo
    print(f"✓ Test 5: KL={kl:.4f} ≥ 0; β alto penaliza más la divergencia")

def test_6_sft_format_chat():
    """Formato ChatML incluye roles correctamente."""
    def fmt(user, assistant):
        return f"<|im_start|>user\n{user}<|im_end|>\n<|im_start|>assistant\n{assistant}<|im_end|>"
    prompt = fmt("Hola", "¡Hola! ¿En qué puedo ayudarte?")
    assert "<|im_start|>user" in prompt
    assert "<|im_start|>assistant" in prompt
    assert "<|im_end|>" in prompt
    print("✓ Test 6: Formato ChatML correcto")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M09 — Fine-tuning SFT y RLHF")
    print("=" * 50)
    test_1_sft_loss_ignora_prompt()
    test_2_reward_model_shape()
    test_3_bradley_terry_loss()
    test_4_dpo_loss_formula()
    test_5_kl_penalty_limita_divergencia()
    test_6_sft_format_chat()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M09 completado")
```

---

## Resumen del Módulo

| Etapa | Datos | Objetivo | Costo |
|---|---|---|---|
| **SFT** | (prompt, respuesta) pares | Seguir instrucciones | Moderado |
| **Reward Model** | (prompt, ganadora, perdedora) | Puntuar calidad | Moderado |
| **RLHF (PPO)** | Online generation + RM | Maximizar preferencia | Alto |
| **DPO** | (prompt, ganadora, perdedora) | Preferencia directa | Bajo |
| **CAI** | Principios constitucionales | Seguridad y alineamiento | Moderado |

**Regla de oro:** La calidad de los datos de SFT determina el techo del modelo. RLHF/DPO optimiza dentro de ese techo.

**Próximo módulo:** M10 — Fine-tuning Eficiente: LoRA, QLoRA, PEFT
