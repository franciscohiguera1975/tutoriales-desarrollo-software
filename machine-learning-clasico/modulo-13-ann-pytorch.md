# Módulo 13 — Redes Neuronales Artificiales (ANN) con PyTorch

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M08 (Evaluación de Clasificación)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [La neurona artificial](#la-neurona-artificial)
3. [Perceptrón Multicapa (MLP)](#perceptrón-multicapa-mlp)
4. [Funciones de activación](#funciones-de-activación)
5. [Backpropagation y optimizadores](#backpropagation-y-optimizadores)
6. [ANN con PyTorch — clasificación binaria](#ann-con-pytorch---clasificación-binaria)
7. [ANN con PyTorch — clasificación multiclase](#ann---clasificación-multiclase)
8. [ANN para regresión](#ann-para-regresión)
9. [Regularización en redes neuronales](#regularización)
10. [MLPClassifier de sklearn](#mlpclassifier)
11. [Checkpoint M13](#checkpoint-m13)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
# PyTorch instalado con CPU (sin GPU):
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
python modulo-13-ann-pytorch.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
# PyTorch ya está instalado en Colab con GPU disponible
import torch
print(f"PyTorch {torch.__version__} | GPU: {torch.cuda.is_available()}")
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## La neurona artificial

```
Neurona biológica → neurona artificial:
  Dendritas (entradas) → x₁, x₂, ..., xₙ
  Pesos sinápticos     → w₁, w₂, ..., wₙ
  Cuerpo celular       → Σ(wᵢxᵢ) + b  (suma ponderada + bias)
  Axón (salida)        → a = f(z)       (función de activación)

  z = w₁x₁ + w₂x₂ + ... + wₙxₙ + b = wᵀx + b
  a = f(z)   →  sigmoid, ReLU, tanh, etc.
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings, os
warnings.filterwarnings('ignore')
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

print(f"PyTorch versión: {torch.__version__}")
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Dispositivo: {device}")

# ── Funciones de activación ───────────────────────────────────────────────────
x = np.linspace(-5, 5, 200)

activaciones = {
    'Sigmoid':   1 / (1 + np.exp(-x)),
    'Tanh':      np.tanh(x),
    'ReLU':      np.maximum(0, x),
    'Leaky ReLU': np.where(x >= 0, x, 0.01 * x),
    'ELU':       np.where(x >= 0, x, np.exp(x) - 1),
    'GELU':      x * 0.5 * (1 + np.vectorize(lambda t: np.math.erf(t / np.sqrt(2)))(x)),
}

fig, axes = plt.subplots(2, 3, figsize=(14, 8))
axes = axes.ravel()
colores = ['blue', 'green', 'red', 'orange', 'purple', 'brown']

for ax, (nombre, y_act), color in zip(axes, activaciones.items(), colores):
    ax.plot(x, y_act, color=color, lw=2)
    ax.axhline(0, color='gray', lw=0.5)
    ax.axvline(0, color='gray', lw=0.5)
    ax.set_title(f'{nombre}', fontsize=11, fontweight='bold')
    ax.set_xlim(-5, 5)
    ax.grid(alpha=0.3)

plt.suptitle('Funciones de activación', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m13_activaciones.png', dpi=120)
plt.show()

print("""
Guía de funciones de activación:
  Sigmoid:    output [0,1] → solo en capa de salida para clasificación binaria
  Softmax:    output [0,1] que suman 1 → salida para multiclase
  Tanh:       output [-1,1] → mejor que Sigmoid para capas ocultas (centrada en 0)
  ReLU:       más usado en capas ocultas — simple, evita gradiente evanescente
  Leaky ReLU: ReLU sin "muerte de neuronas" (pendiente pequeña en negativo)
  ELU/GELU:   variantes modernas, GELU es default en Transformers
""")
```

---

## Perceptrón Multicapa (MLP)

```
Arquitectura típica:
  Input Layer  → n_features neuronas
  Hidden Layer(s) → neuronas con activación (ReLU típico)
  Output Layer → n_clases neuronas (softmax multiclase, sigmoid binario)

Forward pass: z = Wx + b → a = f(z)  →  z₂ = W₂a + b₂  → ...
Backward pass: gradiente descende por backpropagation (regla de la cadena)

Decisiones de arquitectura:
  ¿Cuántas capas?    2-3 capas ocultas suelen ser suficientes
  ¿Cuántas neuronas? Empezar con 64/128 → ajustar con validación
  ¿Qué activación?   ReLU para ocultas, Sigmoid/Softmax para salida
  ¿Qué función de pérdida?
    Clasificación binaria:  Binary Cross-Entropy (BCELoss)
    Clasificación multi:    Cross-Entropy (CrossEntropyLoss)
    Regresión:              MSE (MSELoss) o MAE (L1Loss)
```

---

## ANN con PyTorch — clasificación binaria

```python
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import accuracy_score, roc_auc_score

# ── Datos ─────────────────────────────────────────────────────────────────────
cancer = load_breast_cancer()
X, y = cancer.data, cancer.target.astype(np.float32)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X_train, y_train, test_size=0.15, stratify=y_train, random_state=42
)

sc = StandardScaler()
X_train_sc = sc.fit_transform(X_train).astype(np.float32)
X_val_sc   = sc.transform(X_val).astype(np.float32)
X_test_sc  = sc.transform(X_test).astype(np.float32)

# ── Tensores y DataLoaders ────────────────────────────────────────────────────
def make_loader(X, y, batch_size=32, shuffle=True):
    X_t = torch.FloatTensor(X)
    y_t = torch.FloatTensor(y).unsqueeze(1)
    return DataLoader(TensorDataset(X_t, y_t), batch_size=batch_size, shuffle=shuffle)

loader_train = make_loader(X_train_sc, y_train)
loader_val   = make_loader(X_val_sc, y_val, shuffle=False)
loader_test  = make_loader(X_test_sc, y_test, shuffle=False)

print(f"Train: {X_train_sc.shape} | Val: {X_val_sc.shape} | Test: {X_test_sc.shape}")
print(f"Batches por epoch: {len(loader_train)}")

# ── Definición del modelo ─────────────────────────────────────────────────────
class MLP_Binario(nn.Module):
    def __init__(self, n_features, hidden_dims=[128, 64, 32], dropout=0.3):
        super().__init__()

        capas = []
        in_dim = n_features
        for h_dim in hidden_dims:
            capas += [
                nn.Linear(in_dim, h_dim),
                nn.BatchNorm1d(h_dim),
                nn.ReLU(),
                nn.Dropout(dropout),
            ]
            in_dim = h_dim

        capas.append(nn.Linear(in_dim, 1))
        capas.append(nn.Sigmoid())

        self.red = nn.Sequential(*capas)

    def forward(self, x):
        return self.red(x)

model = MLP_Binario(n_features=X_train_sc.shape[1]).to(device)
print(f"\nArquitectura del modelo:")
print(model)

# Contar parámetros
n_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"\nParámetros entrenables: {n_params:,}")
```

### Entrenamiento con loop explícito

```python
# ── Función de entrenamiento ──────────────────────────────────────────────────
def entrenar_modelo(model, loader_train, loader_val, optimizer, criterion,
                    n_epochs=50, patience=10, device=device):
    """
    Loop de entrenamiento con early stopping.
    Retorna: historial de pérdidas y métricas.
    """
    historia = {'train_loss': [], 'val_loss': [], 'val_acc': []}
    mejor_val_loss = float('inf')
    mejor_estado   = None
    sin_mejora     = 0

    for epoch in range(n_epochs):
        # ── Entrenamiento ──────────────────────────────────────────────────────
        model.train()
        train_losses = []

        for X_batch, y_batch in loader_train:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)

            optimizer.zero_grad()
            y_pred = model(X_batch)
            loss   = criterion(y_pred, y_batch)
            loss.backward()
            optimizer.step()

            train_losses.append(loss.item())

        # ── Validación ─────────────────────────────────────────────────────────
        model.eval()
        val_losses, val_preds, val_trues = [], [], []

        with torch.no_grad():
            for X_batch, y_batch in loader_val:
                X_batch, y_batch = X_batch.to(device), y_batch.to(device)
                y_pred = model(X_batch)
                val_losses.append(criterion(y_pred, y_batch).item())
                val_preds.extend(y_pred.cpu().numpy())
                val_trues.extend(y_batch.cpu().numpy())

        train_loss = np.mean(train_losses)
        val_loss   = np.mean(val_losses)
        val_acc    = accuracy_score(
            np.array(val_trues).ravel(),
            (np.array(val_preds).ravel() > 0.5).astype(int)
        )

        historia['train_loss'].append(train_loss)
        historia['val_loss'].append(val_loss)
        historia['val_acc'].append(val_acc)

        if (epoch + 1) % 10 == 0:
            print(f"Epoch [{epoch+1:3d}/{n_epochs}] "
                  f"Train Loss: {train_loss:.4f} | "
                  f"Val Loss: {val_loss:.4f} | "
                  f"Val Acc: {val_acc:.4f}")

        # Early stopping
        if val_loss < mejor_val_loss - 1e-4:
            mejor_val_loss = val_loss
            mejor_estado   = {k: v.clone() for k, v in model.state_dict().items()}
            sin_mejora     = 0
        else:
            sin_mejora += 1
            if sin_mejora >= patience:
                print(f"  Early stopping en epoch {epoch+1}")
                break

    # Restaurar mejor modelo
    if mejor_estado:
        model.load_state_dict(mejor_estado)
    return historia


# ── Entrenamiento ─────────────────────────────────────────────────────────────
criterion = nn.BCELoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)

print("\nEntrenando MLP...")
historia = entrenar_modelo(
    model, loader_train, loader_val,
    optimizer, criterion,
    n_epochs=100, patience=15
)

# ── Evaluación en test ─────────────────────────────────────────────────────────
model.eval()
y_preds, y_probas = [], []
with torch.no_grad():
    for X_batch, _ in loader_test:
        proba = model(X_batch.to(device)).cpu().numpy().ravel()
        y_probas.extend(proba)
        y_preds.extend((proba > 0.5).astype(int))

acc_test  = accuracy_score(y_test, y_preds)
auc_test  = roc_auc_score(y_test, y_probas)
print(f"\nTest Accuracy: {acc_test:.4f} | Test AUC-ROC: {auc_test:.4f}")
```

### Visualización del entrenamiento

```python
# ── Curvas de aprendizaje ─────────────────────────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(13, 4))

epochs_range = range(1, len(historia['train_loss']) + 1)
axes[0].plot(epochs_range, historia['train_loss'], 'b-', lw=2, label='Train Loss')
axes[0].plot(epochs_range, historia['val_loss'],   'r-', lw=2, label='Val Loss')
axes[0].set_xlabel('Epoch')
axes[0].set_ylabel('Binary Cross-Entropy')
axes[0].set_title('Curvas de pérdida — MLP (Breast Cancer)')
axes[0].legend()
axes[0].grid(alpha=0.3)

axes[1].plot(epochs_range, historia['val_acc'], 'g-', lw=2)
axes[1].axhline(acc_test, color='red', ls='--', label=f'Test Acc={acc_test:.4f}')
axes[1].set_xlabel('Epoch')
axes[1].set_ylabel('Accuracy')
axes[1].set_title('Accuracy de validación')
axes[1].legend()
axes[1].grid(alpha=0.3)

plt.tight_layout()
plt.savefig('outputs/m13_curvas_aprendizaje.png', dpi=120)
plt.show()
```

---

## ANN — clasificación multiclase

```python
# ── Iris: clasificación multiclase ───────────────────────────────────────────
from sklearn.datasets import load_iris

iris = load_iris()
X_ir, y_ir = iris.data.astype(np.float32), iris.target

X_tr_ir, X_te_ir, y_tr_ir, y_te_ir = train_test_split(
    X_ir, y_ir, test_size=0.2, stratify=y_ir, random_state=42
)

sc_ir = StandardScaler()
X_tr_ir_sc = sc_ir.fit_transform(X_tr_ir).astype(np.float32)
X_te_ir_sc = sc_ir.transform(X_te_ir).astype(np.float32)

loader_ir_tr = DataLoader(
    TensorDataset(torch.FloatTensor(X_tr_ir_sc), torch.LongTensor(y_tr_ir)),
    batch_size=16, shuffle=True
)
loader_ir_te = DataLoader(
    TensorDataset(torch.FloatTensor(X_te_ir_sc), torch.LongTensor(y_te_ir)),
    batch_size=16, shuffle=False
)

# ── Modelo multiclase ─────────────────────────────────────────────────────────
class MLP_Multiclase(nn.Module):
    def __init__(self, n_features, n_clases, hidden_dims=[64, 32]):
        super().__init__()
        capas = []
        in_dim = n_features
        for h in hidden_dims:
            capas += [nn.Linear(in_dim, h), nn.ReLU()]
            in_dim = h
        capas.append(nn.Linear(in_dim, n_clases))
        # Sin Softmax aquí — CrossEntropyLoss lo aplica internamente
        self.red = nn.Sequential(*capas)

    def forward(self, x):
        return self.red(x)

model_ir = MLP_Multiclase(n_features=4, n_clases=3).to(device)

criterion_ir = nn.CrossEntropyLoss()  # incluye softmax internamente
optimizer_ir = optim.Adam(model_ir.parameters(), lr=1e-2)

# Loop simplificado
hist_ir = {'train_loss': [], 'train_acc': []}
for epoch in range(100):
    model_ir.train()
    losses, corrects, total = [], 0, 0
    for X_b, y_b in loader_ir_tr:
        X_b, y_b = X_b.to(device), y_b.to(device)
        optimizer_ir.zero_grad()
        logits = model_ir(X_b)
        loss   = criterion_ir(logits, y_b)
        loss.backward()
        optimizer_ir.step()
        losses.append(loss.item())
        corrects += (logits.argmax(1) == y_b).sum().item()
        total    += len(y_b)
    hist_ir['train_loss'].append(np.mean(losses))
    hist_ir['train_acc'].append(corrects / total)

# Evaluación
model_ir.eval()
y_preds_ir = []
with torch.no_grad():
    for X_b, _ in loader_ir_te:
        preds = model_ir(X_b.to(device)).argmax(1).cpu().numpy()
        y_preds_ir.extend(preds)

acc_ir = accuracy_score(y_te_ir, y_preds_ir)
print(f"\nMLP Multiclase — Iris Test Accuracy: {acc_ir:.4f}")

# Probabilidades con softmax explícito
model_ir.eval()
all_probas = []
with torch.no_grad():
    for X_b, _ in loader_ir_te:
        probas = torch.softmax(model_ir(X_b.to(device)), dim=1).cpu().numpy()
        all_probas.append(probas)
probas_np = np.vstack(all_probas)
print(f"Probabilidades para los 3 primeros puntos de test:")
for i in range(3):
    probs_str = ', '.join([f"{iris.target_names[j]}:{probas_np[i,j]:.3f}" for j in range(3)])
    print(f"  Punto {i}: {probs_str}")
```

---

## ANN para regresión

```python
# ── Boston/California Housing: regresión con ANN ─────────────────────────────
from sklearn.datasets import fetch_california_housing
from sklearn.metrics import mean_squared_error, r2_score

housing = fetch_california_housing()
X_h, y_h = housing.data.astype(np.float32), housing.target.astype(np.float32)

X_tr_h, X_te_h, y_tr_h, y_te_h = train_test_split(
    X_h, y_h, test_size=0.2, random_state=42
)

sc_h = StandardScaler()
X_tr_h_sc = sc_h.fit_transform(X_tr_h).astype(np.float32)
X_te_h_sc = sc_h.transform(X_te_h).astype(np.float32)

loader_h_tr = DataLoader(
    TensorDataset(torch.FloatTensor(X_tr_h_sc), torch.FloatTensor(y_tr_h).unsqueeze(1)),
    batch_size=256, shuffle=True
)

# Modelo de regresión — sin activación en la salida (predice valores continuos)
class MLP_Regresion(nn.Module):
    def __init__(self, n_features, hidden_dims=[256, 128, 64]):
        super().__init__()
        capas = []
        in_dim = n_features
        for h in hidden_dims:
            capas += [nn.Linear(in_dim, h), nn.BatchNorm1d(h), nn.ReLU(), nn.Dropout(0.2)]
            in_dim = h
        capas.append(nn.Linear(in_dim, 1))  # salida escalar, sin activación
        self.red = nn.Sequential(*capas)

    def forward(self, x):
        return self.red(x)

model_h = MLP_Regresion(n_features=X_tr_h_sc.shape[1]).to(device)
criterion_h = nn.MSELoss()
optimizer_h = optim.Adam(model_h.parameters(), lr=1e-3, weight_decay=1e-5)
scheduler_h = optim.lr_scheduler.ReduceLROnPlateau(optimizer_h, patience=5, factor=0.5)

hist_mse = []
print("Entrenando ANN de regresión...")
for epoch in range(80):
    model_h.train()
    losses = []
    for X_b, y_b in loader_h_tr:
        X_b, y_b = X_b.to(device), y_b.to(device)
        optimizer_h.zero_grad()
        pred = model_h(X_b)
        loss = criterion_h(pred, y_b)
        loss.backward()
        optimizer_h.step()
        losses.append(loss.item())

    avg_loss = np.mean(losses)
    hist_mse.append(avg_loss)
    scheduler_h.step(avg_loss)

    if (epoch + 1) % 20 == 0:
        print(f"  Epoch {epoch+1:3d} | MSE={avg_loss:.4f} | RMSE={np.sqrt(avg_loss):.4f}")

# Evaluación
model_h.eval()
with torch.no_grad():
    X_te_t = torch.FloatTensor(X_te_h_sc).to(device)
    y_pred_h = model_h(X_te_t).cpu().numpy().ravel()

rmse_ann = np.sqrt(mean_squared_error(y_te_h, y_pred_h))
r2_ann   = r2_score(y_te_h, y_pred_h)
print(f"\nANN Regresión — Test RMSE: {rmse_ann:.4f} | R²: {r2_ann:.4f}")
print("(Comparar con RF de M03 en el mismo dataset)")
```

---

## Regularización

```python
# ── Técnicas de regularización para redes neuronales ─────────────────────────
print("""
Regularización en ANNs:
────────────────────────────────────────────────────────────────────
1. Weight Decay (L2):    optimizer = Adam(lr=1e-3, weight_decay=1e-4)
                         → penaliza pesos grandes en la función de pérdida

2. Dropout:              nn.Dropout(p=0.3)
                         → apaga neuronas aleatoriamente durante training
                         → desactiva solo en model.train(), no en model.eval()

3. Batch Normalization:  nn.BatchNorm1d(n_neuronas)
                         → normaliza activaciones por batch → estabiliza entrenamiento
                         → permite learning rates más altos

4. Early Stopping:       detener cuando val_loss no mejora (implementado arriba)

5. Data Augmentation:    en imágenes (ver M14), para tabular: SMOTE, ruido gaussiano

Diagnóstico:
  train_loss ↓ val_loss ↑ → sobreajuste → más dropout, weight_decay, menos capas
  train_loss ↓ val_loss ↓ → subajuste  → más capas/neuronas, menos dropout
""")

# Demostración: efecto del Dropout
class MLP_ConDropout(nn.Module):
    def __init__(self, dropout_rate=0.5):
        super().__init__()
        self.capa1 = nn.Linear(30, 128)
        self.drop1  = nn.Dropout(dropout_rate)
        self.capa2 = nn.Linear(128, 64)
        self.drop2  = nn.Dropout(dropout_rate)
        self.salida = nn.Linear(64, 1)
        self.act    = nn.Sigmoid()

    def forward(self, x):
        x = torch.relu(self.capa1(x))
        x = self.drop1(x)
        x = torch.relu(self.capa2(x))
        x = self.drop2(x)
        return self.act(self.salida(x))

# Comparativa: sin regularización vs con regularización
from sklearn.metrics import accuracy_score

resultados_reg = {}
for config in [
    ('Sin regulaz.', MLP_Binario(30, [256, 128, 64], dropout=0.0)),
    ('Dropout=0.3',  MLP_Binario(30, [256, 128, 64], dropout=0.3)),
    ('Dropout=0.5',  MLP_Binario(30, [128, 64],      dropout=0.5)),
]:
    nombre, m = config
    m = m.to(device)
    opt = optim.Adam(m.parameters(), lr=1e-3, weight_decay=1e-4)
    h   = entrenar_modelo(m, loader_train, loader_val, opt, nn.BCELoss(),
                           n_epochs=50, patience=10)

    m.eval()
    y_p = []
    with torch.no_grad():
        for X_b, _ in loader_test:
            y_p.extend((m(X_b.to(device)).cpu().numpy().ravel() > 0.5).astype(int))

    acc = accuracy_score(y_test, y_p)
    resultados_reg[nombre] = {'Val Loss Final': h['val_loss'][-1], 'Test Acc': acc}
    print(f"  {nombre:<20}: Val Loss={h['val_loss'][-1]:.4f} | Test Acc={acc:.4f}")
```

---

## MLPClassifier

```python
# ── sklearn MLPClassifier — interfaz familiar ─────────────────────────────────
from sklearn.neural_network import MLPClassifier
from sklearn.model_selection import cross_validate, StratifiedKFold
from sklearn.pipeline import Pipeline

# Equivalente a nuestro MLP_Binario pero más simple de usar
mlp_sklearn = Pipeline([
    ('sc', StandardScaler()),
    ('mlp', MLPClassifier(
        hidden_layer_sizes=(128, 64, 32),
        activation='relu',
        solver='adam',
        alpha=1e-4,        # weight decay (L2)
        batch_size=32,
        learning_rate_init=1e-3,
        max_iter=200,
        early_stopping=True,
        validation_fraction=0.15,
        n_iter_no_change=15,
        random_state=42
    ))
])

cv_mlp = StratifiedKFold(5, shuffle=True, random_state=42)
scores_mlp = cross_validate(mlp_sklearn, X, y, cv=cv_mlp,
                              scoring=['accuracy', 'roc_auc'], n_jobs=-1)

print(f"\nMLPClassifier (sklearn):")
print(f"  CV Accuracy: {scores_mlp['test_accuracy'].mean():.4f} ± {scores_mlp['test_accuracy'].std():.4f}")
print(f"  CV AUC-ROC:  {scores_mlp['test_roc_auc'].mean():.4f} ± {scores_mlp['test_roc_auc'].std():.4f}")

# Comparativa final: ANN vs otros clasificadores de M07
from sklearn.ensemble import RandomForestClassifier
from sklearn.svm import SVC

print("\n─── ANN vs Ensemble (Breast Cancer, CV-5) ───")
modelos_comp = {
    'MLPClassifier':    mlp_sklearn,
    'Random Forest':    Pipeline([('sc', StandardScaler()),
                                   ('clf', RandomForestClassifier(n_estimators=100, random_state=42))]),
    'SVM RBF':          Pipeline([('sc', StandardScaler()),
                                   ('clf', SVC(kernel='rbf', probability=True, random_state=42))]),
}
for nombre, clf in modelos_comp.items():
    sc = cross_validate(clf, X, y, cv=cv_mlp, scoring=['accuracy', 'roc_auc'], n_jobs=-1)
    print(f"  {nombre:<20}: Acc={sc['test_accuracy'].mean():.4f} | AUC={sc['test_roc_auc'].mean():.4f}")
```

---

## Guardar el modelo PyTorch

```python
# ── Serialización de modelos PyTorch ─────────────────────────────────────────
import joblib

# Opción 1: guardar solo los pesos (recomendado para PyTorch)
torch.save(model.state_dict(), 'models/mlp_binario_pesos.pt')
print("Pesos guardados en models/mlp_binario_pesos.pt")

# Cargar
model_cargado = MLP_Binario(n_features=X_train_sc.shape[1])
model_cargado.load_state_dict(torch.load('models/mlp_binario_pesos.pt', map_location='cpu'))
model_cargado.eval()

# Verificación
with torch.no_grad():
    X_te_t = torch.FloatTensor(X_test_sc)
    preds_orig    = (model(X_te_t.to(device)).cpu().numpy().ravel() > 0.5).astype(int)
    preds_cargado = (model_cargado(X_te_t).numpy().ravel() > 0.5).astype(int)
assert np.array_equal(preds_orig, preds_cargado)
print("Verificación: modelo cargado produce las mismas predicciones ✓")

# Opción 2: guardar scaler por separado con joblib
joblib.dump(sc, 'models/scaler_mlp.pkl')
print("Scaler guardado en models/scaler_mlp.pkl")

# Opción 3: sklearn MLPClassifier → joblib directamente
mlp_sklearn.fit(X_train, y_train)
joblib.dump(mlp_sklearn, 'models/mlp_sklearn.pkl')
print("MLPClassifier sklearn guardado en models/mlp_sklearn.pkl")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               RESUMEN M13 — Redes Neuronales (ANN) con PyTorch              │
├───────────────────────┬─────────────────────────────────────────────────────┤
│ Componente            │ Descripción                                          │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ nn.Linear(in, out)    │ Capa densa: z = Wx + b                              │
│ nn.ReLU()             │ Activación recomendada para capas ocultas            │
│ nn.Sigmoid()          │ Salida binaria [0,1]                                 │
│ nn.Softmax(dim=1)     │ Salida multiclase [0,1] que suman 1                 │
│ nn.BatchNorm1d(n)     │ Normalización por batch → estabiliza training        │
│ nn.Dropout(p)         │ Regularización → off en eval()                       │
│ BCELoss               │ Pérdida binaria (con Sigmoid)                        │
│ CrossEntropyLoss      │ Pérdida multiclase (incluye Softmax)                 │
│ MSELoss               │ Pérdida de regresión                                 │
│ Adam                  │ Optimizador más usado: lr=1e-3, weight_decay=1e-4    │
│ ReduceLROnPlateau     │ Reduce lr si la val_loss no mejora                   │
│ model.train()         │ Activa Dropout, BN en modo training                  │
│ model.eval()          │ Desactiva Dropout, BN en modo inferencia             │
│ torch.no_grad()       │ No calcula gradientes en inferencia (más rápido)     │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ MLPClassifier sklearn │ Interfaz sklearn, cross_validate compatible          │
└───────────────────────┴─────────────────────────────────────────────────────┘

Cuándo usar ANN vs Random Forest:
  ANN gana:  datos enormes, features densas, relaciones muy complejas
  RF gana:   tabular data, pocos datos, necesitas interpretabilidad, menos tuning
```

---

## Checkpoint M13

```python
# checkpoint_m13.py
"""
Checkpoint M13 — Redes Neuronales (ANN) con PyTorch
Ejecutar: python checkpoint_m13.py
"""
import numpy as np, torch, torch.nn as nn, torch.optim as optim
import warnings; warnings.filterwarnings('ignore')
from sklearn.datasets import load_breast_cancer, load_iris
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import accuracy_score
from torch.utils.data import DataLoader, TensorDataset
import joblib, os

print("=" * 60)
print("CHECKPOINT M13 — ANN con PyTorch")
print("=" * 60)

device = torch.device('cpu')  # CPU para reproducibilidad

# ── Dataset ───────────────────────────────────────────────────────────────────
cancer = load_breast_cancer()
X, y   = cancer.data.astype(np.float32), cancer.target.astype(np.float32)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)

sc  = StandardScaler()
X_tr_sc = sc.fit_transform(X_tr).astype(np.float32)
X_te_sc = sc.transform(X_te).astype(np.float32)

print(f"\n✓ Dataset: {X.shape} | Train={len(X_tr)} | Test={len(X_te)}")

# ── Modelo ────────────────────────────────────────────────────────────────────
torch.manual_seed(42)

model = nn.Sequential(
    nn.Linear(30, 64), nn.ReLU(), nn.Dropout(0.3),
    nn.Linear(64, 32), nn.ReLU(), nn.Dropout(0.3),
    nn.Linear(32, 1),  nn.Sigmoid()
)
n_params = sum(p.numel() for p in model.parameters())
print(f"✓ Modelo: {n_params} parámetros")
assert n_params > 100, "Modelo demasiado pequeño"

# ── Entrenamiento ─────────────────────────────────────────────────────────────
criterion = nn.BCELoss()
optimizer  = optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)

loader = DataLoader(
    TensorDataset(torch.FloatTensor(X_tr_sc), torch.FloatTensor(y_tr).unsqueeze(1)),
    batch_size=32, shuffle=True
)

model.train()
for epoch in range(50):
    for X_b, y_b in loader:
        optimizer.zero_grad()
        loss = criterion(model(X_b), y_b)
        loss.backward()
        optimizer.step()

print("✓ Entrenamiento 50 epochs completado")

# ── Evaluación ────────────────────────────────────────────────────────────────
model.eval()
with torch.no_grad():
    X_te_t = torch.FloatTensor(X_te_sc)
    probas  = model(X_te_t).numpy().ravel()
    preds   = (probas > 0.5).astype(int)

acc = accuracy_score(y_te, preds)
print(f"✓ Test Accuracy: {acc:.4f}")
assert acc > 0.92, f"Accuracy muy baja: {acc:.4f}"

# ── Verificar train/eval mode ─────────────────────────────────────────────────
model.train()
for m in model.modules():
    if isinstance(m, nn.Dropout):
        assert m.training, "Dropout debe estar activo en train()"

model.eval()
for m in model.modules():
    if isinstance(m, nn.Dropout):
        assert not m.training, "Dropout debe estar inactivo en eval()"

print("✓ model.train() / model.eval() funcionan correctamente")

# ── Serialización ─────────────────────────────────────────────────────────────
os.makedirs('models', exist_ok=True)
torch.save(model.state_dict(), 'models/checkpoint_m13_pesos.pt')

model_load = nn.Sequential(
    nn.Linear(30, 64), nn.ReLU(), nn.Dropout(0.3),
    nn.Linear(64, 32), nn.ReLU(), nn.Dropout(0.3),
    nn.Linear(32, 1),  nn.Sigmoid()
)
model_load.load_state_dict(torch.load('models/checkpoint_m13_pesos.pt', map_location='cpu'))
model_load.eval()

with torch.no_grad():
    preds_load = (model_load(X_te_t).numpy().ravel() > 0.5).astype(int)

assert np.array_equal(preds, preds_load), "Modelo cargado produce predicciones diferentes"
print("✓ Serialización de pesos: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M13 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • nn.Sequential con Linear, ReLU, Dropout, Sigmoid")
print("  • DataLoader y loop de entrenamiento")
print("  • BCELoss + Adam optimizer")
print("  • model.train() / model.eval() correctos")
print("  • torch.save / load_state_dict")
print("=" * 60)
```

---

*Siguiente módulo → M14: Redes Neuronales Convolucionales (CNN) con PyTorch*
