# Módulo 10 — Reglas de Asociación: Apriori y FP-Growth

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M09 (Clustering)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Conceptos: soporte, confianza, lift](#conceptos-fundamentales)
3. [Algoritmo Apriori](#algoritmo-apriori)
4. [FP-Growth — más eficiente que Apriori](#fp-growth)
5. [Filtrado e interpretación de reglas](#filtrado-e-interpretación)
6. [Caso práctico: Market Basket Analysis](#caso-práctico)
7. [Visualización de reglas](#visualización-de-reglas)
8. [Checkpoint M10](#checkpoint-m10)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install mlxtend  # contiene Apriori y FP-Growth
python modulo-10-reglas-asociacion.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
!pip install -q mlxtend
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Conceptos fundamentales

Las reglas de asociación descubren relaciones del tipo:

```
SI {leche, pan} ENTONCES {mantequilla}  →  "quien compra leche y pan suele comprar mantequilla"

Métricas principales:
────────────────────────────────────────────────────────────────────────────
Soporte    support(A → B) = P(A ∪ B) = frecuencia de A y B juntos
                          = transacciones(A ∪ B) / total_transacciones

Confianza  confidence(A → B) = P(B | A) = P(A ∪ B) / P(A)
                             = ¿qué % de veces que aparece A aparece también B?

Lift       lift(A → B) = confidence(A → B) / support(B)
                       = P(A ∪ B) / (P(A) × P(B))
                       > 1: asociación positiva (A y B se atraen)
                       = 1: independencia
                       < 1: asociación negativa (A y B se repelen)

Conviction conviction(A → B) = (1 - P(B)) / (1 - confidence(A → B))
           → alta = la regla no es accidental

Leverage   leverage(A → B) = P(A ∪ B) - P(A) × P(B)
           → cuánto más frecuente que si fueran independientes

Zhang      zhang(A → B) = (P(AB) - P(A)P(B)) / max(P(AB)(1-P(A)), P(A)P(B)(1-P(AB)))
           ∈ [-1, 1] → métrica robusta, similar a correlación de Pearson
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs']: os.makedirs(d, exist_ok=True)

# ── Cálculo manual de métricas ────────────────────────────────────────────────
# Ejemplo: supermercado con 10 transacciones
transacciones_ejemplo = [
    ['leche', 'pan', 'mantequilla'],
    ['leche', 'pan'],
    ['leche', 'mantequilla'],
    ['pan', 'mantequilla'],
    ['leche', 'pan', 'mantequilla', 'huevo'],
    ['pan', 'huevo'],
    ['leche', 'huevo'],
    ['leche', 'pan', 'mantequilla'],
    ['pan', 'mantequilla'],
    ['leche', 'pan'],
]

N = len(transacciones_ejemplo)

def soporte(items, transacciones):
    """Soporte de un conjunto de items."""
    return sum(all(i in t for i in items) for t in transacciones) / len(transacciones)

def confianza(antecedente, consecuente, transacciones):
    """Confianza de la regla A → B."""
    sup_union = soporte(antecedente + consecuente, transacciones)
    sup_ant   = soporte(antecedente, transacciones)
    return sup_union / sup_ant if sup_ant > 0 else 0

def lift(antecedente, consecuente, transacciones):
    """Lift de la regla A → B."""
    conf = confianza(antecedente, consecuente, transacciones)
    sup_consec = soporte(consecuente, transacciones)
    return conf / sup_consec if sup_consec > 0 else 0

# Regla: {leche, pan} → {mantequilla}
A, B = ['leche', 'pan'], ['mantequilla']
print(f"Regla: {{leche, pan}} → {{mantequilla}}")
print(f"  Soporte A∪B:    {soporte(A+B, transacciones_ejemplo):.2f}")
print(f"  Soporte A:      {soporte(A, transacciones_ejemplo):.2f}")
print(f"  Soporte B:      {soporte(B, transacciones_ejemplo):.2f}")
print(f"  Confianza:      {confianza(A, B, transacciones_ejemplo):.2f}")
print(f"  Lift:           {lift(A, B, transacciones_ejemplo):.2f}")
print()

# Regla: {pan} → {leche}
A2, B2 = ['pan'], ['leche']
print(f"Regla: {{pan}} → {{leche}}")
print(f"  Confianza: {confianza(A2, B2, transacciones_ejemplo):.2f}")
print(f"  Lift:      {lift(A2, B2, transacciones_ejemplo):.2f}")
print()

print("Lift > 1 → asociación positiva (no aleatoria)")
print("Lift ≈ 1 → independiente (la regla no aporta información)")
print("Lift < 1 → asociación negativa")
```

---

## Algoritmo Apriori

```
Principio de Apriori (propiedad anti-monótona del soporte):
  Si un itemset es frecuente, todos sus subconjuntos también lo son.
  Si un itemset es infrecuente, ninguno de sus superconjuntos puede ser frecuente.

  → Podamos el espacio de búsqueda: no exploramos superconjuntos de infrecuentes.

Pasos:
  1. Generar itemsets de longitud 1 con support ≥ min_support
  2. Combinar para generar candidatos de longitud 2
  3. Podar los candidatos con subconjuntos infrecuentes
  4. Contar soporte de candidatos supervivientes
  5. Repetir hasta que no haya itemsets frecuentes nuevos
  6. Generar reglas a partir de los itemsets frecuentes
```

```python
from mlxtend.frequent_patterns import apriori, fpgrowth, association_rules
from mlxtend.preprocessing import TransactionEncoder

# ── Dataset Online Retail (UK) — usando versión reducida ─────────────────────
# Generamos un dataset sintético de tienda online para el ejemplo
np.random.seed(42)
productos = ['leche', 'pan', 'mantequilla', 'queso', 'huevo',
             'jugo', 'cereal', 'yogur', 'fruta', 'carne',
             'pasta', 'salsa', 'vino', 'cerveza', 'agua']

def generar_canasta():
    """Genera una transacción con correlaciones realistas."""
    canasta = []
    # Grupos de afinidad
    if np.random.random() < 0.6:
        canasta += np.random.choice(['leche', 'pan', 'mantequilla'],
                                     size=np.random.randint(1,3), replace=False).tolist()
    if np.random.random() < 0.4:
        canasta += np.random.choice(['queso', 'yogur', 'mantequilla'],
                                     size=np.random.randint(1,3), replace=False).tolist()
    if np.random.random() < 0.3:
        canasta += np.random.choice(['vino', 'queso', 'salsa'],
                                     size=np.random.randint(1,3), replace=False).tolist()
    if np.random.random() < 0.25:
        canasta += np.random.choice(['cerveza', 'carne', 'pan'],
                                     size=np.random.randint(1,3), replace=False).tolist()
    # Productos aleatorios adicionales
    extra = np.random.choice(productos, size=np.random.randint(1,4), replace=False).tolist()
    canasta = list(set(canasta + extra))
    return canasta if canasta else ['agua']

dataset = [generar_canasta() for _ in range(1000)]
print(f"Dataset generado: {len(dataset)} transacciones")
print(f"Ejemplo de transacción: {dataset[0]}")
print(f"Tamaño promedio de canasta: {np.mean([len(t) for t in dataset]):.1f} items")

# ── Transformar a formato binario (One-Hot) ───────────────────────────────────
te = TransactionEncoder()
te_array = te.fit_transform(dataset)
df_transacciones = pd.DataFrame(te_array, columns=te.columns_)

print(f"\nForma del DataFrame one-hot: {df_transacciones.shape}")
print(f"Frecuencia por producto:")
print((df_transacciones.sum() / len(df_transacciones)).sort_values(ascending=False).round(3))

# ── Apriori ───────────────────────────────────────────────────────────────────
itemsets_apriori = apriori(
    df_transacciones,
    min_support=0.10,
    use_colnames=True,
    max_len=4
)
print(f"\nItemsets frecuentes (Apriori, min_support=0.10): {len(itemsets_apriori)}")
print(itemsets_apriori.sort_values('support', ascending=False).head(10).to_string())

# Generar reglas
reglas_apriori = association_rules(
    itemsets_apriori,
    metric='lift',
    min_threshold=1.0
)
reglas_apriori = reglas_apriori.sort_values('lift', ascending=False)
print(f"\nReglas generadas: {len(reglas_apriori)}")
print("\nTop 10 por Lift:")
cols = ['antecedents', 'consequents', 'support', 'confidence', 'lift', 'conviction']
print(reglas_apriori[cols].head(10).to_string())
```

---

## FP-Growth

```
FP-Growth (Frequent Pattern Growth) — más eficiente que Apriori:
  • Apriori: múltiples pasadas por la base de datos
  • FP-Growth: solo 2 pasadas y construye un árbol compacto (FP-Tree)
  • Ventaja: O(n) pasos vs O(n²) de Apriori para datasets grandes
  • Memoria: el FP-Tree puede ser compacto si hay correlaciones fuertes
```

```python
import time

# ── Comparativa de velocidad Apriori vs FP-Growth ─────────────────────────────
t0 = time.time()
itemsets_ap = apriori(df_transacciones, min_support=0.08, use_colnames=True)
t_ap = time.time() - t0

t0 = time.time()
itemsets_fp = fpgrowth(df_transacciones, min_support=0.08, use_colnames=True)
t_fp = time.time() - t0

print(f"\nApriori:   {len(itemsets_ap):4d} itemsets en {t_ap:.3f}s")
print(f"FP-Growth: {len(itemsets_fp):4d} itemsets en {t_fp:.3f}s")
print(f"Speedup FP-Growth: {t_ap/t_fp:.1f}x")
print("(La diferencia es mayor con datasets más grandes)")

# Generar reglas desde FP-Growth (mismo resultado)
reglas_fp = association_rules(itemsets_fp, metric='lift', min_threshold=1.0)
reglas_fp = reglas_fp.sort_values('lift', ascending=False)
print(f"\nReglas FP-Growth: {len(reglas_fp)}")
```

---

## Filtrado e interpretación

```python
# ── Filtros avanzados de reglas ───────────────────────────────────────────────

def filtrar_reglas(reglas, min_support=0, min_confidence=0,
                   min_lift=1, max_antecedentes=None):
    """Filtra reglas de asociación con múltiples criterios."""
    mask = (
        (reglas['support']    >= min_support) &
        (reglas['confidence'] >= min_confidence) &
        (reglas['lift']       >= min_lift)
    )
    if max_antecedentes:
        mask &= reglas['antecedents'].apply(len) <= max_antecedentes

    return reglas[mask].sort_values('lift', ascending=False)

# Reglas de alta confianza (actionable)
reglas_accionables = filtrar_reglas(
    reglas_fp,
    min_support=0.10,
    min_confidence=0.60,
    min_lift=1.5,
    max_antecedentes=2
)
print(f"\n─── Reglas accionables (conf≥0.6, lift≥1.5, sup≥0.1) ───")
print(f"Total: {len(reglas_accionables)}")
cols = ['antecedents', 'consequents', 'support', 'confidence', 'lift']
print(reglas_accionables[cols].head(15).to_string())

# Conversión a formato legible
print("\nReglas en formato human-readable:")
for _, regla in reglas_accionables.head(5).iterrows():
    ant  = ', '.join(sorted(regla['antecedents']))
    cons = ', '.join(sorted(regla['consequents']))
    print(f"  SI {{{ant}}} → ENTONCES {{{cons}}}  "
          f"[sup={regla['support']:.2f}, conf={regla['confidence']:.2f}, "
          f"lift={regla['lift']:.2f}]")

# ── Métricas adicionales de mlxtend ──────────────────────────────────────────
# Zhang's metric (más robusta que lift)
reglas_fp['zhang'] = (
    (reglas_fp['support'] - reglas_fp['antecedent support'] * reglas_fp['consequent support']) /
    (np.maximum(
        reglas_fp['support'] * (1 - reglas_fp['antecedent support']),
        reglas_fp['antecedent support'] * (reglas_fp['consequent support'] - reglas_fp['support'])
    ) + 1e-10)
)

print(f"\nTop 5 por Zhang's metric:")
print(reglas_fp[cols + ['zhang']].sort_values('zhang', ascending=False).head(5).to_string())

# Leverage
reglas_fp['leverage'] = (
    reglas_fp['support'] -
    reglas_fp['antecedent support'] * reglas_fp['consequent support']
)
print(f"\nTop 5 por Leverage:")
print(reglas_fp[cols + ['leverage']].sort_values('leverage', ascending=False).head(5).to_string())
```

---

## Caso práctico

```python
# ── Caso práctico: análisis de cesta de compra en tienda online ───────────────
# Generamos un dataset más realista con productos de e-commerce
np.random.seed(123)

categorias_ecom = {
    'electronicos':   ['laptop', 'mouse', 'teclado', 'monitor', 'auriculares', 'webcam'],
    'accesorios':     ['funda_laptop', 'hub_usb', 'cable_hdmi', 'adaptador', 'soporte'],
    'oficina':        ['papel', 'carpeta', 'boligrafo', 'agenda', 'sello'],
    'gaming':         ['controlador', 'auriculares_gaming', 'mousepad', 'silla_gaming'],
}

# Reglas de compra con sentido de negocio
def generar_pedido_ecom():
    pedido = []
    # Laptop → accesorios
    if 'laptop' in pedido or np.random.random() < 0.3:
        pedido.append('laptop')
        if np.random.random() < 0.7:
            pedido += np.random.choice(['mouse', 'teclado', 'funda_laptop'],
                                        size=np.random.randint(1,3), replace=False).tolist()
        if np.random.random() < 0.5:
            pedido += np.random.choice(['hub_usb', 'cable_hdmi', 'monitor'],
                                        size=np.random.randint(1,2), replace=False).tolist()
    # Gaming set
    if np.random.random() < 0.2:
        pedido += np.random.choice(['controlador', 'auriculares_gaming', 'mousepad'],
                                    size=np.random.randint(2,4), replace=False).tolist()
    # Items aleatorios
    todos = [item for cat in categorias_ecom.values() for item in cat]
    extra = np.random.choice(todos, size=np.random.randint(0,3), replace=False).tolist()
    pedido = list(set(pedido + extra))
    return pedido if len(pedido) >= 2 else ['papel', 'boligrafo']

pedidos = [generar_pedido_ecom() for _ in range(2000)]

te_ecom = TransactionEncoder()
df_ecom = pd.DataFrame(te_ecom.fit_transform(pedidos), columns=te_ecom.columns_)
print(f"\n─── E-commerce Dataset ───")
print(f"  {len(pedidos)} pedidos | {df_ecom.shape[1]} productos únicos")
print(f"  Tamaño promedio de pedido: {np.mean([len(p) for p in pedidos]):.1f} items")

# FP-Growth
itemsets_ecom = fpgrowth(df_ecom, min_support=0.05, use_colnames=True)
reglas_ecom = association_rules(itemsets_ecom, metric='lift', min_threshold=1.5)
reglas_ecom = reglas_ecom.sort_values('lift', ascending=False)

print(f"\n  Itemsets frecuentes (sup≥5%): {len(itemsets_ecom)}")
print(f"  Reglas (lift≥1.5): {len(reglas_ecom)}")

# Top reglas de recomendación de productos
print("\n  Top reglas para cross-selling:")
for _, r in reglas_ecom.head(8).iterrows():
    ant  = ', '.join(sorted(r['antecedents']))
    cons = ', '.join(sorted(r['consequents']))
    print(f"    {ant} → {cons}  [conf={r['confidence']:.2f}, lift={r['lift']:.2f}]")

# Guardar reglas
reglas_ecom.to_csv('outputs/reglas_ecommerce.csv', index=False)
print("\n  Reglas guardadas en outputs/reglas_ecommerce.csv")
```

---

## Visualización de reglas

```python
# ── Visualización 1: scatter plot support vs confidence (lift como color) ─────
fig, axes = plt.subplots(1, 2, figsize=(14, 6))

scatter = axes[0].scatter(
    reglas_ecom['support'],
    reglas_ecom['confidence'],
    c=reglas_ecom['lift'],
    cmap='YlOrRd',
    s=30, alpha=0.7,
    edgecolors='gray', linewidths=0.3
)
plt.colorbar(scatter, ax=axes[0], label='Lift')
axes[0].set_xlabel('Soporte')
axes[0].set_ylabel('Confianza')
axes[0].set_title('Reglas de asociación\n(color = Lift, mayor = mejor)')

# Líneas de referencia
axes[0].axvline(0.10, color='blue', ls='--', alpha=0.5, label='sup=0.10')
axes[0].axhline(0.60, color='red', ls='--', alpha=0.5, label='conf=0.60')
axes[0].legend(fontsize=8)

# ── Visualización 2: heatmap de lift entre productos (top 10) ────────────────
from itertools import combinations

# Calcular lift entre pares de productos más frecuentes
top_productos = (df_ecom.sum() / len(df_ecom)).sort_values(ascending=False).head(10).index.tolist()

lift_matrix = pd.DataFrame(np.ones((len(top_productos), len(top_productos))),
                             index=top_productos, columns=top_productos)

for p1, p2 in combinations(top_productos, 2):
    sup_p1 = (df_ecom[p1].sum()) / len(df_ecom)
    sup_p2 = (df_ecom[p2].sum()) / len(df_ecom)
    sup_12 = ((df_ecom[p1] & df_ecom[p2]).sum()) / len(df_ecom)
    lift_val = sup_12 / (sup_p1 * sup_p2) if sup_p1 * sup_p2 > 0 else 1
    lift_matrix.loc[p1, p2] = lift_val
    lift_matrix.loc[p2, p1] = lift_val

im = axes[1].imshow(lift_matrix.values, cmap='RdYlGn',
                     vmin=0.5, vmax=lift_matrix.values.max())
plt.colorbar(im, ax=axes[1], label='Lift')
axes[1].set_xticks(range(len(top_productos)))
axes[1].set_yticks(range(len(top_productos)))
axes[1].set_xticklabels(top_productos, rotation=45, ha='right', fontsize=8)
axes[1].set_yticklabels(top_productos, fontsize=8)
axes[1].set_title('Heatmap de Lift — Top 10 productos')

# Valores en celdas
for i in range(len(top_productos)):
    for j in range(len(top_productos)):
        axes[1].text(j, i, f'{lift_matrix.values[i,j]:.1f}',
                     ha='center', va='center', fontsize=6)

plt.tight_layout()
plt.savefig('outputs/m10_reglas_visualizacion.png', dpi=120)
plt.show()

# ── Visualización 3: grafo de reglas (NetworkX) ───────────────────────────────
try:
    import networkx as nx

    G = nx.DiGraph()

    # Solo top 15 reglas por lift
    top_reglas = reglas_ecom.head(15)
    for _, r in top_reglas.iterrows():
        ant  = ', '.join(sorted(r['antecedents']))
        cons = ', '.join(sorted(r['consequents']))
        G.add_edge(ant, cons, weight=r['lift'], confidence=r['confidence'])

    fig, ax = plt.subplots(figsize=(12, 8))
    pos = nx.spring_layout(G, seed=42, k=2)

    # Nodos por tipo (antecedente o consecuente)
    antecedentes = set(r['antecedents'] for _, r in top_reglas.iterrows()
                       if hasattr(r['antecedents'], '__iter__'))
    # Simplificado: todos los nodos en azul
    nx.draw_networkx_nodes(G, pos, node_color='lightblue',
                           node_size=2000, ax=ax)
    nx.draw_networkx_labels(G, pos, font_size=7, ax=ax)

    edges = G.edges(data=True)
    weights = [d['weight'] for _, _, d in edges]
    nx.draw_networkx_edges(G, pos, edge_color=weights, edge_cmap=plt.cm.Reds,
                           width=2, arrows=True, arrowsize=20,
                           connectionstyle='arc3,rad=0.1', ax=ax)
    ax.set_title('Grafo de Reglas de Asociación (grosor = Lift)', fontsize=12)
    ax.axis('off')
    plt.tight_layout()
    plt.savefig('outputs/m10_grafo_reglas.png', dpi=120)
    plt.show()
    print("Grafo de reglas generado")
except ImportError:
    print("(networkx no instalado — instalar: pip install networkx)")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   RESUMEN M10 — Reglas de Asociación                        │
├─────────────────────┬───────────────────────────────────────────────────────┤
│ Concepto            │ Descripción                                            │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Soporte             │ Frecuencia del itemset completo                        │
│ Confianza           │ P(consecuente | antecedente) — predictibilidad         │
│ Lift                │ >1 positivo, =1 independiente, <1 negativo             │
│ Leverage            │ Diferencia real vs esperado por independencia          │
│ Conviction          │ Alta = regla no es accidental                          │
│ Zhang               │ Métrica robusta [-1,1] análoga a correlación           │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Apriori             │ Simple pero múltiples pasadas por los datos            │
│ FP-Growth           │ Más rápido: solo 2 pasadas, usa FP-Tree                │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Uso típico          │ Market basket analysis, recomendación de productos     │
│                     │ Diagnóstico médico, análisis de log, NLP               │
│ Parámetros clave    │ min_support, min_confidence, min_lift                  │
│ Herramienta         │ mlxtend (apriori, fpgrowth, association_rules)         │
└─────────────────────┴───────────────────────────────────────────────────────┘

Guía de valores típicos:
  min_support:    0.01–0.10 (bajo = más reglas, más lentas de calcular)
  min_confidence: 0.50–0.80 (alta = reglas más confiables)
  min_lift:       > 1.0     (siempre filtrar por lift > 1)
```

---

## Checkpoint M10

```python
# checkpoint_m10.py
"""
Checkpoint M10 — Reglas de Asociación
Ejecutar: python checkpoint_m10.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from mlxtend.preprocessing import TransactionEncoder
from mlxtend.frequent_patterns import apriori, fpgrowth, association_rules
import pandas as pd

print("=" * 60)
print("CHECKPOINT M10 — Reglas de Asociación")
print("=" * 60)

# Dataset simple conocido
transacciones = [
    ['leche', 'pan', 'mantequilla'],
    ['leche', 'pan'],
    ['leche', 'mantequilla'],
    ['pan', 'mantequilla'],
    ['leche', 'pan', 'mantequilla'],
    ['pan'],
    ['leche'],
    ['leche', 'pan', 'mantequilla'],
    ['pan', 'mantequilla'],
    ['leche', 'pan'],
] * 20  # ampliar para más soporte

te = TransactionEncoder()
df = pd.DataFrame(te.fit_transform(transacciones), columns=te.columns_)
print(f"\n✓ Dataset: {len(transacciones)} transacciones | {df.shape[1]} productos")

# 1. Apriori
itemsets = apriori(df, min_support=0.3, use_colnames=True)
print(f"✓ Apriori (min_support=0.3): {len(itemsets)} itemsets frecuentes")
assert len(itemsets) > 0, "Apriori no encontró itemsets"

# 2. FP-Growth
itemsets_fp = fpgrowth(df, min_support=0.3, use_colnames=True)
print(f"✓ FP-Growth (min_support=0.3): {len(itemsets_fp)} itemsets frecuentes")
assert len(itemsets_fp) == len(itemsets), "Apriori y FP-Growth deben dar el mismo número de itemsets"

# 3. Reglas
reglas = association_rules(itemsets_fp, metric='lift', min_threshold=1.0)
print(f"✓ Reglas generadas: {len(reglas)}")
assert len(reglas) > 0, "No se generaron reglas"

# 4. Verificar métricas
assert (reglas['support'] > 0).all(),    "Soporte debe ser positivo"
assert (reglas['confidence'] <= 1).all(), "Confianza debe ser ≤ 1"
assert (reglas['lift'] >= 1.0).all(),     "Lift filtrado debe ser ≥ 1"
print("✓ Métricas (soporte, confianza, lift): rangos correctos")

# 5. Regla de mayor lift
mejor_regla = reglas.loc[reglas['lift'].idxmax()]
ant  = ', '.join(sorted(mejor_regla['antecedents']))
cons = ', '.join(sorted(mejor_regla['consequents']))
print(f"✓ Regla de mayor lift: {{{ant}}} → {{{cons}}} [lift={mejor_regla['lift']:.2f}]")
assert mejor_regla['lift'] > 1.0

# 6. Filtrado
reglas_filtradas = reglas[
    (reglas['confidence'] >= 0.6) &
    (reglas['lift'] >= 1.2)
]
print(f"✓ Reglas filtradas (conf≥0.6, lift≥1.2): {len(reglas_filtradas)}")

print("\n" + "=" * 60)
print("CHECKPOINT M10 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • TransactionEncoder (one-hot encoding de transacciones)")
print("  • Apriori con min_support")
print("  • FP-Growth (mismo resultado que Apriori)")
print("  • Generación y filtrado de reglas de asociación")
print("  • Métricas: soporte, confianza, lift")
print("=" * 60)
```

---

*Siguiente módulo → M11: Reducción de Dimensionalidad — PCA, LDA, t-SNE, UMAP*
