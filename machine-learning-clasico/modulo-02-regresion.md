# Tutorial 1 — Machine Learning Clásico con Python
## Módulo 2 — Regresión Simple y Múltiple

> **Objetivo:** Implementar y comprender regresión lineal simple y múltiple
> desde la matemática hasta la producción. Entender los supuestos del modelo,
> cómo diagnosticar violaciones y cómo interpretar correctamente los
> coeficientes.
>
> **Checkpoint final:** Modelo de regresión múltiple completo con diagnóstico
> de supuestos, selección de variables y comparación de rendimiento entre
> implementaciones (NumPy, Scikit-learn, statsmodels).

---

## Cómo ejecutar este módulo

> Ver la sección completa de entornos en **M01**. El flujo es idéntico
> para todos los módulos — solo cambia la celda de instalación en Colab.

**Script local:**
```bash
conda activate ml-clasico
python modulo_02.py
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
!pip install -q statsmodels missingno optuna
from google.colab import drive; drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
for d in ['data', 'outputs', 'models']: os.makedirs(f'{BASE}/{d}', exist_ok=True)
os.chdir(BASE)
```

---

## 2.1 Regresión Lineal Simple

### Concepto matemático

La regresión lineal simple modela la relación entre una variable
independiente X y una variable dependiente y mediante una línea recta:

```
ŷ = b₀ + b₁·x
```

Donde:
- `b₀` = intercepto (valor de y cuando x = 0)
- `b₁` = pendiente (cambio en y por cada unidad de x)
- `ŷ` = valor predicho

El objetivo es encontrar b₀ y b₁ que minimicen la suma de errores al
cuadrado (OLS — Ordinary Least Squares):

```
MSE = (1/n) · Σ(yᵢ - ŷᵢ)²
```

La solución analítica exacta es:

```
b₁ = Σ(xᵢ - x̄)(yᵢ - ȳ) / Σ(xᵢ - x̄)²
b₀ = ȳ - b₁·x̄
```

### Dataset — Salario vs Años de Experiencia

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import os

os.makedirs('data', exist_ok=True)
os.makedirs('outputs', exist_ok=True)

# Crear dataset
datos = {
    'YearsExperience': [1.1, 1.3, 1.5, 2.0, 2.2, 2.9, 3.0, 3.2,
                         3.2, 3.7, 3.9, 4.0, 4.0, 4.1, 4.5, 4.9,
                         5.1, 5.3, 5.9, 6.0, 6.8, 7.1, 7.9, 8.2,
                         8.7, 9.0, 9.5, 9.6, 10.3, 10.5],
    'Salary':          [39343, 46205, 37731, 43525, 39891, 56642,
                         60150, 54445, 64445, 57189, 63218, 55794,
                         56957, 57081, 61111, 67938, 66029, 83088,
                         81363, 93940, 91738, 98273, 101302, 113812,
                         109431, 105582, 116969, 112635, 122391, 121872]
}

df = pd.DataFrame(datos)
df.to_csv('data/Salary_Data.csv', index=False)

print(df.describe())
print(f"\nCorrelación: {df['YearsExperience'].corr(df['Salary']):.4f}")
```

### Implementación desde cero — NumPy

```python
import numpy as np
import pandas as pd

df = pd.read_csv('data/Salary_Data.csv')
X = df['YearsExperience'].values
y = df['Salary'].values

# ─── Solución analítica OLS ──────────────────────────────────────
x_media = X.mean()
y_media = y.mean()

b1 = np.sum((X - x_media) * (y - y_media)) / np.sum((X - x_media)**2)
b0 = y_media - b1 * x_media

print(f"Intercepto (b₀): {b0:.2f}")
print(f"Pendiente  (b₁): {b1:.2f}")
print(f"Ecuación: Salary = {b0:.2f} + {b1:.2f} * YearsExperience")

# Predicciones
y_pred = b0 + b1 * X

# Métricas
mse  = np.mean((y - y_pred)**2)
rmse = np.sqrt(mse)
mae  = np.mean(np.abs(y - y_pred))
ss_res = np.sum((y - y_pred)**2)
ss_tot = np.sum((y - y_media)**2)
r2   = 1 - (ss_res / ss_tot)

