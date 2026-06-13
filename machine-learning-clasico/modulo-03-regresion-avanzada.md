# Tutorial 1 — Machine Learning Clásico con Python
## Módulo 3 — Regresión Avanzada: Polinomial, SVR, Decision Tree y Random Forest

> **Objetivo:** Implementar modelos de regresión no lineales para relaciones
> que no pueden capturarse con una línea recta. Entender cuándo usar cada
> modelo, sus hiperparámetros críticos y cómo evitar overfitting.
>
> **Checkpoint final:** Comparación de los 5 modelos de regresión en el mismo
> dataset con las mismas métricas, y selección del mejor modelo con justificación.

---

## Cómo ejecutar este módulo

**Script local:**
```bash
conda activate ml-clasico
python modulo_03.py
```

**Jupyter — primera celda:**
```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
```

**Google Colab — primera celda:**
```python
!pip install -q xgboost optuna
from google.colab import drive; drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
for d in ['data', 'outputs', 'models']: os.makedirs(f'{BASE}/{d}', exist_ok=True)
os.chdir(BASE)
```

---

## 3.1 Regresión Polinomial

### Concepto

La regresión polinomial extiende la lineal agregando potencias de X:

```
ŷ = b₀ + b₁x + b₂x² + b₃x³ + ... + bₙxⁿ
```

**Truco:** Sigue siendo una regresión LINEAL en los coeficientes.
Se crean features nuevas (x², x³, ...) y se aplica OLS normal.
Por eso se usa `LinearRegression` combinado con `PolynomialFeatures`.

### Dataset — Nivel de Posición vs Salario

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import os

os.makedirs('data', exist_ok=True)
os.makedirs('outputs', exist_ok=True)

datos = {
    'Level':  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
    'Salary': [45000, 50000, 60000, 80000, 110000,
               150000, 200000, 300000, 500000, 1000000]
}

df = pd.DataFrame(datos)
df.to_csv('data/Position_Salaries.csv', index=False)
print(df)
```

### Comparar regresión lineal vs polinomial

```python
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import Pipeline
from sklearn.metrics import r2_score
import numpy as np

df = pd.read_csv('data/Position_Salaries.csv')
X = df[['Level']]
y = df['Salary']

# ─── Modelo lineal ───────────────────────────────────────────────
lin_reg = LinearRegression().fit(X, y)
r2_lin = r2_score(y, lin_reg.predict(X))

# ─── Modelos polinomiales de grados 2, 3, 4, 5 ───────────────────
grados = [2, 3, 4, 5]
modelos_poly = {}
r2_poly = {}

for grado in grados:
    pipe = Pipeline([
        ('poly', PolynomialFeatures(degree=grado, include_bias=False)),
        ('reg',  LinearRegression()),
    ])
    pipe.fit(X, y)
    r2_poly[grado] = r2_score(y, pipe.predict(X))
    modelos_poly[grado] = pipe

print(f"R² Lineal:          {r2_lin:.4f}")
for g, r2 in r2_poly.items():
    print(f"R² Polinomial g={g}: {r2:.4f}")

# ─── Visualización ───────────────────────────────────────────────
x_plot = np.linspace(1, 10, 300).reshape(-1, 1)
x_plot_df = pd.DataFrame(x_plot, columns=['Level'])

fig, axes = plt.subplots(2, 3, figsize=(16, 10))
axes = axes.flatten()

# Lineal
axes[0].scatter(X, y, color='red', zorder=5, label='Datos reales')
axes[0].plot(x_plot, lin_reg.predict(x_plot_df), color='blue')
axes[0].set_title(f'Lineal (R²={r2_lin:.3f})')
axes[0].set_xlabel('Nivel'); axes[0].set_ylabel('Salario')

# Polinomiales
colores = ['green', 'orange', 'purple', 'brown']
for i, (grado, color) in enumerate(zip(grados, colores)):
    ax = axes[i + 1]
    ax.scatter(X, y, color='red', zorder=5, label='Datos reales')
    ax.plot(x_plot, modelos_poly[grado].predict(x_plot_df), color=color, linewidth=2)
    ax.set_title(f'Grado {grado} (R²={r2_poly[grado]:.4f})')
    ax.set_xlabel('Nivel'); ax.set_ylabel('Salario')

# R² vs Grado
axes[5].plot([1] + grados, [r2_lin] + list(r2_poly.values()),
             marker='o', color='teal', linewidth=2)
