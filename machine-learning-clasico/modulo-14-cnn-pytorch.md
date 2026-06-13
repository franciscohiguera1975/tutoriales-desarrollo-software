# Módulo 14 — Redes Neuronales Convolucionales (CNN) con PyTorch

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M13 (ANN con PyTorch)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [¿Por qué CNN? Motivación desde imágenes](#por-qué-cnn)
3. [La capa de convolución](#la-capa-de-convolución)
4. [Pooling y stride](#pooling-y-stride)
5. [CNN completa para clasificación de imágenes](#cnn-para-clasificación)
6. [Data Augmentation](#data-augmentation)
7. [Transfer Learning con ResNet](#transfer-learning)
8. [CNN para secuencias (1D)](#cnn-1d)
9. [Visualización de lo que aprende una CNN](#visualización)
10. [Checkpoint M14](#checkpoint-m14)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
python modulo-14-cnn-pytorch.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab (recomendado — GPU disponible)

```python
# Colab tiene GPU T4 gratuita — ejecutar > Runtime > Change runtime type > GPU
import torch
print(f"PyTorch {torch.__version__} | GPU: {torch.cuda.is_available()}")
# Si hay GPU: device = torch.device('cuda')
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## ¿Por qué CNN?

```
Problema con MLP para imágenes:
  Una imagen 32×32 RGB = 3072 features por píxel
  Una imagen 224×224 RGB = 150,528 features
  → MLP trataría píxeles distantes como igualmente relacionados
  → Explotan los parámetros, no explotan la localidad espacial

CNN aprovecha dos propiedades de las imágenes:
  1. Localidad: patrones (bordes, texturas) son locales
  2. Invarianza a la traslación: un gato es un gato en cualquier posición

Operación de convolución:
  Kernel (filtro) K deslizante sobre la imagen I
  Salida (feature map) S(i,j) = Σₘ Σₙ I(i+m, j+n) × K(m,n)

  El kernel APRENDE los patrones relevantes durante el entrenamiento.
```

```python
import numpy as np
import matplotlib.pyplot as plt
import warnings, os
warnings.filterwarnings('ignore')
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import DataLoader, random_split
from torchvision.models import resnet18, ResNet18_Weights

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Dispositivo: {device}")

# ── Visualización manual de convolución ──────────────────────────────────────
fig, axes = plt.subplots(2, 4, figsize=(16, 7))

# Imagen sintética (borde vertical)
imagen = np.zeros((8, 8))
imagen[:, 3:5] = 1.0

# Filtros conocidos
filtros = {
    'Borde Vertical':   np.array([[-1,0,1],[-1,0,1],[-1,0,1]]),
    'Borde Horizontal': np.array([[-1,-1,-1],[0,0,0],[1,1,1]]),
    'Difuminar':        np.ones((3,3)) / 9,
    'Nitidez':          np.array([[0,-1,0],[-1,5,-1],[0,-1,0]]),
}

def convolucionar(img, kernel):
    """Convolución 2D manual (sin padding, stride=1)."""
    h, w = img.shape
    kh, kw = kernel.shape
    salida = np.zeros((h-kh+1, w-kw+1))
    for i in range(salida.shape[0]):
        for j in range(salida.shape[1]):
            salida[i, j] = np.sum(img[i:i+kh, j:j+kw] * kernel)
    return salida

axes[0, 0].imshow(imagen, cmap='gray', vmin=0, vmax=1)
axes[0, 0].set_title('Imagen original', fontsize=10)
axes[0, 0].axis('off')

for idx, (nombre, kernel) in enumerate(filtros.items(), 1):
    feature_map = convolucionar(imagen, kernel)
    axes[0, idx].imshow(kernel, cmap='RdBu', vmin=-1, vmax=1)
    axes[0, idx].set_title(f'Filtro: {nombre}', fontsize=9)
    axes[0, idx].axis('off')

    axes[1, idx].imshow(feature_map, cmap='gray')
    axes[1, idx].set_title(f'Feature Map', fontsize=9)
    axes[1, idx].axis('off')

axes[1, 0].axis('off')
plt.suptitle('Convolución manual con distintos filtros', fontsize=12)
plt.tight_layout()
plt.savefig('outputs/m14_convolucion.png', dpi=120)
plt.show()
```

---

## La capa de convolución

```python
# ── Parámetros de nn.Conv2d ───────────────────────────────────────────────────
# nn.Conv2d(in_channels, out_channels, kernel_size, stride=1, padding=0)

print("""
Parámetros de nn.Conv2d:
  in_channels:  número de canales de entrada (1=grayscale, 3=RGB)
  out_channels: número de filtros (= número de feature maps de salida)
  kernel_size:  tamaño del filtro (3=3×3, 5=5×5, etc.)
  stride:       paso del filtro (1=por defecto, 2=reduce a la mitad)
  padding:      relleno de ceros en los bordes
    padding=0:     la salida es más pequeña que la entrada
    padding=1:     con kernel=3, mantiene el tamaño ('same')
    padding='same' (PyTorch ≥ 1.9)

Cálculo del tamaño de salida:
  H_out = ⌊(H_in + 2×padding - kernel_size) / stride + 1⌋

Ejemplo:
  Entrada: 32×32, kernel=3, padding=1, stride=1
  H_out = ⌊(32 + 2 - 3) / 1 + 1⌋ = 32  → mismo tamaño

  Entrada: 32×32, kernel=3, padding=0, stride=1
  H_out = ⌊(32 + 0 - 3) / 1 + 1⌋ = 30  → más pequeño
""")

# Demostración con PyTorch
conv1 = nn.Conv2d(in_channels=1, out_channels=16, kernel_size=3, padding=1)
x_ejemplo = torch.randn(1, 1, 32, 32)  # batch=1, canales=1, H=32, W=32
y_ejemplo  = conv1(x_ejemplo)
print(f"Entrada:  {x_ejemplo.shape}")
print(f"Salida conv (padding=1): {y_ejemplo.shape}  ← mantiene 32×32")

conv_np = nn.Conv2d(1, 16, 3, padding=0)
print(f"Salida conv (padding=0): {conv_np(x_ejemplo).shape}  ← se reduce")

# Parámetros de la capa
n_params_conv = sum(p.numel() for p in conv1.parameters())
print(f"\nParámetros Conv2d(1, 16, 3): {n_params_conv}")
print(f"  Pesos: 16 filtros × 1 canal × 3×3 = {16 * 1 * 9} pesos")
print(f"  Bias:  16 (uno por filtro)")
```

---

## Pooling y stride

```python
# ── MaxPool2d vs AvgPool2d ───────────────────────────────────────────────────
print("""
Pooling — reducción espacial sin parámetros entrenables:

MaxPool2d(kernel_size=2, stride=2):
  Toma el MÁXIMO de cada ventana 2×2
  → preserva las activaciones más fuertes (bordes, esquinas)
  → reduce el tamaño a la mitad (default stride = kernel_size)

AvgPool2d(kernel_size=2, stride=2):
  Toma el PROMEDIO de cada ventana 2×2
  → suaviza más, pierde picos

AdaptiveAvgPool2d((1,1)):  ← muy útil!
  Reduce a 1×1 independientemente del tamaño de entrada
  → permite usar diferentes tamaños de imagen con la misma red
""")

pool  = nn.MaxPool2d(kernel_size=2, stride=2)
x_32  = torch.randn(1, 16, 32, 32)
x_16  = pool(x_32)
print(f"MaxPool2d: {x_32.shape} → {x_16.shape}")

ada_pool = nn.AdaptiveAvgPool2d((1, 1))
print(f"AdaptiveAvgPool2d: {x_16.shape} → {ada_pool(x_16).shape}")
```

---

## CNN para clasificación

```python
# ── Dataset CIFAR-10 ──────────────────────────────────────────────────────────
# 60,000 imágenes 32×32 RGB, 10 clases
transform_base = transforms.Compose([
    transforms.ToTensor(),                                # [0,255] → [0,1]
    transforms.Normalize((0.4914, 0.4822, 0.4465),       # media por canal
                          (0.2470, 0.2435, 0.2616)),      # std por canal
])

train_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=True, download=True, transform=transform_base
)
test_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=False, download=True, transform=transform_base
)

# Validación: 10% del train
n_val    = int(0.1 * len(train_dataset))
n_train  = len(train_dataset) - n_val
train_ds, val_ds = random_split(train_dataset, [n_train, n_val],
                                  generator=torch.Generator().manual_seed(42))

loader_train = DataLoader(train_ds, batch_size=128, shuffle=True,
                           num_workers=0, pin_memory=True)
loader_val   = DataLoader(val_ds,   batch_size=256, shuffle=False, num_workers=0)
loader_test  = DataLoader(test_dataset, batch_size=256, shuffle=False, num_workers=0)

clases_cifar = ['airplane','automobile','bird','cat','deer',
                 'dog','frog','horse','ship','truck']

print(f"CIFAR-10: {n_train} train | {n_val} val | {len(test_dataset)} test")

# Visualizar imágenes
fig, axes = plt.subplots(2, 5, figsize=(14, 6))
mean_c = np.array([0.4914, 0.4822, 0.4465])
std_c  = np.array([0.2470, 0.2435, 0.2616])

for (imgs, labels), axes_row in zip([next(iter(loader_test))], [axes.ravel()]):
    for i, ax in enumerate(axes_row[:10]):
        img = imgs[i].numpy().transpose(1, 2, 0)  # C,H,W → H,W,C
        img = img * std_c + mean_c  # desnormalizar
        img = np.clip(img, 0, 1)
        ax.imshow(img)
        ax.set_title(clases_cifar[labels[i]], fontsize=9)
        ax.axis('off')

plt.suptitle('CIFAR-10 — Ejemplos de imágenes')
plt.tight_layout()
plt.savefig('outputs/m14_cifar10_ejemplos.png', dpi=120)
plt.show()

# ── Arquitectura CNN (estilo VGG simplificado) ────────────────────────────────
class CNN_CIFAR(nn.Module):
    """
    CNN simple para CIFAR-10.
    Arquitectura: Conv-BN-ReLU-Pool × 3 → FC layers
    """
    def __init__(self, n_clases=10, dropout=0.4):
        super().__init__()

        self.features = nn.Sequential(
            # Bloque 1: 32×32 → 16×16
            nn.Conv2d(3, 32, kernel_size=3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(inplace=True),
            nn.Conv2d(32, 32, kernel_size=3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),
            nn.Dropout2d(0.2),

            # Bloque 2: 16×16 → 8×8
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),
            nn.Dropout2d(0.2),

            # Bloque 3: 8×8 → 4×4
            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),
        )

        self.classifier = nn.Sequential(
            nn.AdaptiveAvgPool2d((1, 1)),  # 128 × 1 × 1
            nn.Flatten(),
            nn.Linear(128, 256),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(256, n_clases),
        )

    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x


cnn = CNN_CIFAR().to(device)
n_params = sum(p.numel() for p in cnn.parameters() if p.requires_grad)
print(f"\nCNN CIFAR-10: {n_params:,} parámetros")

# Test de forma
x_test = torch.randn(4, 3, 32, 32).to(device)
y_test_out = cnn(x_test)
print(f"Forward pass: {x_test.shape} → {y_test_out.shape}")
```

### Loop de entrenamiento con scheduler

```python
# ── Entrenamiento de la CNN ───────────────────────────────────────────────────
criterion = nn.CrossEntropyLoss(label_smoothing=0.1)  # label smoothing
optimizer = optim.AdamW(cnn.parameters(), lr=1e-3, weight_decay=1e-4)
scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=30)

def eval_model(model, loader, criterion, device):
    model.eval()
    losses, corrects, total = [], 0, 0
    with torch.no_grad():
        for X_b, y_b in loader:
            X_b, y_b = X_b.to(device), y_b.to(device)
            out = model(X_b)
            losses.append(criterion(out, y_b).item())
            corrects += (out.argmax(1) == y_b).sum().item()
            total    += len(y_b)
    return np.mean(losses), corrects / total

historia = {'train_loss':[], 'val_loss':[], 'train_acc':[], 'val_acc':[]}
EPOCHS = 30  # ~5 min en CPU, ~1 min en GPU T4

print(f"\nEntrenando CNN por {EPOCHS} epochs...")
for epoch in range(EPOCHS):
    # ── Train ──────────────────────────────────────────────────────────────────
    cnn.train()
    t_losses, t_corrects, t_total = [], 0, 0
    for X_b, y_b in loader_train:
        X_b, y_b = X_b.to(device), y_b.to(device)
        optimizer.zero_grad()
        out  = cnn(X_b)
        loss = criterion(out, y_b)
        loss.backward()
        nn.utils.clip_grad_norm_(cnn.parameters(), max_norm=1.0)
        optimizer.step()
        t_losses.append(loss.item())
        t_corrects += (out.argmax(1) == y_b).sum().item()
        t_total    += len(y_b)

    scheduler.step()

    val_loss, val_acc = eval_model(cnn, loader_val, criterion, device)
    historia['train_loss'].append(np.mean(t_losses))
    historia['val_loss'].append(val_loss)
    historia['train_acc'].append(t_corrects / t_total)
    historia['val_acc'].append(val_acc)

    if (epoch + 1) % 5 == 0:
        print(f"Epoch [{epoch+1:2d}/{EPOCHS}] "
              f"Train Acc={t_corrects/t_total:.3f} "
              f"Val Acc={val_acc:.3f} "
              f"LR={scheduler.get_last_lr()[0]:.6f}")

# Evaluación final
test_loss, test_acc = eval_model(cnn, loader_test, criterion, device)
print(f"\nTest Accuracy: {test_acc:.4f} ({test_acc*100:.1f}%)")
print(f"(Baseline random: 10% | Objetivo ≥ 70%)")

# Curvas de entrenamiento
fig, axes = plt.subplots(1, 2, figsize=(13, 4))
ep = range(1, EPOCHS + 1)

axes[0].plot(ep, historia['train_loss'], 'b-', label='Train')
axes[0].plot(ep, historia['val_loss'],   'r-', label='Val')
axes[0].set_title('Pérdida'); axes[0].legend()

axes[1].plot(ep, historia['train_acc'], 'b-', label='Train')
axes[1].plot(ep, historia['val_acc'],   'r-', label='Val')
axes[1].axhline(test_acc, color='green', ls='--', label=f'Test={test_acc:.3f}')
axes[1].set_title('Accuracy'); axes[1].legend()

plt.suptitle('Entrenamiento CNN — CIFAR-10')
plt.tight_layout()
plt.savefig('outputs/m14_training_curves.png', dpi=120)
plt.show()

# Guardar modelo
torch.save(cnn.state_dict(), 'models/cnn_cifar10.pt')
print("Modelo guardado en models/cnn_cifar10.pt")
```

---

## Data Augmentation

```python
# ── Augmentación para mejorar generalización ──────────────────────────────────
transform_aug = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomCrop(32, padding=4),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    transforms.RandomRotation(10),
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465),
                          (0.2470, 0.2435, 0.2616)),
])

# Visualizar augmentaciones
train_aug_ds = torchvision.datasets.CIFAR10(
    root='./data', train=True, download=False, transform=transform_aug
)

# Mostrar la misma imagen con diferentes augmentaciones
img_original, label = train_dataset[0]

fig, axes = plt.subplots(2, 5, figsize=(14, 5))
axes[0, 0].imshow(np.clip(img_original.numpy().transpose(1,2,0) *
                           np.array([0.2470, 0.2435, 0.2616]) +
                           np.array([0.4914, 0.4822, 0.4465]), 0, 1))
axes[0, 0].set_title(f'Original: {clases_cifar[label]}')
axes[0, 0].axis('off')

aug_transform = transforms.Compose([
    transforms.ToPILImage(),
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomCrop(32, padding=4),
    transforms.ColorJitter(0.3, 0.3, 0.3, 0.1),
    transforms.ToTensor(),
])

for i in range(1, 5):
    aug_img = aug_transform(
        (img_original.numpy().transpose(1,2,0) * 255).astype(np.uint8)
    )
    axes[0, i].imshow(aug_img.numpy().transpose(1,2,0))
    axes[0, i].set_title(f'Aug #{i}')
    axes[0, i].axis('off')

for ax in axes[1]:
    ax.axis('off')

plt.suptitle('Data Augmentation — CIFAR-10')
plt.tight_layout()
plt.savefig('outputs/m14_augmentation.png', dpi=120)
plt.show()

print("""
Augmentaciones comunes por tipo de imagen:
  Fotos naturales:    HorizontalFlip, RandomCrop, ColorJitter
  Documentos/texto:   solo rotación pequeña, sin flip
  Imágenes médicas:   rotación, zoom, sin flip (dependiendo del órgano)
  Satélite:           rotación libre (360°), flip H+V

Cutout/Mixup/CutMix: técnicas avanzadas disponibles en torchvision.transforms.v2
""")
```

---

## Transfer Learning

```python
# ── Transfer Learning con ResNet18 preentrenada en ImageNet ───────────────────
print("""
Transfer Learning:
  1. Modelo preentrenado en un dataset grande (ImageNet, 1.4M imágenes, 1000 clases)
  2. Los primeros layers ya aprendieron bordes, texturas, formas generales
  3. Ajustamos solo los últimos layers a nuestro problema específico

Estrategias:
  Feature Extraction:  congela TODOS los layers, solo entrena la cabeza nueva
  Fine-tuning:         congela los primeros layers, entrena los últimos + cabeza
  Fine-tuning total:   descongelamos TODO (requiere más datos y tiempo)
""")

# ResNet18 preentrenada
modelo_pretrain = resnet18(weights=ResNet18_Weights.IMAGENET1K_V1)

# Estrategia 1: Feature Extraction — congelar todo excepto la cabeza
for param in modelo_pretrain.parameters():
    param.requires_grad = False

# Reemplazar la cabeza clasificadora
n_features_resnet = modelo_pretrain.fc.in_features
modelo_pretrain.fc = nn.Sequential(
    nn.Dropout(0.3),
    nn.Linear(n_features_resnet, 10)  # 10 clases CIFAR-10
)

n_trainable = sum(p.numel() for p in modelo_pretrain.parameters() if p.requires_grad)
n_total     = sum(p.numel() for p in modelo_pretrain.parameters())
print(f"\nResNet18 Feature Extraction:")
print(f"  Parámetros totales:       {n_total:,}")
print(f"  Parámetros entrenables:   {n_trainable:,} ({n_trainable/n_total:.1%})")

# CIFAR-10 usa 32×32 pero ResNet espera ≥ 224×224 → upsampling
transform_resnet = transforms.Compose([
    transforms.Resize(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],  # ImageNet normalization
                          std=[0.229, 0.224, 0.225])
])