print(f"\nMétricas:")
print(f"  MSE:  {mse:,.2f}")
print(f"  RMSE: {rmse:,.2f}  ← en las mismas unidades que y (USD)")
print(f"  MAE:  {mae:,.2f}")
print(f"  R²:   {r2:.4f}  ← {r2*100:.1f}% de varianza explicada")
```

### Implementación con Scikit-learn

```python
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score

df = pd.read_csv('data/Salary_Data.csv')
X = df[['YearsExperience']]  # DataFrame 2D requerido por sklearn
y = df['Salary']

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=0
)

modelo = LinearRegression()
modelo.fit(X_train, y_train)

print(f"Intercepto: {modelo.intercept_:.2f}")
print(f"Coeficiente: {modelo.coef_[0]:.2f}")

# Predicciones en test
y_pred = modelo.predict(X_test)

print(f"\nMétricas en TEST:")
print(f"  RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):,.2f}")
print(f"  MAE:  {mean_absolute_error(y_test, y_pred):,.2f}")
print(f"  R²:   {r2_score(y_test, y_pred):.4f}")

# Predicción para un nuevo valor
nuevo = np.array([[8.5]])
prediccion = modelo.predict(nuevo)
print(f"\nPredicción para 8.5 años de experiencia: ${prediccion[0]:,.0f}")
```

### Visualización completa

```python
import matplotlib.pyplot as plt
import numpy as np

fig, axes = plt.subplots(1, 3, figsize=(16, 5))

# ─── Gráfico 1: Datos y recta de regresión ───────────────────────
axes[0].scatter(X_train, y_train, color='blue', label='Train', alpha=0.7)
axes[0].scatter(X_test, y_test, color='orange', label='Test', alpha=0.7)
x_line = np.linspace(X.min().values[0], X.max().values[0], 100).reshape(-1, 1)
axes[0].plot(x_line, modelo.predict(x_line), color='red', linewidth=2,
             label=f'Regresión (R²={r2_score(y_test, y_pred):.3f})')
axes[0].set_xlabel('Años de Experiencia')
axes[0].set_ylabel('Salario (USD)')
axes[0].set_title('Regresión Lineal Simple')
axes[0].legend()

# ─── Gráfico 2: Residuos vs Fitted ──────────────────────────────
y_pred_train = modelo.predict(X_train)
residuos = y_train.values - y_pred_train
axes[1].scatter(y_pred_train, residuos, alpha=0.7, color='green')
axes[1].axhline(y=0, color='red', linestyle='--')
axes[1].set_xlabel('Valores predichos')
axes[1].set_ylabel('Residuos')
axes[1].set_title('Residuos vs Fitted')

# ─── Gráfico 3: Distribución de residuos ────────────────────────
axes[2].hist(residuos, bins=10, color='purple', edgecolor='white', alpha=0.7)
axes[2].set_xlabel('Residuo')
axes[2].set_ylabel('Frecuencia')
axes[2].set_title('Distribución de Residuos')

plt.tight_layout()
plt.savefig('outputs/regresion_simple.png', dpi=150)
plt.show()
```

### Interpretación del R²

```python
# R² = coeficiente de determinación
# R² = 0.95 → el modelo explica el 95% de la variabilidad de y
# R² = 1.0  → ajuste perfecto (sospechar overfitting)
# R² = 0.0  → el modelo no es mejor que predecir siempre la media
# R² < 0    → el modelo es peor que predecir la media (algo está mal)

