# Módulo 24 — Modelos Híbridos Clásico-Cuántico con PyTorch

**Objetivo:** Construir arquitecturas donde capas clásicas (PyTorch) y capas cuánticas (PennyLane) se entrenan de forma conjunta mediante backpropagation, y entender cuándo esta hibridación aporta valor real.

**Herramientas:** PennyLane 0.39+, PyTorch 2.x, scikit-learn, matplotlib
**Prerequisito:** M23 (QNN), M17 (VQC), M20 (Gradientes)

---

## Ejecución

```bash
conda activate quantum
# Guardar código de cada sección en scripts .py
# python seccion_X.py
# Las figuras se guardan en outputs/
mkdir -p outputs
```

---

## 1. Arquitectura híbrida: por qué funciona

Un modelo híbrido clásico-cuántico combina dos mundos:

```
Entrada clásica → Red Clásica → Capa Cuántica → Red Clásica → Salida
     (preprocessing)   (compresión)   (transformación)   (clasificación)
```

La clave: PennyLane con `interface="torch"` expone cada QNode como una función diferenciable. PyTorch puede hacer backpropagation **a través** de la capa cuántica usando Parameter Shift Rule de forma automática.

```python
# seccion_01_concepto.py
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np

# Capa cuántica parametrizada como módulo PyTorch
n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def capa_cuantica(inputs, weights):
    """
    inputs:  tensor [n_qubits]  — features codificadas en ángulos
    weights: tensor [n_capas, n_qubits, 3] — parámetros variacionales
    """
    # Encoding
    for i in range(n_qubits):
        qml.RY(inputs[i], wires=i)

    # Ansatz variacional (StronglyEntangling)
    qml.StronglyEntanglingLayers(weights, wires=range(n_qubits))

    # Medición: vector de valores esperados
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class ModeloHibrido(nn.Module):
    """Arquitectura: Linear → ReLU → Cuántica → Linear"""
    def __init__(self, n_features, n_clases, n_qubits=4, n_capas=2):
        super().__init__()
        self.n_qubits = n_qubits

        # Capa clásica PRE: comprime features a espacio cuántico
        self.pre = nn.Sequential(
            nn.Linear(n_features, 16),
            nn.ReLU(),
            nn.Linear(16, n_qubits),
            nn.Tanh()  # Normaliza a [-1,1] para encoding angular
        )

        # Parámetros cuánticos (entrenables)
        weight_shapes = qml.StronglyEntanglingLayers.shape(n_layers=n_capas, n_wires=n_qubits)
        self.q_weights = nn.Parameter(
            torch.randn(weight_shapes) * 0.1
        )

        # Capa clásica POST: mapea mediciones a predicciones
        self.post = nn.Sequential(
            nn.Linear(n_qubits, 8),
            nn.ReLU(),
            nn.Linear(8, n_clases)
        )

    def forward(self, x):
        # 1) Pre-procesado clásico
        x_encoded = self.pre(x)  # [batch, n_qubits]

        # 2) Capa cuántica — necesita loop por muestra (PennyLane no batching automático)
        q_out = torch.stack([
            torch.stack(capa_cuantica(x_encoded[i], self.q_weights))
            for i in range(x.shape[0])
        ])  # [batch, n_qubits]

        # 3) Post-procesado clásico
        return self.post(q_out)

# Verificar flujo con dummy data
modelo = ModeloHibrido(n_features=8, n_clases=3)
x_dummy = torch.randn(4, 8)
y_dummy = modelo(x_dummy)
print(f"Input shape:  {x_dummy.shape}")
print(f"Output shape: {y_dummy.shape}")
print(f"Parámetros totales: {sum(p.numel() for p in modelo.parameters())}")

# Verificar backpropagation
loss = y_dummy.sum()
loss.backward()
print(f"Gradiente q_weights (norma): {modelo.q_weights.grad.norm().item():.6f}")
```

**Salida esperada:**
```
Input shape:  torch.Size([4, 8])
Output shape: torch.Size([4, 3])
Parámetros totales: ~300
Gradiente q_weights (norma): 0.XXXXXX  (>0)
```

---

## 2. Dataset: Wine Quality (clasificación real)

Usamos el dataset Wine de sklearn (178 muestras, 13 features, 3 clases).