# Submuestra para demo rápida (entrenamiento completo tomaría más tiempo)
from torch.utils.data import Subset
idx_sub = np.random.choice(n_train, 5000, replace=False)
train_sub = Subset(
    torchvision.datasets.CIFAR10(root='./data', train=True,
                                   download=False, transform=transform_resnet),
    idx_sub
)
loader_sub = DataLoader(train_sub, batch_size=64, shuffle=True, num_workers=0)

modelo_pretrain = modelo_pretrain.to(device)
opt_tl  = optim.Adam(modelo_pretrain.fc.parameters(), lr=1e-3)
crit_tl = nn.CrossEntropyLoss()

print("\nEntrenando ResNet18 (Feature Extraction, 5k muestras, 5 epochs)...")
for epoch in range(5):
    modelo_pretrain.train()
    corrects, total = 0, 0
    for X_b, y_b in loader_sub:
        X_b, y_b = X_b.to(device), y_b.to(device)
        opt_tl.zero_grad()
        out  = modelo_pretrain(X_b)
        loss = crit_tl(out, y_b)
        loss.backward()
        opt_tl.step()
        corrects += (out.argmax(1) == y_b).sum().item()
        total    += len(y_b)
    print(f"  Epoch {epoch+1}: Train Acc={corrects/total:.4f}")

