# Módulo 23 — Quantum Neural Networks (QNN)

**Objetivo:** Implementar Quantum Neural Networks con PennyLane y Qiskit ML,
entrenar clasificadores y regresores cuánticos, comparar arquitecturas y
entender las limitaciones actuales del QNN.

**Herramientas:** PennyLane, Qiskit Machine Learning, torch, sklearn, matplotlib
**Prerequisito:** M17 (VQC), M20 (Gradientes), M21 (Encoding)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
pip install qiskit-machine-learning
python modulo-23-qnn.py
```

---

## 1. ¿Qué es un QNN?

```python
# modulo-23-qnn.py
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import pennylane as qml
import torch
import torch.nn as nn
from sklearn.datasets import make_circles, make_moons, load_iris
from sklearn.preprocessing import MinMaxScaler, LabelBinarizer
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Anatomía de un QNN
# ─────────────────────────────────────────────

print("=" * 60)
print("QUANTUM NEURAL NETWORKS (QNN)")
print("=" * 60)

print("""
Un QNN es un VQC interpretado como red neuronal:
  • Capa de entrada: encoding U_enc(x)
  • Capas ocultas: ansatz variacional U_var(θ)
  • Capa de salida: medición de observables ⟨O₁⟩,...,⟨Oₖ⟩

Comparación con redes clásicas:
  Clásica:  x → W₁·σ(·) → W₂·σ(·) → ... → salida
  QNN:      x → U_enc(x) → U_var(θ) → ⟨O⟩ → postprocesado

Tipos de QNN:
  1. QNN puro: solo capas cuánticas → salida = ⟨Zᵢ⟩
  2. QNN híbrido: capas cuánticas + clásicas alternadas
  3. QNN en PyTorch: capa cuántica como nn.Module
  4. Dressed QNN: linear → QNN → linear

Función de activación cuántica:
  El no-lineal viene del ENCODING de los datos y de la medición.
  Las rotaciones Ry(θ) son lineales en θ pero el estado resultante
  tiene amplitudes periódicas (no lineales en los datos).
""")

# Definir dispositivo
dev_qnn = qml.device("default.qubit", wires=4)
```

---

## 2. QNN básico para clasificación binaria

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: QNN para clasificación binaria
# ─────────────────────────────────────────────

print("=" * 60)
print("QNN BÁSICO — CLASIFICACIÓN BINARIA")
print("=" * 60)

N_QUBITS = 4
N_CAPAS = 3

@qml.qnode(dev_qnn, diff_method="parameter-shift", interface="torch")
def qnn_clasificador(inputs, weights):
    """
    QNN para clasificación binaria.
    inputs: features de entrada (4 valores)
    weights: parámetros variacionales [n_capas, n_qubits, 3]
    Retorna: ⟨Z₀⟩ ∈ [-1, 1] → sigmoid → P(clase=1)
    """
    # Encoding: AngleEmbedding
    qml.AngleEmbedding(inputs, wires=range(N_QUBITS), rotation="Y")

    # Ansatz variacional: StronglyEntanglingLayers
    qml.StronglyEntanglingLayers(weights, wires=range(N_QUBITS))

    return qml.expval(qml.PauliZ(0))

# Preprocesar datos
np.random.seed(42)
X_raw, y_raw = make_moons(n_samples=200, noise=0.15, random_state=42)
scaler_qnn = MinMaxScaler(feature_range=(0, np.pi))
X_qnn = scaler_qnn.fit_transform(X_raw)

# Padding a 4 features
X_qnn_4d = np.pad(X_qnn, ((0,0),(0,2)), mode="constant")  # añadir 2 columnas de 0

X_tr, X_te, y_tr, y_te = train_test_split(X_qnn_4d, y_raw, test_size=0.2, random_state=42)

# Convertir a tensores PyTorch
X_tr_t = torch.FloatTensor(X_tr)
X_te_t = torch.FloatTensor(X_te)
y_tr_t = torch.FloatTensor(y_tr)
y_te_t = torch.FloatTensor(y_te)

# Clase QNN como nn.Module
class QNNClasificador(nn.Module):
    """QNN como módulo PyTorch para clasificación binaria."""
    def __init__(self, n_qubits: int, n_capas: int):
        super().__init__()
        weight_shape = qml.StronglyEntanglingLayers.shape(n_capas, n_qubits)
        self.weights = nn.Parameter(
            torch.FloatTensor(
                np.random.uniform(0, 2*np.pi, weight_shape).astype(np.float32)
            )
        )
        self.qnode = qnn_clasificador

    def forward(self, x):
        # Ejecutar QNN para cada muestra en el batch
        outputs = []
        for xi in x:
            val = self.qnode(xi, self.weights)
            outputs.append(val)
        return torch.stack(outputs)

# Entrenar
modelo_qnn = QNNClasificador(N_QUBITS, N_CAPAS)
optimizer_qnn = torch.optim.Adam(modelo_qnn.parameters(), lr=0.05)
criterion_qnn = nn.BCEWithLogitsLoss()

print(f"QNN: {N_QUBITS} qubits, {N_CAPAS} capas")
print(f"Parámetros: {sum(p.numel() for p in modelo_qnn.parameters())}")
print(f"\nEntrenando {40} épocas:")

historial_qnn = []
for epoch in range(40):
    modelo_qnn.train()
    optimizer_qnn.zero_grad()
    pred = modelo_qnn(X_tr_t)
    loss = criterion_qnn(pred, y_tr_t)
    loss.backward()
    optimizer_qnn.step()

    if (epoch + 1) % 10 == 0:
        modelo_qnn.eval()
        with torch.no_grad():
            pred_te = modelo_qnn(X_te_t)
            pred_clases = (torch.sigmoid(pred_te) > 0.5).float()
            acc = (pred_clases == y_te_t).float().mean().item()
        historial_qnn.append({"epoch": epoch+1, "loss": loss.item(), "acc": acc})
        print(f"  Época {epoch+1:3d}: loss={loss.item():.4f}, acc_test={acc:.4f}")
```