r2 = r2_score(y_test, y_pred)
print(f"R² = {r2:.4f}")
print(f"El modelo explica el {r2*100:.1f}% de la varianza del salario")
print(f"El {(1-r2)*100:.1f}% restante depende de factores no incluidos")
print(f"  (sector, empresa, educación, habilidades, etc.)")
```

---

## 2.2 Supuestos de la Regresión Lineal

La regresión OLS tiene 5 supuestos. Violarlos invalida los resultados.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy import stats
from sklearn.linear_model import LinearRegression

df = pd.read_csv('data/Salary_Data.csv')
X = df[['YearsExperience']]
y = df['Salary']

modelo = LinearRegression().fit(X, y)
y_pred = modelo.predict(X)
residuos = y.values - y_pred

fig, axes = plt.subplots(2, 3, figsize=(16, 10))

# ─── Supuesto 1: Linealidad ──────────────────────────────────────
# Los residuos no deben mostrar patrón sistemático
axes[0, 0].scatter(y_pred, residuos, alpha=0.7)
axes[0, 0].axhline(0, color='red', linestyle='--')
axes[0, 0].set_title('1. Linealidad\n(Residuos vs Fitted)')
axes[0, 0].set_xlabel('Fitted'); axes[0, 0].set_ylabel('Residuos')

# ─── Supuesto 2: Normalidad de residuos ─────────────────────────
# Q-Q plot: los puntos deben seguir la diagonal
stats.probplot(residuos, dist="norm", plot=axes[0, 1])
axes[0, 1].set_title('2. Normalidad\n(Q-Q Plot)')

# Prueba estadística
stat, p = stats.shapiro(residuos)
print(f"Test de Shapiro-Wilk: W={stat:.4f}, p={p:.4f}")
print(f"Normalidad: {'✓ OK (p > 0.05)' if p > 0.05 else '✗ Violada (p ≤ 0.05)'}")

# ─── Supuesto 3: Homocedasticidad ───────────────────────────────
# Los residuos deben tener varianza constante (no ensancharse)
residuos_std = residuos / residuos.std()
axes[0, 2].scatter(y_pred, np.sqrt(np.abs(residuos_std)), alpha=0.7)
axes[0, 2].set_title('3. Homocedasticidad\n(Scale-Location)')
axes[0, 2].set_xlabel('Fitted'); axes[0, 2].set_ylabel('√|Residuos std|')

# ─── Supuesto 4: Independencia (Durbin-Watson) ───────────────────
from statsmodels.stats.stattools import durbin_watson
dw = durbin_watson(residuos)
print(f"\nDurbin-Watson: {dw:.4f}")
print(f"Independencia: {'✓ OK (cercano a 2)' if 1.5 < dw < 2.5 else '✗ Posible autocorrelación'}")

axes[1, 0].plot(residuos, marker='o', linestyle='-', alpha=0.7)
axes[1, 0].axhline(0, color='red', linestyle='--')
axes[1, 0].set_title(f'4. Independencia\n(Durbin-Watson = {dw:.3f})')
axes[1, 0].set_xlabel('Orden'); axes[1, 0].set_ylabel('Residuos')

# ─── Distribución de residuos ────────────────────────────────────
axes[1, 1].hist(residuos, bins=12, edgecolor='white', alpha=0.7, color='purple')
axes[1, 1].set_title('Distribución de Residuos')

# ─── Valores reales vs predichos ────────────────────────────────
axes[1, 2].scatter(y, y_pred, alpha=0.7, color='teal')
axes[1, 2].plot([y.min(), y.max()], [y.min(), y.max()], 'r--', linewidth=2)
axes[1, 2].set_title('Real vs Predicho\n(Línea ideal = diagonal)')
axes[1, 2].set_xlabel('Real'); axes[1, 2].set_ylabel('Predicho')

plt.tight_layout()
plt.savefig('outputs/supuestos_regresion.png', dpi=150)
plt.show()
```

---

## 2.3 Regresión Lineal Múltiple

### Concepto matemático

Cuando hay múltiples variables independientes:

```
ŷ = b₀ + b₁x₁ + b₂x₂ + ... + bₙxₙ
```

En forma matricial:

```
ŷ = Xβ
β = (XᵀX)⁻¹Xᵀy   ← solución analítica OLS
```

### Dataset — 50 Startups

