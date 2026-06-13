# Módulo 26 — QGAN: Quantum Generative Adversarial Networks

**Objetivo:** Implementar redes generativas adversariales cuánticas donde el generador es un circuito cuántico variacional que aprende a producir distribuciones de datos. Comparar con GANs clásicas y entender el paisaje de entrenamiento.

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

## 1. Arquitectura QGAN

```
Ruido z ~ N(0,I) → Generador Cuántico G(z;θ) → Muestras ficticias
                                                        ↓
Datos reales x ~ p_data              ↓          Discriminador D(x;φ)
                                                        ↓
                               min_θ max_φ E[log D(x)] + E[log(1-D(G(z)))]
```

```python
# seccion_01_arquitectura.py
"""
QGAN: Quantum Generative Adversarial Network.
- Generador: circuito cuántico variacional G(z;θ)
  * Entrada: ruido z → codificado en ángulos de rotación
  * Salida: muestras de distribución cuántica p_θ
- Discriminador: red neuronal clásica D(x;φ)
  * Decide si x proviene de datos reales o del generador
- Entrenamiento: juego minimax alternado
"""
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np

n_qubits = 4
n_capas_gen = 3  # Profundidad del generador cuántico
latent_dim = n_qubits  # Dimensión del ruido de entrada

dev = qml.device("default.qubit", wires=n_qubits)

# === GENERADOR CUÁNTICO ===
@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def generador_qnode(z, theta):
    """
    Circuito cuántico generador.
    z:     ruido de entrada [n_qubits] — codificado en ángulos Ry
    theta: pesos variacionales [n_capas, n_qubits, 3]
    """
    # Encoding del ruido latente
    for i in range(n_qubits):
        qml.RY(z[i], wires=i)

    # Ansatz variacional (StronglyEntangling)
    qml.StronglyEntanglingLayers(theta, wires=range(n_qubits))

    # Medición: vector de valores esperados en [-1, 1]
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class GeneradorCuantico(nn.Module):
    """Generador cuántico encapsulado como módulo PyTorch."""
    def __init__(self, latent_dim, n_qubits, n_capas):
        super().__init__()
        self.latent_dim = latent_dim
        self.n_qubits   = n_qubits

        shape = qml.StronglyEntanglingLayers.shape(n_capas, n_qubits)
        self.theta = nn.Parameter(torch.randn(shape) * 0.1)

        # Capa post-cuántica: mapea [-1,1]^n_qubits → espacio de datos
        self.post = nn.Sequential(
            nn.Linear(n_qubits, 16),
            nn.Tanh(),
            nn.Linear(16, 2)  # Generar puntos 2D
        )

    def forward(self, z):
        """z: [batch, latent_dim] → x_gen: [batch, 2]"""
        z_scaled = torch.tanh(z) * np.pi  # Normalizar a [-π, π]

        q_out = torch.stack([
            torch.stack(generador_qnode(z_scaled[i], self.theta))
            for i in range(z.shape[0])
        ])  # [batch, n_qubits]

        return self.post(q_out)

# === DISCRIMINADOR CLÁSICO ===
class DiscriminadorClasico(nn.Module):
    """Discriminador puramente clásico."""
    def __init__(self, data_dim=2):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(data_dim, 64),
            nn.LeakyReLU(0.2),
            nn.Linear(64, 32),
            nn.LeakyReLU(0.2),
            nn.Linear(32, 1)  # logit (sin sigmoid — BCEWithLogitsLoss)
        )
    def forward(self, x):
        return self.red(x)

# Verificar arquitectura
gen = GeneradorCuantico(latent_dim, n_qubits, n_capas_gen)
dis = DiscriminadorClasico(data_dim=2)

z_test = torch.randn(4, latent_dim)
x_gen  = gen(z_test)
d_out  = dis(x_gen)

print("=== Arquitectura QGAN ===")
print(f"Ruido z:         {z_test.shape}")
print(f"Muestras gen:    {x_gen.shape}")
print(f"Discriminador:   {d_out.shape}")
print(f"\nParámetros generador:      {sum(p.numel() for p in gen.parameters())}")
print(f"Parámetros discriminador:  {sum(p.numel() for p in dis.parameters())}")
print(f"  - θ cuánticos: {gen.theta.numel()}")
```