axes[5].set_xlabel('Grado del polinomio')
axes[5].set_ylabel('R²')
axes[5].set_title('R² vs Grado (cuidado con overfitting)')
axes[5].grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('outputs/regresion_polinomial.png', dpi=150)
plt.show()
```

### Predecir con el modelo polinomial

```python
# Predecir salario para Nivel 6.5
nivel_nuevo = pd.DataFrame([[6.5]], columns=['Level'])

pred_lin  = lin_reg.predict(nivel_nuevo)[0]
pred_poly4 = modelos_poly[4].predict(nivel_nuevo)[0]

print(f"Predicción Nivel 6.5:")
print(f"  Lineal:           ${pred_lin:,.0f}")
print(f"  Polinomial g=4:   ${pred_poly4:,.0f}")
print(f"\nEl salario real en nivel 6 es $150,000 y nivel 7 es $200,000")
print(f"El polinomial interpola mejor la realidad no lineal")
```

### Overfitting en regresión polinomial

```python
# Demostración de overfitting con grado alto en pocos datos
np.random.seed(42)
X_demo = np.linspace(0, 1, 10).reshape(-1, 1)
y_demo = np.sin(2 * np.pi * X_demo).ravel() + np.random.randn(10) * 0.2

x_test_demo = np.linspace(0, 1, 200).reshape(-1, 1)

fig, axes = plt.subplots(1, 3, figsize=(15, 5))
grados_demo = [1, 4, 15]

for ax, g in zip(axes, grados_demo):
    pipe = Pipeline([
        ('poly', PolynomialFeatures(degree=g)),
        ('reg',  LinearRegression()),
    ])
    pipe.fit(X_demo, y_demo)
    y_plot = pipe.predict(x_test_demo)

    r2_train = r2_score(y_demo, pipe.predict(X_demo))
    ax.scatter(X_demo, y_demo, color='red', s=80, zorder=5, label='Datos')
    ax.plot(x_test_demo, y_plot, linewidth=2, label=f'Grado {g}')
    ax.set_ylim(-3, 3)
    ax.set_title(f'Grado {g} | R² train={r2_train:.3f}')
    ax.legend()

axes[0].set_title('Grado 1 — Underfitting')
axes[1].set_title('Grado 4 — Buen ajuste')
axes[2].set_title('Grado 15 — Overfitting')

plt.tight_layout()
plt.savefig('outputs/overfitting_polinomial.png', dpi=150)
plt.show()

print("Grado 1:  underfitting — muy simple para los datos")
print("Grado 4:  buen ajuste — captura la curvatura real")
print("Grado 15: overfitting — memoriza el ruido, falla en nuevos datos")
```

---

## 3.2 Support Vector Regression (SVR)

### Concepto

SVR busca un **tubo** (epsilon-insensitive tube) alrededor de los datos.
Solo los puntos FUERA del tubo contribuyen al error. Los puntos dentro
del tubo se ignoran — esto hace a SVR robusto a outliers.

```
Minimizar: (1/2)||w||² + C·Σ(ξᵢ + ξᵢ*)
```

Donde:
- `ε` (epsilon) = ancho del tubo — puntos dentro no penalizan
- `C` = regularización — controla el trade-off sesgo/varianza
- `kernel` = función que mapea a espacio de mayor dimensión

### Implementación SVR

```python
from sklearn.svm import SVR
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.model_selection import GridSearchCV
import numpy as np
import pandas as pd

df = pd.read_csv('data/Position_Salaries.csv')
X = df[['Level']].values
y = df['Salary'].values

# ⚠️ SVR requiere feature scaling — muy importante
# Los salarios varían de 45K a 1M → sin escalar el kernel no funciona bien

pipeline_svr = Pipeline([
    ('scaler_x', StandardScaler()),
    ('svr', SVR(kernel='rbf', C=1e3, gamma=0.1, epsilon=0.1)),
])

pipeline_svr.fit(X, y)

# Predicciones
x_plot = np.linspace(1, 10, 300).reshape(-1, 1)
y_pred_svr = pipeline_svr.predict(x_plot)
y_train_pred = pipeline_svr.predict(X)

r2_svr = r2_score(y, y_train_pred)
print(f"SVR R² en train: {r2_svr:.4f}")
print(f"Predicción Nivel 6.5: ${pipeline_svr.predict([[6.5]])[0]:,.0f}")