print("""
Fine-tuning parcial (descongelar layer4):
  for name, param in modelo_pretrain.named_parameters():
      if 'layer4' in name or 'fc' in name:
          param.requires_grad = True

Fine-tuning total:
  for param in modelo_pretrain.parameters():
      param.requires_grad = True
  → usar learning rate muy pequeño (lr=1e-5) para no destruir pesos aprendidos
""")
```

---

## CNN 1D

```python
# ── CNN 1D para secuencias (señales, series temporales, texto) ───────────────
print("""
Conv1d — convolución sobre secuencias:
  in_channels:  dimensión de cada paso temporal (= embedding_size para texto)
  out_channels: número de filtros
  kernel_size:  tamaño de la ventana temporal

Aplicaciones:
  • Clasificación de texto (alternativa a RNN)
  • Señales de audio / ECG / sensores
  • Series temporales univariadas y multivariadas
""")

# Ejemplo: clasificación de texto con CNN1D
vocab_size = 5000
emb_dim    = 64
max_len    = 128
n_clases   = 4  # 20 Newsgroups

class TextCNN(nn.Module):
    """
    CNN para clasificación de texto.
    Inspira en "Convolutional Neural Networks for Sentence Classification" (Kim 2014)
    """
    def __init__(self, vocab_size, emb_dim, n_clases,
                 filter_sizes=(2, 3, 4, 5), n_filters=64, dropout=0.5):
        super().__init__()

        self.embedding = nn.Embedding(vocab_size, emb_dim, padding_idx=0)

        # Múltiples tamaños de kernel — capturan diferentes n-gramas
        self.convs = nn.ModuleList([
            nn.Conv1d(emb_dim, n_filters, fs) for fs in filter_sizes
        ])
        self.dropout = nn.Dropout(dropout)
        self.fc      = nn.Linear(n_filters * len(filter_sizes), n_clases)

    def forward(self, x):
        # x: (batch, seq_len)
        emb = self.embedding(x).permute(0, 2, 1)  # → (batch, emb_dim, seq_len)

        # Convolución + global max pooling por cada tamaño de kernel
        pooled = []
        for conv in self.convs:
            c = torch.relu(conv(emb))             # (batch, n_filters, L)
            p = torch.max(c, dim=2).values        # (batch, n_filters)
            pooled.append(p)

        cat = torch.cat(pooled, dim=1)            # (batch, n_filters * n_kernels)
        out = self.fc(self.dropout(cat))          # (batch, n_clases)
        return out

