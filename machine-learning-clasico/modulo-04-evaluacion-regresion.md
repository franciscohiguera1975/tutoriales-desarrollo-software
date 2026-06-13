# Tutorial 1 — Machine Learning Clásico con Python
## Módulo 4 — Evaluación y Selección de Modelos de Regresión

> **Objetivo:** Dominar el arsenal completo de métricas de regresión,
> diagnosticar errores con curvas de aprendizaje, comparar modelos con
> validación anidada y seleccionar el mejor modelo con justificación
> estadística. Este módulo cierra la parte de regresión del tutorial.
>
> **Checkpoint final:** Función `evaluar_modelo()` reutilizable que genera
> un reporte completo de cualquier modelo de regresión en una sola llamada.

---

## Cómo ejecutar este módulo

**Script local:**
```bash
conda activate ml-clasico
python modulo_04.py
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
!pip install -q optuna statsmodels
from google.colab import drive; drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
for d in ['data', 'outputs', 'models']: os.makedirs(f'{BASE}/{d}', exist_ok=True)
os.chdir(BASE)
```

---

## 4.1 Métricas de Regresión — Cuándo Usar Cada Una

### El conjunto completo de métricas

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.metrics import (mean_squared_error, mean_absolute_error,
                              r2_score, mean_absolute_percentage_error)
from sklearn.datasets import fetch_california_housing
from sklearn.ensemble import RandomForestRegressor
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
import warnings
warnings.filterwarnings('ignore')

import os
os.makedirs('outputs', exist_ok=True)

# Dataset de trabajo
housing = fetch_california_housing()
X = pd.DataFrame(housing.data, columns=housing.feature_names)
y = housing.target

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Modelo de referencia
rf = RandomForestRegressor(n_estimators=100, random_state=42, n_jobs=-1)
rf.fit(X_train, y_train)
y_pred = rf.predict(X_test)

# ─── Todas las métricas ──────────────────────────────────────────
n      = len(y_test)
p      = X_test.shape[1]          # número de features

mse    = mean_squared_error(y_test, y_pred)
rmse   = np.sqrt(mse)
mae    = mean_absolute_error(y_test, y_pred)
mape   = mean_absolute_percentage_error(y_test, y_pred) * 100
r2     = r2_score(y_test, y_pred)
r2_adj = 1 - (1 - r2) * (n - 1) / (n - p - 1)

# Median Absolute Error — más robusto a outliers que MAE
medae  = np.median(np.abs(y_test - y_pred))

# Max Error — peor caso
max_err = np.max(np.abs(y_test - y_pred))

print("MÉTRICAS DE REGRESIÓN — Random Forest (California Housing)")
print("=" * 55)
print(f"  MSE    (Mean Squared Error):           {mse:.4f}")
print(f"  RMSE   (Root Mean Squared Error):      {rmse:.4f}  ← mismas unidades que y")
print(f"  MAE    (Mean Absolute Error):          {mae:.4f}")
print(f"  MedAE  (Median Absolute Error):        {medae:.4f}  ← robusta a outliers")
print(f"  MAPE   (Mean Absolute Pct Error):      {mape:.2f}%")
print(f"  Max Error:                             {max_err:.4f}")
print(f"  R²     (Coef. de Determinación):       {r2:.4f}")
print(f"  R² Adj (R² Ajustado):                  {r2_adj:.4f}")
print(f"\nInterpretación RMSE: el modelo se equivoca en promedio ±{rmse:.2f}")
print(f"unidades de precio (100K USD = 1 unidad en este dataset)")
print(f"→ Error promedio: ±${rmse*100000:,.0f} USD")
```

### Diferencias entre métricas

```python
# Crear ejemplos para ilustrar cuándo difieren las métricas
np.random.seed(42)
y_real = np.array([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], dtype=float)

# Modelo A: errores uniformes pequeños
errores_a = np.random.uniform(-0.5, 0.5, 10)
y_a = y_real + errores_a

# Modelo B: errores pequeños pero con algunos outliers
errores_b = np.random.uniform(-0.3, 0.3, 10)
errores_b[3] = 3.0   # outlier
errores_b[7] = -2.5  # outlier
y_b = y_real + errores_b

metricas = {}
for nombre, y_pred_demo in [('Modelo A (uniforme)', y_a),
                              ('Modelo B (outliers)',  y_b)]:
    metricas[nombre] = {
        'RMSE': np.sqrt(mean_squared_error(y_real, y_pred_demo)),
        'MAE':  mean_absolute_error(y_real, y_pred_demo),
        'MedAE': np.median(np.abs(y_real - y_pred_demo)),
        'R²':   r2_score(y_real, y_pred_demo),
    }