```python
import pandas as pd
import numpy as np
import os

os.makedirs('data', exist_ok=True)

# Dataset: gastos de 50 startups vs ganancia
datos_50 = {
    'R&D Spend':       [165349.2, 162597.7, 153441.51, 144372.41, 142107.34,
                         131876.9, 134615.46, 130298.13, 120542.52, 123334.88,
                         101913.08, 100671.96, 93863.75, 91992.39, 119943.24,
                         114523.61, 78013.11, 94657.16, 91749.16, 86419.7,
                         76253.86, 78389.47, 73994.56, 67532.53, 77044.01,
                         64664.71, 75328.87, 72107.6, 66051.52, 65605.48,
                         61994.48, 61136.38, 63408.86, 55493.95, 46426.07,
                         46014.02, 28663.76, 44069.95, 20229.59, 38558.51,
                         28754.33, 27892.92, 23640.93, 15505.73, 22177.74,
                         1000.23, 1315.46, 0, 542.05, 0],
    'Administration':  [136897.8, 151377.59, 101145.55, 118671.85, 91391.77,
                         99814.71, 147198.87, 145530.06, 148718.95, 108679.17,
                         110594.11, 91790.61, 127320.38, 135495.07, 156547.42,
                         122616.84, 121597.55, 145077.58, 114175.79, 153514.11,
                         113867.3, 153773.43, 122782.75, 116861.72, 99281.34,
                         139553.16, 144135.98, 127864.55, 182645.56, 153032.06,
                         115641.28, 152701.92, 129219.61, 103057.49, 157693.92,
                         85047.44, 44069.95, 51283.14, 65947.93, 82982.09,
                         54906.32, 51239.39, 58612.01, 35534.17, 28334.72,
                         124153.04, 115816.21, 113000.0, 51827.84, 0],
    'Marketing Spend': [471784.1, 443898.53, 407934.54, 383199.62, 366168.42,
                         362861.36, 127716.82, 323876.68, 311613.29, 304981.62,
                         229160.95, 249744.55, 249839.44, 252664.93, 256512.92,
                         261776.23, 264346.06, 282574.31, 294919.57, 0,
                         298664.47, 299737.29, 303319.26, 304768.73, 140574.81,
                         137448.87, 134050.07, 353183.81, 118148.2, 107138.38,
                         91131.24, 88218.23, 46085.25, 214634.81, 210797.67,
                         205517.64, 0, 182806.75, 185265.1, 174999.3,
                         0, 0, 0, 0, 0, 1903.93, 297114.46, 0, 0, 0],
    'State':           ['New York', 'California', 'Florida', 'New York', 'Florida',
                         'New York', 'California', 'Florida', 'New York', 'California',
                         'Florida', 'California', 'Florida', 'California', 'Florida',
                         'New York', 'California', 'California', 'Florida', 'New York',
                         'California', 'Florida', 'New York', 'California', 'California',
                         'Florida', 'California', 'New York', 'California', 'New York',
                         'California', 'New York', 'California', 'Florida', 'New York',
                         'California', 'Florida', 'New York', 'California', 'New York',
                         'California', 'Florida', 'New York', 'Florida', 'California',
                         'New York', 'California', 'Florida', 'California', 'New York'],
    'Profit':          [192261.83, 191792.06, 191050.39, 182901.99, 166187.94,
                         156991.12, 156122.51, 155752.6, 152211.77, 149759.96,
                         146121.95, 144259.4, 141585.52, 134307.35, 132602.65,
                         129917.04, 126992.93, 125370.37, 124266.9, 122776.86,
                         118474.03, 111313.02, 110352.25, 108733.99, 108552.04,
                         107404.34, 105733.54, 105008.31, 103282.38, 101004.64,
                         99937.59, 97483.56, 97427.84, 96778.92, 96712.8,
                         96479.51, 90708.19, 89949.14, 81229.06, 81005.76,
                         78239.91, 77798.83, 71498.49, 69758.98, 65200.33,
                         64926.08, 49490.75, 42559.73, 35673.41, 14681.4]
}

df50 = pd.DataFrame(datos_50)
df50.to_csv('data/50_Startups.csv', index=False)
print(f"Dataset 50 Startups: {df50.shape}")
print(df50.head())
```

### Análisis exploratorio para regresión múltiple