---

## 2. Datos de entrenamiento

Distribuciones 2D que el QGAN debe aprender a imitar.

```python
# seccion_02_datos.py
import numpy as np
import torch
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

def distribucion_gaussiana_mixta(n_samples, n_modos=3, seed=42):
    """Mezcla de Gaussianas en 2D."""
    rng = np.random.RandomState(seed)
    angulos = np.linspace(0, 2*np.pi, n_modos, endpoint=False)
    radio = 1.5

    samples = []
    for _ in range(n_samples):
        modo = rng.randint(n_modos)
        centro = np.array([radio * np.cos(angulos[modo]),
                           radio * np.sin(angulos[modo])])
        samples.append(centro + 0.15 * rng.randn(2))

    return np.array(samples, dtype=np.float32)

def distribucion_ring(n_samples, radio=1.5, ruido=0.1, seed=42):
    """Distribución en anillo."""
    rng = np.random.RandomState(seed)
    theta = rng.uniform(0, 2*np.pi, n_samples)
    r = rng.normal(radio, ruido, n_samples)
    X = np.column_stack([r * np.cos(theta), r * np.sin(theta)])
    return X.astype(np.float32)

def distribucion_chess(n_samples, seed=42):
    """Cuadrícula tipo tablero de ajedrez."""
    rng = np.random.RandomState(seed)
    samples = []
    while len(samples) < n_samples:
        x = rng.uniform(-2, 2, 2)
        grid_x = int(np.floor(x[0]))
        grid_y = int(np.floor(x[1]))
        if (grid_x + grid_y) % 2 == 0:
            samples.append(x)
    return np.array(samples[:n_samples], dtype=np.float32)

# Normalizar datos a [-1, 1]
def normalizar(X):
    X_min = X.min(axis=0)
    X_max = X.max(axis=0)
    return 2 * (X - X_min) / (X_max - X_min + 1e-8) - 1

# Visualizar distribuciones
datasets = {
    "Gaussiana Mixta (3 modos)": distribucion_gaussiana_mixta(500),
    "Anillo":                     distribucion_ring(500),
}

fig, axes = plt.subplots(1, 2, figsize=(10, 4))
fig.suptitle("Distribuciones Objetivo para QGAN", fontsize=13)

for ax, (nombre, data) in zip(axes, datasets.items()):
    ax.scatter(data[:, 0], data[:, 1], s=8, alpha=0.6, c="steelblue")
    ax.set_title(nombre)
    ax.set_xlabel("x₁"); ax.set_ylabel("x₂")
    ax.set_aspect("equal")
    ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m26_datos.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m26_datos.png")
plt.close()
```

---

## 3. Entrenamiento QGAN

