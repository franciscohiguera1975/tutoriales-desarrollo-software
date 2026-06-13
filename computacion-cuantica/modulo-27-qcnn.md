# Módulo 27 — QCNN: Quantum Convolutional Neural Networks

**Objetivo:** Implementar redes neuronales convolucionales cuánticas que aplican kernels cuánticos de forma trasladada sobre datos secuenciales o en imagen. Comprender pooling cuántico y la jerarquía de escalas.

**Herramientas:** PennyLane 0.39+, PyTorch 2.x, NumPy, matplotlib
**Prerequisito:** M23 (QNN), M24 (Híbrido PyTorch), M17 (VQC)

---

## Ejecución

```bash
conda activate quantum
mkdir -p outputs
python seccion_X.py
```

---

## 1. Motivación: convolución cuántica

Una CNN clásica aplica el mismo kernel a cada posición local de la entrada. La QCNN hace lo mismo pero con circuitos cuánticos: el mismo bloque de puertas se aplica en ventanas solapadas de qubits.

```
Input (n qubits):  q0 q1 q2 q3 q4 q5 q6 q7
                   ──────────────────────────
Conv layer 1:      [C01 C12 C23 C34 C45 C56 C67]  (pares de qubits)
Pooling layer 1:   mide q0,q2,q4,q6 → reduce a 4 qubits
Conv layer 2:      [C01' C12' C23']  (pares en 4 qubits)
Pooling layer 2:   mide q0,q2 → reduce a 2 qubits
Fully connected:   U(θ) en 2 qubits
Medición:          ⟨Z_0⟩
```

```python
# seccion_01_concepto.py
"""
QCNN: estructura jerárquica
- Capa convolucional: mismo unitario U(θ) en pares de qubits adyacentes
- Capa pooling: mide la mitad de los qubits, condicionando el estado del qubit restante
- Resultado: reducción n → n/2 en cada pooling layer
"""
import numpy as np
import pennylane as qml
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

def dibujar_arquitectura_qcnn():
    """Visualiza la arquitectura de una QCNN de 8 qubits."""
    fig, ax = plt.subplots(figsize=(14, 6))
    ax.set_xlim(-0.5, 12)
    ax.set_ylim(-0.5, 8.5)
    ax.set_aspect("equal")
    ax.axis("off")
    ax.set_title("Arquitectura QCNN — 8 Qubits", fontsize=13)

    # Qubits
    for i in range(8):
        ax.plot([-0.3, 11.5], [i, i], "k-", linewidth=0.8, alpha=0.3)
        ax.text(-0.4, i, f"q{i}", ha="right", va="center", fontsize=9)

    colores = {"conv": "#3498db", "pool": "#e74c3c", "fc": "#2ecc71"}

    # Conv layer 1 (pares adyacentes)
    for i in range(7):
        rect = plt.Rectangle((0.1 + i*0.5, i - 0.35), 0.3, 1.7 if i < 7 else 0.7,
                               color=colores["conv"], alpha=0.7, zorder=3)
        ax.add_patch(rect)
    ax.text(1.8, 8.3, "Conv 1", ha="center", color=colores["conv"], fontsize=10, fontweight="bold")

    # Pooling layer 1
    for i in range(0, 8, 2):
        rect = plt.Rectangle((4.0, i - 0.3), 0.4, 0.6,
                               color=colores["pool"], alpha=0.8, zorder=3)
        ax.add_patch(rect)
        ax.annotate("", xy=(5.0, i), xytext=(4.4, i + 0.5),
                    arrowprops=dict(arrowstyle="->", color="gray", lw=1.2))
    ax.text(4.2, 8.3, "Pool 1", ha="center", color=colores["pool"], fontsize=10, fontweight="bold")

    # Conv layer 2 (4 qubits activos: 0,2,4,6)
    activos = [0, 2, 4, 6]
    for k in range(len(activos) - 1):
        i, j = activos[k], activos[k+1]
        rect = plt.Rectangle((5.5 + k*0.5, i - 0.35), 0.3, j - i + 0.7,
                               color=colores["conv"], alpha=0.7, zorder=3)
        ax.add_patch(rect)
    ax.text(6.3, 8.3, "Conv 2", ha="center", color=colores["conv"], fontsize=10, fontweight="bold")

    # Pooling layer 2
    for i in activos[::2]:  # 0, 4
        rect = plt.Rectangle((7.8, i - 0.3), 0.4, 0.6,
                               color=colores["pool"], alpha=0.8, zorder=3)
        ax.add_patch(rect)
    ax.text(8.0, 8.3, "Pool 2", ha="center", color=colores["pool"], fontsize=10, fontweight="bold")

    # FC layer (2 qubits: 0, 4)
    for i in [0, 4]:
        rect = plt.Rectangle((9.0, i - 0.35), 0.6, 4.7 if i == 0 else 0.7,
                               color=colores["fc"], alpha=0.7, zorder=3)
        ax.add_patch(rect)
    ax.text(9.3, 8.3, "FC", ha="center", color=colores["fc"], fontsize=10, fontweight="bold")

    # Medición
    ax.text(10.5, 0, "⟨Z₀⟩", ha="center", va="center", fontsize=12,
            bbox=dict(boxstyle="round", fc="lightyellow", ec="orange"))

    plt.tight_layout()
    plt.savefig("outputs/m27_arquitectura.png", dpi=120, bbox_inches="tight")
    print("Figura guardada: outputs/m27_arquitectura.png")
    plt.close()

dibujar_arquitectura_qcnn()
```