---

## 3. QNN para clasificación multiclase

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Clasificación multiclase (Iris)
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("QNN MULTICLASE — IRIS (3 CLASES)")
print("="*60)

# Cargar Iris (4 features, 3 clases)
iris = load_iris()
X_iris = iris.data.astype(np.float32)
y_iris = iris.target

scaler_iris = MinMaxScaler(feature_range=(0, np.pi))
X_iris_sc = scaler_iris.fit_transform(X_iris)

X_i_tr, X_i_te, y_i_tr, y_i_te = train_test_split(
    X_iris_sc, y_iris, test_size=0.2, stratify=y_iris, random_state=42
)

dev_iris = qml.device("default.qubit", wires=4)

@qml.qnode(dev_iris, diff_method="parameter-shift", interface="torch")
def qnn_multiclase(inputs, weights):
    """
    QNN multiclase: mide 3 observables → softmax → 3 clases.
    Retorna [⟨Z₀⟩, ⟨Z₁⟩, ⟨Z₂⟩]
    """
    qml.AngleEmbedding(inputs, wires=range(4), rotation="Y")
    qml.StronglyEntanglingLayers(weights, wires=range(4))
    return [qml.expval(qml.PauliZ(i)) for i in range(3)]

class QNNMulticlase(nn.Module):
    def __init__(self, n_qubits: int = 4, n_capas: int = 2):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape(n_capas, n_qubits)
        self.weights = nn.Parameter(
            torch.FloatTensor(
                np.random.uniform(0, 2*np.pi, shape).astype(np.float32)
            )
        )

    def forward(self, x):
        outputs = []
        for xi in x:
            val = torch.stack(qnn_multiclase(xi, self.weights))
            outputs.append(val)
        return torch.stack(outputs)  # [batch, 3]

modelo_mc = QNNMulticlase(n_qubits=4, n_capas=2)
optimizer_mc = torch.optim.Adam(modelo_mc.parameters(), lr=0.08)
criterion_mc = nn.CrossEntropyLoss()

X_i_tr_t = torch.FloatTensor(X_i_tr)
y_i_tr_t = torch.LongTensor(y_i_tr)
X_i_te_t = torch.FloatTensor(X_i_te)

print(f"QNN Multiclase: 4 qubits, 2 capas, 3 observables de salida")
print(f"Parámetros: {sum(p.numel() for p in modelo_mc.parameters())}")