```python
import seaborn as sns
import matplotlib.pyplot as plt

df = pd.read_csv('data/50_Startups.csv')

# Matriz de correlación
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

num_cols = ['R&D Spend', 'Administration', 'Marketing Spend', 'Profit']
corr = df[num_cols].corr()
sns.heatmap(corr, annot=True, fmt='.3f', cmap='coolwarm',
            center=0, ax=axes[0])
axes[0].set_title('Matriz de Correlación')

# Scatter matrix
pd.plotting.scatter_matrix(df[num_cols], alpha=0.6,
                            figsize=(8, 8), diagonal='hist')
plt.suptitle('Scatter Matrix — 50 Startups', y=1.01)

plt.tight_layout()
plt.savefig('outputs/exploracion_50startups.png', dpi=150)
plt.show()

# Correlaciones con Profit
print("\nCorrelación con Profit:")
for col in ['R&D Spend', 'Administration', 'Marketing Spend']:
    r = df[col].corr(df['Profit'])
    print(f"  {col:<20}: {r:.4f}")
```

### Preprocesamiento y entrenamiento

```python
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import OneHotEncoder
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.metrics import r2_score, mean_squared_error
import numpy as np

df = pd.read_csv('data/50_Startups.csv')

X = df.drop('Profit', axis=1)
y = df['Profit']

# Pipeline con One-Hot para 'State'
preprocessor = ColumnTransformer([
    ('ohe', OneHotEncoder(drop='first'), ['State']),
    # drop='first' evita multicolinealidad perfecta (dummy variable trap)
], remainder='passthrough')

pipeline = Pipeline([
    ('pre', preprocessor),
    ('reg', LinearRegression()),
])

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=0
)

pipeline.fit(X_train, y_train)
y_pred = pipeline.predict(X_test)

rmse = np.sqrt(mean_squared_error(y_test, y_pred))
r2   = r2_score(y_test, y_pred)

print(f"RMSE:  ${rmse:,.2f}")
print(f"R²:    {r2:.4f}")

# Coeficientes
reg = pipeline.named_steps['reg']
feature_names = (
    pipeline.named_steps['pre']
    .get_feature_names_out()
)
coefs = pd.Series(reg.coef_, index=feature_names).sort_values()
print(f"\nIntercepto: ${reg.intercept_:,.2f}")
print("\nCoeficientes (ordenados por magnitud):")
print(coefs.to_string())
```

### Interpretación de coeficientes

```python
# Interpretación: manteniendo todo lo demás constante...
print("Interpretación de coeficientes:")
print(f"  R&D Spend    → por cada $1 extra en I+D, la ganancia aumenta ~${reg.coef_[-3]:.4f}")
print(f"  Marketing    → por cada $1 extra en marketing, la ganancia aumenta ~${reg.coef_[-1]:.4f}")
print(f"  Admin        → impacto en ganancia: ${reg.coef_[-2]:.4f} por dólar")
print("\nObs: R&D Spend tiene el mayor impacto en el profit")
```

---

## 2.4 Selección de Variables — Backward Elimination

Con muchas variables, no todas contribuyen significativamente.
La eliminación hacia atrás usa el p-value para seleccionar las
variables más relevantes.

```python
import statsmodels.api as sm
import pandas as pd
import numpy as np

df = pd.read_csv('data/50_Startups.csv')

# Preprocesar manualmente para usar statsmodels
X = pd.get_dummies(df.drop('Profit', axis=1),
                   columns=['State'], drop_first=True)
X = X.astype(float)
y = df['Profit']

# Agregar intercepto manualmente (statsmodels no lo agrega por defecto)
X_sm = sm.add_constant(X)

def backward_elimination(X, y, significancia=0.05, verbose=True):
    """
    Backward Elimination: elimina variables con p-value > significancia.

    Args:
        X: DataFrame de features con constante
        y: target
        significancia: umbral de p-value (típicamente 0.05)

    Returns:
        X con solo las variables significativas
    """
    num_vars = X.shape[1]
    X_opt = X.copy()
    iteracion = 0

    while True:
        iteracion += 1
        modelo = sm.OLS(y, X_opt).fit()
        p_values = modelo.pvalues

        max_p = p_values.max()
        if max_p > significancia:
            var_eliminar = p_values.idxmax()
            if verbose:
                print(f"Iter {iteracion}: eliminando '{var_eliminar}' (p={max_p:.4f})")
            X_opt = X_opt.drop(columns=[var_eliminar])
        else:
            if verbose:
                print(f"\n✓ Convergencia en iteración {iteracion}")
                print(f"Variables finales: {list(X_opt.columns)}")
            break

    return X_opt, modelo

print("Backward Elimination (significancia = 0.05):")
print("=" * 50)
X_opt, modelo_final = backward_elimination(X_sm, y)

print("\nResumen del modelo final:")
print(modelo_final.summary())
```