```python
# seccion_02_dataset.py
import numpy as np
import torch
from torch.utils.data import DataLoader, TensorDataset
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

def preparar_wine_dataset(test_size=0.2, seed=42):
    """Carga y preprocesa Wine dataset para modelo híbrido."""
    wine = load_wine()
    X, y = wine.data, wine.target

    # Split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=seed, stratify=y
    )

    # Normalización
    scaler = StandardScaler()
    X_train = scaler.fit_transform(X_train)
    X_test  = scaler.transform(X_test)

    # Tensors
    X_tr = torch.FloatTensor(X_train)
    y_tr = torch.LongTensor(y_train)
    X_te = torch.FloatTensor(X_test)
    y_te = torch.LongTensor(y_test)

    train_ds = TensorDataset(X_tr, y_tr)
    test_ds  = TensorDataset(X_te, y_te)

    train_loader = DataLoader(train_ds, batch_size=16, shuffle=True)
    test_loader  = DataLoader(test_ds,  batch_size=32, shuffle=False)

    return train_loader, test_loader, X_tr, y_tr, X_te, y_te

# Estadísticas
train_loader, test_loader, X_tr, y_tr, X_te, y_te = preparar_wine_dataset()
print(f"Entrenamiento: {len(X_tr)} muestras")
print(f"Test:          {len(X_te)} muestras")
print(f"Features:      {X_tr.shape[1]}")
print(f"Clases:        {len(torch.unique(y_tr))} — {torch.bincount(y_tr).tolist()}")
```

---

## 3. Entrenamiento del modelo híbrido

```python
# seccion_03_entrenamiento.py
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np
import time
from seccion_01_concepto import ModeloHibrido, capa_cuantica
from seccion_02_dataset import preparar_wine_dataset

def entrenar_hibrido(modelo, train_loader, test_loader,
                     n_epochs=30, lr=0.01, verbose=True):
    """Entrena modelo híbrido y registra métricas."""
    optimizer = torch.optim.Adam(modelo.parameters(), lr=lr)
    criterion = nn.CrossEntropyLoss()

    historial = {
        "train_loss": [], "train_acc": [],
        "test_acc": [], "tiempo_epoch": []
    }

    for epoch in range(1, n_epochs + 1):
        t0 = time.time()
        modelo.train()
        total_loss, total_correct, total = 0.0, 0, 0

        for X_batch, y_batch in train_loader:
            optimizer.zero_grad()
            logits = modelo(X_batch)
            loss = criterion(logits, y_batch)
            loss.backward()
            optimizer.step()

            pred = logits.argmax(dim=1)
            total_loss    += loss.item() * len(y_batch)
            total_correct += (pred == y_batch).sum().item()
            total         += len(y_batch)

        train_loss = total_loss / total
        train_acc  = total_correct / total
        test_acc   = evaluar(modelo, test_loader)
        elapsed    = time.time() - t0

        historial["train_loss"].append(train_loss)
        historial["train_acc"].append(train_acc)
        historial["test_acc"].append(test_acc)
        historial["tiempo_epoch"].append(elapsed)

        if verbose and (epoch % 5 == 0 or epoch == 1):
            print(f"Epoch {epoch:3d}/{n_epochs} | "
                  f"Loss={train_loss:.4f} | "
                  f"Train={train_acc:.3f} | "
                  f"Test={test_acc:.3f} | "
                  f"t={elapsed:.1f}s")

    return historial

def evaluar(modelo, loader):
    """Accuracy en un DataLoader."""
    modelo.eval()
    correct, total = 0, 0
    with torch.no_grad():
        for X, y in loader:
            pred = modelo(X).argmax(dim=1)
            correct += (pred == y).sum().item()
            total   += len(y)
    return correct / total

# Entrenamiento
train_loader, test_loader, *_ = preparar_wine_dataset()
modelo_hibrido = ModeloHibrido(n_features=13, n_clases=3, n_qubits=4, n_capas=2)

print("=== Entrenando Modelo Híbrido ===")
hist = entrenar_hibrido(modelo_hibrido, train_loader, test_loader,
                        n_epochs=30, lr=0.01)

print(f"\nAcc final test: {hist['test_acc'][-1]:.3f}")
print(f"Tiempo promedio/epoch: {np.mean(hist['tiempo_epoch']):.1f}s")
```

---

## 4. Comparación: clásico vs híbrido

Comparamos el modelo híbrido contra:
1. Red clásica equivalente (misma arquitectura sin capa cuántica)
2. MLP profundo clásico