```python
# seccion_03_entrenamiento.py
import torch
import torch.nn as nn
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_01_arquitectura import GeneradorCuantico, DiscriminadorClasico
from seccion_02_datos import distribucion_gaussiana_mixta, normalizar

# Hiperparámetros
n_qubits    = 4
latent_dim  = 4
n_capas_gen = 3
batch_size  = 32
n_epochs    = 200
lr_gen      = 0.005
lr_dis      = 0.01

# Datos reales
X_real = normalizar(distribucion_gaussiana_mixta(500))
X_tensor = torch.FloatTensor(X_real)

# Modelos
generador     = GeneradorCuantico(latent_dim, n_qubits, n_capas_gen)
discriminador = DiscriminadorClasico(data_dim=2)

opt_gen = torch.optim.Adam(generador.parameters(), lr=lr_gen)
opt_dis = torch.optim.Adam(discriminador.parameters(), lr=lr_dis)
criterio = nn.BCEWithLogitsLoss()

historial = {"loss_gen": [], "loss_dis": [], "d_real": [], "d_fake": []}

print("=== Entrenando QGAN ===")
print(f"Epochs: {n_epochs}, batch: {batch_size}, lr_gen={lr_gen}, lr_dis={lr_dis}")

for epoch in range(1, n_epochs + 1):
    # Seleccionar mini-batch aleatorio
    idx = torch.randperm(len(X_tensor))[:batch_size]
    x_real = X_tensor[idx]

    # ===== Entrenar Discriminador =====
    opt_dis.zero_grad()

    # Reales: D(x_real) → 1
    d_real = discriminador(x_real)
    y_real = torch.ones_like(d_real)
    loss_real = criterio(d_real, y_real)

    # Falsos: D(G(z)) → 0
    z = torch.randn(batch_size, latent_dim)
    with torch.no_grad():
        x_fake = generador(z)
    d_fake = discriminador(x_fake)
    y_fake = torch.zeros_like(d_fake)
    loss_fake = criterio(d_fake, y_fake)

    loss_dis = (loss_real + loss_fake) / 2
    loss_dis.backward()
    opt_dis.step()

    # ===== Entrenar Generador =====
    opt_gen.zero_grad()

    z = torch.randn(batch_size, latent_dim)
    x_fake = generador(z)
    d_fake_gen = discriminador(x_fake)
    # Generador quiere que D clasifique sus muestras como reales
    y_gen = torch.ones_like(d_fake_gen)
    loss_gen = criterio(d_fake_gen, y_gen)
    loss_gen.backward()
    opt_gen.step()

    # Registrar métricas
    historial["loss_gen"].append(loss_gen.item())
    historial["loss_dis"].append(loss_dis.item())
    historial["d_real"].append(torch.sigmoid(d_real).mean().item())
    historial["d_fake"].append(torch.sigmoid(d_fake_gen).mean().item())

    if epoch % 50 == 0 or epoch == 1:
        print(f"Epoch {epoch:4d} | L_D={loss_dis.item():.4f} | "
              f"L_G={loss_gen.item():.4f} | "
              f"D(x)={historial['d_real'][-1]:.3f} | "
              f"D(G)={historial['d_fake'][-1]:.3f}")

# Guardar modelos para análisis
torch.save(generador.state_dict(),     "outputs/qgan_generador.pt")
torch.save(discriminador.state_dict(), "outputs/qgan_discriminador.pt")
print("\nModelos guardados.")

# Generar muestras finales
generador.eval()
with torch.no_grad():
    z_final = torch.randn(500, latent_dim)
    muestras_qgan = generador(z_final).numpy()

print(f"Muestras generadas: {muestras_qgan.shape}")
print(f"Rango x₁: [{muestras_qgan[:,0].min():.3f}, {muestras_qgan[:,0].max():.3f}]")
print(f"Rango x₂: [{muestras_qgan[:,1].min():.3f}, {muestras_qgan[:,1].max():.3f}]")
```

---

## 4. GAN clásica de referencia

Para comparar la QGAN con una GAN puramente clásica de similar capacidad.