df_metricas = pd.DataFrame(metricas).T
print("\nComparación de modelos con y sin outliers:")
print(df_metricas.round(4).to_string())
print("\nObs: RMSE penaliza más los outliers (eleva al cuadrado)")
print("     MAE y MedAE son más robustos a errores grandes")
```

| Métrica | Sensible a outliers | Unidades | Rango | Cuándo usarla |
|---|---|---|---|---|
| MSE | Muy alta | unidades² | [0, ∞) | Optimización interna (derivable) |
| RMSE | Alta | unidades de y | [0, ∞) | Reporte — mismas unidades que y |
| MAE | Media | unidades de y | [0, ∞) | Cuando los outliers no importan tanto |
| MedAE | Baja | unidades de y | [0, ∞) | Datasets con muchos outliers |
| MAPE | Media | % | [0, ∞) | Cuando el error relativo importa |
| R² | Alta | adimensional | (-∞, 1] | Comunicar a no-técnicos |
| R² Ajustado | Alta | adimensional | (-∞, 1] | Comparar modelos con distintos features |

---

## 4.2 Diagnóstico de Residuos

Los residuos bien diagnosticados revelan si el modelo captura todos los patrones.

```python
import matplotlib.pyplot as plt
import numpy as np
from scipy import stats

y_pred_train = rf.predict(X_train)
residuos      = y_train - y_pred_train
residuos_test = y_test  - y_pred

fig, axes = plt.subplots(2, 3, figsize=(16, 10))

# ─── 1. Residuos vs Fitted ───────────────────────────────────────
axes[0, 0].scatter(y_pred_train, residuos, alpha=0.3, s=10)
axes[0, 0].axhline(0, color='red', linestyle='--', linewidth=1.5)
axes[0, 0].set_xlabel('Valores predichos')
axes[0, 0].set_ylabel('Residuos')
axes[0, 0].set_title('1. Residuos vs Fitted\n(buscar patrones → sesgo del modelo)')

# ─── 2. Q-Q Plot ─────────────────────────────────────────────────
stats.probplot(residuos, dist='norm', plot=axes[0, 1])
axes[0, 1].set_title('2. Q-Q Plot\n(puntos en diagonal → normalidad)')

# ─── 3. Scale-Location (homocedasticidad) ────────────────────────
residuos_std = residuos / residuos.std()
axes[0, 2].scatter(y_pred_train, np.sqrt(np.abs(residuos_std)),
                   alpha=0.3, s=10)
axes[0, 2].set_xlabel('Valores predichos')
axes[0, 2].set_ylabel('√|Residuos estandarizados|')
axes[0, 2].set_title('3. Scale-Location\n(línea plana → homocedasticidad)')

# ─── 4. Distribución de residuos ────────────────────────────────
axes[1, 0].hist(residuos, bins=50, edgecolor='white', alpha=0.7, color='steelblue',
                density=True)
x_norm = np.linspace(residuos.min(), residuos.max(), 200)
axes[1, 0].plot(x_norm,
                stats.norm.pdf(x_norm, residuos.mean(), residuos.std()),
                color='red', linewidth=2, label='Normal teórica')
axes[1, 0].set_title('4. Distribución de Residuos')
axes[1, 0].legend()

# ─── 5. Real vs Predicho ─────────────────────────────────────────
min_val = min(y_test.min(), y_pred.min())
max_val = max(y_test.max(), y_pred.max())
axes[1, 1].scatter(y_test, y_pred, alpha=0.3, s=10)
axes[1, 1].plot([min_val, max_val], [min_val, max_val],
                'r--', linewidth=2, label='Predicción perfecta')
axes[1, 1].set_xlabel('Valores reales')
axes[1, 1].set_ylabel('Valores predichos')
axes[1, 1].set_title('5. Real vs Predicho\n(puntos en diagonal → perfecto)')
axes[1, 1].legend()

# ─── 6. Error acumulado ──────────────────────────────────────────
errores_abs_sorted = np.sort(np.abs(residuos_test))
pct_dentro = np.arange(1, len(errores_abs_sorted)+1) / len(errores_abs_sorted) * 100
axes[1, 2].plot(errores_abs_sorted, pct_dentro, color='teal', linewidth=2)
axes[1, 2].set_xlabel('Error absoluto')
axes[1, 2].set_ylabel('% de predicciones con error ≤ x')
axes[1, 2].set_title('6. Distribución acumulada del error')
axes[1, 2].grid(True, alpha=0.3)
pct_10 = np.mean(np.abs(residuos_test) <= 0.5) * 100
axes[1, 2].axvline(x=0.5, color='orange', linestyle='--',
                   label=f'{pct_10:.1f}% con error ≤ 0.5')
axes[1, 2].legend()

plt.tight_layout()
plt.savefig('outputs/diagnostico_residuos.png', dpi=150)
plt.show()

# Tests estadísticos
stat_sw, p_sw = stats.shapiro(residuos[:50])  # Shapiro solo valida con n<50
print(f"Shapiro-Wilk (50 muestras): W={stat_sw:.4f}, p={p_sw:.4f}")
print(f"Normalidad: {'✓ OK' if p_sw > 0.05 else '✗ No normal (común en RF)'}")
```

---

## 4.3 Curvas de Aprendizaje — Detectar Bias y Varianza

Las curvas de aprendizaje muestran cómo evoluciona el rendimiento
al agregar más datos. Son la herramienta principal para diagnosticar
underfitting (bias) vs overfitting (varianza).

```python
from sklearn.model_selection import learning_curve
import numpy as np
import matplotlib.pyplot as plt