# Visualización
plt.figure(figsize=(8, 5))
plt.scatter(X, y, color='red', label='Datos reales', zorder=5)
plt.plot(x_plot, y_pred_svr, color='blue', linewidth=2, label='SVR (RBF)')
plt.xlabel('Nivel de Posición')
plt.ylabel('Salario (USD)')
plt.title(f'SVR con Kernel RBF (R²={r2_svr:.4f})')
plt.legend()
plt.tight_layout()
plt.savefig('outputs/svr.png', dpi=150)
plt.show()
```

### Kernels disponibles en SVR

```python
import matplotlib.pyplot as plt

kernels = {
    'linear':        SVR(kernel='linear', C=1e3),
    'poly (g=3)':    SVR(kernel='poly', degree=3, C=1e3, gamma='scale'),
    'rbf':           SVR(kernel='rbf', C=1e3, gamma=0.1),
    'sigmoid':       SVR(kernel='sigmoid', C=1e3, gamma='scale'),
}

fig, axes = plt.subplots(2, 2, figsize=(14, 10))
axes = axes.flatten()

for ax, (nombre, svr) in zip(axes, kernels.items()):
    pipe = Pipeline([
        ('scl', StandardScaler()),
        ('svr', svr),
    ])
    pipe.fit(X, y)
    y_plot = pipe.predict(x_plot)
    r2     = r2_score(y, pipe.predict(X))

    ax.scatter(X, y, color='red', zorder=5, s=80)
    ax.plot(x_plot, y_plot, linewidth=2, color='navy')
    ax.set_title(f'SVR kernel={nombre} | R²={r2:.4f}')
    ax.set_xlabel('Nivel'); ax.set_ylabel('Salario')

plt.tight_layout()
plt.savefig('outputs/svr_kernels.png', dpi=150)
plt.show()
```

### GridSearch de hiperparámetros SVR

```python
from sklearn.model_selection import GridSearchCV, KFold

param_grid = {
    'svr__C':       [0.1, 1, 10, 100, 1000],
    'svr__gamma':   [0.01, 0.1, 1, 'scale', 'auto'],
    'svr__epsilon': [0.01, 0.1, 0.5, 1.0],
}

grid_search = GridSearchCV(
    Pipeline([('scl', StandardScaler()), ('svr', SVR(kernel='rbf'))]),
    param_grid,
    cv=KFold(n_splits=5, shuffle=True, random_state=42),
    scoring='r2',
    n_jobs=-1,
    verbose=1
)

grid_search.fit(X, y)
print(f"Mejores parámetros: {grid_search.best_params_}")
print(f"Mejor R² (CV-5):   {grid_search.best_score_:.4f}")
```

---

## 3.3 Decision Tree Regression

### Concepto

Un árbol de decisión divide recursivamente el espacio de features en
regiones rectangulares. Para regresión, predice la **media** del target
en cada hoja.

```
Si Level <= 6.5:
    Si Level <= 3.5:
        Predicción = 53,333
    Sino:
        Predicción = 120,000
Sino:
    Si Level <= 8.5:
        Predicción = 250,000
    Sino:
        Predicción = 750,000
```

### Implementación

```python
from sklearn.tree import DecisionTreeRegressor, export_text, plot_tree
from sklearn.metrics import r2_score, mean_squared_error
import numpy as np

df = pd.read_csv('data/Position_Salaries.csv')
X = df[['Level']].values
y = df['Salary'].values

# Árbol sin restricciones — puede overfit
dt_full = DecisionTreeRegressor(random_state=42)
dt_full.fit(X, y)
r2_full = r2_score(y, dt_full.predict(X))
print(f"Árbol sin restricciones:")
print(f"  Profundidad: {dt_full.get_depth()}")
print(f"  Hojas:       {dt_full.get_n_leaves()}")
print(f"  R²:          {r2_full:.4f}  ← memoriza los datos")

# Árbol con profundidad máxima (regularización)
dt_reg = DecisionTreeRegressor(max_depth=3, min_samples_leaf=2, random_state=42)
dt_reg.fit(X, y)
r2_reg = r2_score(y, dt_reg.predict(X))
print(f"\nÁrbol con max_depth=3:")
print(f"  Profundidad: {dt_reg.get_depth()}")
print(f"  Hojas:       {dt_reg.get_n_leaves()}")
print(f"  R²:          {r2_reg:.4f}")