### Comparar modelos con todas vs seleccionadas variables

```python
# Modelo con todas las variables
modelo_completo = sm.OLS(y, X_sm).fit()

print("COMPARACIÓN DE MODELOS")
print(f"{'Métrica':<25} {'Completo':>12} {'Seleccionado':>14}")
print("-" * 55)
print(f"{'R²':<25} {modelo_completo.rsquared:>12.4f} {modelo_final.rsquared:>14.4f}")
print(f"{'R² Ajustado':<25} {modelo_completo.rsquared_adj:>12.4f} {modelo_final.rsquared_adj:>14.4f}")
print(f"{'AIC':<25} {modelo_completo.aic:>12.2f} {modelo_final.aic:>14.2f}")
print(f"{'BIC':<25} {modelo_completo.bic:>12.2f} {modelo_final.bic:>14.2f}")
print(f"{'# Variables':<25} {X_sm.shape[1]:>12} {X_opt.shape[1]:>14}")
print("\nObs: R² Ajustado penaliza la complejidad — más fiable que R² puro")
print("     AIC/BIC menores = modelo más parsimonioso")
```

---

## 2.5 Regularización — Ridge, Lasso y ElasticNet

Cuando hay muchas variables o multicolinealidad, la regularización
controla el sobreajuste penalizando los coeficientes grandes.

```python
from sklearn.linear_model import Ridge, Lasso, ElasticNet
from sklearn.model_selection import cross_val_score, GridSearchCV
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
import numpy as np
import pandas as pd

df = pd.read_csv('data/50_Startups.csv')
X = pd.get_dummies(df.drop('Profit', axis=1),
                   columns=['State'], drop_first=True).astype(float)
y = df['Profit']

from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=0
)

# ─── Comparación de modelos de regularización ────────────────────
modelos = {
    'OLS (sin regularización)': LinearRegression(),
    'Ridge (L2)':               Ridge(alpha=1.0),
    'Lasso (L1)':               Lasso(alpha=100.0),
    'ElasticNet (L1+L2)':       ElasticNet(alpha=100.0, l1_ratio=0.5),
}

print(f"{'Modelo':<30} {'R² Train':>10} {'R² Test':>10} {'RMSE Test':>12}")
print("-" * 65)

for nombre, modelo in modelos.items():
    pipe = Pipeline([('scl', StandardScaler()), ('reg', modelo)])
    pipe.fit(X_train, y_train)
    r2_train = pipe.score(X_train, y_train)
    r2_test  = pipe.score(X_test, y_test)
    y_pred   = pipe.predict(X_test)
    rmse     = np.sqrt(mean_squared_error(y_test, y_pred))
    print(f"{nombre:<30} {r2_train:>10.4f} {r2_test:>10.4f} {rmse:>12,.2f}")
```

### Efecto de alpha en Ridge y Lasso

```python
import matplotlib.pyplot as plt

alphas = np.logspace(-2, 5, 100)
coefs_ridge = []
coefs_lasso = []

scaler = StandardScaler()
X_train_sc = scaler.fit_transform(X_train)
X_test_sc  = scaler.transform(X_test)

for alpha in alphas:
    ridge = Ridge(alpha=alpha).fit(X_train_sc, y_train)
    lasso = Lasso(alpha=alpha, max_iter=10000).fit(X_train_sc, y_train)
    coefs_ridge.append(ridge.coef_)
    coefs_lasso.append(lasso.coef_)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

for ax, coefs, titulo in zip(axes,
                               [coefs_ridge, coefs_lasso],
                               ['Ridge (L2) — coeficientes nunca = 0',
                                'Lasso (L1) — coeficientes pueden = 0 (selección)']):
    for i in range(X_train.shape[1]):
        ax.plot(alphas, [c[i] for c in coefs], label=X_train.columns[i])
    ax.set_xscale('log')
    ax.set_xlabel('Alpha (regularización)')
    ax.set_ylabel('Valor del coeficiente')
    ax.set_title(titulo)
    ax.axhline(0, color='black', linestyle='--', linewidth=0.8)

plt.tight_layout()
plt.savefig('outputs/regularizacion.png', dpi=150)
plt.show()

print("Ridge: encoge los coeficientes hacia 0 pero nunca exactamente 0")
print("Lasso: puede hacer coeficientes exactamente 0 → selección de variables automática")
print("ElasticNet: combinación de ambas penalizaciones")
```