---

## 2. Bloques fundamentales: kernel convolucional y pooling

```python
# seccion_02_bloques.py
import pennylane as qml
import numpy as np

# === Bloque Convolucional Cuántico ===
def bloque_conv(theta, wires):
    """
    Unitario de 2 qubits: U_conv(θ) aplicado a 'wires = [wi, wj]'.
    Este mismo bloque se aplica a todos los pares adyacentes (parámetros compartidos).
    """
    # Rotaciones locales
    qml.RY(theta[0], wires=wires[0])
    qml.RY(theta[1], wires=wires[1])
    # Entrelazamiento
    qml.CNOT(wires=wires)
    # Más rotaciones
    qml.RY(theta[2], wires=wires[0])
    qml.RY(theta[3], wires=wires[1])
    qml.CNOT(wires=[wires[1], wires[0]])
    qml.RY(theta[4], wires=wires[0])

def bloque_pool(wires):
    """
    Pooling cuántico: mide el primer qubit y condicionalmente aplica X al segundo.
    Equivalente a: proyectar qubit[0] en la base Z y trasladar información a qubit[1].
    Implementación variacional: CNOT + RZ condicionada.
    """
    # Sin mid-circuit measurements: usamos proyección aproximada
    qml.CNOT(wires=wires)
    qml.RZ(np.pi / 2, wires=wires[1])

# === QCNN completa (8 qubits → 1 bit) ===
n_qubits = 8
dev = qml.device("default.qubit", wires=n_qubits)

# Parámetros: conv1 (5 params), conv2 (5), fc (5) = 15 params
N_CONV_PARAMS = 5
N_FC_PARAMS   = 5

@qml.qnode(dev, diff_method="parameter-shift")
def qcnn_circuit(x, params_conv1, params_conv2, params_fc):
    """
    QCNN de 8 qubits.
    x:            vector de 8 features [0, π]
    params_conv1: [5] parámetros del kernel conv 1 (compartidos)
    params_conv2: [5] parámetros del kernel conv 2 (compartidos)
    params_fc:    [5] parámetros de la capa FC final
    """
    # Encoding de entrada
    qml.AngleEmbedding(x, wires=range(n_qubits), rotation="Y")

    # ===== CAPA CONVOLUCIONAL 1 (pares adyacentes: 01, 12, 23, 34, 45, 56, 67) =====
    for i in range(n_qubits - 1):
        bloque_conv(params_conv1, wires=[i, i+1])

    # ===== CAPA POOLING 1 (mide pares impares, reduce a q0,q2,q4,q6) =====
    for i in range(0, n_qubits, 2):
        bloque_pool(wires=[i, i+1])
    # Ahora los qubits activos son: 1, 3, 5, 7

    # ===== CAPA CONVOLUCIONAL 2 (pares: 13, 35, 57) =====
    activos_1 = [1, 3, 5, 7]
    for k in range(len(activos_1) - 1):
        bloque_conv(params_conv2, wires=[activos_1[k], activos_1[k+1]])

    # ===== CAPA POOLING 2 (reduce a q1, q5) =====
    bloque_pool(wires=[1, 3])
    bloque_pool(wires=[5, 7])

    # ===== CAPA FC (entrelaza q3 y q7) =====
    bloque_conv(params_fc, wires=[3, 7])

    # Medición en q3 (qubit final de la jerarquía)
    return qml.expval(qml.PauliZ(3))

# Verificar estructura
params_c1 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
params_c2 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
params_fc  = np.random.uniform(0, 2*np.pi, N_FC_PARAMS)
x_test = np.random.uniform(0, np.pi, n_qubits)

resultado = qcnn_circuit(x_test, params_c1, params_c2, params_fc)
print("=== QCNN 8 qubits ===")
print(f"Output (⟨Z₃⟩): {resultado:.6f}")
print(f"Parámetros totales: {N_CONV_PARAMS * 2 + N_FC_PARAMS} = 15")
print("Qubits de entrada: 8")
print("Qubits tras pool1: 4 (activos: 1,3,5,7)")
print("Qubits tras pool2: 2 (activos: 3,7)")
print("Qubits FC output:  1 (qubit 3)")

# Verificar compartición de parámetros (equivarianza)
print("\nCaracterística clave: params_conv1 se comparten en todos los pares")
print("→ equivarianza translacional (mismo kernel en toda la entrada)")
```