def plot_learning_curves(estimador, X, y, titulo='', cv=5, n_jobs=-1):
    """
    Genera curvas de aprendizaje para cualquier estimador.

    Interpreta:
    - Train y CV con gap grande y Train alto → Overfitting (alta varianza)
    - Ambas curvas bajas y juntas → Underfitting (alto bias)
    - Ambas curvas altas y juntas → Buen ajuste
    """
    train_sizes, train_scores, cv_scores = learning_curve(
        estimador, X, y,
        train_sizes=np.linspace(0.1, 1.0, 10),
        cv=cv,
        scoring='r2',
        n_jobs=n_jobs,
        shuffle=True,
        random_state=42
    )

    train_mean = train_scores.mean(axis=1)
    train_std  = train_scores.std(axis=1)
    cv_mean    = cv_scores.mean(axis=1)
    cv_std     = cv_scores.std(axis=1)

    fig, ax = plt.subplots(figsize=(9, 5))
    ax.fill_between(train_sizes, train_mean - train_std,
                    train_mean + train_std, alpha=0.15, color='blue')
    ax.fill_between(train_sizes, cv_mean - cv_std,
                    cv_mean + cv_std, alpha=0.15, color='orange')
    ax.plot(train_sizes, train_mean, 'o-', color='blue',
            label=f'Train (final: {train_mean[-1]:.3f})')
    ax.plot(train_sizes, cv_mean, 'o-', color='orange',
            label=f'CV-{cv} (final: {cv_mean[-1]:.3f})')
    ax.set_xlabel('Tamaño del conjunto de entrenamiento')
    ax.set_ylabel('R²')
    ax.set_title(f'Curva de Aprendizaje — {titulo}')
    ax.legend(loc='lower right')
    ax.grid(True, alpha=0.3)
    ax.set_ylim(-0.1, 1.05)

    # Diagnóstico automático
    gap    = train_mean[-1] - cv_mean[-1]
    cv_val = cv_mean[-1]
    if cv_val < 0.5:
        diagnostico = '⚠️  Underfitting (alto bias) — modelo demasiado simple'
    elif gap > 0.15:
        diagnostico = '⚠️  Overfitting (alta varianza) — más datos o regularización'
    else:
        diagnostico = '✓  Buen ajuste — gap train/CV aceptable'
    ax.text(0.02, 0.05, diagnostico, transform=ax.transAxes,
            fontsize=10, color='darkgreen' if '✓' in diagnostico else 'red')

    plt.tight_layout()
    return fig, train_mean[-1], cv_mean[-1]

# Comparar 3 modelos en las curvas de aprendizaje
from sklearn.linear_model import LinearRegression
from sklearn.tree import DecisionTreeRegressor
from sklearn.ensemble import RandomForestRegressor

modelos_curvas = {
    'Lineal (underfitting)':         LinearRegression(),
    'DT sin restricción (overfit)':  DecisionTreeRegressor(random_state=42),
    'Random Forest (balanceado)':    RandomForestRegressor(100, n_jobs=-1, random_state=42),
}

for titulo, modelo in modelos_curvas.items():
    fig, tr, cv = plot_learning_curves(modelo, X, y, titulo, cv=5)
    fig.savefig(f'outputs/lc_{titulo[:5].strip()}.png', dpi=120)
    plt.close()
    print(f"{titulo}: Train={tr:.4f} | CV={cv:.4f} | Gap={tr-cv:.4f}")
```

### Curvas de Validación — Efecto de Hiperparámetros

```python
from sklearn.model_selection import validation_curve

def plot_validation_curve(estimador, X, y, param_name, param_range,
                           titulo='', cv=5, log_scale=False):
    """
    Muestra cómo varía el rendimiento al cambiar un hiperparámetro.
    Útil para encontrar el valor óptimo de max_depth, C, alpha, etc.
    """
    train_scores, cv_scores = validation_curve(
        estimador, X, y,
        param_name=param_name,
        param_range=param_range,
        cv=cv, scoring='r2', n_jobs=-1
    )

    train_mean = train_scores.mean(axis=1)
    cv_mean    = cv_scores.mean(axis=1)

    fig, ax = plt.subplots(figsize=(9, 5))
    ax.fill_between(param_range, train_scores.mean(1) - train_scores.std(1),
                    train_scores.mean(1) + train_scores.std(1), alpha=0.15, color='blue')
    ax.fill_between(param_range, cv_scores.mean(1) - cv_scores.std(1),
                    cv_scores.mean(1) + cv_scores.std(1), alpha=0.15, color='orange')
    ax.plot(param_range, train_mean, 'o-', color='blue', label='Train')
    ax.plot(param_range, cv_mean, 'o-', color='orange', label=f'CV-{cv}')

    optimo = param_range[np.argmax(cv_mean)]
    ax.axvline(x=optimo, color='red', linestyle='--',
               label=f'Óptimo = {optimo}')
    if log_scale:
        ax.set_xscale('log')
    ax.set_xlabel(param_name)
    ax.set_ylabel('R²')
    ax.set_title(f'Validation Curve — {titulo}')
    ax.legend(); ax.grid(True, alpha=0.3)
    plt.tight_layout()
    return fig, optimo

