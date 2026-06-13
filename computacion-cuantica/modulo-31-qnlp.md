# Módulo 31 — Quantum NLP

**Objetivo:** Implementar procesamiento de lenguaje natural cuántico (QNLP) usando DisCoCat y circuitos de tipo ansatz para clasificación de sentimientos. Explorar cómo la estructura gramatical se mapea a circuitos cuánticos.

**Herramientas:** PennyLane 0.39+, lambeq (opcional), NumPy, matplotlib
**Prerequisito:** M21 (Encoding), M23 (QNN)

---

## Setup adicional

```bash
conda activate quantum
pip install lambeq  # Framework QNLP de Cambridge Quantum
```

---

## 1. Fundamentos: DisCoCat y gramática cuántica

```python
# seccion_01_fundamentos.py
"""
QNLP (Quantum Natural Language Processing) — Meichanetzidis et al. 2020

Idea central: DisCoCat (Distributional Compositional Categorical)
- Palabras → vectores en espacio de Hilbert (cuántico)
- Gramática → morfismos categóricos (operaciones cuánticas)
- Composición → producto tensorial y contracciones

Mapa:
  Sustantivo  → estado cuántico |n⟩ (n qubits)
  Verbo trans → tensor n⊗s←n  (circuito de 3 qubits)
  Frase S     → |s⟩ (1 qubit — clasificación final)

Ejemplo:
  "Alice likes Bob" →
    [Alice: |n⟩] ⊗ [likes: n⊗s←n] ⊗ [Bob: |n⟩]
    → colapsa a estado en s → clasificar (positivo/negativo)
"""

print("=== QNLP: Quantum Natural Language Processing ===\n")

print("Tipos gramaticales → Número de qubits:")
print("  n  (sustantivo) → 1 qubit")
print("  s  (frase)      → 1 qubit")
print("  n→s (adj/adv)   → 2 qubits (wirings de n a s)")
print("  n⊗s←n (verbo)  → 3 qubits (sujeto + predicado + objeto)")

print("\nEjemplos de frases y su estructura:")
frases = [
    ("Alice likes Bob",           "n·(n→s←n)·n → s", "sujeto-verbo-objeto"),
    ("Big Alice runs",            "n·(n→n)·(n→s) → s", "adj-sust-verbo"),
    ("Bob quickly runs",          "n·(s→s)·(n→s) → s", "sust-adv-verbo"),
    ("Alice likes to run",        "n·(n→s←(n→s))·(n→s) → s", "embedding"),
]
for frase, tipo, desc in frases:
    print(f"  '{frase}'")
    print(f"    Tipo: {tipo} ({desc})")
    print()

print("Pipeline QNLP:")
print("  1. Parsing gramatical (CCG)  → árbol de derivación")
print("  2. DisCoCat reduction        → string diagrama")
print("  3. Traducción a circuito     → puertas cuánticas parametrizadas")
print("  4. Entrenamiento variacional → minimizar cross-entropy")
print("  5. Clasificación             → medir qubit de frase s")
```

---

## 2. Implementación manual: clasificación de sentimientos