# Texto del árbol
print("\nEstructura del árbol (max_depth=3):")
print(export_text(dt_reg, feature_names=['Level']))
```

### Visualizar el árbol

```python
fig, axes = plt.subplots(1, 2, figsize=(18, 6))

# Árbol visual
plot_tree(dt_reg, feature_names=['Level'],
          filled=True, rounded=True, fontsize=10, ax=axes[0])
axes[0].set_title('Decision Tree (max_depth=3)')

# Curva de predicción — escalera característica de los árboles
x_plot = np.linspace(1, 10, 1000).reshape(-1, 1)
axes[1].scatter(X, y, color='red', zorder=5, s=80, label='Datos')
axes[1].plot(x_plot, dt_full.predict(x_plot), color='blue',
             linewidth=2, label='Sin restricciones')
axes[1].plot(x_plot, dt_reg.predict(x_plot), color='green',
             linewidth=2, label='max_depth=3')
axes[1].set_xlabel('Nivel'); axes[1].set_ylabel('Salario')
axes[1].set_title('Decision Tree — Predicciones en forma de escalera')
axes[1].legend()

plt.tight_layout()
plt.savefig('outputs/decision_tree_reg.png', dpi=150)
plt.show()
```

### Importancia de variables

```python
# En datasets con múltiples features, los árboles dan importancia automática
from sklearn.datasets import fetch_california_housing

housing = fetch_california_housing()
X_h = pd.DataFrame(housing.data, columns=housing.feature_names)
y_h = housing.target

dt_housing = DecisionTreeRegressor(max_depth=5, random_state=42)
dt_housing.fit(X_h, y_h)

importancias = pd.Series(
    dt_housing.feature_importances_,
    index=housing.feature_names
).sort_values(ascending=True)

fig, ax = plt.subplots(figsize=(8, 5))
importancias.plot(kind='barh', ax=ax, color='steelblue')
ax.set_xlabel('Importancia (Gini importance)')
ax.set_title('Feature Importance — Decision Tree (California Housing)')
plt.tight_layout()
plt.savefig('outputs/feature_importance_dt.png', dpi=150)
plt.show()

print("Feature Importances:")
print(importancias.sort_values(ascending=False).to_string())
```

### Hiperparámetros críticos del Decision Tree

```python
from sklearn.model_selection import cross_val_score

# Efecto de max_depth
profundidades = range(1, 15)
scores_train = []
scores_cv    = []

for depth in profundidades:
    dt = DecisionTreeRegressor(max_depth=depth, random_state=42)
    dt.fit(X_h, y_h)
    scores_train.append(r2_score(y_h, dt.predict(X_h)))
    cv = cross_val_score(dt, X_h, y_h, cv=5, scoring='r2')
    scores_cv.append(cv.mean())

fig, ax = plt.subplots(figsize=(9, 5))
ax.plot(profundidades, scores_train, 'o-', label='Train R²', color='blue')
ax.plot(profundidades, scores_cv,    'o-', label='CV R²',    color='orange')
ax.axvline(x=np.argmax(scores_cv)+1, color='red', linestyle='--',
           label=f'Mejor depth={np.argmax(scores_cv)+1}')
ax.set_xlabel('max_depth')
ax.set_ylabel('R²')
ax.set_title('Bias-Variance Tradeoff — max_depth')
ax.legend(); ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/bias_variance_dt.png', dpi=150)
plt.show()

print(f"Mejor max_depth por CV: {np.argmax(scores_cv)+1}")
print(f"R² CV máximo: {max(scores_cv):.4f}")
```

---

## 3.4 Random Forest Regression

### Concepto

Random Forest es un **ensemble** de muchos árboles de decisión.
Cada árbol se entrena con:
1. **Bootstrap sampling**: muestra aleatoria CON reemplazo del dataset
2. **Feature subsampling**: en cada nodo, solo considera √n features

La predicción final es el **promedio** de todos los árboles.
Esto reduce el overfitting drásticamente respecto a un árbol solo.

```
RF Prediction = (1/N) · Σᵢ TreePredictionᵢ
```

### Implementación

```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import r2_score, mean_squared_error
import numpy as np

df = pd.read_csv('data/Position_Salaries.csv')
X = df[['Level']].values
y = df['Salary'].values

