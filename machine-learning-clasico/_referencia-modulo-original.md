# Tutorial de Computación Cuántica
## Módulo 1 — Machine Learning Clásico: Fundamentos para el Salto Cuántico

> **Objetivo:** Dominar los conceptos de ML clásico que tienen equivalente directo en computación cuántica. Este módulo no pretende enseñar ML completo — solo los bloques que reaparecerán transformados en los módulos siguientes.
>
> **Enfoque quirúrgico:** Cada sección termina con un recuadro **"¿Por qué importa en cuántica?"** que conecta el concepto clásico con su contraparte cuántica.
>
> **Checkpoint final:** Clasificador binario funcional en NumPy puro + versión PyTorch. Servirá de baseline cuando comparemos con el clasificador cuántico del Módulo 6.

---

## 1.1 Setup del Entorno

### Python y gestor de entornos

Esta serie usa Python 3.11. Se recomienda `conda` porque los frameworks cuánticos (Qiskit, PennyLane, Cirq) tienen dependencias nativas que `conda` resuelve mejor que `pip` solo.

**Instalar Miniconda (si no lo tienes):**

```bash
# Linux
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

# macOS (Apple Silicon)
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh
bash Miniconda3-latest-MacOSX-arm64.sh
```

**Crear el entorno del tutorial:**

```bash
conda create -n quantum-tutorial python=3.11 -y
conda activate quantum-tutorial
```

Usaremos este mismo entorno en todos los módulos. En el Módulo 3 agregaremos los frameworks cuánticos.

### Dependencias del Módulo 1

```bash
pip install numpy scipy matplotlib scikit-learn torch torchvision jupyter ipykernel
```

**Registrar el entorno en Jupyter:**

```bash
python -m ipykernel install --user --name quantum-tutorial --display-name "Quantum Tutorial"
jupyter notebook
```

### Verificar instalación

```python
# verification.py
import numpy as np
import torch
import sklearn
import matplotlib

print(f"NumPy:      {np.__version__}")
print(f"PyTorch:    {torch.__version__}")
print(f"Scikit:     {sklearn.__version__}")
print(f"Matplotlib: {matplotlib.__version__}")
print("✓ Entorno listo")
```

```bash
python verification.py
# NumPy:      1.26.x
# PyTorch:    2.x.x
# Scikit:     1.4.x
# Matplotlib: 3.8.x
# ✓ Entorno listo
```

### Tabla de versiones

| Librería | Versión mínima | Rol en este módulo |
|---|---|---|
| Python | 3.11 | Lenguaje base |
| NumPy | 1.26 | Álgebra lineal, vectores de estado |
| PyTorch | 2.0 | Redes neuronales, gradientes |
| Scikit-learn | 1.4 | Datasets, métricas |
| Matplotlib | 3.8 | Visualizaciones |
| Jupyter | 7.x | Entorno interactivo |

---

## 1.2 Álgebra Lineal — Solo lo que Reaparece en Cuántica

El formalismo matemático de la mecánica cuántica es álgebra lineal sobre números complejos. Cada concepto aquí tiene un nombre cuántico que veremos en el Módulo 2.

### Vectores y representación de estado

Un vector es la forma de representar el **estado** de un sistema. En ML clásico un estado es un punto en el espacio de características. En cuántica, un estado es la descripción completa de un qubit.

```python
import numpy as np

# Vector de características clásico — un ejemplo con 3 features
x = np.array([0.5, 0.3, 0.8])

# Normalización L2 — crítica en cuántica (los vectores cuánticos son siempre unitarios)
x_norm = x / np.linalg.norm(x)
print(f"Vector original:     {x}")
print(f"Norma:               {np.linalg.norm(x):.4f}")
print(f"Vector normalizado:  {x_norm}")
print(f"Norma tras normalizar: {np.linalg.norm(x_norm):.4f}")  # siempre 1.0
```

> **¿Por qué importa en cuántica?**
> Los vectores de estado cuántico (|ψ⟩) son **siempre** de norma 1. Cuando codifiques datos clásicos en qubits (Módulo 5), la primera operación será siempre normalizar. Un vector de características normalizado en ML clásico y un vector de estado cuántico son matemáticamente la misma estructura.

### Producto interno (dot product)