```python
# seccion_04_gan_clasica.py
import torch
import torch.nn as nn
import numpy as np
from seccion_02_datos import distribucion_gaussiana_mixta, normalizar

class GeneradorClasico(nn.Module):
    def __init__(self, latent_dim=4, out_dim=2):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(latent_dim, 32),
            nn.Tanh(),
            nn.Linear(32, 16),
            nn.Tanh(),
            nn.Linear(16, out_dim)
        )
    def forward(self, z):
        return self.red(z)

class DiscriminadorClasico2(nn.Module):
    def __init__(self, data_dim=2):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(data_dim, 64),
            nn.LeakyReLU(0.2),
            nn.Linear(64, 32),
            nn.LeakyReLU(0.2),
            nn.Linear(32, 1)
        )
    def forward(self, x):
        return self.red(x)

def entrenar_gan_clasica(X_data, n_epochs=200, batch_size=32, lr=0.01, latent_dim=4):
    """Entrena GAN clásica y retorna historial + generador."""
    X_tensor = torch.FloatTensor(X_data)
    gen = GeneradorClasico(latent_dim)
    dis = DiscriminadorClasico2()
    opt_g = torch.optim.Adam(gen.parameters(), lr=lr)
    opt_d = torch.optim.Adam(dis.parameters(), lr=lr)
    crit  = nn.BCEWithLogitsLoss()

    historial = {"loss_gen": [], "loss_dis": []}

    for epoch in range(n_epochs):
        idx = torch.randperm(len(X_tensor))[:batch_size]
        x_real = X_tensor[idx]

        # Discriminador
        opt_d.zero_grad()
        d_r = dis(x_real)
        z = torch.randn(batch_size, latent_dim)
        with torch.no_grad():
            x_f = gen(z)
        d_f = dis(x_f)
        loss_d = (crit(d_r, torch.ones_like(d_r)) + crit(d_f, torch.zeros_like(d_f))) / 2
        loss_d.backward(); opt_d.step()

        # Generador
        opt_g.zero_grad()
        z = torch.randn(batch_size, latent_dim)
        x_f = gen(z)
        loss_g = crit(dis(x_f), torch.ones(batch_size, 1))
        loss_g.backward(); opt_g.step()

        historial["loss_gen"].append(loss_g.item())
        historial["loss_dis"].append(loss_d.item())

    return gen, historial

# Entrenar GAN clásica
print("Entrenando GAN clásica de referencia...")
X_data = normalizar(distribucion_gaussiana_mixta(500))
gen_cl, hist_cl = entrenar_gan_clasica(X_data, n_epochs=200)

gen_cl.eval()
with torch.no_grad():
    muestras_gan = gen_cl(torch.randn(500, 4)).numpy()

print(f"Muestras GAN clásica: {muestras_gan.shape}")
print(f"Params GAN clásica: {sum(p.numel() for p in gen_cl.parameters())}")
```

---

## 5. Evaluación con métricas de distribución

```python
# seccion_05_metricas.py
"""
Métricas para evaluar calidad de muestras generadas:
1. Maximum Mean Discrepancy (MMD): distancia entre distribuciones via kernel
2. Frechet Distance (simplificada para 2D)
3. Coverage visual
"""
import numpy as np
import torch
from sklearn.metrics import pairwise_distances

def mmd_rbf(X, Y, sigma=1.0):
    """
    Maximum Mean Discrepancy con kernel RBF.
    MMD²(P,Q) = E[k(x,x')] - 2E[k(x,y)] + E[k(y,y')]
    MMD=0 → distribuciones idénticas.
    """
    def rbf(A, B, s):
        D = pairwise_distances(A, B, metric="euclidean") ** 2
        return np.exp(-D / (2 * s**2))

    K_xx = rbf(X, X, sigma)
    K_yy = rbf(Y, Y, sigma)
    K_xy = rbf(X, Y, sigma)

    n, m = len(X), len(Y)
    # Excluir diagonal para E[k(x,x')] con x≠x'
    np.fill_diagonal(K_xx, 0)
    np.fill_diagonal(K_yy, 0)

    mmd2 = (K_xx.sum() / (n*(n-1)) +
            K_yy.sum() / (m*(m-1)) -
            2 * K_xy.mean())
    return max(0, mmd2)

def frechet_distance_2d(X, Y):
    """
    Distancia de Fréchet simplificada (asumiendo Gaussiana).
    FD = ‖μ_X - μ_Y‖² + Tr(Σ_X + Σ_Y - 2√(Σ_X Σ_Y))
    """
    mu_x, mu_y = X.mean(0), Y.mean(0)
    Σ_x = np.cov(X.T)
    Σ_y = np.cov(Y.T)

    diff = mu_x - mu_y
    producto = Σ_x @ Σ_y
    # sqrt de matriz 2x2 via eigendescomposición
    vals, vecs = np.linalg.eigh(producto)
    sqrt_prod = vecs @ np.diag(np.sqrt(np.maximum(vals, 0))) @ vecs.T

    fd = diff @ diff + np.trace(Σ_x + Σ_y - 2 * sqrt_prod)
    return max(0, fd)

# Cargar/generar muestras para evaluación
from seccion_02_datos import distribucion_gaussiana_mixta, normalizar

X_real = normalizar(distribucion_gaussiana_mixta(500, seed=0))

# Simular muestras QGAN y GAN clásica
# (En práctica: cargar modelos entrenados)
np.random.seed(42)
# Ruido gaussiano como proxy de un generador sin entrenar
X_ruido  = np.random.randn(500, 2).astype(np.float32)

# Ideal: QGAN bien entrenado ≈ distribución real
# Para demo, usamos datos con pequeño ruido
X_qgan_aprox = X_real + 0.1 * np.random.randn(*X_real.shape)
X_gan_aprox  = X_real + 0.15 * np.random.randn(*X_real.shape)

print("=== Métricas de Calidad de Generación ===")
print(f"\n{'Método':<20} {'MMD²':>10} {'FD':>10}")
print("-" * 45)

for nombre, X_gen in [
    ("Ruido puro", X_ruido),
    ("GAN clásica", X_gan_aprox),
    ("QGAN", X_qgan_aprox),
]:
    mmd = mmd_rbf(X_real, X_gen)
    fd  = frechet_distance_2d(X_real, X_gen)
    print(f"{nombre:<20} {mmd:>10.6f} {fd:>10.6f}")

print("\nInterpretación:")
print("  MMD² ≈ 0 y FD ≈ 0 → distribuciones casi idénticas")
print("  MMD² > 0.1 → diferencia significativa")
```