historial_mc = []
for epoch in range(50):
    modelo_mc.train()
    optimizer_mc.zero_grad()
    pred = modelo_mc(X_i_tr_t)
    loss = criterion_mc(pred, y_i_tr_t)
    loss.backward()
    optimizer_mc.step()

    if (epoch + 1) % 10 == 0:
        modelo_mc.eval()
        with torch.no_grad():
            pred_te = modelo_mc(X_i_te_t)
            clases_pred = pred_te.argmax(dim=1).numpy()
            acc = accuracy_score(y_i_te, clases_pred)
        historial_mc.append(acc)
        print(f"  Época {epoch+1:3d}: loss={loss.item():.4f}, acc_test={acc:.4f}")
```

---

## 4. Dressed QNN — capa clásica + cuántica

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Dressed QNN
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("DRESSED QNN — LINEAR + QUANTUM + LINEAR")
print("="*60)

print("""
El "Dressed QNN" (Mari et al. 2020) añade capas lineales
antes y después del QNN para mejorar expresividad.

Arquitectura:
  x → Linear(n_feat, n_qubits) → QNN(n_qubits) → Linear(n_qubits, n_out) → ŷ

Ventajas:
  ✓ Las capas clásicas adaptan la escala de los inputs/outputs
  ✓ El QNN puede ser más pequeño (menos qubits necesarios)
  ✓ Más expresivo que QNN puro de la misma profundidad
  ✓ Gradient flow más estable
""")

class DressedQNN(nn.Module):
    """
    Dressed QNN: Linear(in → n_q) + QNN(n_q) + Linear(n_q → out).
    """
    def __init__(self, n_input: int, n_qubits: int, n_capas: int, n_output: int):
        super().__init__()
        self.n_qubits = n_qubits

        # Capas clásicas pre/post
        self.pre = nn.Sequential(
            nn.Linear(n_input, n_qubits),
            nn.Tanh(),   # escalar a [-1,1], luego mapear a ángulos
        )
        self.post = nn.Linear(n_qubits, n_output)

        # Pesos cuánticos
        shape = qml.StronglyEntanglingLayers.shape(n_capas, n_qubits)
        self.q_weights = nn.Parameter(
            torch.FloatTensor(np.random.uniform(0, 2*np.pi, shape).astype(np.float32))
        )

        dev_dressed = qml.device("default.qubit", wires=n_qubits)
        wires = range(n_qubits)

        @qml.qnode(dev_dressed, interface="torch", diff_method="parameter-shift")
        def qnode(inputs, weights):
            # Escalar inputs de [-1,1] a [0, π]
            inputs_scaled = (inputs + 1) * np.pi / 2
            qml.AngleEmbedding(inputs_scaled, wires=wires, rotation="Y")
            qml.StronglyEntanglingLayers(weights, wires=wires)
            return [qml.expval(qml.PauliZ(i)) for i in wires]

        self._qnode = qnode

    def forward(self, x):
        h = self.pre(x)   # [batch, n_qubits]
        q_out = []
        for hi in h:
            vals = torch.stack(self._qnode(hi, self.q_weights))
            q_out.append(vals)
        q_out = torch.stack(q_out)   # [batch, n_qubits]
        return self.post(q_out)      # [batch, n_output]

# Entrenar Dressed QNN en Iris
modelo_dressed = DressedQNN(n_input=4, n_qubits=4, n_capas=2, n_output=3)
optimizer_d = torch.optim.Adam(modelo_dressed.parameters(), lr=0.05)

print(f"\nDressed QNN: Lin(4→4) + QNN(4q,2L) + Lin(4→3)")
print(f"Parámetros totales: {sum(p.numel() for p in modelo_dressed.parameters())}")

hist_dressed = []
for epoch in range(50):
    modelo_dressed.train()
    optimizer_d.zero_grad()
    pred = modelo_dressed(X_i_tr_t.float())
    loss = criterion_mc(pred, y_i_tr_t)
    loss.backward()
    optimizer_d.step()

    if (epoch + 1) % 10 == 0:
        modelo_dressed.eval()
        with torch.no_grad():
            pred_te = modelo_dressed(X_i_te_t.float())
            acc = accuracy_score(y_i_te, pred_te.argmax(1).numpy())
        hist_dressed.append(acc)
        print(f"  Época {epoch+1:3d}: loss={loss.item():.4f}, acc_test={acc:.4f}")
```

---

## 5. Comparación y visualización

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Visualización
# ─────────────────────────────────────────────

fig, axes = plt.subplots(1, 3, figsize=(16, 5))