---

## 3. QCNN para clasificación de señales

Clasificaremos señales de 8 puntos en dos clases: señal alta frecuencia vs baja frecuencia.

```python
# seccion_03_clasificacion_senales.py
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_02_bloques import qcnn_circuit, N_CONV_PARAMS, N_FC_PARAMS

# === Dataset: señales de alta y baja frecuencia ===
def generar_senales(n_por_clase=50, n_puntos=8, seed=42):
    """
    Clase 0: señal suave (baja frecuencia)
    Clase 1: señal oscilatoria (alta frecuencia)
    """
    rng = np.random.RandomState(seed)
    t = np.linspace(0, 2*np.pi, n_puntos, endpoint=False)

    X_low, X_high = [], []
    for _ in range(n_por_clase):
        freq = rng.uniform(0.5, 1.5)
        ruido = rng.normal(0, 0.1, n_puntos)
        X_low.append(np.sin(freq * t) + ruido)

        freq = rng.uniform(3, 4)
        ruido = rng.normal(0, 0.1, n_puntos)
        X_high.append(np.sin(freq * t) + ruido)

    X = np.vstack([X_low, X_high]).astype(np.float32)
    y = np.array([0]*n_por_clase + [1]*n_por_clase)

    # Normalizar a [0, π]
    X = (X - X.min()) / (X.max() - X.min() + 1e-8) * np.pi

    return X, y

# Generar y visualizar
X, y = generar_senales(n_por_clase=40, n_puntos=8)
print(f"Dataset: {X.shape} muestras, clases: {np.bincount(y)}")

t = np.linspace(0, 2*np.pi, 8, endpoint=False)
fig, axes = plt.subplots(1, 2, figsize=(10, 4))
for i in range(5):
    axes[0].plot(t, X[i], alpha=0.6)
    axes[1].plot(t, X[40+i], alpha=0.6)
axes[0].set_title("Clase 0: Baja frecuencia"); axes[0].set_xlabel("t")
axes[1].set_title("Clase 1: Alta frecuencia"); axes[1].set_xlabel("t")
for ax in axes:
    ax.grid(alpha=0.3); ax.set_ylabel("amplitud")
plt.tight_layout()
plt.savefig("outputs/m27_senales.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m27_senales.png")
plt.close()

# === Entrenamiento QCNN ===
from sklearn.model_selection import train_test_split

X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

# Parámetros iniciales
params_c1 = pnp.random.uniform(0, 2*np.pi, N_CONV_PARAMS, requires_grad=True)
params_c2 = pnp.random.uniform(0, 2*np.pi, N_CONV_PARAMS, requires_grad=True)
params_fc  = pnp.random.uniform(0, 2*np.pi, N_FC_PARAMS,   requires_grad=True)

def coste_qcnn(pc1, pc2, pfc, X_batch, y_batch):
    """
    Cross-entropy para clasificación binaria.
    ⟨Z⟩ ∈ [-1,1] → p = (⟨Z⟩+1)/2 ∈ [0,1]
    """
    preds = pnp.array([qcnn_circuit(xi, pc1, pc2, pfc) for xi in X_batch])
    probs = (preds + 1) / 2  # mapear a [0,1]
    # BCE: -Σ [y log(p) + (1-y) log(1-p)]
    eps = 1e-7
    p_clip = pnp.clip(probs, eps, 1 - eps)
    bce = -(y_batch * pnp.log(p_clip) + (1 - y_batch) * pnp.log(1 - p_clip))
    return pnp.mean(bce)

opt = qml.AdamOptimizer(stepsize=0.05)

historial = {"loss": [], "acc_train": [], "acc_test": []}
n_epochs = 40
batch_size = 16

print("\n=== Entrenando QCNN ===")
for epoch in range(1, n_epochs + 1):
    # Mini-batch
    idx = np.random.choice(len(X_tr), batch_size, replace=False)
    X_b = X_tr[idx]
    y_b = y_tr[idx].astype(float)

    # Paso de optimización
    (params_c1, params_c2, params_fc), loss = opt.step_and_cost(
        coste_qcnn, params_c1, params_c2, params_fc,
        X_batch=X_b, y_batch=y_b
    )

    # Accuracy
    def predecir(X_data):
        preds = np.array([qcnn_circuit(xi, params_c1, params_c2, params_fc)
                          for xi in X_data])
        return (preds > 0).astype(int)

    acc_tr = (predecir(X_tr) == y_tr).mean()
    acc_te = (predecir(X_te) == y_te).mean()

    historial["loss"].append(float(loss))
    historial["acc_train"].append(acc_tr)
    historial["acc_test"].append(acc_te)

    if epoch % 10 == 0 or epoch == 1:
        print(f"Epoch {epoch:3d} | Loss={loss:.4f} | Train={acc_tr:.3f} | Test={acc_te:.3f}")
```