# Random Forest con 300 árboles
rf = RandomForestRegressor(
    n_estimators=300,    # número de árboles
    max_depth=None,      # árboles completos
    max_features=1.0,    # usar todas las features (dataset pequeño)
    random_state=42,
    n_jobs=-1            # usar todos los cores
)
rf.fit(X, y)

x_plot = np.linspace(1, 10, 1000).reshape(-1, 1)
y_rf   = rf.predict(x_plot)
r2_rf  = r2_score(y, rf.predict(X))

print(f"Random Forest R²: {r2_rf:.4f}")
print(f"Predicción Nivel 6.5: ${rf.predict([[6.5]])[0]:,.0f}")

plt.figure(figsize=(9, 5))
plt.scatter(X, y, color='red', zorder=5, s=80, label='Datos')
plt.plot(x_plot, y_rf, color='green', linewidth=2, label='Random Forest (300 árboles)')
plt.xlabel('Nivel'); plt.ylabel('Salario')
plt.title(f'Random Forest Regression (R²={r2_rf:.4f})')
plt.legend()
plt.tight_layout()
plt.savefig('outputs/random_forest_reg.png', dpi=150)
plt.show()
```

### Efecto del número de árboles

```python
n_trees_range = [1, 5, 10, 50, 100, 200, 300, 500]

housing = fetch_california_housing()
X_h = pd.DataFrame(housing.data, columns=housing.feature_names)
y_h = housing.target
X_tr, X_te, y_tr, y_te = train_test_split(X_h, y_h, test_size=0.2, random_state=42)

r2_train_list = []
r2_test_list  = []

for n in n_trees_range:
    rf_n = RandomForestRegressor(n_estimators=n, n_jobs=-1, random_state=42)
    rf_n.fit(X_tr, y_tr)
    r2_train_list.append(r2_score(y_tr, rf_n.predict(X_tr)))
    r2_test_list.append(r2_score(y_te, rf_n.predict(X_te)))

plt.figure(figsize=(9, 5))
plt.plot(n_trees_range, r2_train_list, 'o-', label='Train R²')
plt.plot(n_trees_range, r2_test_list,  'o-', label='Test R²')
plt.xlabel('Número de árboles (n_estimators)')
plt.ylabel('R²')
plt.title('R² vs Número de Árboles — Random Forest')
plt.legend(); plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('outputs/rf_n_estimators.png', dpi=150)
plt.show()

print(f"R² Test con 1 árbol:   {r2_test_list[0]:.4f}  ← inestable")
print(f"R² Test con 100 árboles: {r2_test_list[4]:.4f}")
print(f"R² Test con 500 árboles: {r2_test_list[-1]:.4f}")
print("\nObs: Más árboles no siempre mejora — hay un punto de retorno decreciente")
```

### Feature Importance en Random Forest

```python
rf_housing = RandomForestRegressor(n_estimators=200, n_jobs=-1, random_state=42)
rf_housing.fit(X_tr, y_tr)

importancias_rf = pd.Series(
    rf_housing.feature_importances_,
    index=housing.feature_names
).sort_values(ascending=True)

fig, ax = plt.subplots(figsize=(8, 5))
importancias_rf.plot(kind='barh', ax=ax, color='forestgreen')
ax.set_xlabel('Importancia')
ax.set_title('Feature Importance — Random Forest (California Housing)')
plt.tight_layout()
plt.savefig('outputs/feature_importance_rf.png', dpi=150)
plt.show()
```

### RandomizedSearchCV — Búsqueda eficiente de hiperparámetros

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

param_distributions = {
    'n_estimators':    randint(50, 500),
    'max_depth':       [None, 5, 10, 20, 30],
    'min_samples_split': randint(2, 20),
    'min_samples_leaf':  randint(1, 10),
    'max_features':    ['sqrt', 'log2', 0.5, 0.7, 1.0],
}

random_search = RandomizedSearchCV(
    RandomForestRegressor(n_jobs=-1, random_state=42),
    param_distributions=param_distributions,
    n_iter=30,          # probar 30 combinaciones aleatorias
    cv=5,
    scoring='r2',
    n_jobs=-1,
    random_state=42,
    verbose=1
)

random_search.fit(X_tr, y_tr)
print(f"\nMejores parámetros: {random_search.best_params_}")
print(f"Mejor R² (CV-5):    {random_search.best_score_:.4f}")
print(f"R² en test:         {r2_score(y_te, random_search.predict(X_te)):.4f}")
```

---

## 3.5 Comparación de Todos los Modelos

