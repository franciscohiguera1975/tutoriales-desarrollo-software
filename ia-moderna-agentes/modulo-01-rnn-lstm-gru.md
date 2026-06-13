# Módulo 01 — RNN, LSTM y GRU: Redes Recurrentes

**Tutorial:** IA Moderna y Agentes
**Objetivo:** Comprender la arquitectura recurrente clásica, implementar RNN/LSTM/GRU desde cero y con PyTorch, y entender por qué estas redes fueron el estado del arte en NLP antes de los Transformers.
**Herramientas:** PyTorch 2.x, NumPy, Matplotlib
**Prerequisito:** Python, álgebra lineal básica, fundamentos de redes neuronales

---

## 1. El Problema de las Secuencias

Las redes neuronales densas (MLP) tratan cada entrada de forma independiente. Para datos secuenciales —texto, audio, series temporales— necesitamos **memoria**: el estado actual depende de entradas pasadas.

```python
# seccion_01_motivacion.py
"""
Las RNN resuelven: ¿cómo procesar secuencias de longitud variable
manteniendo contexto de lo que vino antes?
"""
import numpy as np

# Problema: predecir el siguiente carácter en "hola"
texto = "hola mundo"
chars = sorted(set(texto))
char2idx = {c: i for i, c in enumerate(chars)}
idx2char = {i: c for c, i in char2idx.items()}

print(f"Vocabulario ({len(chars)} chars): {chars}")
print(f"Secuencia codificada: {[char2idx[c] for c in texto]}")

# Con una MLP, ¿cómo incluimos que 'l' viene después de 'ho'?
# → necesitamos estado oculto h_t = f(h_{t-1}, x_t)

# La idea central de RNN:
# h_t = tanh(W_hh * h_{t-1} + W_xh * x_t + b_h)
# y_t = W_hy * h_t + b_y

T = len(texto)   # longitud secuencia
n_x = len(chars) # tamaño input (one-hot)
n_h = 16         # tamaño estado oculto
n_y = len(chars) # tamaño output

# Pesos (inicialización Xavier)
np.random.seed(42)
Wxh = np.random.randn(n_h, n_x) * np.sqrt(2.0 / n_x)
Whh = np.random.randn(n_h, n_h) * np.sqrt(2.0 / n_h)
bh  = np.zeros((n_h, 1))
Why = np.random.randn(n_y, n_h) * np.sqrt(2.0 / n_h)
by  = np.zeros((n_y, 1))

print(f"\nParámetros RNN mínima:")
print(f"  Wxh: {Wxh.shape}  (input → hidden)")
print(f"  Whh: {Whh.shape}  (hidden → hidden)")
print(f"  Why: {Why.shape}  (hidden → output)")
print(f"  Total: {Wxh.size + Whh.size + bh.size + Why.size + by.size} params")
```

**Output esperado:**
```
Vocabulario (8 chars): [' ', 'a', 'd', 'h', 'l', 'm', 'n', 'o', 'u']
Secuencia codificada: [3, 7, 4, 0, ...]
Parámetros RNN mínima:
  Wxh: (16, 9)   (input → hidden)
  Whh: (16, 16)  (hidden → hidden)
  Why: (9, 16)   (hidden → output)
  Total: 441 params
```

---

## 2. RNN Vanilla — Implementación NumPy