```python
# Dos vectores de ejemplo
a = np.array([1.0, 0.0, 0.0])  # clase A
b = np.array([0.0, 1.0, 0.0])  # clase B
c = np.array([0.8, 0.6, 0.0])  # ejemplo nuevo

# Producto interno — mide "similitud"
print(f"⟨a|c⟩ = {np.dot(a, c):.4f}")  # 0.8 — c es similar a a
print(f"⟨b|c⟩ = {np.dot(b, c):.4f}")  # 0.6 — c tiene algo de b
print(f"⟨a|b⟩ = {np.dot(a, b):.4f}")  # 0.0 — a y b son ortogonales

# El cuadrado del producto interno es la probabilidad de "colapsar" a ese estado
prob_a = np.dot(a, c) ** 2
prob_b = np.dot(b, c) ** 2
print(f"\nProbabilidad de medir 'a': {prob_a:.4f}")  # 0.64
print(f"Probabilidad de medir 'b': {prob_b:.4f}")  # 0.36
print(f"Suma de probabilidades:    {prob_a + prob_b:.4f}")  # 1.0
```

> **¿Por qué importa en cuántica?**
> La notación de Dirac ⟨ψ|φ⟩ es exactamente el producto interno. La **Born rule** dice que la probabilidad de medir un estado es |⟨ψ|φ⟩|². Acabas de calcular probabilidades cuánticas con NumPy.

### Matrices de transformación y unitariedad

```python
# Transformación lineal clásica
A = np.array([[0.8, 0.2],
              [0.1, 0.9]])

v = np.array([1.0, 0.0])
v_transformado = A @ v
print(f"Vector original:     {v}")
print(f"Vector transformado: {v_transformado}")

# Matrices unitarias — la propiedad más importante en cuántica
# U es unitaria si U @ U† = I (donde U† es la conjugada transpuesta)
def es_unitaria(M, tol=1e-10):
    producto = M @ M.conj().T
    identidad = np.eye(M.shape[0])
    return np.allclose(producto, identidad, atol=tol)

# Puerta Hadamard — la más importante en cuántica
H = np.array([[1,  1],
              [1, -1]]) / np.sqrt(2)

print(f"\nPuerta Hadamard:")
print(H)
print(f"¿Es unitaria? {es_unitaria(H)}")  # True

# Puerta Pauli-X (equivalente cuántico de NOT)
X = np.array([[0, 1],
              [1, 0]])
print(f"\nPuerta Pauli-X (NOT cuántico):")
print(X)
print(f"¿Es unitaria? {es_unitaria(X)}")  # True
```

> **¿Por qué importa en cuántica?**
> Todas las operaciones cuánticas son matrices **unitarias**. La unitariedad garantiza que la norma del vector de estado se preserve (probabilidades siempre suman 1). La "capa" de una red neuronal clásica multiplica por una matriz arbitraria; la "puerta" cuántica multiplica por una matriz unitaria.

### Autovalores y autovectores

```python
# PCA clásico usa autovectores de la matriz de covarianza
from numpy.linalg import eig, eigh

# Matriz de covarianza simétrica (común en ML)
C = np.array([[3.0, 1.5],
              [1.5, 2.0]])

autovalores, autovectores = eigh(C)  # eigh para matrices hermitianas/simétricas
print("Autovalores:", autovalores)
print("Autovectores:")
print(autovectores)

# La dirección de mayor varianza es el autovector con mayor autovalor
idx_max = np.argmax(autovalores)
print(f"\nPrimera componente principal: {autovectores[:, idx_max]}")
```

> **¿Por qué importa en cuántica?**
> El **Hamiltoniano** cuántico es una matriz hermitiana. Sus autovalores son los niveles de energía del sistema. El algoritmo VQE (Módulo 5) busca el autovalor mínimo del Hamiltoniano — el estado de menor energía. PCA y VQE usan la misma operación matemática con diferente interpretación física.

---

## 1.3 Regresión — Gradiente Descendente desde Cero

El gradiente descendente es el corazón del entrenamiento en ML. En cuántica existe el equivalente exacto: la **parameter shift rule** que calcula gradientes de circuitos cuánticos. Entender el gradiente clásico hace trivial entender el cuántico.

### Regresión lineal con NumPy puro

```python
import numpy as np
import matplotlib.pyplot as plt

# Dataset sintético
np.random.seed(42)
N = 100
X = np.random.randn(N, 1)          # features
y = 2.5 * X.squeeze() + 0.8 + np.random.randn(N) * 0.5  # target con ruido

# Modelo: ŷ = w*x + b
# Parámetros iniciales
w = np.random.randn()
b = np.random.randn()

# Hiperparámetros
lr = 0.01
epochs = 200
losses = []

for epoch in range(epochs):
    # Forward pass
    y_pred = w * X.squeeze() + b

    # Pérdida MSE
    loss = np.mean((y_pred - y) ** 2)
    losses.append(loss)

    # Gradientes (derivada analítica)
    dL_dw = 2 * np.mean((y_pred - y) * X.squeeze())
    dL_db = 2 * np.mean(y_pred - y)

    # Actualización de parámetros
    w -= lr * dL_dw
    b -= lr * dL_db

    if epoch % 50 == 0:
        print(f"Epoch {epoch:3d} | Loss: {loss:.4f} | w={w:.4f} | b={b:.4f}")

print(f"\nParámetros finales: w={w:.4f} (real: 2.5), b={b:.4f} (real: 0.8)")
```