model_text_cnn = TextCNN(vocab_size=vocab_size, emb_dim=emb_dim, n_clases=n_clases)
n_params_text = sum(p.numel() for p in model_text_cnn.parameters())
print(f"\nTextCNN: {n_params_text:,} parámetros")

# Test de forma
x_test_text = torch.randint(0, vocab_size, (8, max_len))
y_test_text  = model_text_cnn(x_test_text)
print(f"Forward TextCNN: {x_test_text.shape} → {y_test_text.shape}")

print("""
TextCNN con múltiples kernel sizes:
  kernel=2 → captura bigramas
  kernel=3 → trigramas
  kernel=4 → 4-gramas
  kernel=5 → 5-gramas
  Global max pooling → el "mejor" n-grama de cada tipo

Rendimiento típico en clasificación de sentimiento:
  TextCNN ≈ 88-92% accuracy en SST-2
  BERT fine-tuned ≈ 93-95% (mucho más lento)
  → TextCNN es una excelente opción cuando no hay GPU
""")
```

---

## Visualización

```python
# ── Visualizar feature maps aprendidos ────────────────────────────────────────
cnn.eval()

# Capturar feature maps del primer bloque
activaciones_guardadas = {}

def hook_fn(nombre):
    def hook(module, input, output):
        activaciones_guardadas[nombre] = output.detach()
    return hook