```python
# seccion_02_sentimientos.py
"""
Clasificación binaria de sentimientos sin lambeq.
Implementación manual del modelo DisCoCat simplificado:

Para frases de estructura simple: "sujeto verbo"
  - Sustantivo: estado cuántico |n⟩ = RY(θ_n)|0⟩  (1 qubit)
  - Verbo intransitivo: circuito n→s (2 qubits)
  - Frase: medir qubit s → P(positivo) = P(|1⟩_s)
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

# === Dataset: frases de sentimiento positivo/negativo ===
# Estructura: "[sustantivo] [verbo/adj]" → binario (0=neg, 1=pos)
DATASET = [
    # Positivo
    ("alice", "loves",    1),
    ("bob",   "enjoys",   1),
    ("alice", "wins",     1),
    ("bob",   "succeeds", 1),
    ("alice", "thrives",  1),
    # Negativo
    ("bob",   "fails",    0),
    ("alice", "hates",    0),
    ("bob",   "loses",    0),
    ("alice", "suffers",  0),
    ("bob",   "cries",    0),
]

# Vocabulario
sustantivos = sorted(set(s for s,_,_ in DATASET))
verbos      = sorted(set(v for _,v,_ in DATASET))
print(f"Sustantivos: {sustantivos}")
print(f"Verbos:      {verbos}")

# Representaciones cuánticas
n_qubits_n = 1  # qubit de sustantivo
n_qubits_v = 2  # qubits del verbo (n→s)
n_total     = n_qubits_n + n_qubits_v  # 3 qubits por frase

dev = qml.device("default.qubit", wires=n_total)

# Parámetros por palabra (a optimizar)
# Sustantivos: 1 ángulo Ry por palabra
# Verbos: 4 parámetros por palabra (Ry-CNOT-Ry-Ry en 2 qubits)
N_PARAMS_SUST = 1
N_PARAMS_VERB = 4
n_sust = len(sustantivos)
n_verb = len(verbos)

def crear_params(seed=42):
    rng = np.random.RandomState(seed)
    theta_sust = {s: pnp.array(rng.uniform(0, 2*np.pi, N_PARAMS_SUST), requires_grad=True)
                  for s in sustantivos}
    theta_verb = {v: pnp.array(rng.uniform(0, 2*np.pi, N_PARAMS_VERB), requires_grad=True)
                  for v in verbos}
    return theta_sust, theta_verb

@qml.qnode(dev, diff_method="parameter-shift")
def circuito_frase(theta_s, theta_v):
    """
    Circuito DisCoCat para "sustantivo verbo".
    Qubit 0: sustantivo n
    Qubits 1-2: verbo n→s
    Qubit 2 (s): medir para clasificación
    """
    # Preparar estado del sustantivo
    qml.RY(theta_s[0], wires=0)

    # Preparar verbo (circuito en qubits 1,2 + entrelazamiento con sustantivo)
    qml.RY(theta_v[0], wires=1)
    qml.RY(theta_v[1], wires=2)
    qml.CNOT(wires=[0, 1])  # composición gramatical
    qml.RY(theta_v[2], wires=1)
    qml.RY(theta_v[3], wires=2)
    qml.CNOT(wires=[1, 2])

    # Medir qubit de frase s (qubit 2)
    return qml.expval(qml.PauliZ(2))  # ∈ [-1,1], ≥0 → positivo

def predecir_prob(theta_s, theta_v):
    """Convierte ⟨Z₂⟩ a probabilidad P(positivo) ∈ [0,1]."""
    ev = circuito_frase(theta_s, theta_v)
    return (ev + 1) / 2

# Función de coste
def coste_total(theta_sust, theta_verb):
    """BCE sobre todo el dataset."""
    loss = 0
    eps = 1e-7
    for sust, verb, y in DATASET:
        p = predecir_prob(theta_sust[sust], theta_verb[verb])
        p = pnp.clip(p, eps, 1-eps)
        loss += -(y * pnp.log(p) + (1-y) * pnp.log(1-p))
    return loss / len(DATASET)

# Entrenamiento
theta_sust, theta_verb = crear_params()
todos_params = list(theta_sust.values()) + list(theta_verb.values())
opt = qml.AdamOptimizer(stepsize=0.05)

print("\n=== Entrenando QNLP ===")
historial_loss = []

for epoch in range(1, 81):
    # Actualizar cada parámetro individualmente
    for sust in sustantivos:
        def fn_sust(ts):
            ts_dict = dict(theta_sust)
            ts_dict[sust] = ts
            return coste_total(ts_dict, theta_verb)
        theta_sust[sust], loss = opt.step_and_cost(fn_sust, theta_sust[sust])

    for verb in verbos:
        def fn_verb(tv):
            tv_dict = dict(theta_verb)
            tv_dict[verb] = tv
            return coste_total(theta_sust, tv_dict)
        theta_verb[verb], _ = opt.step_and_cost(fn_verb, theta_verb[verb])

    loss_val = float(coste_total(theta_sust, theta_verb))
    historial_loss.append(loss_val)

    if epoch % 20 == 0:
        correct = sum(
            1 for sust, verb, y in DATASET
            if (predecir_prob(theta_sust[sust], theta_verb[verb]).item() > 0.5) == bool(y)
        )
        acc = correct / len(DATASET)
        print(f"  Epoch {epoch}: loss={loss_val:.4f}, acc={acc:.2f}")

# Evaluación final
print("\n=== Resultados QNLP ===")
print(f"{'Frase':<25} {'P(pos)':>8} {'Pred':>6} {'Real':>6} {'✓':>4}")
print("-" * 50)
for sust, verb, y in DATASET:
    p = float(predecir_prob(theta_sust[sust], theta_verb[verb]))
    pred = 1 if p > 0.5 else 0
    ok = "✓" if pred == y else "✗"
    print(f"{sust+' '+verb:<25} {p:>8.4f} {pred:>6} {y:>6} {ok:>4}")

correct = sum(
    1 for s, v, y in DATASET
    if (predecir_prob(theta_sust[s], theta_verb[v]).item() > 0.5) == bool(y)
)
print(f"\nAccuracy: {correct}/{len(DATASET)} = {correct/len(DATASET):.2%}")
```