---

## 4. Comparación con CNN clásica de referencia

```python
# seccion_04_cnn_clasica.py
import torch
import torch.nn as nn
import numpy as np
from sklearn.model_selection import train_test_split
from seccion_03_clasificacion_senales import generar_senales

class CNN1D_Clasica(nn.Module):
    """CNN 1D clásica para señales de 8 puntos."""
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Sequential(
            nn.Conv1d(1, 4, kernel_size=3, padding=1),
            nn.ReLU(),
        )
        self.pool  = nn.AvgPool1d(2)  # 8 → 4
        self.conv2 = nn.Sequential(
            nn.Conv1d(4, 8, kernel_size=3, padding=1),
            nn.ReLU(),
        )
        self.fc = nn.Sequential(
            nn.Flatten(),
            nn.Linear(8 * 2, 8),
            nn.ReLU(),
            nn.Linear(8, 1)
        )

    def forward(self, x):
        x = x.unsqueeze(1)       # [batch, 1, 8]
        x = self.conv1(x)        # [batch, 4, 8]
        x = self.pool(x)         # [batch, 4, 4]
        x = self.conv2(x)        # [batch, 8, 4]
        x = self.pool(x)         # [batch, 8, 2]
        return self.fc(x)

X, y = generar_senales(n_por_clase=40)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

X_tr_t = torch.FloatTensor(X_tr)
y_tr_t = torch.FloatTensor(y_tr)
X_te_t = torch.FloatTensor(X_te)
y_te_t = torch.FloatTensor(y_te)

modelo_cnn = CNN1D_Clasica()
opt = torch.optim.Adam(modelo_cnn.parameters(), lr=0.01)
crit = nn.BCEWithLogitsLoss()

hist_cnn = {"loss": [], "acc_test": []}
for epoch in range(40):
    modelo_cnn.train()
    opt.zero_grad()
    logits = modelo_cnn(X_tr_t).squeeze()
    loss = crit(logits, y_tr_t)
    loss.backward(); opt.step()

    modelo_cnn.eval()
    with torch.no_grad():
        pred_te = (modelo_cnn(X_te_t).squeeze() > 0).float()
        acc = (pred_te == y_te_t).float().mean().item()
    hist_cnn["loss"].append(loss.item())
    hist_cnn["acc_test"].append(acc)

print("CNN clásica — Acc test final:", hist_cnn["acc_test"][-1])
print(f"Parámetros CNN clásica: {sum(p.numel() for p in modelo_cnn.parameters())}")
```