# Plot 1: Convergencia QNN binario
epochs_bin = [h["epoch"] for h in historial_qnn]
accs_bin = [h["acc"] for h in historial_qnn]
axes[0].plot(epochs_bin, accs_bin, "b-o", linewidth=2, markersize=8)
axes[0].axhline(y=1.0, color="green", linestyle="--", alpha=0.5, label="Acc=1.0")
axes[0].set_ylim(0, 1.1)
axes[0].set_xlabel("Época"); axes[0].set_ylabel("Accuracy (test)")
axes[0].set_title("QNN Binario — Moons\n(4 qubits, 3 capas)")
axes[0].legend(fontsize=8); axes[0].grid(True, alpha=0.3)

# Plot 2: Convergencia QNN Iris
x_mc = [(i+1)*10 for i in range(len(historial_mc))]
x_dr = [(i+1)*10 for i in range(len(hist_dressed))]
axes[1].plot(x_mc, historial_mc, "r-s", linewidth=2, markersize=8, label="QNN puro")
axes[1].plot(x_dr, hist_dressed, "g-^", linewidth=2, markersize=8, label="Dressed QNN")
axes[1].axhline(y=1.0, color="gray", linestyle="--", alpha=0.5)
axes[1].set_ylim(0, 1.1)
axes[1].set_xlabel("Época"); axes[1].set_ylabel("Accuracy (test)")
axes[1].set_title("QNN vs Dressed QNN — Iris\n(4 qubits, 3 clases)")
axes[1].legend(fontsize=8); axes[1].grid(True, alpha=0.3)

# Plot 3: Diagrama arquitectura
ax3 = axes[2]
ax3.axis("off")
# Dibujar arquitectura visual
capas_info = [
    ("INPUT\n4 features", 0.1, 0.5, "#AED6F1"),
    ("ENCODING\nAngleEmbed", 0.3, 0.5, "#A9DFBF"),
    ("ANSATZ\nStronglyEntang", 0.5, 0.5, "#F9E79F"),
    ("MEAS\n⟨Z₀⟩,⟨Z₁⟩,⟨Z₂⟩", 0.7, 0.5, "#FAD7A0"),
    ("OUTPUT\n3 clases", 0.9, 0.5, "#D2B4DE"),
]
for texto, x, y, color in capas_info:
    ax3.add_patch(plt.Rectangle((x-0.07, y-0.12), 0.14, 0.24, color=color,
                                  transform=ax3.transAxes, clip_on=False))
    ax3.text(x, y, texto, ha="center", va="center",
            transform=ax3.transAxes, fontsize=8, fontweight="bold")

for i in range(len(capas_info)-1):
    x1 = capas_info[i][1] + 0.07
    x2 = capas_info[i+1][1] - 0.07
    y_mid = 0.5
    ax3.annotate("", xy=(x2, y_mid), xytext=(x1, y_mid),
                xycoords="axes fraction", textcoords="axes fraction",
                arrowprops=dict(arrowstyle="->", color="black", lw=2))

ax3.set_title("Arquitectura QNN\n(Multiclase, 4 qubits)", fontsize=10)

# Añadir tabla comparativa
comparativa = [
    ["Modelo", "Params", "Acc. final"],
    ["QNN puro", str(sum(p.numel() for p in modelo_mc.parameters())),
     f"{historial_mc[-1] if historial_mc else 0:.3f}"],
    ["Dressed QNN", str(sum(p.numel() for p in modelo_dressed.parameters())),
     f"{hist_dressed[-1] if hist_dressed else 0:.3f}"],
]
t = ax3.table(cellText=comparativa[1:], colLabels=comparativa[0],
              bbox=[0.0, 0.05, 1.0, 0.2], cellLoc="center")
t.auto_set_font_size(False); t.set_fontsize(8)

plt.suptitle("Quantum Neural Networks — Clasificación con PennyLane + PyTorch",
             fontsize=12, fontweight="bold")
plt.tight_layout()
plt.savefig("outputs/m23_qnn_clasificacion.png", dpi=150, bbox_inches="tight")
plt.close()
print("→ Guardado: outputs/m23_qnn_clasificacion.png")
print("\n✓ Módulo 23 completado")
```

---

## Checkpoint M23

```python
# checkpoint_m23.py
"""Verificaciones del Módulo 23 — QNN"""
import numpy as np