hook1 = cnn.features[0].register_forward_hook(hook_fn('conv1'))
hook2 = cnn.features[6].register_forward_hook(hook_fn('conv2'))

# Forward pass de una imagen de test
img_sample, label_sample = test_dataset[0]
img_tensor = img_sample.unsqueeze(0).to(device)

with torch.no_grad():
    _ = cnn(img_tensor)

# Visualizar los primeros 16 feature maps del conv1
fig, axes = plt.subplots(4, 8, figsize=(18, 8))

# Imagen original
ax_orig = axes[0, 0]
img_np = img_sample.numpy().transpose(1, 2, 0)
img_np = img_np * np.array([0.2470, 0.2435, 0.2616]) + np.array([0.4914, 0.4822, 0.4465])
ax_orig.imshow(np.clip(img_np, 0, 1))
ax_orig.set_title(f'Original\n{clases_cifar[label_sample]}', fontsize=8)
ax_orig.axis('off')

# Feature maps de conv1 (32 filtros) — mostrar 16
fm1 = activaciones_guardadas['conv1'][0].cpu().numpy()
for i in range(15):
    ax = axes[0, i+1] if i < 7 else axes[1, i-7]
    ax.imshow(fm1[i], cmap='viridis')
    ax.set_title(f'FM{i+1}', fontsize=7)
    ax.axis('off')