**Visualizar el entrenamiento:**

```python
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))

ax1.scatter(X, y, alpha=0.5, label='Datos')
ax1.plot(X, w * X + b, color='red', label=f'Ajuste: y={w:.2f}x+{b:.2f}')
ax1.set_title('Regresión Lineal')
ax1.legend()

ax2.plot(losses)
ax2.set_title('Curva de pérdida')
ax2.set_xlabel('Epoch')
ax2.set_ylabel('MSE Loss')

plt.tight_layout()
plt.savefig('regresion_clasica.png', dpi=150)
plt.show()
```

> **¿Por qué importa en cuántica?**
> Los parámetros `w` y `b` son ángulos de rotación en un circuito cuántico. El bucle de entrenamiento es idéntico. La única diferencia es cómo se calcula el gradiente: en clásico usamos derivadas analíticas o autograd; en cuántico usamos la **parameter shift rule**: `∂⟨H⟩/∂θ = [⟨H⟩(θ+π/2) - ⟨H⟩(θ-π/2)] / 2`.

---

## 1.4 Clasificación Binaria — El Modelo que Llevaremos a Cuántica

El clasificador binario es el caballo de batalla del QML. En el Módulo 6 reemplazaremos la capa densa por un circuito cuántico; todo lo demás permanece igual.

### Dataset

```python
from sklearn.datasets import make_moons
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
import numpy as np

# make_moons es separable pero no linealmente — buen test para circuitos cuánticos
X, y = make_moons(n_samples=200, noise=0.2, random_state=42)

# Normalizar — en cuántica los datos deben estar en [-π, π] para codificarse como ángulos
scaler = StandardScaler()
X = scaler.fit_transform(X)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

print(f"Train: {X_train.shape}, Test: {X_test.shape}")
print(f"Features min/max: {X.min():.2f} / {X.max():.2f}")
```

### Clasificador NumPy puro (sin frameworks)

```python
import numpy as np

class ClasificadorBinario:
    """
    Red neuronal de una capa oculta implementada en NumPy.
    Arquitectura: 2 → 4 → 1
    """
    def __init__(self, n_input=2, n_hidden=4, lr=0.05):
        self.lr = lr
        # Inicialización Xavier
        self.W1 = np.random.randn(n_input, n_hidden) * np.sqrt(2 / n_input)
        self.b1 = np.zeros(n_hidden)
        self.W2 = np.random.randn(n_hidden, 1) * np.sqrt(2 / n_hidden)
        self.b2 = np.zeros(1)

    def sigmoid(self, z):
        return 1 / (1 + np.exp(-np.clip(z, -500, 500)))

    def relu(self, z):
        return np.maximum(0, z)

    def forward(self, X):
        self.z1 = X @ self.W1 + self.b1
        self.a1 = self.relu(self.z1)
        self.z2 = self.a1 @ self.W2 + self.b2
        self.a2 = self.sigmoid(self.z2)
        return self.a2

    def loss(self, y_pred, y_true):
        # Binary cross-entropy
        eps = 1e-8
        y_true = y_true.reshape(-1, 1)
        return -np.mean(y_true * np.log(y_pred + eps) +
                        (1 - y_true) * np.log(1 - y_pred + eps))

    def backward(self, X, y_true):
        m = X.shape[0]
        y_true = y_true.reshape(-1, 1)

        # Gradientes capa de salida
        dL_da2 = (self.a2 - y_true) / m
        dL_dW2 = self.a1.T @ dL_da2
        dL_db2 = dL_da2.sum(axis=0)

        # Gradientes capa oculta
        dL_da1 = dL_da2 @ self.W2.T
        dL_dz1 = dL_da1 * (self.z1 > 0)  # derivada de ReLU
        dL_dW1 = X.T @ dL_dz1
        dL_db1 = dL_dz1.sum(axis=0)

        # Actualización
        self.W2 -= self.lr * dL_dW2
        self.b2 -= self.lr * dL_db2
        self.W1 -= self.lr * dL_dW1
        self.b1 -= self.lr * dL_db1

    def predict(self, X, threshold=0.5):
        return (self.forward(X) >= threshold).astype(int).squeeze()

    def accuracy(self, X, y):
        return np.mean(self.predict(X) == y)


# Entrenamiento
modelo = ClasificadorBinario(lr=0.05)
historial = {'loss': [], 'acc_train': [], 'acc_test': []}

for epoch in range(300):
    y_pred = modelo.forward(X_train)
    loss = modelo.loss(y_pred, y_train)
    modelo.backward(X_train, y_train)

    if epoch % 10 == 0:
        acc_train = modelo.accuracy(X_train, y_train)
        acc_test  = modelo.accuracy(X_test, y_test)
        historial['loss'].append(loss)
        historial['acc_train'].append(acc_train)
        historial['acc_test'].append(acc_test)

        if epoch % 50 == 0:
            print(f"Epoch {epoch:3d} | Loss: {loss:.4f} | "
                  f"Train Acc: {acc_train:.3f} | Test Acc: {acc_test:.3f}")

print(f"\n{'='*50}")
print(f"BASELINE CLÁSICO (NumPy)")
print(f"Accuracy final en test: {modelo.accuracy(X_test, y_test):.4f}")
print(f"{'='*50}")
print("Guardar este número — lo compararemos con el clasificador cuántico en Módulo 6")
```