```python
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures, StandardScaler
from sklearn.svm import SVR
from sklearn.tree import DecisionTreeRegressor
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline
from sklearn.model_selection import cross_val_score, train_test_split
from sklearn.metrics import r2_score, mean_squared_error, mean_absolute_error
from sklearn.datasets import fetch_california_housing
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

housing = fetch_california_housing()
X = pd.DataFrame(housing.data, columns=housing.feature_names)
y = housing.target

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

modelos = {
    'Lineal':           Pipeline([('scl', StandardScaler()),
                                   ('reg', LinearRegression())]),
    'Polinomial g=2':   Pipeline([('poly', PolynomialFeatures(2, include_bias=False)),
                                   ('scl', StandardScaler()),
                                   ('reg', LinearRegression())]),
    'SVR (RBF)':        Pipeline([('scl', StandardScaler()),
                                   ('svr', SVR(kernel='rbf', C=10, gamma=0.1))]),
    'Decision Tree':    DecisionTreeRegressor(max_depth=6, random_state=42),
    'Random Forest':    RandomForestRegressor(n_estimators=200, n_jobs=-1,
                                              random_state=42),
}

resultados = []

for nombre, modelo in modelos.items():
    modelo.fit(X_train, y_train)
    y_pred = modelo.predict(X_test)
    r2     = r2_score(y_test, y_pred)
    rmse   = np.sqrt(mean_squared_error(y_test, y_pred))
    mae    = mean_absolute_error(y_test, y_pred)
    cv     = cross_val_score(modelo, X, y, cv=5, scoring='r2',
                              n_jobs=-1).mean()
    resultados.append({
        'Modelo': nombre,
        'R² Test': round(r2, 4),
        'RMSE': round(rmse, 4),
        'MAE': round(mae, 4),
        'R² CV-5': round(cv, 4),
    })

df_resultados = pd.DataFrame(resultados).sort_values('R² CV-5', ascending=False)
print("COMPARACIÓN DE MODELOS — California Housing")
print("=" * 65)
print(df_resultados.to_string(index=False))

# Visualización comparativa
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# R² Test y CV
x = np.arange(len(df_resultados))
w = 0.35
axes[0].bar(x - w/2, df_resultados['R² Test'], w, label='R² Test', color='steelblue')
axes[0].bar(x + w/2, df_resultados['R² CV-5'], w, label='R² CV-5', color='orange')
axes[0].set_xticks(x)
axes[0].set_xticklabels(df_resultados['Modelo'], rotation=15, ha='right')
axes[0].set_ylabel('R²')
axes[0].set_title('R² Test vs R² Cross-Validation')
axes[0].legend(); axes[0].grid(True, alpha=0.3, axis='y')

# RMSE
axes[1].bar(df_resultados['Modelo'], df_resultados['RMSE'],
            color=['#e74c3c', '#e67e22', '#3498db', '#27ae60', '#9b59b6'])
axes[1].set_xticklabels(df_resultados['Modelo'], rotation=15, ha='right')
axes[1].set_ylabel('RMSE')
axes[1].set_title('RMSE por Modelo (menor es mejor)')
axes[1].grid(True, alpha=0.3, axis='y')

plt.tight_layout()
plt.savefig('outputs/comparacion_modelos_regresion.png', dpi=150)
plt.show()
```

### Cuándo usar cada modelo

```python
guia = {
    'Lineal':         'Relación lineal entre features y target. Pocos features. Interpretabilidad requerida.',
    'Polinomial':     'Relación curvilínea clara. Pocos features. Cuidado con overfitting en grado alto.',
    'SVR':            'Datasets medianos (<50K filas). Robusto a outliers. Requiere scaling. RBF por defecto.',
    'Decision Tree':  'Relaciones no lineales. Alta interpretabilidad. Propenso a overfitting sin max_depth.',
    'Random Forest':  'Caso general. Alta precisión. Tolerante a outliers. Robusto sin mucho tuning.',
}

print("GUÍA DE SELECCIÓN DE MODELO DE REGRESIÓN")
print("=" * 70)
for modelo, cuando in guia.items():
    print(f"\n{modelo}:")
    print(f"  → {cuando}")
```

---

## ✅ Checkpoint Módulo 3