# Feature maps de conv2
fm2 = activaciones_guardadas['conv2'][0].cpu().numpy()
for i in range(16):
    ax = axes[2, i] if i < 8 else axes[3, i-8]
    ax.imshow(fm2[i], cmap='viridis')
    ax.set_title(f'FM{i+1}', fontsize=7)
    ax.axis('off')

hook1.remove()
hook2.remove()

plt.suptitle('Feature Maps — Conv1 (fila 1-2) y Conv2 (fila 3-4)', fontsize=12)
plt.tight_layout()
plt.savefig('outputs/m14_feature_maps.png', dpi=120)
plt.show()

print("""
Los feature maps más oscuros = zonas que no activaron ese filtro
Los feature maps brillantes = zonas que activaron fuertemente el filtro

Los primeros filtros suelen detectar:
  bordes, líneas, gradientes de color

Los filtros más profundos detectan:
  patrones más abstractos: texturas, partes de objetos, objetos completos
""")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESUMEN M14 — CNN con PyTorch                            │
├───────────────────────┬─────────────────────────────────────────────────────┤
│ Componente            │ Descripción                                          │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ nn.Conv2d             │ Convolución 2D — aprende filtros espaciales          │
│ nn.BatchNorm2d        │ Normalización por batch — estabiliza training        │
│ nn.MaxPool2d          │ Reduce tamaño espacial, sin parámetros               │
│ nn.AdaptiveAvgPool2d  │ Reduce a tamaño fijo — conecta con FC layers         │
│ nn.Dropout2d          │ Apaga canales completos (más efectivo que Dropout)   │
│ CosineAnnealingLR     │ Scheduler que reduce lr suavemente                   │
│ label_smoothing=0.1   │ Suavizado de etiquetas → regularización              │
│ clip_grad_norm_       │ Previene explosión de gradientes                     │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ Data Augmentation     │ RandomFlip, RandomCrop, ColorJitter → +2-5% acc     │
│ Transfer Learning     │ Pretrain (ImageNet) → fine-tune → mucho más rápido  │
│ TextCNN (Conv1d)      │ CNN sobre texto — alternativa simple a LSTM/BERT     │
└───────────────────────┴─────────────────────────────────────────────────────┘