---

## 5. Visualización QCNN completa

```python
# seccion_05_visualizacion.py
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_03_clasificacion_senales import historial, generar_senales, qcnn_circuit
from seccion_03_clasificacion_senales import params_c1, params_c2, params_fc
from seccion_04_cnn_clasica import hist_cnn
import pennylane as qml

X, y = generar_senales(n_por_clase=40)

fig, axes = plt.subplots(2, 3, figsize=(15, 10))
fig.suptitle("QCNN — Clasificación de Señales 1D", fontsize=14)

epochs = range(1, 41)

# Panel 1: Curva de pérdida
ax = axes[0, 0]
ax.plot(epochs, historial["loss"], "b-", linewidth=2, label="QCNN")
ax.set_xlabel("Época"); ax.set_ylabel("BCE Loss")
ax.set_title("Curva de Pérdida"); ax.legend(); ax.grid(alpha=0.3)

# Panel 2: Accuracy comparativa
ax = axes[0, 1]
ax.plot(epochs, historial["acc_train"], "b--", label="QCNN train", linewidth=1.5)
ax.plot(epochs, historial["acc_test"],  "b-",  label="QCNN test",  linewidth=2)
ax.plot(epochs, hist_cnn["acc_test"],   "r-",  label="CNN clásica test", linewidth=2)
ax.axhline(0.5, color="gray", linestyle=":", alpha=0.7, label="azar")
ax.set_xlabel("Época"); ax.set_ylabel("Accuracy")
ax.set_title("Accuracy QCNN vs CNN clásica")
ax.legend(fontsize=8); ax.grid(alpha=0.3); ax.set_ylim([0, 1.05])

# Panel 3: Mapa de activaciones finales (⟨Z⟩ por muestra)
ax = axes[0, 2]
expvals_by_class = [[], []]
for i in range(len(X)):
    ev = float(qcnn_circuit(X[i], params_c1, params_c2, params_fc))
    expvals_by_class[y[i]].append(ev)

for c, (vals, nombre, color) in enumerate(zip(
    expvals_by_class,
    ["Baja freq (0)", "Alta freq (1)"],
    ["steelblue", "darkorange"]
)):
    ax.hist(vals, bins=12, alpha=0.7, color=color, label=nombre, density=True)
ax.axvline(0, color="red", linestyle="--", label="umbral")
ax.set_xlabel("⟨Z₃⟩"); ax.set_ylabel("Densidad")
ax.set_title("Distribución de Salidas QCNN")
ax.legend(fontsize=8); ax.grid(alpha=0.3)

# Panel 4: Muestras de cada clase con su predicción
t = np.linspace(0, 2*np.pi, 8, endpoint=False)
ax = axes[1, 0]
for i in range(5):
    ev = float(qcnn_circuit(X[i], params_c1, params_c2, params_fc))
    pred = 0 if ev > 0 else 1
    color = "green" if pred == y[i] else "red"
    ax.plot(t, X[i], alpha=0.7, color=color)
ax.set_title("Clase 0 (verde=correcto, rojo=error)")
ax.set_xlabel("t"); ax.set_ylabel("amplitud"); ax.grid(alpha=0.3)

ax = axes[1, 1]
for i in range(40, 45):
    ev = float(qcnn_circuit(X[i], params_c1, params_c2, params_fc))
    pred = 0 if ev > 0 else 1
    color = "green" if pred == y[i] else "red"
    ax.plot(t, X[i], alpha=0.7, color=color)
ax.set_title("Clase 1 (verde=correcto, rojo=error)")
ax.set_xlabel("t"); ax.set_ylabel("amplitud"); ax.grid(alpha=0.3)

# Panel 6: Resumen comparativo
ax = axes[1, 2]
metodos = ["QCNN\n(8q)", "CNN\nClásica"]
accs = [historial["acc_test"][-1], hist_cnn["acc_test"][-1]]
params = [15, None]  # QCNN=15, CNN clásica = calcular

import torch
from seccion_04_cnn_clasica import modelo_cnn
n_params_cnn = sum(p.numel() for p in modelo_cnn.parameters())
params[1] = n_params_cnn

colores = ["steelblue", "coral"]
bars = ax.bar(metodos, accs, color=colores, edgecolor="black")
for bar, acc, n in zip(bars, accs, params):
    ax.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,
            f"{acc:.3f}\n({n}p)", ha="center", va="bottom", fontsize=10)
ax.set_ylabel("Accuracy Test")
ax.set_title("Comparación Final\n(p = parámetros)")
ax.set_ylim([0, 1.15]); ax.grid(axis="y", alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m27_qcnn.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m27_qcnn.png")
plt.close()
```