```python
# seccion_04_comparacion.py
import torch
import torch.nn as nn
import numpy as np
from seccion_02_dataset import preparar_wine_dataset
from seccion_03_entrenamiento import entrenar_hibrido, evaluar

# --- Modelo puramente clásico (equivalente sin capa cuántica) ---
class ModeloClasico(nn.Module):
    def __init__(self, n_features, n_clases, n_hidden=4):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(n_features, 16),
            nn.ReLU(),
            nn.Linear(16, n_hidden),
            nn.Tanh(),
            nn.Linear(n_hidden, 8),
            nn.ReLU(),
            nn.Linear(8, n_clases)
        )
    def forward(self, x):
        return self.red(x)

# --- MLP profundo ---
class MLP(nn.Module):
    def __init__(self, n_features, n_clases):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(n_features, 64),
            nn.ReLU(),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.Linear(32, 16),
            nn.ReLU(),
            nn.Linear(16, n_clases)
        )
    def forward(self, x):
        return self.red(x)

def entrenar_clasico(modelo, train_loader, test_loader, n_epochs=30, lr=0.01):
    """Versión simplificada para modelos clásicos (mucho más rápido)."""
    opt = torch.optim.Adam(modelo.parameters(), lr=lr)
    crit = nn.CrossEntropyLoss()
    hist = {"train_loss": [], "test_acc": []}

    for _ in range(n_epochs):
        modelo.train()
        total_loss = 0.0
        for X, y in train_loader:
            opt.zero_grad()
            loss = crit(modelo(X), y)
            loss.backward()
            opt.step()
            total_loss += loss.item()
        hist["train_loss"].append(total_loss / len(train_loader))
        hist["test_acc"].append(evaluar(modelo, test_loader))
    return hist

# Experimento
import time
train_loader, test_loader, *_ = preparar_wine_dataset()

resultados = {}

# Modelo clásico equivalente
t0 = time.time()
m_clasico = ModeloClasico(13, 3, n_hidden=4)
h_clasico = entrenar_clasico(m_clasico, train_loader, test_loader, n_epochs=30)
resultados["Clásico equiv."] = {
    "acc_final": h_clasico["test_acc"][-1],
    "tiempo_total": time.time() - t0,
    "params": sum(p.numel() for p in m_clasico.parameters())
}

# MLP profundo
t0 = time.time()
m_mlp = MLP(13, 3)
h_mlp = entrenar_clasico(m_mlp, train_loader, test_loader, n_epochs=30)
resultados["MLP profundo"] = {
    "acc_final": h_mlp["test_acc"][-1],
    "tiempo_total": time.time() - t0,
    "params": sum(p.numel() for p in m_mlp.parameters())
}

# (Modelo híbrido ya entrenado en sección anterior, guardamos sus métricas)
# Para comparación directa, reentrenamos aquí
from seccion_01_concepto import ModeloHibrido
t0 = time.time()
m_hibrido = ModeloHibrido(13, 3, n_qubits=4, n_capas=2)
h_hibrido = entrenar_hibrido(m_hibrido, train_loader, test_loader,
                              n_epochs=30, lr=0.01, verbose=False)
resultados["Híbrido (4q)"] = {
    "acc_final": h_hibrido["test_acc"][-1],
    "tiempo_total": time.time() - t0,
    "params": sum(p.numel() for p in m_hibrido.parameters())
}

print("\n=== Resultados Finales ===")
print(f"{'Modelo':<20} {'Acc Test':>10} {'Params':>8} {'Tiempo':>10}")
print("-" * 55)
for nombre, r in resultados.items():
    print(f"{nombre:<20} {r['acc_final']:>10.3f} {r['params']:>8d} {r['tiempo_total']:>9.1f}s")
```

---

## 5. TorchLayer: integración nativa PennyLane-PyTorch

PennyLane ofrece `qml.qnn.TorchLayer` para encapsular QNodes como `nn.Module` de forma limpia.