Arquitecturas populares a explorar:
  VGG16, ResNet50, EfficientNet (torchvision.models)
  Para detección de objetos: YOLO, Faster R-CNN
  Para segmentación: U-Net
  → Todos disponibles en torchvision y HuggingFace
```

---

## Checkpoint M14

```python
# checkpoint_m14.py
"""
Checkpoint M14 — CNN con PyTorch
Ejecutar: python checkpoint_m14.py
Nota: usa MNIST (más pequeño que CIFAR-10 → más rápido)
"""
import numpy as np, torch, torch.nn as nn, torch.optim as optim
import warnings; warnings.filterwarnings('ignore')
import torchvision, torchvision.transforms as transforms
from torch.utils.data import DataLoader, Subset
from sklearn.metrics import accuracy_score
import os

print("=" * 60)
print("CHECKPOINT M14 — CNN con PyTorch")
print("=" * 60)

device = torch.device('cpu')
torch.manual_seed(42)

# ── MNIST (28×28 grayscale, 10 dígitos) ───────────────────────────────────────
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

train_ds = torchvision.datasets.MNIST('./data', train=True,  download=True, transform=transform)
test_ds  = torchvision.datasets.MNIST('./data', train=False, download=True, transform=transform)

# Submuestra para rapidez
train_sub = Subset(train_ds, np.random.choice(len(train_ds), 5000, replace=False))
test_sub  = Subset(test_ds,  np.random.choice(len(test_ds),  1000, replace=False))