### Ajuste de alpha con Cross-Validation

```python
from sklearn.linear_model import RidgeCV, LassoCV

# RidgeCV prueba múltiples alphas con CV interno
ridge_cv = Pipeline([
    ('scl', StandardScaler()),
    ('reg', RidgeCV(alphas=np.logspace(-2, 5, 50), cv=5))
])
ridge_cv.fit(X_train, y_train)
alpha_optimo = ridge_cv.named_steps['reg'].alpha_
print(f"Ridge — Alpha óptimo (CV-5): {alpha_optimo:.4f}")
print(f"R² Test: {ridge_cv.score(X_test, y_test):.4f}")
```

---

## 2.6 Validación Cruzada

La validación cruzada es más robusta que un solo split train/test.

```python
from sklearn.model_selection import KFold, cross_val_score, cross_validate
from sklearn.linear_model import LinearRegression
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
import numpy as np

df = pd.read_csv('data/50_Startups.csv')
X = pd.get_dummies(df.drop('Profit', axis=1),
                   columns=['State'], drop_first=True).astype(float)
y = df['Profit']

pipe = Pipeline([
    ('scl', StandardScaler()),
    ('reg', LinearRegression()),
])

# K-Fold Cross Validation (k=5)
kf = KFold(n_splits=5, shuffle=True, random_state=42)

resultados = cross_validate(
    pipe, X, y,
    cv=kf,
    scoring=['r2', 'neg_mean_squared_error', 'neg_mean_absolute_error'],
    return_train_score=True
)

r2_cv   = resultados['test_r2']
rmse_cv = np.sqrt(-resultados['test_neg_mean_squared_error'])

print(f"Cross-Validation (5 folds):")
print(f"  R²   por fold: {r2_cv.round(4)}")
print(f"  R²   media:    {r2_cv.mean():.4f} ± {r2_cv.std():.4f}")
print(f"  RMSE por fold: {rmse_cv.round(0)}")
print(f"  RMSE media:    {rmse_cv.mean():,.2f} ± {rmse_cv.std():,.2f}")

# Visualizar folds
fig, ax = plt.subplots(figsize=(8, 4))
folds = range(1, 6)
ax.bar(folds, r2_cv, color='steelblue', edgecolor='white')
ax.axhline(r2_cv.mean(), color='red', linestyle='--',
           label=f'Media = {r2_cv.mean():.4f}')
ax.set_xlabel('Fold'); ax.set_ylabel('R²')
ax.set_title('R² por fold — K-Fold CV (k=5)')
ax.legend()
plt.tight_layout()
plt.savefig('outputs/cross_validation.png', dpi=150)
plt.show()
```

---

## ✅ Checkpoint Módulo 2