### Visualizar la frontera de decisión

```python
def plot_decision_boundary(modelo, X, y, titulo='Frontera de decisión'):
    h = 0.02
    x_min, x_max = X[:, 0].min() - 0.5, X[:, 0].max() + 0.5
    y_min, y_max = X[:, 1].min() - 0.5, X[:, 1].max() + 0.5

    xx, yy = np.meshgrid(np.arange(x_min, x_max, h),
                          np.arange(y_min, y_max, h))
    Z = modelo.predict(np.c_[xx.ravel(), yy.ravel()])
    Z = Z.reshape(xx.shape)

    plt.figure(figsize=(8, 6))
    plt.contourf(xx, yy, Z, alpha=0.3, cmap='RdYlBu')
    plt.scatter(X[:, 0], X[:, 1], c=y, cmap='RdYlBu', edgecolors='k', s=40)
    plt.title(titulo)
    plt.xlabel('Feature 1')
    plt.ylabel('Feature 2')
    plt.savefig('frontera_clasica.png', dpi=150)
    plt.show()

plot_decision_boundary(modelo, X_test, y_test,
                       'Clasificador Clásico — Frontera de Decisión')
```

---

## 1.5 Clasificador PyTorch — La Versión que Adaptaremos

PyTorch es el framework de elección para modelos híbridos clásico-cuánticos. PennyLane se integra directamente con el autograd de PyTorch, por lo que la capa cuántica reemplaza `nn.Linear` sin cambiar el resto del código.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

# Convertir datos a tensores
X_train_t = torch.FloatTensor(X_train)
y_train_t = torch.FloatTensor(y_train)
X_test_t  = torch.FloatTensor(X_test)
y_test_t  = torch.FloatTensor(y_test)

dataset = TensorDataset(X_train_t, y_train_t)
loader  = DataLoader(dataset, batch_size=32, shuffle=True)


class RedClasica(nn.Module):
    """
    Arquitectura: 2 → 4 → 1
    Esta misma estructura, con la capa oculta reemplazada por un circuito
    cuántico, será el modelo híbrido del Módulo 6.
    """
    def __init__(self):
        super().__init__()
        self.capa_entrada  = nn.Linear(2, 4)  # ← se reemplaza por circuito cuántico
        self.capa_salida   = nn.Linear(4, 1)
        self.relu          = nn.ReLU()
        self.sigmoid       = nn.Sigmoid()

    def forward(self, x):
        x = self.relu(self.capa_entrada(x))  # ← en Módulo 6: qnode(x)
        x = self.sigmoid(self.capa_salida(x))
        return x.squeeze()


modelo_torch = RedClasica()
criterio     = nn.BCELoss()
optimizador  = optim.Adam(modelo_torch.parameters(), lr=0.01)

# Entrenamiento
for epoch in range(50):
    modelo_torch.train()
    for X_batch, y_batch in loader:
        optimizador.zero_grad()
        y_pred = modelo_torch(X_batch)
        loss   = criterio(y_pred, y_batch)
        loss.backward()
        optimizador.step()

    if epoch % 10 == 0:
        modelo_torch.eval()
        with torch.no_grad():
            y_pred_test = modelo_torch(X_test_t)
            acc = ((y_pred_test >= 0.5).float() == y_test_t).float().mean()
            print(f"Epoch {epoch:2d} | Acc test: {acc:.4f}")