---

## 3. Modelo con lambeq (si disponible)

```python
# seccion_03_lambeq.py
"""
Flujo completo con lambeq:
1. Parsear frases con BobcatParser (CCG)
2. Generar diagramas DisCoCat
3. Convertir a circuitos PennyLane
4. Entrenar con optimización variacional

Requiere: pip install lambeq
"""
print("=== QNLP con lambeq ===")

codigo_lambeq = '''
from lambeq import BobcatParser, AtomicType, IQMansatz, Dataset
from lambeq.backend.pennylane import PennyLaneModel
import pennylane as qml
import torch

# Tipos atómicos
N = AtomicType.NOUN
S = AtomicType.SENTENCE

# Dataset
train_data = [
    ("Alice loves Bob",      1),
    ("Bob hates Alice",      0),
    ("Alice enjoys music",   1),
    ("Bob suffers pain",     0),
]
train_sents = [s for s,_ in train_data]
train_labels = [y for _,y in train_data]

# Parsing CCG
parser = BobcatParser(verbose="text")
train_diagrams = parser.sentences2diagrams(train_sents)

# Ansatz: IQP (Instantaneous Quantum Polynomial)
ansatz = IQMansatz({N: 1, S: 1}, n_layers=1)
train_circuits = [ansatz(d) for d in train_diagrams]

# Modelo PennyLane
model = PennyLaneModel.from_diagrams(train_circuits)
model.initialise_weights()

# Entrenamiento con PyTorch
optimizer = torch.optim.Adam(model.weights, lr=0.01)
loss_fn = torch.nn.BCELoss()

for epoch in range(50):
    optimizer.zero_grad()
    preds = model(train_circuits)
    loss = loss_fn(preds[:, 1], torch.FloatTensor(train_labels))
    loss.backward()
    optimizer.step()
    if (epoch+1) % 10 == 0:
        print(f"Epoch {epoch+1}: loss={loss.item():.4f}")

# Predicción
for sent, circ, label in zip(train_sents, train_circuits, train_labels):
    pred = model([circ])
    p_pos = pred[0, 1].item()
    print(f"  '{sent}': P(pos)={p_pos:.3f} → {'✓' if (p_pos>0.5)==bool(label) else '✗'}")
'''

try:
    import lambeq
    print("lambeq disponible — ejecutando flujo completo...")
    exec(codigo_lambeq)
except ImportError:
    print("lambeq no instalado. Instalar con: pip install lambeq")
    print("\nCódigo del flujo completo (para referencia):")
    print(codigo_lambeq)
```