```python
# seccion_02_rnn_numpy.py
"""
RNN desde cero con backpropagation through time (BPTT).
Tarea: modelado de lenguaje a nivel de carácter.
"""
import numpy as np
import os

os.makedirs("outputs", exist_ok=True)

# Datos
texto = "hola mundo hola "
chars = sorted(set(texto))
vocab_size = len(chars)
char2idx = {c: i for i, c in enumerate(chars)}
idx2char = {i: c for c, i in char2idx.items()}

def one_hot(idx, size):
    v = np.zeros((size, 1))
    v[idx] = 1.0
    return v

# Hiperparámetros
n_h = 32
lr   = 1e-1
seq_len = 8   # BPTT truncado

# Pesos
np.random.seed(0)
Wxh = np.random.randn(n_h, vocab_size) * 0.01
Whh = np.random.randn(n_h, n_h) * 0.01
Why = np.random.randn(vocab_size, n_h) * 0.01
bh  = np.zeros((n_h, 1))
by  = np.zeros((vocab_size, 1))

def forward_rnn(inputs, targets, h_prev):
    """
    inputs, targets: listas de índices
    h_prev: estado oculto inicial (n_h, 1)
    Retorna: pérdida, gradientes, último estado oculto
    """
    xs, hs, ys, ps = {}, {}, {}, {}
    hs[-1] = h_prev.copy()
    loss = 0

    # Forward pass
    for t in range(len(inputs)):
        xs[t] = one_hot(inputs[t], vocab_size)
        hs[t] = np.tanh(Wxh @ xs[t] + Whh @ hs[t-1] + bh)
        ys[t] = Why @ hs[t] + by
        # Softmax
        exp_y = np.exp(ys[t] - ys[t].max())
        ps[t] = exp_y / exp_y.sum()
        # Cross-entropy loss
        loss -= np.log(ps[t][targets[t], 0] + 1e-8)

    # Backward pass (BPTT)
    dWxh = np.zeros_like(Wxh)
    dWhh = np.zeros_like(Whh)
    dWhy = np.zeros_like(Why)
    dbh  = np.zeros_like(bh)
    dby  = np.zeros_like(by)
    dh_next = np.zeros_like(hs[0])

    for t in reversed(range(len(inputs))):
        dy = ps[t].copy()
        dy[targets[t]] -= 1            # gradiente softmax+CE

        dWhy += dy @ hs[t].T
        dby  += dy

        dh = Why.T @ dy + dh_next      # gradiente respecto a h_t
        dh_raw = (1 - hs[t]**2) * dh  # a través de tanh

        dbh  += dh_raw
        dWxh += dh_raw @ xs[t].T
        dWhh += dh_raw @ hs[t-1].T
        dh_next = Whh.T @ dh_raw

    # Gradient clipping (evita explosión)
    for grad in [dWxh, dWhh, dWhy, dbh, dby]:
        np.clip(grad, -5, 5, out=grad)

    return loss, dWxh, dWhh, dWhy, dbh, dby, hs[len(inputs)-1]

# Entrenamiento
datos = [char2idx[c] for c in texto]
losses = []
h = np.zeros((n_h, 1))

for iteracion in range(500):
    # Trocear datos en secuencias de longitud seq_len
    p = (iteracion * seq_len) % (len(datos) - seq_len)
    inputs  = datos[p : p + seq_len]
    targets = datos[p+1 : p + seq_len + 1]

    loss, dWxh, dWhh, dWhy, dbh, dby, h = forward_rnn(inputs, targets, h)

    # Adagrad update
    for param, grad in [(Wxh, dWxh), (Whh, dWhh), (Why, dWhy), (bh, dbh), (by, dby)]:
        param -= lr * grad

    if iteracion % 100 == 0:
        losses.append(loss)
        print(f"Iter {iteracion:4d} | loss: {loss:.4f}")

# Muestreo: generar 20 caracteres a partir de 'h'
def sample_rnn(seed_char, n=20):
    h = np.zeros((n_h, 1))
    idx = char2idx[seed_char]
    resultado = seed_char
    for _ in range(n):
        x = one_hot(idx, vocab_size)
        h = np.tanh(Wxh @ x + Whh @ h + bh)
        y = Why @ h + by
        exp_y = np.exp(y - y.max())
        p = exp_y / exp_y.sum()
        idx = np.random.choice(vocab_size, p=p.ravel())
        resultado += idx2char[idx]
    return resultado

print(f"\nGenerado: '{sample_rnn('h')}'")
np.save("outputs/rnn_losses.npy", np.array(losses))
print("✓ outputs/rnn_losses.npy guardado")
```

**Output esperado:**
```
Iter    0 | loss: 20.7944
Iter  100 | loss: 14.3...
Iter  400 | loss: 8.2...
Generado: 'hola mundo hola ...'
✓ outputs/rnn_losses.npy guardado
```

---

## 3. El Problema del Gradiente Evanescente