# Guardar el modelo — baseline para comparar en Módulo 6
torch.save(modelo_torch.state_dict(), 'baseline_clasico.pt')
print("\n✓ Modelo guardado como baseline_clasico.pt")
```

> **¿Por qué importa en cuántica?**
> En el Módulo 6, `self.capa_entrada = nn.Linear(2, 4)` se reemplaza por:
> ```python
> self.capa_entrada = qml.qnode(dev, interface='torch')(circuito_cuantico)
> ```
> El resto del código — el optimizador, el loop de entrenamiento, la función de pérdida — no cambia. Esta es la promesa del **quantum-classical hybrid computing**.

---

## 1.6 Conceptos Clave — Tabla de Traducción Clásico → Cuántico

Esta tabla será tu referencia en todos los módulos siguientes.

| Concepto Clásico | Concepto Cuántico | Notación | Dónde aparece |
|---|---|---|---|
| Vector de características | Vector de estado | \|ψ⟩ | Módulos 2, 3, 5 |
| Normalización L2 | Norma unitaria | ‖ψ‖ = 1 | Módulo 5 (encoding) |
| Producto punto | Producto interno de Dirac | ⟨φ\|ψ⟩ | Módulo 2 |
| Probabilidad de clase | Born rule | P = \|⟨φ\|ψ⟩\|² | Módulo 2 |
| Transformación lineal | Puerta cuántica | U\|ψ⟩ | Módulos 3, 4 |
| Matriz de pesos | Hamiltoniano | H | Módulos 4, 5 |
| Parámetro entrenable | Ángulo de rotación | θ en Rₓ(θ) | Módulos 5, 6 |
| Gradiente descendente | Parameter shift rule | ∂⟨H⟩/∂θ | Módulo 5 |
| Capa de red neuronal | Capa de puertas cuánticas | U(θ) | Módulo 6 |
| Función de activación | Medición en base | ⟨Z⟩ | Módulos 2, 6 |
| Autovector de covarianza | Autoestado del Hamiltoniano | \|E_n⟩ | Módulo 5 (VQE) |
| Clasificador SVM | Quantum SVM (QSVM) | kernel cuántico | Módulo 6 |
| Convolución | Quantum Convolution | QCNN | Módulo 7 |

---

## ✅ Checkpoint Módulo 1

### Archivos generados

| Archivo | Descripción |
|---|---|
| `verification.py` | Verificación del entorno |
| `regresion_clasica.png` | Visualización de la regresión |
| `frontera_clasica.png` | Frontera de decisión del clasificador |
| `baseline_clasico.pt` | Pesos del modelo PyTorch guardados |

### Verificaciones numéricas

Antes de continuar, anota estos valores — los usarás para comparar en el Módulo 6:

```bash
# Ejecuta y anota los resultados
python -c "
import numpy as np
import torch
from sklearn.datasets import make_moons
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

X, y = make_moons(200, noise=0.2, random_state=42)
X = StandardScaler().fit_transform(X)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
print(f'Dataset: {X_train.shape[0]} train, {X_test.shape[0]} test')
print(f'Clases: {np.bincount(y_train)} train | {np.bincount(y_test)} test')
"
```

### Checklist

| Verificación | Esperado |
|---|---|
| `conda activate quantum-tutorial` funciona | Sin errores |
| `python verification.py` muestra las 4 versiones | ✓ Entorno listo |
| Clasificador NumPy entrena 300 epochs | Acc test > 0.85 |
| Clasificador PyTorch entrena 50 epochs | Acc test > 0.85 |
| `baseline_clasico.pt` existe en disco | Archivo creado |
| Tabla de traducción clásico→cuántico comprendida | Revisada |

### Verificar entorno completo

```bash
conda activate quantum-tutorial
python -c "import numpy, torch, sklearn, matplotlib; print('✓ Todo OK')"
```

---

## Resumen

| Concepto | Estado |
|---|---|
| Entorno Python 3.11 con conda | ✅ |
| NumPy: vectores, normas, producto interno, matrices unitarias | ✅ |
| Gradiente descendente implementado desde cero | ✅ |
| Clasificador binario en NumPy (baseline) | ✅ |
| Clasificador binario en PyTorch (para híbrido futuro) | ✅ |
| Tabla de traducción clásico → cuántico | ✅ |

**Siguiente módulo →** M2: Mecánica Cuántica para Programadores — Qubits, Superposición, Entrelazamiento, Puertas y Medición