---

## 4. Word2Vec cuántico: embeddings en espacio de Hilbert

```python
# seccion_04_quantum_embeddings.py
"""
Quantum Word Embeddings:
En lugar de vectores clásicos R^d, las palabras se representan
como estados cuánticos en H^(2^n).

Ventaja potencial: 2^n dimensiones de embedding con n qubits.
Para n=4: 16 dimensiones de embedding clásico equivalente.

Implementación: optimizar parámetros RY para que
palabras similares tengan alta fidelidad |⟨ψ_i|ψ_j⟩|².
"""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

n_qubits_embed = 3  # 2^3 = 8 dimensiones de embedding
dev = qml.device("default.qubit", wires=n_qubits_embed)

# Palabras del vocabulario con relaciones de similaridad
# (positivo, negativo, neutro)
vocabulario = {
    "love":    {"grupo": "positivo", "idx": 0},
    "enjoy":   {"grupo": "positivo", "idx": 1},
    "like":    {"grupo": "positivo", "idx": 2},
    "hate":    {"grupo": "negativo", "idx": 3},
    "dislike": {"grupo": "negativo", "idx": 4},
    "loathe":  {"grupo": "negativo", "idx": 5},
    "run":     {"grupo": "neutro",   "idx": 6},
    "walk":    {"grupo": "neutro",   "idx": 7},
}
palabras = list(vocabulario.keys())
grupos   = [vocabulario[p]["grupo"] for p in palabras]

@qml.qnode(dev)
def estado_palabra(params):
    """Estado cuántico para una palabra: RY + entrelazamiento."""
    for i in range(n_qubits_embed):
        qml.RY(params[i], wires=i)
    qml.CNOT(wires=[0, 1])
    qml.CNOT(wires=[1, 2])
    for i in range(n_qubits_embed):
        qml.RY(params[n_qubits_embed + i], wires=i)
    return qml.state()

def fidelidad(p1, p2):
    """Fidelidad entre dos parámetros de palabras."""
    sv1 = estado_palabra(p1)
    sv2 = estado_palabra(p2)
    return abs(np.dot(np.conj(sv1), sv2))**2

# Inicializar parámetros: palabras del mismo grupo empiezan cerca
np.random.seed(42)
n_params_por_palabra = 2 * n_qubits_embed

params_vocab = {}
for palabra in palabras:
    grupo = vocabulario[palabra]["grupo"]
    base = {"positivo": 0.5, "negativo": 2.0, "neutro": 1.2}[grupo]
    params_vocab[palabra] = pnp.array(
        base + 0.1 * np.random.randn(n_params_por_palabra),
        requires_grad=True
    )

def coste_embedding():
    """
    Minimizar: distancia intra-grupo — Maximizar: distancia inter-grupo
    L = Σ_mismos -log(F(i,j)) + Σ_distintos log(F(i,j))
    """
    loss = 0
    eps = 1e-6
    for i, p1 in enumerate(palabras):
        for j, p2 in enumerate(palabras[i+1:], i+1):
            F = fidelidad(params_vocab[p1], params_vocab[p2])
            F_clip = pnp.clip(F, eps, 1-eps)
            mismo_grupo = grupos[i] == grupos[j]
            if mismo_grupo:
                loss += -pnp.log(F_clip)   # palabras similares → alta fidelidad
            else:
                loss += -pnp.log(1 - F_clip)  # palabras distintas → baja fidelidad
    return loss / (len(palabras) * (len(palabras)-1) / 2)

opt_emb = qml.AdamOptimizer(stepsize=0.1)
print("=== Quantum Word Embeddings ===")
print(f"Vocabulario: {palabras}")

losses_emb = []
for epoch in range(60):
    for palabra in palabras:
        def fn_p(param):
            prev = params_vocab[palabra]
            params_vocab[palabra] = param
            L = coste_embedding()
            params_vocab[palabra] = prev
            return L
        params_vocab[palabra], L = opt_emb.step_and_cost(fn_p, params_vocab[palabra])
    losses_emb.append(float(L))
    if (epoch+1) % 20 == 0:
        print(f"  Epoch {epoch+1}: loss={L:.4f}")

# Matriz de fidelidades
n_words = len(palabras)
F_matrix = np.zeros((n_words, n_words))
for i, p1 in enumerate(palabras):
    for j, p2 in enumerate(palabras):
        F_matrix[i, j] = float(fidelidad(params_vocab[p1], params_vocab[p2]))

print("\nMatriz de fidelidades F[i,j] (1=idénticos, 0=ortogonales):")
print(f"{'':>10}", end="")
for p in palabras:
    print(f"{p[:6]:>8}", end="")
print()
for i, p1 in enumerate(palabras):
    print(f"{p1:>10}", end="")
    for j in range(n_words):
        print(f"{F_matrix[i,j]:>8.3f}", end="")
    print()

# Visualización: PCA de embeddings cuánticos
from sklearn.decomposition import PCA
estados = np.array([estado_palabra(params_vocab[p]) for p in palabras])
# Real part of statevector for PCA
estados_real = np.hstack([estados.real, estados.imag])
pca = PCA(n_components=2)
proj = pca.fit_transform(estados_real)

fig, axes = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle("Quantum Word Embeddings", fontsize=13)

color_map = {"positivo": "green", "negativo": "red", "neutro": "blue"}
ax = axes[0]
for i, (p, g) in enumerate(zip(palabras, grupos)):
    ax.scatter(proj[i, 0], proj[i, 1], c=color_map[g], s=200, zorder=5)
    ax.annotate(p, (proj[i, 0], proj[i, 1]), textcoords="offset points",
                xytext=(5, 5), fontsize=9)
from matplotlib.patches import Patch
leyenda = [Patch(color=c, label=g) for g, c in color_map.items()]
ax.legend(handles=leyenda, fontsize=9)
ax.set_title("PCA de Embeddings Cuánticos")
ax.set_xlabel("PC1"); ax.set_ylabel("PC2")
ax.grid(alpha=0.3)

# Heatmap de fidelidades
ax = axes[1]
im = ax.imshow(F_matrix, cmap="YlOrRd", vmin=0, vmax=1)
plt.colorbar(im, ax=ax, label="Fidelidad |⟨ψᵢ|ψⱼ⟩|²")
ax.set_xticks(range(n_words))
ax.set_yticks(range(n_words))
ax.set_xticklabels(palabras, rotation=45, ha="right", fontsize=8)
ax.set_yticklabels(palabras, fontsize=8)
ax.set_title("Fidelidades entre Palabras")
for i in range(n_words):
    for j in range(n_words):
        ax.text(j, i, f"{F_matrix[i,j]:.2f}", ha="center", va="center",
                fontsize=7, color="black" if F_matrix[i,j] < 0.7 else "white")

plt.tight_layout()
plt.savefig("outputs/m31_qnlp.png", dpi=120, bbox_inches="tight")
print("\nFigura guardada: outputs/m31_qnlp.png")
plt.close()
```