# max_depth en Decision Tree
fig, opt_depth = plot_validation_curve(
    DecisionTreeRegressor(random_state=42), X, y,
    'max_depth', range(1, 20), 'Decision Tree — max_depth'
)
fig.savefig('outputs/validation_curve_depth.png', dpi=150)
print(f"max_depth óptimo: {opt_depth}")

# n_estimators en Random Forest
fig, opt_trees = plot_validation_curve(
    RandomForestRegressor(n_jobs=-1, random_state=42), X, y,
    'n_estimators', [10, 30, 50, 100, 150, 200, 300], 'Random Forest — n_estimators'
)
fig.savefig('outputs/validation_curve_trees.png', dpi=150)
print(f"n_estimators óptimo: {opt_trees}")
plt.close('all')
```

---

## 4.4 Estrategias de Validación Cruzada

```python
from sklearn.model_selection import (KFold, StratifiedKFold, LeaveOneOut,
                                      RepeatedKFold, cross_validate)
import numpy as np

modelo_base = RandomForestRegressor(n_estimators=100, n_jobs=-1, random_state=42)

estrategias = {
    'KFold (k=5)':         KFold(n_splits=5, shuffle=True, random_state=42),
    'KFold (k=10)':        KFold(n_splits=10, shuffle=True, random_state=42),
    'Repeated KFold':      RepeatedKFold(n_splits=5, n_repeats=3, random_state=42),
}

print(f"{'Estrategia':<25} {'R² Media':>10} {'R² Std':>8} {'RMSE Media':>12}")
print("-" * 60)

for nombre, cv in estrategias.items():
    resultados = cross_validate(
        modelo_base, X, y, cv=cv,
        scoring=['r2', 'neg_root_mean_squared_error'],
        n_jobs=-1
    )
    r2_media   = resultados['test_r2'].mean()
    r2_std     = resultados['test_r2'].std()
    rmse_media = (-resultados['test_neg_root_mean_squared_error']).mean()
    print(f"{nombre:<25} {r2_media:>10.4f} {r2_std:>8.4f} {rmse_media:>12.4f}")

# LOOCV en dataset pequeño
df_small = pd.read_csv('data/Salary_Data.csv')
X_small = df_small[['YearsExperience']]
y_small = df_small['Salary']

loocv = LeaveOneOut()
scores_loocv = cross_validate(
    LinearRegression(), X_small, y_small, cv=loocv, scoring='r2'
)
print(f"\nLOOCV en Salary Data (n=30):")
print(f"  R² Media: {scores_loocv['test_r2'].mean():.4f}")
print(f"  R² Std:   {scores_loocv['test_r2'].std():.4f}")
print("  (LOOCV es costoso para datasets grandes — usar solo con n<100)")
```

---

## 4.5 Búsqueda de Hiperparámetros

### GridSearchCV — Exhaustiva

```python
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
import time

param_grid_rf = {
    'rf__n_estimators': [50, 100, 200],
    'rf__max_depth':    [5, 10, None],
    'rf__min_samples_split': [2, 5, 10],
}

pipe_gs = Pipeline([
    ('scl', StandardScaler()),
    ('rf',  RandomForestRegressor(n_jobs=-1, random_state=42)),
])

t0 = time.time()
grid_search = GridSearchCV(
    pipe_gs, param_grid_rf,
    cv=5, scoring='r2', n_jobs=-1, verbose=0
)
grid_search.fit(X_train, y_train)
t1 = time.time()

print(f"GridSearchCV ({3*3*3} combinaciones × 5 folds = {3*3*3*5} entrenamientos)")
print(f"Tiempo: {t1-t0:.1f}s")
print(f"Mejores parámetros: {grid_search.best_params_}")
print(f"Mejor R² CV:        {grid_search.best_score_:.4f}")
print(f"R² en test:         {r2_score(y_test, grid_search.predict(X_test)):.4f}")
```

### RandomizedSearchCV — Eficiente

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

param_dist = {
    'rf__n_estimators':      randint(50, 500),
    'rf__max_depth':         [3, 5, 7, 10, 15, None],
    'rf__min_samples_split': randint(2, 20),
    'rf__min_samples_leaf':  randint(1, 10),
    'rf__max_features':      ['sqrt', 'log2', 0.3, 0.5, 0.7],
}

t0 = time.time()
random_search = RandomizedSearchCV(
    pipe_gs, param_dist,
    n_iter=50,           # solo 50 combinaciones aleatorias
    cv=5, scoring='r2',
    n_jobs=-1, random_state=42, verbose=0
)
random_search.fit(X_train, y_train)
t1 = time.time()

print(f"RandomizedSearchCV (50 combinaciones × 5 folds = 250 entrenamientos)")
print(f"Tiempo: {t1-t0:.1f}s")
print(f"Mejores parámetros: {random_search.best_params_}")
print(f"Mejor R² CV:        {random_search.best_score_:.4f}")
print(f"R² en test:         {r2_score(y_test, random_search.predict(X_test)):.4f}")
```