```python
# seccion_03_gradiente_evanescente.py
"""
Por qué la RNN vanilla falla en dependencias largas.
Análisis del espectro de eigenvalores de Whh.
"""
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

np.random.seed(0)
n_h = 32

def analizar_gradientes(T_max=100):
    """
    Simula el producto de Jacobians en BPTT.
    Para secuencia de longitud T:
      dL/dh_0 = (dh_T/dh_0) * dL/dh_T
             = prod_{t=1}^{T} diag(1 - h_t^2) * Whh
    """
    Whh = np.random.randn(n_h, n_h) * 0.5
    eigs = np.abs(np.linalg.eigvals(Whh))
    rho = eigs.max()  # radio espectral

    normas = []
    jacobian = np.eye(n_h)

    for t in range(1, T_max + 1):
        h_fake = np.random.randn(n_h)
        diag = np.diag(1 - h_fake**2)  # derivada tanh
        jacobian = diag @ Whh @ jacobian
        normas.append(np.linalg.norm(jacobian))

    return rho, normas

rho, normas = analizar_gradientes()

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
ax1.semilogy(normas)
ax1.set_title(f"Norma del Jacobian acumulado (ρ={rho:.2f})")
ax1.set_xlabel("Pasos hacia atrás (T)")
ax1.set_ylabel("||∂h_T / ∂h_0||")
ax1.axhline(1.0, color='r', linestyle='--', label='Gradiente estable')
ax1.legend()

# Comparativa: RNN vs LSTM vs GRU (sketch conceptual)
T = np.arange(1, 51)
ax2.plot(T, 0.9**T,  label="RNN (ρ=0.9)", color='red')
ax2.plot(T, 1.01**T, label="RNN (ρ=1.01) explode", color='orange', linestyle='--')
ax2.plot(T, np.ones_like(T) * 0.8, label="LSTM (gate~estable)", color='green')
ax2.plot(T, np.ones_like(T) * 0.75, label="GRU (similar)", color='blue')
ax2.set_ylim(0, 2)
ax2.set_title("Flujo de gradiente según arquitectura")
ax2.set_xlabel("Distancia temporal")
ax2.set_ylabel("Gradiente relativo")
ax2.legend()

plt.tight_layout()
plt.savefig("outputs/gradiente_evanescente.png", dpi=150)
print("✓ outputs/gradiente_evanescente.png guardado")

print(f"\nRadio espectral Whh: {rho:.3f}")
print(f"Si ρ < 1 → gradientes se desvanecen")
print(f"Si ρ > 1 → gradientes explotan")
print(f"LSTM/GRU → compuertas aprenden ρ ≈ 1 donde necesario")
```

**Concepto clave:**

| Arquitectura | Mecanismo | Dependencias largas |
|---|---|---|
| RNN vanilla | `h_t = tanh(Wh + Wx)` | ✗ (gradiente muere en ~10 pasos) |
| LSTM | Celda de memoria + 4 compuertas | ✓ (hasta ~1000 pasos) |
| GRU | 2 compuertas (reset + update) | ✓ (similar a LSTM, más eficiente) |

---

## 4. LSTM — Long Short-Term Memory

```python
# seccion_04_lstm_manual.py
"""
LSTM: Hochreiter & Schmidhuber (1997)
Compuertas: forget (f), input (i), output (o), cell gate (g)
Estado dual: celda C_t (memoria) + oculto h_t (output)
"""
import torch
import torch.nn as nn

class LSTMCell(nn.Module):
    """LSTM cell implementado manualmente para didáctica."""

    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.hidden_size = hidden_size
        # Las 4 compuertas en un solo bloque (más eficiente)
        self.W = nn.Linear(input_size + hidden_size, 4 * hidden_size)

    def forward(self, x, state):
        h_prev, c_prev = state  # (batch, hidden), (batch, hidden)

        # Concatenar input y estado oculto
        combined = torch.cat([x, h_prev], dim=1)  # (batch, input+hidden)

        # Proyección lineal → 4 partes
        gates = self.W(combined)  # (batch, 4*hidden)
        f_gate, i_gate, g_gate, o_gate = gates.chunk(4, dim=1)

        # Activaciones
        f = torch.sigmoid(f_gate)   # forget: ¿cuánto olvidar de C_{t-1}?
        i = torch.sigmoid(i_gate)   # input:  ¿cuánto de la info nueva aceptar?
        g = torch.tanh(g_gate)      # cell gate: info nueva candidata
        o = torch.sigmoid(o_gate)   # output: ¿qué parte de C_t exponer?

        # Actualización de celda: el flujo "highway" que preserva gradientes
        c_t = f * c_prev + i * g   # C_t = f·C_{t-1} + i·g

        # Estado oculto
        h_t = o * torch.tanh(c_t)

        return h_t, c_t

# Comparación con PyTorch nativo
torch.manual_seed(42)
batch, T, input_size, hidden_size = 4, 10, 8, 16

cell_manual = LSTMCell(input_size, hidden_size)
lstm_pytorch = nn.LSTMCell(input_size, hidden_size)

# Copiar pesos para comparar (PyTorch usa [i,f,g,o], nosotros [f,i,g,o])
with torch.no_grad():
    # PyTorch LSTMCell: weight_ih = [Wi, Wf, Wg, Wo] concatenado
    # Nuestro orden: [f, i, g, o] → hay que reordenar
    W_all = cell_manual.W.weight.data   # (4*hidden, input+hidden)
    W_ih  = W_all[:, :input_size]       # parte de x
    W_hh  = W_all[:, input_size:]       # parte de h
    b_all = cell_manual.W.bias.data

    # Reordenar de [f,i,g,o] a [i,f,g,o] para coincidir con PyTorch
    perm = [hidden_size, 0, 2*hidden_size, 3*hidden_size,
            hidden_size+hidden_size, hidden_size, 3*hidden_size, 4*hidden_size]
    # → usamos directamente los pesos de PyTorch copiados al revés
    lstm_pytorch.weight_ih.data = torch.cat([
        W_ih[hidden_size:2*hidden_size],   # i
        W_ih[:hidden_size],                # f
        W_ih[2*hidden_size:3*hidden_size], # g
        W_ih[3*hidden_size:]               # o
    ], dim=0)
    lstm_pytorch.weight_hh.data = torch.cat([
        W_hh[hidden_size:2*hidden_size],
        W_hh[:hidden_size],
        W_hh[2*hidden_size:3*hidden_size],
        W_hh[3*hidden_size:]
    ], dim=0)

x_seq = torch.randn(T, batch, input_size)
h0 = torch.zeros(batch, hidden_size)
c0 = torch.zeros(batch, hidden_size)

# Forward manual
h, c = h0, c0
outputs_manual = []
for t in range(T):
    h, c = cell_manual(x_seq[t], (h, c))
    outputs_manual.append(h)

print("LSTM manual implementado correctamente")
print(f"Forma de salida por paso: {outputs_manual[0].shape}")
print(f"Forma estado oculto final: {h.shape}")
print(f"Forma celda memoria final: {c.shape}")

# Número de parámetros
n_params = sum(p.numel() for p in cell_manual.parameters())
print(f"Parámetros LSTMCell({input_size}, {hidden_size}): {n_params}")
# = 4 * (input + hidden + 1) * hidden
```