loader_tr = DataLoader(train_sub, batch_size=64, shuffle=True)
loader_te = DataLoader(test_sub,  batch_size=256, shuffle=False)

print(f"\n✓ MNIST: {len(train_sub)} train | {len(test_sub)} test")

# ── CNN simple ────────────────────────────────────────────────────────────────
class CNN_MNIST(nn.Module):
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 16, 3, padding=1),   # 28×28 → 28×28
            nn.ReLU(),
            nn.MaxPool2d(2),                   # → 14×14
            nn.Conv2d(16, 32, 3, padding=1),   # 14×14 → 14×14
            nn.ReLU(),
            nn.MaxPool2d(2),                   # → 7×7
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(32 * 7 * 7, 128),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(128, 10),
        )

    def forward(self, x):
        return self.classifier(self.features(x))

cnn = CNN_MNIST().to(device)
n_params = sum(p.numel() for p in cnn.parameters())
print(f"✓ CNN MNIST: {n_params:,} parámetros")
assert n_params < 500_000, "Demasiados parámetros para MNIST simple"

# Test forward pass
x_t = torch.randn(4, 1, 28, 28)
y_t = cnn(x_t)
assert y_t.shape == (4, 10), f"Forma incorrecta: {y_t.shape}"
print(f"✓ Forward pass: (4,1,28,28) → (4,10)")

# ── Entrenamiento ─────────────────────────────────────────────────────────────
criterion = nn.CrossEntropyLoss()
optimizer  = optim.Adam(cnn.parameters(), lr=1e-3)

for epoch in range(5):
    cnn.train()
    for X_b, y_b in loader_tr:
        optimizer.zero_grad()
        loss = criterion(cnn(X_b), y_b)
        loss.backward()
        optimizer.step()

print("✓ Entrenamiento 5 epochs: completado")

# ── Evaluación ────────────────────────────────────────────────────────────────
cnn.eval()
preds = []
with torch.no_grad():
    for X_b, _ in loader_te:
        preds.extend(cnn(X_b).argmax(1).numpy())

labels = [test_ds.targets[i].item() for i in test_sub.indices]
acc = accuracy_score(labels, preds)
print(f"✓ Test Accuracy: {acc:.4f}")
assert acc > 0.90, f"Accuracy baja para MNIST: {acc:.4f}"

# ── Verificar train/eval mode ─────────────────────────────────────────────────
cnn.train()
has_dropout = any(isinstance(m, nn.Dropout) for m in cnn.modules())
assert has_dropout, "El modelo debe tener Dropout"
print("✓ Dropout presente en el modelo")

# ── Serialización ─────────────────────────────────────────────────────────────
os.makedirs('models', exist_ok=True)
torch.save(cnn.state_dict(), 'models/checkpoint_m14_cnn.pt')
cnn_load = CNN_MNIST()
cnn_load.load_state_dict(torch.load('models/checkpoint_m14_cnn.pt', map_location='cpu'))
cnn_load.eval()

with torch.no_grad():
    X_batch, _ = next(iter(loader_te))
    p1 = cnn(X_batch).argmax(1).numpy()
    p2 = cnn_load(X_batch).argmax(1).numpy()
assert np.array_equal(p1, p2)
print("✓ Serialización CNN: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M14 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • Conv2d + MaxPool2d + Flatten + Linear")
print("  • Forward pass: formas correctas")
print("  • Entrenamiento: CrossEntropyLoss + Adam")
print("  • Test accuracy > 90% en MNIST")
print("  • Serialización con state_dict")
print("=" * 60)
```

---

*Siguiente módulo → M15: Selección de Modelos, Pipelines Avanzados y XGBoost Avanzado*