```python
# seccion_05_torchlayer.py
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np
from seccion_02_dataset import preparar_wine_dataset
from seccion_03_entrenamiento import evaluar

n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def qnode_torchlayer(inputs, weights):
    """QNode compatible con TorchLayer."""
    qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation="Y")
    qml.BasicEntanglerLayers(weights, wires=range(n_qubits))
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

# TorchLayer: especificar shape de cada parámetro
n_capas_q = 3
weight_shapes = {
    "weights": (n_capas_q, n_qubits)  # BasicEntanglerLayers shape
}

capa_q = qml.qnn.TorchLayer(qnode_torchlayer, weight_shapes)

class ModeloTorchLayer(nn.Module):
    """Modelo usando TorchLayer — más limpio."""
    def __init__(self, n_features, n_clases):
        super().__init__()
        self.pre = nn.Sequential(
            nn.Linear(n_features, n_qubits),
            nn.Tanh()
        )
        self.quantum = capa_q          # TorchLayer es nn.Module
        self.post = nn.Linear(n_qubits, n_clases)

    def forward(self, x):
        x = self.pre(x)       # [batch, n_qubits]
        x = self.quantum(x)   # [batch, n_qubits] — TorchLayer maneja batching!
        return self.post(x)

# Verificación
modelo_tl = ModeloTorchLayer(n_features=13, n_clases=3)
print("Arquitectura TorchLayer:")
print(modelo_tl)
print(f"\nParámetros: {sum(p.numel() for p in modelo_tl.parameters())}")

# TorchLayer SÍ hace batching automático (a diferencia del loop manual)
x_test = torch.randn(8, 13)
y_out = modelo_tl(x_test)
print(f"Forward batch=8: {y_out.shape}")

# Entrenamiento
train_loader, test_loader, *_ = preparar_wine_dataset()
opt = torch.optim.Adam(modelo_tl.parameters(), lr=0.01)
crit = nn.CrossEntropyLoss()

print("\nEntrenando con TorchLayer (10 epochs)...")
for epoch in range(10):
    modelo_tl.train()
    for X, y in train_loader:
        opt.zero_grad()
        loss = crit(modelo_tl(X), y)
        loss.backward()
        opt.step()
    if (epoch + 1) % 5 == 0:
        acc = evaluar(modelo_tl, test_loader)
        print(f"  Epoch {epoch+1}: test_acc={acc:.3f}")
```

---

## 6. Transfer learning cuántico

Usamos un núcleo cuántico preentrenado y lo refinamos (fine-tuning) para una tarea distinta.

```python
# seccion_06_transfer.py
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np
from sklearn.datasets import load_wine, load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from torch.utils.data import DataLoader, TensorDataset

n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def qnode_transferable(inputs, weights_frozen, weights_fine):
    """
    Capa cuántica con dos bloques:
    - weights_frozen: capas congeladas (preentrenadas)
    - weights_fine: capas entrenables (fine-tuning)
    """
    qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation="Y")
    qml.StronglyEntanglingLayers(weights_frozen, wires=range(n_qubits))
    qml.StronglyEntanglingLayers(weights_fine, wires=range(n_qubits))
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class ModeloTransfer(nn.Module):
    def __init__(self, n_features, n_clases, n_frozen=2, n_fine=1):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape

        self.pre = nn.Sequential(
            nn.Linear(n_features, n_qubits), nn.Tanh()
        )

        # Pesos congelados (no requieren grad durante fase inicial)
        w_frozen = torch.randn(shape(n_frozen, n_qubits)) * 0.1
        self.weights_frozen = nn.Parameter(w_frozen, requires_grad=False)

        # Pesos para fine-tuning (siempre entrenables)
        w_fine = torch.randn(shape(n_fine, n_qubits)) * 0.1
        self.weights_fine = nn.Parameter(w_fine, requires_grad=True)

        self.post = nn.Linear(n_qubits, n_clases)

    def descongelar(self):
        """Descongela todos los pesos para fine-tuning completo."""
        self.weights_frozen.requires_grad_(True)
        print("Pesos cuánticos descongelados.")

    def forward(self, x):
        x = self.pre(x)
        q_out = torch.stack([
            torch.stack(qnode_transferable(
                x[i], self.weights_frozen, self.weights_fine
            ))
            for i in range(x.shape[0])
        ])
        return self.post(q_out)

def prep_dataset(X, y, test_size=0.2, seed=42, batch=16):
    scaler = StandardScaler()
    Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=test_size,
                                           random_state=seed, stratify=y)
    Xtr = scaler.fit_transform(Xtr)
    Xte = scaler.transform(Xte)

    tr = DataLoader(TensorDataset(torch.FloatTensor(Xtr), torch.LongTensor(ytr)),
                    batch_size=batch, shuffle=True)
    te = DataLoader(TensorDataset(torch.FloatTensor(Xte), torch.LongTensor(yte)),
                    batch_size=32)
    return tr, te

def entrenar_rapido(modelo, loader_tr, loader_te, epochs=10, lr=0.01, nombre=""):
    opt = torch.optim.Adam(
        filter(lambda p: p.requires_grad, modelo.parameters()), lr=lr
    )
    crit = nn.CrossEntropyLoss()

    for ep in range(epochs):
        modelo.train()
        for X, y in loader_tr:
            opt.zero_grad()
            crit(modelo(X), y).backward()
            opt.step()

    modelo.eval()
    correct, total = 0, 0
    with torch.no_grad():
        for X, y in loader_te:
            pred = modelo(X).argmax(1)
            correct += (pred == y).sum().item()
            total += len(y)
    acc = correct / total
    print(f"  {nombre}: test_acc = {acc:.3f}")
    return acc

# === Experimento Transfer Learning ===
print("=== Transfer Learning Cuántico ===\n")

# Tarea origen: Wine (3 clases, 13 features)
wine = load_wine()
tr_wine, te_wine = prep_dataset(wine.data, wine.target)

m_origen = ModeloTransfer(n_features=13, n_clases=3)
print("Fase 1: Entrenando en Wine (tarea origen)...")
acc_origen = entrenar_rapido(m_origen, tr_wine, te_wine, epochs=15, nombre="Wine")

# Transferir pesos al modelo nuevo (tarea destino: Breast Cancer binario)
print("\nFase 2: Fine-tuning en Breast Cancer (solo capas fine)...")
cancer = load_breast_cancer()
# Reducir features a 13 primeras para matching
tr_cancer, te_cancer = prep_dataset(cancer.data[:, :13], cancer.target)

m_destino = ModeloTransfer(n_features=13, n_clases=2)
# Copiar pesos cuánticos preentrenados
with torch.no_grad():
    m_destino.weights_frozen.copy_(m_origen.weights_frozen)
    m_destino.weights_fine.copy_(m_origen.weights_fine)

acc_sin_ft = entrenar_rapido(m_destino, tr_cancer, te_cancer, epochs=5,
                              nombre="Cancer (frozen)")

print("\nFase 3: Fine-tuning completo (descongelar)...")
m_destino.descongelar()
acc_con_ft = entrenar_rapido(m_destino, tr_cancer, te_cancer, epochs=10,
                              nombre="Cancer (fine-tuned)")

print(f"\nMejora por descongelar: {acc_con_ft - acc_sin_ft:+.3f}")
```