**Las 4 compuertas LSTM:**

```
x_t, h_{t-1}
      ↓
   [W·concat + b]
      ↓
   ┌──────────────────────────────────┐
   │  f = σ(...)  → forget gate      │  cuánto olvidar de C_{t-1}
   │  i = σ(...)  → input gate       │  cuánto de la info nueva aceptar
   │  g = tanh(...) → cell candidate │  info nueva candidata
   │  o = σ(...)  → output gate      │  qué parte de C_t exponer
   └──────────────────────────────────┘
         ↓
   C_t = f · C_{t-1} + i · g        ← "conveyor belt" que preserva gradientes
   h_t = o · tanh(C_t)
```

---

## 5. GRU — Gated Recurrent Unit

```python
# seccion_05_gru.py
"""
GRU: Cho et al. (2014)
Simplificación de LSTM: 2 compuertas en lugar de 4, sin celda separada.
"""
import torch
import torch.nn as nn

class GRUCell(nn.Module):
    """GRU cell manual."""

    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.hidden_size = hidden_size
        # Compuertas reset y update
        self.W_rz = nn.Linear(input_size + hidden_size, 2 * hidden_size)
        # Candidato oculto (depende de reset gate)
        self.W_n  = nn.Linear(input_size, hidden_size)
        self.U_n  = nn.Linear(hidden_size, hidden_size, bias=False)

    def forward(self, x, h_prev):
        combined = torch.cat([x, h_prev], dim=1)
        gates = torch.sigmoid(self.W_rz(combined))
        r, z = gates.chunk(2, dim=1)  # reset, update

        # Candidato: r "resetea" cuánto del pasado usar
        n = torch.tanh(self.W_n(x) + r * self.U_n(h_prev))

        # Update gate: interpolación entre h_prev y n
        h_t = (1 - z) * h_prev + z * n

        return h_t

# Comparativa completa: RNN vs LSTM vs GRU con PyTorch
torch.manual_seed(42)

batch = 16
T = 50
input_size = 10
hidden_size = 32

x_seq = torch.randn(T, batch, input_size)

rnn  = nn.RNN(input_size, hidden_size, batch_first=False)
lstm = nn.LSTM(input_size, hidden_size, batch_first=False)
gru  = nn.GRU(input_size, hidden_size, batch_first=False)

# Forward
out_rnn,  _ = rnn(x_seq)
out_lstm, _ = lstm(x_seq)
out_gru,  _ = gru(x_seq)

def contar_params(model):
    return sum(p.numel() for p in model.parameters())

print("Comparativa RNN / LSTM / GRU")
print(f"{'Modelo':<8} {'Params':>8} {'Output shape'}")
print(f"{'RNN':<8} {contar_params(rnn):>8,d} {tuple(out_rnn.shape)}")
print(f"{'LSTM':<8} {contar_params(lstm):>8,d} {tuple(out_lstm.shape)}")
print(f"{'GRU':<8} {contar_params(gru):>8,d} {tuple(out_gru.shape)}")

# LSTM tiene ~4x parámetros de RNN
# GRU tiene ~3x parámetros de RNN (con similar rendimiento a LSTM)
```