def test_qnn_output_range():
    """⟨Z⟩ ∈ [-1, 1] para cualquier estado cuántico."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev)
    def qnn(params):
        qml.AngleEmbedding(params[:2], wires=[0,1], rotation="Y")
        qml.CNOT([0,1])
        return qml.expval(qml.PauliZ(0))

    np.random.seed(42)
    for _ in range(20):
        p = np.random.uniform(0, 2*np.pi, 2)
        val = qnn(p)
        assert -1.01 <= val <= 1.01, f"⟨Z⟩ debe estar en [-1,1], got {val}"
    print("✓ test_qnn_output_range")

def test_qnn_gradiente_no_nulo():
    """El gradiente de un QNN bien inicializado no es cero."""
    import pennylane as qml
    dev = qml.device("default.qubit", wires=3)

    @qml.qnode(dev, diff_method="parameter-shift")
    def qnn(x, weights):
        qml.AngleEmbedding(x, wires=range(3), rotation="Y")
        qml.StronglyEntanglingLayers(weights, wires=range(3))
        return qml.expval(qml.PauliZ(0))

    x = np.array([0.5, 1.2, 0.8])
    shape = qml.StronglyEntanglingLayers.shape(2, 3)
    weights = np.random.uniform(0.1, 0.5, shape)
    grad = qml.grad(lambda w: qnn(x, w))(weights)
    assert np.any(np.abs(grad) > 1e-6), "Gradiente no debe ser cero"
    print("✓ test_qnn_gradiente_no_nulo")

def test_qnn_pytorch_backward():
    """Backward pass de PyTorch funciona en QNN cuántico."""
    import pennylane as qml
    import torch
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev, interface="torch", diff_method="parameter-shift")
    def qnode(x, w):
        qml.AngleEmbedding(x, wires=[0,1], rotation="Y")
        qml.RY(w[0], 0); qml.RY(w[1], 1)
        qml.CNOT([0,1])
        return qml.expval(qml.PauliZ(0))

    x = torch.FloatTensor([0.5, 1.2])
    w = torch.nn.Parameter(torch.FloatTensor([0.1, 0.2]))
    out = qnode(x, w)
    loss = (out - 0.5)**2
    loss.backward()
    assert w.grad is not None, "Gradiente de w debe estar disponible"
    assert abs(w.grad.sum().item()) > 1e-6, "Gradiente no debe ser cero"
    print(f"✓ test_qnn_pytorch_backward (grad={w.grad.tolist()})")

def test_strongly_entangling_shapes():
    """StronglyEntanglingLayers shape correcta."""
    import pennylane as qml
    for n_l, n_q in [(1, 2), (2, 3), (3, 4)]:
        shape = qml.StronglyEntanglingLayers.shape(n_l, n_q)
        assert shape == (n_l, n_q, 3), f"Shape esperada ({n_l},{n_q},3), got {shape}"
    print("✓ test_strongly_entangling_shapes")

def test_dressed_qnn_forward():
    """DressedQNN produce salida de forma correcta."""
    import pennylane as qml
    import torch
    import torch.nn as nn
    dev = qml.device("default.qubit", wires=2)

    @qml.qnode(dev, interface="torch")
    def qnode(x, w):
        qml.AngleEmbedding((x + 1) * np.pi / 2, wires=[0,1])
        qml.RY(w[0], 0); qml.RY(w[1], 1)
        qml.CNOT([0,1])
        return [qml.expval(qml.PauliZ(i)) for i in [0,1]]

    class SimpleQNN(nn.Module):
        def __init__(self):
            super().__init__()
            self.pre = nn.Linear(3, 2)
            self.q_w = nn.Parameter(torch.FloatTensor([0.1, 0.2]))
            self.post = nn.Linear(2, 2)
        def forward(self, x):
            h = torch.tanh(self.pre(x))
            q = torch.stack([torch.stack(qnode(hi, self.q_w)) for hi in h])
            return self.post(q)

    model = SimpleQNN()
    x = torch.randn(5, 3)
    out = model(x)
    assert out.shape == (5, 2), f"Salida debe ser (5,2), got {out.shape}"
    print(f"✓ test_dressed_qnn_forward (output shape={tuple(out.shape)})")

if __name__ == "__main__":
    print("Ejecutando checkpoint M23 — QNN\n")
    test_qnn_output_range()
    test_qnn_gradiente_no_nulo()
    test_qnn_pytorch_backward()
    test_strongly_entangling_shapes()
    test_dressed_qnn_forward()
    print("\n✓ Todos los tests de M23 pasaron")
```