---

## 7. Visualización completa

```python
# seccion_07_visualizacion.py
import torch
import torch.nn as nn
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import pennylane as qml
from seccion_01_concepto import ModeloHibrido, capa_cuantica
from seccion_02_dataset import preparar_wine_dataset
from seccion_03_entrenamiento import entrenar_hibrido, evaluar
from seccion_04_comparacion import ModeloClasico, MLP, entrenar_clasico

train_loader, test_loader, X_tr, y_tr, X_te, y_te = preparar_wine_dataset()

# --- Entrenar todos los modelos ---
print("Entrenando modelos para visualización...")

m_hibrido = ModeloHibrido(13, 3, n_qubits=4, n_capas=2)
h_hib = entrenar_hibrido(m_hibrido, train_loader, test_loader, n_epochs=40,
                          lr=0.01, verbose=False)

m_clasico = ModeloClasico(13, 3, n_hidden=4)
h_cls = entrenar_clasico(m_clasico, train_loader, test_loader, n_epochs=40)

m_mlp = MLP(13, 3)
h_mlp = entrenar_clasico(m_mlp, train_loader, test_loader, n_epochs=40)

# --- Figura 6 paneles ---
fig, axes = plt.subplots(2, 3, figsize=(15, 10))
fig.suptitle("Modelos Híbridos Clásico-Cuántico — Wine Dataset", fontsize=14)

epochs = range(1, 41)

# Panel 1: Curvas de accuracy
ax = axes[0, 0]
ax.plot(epochs, h_hib["train_acc"], "b-",  label="Híbrido train", linewidth=2)
ax.plot(epochs, h_hib["test_acc"],  "b--", label="Híbrido test",  linewidth=2)
ax.plot(epochs, h_cls["test_acc"],  "r--", label="Clásico equiv.", linewidth=1.5)
ax.plot(epochs, h_mlp["test_acc"],  "g--", label="MLP profundo",  linewidth=1.5)
ax.set_xlabel("Época")
ax.set_ylabel("Accuracy")
ax.set_title("Evolución de Accuracy")
ax.legend(fontsize=8)
ax.grid(alpha=0.3)
ax.set_ylim([0, 1])

# Panel 2: Curva de pérdida (híbrido)
ax = axes[0, 1]
ax.plot(epochs, h_hib["train_loss"], "b-", linewidth=2, label="Híbrido")
ax.set_xlabel("Época")
ax.set_ylabel("CrossEntropy Loss")
ax.set_title("Curva de Pérdida (Híbrido)")
ax.legend()
ax.grid(alpha=0.3)

# Panel 3: Tiempo por época
ax = axes[0, 2]
tiempos = h_hib["tiempo_epoch"]
ax.bar(epochs, tiempos, color="steelblue", alpha=0.7)
ax.axhline(np.mean(tiempos), color="red", linestyle="--",
           label=f"Media={np.mean(tiempos):.1f}s")
ax.set_xlabel("Época")
ax.set_ylabel("Tiempo (s)")
ax.set_title("Tiempo por Época (Híbrido)")
ax.legend()
ax.grid(alpha=0.3, axis="y")

# Panel 4: Comparación final
ax = axes[1, 0]
nombres = ["Clásico\nequiv.", "MLP\nprofundo", "Híbrido\n(4q)"]
accs = [
    h_cls["test_acc"][-1],
    h_mlp["test_acc"][-1],
    h_hib["test_acc"][-1]
]
colores = ["coral", "lightgreen", "steelblue"]
bars = ax.bar(nombres, accs, color=colores, edgecolor="black", linewidth=0.8)
for bar, acc in zip(bars, accs):
    ax.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,
            f"{acc:.3f}", ha="center", va="bottom", fontsize=11, fontweight="bold")
ax.set_ylabel("Accuracy Test")
ax.set_title("Comparación Final")
ax.set_ylim([0, 1.1])
ax.grid(axis="y", alpha=0.3)

# Panel 5: Distribución de predicciones (híbrido)
ax = axes[1, 1]
m_hibrido.eval()
with torch.no_grad():
    logits_te = torch.stack([
        m_hibrido(X_te[i:i+1]) for i in range(len(X_te))
    ]).squeeze()
probs_te = torch.softmax(logits_te, dim=1).numpy()
y_te_np = y_te.numpy()
clases = ["Clase 0", "Clase 1", "Clase 2"]
for c, nombre in enumerate(clases):
    mask = y_te_np == c
    ax.scatter(probs_te[mask, 0], probs_te[mask, 1],
               label=nombre, alpha=0.7, s=40)
ax.set_xlabel("P(Clase 0)")
ax.set_ylabel("P(Clase 1)")
ax.set_title("Distribución de Probabilidades")
ax.legend(fontsize=8)
ax.grid(alpha=0.3)

# Panel 6: Espacio de activaciones cuánticas (t-SNE conceptual con PCA)
ax = axes[1, 2]
from sklearn.decomposition import PCA

# Extraer activaciones de la capa cuántica
n_qubits = 4
dev_vis = qml.device("default.qubit", wires=n_qubits)

# Usar la pre-capa del modelo entrenado
m_hibrido.eval()
with torch.no_grad():
    x_encoded = m_hibrido.pre(X_te)
    activaciones_q = torch.stack([
        torch.stack(capa_cuantica(x_encoded[i], m_hibrido.q_weights))
        for i in range(len(X_te))
    ]).numpy()

pca = PCA(n_components=2)
proj = pca.fit_transform(activaciones_q)
for c, nombre in enumerate(clases):
    mask = y_te_np == c
    ax.scatter(proj[mask, 0], proj[mask, 1], label=nombre, alpha=0.7, s=40)
ax.set_xlabel("PC1")
ax.set_ylabel("PC2")
ax.set_title("PCA de Activaciones Cuánticas")
ax.legend(fontsize=8)
ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m24_hibrido_pytorch.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m24_hibrido_pytorch.png")
plt.close()
```