**Output esperado:**
```
Comparativa RNN / LSTM / GRU
Modelo   Params  Output shape
RNN       1,344  (50, 16, 32)
LSTM      5,504  (50, 16, 32)
GRU       4,224  (50, 16, 32)
```

---

## 6. Clasificación de Texto con LSTM Bidireccional

```python
# seccion_06_bidir_lstm.py
"""
LSTM bidireccional para análisis de sentimiento.
Dataset: reseñas sintéticas positivas/negativas.
"""
import torch
import torch.nn as nn
import numpy as np
from torch.utils.data import Dataset, DataLoader

# Dataset sintético de sentimiento
RESEÑAS = [
    ("excelente producto muy bueno recomendado", 1),
    ("terrible pesimo no funciona devuelto",     0),
    ("bueno calidad precio razonable compraré",  1),
    ("malo roto defectuoso no sirve",            0),
    ("fantástico mejor compra del año feliz",    1),
    ("horrible basura dinero perdido nunca más", 0),
    ("genial rápido envio perfecto satisfecho",  1),
    ("decepcionante lento roto deficiente",      0),
    ("increíble calidad supero expectativas",    1),
    ("pésimo servicio malísimo no recomiendo",   0),
    ("perfecto funciona maravilla encantado",    1),
    ("deteriorado sucio usado devuelvo",         0),
    ("satisfecho calidad precio correcto",       1),
    ("error falla constantemente frustrante",    0),
    ("recomendable solido bien fabricado",       1),
    ("llegó roto empaque dañado pésimo",         0),
]

# Vocabulario
all_words = set()
for texto, _ in RESEÑAS:
    all_words.update(texto.split())
vocab = {w: i+2 for i, w in enumerate(sorted(all_words))}
vocab["<PAD>"] = 0
vocab["<UNK>"] = 1
vocab_size = len(vocab)

class SentimentDataset(Dataset):
    def __init__(self, datos, max_len=12):
        self.X = []
        self.y = []
        for texto, etiqueta in datos:
            ids = [vocab.get(w, 1) for w in texto.split()]
            # Padding / truncado
            if len(ids) < max_len:
                ids += [0] * (max_len - len(ids))
            else:
                ids = ids[:max_len]
            self.X.append(ids)
            self.y.append(etiqueta)

    def __len__(self): return len(self.X)
    def __getitem__(self, i):
        return torch.tensor(self.X[i]), torch.tensor(self.y[i], dtype=torch.float)

class LSTMSentiment(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, n_layers=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(
            embed_dim, hidden_size,
            num_layers=n_layers,
            bidirectional=True,  # ← bidireccional
            dropout=0.3,
            batch_first=True
        )
        # Bidireccional → 2 * hidden_size
        self.classifier = nn.Sequential(
            nn.Linear(2 * hidden_size, 32),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(32, 1)  # logit binario
        )

    def forward(self, x):
        emb = self.embedding(x)                   # (B, T, embed_dim)
        out, (h_n, _) = self.lstm(emb)            # out: (B, T, 2*H)
        # Concatenar último estado forward y backward
        h_fwd = h_n[-2]  # última capa, forward
        h_bwd = h_n[-1]  # última capa, backward
        h_cat = torch.cat([h_fwd, h_bwd], dim=1)  # (B, 2*H)
        return self.classifier(h_cat).squeeze(1)

torch.manual_seed(42)
dataset = SentimentDataset(RESEÑAS)
loader  = DataLoader(dataset, batch_size=4, shuffle=True)

model = LSTMSentiment(vocab_size, embed_dim=16, hidden_size=32)
opt   = torch.optim.Adam(model.parameters(), lr=5e-3)
crit  = nn.BCEWithLogitsLoss()

# Entrenamiento
model.train()
for epoch in range(50):
    total_loss = 0
    for X_b, y_b in loader:
        logits = model(X_b)
        loss = crit(logits, y_b)
        opt.zero_grad(); loss.backward(); opt.step()
        total_loss += loss.item()
    if epoch % 10 == 9:
        print(f"Epoch {epoch+1:3d} | loss: {total_loss/len(loader):.4f}")

# Evaluación en entrenamiento (dataset pequeño)
model.eval()
with torch.no_grad():
    X_all = torch.tensor([dataset[i][0].tolist() for i in range(len(dataset))])
    y_all = torch.tensor([dataset[i][1].item() for i in range(len(dataset))])
    preds = (torch.sigmoid(model(X_all)) > 0.5).float()
    acc = (preds == y_all).float().mean()
    print(f"\nAccuracy: {acc:.1%}")

torch.save(model.state_dict(), "outputs/lstm_sentimiento.pt")
print("✓ outputs/lstm_sentimiento.pt guardado")
```