---

## 6. Visualización completa QGAN

```python
# seccion_06_visualizacion.py
import torch
import torch.nn as nn
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_01_arquitectura import GeneradorCuantico, DiscriminadorClasico
from seccion_02_datos import distribucion_gaussiana_mixta, normalizar
from seccion_03_entrenamiento import historial, muestras_qgan
from seccion_05_metricas import mmd_rbf, frechet_distance_2d

X_real = normalizar(distribucion_gaussiana_mixta(500))

# Cargar muestras de GAN clásica
from seccion_04_gan_clasica import muestras_gan, hist_cl

fig, axes = plt.subplots(2, 3, figsize=(15, 10))
fig.suptitle("QGAN — Entrenamiento y Evaluación", fontsize=14)

# Panel 1: Datos reales
ax = axes[0, 0]
ax.scatter(X_real[:, 0], X_real[:, 1], s=8, alpha=0.5, c="steelblue")
ax.set_title("Distribución Real (Gaussiana Mixta)")
ax.set_xlabel("x₁"); ax.set_ylabel("x₂")
ax.set_aspect("equal"); ax.grid(alpha=0.3)
ax.set_xlim(-1.5, 1.5); ax.set_ylim(-1.5, 1.5)

# Panel 2: Muestras QGAN
ax = axes[0, 1]
ax.scatter(muestras_qgan[:, 0], muestras_qgan[:, 1], s=8, alpha=0.5, c="darkorange")
mmd_q = mmd_rbf(X_real, muestras_qgan)
ax.set_title(f"QGAN (MMD²={mmd_q:.4f})")
ax.set_xlabel("x₁"); ax.set_ylabel("x₂")
ax.set_aspect("equal"); ax.grid(alpha=0.3)
ax.set_xlim(-1.5, 1.5); ax.set_ylim(-1.5, 1.5)

# Panel 3: Muestras GAN clásica
ax = axes[0, 2]
ax.scatter(muestras_gan[:, 0], muestras_gan[:, 1], s=8, alpha=0.5, c="green")
mmd_c = mmd_rbf(X_real, muestras_gan)
ax.set_title(f"GAN Clásica (MMD²={mmd_c:.4f})")
ax.set_xlabel("x₁"); ax.set_ylabel("x₂")
ax.set_aspect("equal"); ax.grid(alpha=0.3)
ax.set_xlim(-1.5, 1.5); ax.set_ylim(-1.5, 1.5)

# Panel 4: Curvas de pérdida QGAN
ax = axes[1, 0]
epochs = range(1, len(historial["loss_gen"]) + 1)
ax.plot(epochs, historial["loss_gen"], label="L_Generador",     color="darkorange")
ax.plot(epochs, historial["loss_dis"], label="L_Discriminador", color="steelblue")
ax.axhline(0.693, color="gray", linestyle="--", alpha=0.7, label="óptimo teórico (log2)")
ax.set_xlabel("Época"); ax.set_ylabel("BCELoss")
ax.set_title("Curvas de Pérdida QGAN")
ax.legend(fontsize=8); ax.grid(alpha=0.3)

# Panel 5: D(x) y D(G(z)) durante entrenamiento
ax = axes[1, 1]
ax.plot(epochs, historial["d_real"], label="D(x_real)", color="steelblue")
ax.plot(epochs, historial["d_fake"], label="D(G(z))",   color="darkorange")
ax.axhline(0.5, color="gray", linestyle="--", alpha=0.7, label="equilibrio Nash")
ax.set_xlabel("Época"); ax.set_ylabel("P(real)")
ax.set_title("Clasificación del Discriminador")
ax.legend(fontsize=8); ax.grid(alpha=0.3)
ax.set_ylim([0, 1])

# Panel 6: Comparativa final
ax = axes[1, 2]
metodos = ["QGAN", "GAN Clásica"]
mmds = [mmd_q, mmd_c]
colores = ["darkorange", "green"]
bars = ax.bar(metodos, mmds, color=colores, edgecolor="black", linewidth=0.8)
for bar, val in zip(bars, mmds):
    ax.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.0005,
            f"{val:.4f}", ha="center", va="bottom", fontweight="bold")
ax.set_ylabel("MMD² (menor es mejor)")
ax.set_title("Comparación de Calidad")
ax.grid(axis="y", alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m26_qgan.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m26_qgan.png")
plt.close()
```