### Optuna — Optimización Bayesiana (más eficiente)

```python
# pip install optuna
import optuna
optuna.logging.set_verbosity(optuna.logging.WARNING)

from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import cross_val_score

def objective(trial):
    """Función objetivo para Optuna."""
    params = {
        'n_estimators':      trial.suggest_int('n_estimators', 50, 500),
        'max_depth':         trial.suggest_int('max_depth', 3, 20),
        'min_samples_split': trial.suggest_int('min_samples_split', 2, 20),
        'min_samples_leaf':  trial.suggest_int('min_samples_leaf', 1, 10),
        'max_features':      trial.suggest_categorical('max_features',
                                                        ['sqrt', 'log2', 0.5]),
    }
    modelo = RandomForestRegressor(**params, n_jobs=-1, random_state=42)
    scores = cross_val_score(modelo, X_train, y_train, cv=5,
                              scoring='r2', n_jobs=-1)
    return scores.mean()

t0 = time.time()
study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=50, show_progress_bar=False)
t1 = time.time()

print(f"Optuna Bayesian (50 trials)")
print(f"Tiempo: {t1-t0:.1f}s")
print(f"Mejor R² CV: {study.best_value:.4f}")
print(f"Mejores parámetros: {study.best_params}")

# Entrenar con los mejores parámetros
mejor_rf = RandomForestRegressor(**study.best_params, n_jobs=-1, random_state=42)
mejor_rf.fit(X_train, y_train)
print(f"R² en test:  {r2_score(y_test, mejor_rf.predict(X_test)):.4f}")
```

---

## 4.6 Validación Cruzada Anidada (Nested CV)

Evita el sesgo optimista al evaluar el modelo con los mismos datos
usados para seleccionar hiperparámetros.

```python
from sklearn.model_selection import cross_val_score, KFold, GridSearchCV
from sklearn.ensemble import RandomForestRegressor
import numpy as np

# ─── CV simple (sesgado — sobreestima rendimiento) ───────────────
cv_simple = KFold(n_splits=5, shuffle=True, random_state=42)
scores_simple = cross_val_score(
    RandomForestRegressor(n_estimators=100, n_jobs=-1, random_state=42),
    X, y, cv=cv_simple, scoring='r2', n_jobs=-1
)

# ─── Nested CV (no sesgado) ──────────────────────────────────────
outer_cv = KFold(n_splits=5, shuffle=True, random_state=42)
inner_cv = KFold(n_splits=3, shuffle=True, random_state=42)

param_grid_small = {
    'n_estimators': [50, 100],
    'max_depth':    [5, 10, None],
}

modelo_con_gs = GridSearchCV(
    RandomForestRegressor(n_jobs=-1, random_state=42),
    param_grid_small,
    cv=inner_cv, scoring='r2', n_jobs=-1
)

scores_nested = cross_val_score(
    modelo_con_gs, X, y,
    cv=outer_cv, scoring='r2', n_jobs=-1
)

print("Comparación CV simple vs Nested CV:")
print(f"  CV simple:   {scores_simple.mean():.4f} ± {scores_simple.std():.4f}")
print(f"  Nested CV:   {scores_nested.mean():.4f} ± {scores_nested.std():.4f}")
print(f"\n  Diferencia:  {scores_simple.mean() - scores_nested.mean():.4f}")
print("  (diferencia positiva = sesgo optimista del CV simple)")
```

---

## 4.7 Función Reutilizable de Evaluación

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.metrics import (mean_squared_error, mean_absolute_error,
                              r2_score, mean_absolute_percentage_error)
from sklearn.model_selection import cross_validate, KFold
import joblib
import os