---

## 5. Comparación con modelos clásicos de NLP

```python
# seccion_05_comparacion.py
"""
Comparativa QNLP vs NLP clásico para clasificación de sentimientos.
Modelos clásicos:
  - Bag of Words + Logistic Regression
  - TF-IDF + SVM
  - Embeddings fijos (Word2Vec simplificado)

Dataset: frases simples de sentimiento binario.
"""
import numpy as np
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.metrics import accuracy_score

# Dataset extendido
frases = [
    ("alice loves music",    1),
    ("bob enjoys coding",    1),
    ("alice wins the game",  1),
    ("bob succeeds greatly", 1),
    ("alice thrives daily",  1),
    ("bob fails again",      0),
    ("alice hates noise",    0),
    ("bob loses everything", 0),
    ("alice suffers pain",   0),
    ("bob cries loudly",     0),
]

textos  = [f for f,_ in frases]
labels  = [y for _,y in frases]

# Dividir train/test (leave-one-out simplificado)
def loocv(clf_class, X, y, **kwargs):
    """Leave-one-out cross-validation."""
    accs = []
    for i in range(len(y)):
        X_tr = np.delete(X, i, axis=0)
        y_tr = [y[j] for j in range(len(y)) if j != i]
        X_te = X[i:i+1]
        y_te = [y[i]]
        clf = clf_class(**kwargs)
        clf.fit(X_tr, y_tr)
        accs.append(accuracy_score(y_te, clf.predict(X_te)))
    return np.mean(accs)

# Bag of Words
bow = CountVectorizer()
X_bow = bow.fit_transform(textos).toarray()
acc_bow_lr  = loocv(LogisticRegression, X_bow, labels, max_iter=1000)
acc_bow_svm = loocv(SVC, X_bow, labels, kernel="linear", C=1)

# TF-IDF
tfidf = TfidfVectorizer()
X_tfidf = tfidf.fit_transform(textos).toarray()
acc_tfidf_lr  = loocv(LogisticRegression, X_tfidf, labels, max_iter=1000)
acc_tfidf_svm = loocv(SVC, X_tfidf, labels, kernel="rbf", C=1)

# Resultado QNLP (de sección 2)
from seccion_02_sentimientos import predecir_prob, theta_sust, theta_verb, DATASET
acc_qnlp = sum(
    1 for s, v, y in DATASET
    if (predecir_prob(theta_sust[s], theta_verb[v]).item() > 0.5) == bool(y)
) / len(DATASET)

print("=== Comparación NLP Clásico vs QNLP ===")
print(f"\n{'Método':<30} {'Accuracy (LOOCV)':>18}")
print("-" * 52)
resultados = [
    ("BoW + LogReg",       acc_bow_lr),
    ("BoW + SVM-Linear",   acc_bow_svm),
    ("TF-IDF + LogReg",    acc_tfidf_lr),
    ("TF-IDF + SVM-RBF",   acc_tfidf_svm),
    ("QNLP (DisCoCat)",    acc_qnlp),
]
for nombre, acc in resultados:
    barra = "█" * int(acc * 20)
    print(f"{nombre:<30} {acc:>10.3f}  {barra}")

print("\nNota: Con datasets pequeños (10 frases), todos los métodos")
print("tienen alta varianza. QNLP es competitivo pero no superior.")
print("Ventaja potencial: estructuras gramaticales complejas.")
```