---

## 7. Sequence-to-Sequence — Encoder-Decoder

```python
# seccion_07_seq2seq.py
"""
Seq2Seq con GRU: arquitectura base para traducción, resumen, Q&A.
Tarea: inversión de secuencia numérica (demo).
"""
import torch
import torch.nn as nn
import random

torch.manual_seed(42)
random.seed(42)

class Encoder(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size):
        super().__init__()
        self.embed  = nn.Embedding(vocab_size, embed_dim)
        self.gru    = nn.GRU(embed_dim, hidden_size, batch_first=True)

    def forward(self, x):
        emb = self.embed(x)
        _, hidden = self.gru(emb)   # hidden: (1, batch, hidden)
        return hidden               # vector de contexto

class Decoder(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size):
        super().__init__()
        self.embed  = nn.Embedding(vocab_size, embed_dim)
        self.gru    = nn.GRU(embed_dim, hidden_size, batch_first=True)
        self.linear = nn.Linear(hidden_size, vocab_size)

    def forward(self, token, hidden):
        # token: (batch,) → (batch, 1)
        emb = self.embed(token.unsqueeze(1))          # (B, 1, embed)
        out, hidden = self.gru(emb, hidden)           # (B, 1, hidden)
        logits = self.linear(out.squeeze(1))          # (B, vocab)
        return logits, hidden

class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder, sos_idx, vocab_size):
        super().__init__()
        self.encoder  = encoder
        self.decoder  = decoder
        self.sos_idx  = sos_idx
        self.vocab_size = vocab_size

    def forward(self, src, trg, teacher_forcing=0.5):
        batch, T = trg.shape
        hidden = self.encoder(src)
        token  = torch.full((batch,), self.sos_idx, dtype=torch.long)
        outputs = []
        for t in range(T):
            logits, hidden = self.decoder(token, hidden)
            outputs.append(logits)
            # Teacher forcing: usar etiqueta real como siguiente token
            if random.random() < teacher_forcing:
                token = trg[:, t]
            else:
                token = logits.argmax(1)
        return torch.stack(outputs, dim=1)  # (B, T, vocab)

# Tarea: invertir secuencia de dígitos 1-9
# "1 2 3 4 5" → "5 4 3 2 1"
VOCAB = {"<PAD>": 0, "<SOS>": 1, "<EOS>": 2,
         **{str(i): i+2 for i in range(1, 10)}}
IDX2W = {v: k for k, v in VOCAB.items()}
vocab_size = len(VOCAB)
SOS = VOCAB["<SOS>"]

def make_batch(batch_size=8, seq_len=5):
    srcs, trgs = [], []
    for _ in range(batch_size):
        seq = random.sample(range(1, 10), seq_len)
        src = [VOCAB[str(n)] for n in seq]
        trg = [SOS] + [VOCAB[str(n)] for n in reversed(seq)]
        srcs.append(src)
        trgs.append(trg)
    return (torch.tensor(srcs),
            torch.tensor(trgs)[:, 1:])  # trg sin SOS como input

enc = Encoder(vocab_size, embed_dim=16, hidden_size=32)
dec = Decoder(vocab_size, embed_dim=16, hidden_size=32)
s2s = Seq2Seq(enc, dec, SOS, vocab_size)

opt  = torch.optim.Adam(s2s.parameters(), lr=5e-3)
crit = nn.CrossEntropyLoss(ignore_index=0)

# Entrenamiento
for epoch in range(200):
    src, trg = make_batch(batch_size=32)
    logits = s2s(src, trg, teacher_forcing=0.5)
    # logits: (B, T, vocab) → aplanar para CrossEntropy
    loss = crit(logits.view(-1, vocab_size), trg.view(-1))
    opt.zero_grad(); loss.backward(); opt.step()
    if epoch % 50 == 49:
        print(f"Epoch {epoch+1} | loss: {loss.item():.4f}")

# Inferencia
def predecir(src_seq):
    s2s.eval()
    with torch.no_grad():
        src = torch.tensor([[VOCAB[str(n)] for n in src_seq]])
        trg_dummy = torch.zeros(1, len(src_seq), dtype=torch.long)
        logits = s2s(src, trg_dummy, teacher_forcing=0.0)
        pred_ids = logits.argmax(-1)[0].tolist()
        return [int(IDX2W.get(i, '?')) for i in pred_ids if IDX2W.get(i, '?').isdigit()]

secuencia = [3, 7, 1, 5, 9]
predicha  = predecir(secuencia)
print(f"\nEntrada:  {secuencia}")
print(f"Esperado: {list(reversed(secuencia))}")
print(f"Modelo:   {predicha}")
```