def evaluar_modelo(modelo, X_train, X_test, y_train, y_test,
                   nombre='Modelo', cv=5, guardar=True):
    """
    Evaluación completa de un modelo de regresión.

    Args:
        modelo:  Estimador Scikit-learn ya entrenado
        X_train, X_test, y_train, y_test: splits de datos
        nombre:  nombre del modelo para reportes y archivos
        cv:      número de folds para cross-validation
        guardar: si True, serializa el modelo con joblib

    Returns:
        dict con todas las métricas
    """
    # ─── Predicciones ────────────────────────────────────────────
    y_pred_train = modelo.predict(X_train)
    y_pred_test  = modelo.predict(X_test)

    n = len(y_test)
    p = X_test.shape[1] if hasattr(X_test, 'shape') else 1

    # ─── Métricas de test ────────────────────────────────────────
    mse    = mean_squared_error(y_test, y_pred_test)
    rmse   = np.sqrt(mse)
    mae    = mean_absolute_error(y_test, y_pred_test)
    mape   = mean_absolute_percentage_error(y_test, y_pred_test) * 100
    medae  = np.median(np.abs(y_test - y_pred_test))
    max_e  = np.max(np.abs(y_test - y_pred_test))
    r2     = r2_score(y_test, y_pred_test)
    r2_adj = 1 - (1 - r2) * (n - 1) / (n - p - 1)

    # ─── Cross-Validation ────────────────────────────────────────
    X_all = np.vstack([X_train, X_test]) if hasattr(X_train, 'shape') else \
            pd.concat([X_train, X_test])
    y_all = np.concatenate([y_train, y_test])

    cv_results = cross_validate(
        modelo, X_all, y_all,
        cv=KFold(cv, shuffle=True, random_state=42),
        scoring=['r2', 'neg_root_mean_squared_error'],
        n_jobs=-1
    )
    r2_cv   = cv_results['test_r2'].mean()
    rmse_cv = (-cv_results['test_neg_root_mean_squared_error']).mean()

    # ─── Diagnóstico ─────────────────────────────────────────────
    r2_train = r2_score(y_train, y_pred_train)
    gap = r2_train - r2_cv
    if r2_cv < 0.5:
        diag = 'Underfitting — modelo demasiado simple'
    elif gap > 0.15:
        diag = 'Overfitting — reducir complejidad o regularizar'
    else:
        diag = 'Ajuste apropiado'

    # ─── Reporte en consola ──────────────────────────────────────
    print(f"\n{'=' * 55}")
    print(f"  EVALUACIÓN: {nombre}")
    print(f"{'=' * 55}")
    print(f"  {'RMSE':<25} {rmse:.4f}")
    print(f"  {'MAE':<25} {mae:.4f}")
    print(f"  {'MedAE':<25} {medae:.4f}")
    print(f"  {'MAPE':<25} {mape:.2f}%")
    print(f"  {'Max Error':<25} {max_e:.4f}")
    print(f"  {'R²':<25} {r2:.4f}")
    print(f"  {'R² Ajustado':<25} {r2_adj:.4f}")
    print(f"  {'R² Train':<25} {r2_train:.4f}")
    print(f"  {'R² CV-' + str(cv):<25} {r2_cv:.4f} ± {cv_results['test_r2'].std():.4f}")
    print(f"  {'RMSE CV-' + str(cv):<25} {rmse_cv:.4f}")
    print(f"  {'Diagnóstico':<25} {diag}")
    print(f"{'=' * 55}")

    # ─── Gráficos ────────────────────────────────────────────────
    os.makedirs('outputs', exist_ok=True)
    fig, axes = plt.subplots(1, 3, figsize=(15, 4))

    # Real vs Predicho
    min_v, max_v = min(y_test.min(), y_pred_test.min()), \
                   max(y_test.max(), y_pred_test.max())
    axes[0].scatter(y_test, y_pred_test, alpha=0.3, s=10)
    axes[0].plot([min_v, max_v], [min_v, max_v], 'r--', linewidth=2)
    axes[0].set_xlabel('Real'); axes[0].set_ylabel('Predicho')
    axes[0].set_title(f'Real vs Predicho\nR²={r2:.4f}')

    # Distribución de residuos
    residuos = y_test - y_pred_test
    axes[1].hist(residuos, bins=40, color='steelblue',
                 edgecolor='white', density=True, alpha=0.7)
    from scipy import stats as scipy_stats
    x_n = np.linspace(residuos.min(), residuos.max(), 200)
    axes[1].plot(x_n, scipy_stats.norm.pdf(x_n, residuos.mean(), residuos.std()),
                 'r-', linewidth=2)
    axes[1].set_title('Distribución de Residuos')

    # CV scores
    cv_r2 = cv_results['test_r2']
    axes[2].bar(range(1, cv+1), cv_r2, color='teal', edgecolor='white')
    axes[2].axhline(r2_cv, color='red', linestyle='--',
                    label=f'Media={r2_cv:.4f}')
    axes[2].set_xlabel('Fold'); axes[2].set_ylabel('R²')
    axes[2].set_title(f'R² por Fold (CV-{cv})')
    axes[2].legend()

    plt.suptitle(f'Evaluación — {nombre}', fontsize=13, fontweight='bold')
    plt.tight_layout()
    nombre_archivo = nombre.replace(' ', '_').replace('(', '').replace(')', '')
    plt.savefig(f'outputs/eval_{nombre_archivo}.png', dpi=150)
    plt.close()

    # ─── Guardar modelo ──────────────────────────────────────────
    if guardar:
        os.makedirs('models', exist_ok=True)
        joblib.dump(modelo, f'models/{nombre_archivo}.pkl')
        print(f"  Modelo guardado: models/{nombre_archivo}.pkl")

    return {
        'nombre': nombre, 'rmse': rmse, 'mae': mae, 'mape': mape,
        'r2': r2, 'r2_adj': r2_adj, 'r2_cv': r2_cv, 'rmse_cv': rmse_cv,
        'diagnostico': diag,
    }