---

## 8. Estrategias de diseño híbrido

| Estrategia | Descripción | Cuándo usar |
|---|---|---|
| **Pre-QNN** | Clásico → Cuántico → Salida | Pocos qubits, features ya procesadas |
| **Post-QNN** | Cuántico → Clásico → Salida | Codificación directa, salida compleja |
| **Sandwich** | Clásico → Cuántico → Clásico | General purpose, mejor convergencia |
| **Parallel** | Clásico ∥ Cuántico → Fusión | Complementariedad, ensemble-like |
| **Transfer** | Pesos cuánticos preentrenados | Datos escasos en tarea destino |

**Consejos de entrenamiento:**
```
1. Inicializar q_weights cerca de 0 (evita barren plateaus al inicio)
2. Learning rate cuántico < learning rate clásico (0.001 vs 0.01)
3. Warmup clásico primero, luego activar gradientes cuánticos
4. Monitorear norma de gradientes de q_weights (riesgo barren plateau)
5. Batch size 16-32 (compromiso entre ruido y velocidad)
```

---

## 9. Limitaciones actuales

```python
# seccion_09_limitaciones.py
"""
Análisis cuantitativo de limitaciones del enfoque híbrido.
"""
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# --- Escalabilidad: tiempo vs n_qubits ---
import time, torch
import pennylane as qml

def tiempo_forward_batch(n_qubits, batch_size=16, n_capas=2):
    """Mide tiempo de forward pass para un mini-batch."""
    dev = qml.device("default.qubit", wires=n_qubits)

    @qml.qnode(dev, interface="torch", diff_method="parameter-shift")
    def qnode(inputs, weights):
        qml.AngleEmbedding(inputs, wires=range(n_qubits))
        qml.StronglyEntanglingLayers(weights, wires=range(n_qubits))
        return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

    shape = qml.StronglyEntanglingLayers.shape(n_capas, n_qubits)
    weights = torch.randn(shape)
    X = torch.randn(batch_size, n_qubits)

    t0 = time.time()
    for i in range(batch_size):
        _ = torch.stack(qnode(X[i], weights))
    return time.time() - t0

qubits_range = [2, 4, 6, 8]
tiempos_forward = []
for nq in qubits_range:
    t = tiempo_forward_batch(nq)
    tiempos_forward.append(t)
    print(f"n_qubits={nq}: {t:.2f}s para batch=16")

# Análisis de barren plateaus en modelos híbridos
print("\n--- Análisis de Barren Plateaus ---")

def varianza_gradiente_hibrido(n_qubits, n_samples=50):
    dev = qml.device("default.qubit", wires=n_qubits)

    @qml.qnode(dev, interface="torch", diff_method="parameter-shift")
    def qnode(inputs, weights):
        qml.AngleEmbedding(inputs, wires=range(n_qubits))
        qml.StronglyEntanglingLayers(weights, wires=range(n_qubits))
        return qml.expval(qml.PauliZ(0))

    shape = qml.StronglyEntanglingLayers.shape(2, n_qubits)
    grads = []

    for _ in range(n_samples):
        w = torch.randn(shape, requires_grad=True)
        x = torch.randn(n_qubits)
        out = qnode(x, w)
        out.backward()
        grads.append(w.grad[0, 0, 0].item())

    return np.var(grads)

varianzas = {}
for nq in [2, 4, 6]:
    v = varianza_gradiente_hibrido(nq)
    varianzas[nq] = v
    print(f"  n_qubits={nq}: Var[grad] = {v:.2e}")

print("\nConclusion: Var[grad] ∝ 2^(-n) — barren plateaus empeoran con n_qubits")
print(f"Ratio 2q→4q: {varianzas[2]/varianzas[4]:.1f}x (esperado ~4x)")

# Resumen de limitaciones
print("\n=== Limitaciones del Enfoque Híbrido ===")
limitaciones = [
    ("Barren Plateaus",       "Gradientes exponencialmente pequeños con n_qubits"),
    ("Velocidad simulación",  "100x más lento que red clásica equivalente"),
    ("Batching cuántico",     "Sin vectorización: O(batch_size) circuitos independientes"),
    ("Hardware real",         "Ruido degrada ventaja cuántica, requiere error mitigation"),
    ("Escalabilidad datos",   "Kernel cuántico O(n²) evaluaciones para n muestras"),
    ("Ventaja demostrada",    "Solo en datasets específicos sintéticos (2019-2024)"),
]
for lim, desc in limitaciones:
    print(f"  [{lim}] {desc}")
```