---

## 7. Modo de colapso y estabilidad

```python
# seccion_07_estabilidad.py
"""
El mode collapse (colapso de modos) es el principal problema de las GANs.
En QGANs: el generador aprende a producir siempre el mismo punto.
Diagnóstico: varianza de muestras → 0.
"""
import numpy as np
import torch
from seccion_01_arquitectura import GeneradorCuantico

def diagnosticar_mode_collapse(generador, latent_dim=4, n_muestras=200):
    """
    Detecta si el generador ha colapsado.
    Señales:
    - Varianza muy baja (< 0.01) → colapso
    - Rango muy estrecho → colapso parcial
    """
    generador.eval()
    with torch.no_grad():
        z = torch.randn(n_muestras, latent_dim)
        muestras = generador(z).numpy()

    var_x1 = muestras[:, 0].var()
    var_x2 = muestras[:, 1].var()
    rango = np.ptp(muestras, axis=0)  # max - min

    print("=== Diagnóstico de Mode Collapse ===")
    print(f"Varianza x₁: {var_x1:.4f}")
    print(f"Varianza x₂: {var_x2:.4f}")
    print(f"Rango x₁:    {rango[0]:.4f}")
    print(f"Rango x₂:    {rango[1]:.4f}")

    colapso = var_x1 < 0.01 or var_x2 < 0.01
    parcial = (rango[0] < 0.5 or rango[1] < 0.5) and not colapso

    if colapso:
        print("⚠ COLAPSO DETECTADO: varianza < 0.01")
    elif parcial:
        print("⚠ Colapso parcial: rango estrecho")
    else:
        print("OK: diversidad de muestras es aceptable")

    return {"var": [var_x1, var_x2], "rango": rango.tolist(), "colapso": colapso}

# Generador no entrenado (debe tener varianza alta = sin colapso)
gen_nuevo = GeneradorCuantico(4, 4, 3)
diagnosticar_mode_collapse(gen_nuevo)

print("\n--- Técnicas anti-colapso en QGAN ---")
tecnicas = [
    ("Minibatch discrimination",
     "Agrega diversidad de batch como feature al discriminador"),
    ("Wasserstein loss",
     "Reemplaza BCE por distancia Wasserstein — más estable"),
    ("Gradient penalty",
     "Penaliza gradientes grandes del discriminador (WGAN-GP)"),
    ("Learning rate schedule",
     "Reducir lr_gen si la varianza cae bruscamente"),
    ("Parameter shift warmup",
     "Entrenar solo discriminador los primeros N epochs"),
]
for nombre, desc in tecnicas:
    print(f"  [{nombre}] {desc}")
```

---

## Checkpoint