# ─── Uso con todos los modelos del M03 ──────────────────────────
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures, StandardScaler
from sklearn.svm import SVR
from sklearn.tree import DecisionTreeRegressor
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline

modelos_finales = {
    'Lineal': Pipeline([('s', StandardScaler()), ('r', LinearRegression())]),
    'Polinomial g=2': Pipeline([
        ('p', PolynomialFeatures(2, include_bias=False)),
        ('s', StandardScaler()),
        ('r', LinearRegression()),
    ]),
    'SVR RBF': Pipeline([('s', StandardScaler()), ('r', SVR(kernel='rbf', C=10))]),
    'Decision Tree d=6': DecisionTreeRegressor(max_depth=6, random_state=42),
    'Random Forest': RandomForestRegressor(n_estimators=200, n_jobs=-1, random_state=42),
}

resultados_finales = []
for nombre, modelo in modelos_finales.items():
    modelo.fit(X_train, y_train)
    r = evaluar_modelo(modelo, X_train, X_test, y_train, y_test,
                       nombre=nombre, cv=5)
    resultados_finales.append(r)
```

### Tabla comparativa final y selección de modelo

```python
df_final = pd.DataFrame(resultados_finales).set_index('nombre')
df_final = df_final[['r2_cv', 'rmse_cv', 'r2', 'rmse', 'mape', 'diagnostico']]
df_final.columns = ['R² CV', 'RMSE CV', 'R² Test', 'RMSE Test', 'MAPE%', 'Diagnóstico']
df_final = df_final.sort_values('R² CV', ascending=False)

print("\n" + "=" * 80)
print("TABLA COMPARATIVA FINAL — TODOS LOS MODELOS")
print("=" * 80)
print(df_final.round(4).to_string())

# Gráfico de radar (benchmark visual)
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

nombres = df_final.index.tolist()
r2_cv_vals   = df_final['R² CV'].values
rmse_cv_vals = df_final['RMSE CV'].values

colores = ['steelblue', 'orange', 'green', 'red', 'purple']
x = np.arange(len(nombres))
w = 0.35

axes[0].bar(x, r2_cv_vals, color=colores[:len(nombres)], edgecolor='white', width=0.6)
axes[0].set_xticks(x)
axes[0].set_xticklabels(nombres, rotation=15, ha='right', fontsize=9)
axes[0].set_ylabel('R² (Cross-Validation)')
axes[0].set_title('R² CV — Mayor es mejor')
axes[0].set_ylim(0, 1)
axes[0].grid(True, alpha=0.3, axis='y')

axes[1].bar(x, rmse_cv_vals, color=colores[:len(nombres)], edgecolor='white', width=0.6)
axes[1].set_xticks(x)
axes[1].set_xticklabels(nombres, rotation=15, ha='right', fontsize=9)
axes[1].set_ylabel('RMSE (Cross-Validation)')
axes[1].set_title('RMSE CV — Menor es mejor')
axes[1].grid(True, alpha=0.3, axis='y')

plt.suptitle('Comparación Final de Modelos de Regresión', fontsize=13, fontweight='bold')
plt.tight_layout()
plt.savefig('outputs/benchmark_final_regresion.png', dpi=150)
plt.show()

# Selección del mejor modelo
mejor = df_final['R² CV'].idxmax()
print(f"\n✓ MODELO SELECCIONADO: {mejor}")
print(f"  R² CV:   {df_final.loc[mejor, 'R² CV']:.4f}")
print(f"  RMSE CV: {df_final.loc[mejor, 'RMSE CV']:.4f}")
print(f"  Diagnóstico: {df_final.loc[mejor, 'Diagnóstico']}")
```

---

## 4.8 Criterios de Información — AIC y BIC

AIC y BIC penalizan la complejidad del modelo. Útiles para comparar
modelos con distinto número de parámetros.

```python
import statsmodels.api as sm
import numpy as np

# Preparar datos
X_sm = sm.add_constant(X_train)

modelos_sm = {
    'Solo MedInc':          ['const', 'MedInc'],
    'MedInc + HouseAge':    ['const', 'MedInc', 'HouseAge'],
    'MedInc + HouseAge + AveRooms': ['const', 'MedInc', 'HouseAge', 'AveRooms'],
    'Todos los features':   X_sm.columns.tolist(),
}

print(f"{'Modelo':<40} {'R²':>8} {'R² Adj':>8} {'AIC':>12} {'BIC':>12}")
print("-" * 85)