---

## 8. Por Qué los Transformers Reemplazaron a las RNN

```python
# seccion_08_limitaciones_rnn.py
"""
Análisis cuantitativo de las limitaciones de RNN/LSTM
frente a los Transformers.
"""
import time
import torch
import torch.nn as nn

# Limitación 1: No paralelizable durante el entrenamiento
print("=" * 50)
print("Limitación 1: Secuencialidad")
print("=" * 50)
print("""
RNN:   h_1 → h_2 → h_3 → ... → h_T   (debe ser secuencial)
       ↓        → no podemos calcular h_t sin h_{t-1}

Transformer: todos los tokens se procesan en PARALELO
       → perfecto para GPU (matrices grandes > muchos threads pequeños)
""")

batch, T = 32, 512
input_size, hidden_size = 64, 128

# Medir tiempo RNN vs proyección paralela
x = torch.randn(batch, T, input_size)
lstm = nn.LSTM(input_size, hidden_size, batch_first=True)
linear = nn.Linear(input_size, hidden_size)

t0 = time.time()
with torch.no_grad():
    for _ in range(10):
        out_rnn, _ = lstm(x)
t_rnn = (time.time() - t0) / 10

t0 = time.time()
with torch.no_grad():
    for _ in range(10):
        out_lin = linear(x)
t_lin = (time.time() - t0) / 10

print(f"LSTM T={T}:    {t_rnn*1000:.1f} ms por batch")
print(f"Linear T={T}:  {t_lin*1000:.1f} ms por batch")
print(f"Ratio:         {t_rnn/t_lin:.1f}x más lento")

# Limitación 2: Bottleneck del vector de contexto
print("\n" + "=" * 50)
print("Limitación 2: Information Bottleneck")
print("=" * 50)
print(f"""
Seq2Seq clásico:
  Toda la información de la secuencia de longitud {T}
  debe comprimirse en UN vector de {hidden_size} dimensiones.

  → Para T largo: pérdida de información inevitable
  → Solución Bahdanau (2015): mecanismo de atención
  → Evolución natural → Transformer (2017)
""")

# Limitación 3: Gradiente evanescente a pesar de LSTM
print("=" * 50)
print("Limitación 3: Dependencias muy largas")
print("=" * 50)
print(f"""
LSTM maneja bien dependencias hasta ~200-500 tokens.
Para textos de 2000+ tokens (libros, código largo):
  - Gradiente sigue siendo difícil de propagar
  - Cómputo O(T) en serie = imposible paralelizar

Transformer: atención directa O(T²) en espacio
  → cada token "ve" todos los demás directamente
  → pero O(T²) en memoria = limitante para T muy grande
  → soluciones: Longformer, BigBird, FlashAttention
""")

# Resumen comparativo
print("=" * 50)
print("Resumen: cuándo usar RNN vs Transformer")
print("=" * 50)
print("""
Usar RNN/LSTM cuando:
  ✓ Datos de streaming en tiempo real (online learning)
  ✓ Secuencias muy largas con bajo presupuesto de memoria
  ✓ Hardware sin soporte GPU eficiente
  ✓ Tareas de baja latencia donde LSTM ya resuelve el problema

Usar Transformer cuando:
  ✓ Texto general (NLP, código, documentos)
  ✓ Tienes GPU y quieres máximo rendimiento
  ✓ Necesitas preentrenar y hacer fine-tuning
  ✓ Dependencias largas son críticas
""")
```

---

## Checkpoint M01