```python
# checkpoint_m03.py
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures, StandardScaler
from sklearn.svm import SVR
from sklearn.tree import DecisionTreeRegressor
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline
from sklearn.metrics import r2_score
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.datasets import fetch_california_housing
import warnings
warnings.filterwarnings('ignore')

print("Checkpoint Módulo 3 — Regresión Avanzada")
print("=" * 45)

housing = fetch_california_housing()
X = pd.DataFrame(housing.data, columns=housing.feature_names)
y = housing.target
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

# 1. Polinomial
pipe_poly = Pipeline([
    ('poly', PolynomialFeatures(degree=2, include_bias=False)),
    ('scl',  StandardScaler()),
    ('reg',  LinearRegression()),
])
pipe_poly.fit(X_tr, y_tr)
r2_poly = r2_score(y_te, pipe_poly.predict(X_te))
assert r2_poly > 0.50, f"R² polinomial bajo: {r2_poly}"
print(f"✓ Polinomial g=2  R² Test = {r2_poly:.4f}")

# 2. SVR
pipe_svr = Pipeline([('scl', StandardScaler()), ('svr', SVR(kernel='rbf', C=10))])
pipe_svr.fit(X_tr, y_tr)
r2_svr = r2_score(y_te, pipe_svr.predict(X_te))
assert r2_svr > 0.50
print(f"✓ SVR RBF         R² Test = {r2_svr:.4f}")

# 3. Decision Tree
dt = DecisionTreeRegressor(max_depth=6, random_state=42)
dt.fit(X_tr, y_tr)
r2_dt = r2_score(y_te, dt.predict(X_te))
assert r2_dt > 0.60
print(f"✓ Decision Tree   R² Test = {r2_dt:.4f}")

# 4. Random Forest
rf = RandomForestRegressor(n_estimators=100, n_jobs=-1, random_state=42)
rf.fit(X_tr, y_tr)
r2_rf = r2_score(y_te, rf.predict(X_te))
assert r2_rf > 0.75, f"R² RF bajo: {r2_rf}"
print(f"✓ Random Forest   R² Test = {r2_rf:.4f}")

# 5. Ranking
modelos_r2 = {'Polinomial': r2_poly, 'SVR': r2_svr, 'DT': r2_dt, 'RF': r2_rf}
mejor = max(modelos_r2, key=modelos_r2.get)
print(f"\n✓ Mejor modelo en este dataset: {mejor} (R²={modelos_r2[mejor]:.4f})")
print("\n✓ CHECKPOINT M03 COMPLETADO")
```

### Archivos generados

| Archivo | Descripción |
|---|---|
| `data/Position_Salaries.csv` | Dataset Nivel vs Salario |
| `outputs/regresion_polinomial.png` | Comparación grados 1–5 |
| `outputs/overfitting_polinomial.png` | Under/Over/Buen ajuste |
| `outputs/svr.png` | SVR con kernel RBF |
| `outputs/svr_kernels.png` | Comparación de 4 kernels |
| `outputs/decision_tree_reg.png` | Árbol visual + predicciones |
| `outputs/bias_variance_dt.png` | Train vs CV por profundidad |
| `outputs/random_forest_reg.png` | RF vs datos reales |
| `outputs/rf_n_estimators.png` | R² vs número de árboles |
| `outputs/comparacion_modelos_regresion.png` | Benchmark final |

### Checklist

| Técnica | Implementada |
|---|---|
| Regresión Polinomial grados 2–5 | ✅ |
| Visualización overfitting vs underfitting | ✅ |
| SVR con 4 kernels (linear, poly, rbf, sigmoid) | ✅ |
| GridSearch de hiperparámetros SVR | ✅ |
| Decision Tree — profundidad controlada | ✅ |
| Visualización árbol y curva escalera | ✅ |
| Feature importance Decision Tree | ✅ |
| Bias-variance tradeoff por max_depth | ✅ |
| Random Forest 300 árboles | ✅ |
| Efecto de n_estimators | ✅ |
| RandomizedSearchCV | ✅ |
| Benchmark comparativo 5 modelos | ✅ |

---

## Resumen

| Modelo | Fortaleza principal | Debilidad principal |
|---|---|---|
| Polinomial | Curvas simples, interpretable | Overfitting en grado alto |
| SVR | Robusto a outliers, kernel trick | Lento en datasets grandes |
| Decision Tree | Interpretable, no lineal | Overfitting sin poda |
| Random Forest | Alta precisión, robusto | Menos interpretable |

**Siguiente módulo →** M04: Evaluación y Selección de Modelos de Regresión