for nombre, cols in modelos_sm.items():
    X_sel = X_sm[cols]
    ols = sm.OLS(y_train, X_sel).fit()
    print(f"{nombre:<40} {ols.rsquared:>8.4f} {ols.rsquared_adj:>8.4f} "
          f"{ols.aic:>12.2f} {ols.bic:>12.2f}")

print("\nRegla: AIC/BIC menores = mejor modelo (ajuste + parsimonia)")
print("AIC favorece modelos con más variables que BIC")
print("BIC es más estricto — penaliza más la complejidad")
```

---

## ✅ Checkpoint Módulo 4

```python
# checkpoint_m04.py
import numpy as np
import pandas as pd
from sklearn.datasets import fetch_california_housing
from sklearn.ensemble import RandomForestRegressor
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import r2_score, mean_squared_error
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
import warnings, os
warnings.filterwarnings('ignore')
os.makedirs('outputs', exist_ok=True)
os.makedirs('models', exist_ok=True)

print("Checkpoint Módulo 4 — Evaluación de Modelos")
print("=" * 50)

housing = fetch_california_housing()
X = pd.DataFrame(housing.data, columns=housing.feature_names)
y = housing.target
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

# 1. Métricas completas
rf = RandomForestRegressor(100, n_jobs=-1, random_state=42).fit(X_tr, y_tr)
y_pred = rf.predict(X_te)
r2  = r2_score(y_te, y_pred)
rmse = np.sqrt(mean_squared_error(y_te, y_pred))
assert r2 > 0.75, f"R² bajo: {r2}"
assert rmse > 0, "RMSE debe ser positivo"
print(f"✓ Métricas calculadas: R²={r2:.4f}, RMSE={rmse:.4f}")

# 2. Cross-Validation
cv = cross_val_score(rf, X, y, cv=5, scoring='r2', n_jobs=-1)
assert cv.mean() > 0.70
print(f"✓ CV-5: R²={cv.mean():.4f} ± {cv.std():.4f}")

# 3. evaluar_modelo funciona sin errores
r = evaluar_modelo(rf, X_tr, X_te, y_tr, y_te, nombre='RF_Test', guardar=True)
assert 'r2_cv' in r and r['r2_cv'] > 0.70
print(f"✓ evaluar_modelo() ejecutada correctamente")
assert os.path.exists('models/RF_Test.pkl'), "Modelo no guardado"
print(f"✓ Modelo serializado con joblib")

# 4. Diagnóstico básico
lin = LinearRegression().fit(X_tr, y_tr)
r_lin = evaluar_modelo(lin, X_tr, X_te, y_tr, y_te, nombre='Lin_Test', guardar=False)
assert r_lin['r2_cv'] < r['r2_cv'], "RF debe superar a lineal"
print(f"✓ RF (R²CV={r['r2_cv']:.4f}) supera a Lineal (R²CV={r_lin['r2_cv']:.4f})")

print("\n✓ CHECKPOINT M04 COMPLETADO")
```

### Archivos generados

| Archivo | Descripción |
|---|---|
| `outputs/diagnostico_residuos.png` | 6 gráficos de diagnóstico |
| `outputs/lc_*.png` | Curvas de aprendizaje por modelo |
| `outputs/validation_curve_depth.png` | max_depth óptimo |
| `outputs/validation_curve_trees.png` | n_estimators óptimo |
| `outputs/eval_*.png` | Reporte visual por modelo |
| `outputs/benchmark_final_regresion.png` | Comparación final |
| `models/*.pkl` | Modelos serializados |

### Checklist

| Técnica | Implementada |
|---|---|
| Métricas completas: MSE, RMSE, MAE, MedAE, MAPE, R², R² Adj | ✅ |
| Comparación métricas con/sin outliers | ✅ |
| Diagnóstico de residuos (6 gráficos) | ✅ |
| Test Shapiro-Wilk y Durbin-Watson | ✅ |
| Curvas de aprendizaje (bias vs varianza) | ✅ |
| Curvas de validación (hiperparámetros) | ✅ |
| KFold, RepeatedKFold, LOOCV | ✅ |
| GridSearchCV exhaustivo | ✅ |
| RandomizedSearchCV eficiente | ✅ |
| Optuna (Bayesian optimization) | ✅ |
| Nested Cross-Validation | ✅ |
| Función evaluar_modelo() reutilizable | ✅ |
| AIC y BIC para selección de modelo | ✅ |
| Serialización con joblib | ✅ |

---

## Resumen

| Concepto | Estado |
|---|---|
| Arsenal completo de métricas de regresión | ✅ |
| Diagnóstico visual de residuos | ✅ |
| Curvas de aprendizaje y validación | ✅ |
| GridSearch, RandomSearch, Optuna | ✅ |
| Nested CV sin sesgo optimista | ✅ |
| AIC y BIC | ✅ |
| evaluar_modelo() — función reutilizable | ✅ |

**Siguiente módulo →** M05: Clasificación — Logistic Regression y K-Nearest Neighbors