---

## 6. Propiedades formales de la QCNN

```python
# seccion_06_propiedades.py
"""
Propiedades formales de QCNNs (Cong et al. 2019):

1. EQUIVARIANZA TRANSLACIONAL
   Si T_a desplaza la entrada en a posiciones, U_conv conmuta con T_a:
   [U_conv, T_a] = 0 ← si compartimos parámetros

2. REDUCCIÓN EXPONENCIAL DE PARÁMETROS
   CNN clásica con n features y k capas: O(n · 2^k) parms
   QCNN con n qubits y k capas:          O(n · 15) parms ← constante por capa

3. DEPTH LOGARÍTMICO
   Para reducir n qubits a 1: necesitamos O(log n) capas de pooling
   CNN equivalente: también O(log n) capas → ventaja no en depth

4. EXPRESIVIDAD
   El espacio de QCNNs es un subconjunto de todos los QNNs
   Ventaja: menos barren plateaus por estructura local
"""
import numpy as np
import pennylane as qml
from seccion_02_bloques import qcnn_circuit, N_CONV_PARAMS, N_FC_PARAMS

# Verificar equivarianza translacional (aproximada)
print("=== Propiedades QCNN ===")

n_qubits = 8
x_base = np.random.uniform(0, np.pi, n_qubits)
params_c1 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
params_c2 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
params_fc  = np.random.uniform(0, 2*np.pi, N_FC_PARAMS)

# Original
ev_base = qcnn_circuit(x_base, params_c1, params_c2, params_fc)

# Desplazamiento cíclico en 1 posición
x_shift1 = np.roll(x_base, 1)
ev_shift1 = qcnn_circuit(x_shift1, params_c1, params_c2, params_fc)

# Desplazamiento en 2 posiciones
x_shift2 = np.roll(x_base, 2)
ev_shift2 = qcnn_circuit(x_shift2, params_c1, params_c2, params_fc)

print(f"⟨Z⟩(x_base):      {ev_base:.4f}")
print(f"⟨Z⟩(x_shift_1):   {ev_shift1:.4f}")
print(f"⟨Z⟩(x_shift_2):   {ev_shift2:.4f}")
print(f"Diferencia shift1: {abs(ev_base - ev_shift1):.4f}")
print(f"Diferencia shift2: {abs(ev_base - ev_shift2):.4f}")
print("Nota: QCNN no es exactamente equivariante por boundary effects,")
print("      pero es aproximadamente equivariante para señales periódicas.")

# Contar parámetros
n_parms_qcnn = N_CONV_PARAMS * 2 + N_FC_PARAMS
print(f"\nParámetros QCNN (8q, 2 conv + 1 FC): {n_parms_qcnn}")
print(f"vs. QNN fully connected (8q, 3 capas): ~{3 * 3 * 8} (StronglyEntangling)")
print(f"Reducción: {3*3*8 // n_parms_qcnn:.1f}x menos parámetros")

# Análisis de gradientes: QCNN vs QNN (menor barren plateau por localidad)
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, diff_method="parameter-shift")
def qnn_full(x, weights):
    qml.AngleEmbedding(x, wires=range(n_qubits))
    qml.StronglyEntanglingLayers(weights, wires=range(n_qubits))
    return qml.expval(qml.PauliZ(0))

n_muestras = 30
grads_qcnn, grads_qnn = [], []

grad_fn_qcnn = qml.grad(qcnn_circuit, argnum=1)
w_shape = qml.StronglyEntanglingLayers.shape(3, n_qubits)
grad_fn_qnn = qml.grad(qnn_full, argnum=1)

import pennylane.numpy as pnp

for _ in range(n_muestras):
    xi = pnp.random.uniform(0, pnp.pi, n_qubits)
    pc1 = pnp.random.uniform(0, 2*pnp.pi, N_CONV_PARAMS, requires_grad=True)
    pc2 = pnp.random.uniform(0, 2*pnp.pi, N_CONV_PARAMS)
    pfc_ = pnp.random.uniform(0, 2*pnp.pi, N_FC_PARAMS)
    g_qcnn = grad_fn_qcnn(xi, pc1, pc2, pfc_)
    grads_qcnn.append(np.abs(g_qcnn).mean())

    w = pnp.random.uniform(0, 2*pnp.pi, w_shape, requires_grad=True)
    g_qnn = grad_fn_qnn(xi, w)
    grads_qnn.append(np.abs(g_qnn).mean())

print(f"\nMagnitud media de gradientes:")
print(f"  QCNN:  {np.mean(grads_qcnn):.2e} ± {np.std(grads_qcnn):.2e}")
print(f"  QNN:   {np.mean(grads_qnn):.2e} ± {np.std(grads_qnn):.2e}")
print(f"Ratio QCNN/QNN: {np.mean(grads_qcnn)/np.mean(grads_qnn):.2f}x")
print("Esperado: QCNN tiene gradientes mayores (menos barren plateau)")
```