```python
# checkpoint_m26.py
"""Tests para Módulo 26 — QGAN."""
import torch
import torch.nn as nn
import pennylane as qml
import numpy as np

n_qubits = 4
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, interface="torch", diff_method="parameter-shift")
def gen_qnode(z, theta):
    qml.AngleEmbedding(z * np.pi, wires=range(n_qubits), rotation="Y")
    qml.StronglyEntanglingLayers(theta, wires=range(n_qubits))
    return [qml.expval(qml.PauliZ(i)) for i in range(n_qubits)]

class GenTest(nn.Module):
    def __init__(self):
        super().__init__()
        shape = qml.StronglyEntanglingLayers.shape(2, n_qubits)
        self.theta = nn.Parameter(torch.randn(shape) * 0.1)
        self.post  = nn.Linear(n_qubits, 2)

    def forward(self, z):
        q = torch.stack([
            torch.stack(gen_qnode(z[i], self.theta))
            for i in range(z.shape[0])
        ])
        return self.post(q)

class DisTest(nn.Module):
    def __init__(self):
        super().__init__()
        self.red = nn.Sequential(
            nn.Linear(2, 16), nn.LeakyReLU(0.2), nn.Linear(16, 1)
        )
    def forward(self, x):
        return self.red(x)

# T1: Generador produce shape correcta [batch, 2]
def test_gen_shape():
    gen = GenTest()
    z = torch.randn(6, n_qubits)
    out = gen(z)
    assert out.shape == (6, 2), f"Shape esperada (6,2), obtenida {out.shape}"
    print("T1 PASS: shape generador correcta")

# T2: Discriminador produce shape [batch, 1]
def test_dis_shape():
    dis = DisTest()
    x = torch.randn(6, 2)
    out = dis(x)
    assert out.shape == (6, 1), f"Shape esperada (6,1), obtenida {out.shape}"
    print("T2 PASS: shape discriminador correcta")

# T3: Backprop llega a theta cuántico
def test_backprop_gen():
    gen = GenTest()
    dis = DisTest()
    crit = nn.BCEWithLogitsLoss()

    z = torch.randn(4, n_qubits)
    x_fake = gen(z)
    loss = crit(dis(x_fake), torch.ones(4, 1))
    loss.backward()

    assert gen.theta.grad is not None
    assert gen.theta.grad.norm().item() > 0
    print("T3 PASS: backprop llega a parámetros cuánticos")

# T4: Discriminador distingue real de ruido antes de entrenar
def test_discriminador_logit():
    dis = DisTest()
    x_real = torch.randn(10, 2)
    x_fake = torch.zeros(10, 2)  # trivialmente diferente
    # Ambos dan logits — sin entrenar son aleatorios, pero deben ser finitos
    d_r = dis(x_real)
    d_f = dis(x_fake)
    assert not torch.isnan(d_r).any()
    assert not torch.isnan(d_f).any()
    print("T4 PASS: discriminador produce logits finitos")

# T5: MMD entre distribución y sí misma es ~0
def test_mmd_identidad():
    from seccion_05_metricas import mmd_rbf
    X = np.random.randn(100, 2)
    mmd = mmd_rbf(X, X)
    assert mmd < 1e-8, f"MMD(X,X) debe ser ~0, obtenido {mmd:.2e}"
    print(f"T5 PASS: MMD(X,X) = {mmd:.2e} ≈ 0")

# T6: Varianza de generador no entrenado es >0 (no hay colapso inicial)
def test_sin_colapso_inicial():
    gen = GenTest()
    gen.eval()
    with torch.no_grad():
        z = torch.randn(100, n_qubits)
        muestras = gen(z).numpy()
    assert muestras[:, 0].var() > 0.01, "Varianza inicial muy baja — riesgo de colapso"
    assert muestras[:, 1].var() > 0.01
    print("T6 PASS: generador inicial sin colapso")

if __name__ == "__main__":
    print("=== Checkpoint M26: QGAN ===\n")
    test_gen_shape()
    test_dis_shape()
    test_backprop_gen()
    test_discriminador_logit()
    test_mmd_identidad()
    test_sin_colapso_inicial()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M27 — QCNN — Quantum Convolutional Neural Networks](./modulo-27-qcnn.md)