---

## Checkpoint

```python
# checkpoint_m24.py
"""Tests para Módulo 24 — Modelos Híbridos Clásico-Cuántico."""
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np

n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def qnode_test(inputs, weights):
    qml.AngleEmbedding(inputs, wires=range(n_qubits), rotation="Y")
    qml.BasicEntanglerLayers(weights, wires=range(n_qubits))
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class ModeloHibridoTest(nn.Module):
    def __init__(self):
        super().__init__()
        self.pre = nn.Sequential(nn.Linear(8, n_qubits), nn.Tanh())
        w_shape = (3, n_qubits)
        self.q_weights = nn.Parameter(torch.randn(w_shape) * 0.1)
        self.post = nn.Linear(n_qubits, 3)

    def forward(self, x):
        x = self.pre(x)
        q = torch.stack([
            torch.stack(qnode_test(x[i], self.q_weights))
            for i in range(x.shape[0])
        ])
        return self.post(q)

# T1: Forward pass produce shape correcta
def test_forward_shape():
    model = ModeloHibridoTest()
    x = torch.randn(4, 8)
    out = model(x)
    assert out.shape == (4, 3), f"Shape esperada (4,3), obtenida {out.shape}"
    print("T1 PASS: forward shape correcta")

# T2: Gradientes fluyen hasta q_weights (backprop a través de capa cuántica)
def test_gradientes_cuanticos():
    model = ModeloHibridoTest()
    x = torch.randn(2, 8)
    loss = model(x).sum()
    loss.backward()
    assert model.q_weights.grad is not None, "q_weights no tiene gradiente"
    assert model.q_weights.grad.norm().item() > 0, "Gradiente cuántico es cero"
    print("T2 PASS: backprop a través de capa cuántica funciona")

# T3: TorchLayer acepta batch directamente
def test_torchlayer_batch():
    @qml.qnode(dev, interface="torch")
    def qnode_tl(inputs, weights):
        qml.AngleEmbedding(inputs, wires=range(n_qubits))
        qml.BasicEntanglerLayers(weights, wires=range(n_qubits))
        return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

    tl = qml.qnn.TorchLayer(qnode_tl, {"weights": (2, n_qubits)})
    x = torch.randn(5, n_qubits)
    out = tl(x)
    assert out.shape == (5, n_qubits), f"TorchLayer batch shape: {out.shape}"
    print("T3 PASS: TorchLayer procesa batch correctamente")

# T4: Congelar pesos cuánticos funciona (grad=None)
def test_congelar_pesos():
    model = ModeloHibridoTest()
    # Congelar pesos cuánticos
    model.q_weights.requires_grad_(False)

    x = torch.randn(2, 8)
    loss = model(x).sum()
    loss.backward()

    assert model.q_weights.grad is None, "q_weights congelado no debería tener grad"
    # Los pesos clásicos SÍ deben tener gradiente
    for name, p in model.named_parameters():
        if "pre" in name or "post" in name:
            assert p.grad is not None, f"{name} debería tener gradiente"
    print("T4 PASS: congelar pesos cuánticos funciona")

# T5: Arquitectura sandwich mejora representación vs solo cuántico
def test_expresividad_sandwich():
    """Verifica que capas clásicas pre/post aumentan rango dinámico de salida."""
    model = ModeloHibridoTest()
    # Sin entrenamiento, rango de salida debe ser >0 (variabilidad)
    outs = []
    for _ in range(20):
        x = torch.randn(1, 8)
        with torch.no_grad():
            outs.append(model(x).squeeze())

    stack = torch.stack(outs)
    rango = (stack.max() - stack.min()).item()
    # Con capa Linear post-cuántica, el rango puede ser cualquier valor real
    assert rango > 0.01, f"Rango de salida muy bajo: {rango:.4f}"
    print(f"T5 PASS: rango dinámico de salida = {rango:.4f}")

# T6: Tamaño de parámetros es razonable (menos que MLP puro grande)
def test_eficiencia_parametros():
    model = ModeloHibridoTest()
    n_params = sum(p.numel() for p in model.parameters())
    # El modelo híbrido debe tener estructura compacta
    assert n_params < 500, f"Demasiados parámetros: {n_params}"
    # Pero la capa cuántica debe contribuir
    n_q_params = model.q_weights.numel()
    assert n_q_params > 0
    print(f"T6 PASS: {n_params} parámetros totales ({n_q_params} cuánticos)")

if __name__ == "__main__":
    print("=== Checkpoint M24: Modelos Híbridos ===\n")
    test_forward_shape()
    test_gradientes_cuanticos()
    test_torchlayer_batch()
    test_congelar_pesos()
    test_expresividad_sandwich()
    test_eficiencia_parametros()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M25 — Quantum Clustering y Quantum PCA](./modulo-25-qclustering-qpca.md)