```python
# checkpoint_m02.py
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression, Ridge, Lasso
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import r2_score, mean_squared_error
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.compose import ColumnTransformer
import warnings
warnings.filterwarnings('ignore')

print("Checkpoint Módulo 2 — Regresión")
print("=" * 45)

# ─── 1. Regresión Simple ─────────────────────────────────────────
df_s = pd.read_csv('data/Salary_Data.csv')
X_s = df_s[['YearsExperience']]
y_s = df_s['Salary']
X_tr, X_te, y_tr, y_te = train_test_split(X_s, y_s,
                                            test_size=0.2, random_state=0)
m_simple = LinearRegression().fit(X_tr, y_tr)
r2_simple = r2_score(y_te, m_simple.predict(X_te))
assert r2_simple > 0.85, f"R² simple muy bajo: {r2_simple:.4f}"
print(f"✓ Regresión Simple   R² = {r2_simple:.4f}")

# ─── 2. Regresión Múltiple ───────────────────────────────────────
df_m = pd.read_csv('data/50_Startups.csv')
X_m = df_m.drop('Profit', axis=1)
y_m = df_m['Profit']

pipe = Pipeline([
    ('pre', ColumnTransformer([
        ('ohe', OneHotEncoder(drop='first'), ['State']),
    ], remainder='passthrough')),
    ('reg', LinearRegression()),
])
X_tr2, X_te2, y_tr2, y_te2 = train_test_split(
    X_m, y_m, test_size=0.2, random_state=0
)
pipe.fit(X_tr2, y_tr2)
r2_multiple = r2_score(y_te2, pipe.predict(X_te2))
assert r2_multiple > 0.90, f"R² múltiple muy bajo: {r2_multiple:.4f}"
print(f"✓ Regresión Múltiple R² = {r2_multiple:.4f}")

# ─── 3. Cross-Validation ─────────────────────────────────────────
X_num = pd.get_dummies(X_m, columns=['State'], drop_first=True).astype(float)
cv_scores = cross_val_score(
    Pipeline([('s', StandardScaler()), ('r', LinearRegression())]),
    X_num, y_m, cv=5, scoring='r2'
)
assert cv_scores.mean() > 0.80, f"CV R² bajo: {cv_scores.mean():.4f}"
print(f"✓ Cross-Validation   R² = {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# ─── 4. Regularización ───────────────────────────────────────────
X_tr_n = StandardScaler().fit_transform(
    pd.get_dummies(X_tr2, columns=['State'], drop_first=True).astype(float)
)
X_te_n = StandardScaler().fit_transform(
    pd.get_dummies(X_te2, columns=['State'], drop_first=True).astype(float)
)
ridge = Ridge(alpha=1.0).fit(X_tr_n, y_tr2)
r2_ridge = r2_score(y_te2, ridge.predict(X_te_n))
print(f"✓ Ridge (L2)         R² = {r2_ridge:.4f}")

print("\n✓ CHECKPOINT M02 COMPLETADO")
```

### Archivos generados

| Archivo | Descripción |
|---|---|
| `data/Salary_Data.csv` | Dataset regresión simple |
| `data/50_Startups.csv` | Dataset regresión múltiple |
| `outputs/regresion_simple.png` | Recta ajustada + residuos |
| `outputs/supuestos_regresion.png` | Diagnóstico de supuestos |
| `outputs/exploracion_50startups.png` | Correlaciones y scatter matrix |
| `outputs/regularizacion.png` | Efecto de alpha en Ridge y Lasso |
| `outputs/cross_validation.png` | R² por fold |

### Checklist

| Técnica | Implementada |
|---|---|
| Regresión simple — solución analítica OLS en NumPy | ✅ |
| Regresión simple — Scikit-learn | ✅ |
| Diagnóstico de supuestos (linealidad, normalidad, homocedasticidad) | ✅ |
| Test de Shapiro-Wilk y Durbin-Watson | ✅ |
| Regresión múltiple con categóricas (One-Hot) | ✅ |
| Dummy variable trap (drop='first') | ✅ |
| Backward Elimination con p-values (statsmodels) | ✅ |
| R², R² ajustado, AIC, BIC | ✅ |
| Ridge (L2), Lasso (L1), ElasticNet | ✅ |
| Efecto de alpha en coeficientes | ✅ |
| RidgeCV y LassoCV para alpha óptimo | ✅ |
| K-Fold Cross Validation | ✅ |

---

## Resumen

| Concepto | Estado |
|---|---|
| Regresión simple — OLS desde cero y con Scikit-learn | ✅ |
| Diagnóstico de supuestos — 4 tests | ✅ |
| Regresión múltiple con variables categóricas | ✅ |
| Selección de variables — Backward Elimination | ✅ |
| Regularización — Ridge, Lasso, ElasticNet | ✅ |
| Validación cruzada K-Fold | ✅ |

**Siguiente módulo →** M03: Regresión Polinomial, SVR, Decision Tree y Random Forest