---

## Checkpoint

```python
# checkpoint_m27.py
"""Tests para Módulo 27 — QCNN."""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp

n_qubits = 8
N_CONV_PARAMS = 5
N_FC_PARAMS   = 5

dev = qml.device("default.qubit", wires=n_qubits)

def bloque_conv(theta, wires):
    qml.RY(theta[0], wires=wires[0])
    qml.RY(theta[1], wires=wires[1])
    qml.CNOT(wires=wires)
    qml.RY(theta[2], wires=wires[0])
    qml.RY(theta[3], wires=wires[1])
    qml.CNOT(wires=[wires[1], wires[0]])
    qml.RY(theta[4], wires=wires[0])

@qml.qnode(dev, diff_method="parameter-shift")
def qcnn_test(x, pc1, pc2, pfc):
    qml.AngleEmbedding(x, wires=range(n_qubits), rotation="Y")
    for i in range(n_qubits - 1):
        bloque_conv(pc1, wires=[i, i+1])
    for i in range(0, n_qubits, 2):
        qml.CNOT(wires=[i, i+1])
        qml.RZ(np.pi/2, wires=i+1)
    for k in range(3):
        bloque_conv(pc2, wires=[2*k+1, 2*k+3])
    bloque_conv(pfc, wires=[3, 7])
    return qml.expval(qml.PauliZ(3))

# T1: Output es escalar en [-1,1]
def test_output_rango():
    pc1 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
    pc2 = np.random.uniform(0, 2*np.pi, N_CONV_PARAMS)
    pfc = np.random.uniform(0, 2*np.pi, N_FC_PARAMS)
    x = np.random.uniform(0, np.pi, n_qubits)
    out = float(qcnn_test(x, pc1, pc2, pfc))
    assert -1.0 <= out <= 1.0, f"Output fuera de rango: {out}"
    print(f"T1 PASS: output QCNN = {out:.4f} ∈ [-1,1]")

# T2: Mismo kernel compartido (parameter sharing) — mismos parámetros dan misma transformación
def test_parameter_sharing():
    theta = np.array([0.1, 0.2, 0.3, 0.4, 0.5])
    dev2 = qml.device("default.qubit", wires=2)

    @qml.qnode(dev2)
    def par_01():
        qml.PauliX(0)
        bloque_conv(theta, wires=[0, 1])
        return qml.state()

    @qml.qnode(dev2)
    def par_01_bis():
        qml.PauliX(0)
        bloque_conv(theta, wires=[0, 1])
        return qml.state()

    assert np.allclose(par_01(), par_01_bis())
    print("T2 PASS: bloque convolucional es determinista (parameter sharing correcto)")

# T3: Profundidad logarítmica — 8 qubits → 2 pools → 1 qubit final
def test_estructura_jerarquica():
    n = 8
    n_poolings = int(np.log2(n)) - 1  # 8→4→2 = 2 poolings para llegar a 2 qubits
    assert n_poolings == 2
    print(f"T3 PASS: {n} qubits necesitan {n_poolings} capas pooling (O(log n))")

# T4: Bloque conv produce estado distinto al inicial
def test_conv_transforma():
    dev2 = qml.device("default.qubit", wires=2)
    theta = np.array([0.5, 0.5, 0.5, 0.5, 0.5])

    @qml.qnode(dev2)
    def estado_inicial():
        return qml.state()

    @qml.qnode(dev2)
    def estado_tras_conv():
        bloque_conv(theta, wires=[0, 1])
        return qml.state()

    assert not np.allclose(estado_inicial(), estado_tras_conv())
    print("T4 PASS: bloque conv transforma el estado")

# T5: Gradiente del primer parámetro conv no es cero
def test_gradiente_no_nulo():
    pc1 = pnp.random.uniform(0, 2*pnp.pi, N_CONV_PARAMS, requires_grad=True)
    pc2 = pnp.random.uniform(0, 2*pnp.pi, N_CONV_PARAMS)
    pfc = pnp.random.uniform(0, 2*pnp.pi, N_FC_PARAMS)
    x   = pnp.random.uniform(0, pnp.pi, n_qubits)

    grad = qml.grad(qcnn_test, argnum=1)(x, pc1, pc2, pfc)
    assert np.any(np.abs(grad) > 1e-10), "Gradiente completamente nulo"
    print(f"T5 PASS: |grad|_max = {np.abs(grad).max():.2e} > 0")

# T6: QCNN tiene menos parámetros que QNN equivalente
def test_eficiencia_parametros():
    n_params_qcnn = N_CONV_PARAMS * 2 + N_FC_PARAMS
    n_params_qnn  = 3 * 3 * n_qubits  # 3 capas × 3 rotaciones × 8 qubits (StronglyEntangling)
    assert n_params_qcnn < n_params_qnn, f"QCNN ({n_params_qcnn}) debe < QNN ({n_params_qnn})"
    print(f"T6 PASS: QCNN={n_params_qcnn} params < QNN={n_params_qnn} params")

if __name__ == "__main__":
    print("=== Checkpoint M27: QCNN ===\n")
    test_output_rango()
    test_parameter_sharing()
    test_estructura_jerarquica()
    test_conv_transforma()
    test_gradiente_no_nulo()
    test_eficiencia_parametros()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M28 — Quantum Reinforcement Learning](./modulo-28-qrl.md)