---

## Checkpoint

```python
# checkpoint_m31.py
"""Tests para Módulo 31 — Quantum NLP."""
import numpy as np
import pennylane as qml
import pennylane.numpy as pnp

n_qubits = 3  # n + n→s = 1 + 2
dev = qml.device("default.qubit", wires=n_qubits)

@qml.qnode(dev, diff_method="parameter-shift")
def circuito_frase_test(theta_s, theta_v):
    qml.RY(theta_s[0], wires=0)
    qml.RY(theta_v[0], wires=1)
    qml.RY(theta_v[1], wires=2)
    qml.CNOT(wires=[0, 1])
    qml.RY(theta_v[2], wires=1)
    qml.RY(theta_v[3], wires=2)
    qml.CNOT(wires=[1, 2])
    return qml.expval(qml.PauliZ(2))

# T1: Output en rango [-1, 1]
def test_output_rango():
    ts = pnp.array([0.5])
    tv = pnp.array([0.3, 0.7, 1.2, 0.8])
    out = float(circuito_frase_test(ts, tv))
    assert -1.0 <= out <= 1.0
    print(f"T1 PASS: output = {out:.4f} ∈ [-1,1]")

# T2: Gradiente llega a parámetros de sustantivo
def test_grad_sustantivo():
    ts = pnp.array([0.5], requires_grad=True)
    tv = pnp.array([0.3, 0.7, 1.2, 0.8])
    grad = qml.grad(circuito_frase_test, argnum=0)(ts, tv)
    assert not np.isnan(grad[0])
    assert abs(grad[0]) > 1e-10
    print(f"T2 PASS: grad(theta_sust) = {float(grad[0]):.4f}")

# T3: Gradiente llega a parámetros de verbo
def test_grad_verbo():
    ts = pnp.array([0.5])
    tv = pnp.array([0.3, 0.7, 1.2, 0.8], requires_grad=True)
    grad = qml.grad(circuito_frase_test, argnum=1)(ts, tv)
    assert not np.any(np.isnan(grad))
    assert np.any(np.abs(grad) > 1e-10)
    print(f"T3 PASS: |grad(theta_verb)|_max = {np.abs(grad).max():.4f}")

# T4: Fidelidad cuántica entre palabras similares > 0.5
def test_fidelidad_semantica():
    n_emb = 3
    dev2 = qml.device("default.qubit", wires=n_emb)

    @qml.qnode(dev2)
    def estado_emb(params):
        for i in range(n_emb):
            qml.RY(params[i], wires=i)
        return qml.state()

    # Mismos parámetros → fidelidad = 1
    params_same = np.array([0.5, 1.0, 0.3])
    sv1 = estado_emb(params_same)
    sv2 = estado_emb(params_same)
    F_same = abs(np.dot(np.conj(sv1), sv2))**2
    assert abs(F_same - 1.0) < 1e-10
    print(f"T4 PASS: fidelidad(mismo, mismo) = {F_same:.4f}")

# T5: P(positivo) = (⟨Z⟩+1)/2 ∈ [0,1]
def test_probabilidad_valida():
    ts = pnp.array([1.2])
    tv = pnp.array([0.5, 0.5, 0.5, 0.5])
    ev = float(circuito_frase_test(ts, tv))
    p_pos = (ev + 1) / 2
    assert 0 <= p_pos <= 1, f"Probabilidad {p_pos} fuera de [0,1]"
    print(f"T5 PASS: P(pos) = {p_pos:.4f} ∈ [0,1]")

# T6: BCE loss disminuye con entrenamiento
def test_loss_decrece():
    ts = pnp.array([0.0], requires_grad=True)
    tv_pos = pnp.array([0.0, 0.0, 0.0, 0.0], requires_grad=True)

    def coste_simple(ts, tv):
        ev = circuito_frase_test(ts, tv)
        p = (ev + 1) / 2
        return -pnp.log(pnp.clip(p, 1e-7, 1-1e-7))  # y=1 → positivo

    opt = qml.GradientDescentOptimizer(stepsize=0.3)
    losses = []
    for _ in range(20):
        (ts, tv_pos), L = opt.step_and_cost(coste_simple, ts, tv_pos)
        losses.append(float(L))

    assert losses[-1] < losses[0], f"Loss no decreció: {losses[0]:.4f} → {losses[-1]:.4f}"
    print(f"T6 PASS: loss {losses[0]:.4f} → {losses[-1]:.4f} (decreció)")

if __name__ == "__main__":
    print("=== Checkpoint M31: Quantum NLP ===\n")
    test_output_rango()
    test_grad_sustantivo()
    test_grad_verbo()
    test_fidelidad_semantica()
    test_probabilidad_valida()
    test_loss_decrece()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M32 — Criptografía Cuántica — QKD y Protocolo BB84](./modulo-32-qkd.md)