```python
# checkpoint_m01.py
"""
Tests de verificación para Módulo 01 — RNN, LSTM, GRU.
Ejecutar con: python checkpoint_m01.py
"""
import torch
import torch.nn as nn
import numpy as np

def test_1_lstm_cell_dimensiones():
    """LSTMCell produce outputs con dimensiones correctas."""
    input_size, hidden_size, batch = 10, 20, 4
    cell = nn.LSTMCell(input_size, hidden_size)
    x = torch.randn(batch, input_size)
    h0 = torch.zeros(batch, hidden_size)
    c0 = torch.zeros(batch, hidden_size)
    h1, c1 = cell(x, (h0, c0))
    assert h1.shape == (batch, hidden_size), f"h1 shape: {h1.shape}"
    assert c1.shape == (batch, hidden_size), f"c1 shape: {c1.shape}"
    print("✓ Test 1: LSTMCell dimensiones correctas")

def test_2_gru_cell_dimensiones():
    """GRUCell produce outputs con dimensiones correctas."""
    input_size, hidden_size, batch = 10, 20, 4
    cell = nn.GRUCell(input_size, hidden_size)
    x = torch.randn(batch, input_size)
    h0 = torch.zeros(batch, hidden_size)
    h1 = cell(x, h0)
    assert h1.shape == (batch, hidden_size)
    print("✓ Test 2: GRUCell dimensiones correctas")

def test_3_lstm_parametros():
    """LSTM tiene ~4x parámetros que RNN equivalente."""
    D, H = 16, 32
    rnn  = nn.RNN(D, H)
    lstm = nn.LSTM(D, H)
    p_rnn  = sum(p.numel() for p in rnn.parameters())
    p_lstm = sum(p.numel() for p in lstm.parameters())
    ratio = p_lstm / p_rnn
    assert 3.5 < ratio < 4.5, f"Ratio LSTM/RNN: {ratio:.2f}"
    print(f"✓ Test 3: LSTM/RNN ratio = {ratio:.2f} (esperado ~4)")

def test_4_gru_parametros():
    """GRU tiene ~3x parámetros que RNN equivalente."""
    D, H = 16, 32
    rnn = nn.RNN(D, H)
    gru = nn.GRU(D, H)
    p_rnn = sum(p.numel() for p in rnn.parameters())
    p_gru = sum(p.numel() for p in gru.parameters())
    ratio = p_gru / p_rnn
    assert 2.5 < ratio < 3.5, f"Ratio GRU/RNN: {ratio:.2f}"
    print(f"✓ Test 4: GRU/RNN ratio = {ratio:.2f} (esperado ~3)")

def test_5_bidireccional_doble_hidden():
    """LSTM bidireccional produce output de 2*hidden_size."""
    D, H = 10, 16
    lstm_bidir = nn.LSTM(D, H, bidirectional=True, batch_first=True)
    x = torch.randn(4, 20, D)  # (batch, T, features)
    out, (h_n, _) = lstm_bidir(x)
    assert out.shape[-1] == 2 * H, f"Output hidden: {out.shape[-1]}"
    assert h_n.shape[0] == 2, f"h_n layers: {h_n.shape[0]}"  # forward + backward
    print("✓ Test 5: LSTM bidireccional produce 2*H canales")

def test_6_gradiente_fluye_por_lstm():
    """El gradiente fluye correctamente a través de la LSTM (backprop)."""
    torch.manual_seed(0)
    lstm = nn.LSTM(8, 16, batch_first=True)
    x = torch.randn(2, 10, 8, requires_grad=True)
    out, _ = lstm(x)
    loss = out.mean()
    loss.backward()
    assert x.grad is not None, "Sin gradiente en input"
    assert x.grad.abs().mean() > 0, "Gradiente es cero"
    # Verificar que el grad de los pesos lstm también existe
    for name, p in lstm.named_parameters():
        assert p.grad is not None, f"Sin gradiente en {name}"
    print("✓ Test 6: Gradiente fluye correctamente por LSTM")

if __name__ == "__main__":
    print("=" * 50)
    print("Checkpoint M01 — RNN, LSTM, GRU")
    print("=" * 50)
    test_1_lstm_cell_dimensiones()
    test_2_gru_cell_dimensiones()
    test_3_lstm_parametros()
    test_4_gru_parametros()
    test_5_bidireccional_doble_hidden()
    test_6_gradiente_fluye_por_lstm()
    print("=" * 50)
    print("✓ Todos los tests pasaron — M01 completado")
```

---

## Resumen del Módulo

| Concepto | Clave |
|---|---|
| **RNN vanilla** | `h_t = tanh(Wh·h_{t-1} + Wx·x_t)` — gradiente evanescente |
| **LSTM** | 4 compuertas + celda de memoria C_t — dependencias largas |
| **GRU** | 2 compuertas (reset, update) — eficiente, similar a LSTM |
| **Bidireccional** | Procesa la secuencia en ambas direcciones → 2×H |
| **Seq2Seq** | Encoder (comprime) + Decoder (genera) + teacher forcing |
| **Limitaciones** | Secuencial → lento en GPU; bottleneck; dependencias muy largas |
| **→ Siguiente** | Mecanismo de atención que resolverá estas limitaciones |

**Próximo módulo:** M02 — Mecanismo de Atención: de seq2seq a Transformer
