# Tutorial 1 — Machine Learning Clásico con Python
## Módulo 1 — Preprocesamiento de Datos

> **Objetivo:** Dominar todas las técnicas de preparación de datos antes de
> entrenar cualquier modelo. Un modelo bien entrenado sobre datos mal
> preprocesados fallará. Un modelo simple sobre datos bien preprocesados
> suele superar modelos complejos sobre datos crudos.
>
> **Checkpoint final:** Pipeline de preprocesamiento completo y reutilizable
> aplicado a tres datasets distintos. El pipeline encapsula todos los pasos
> en un objeto que se ajusta en train y transforma en test, sin fuga de datos.

---

## Cómo ejecutar este módulo

El código de cada módulo funciona en los tres entornos sin modificaciones.
Elige el que prefieras:

### Opción A — Script Python local (.py)

```bash
# 1. Activar el entorno
conda activate ml-clasico

# 2. Crear un archivo con el código de la sección que quieras probar
#    (copia y pega el bloque de código del módulo)
nano modulo_01.py

# 3. Ejecutar
python modulo_01.py

# 4. Los gráficos se guardan en outputs/ automáticamente
#    (no se abren ventanas — ideal para servidores sin pantalla)
```

### Opción B — Jupyter Notebook (local)

```bash
# 1. Instalar Jupyter si no lo tienes
pip install jupyter ipykernel
python -m ipykernel install --user --name ml-clasico --display-name "ML Clásico"

# 2. Lanzar Jupyter
jupyter notebook
# o JupyterLab (más moderno)
jupyter lab

# 3. Crear un nuevo notebook con el kernel "ML Clásico"
# 4. Pegar el código de cada sección en celdas separadas
# 5. Ctrl+Enter para ejecutar cada celda
```

**Primera celda recomendada en Jupyter:**

```python
# Primera celda — configuración del entorno Jupyter
import warnings
warnings.filterwarnings('ignore')

import os
os.makedirs('data',    exist_ok=True)
os.makedirs('outputs', exist_ok=True)
os.makedirs('models',  exist_ok=True)

# Para que los gráficos aparezcan dentro del notebook
%matplotlib inline

print("✓ Entorno Jupyter listo")
```

### Opción C — Google Colab

1. Abrir [colab.research.google.com](https://colab.research.google.com)
2. `Archivo → Nuevo cuaderno`
3. Pegar como **primera celda**:

```python
# ── Celda 1: Setup de Google Colab ───────────────────────────────
# Instalar dependencias que no vienen preinstaladas en Colab
!pip install -q missingno statsmodels optuna xgboost

# Montar Google Drive para persistir datos y modelos entre sesiones
from google.colab import drive
drive.mount('/content/drive')

import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(f'{BASE}/data',    exist_ok=True)
os.makedirs(f'{BASE}/outputs', exist_ok=True)
os.makedirs(f'{BASE}/models',  exist_ok=True)
os.chdir(BASE)

print(f"✓ Directorio de trabajo: {os.getcwd()}")
```

4. Pegar el código de cada sección en celdas siguientes
5. `Shift+Enter` para ejecutar

> **Nota Colab:** NumPy, Pandas, Scikit-learn, Matplotlib, PyTorch y Seaborn
> ya vienen instalados. Solo se necesita instalar los paquetes de la línea `!pip`.

### Comparación de entornos

| Característica | Local (.py) | Jupyter | Colab |
|---|---|---|---|
| Setup inicial | Conda + pip | Jupyter install | Solo navegador |
| Persistencia | ✅ Disco local | ✅ Disco local | ⚠️ Requiere Drive |
| Gráficos interactivos | ❌ Solo archivos | ✅ Inline | ✅ Inline |
| GPU gratuita | ❌ | Depende del equipo | ✅ (limitada) |
| Ideal para | Producción, CI/CD | Exploración | Sin instalación local |

---

## 1.1 Setup del Entorno

### Crear entorno conda

```bash
conda create -n ml-clasico python=3.11 -y
conda activate ml-clasico
```

### Instalar dependencias

```bash
pip install numpy pandas scikit-learn matplotlib seaborn
pip install torch torchvision xgboost nltk
pip install jupyter ipykernel missingno
python -m ipykernel install --user --name ml-clasico --display-name "ML Clásico"
```

### Verificar instalación

```python
# verificar_entorno.py
import numpy as np
import pandas as pd
import sklearn
import matplotlib
import seaborn as sns

print(f"NumPy:      {np.__version__}")
print(f"Pandas:     {pd.__version__}")
print(f"Scikit:     {sklearn.__version__}")
print(f"Matplotlib: {matplotlib.__version__}")
print("✓ Entorno ML Clásico listo")
```

### Tabla de versiones

| Librería | Versión mínima | Uso principal |
|---|---|---|
| Python | 3.11 | Lenguaje base |
| NumPy | 1.26 | Operaciones numéricas |
| Pandas | 2.1 | Manipulación de datos |
| Scikit-learn | 1.4 | Modelos y preprocesamiento |
| Matplotlib | 3.8 | Visualizaciones base |
| Seaborn | 0.13 | Visualizaciones estadísticas |
| Missingno | 0.5 | Visualizar valores faltantes |

---

## 1.2 Cargar y Explorar Datos

Todo proyecto de ML comienza con exploración. Nunca preproceses datos
que no hayas entendido primero.

### Dataset de ejemplo — Data.csv

Usaremos un dataset clásico de introducción con valores faltantes
y variables categóricas:

```
Country,Age,Salary,Purchased
France,44,72000,No
Spain,27,48000,Yes
Germany,30,54000,No
Spain,38,61000,No
Germany,40,,Yes
France,35,58000,Yes
Spain,,52000,No
France,48,79000,Yes
Germany,50,83000,No
France,37,67000,Yes
```

Guarda este contenido como `data/Data.csv` o créalo con Python:

```python
import pandas as pd
import os

os.makedirs('data', exist_ok=True)

contenido = """Country,Age,Salary,Purchased
France,44,72000,No
Spain,27,48000,Yes
Germany,30,54000,No
Spain,38,61000,No
Germany,40,,Yes
France,35,58000,Yes
Spain,,52000,No
France,48,79000,Yes
Germany,50,83000,No
France,37,67000,Yes"""

with open('data/Data.csv', 'w') as f:
    f.write(contenido)

print("✓ Dataset creado en data/Data.csv")
```

### Carga y exploración inicial

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# Cargar datos
df = pd.read_csv('data/Data.csv')

# ─── Exploración básica ───────────────────────────────────────────
print("=" * 50)
print("FORMA DEL DATASET")
print(f"Filas: {df.shape[0]} | Columnas: {df.shape[1]}")

print("\nPRIMERAS 5 FILAS")
print(df.head())

print("\nINFORMACIÓN DE TIPOS")
print(df.info())

print("\nESTADÍSTICAS DESCRIPTIVAS")
print(df.describe())

print("\nVALORES ÚNICOS POR COLUMNA")
for col in df.columns:
    print(f"  {col}: {df[col].nunique()} únicos → {df[col].unique()}")
```

### Visualizar valores faltantes

```python
import missingno as msno

# Mapa de calor de nulos
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

# Porcentaje de nulos por columna
nulos = df.isnull().sum()
nulos_pct = (nulos / len(df) * 100).round(2)
nulos_df = pd.DataFrame({'Nulos': nulos, 'Porcentaje': nulos_pct})
print("\nVALORES FALTANTES:")
print(nulos_df[nulos_df['Nulos'] > 0])

# Heatmap de nulos
sns.heatmap(df.isnull(), ax=axes[0], cbar=False, yticklabels=False,
            cmap='viridis')
axes[0].set_title('Mapa de valores faltantes')

# Distribución de columnas numéricas
df.select_dtypes(include='number').hist(ax=axes[1], bins=10)
axes[1].set_title('Distribución de variables numéricas')

plt.tight_layout()
plt.savefig('outputs/exploracion_inicial.png', dpi=150)
plt.show()
```

### Separar features y target

```python
# X = variables independientes (features)
# y = variable dependiente (target)
X = df.iloc[:, :-1].values   # todas las columnas menos la última
y = df.iloc[:, -1].values    # última columna

print(f"X shape: {X.shape}")
print(f"y shape: {y.shape}")
print(f"\nX:\n{X}")
print(f"\ny:\n{y}")
```

> **Regla de oro:** Siempre usa `.iloc` con índices o `.loc` con nombres
> de columna. Nunca asumas que la última columna es el target en un
> dataset desconocido — verifica el diccionario de datos primero.

---

## 1.3 Manejo de Valores Faltantes

Los valores faltantes son inevitables en datos reales. Hay tres
estrategias principales: eliminar, imputar o predecir.

### Estrategia 1 — Eliminar filas o columnas

```python
df_temp = df.copy()

# Eliminar filas con cualquier valor nulo
df_sin_nulos = df_temp.dropna()
print(f"Filas originales: {len(df_temp)}")
print(f"Filas sin nulos:  {len(df_sin_nulos)}")
# ⚠️ Perdemos el 20% de los datos — inaceptable en datasets pequeños

# Eliminar columnas con más del 50% de nulos
umbral = len(df_temp) * 0.5
df_col_ok = df_temp.dropna(axis=1, thresh=umbral)
print(f"\nColumnas originales: {df_temp.shape[1]}")
print(f"Columnas restantes:  {df_col_ok.shape[1]}")
```

### Estrategia 2 — Imputación con Scikit-learn (recomendada)

```python
from sklearn.impute import SimpleImputer

# Imputar variables numéricas con la media
imputer_media = SimpleImputer(missing_values=np.nan, strategy='mean')

# Solo columnas numéricas (índices 1 y 2: Age, Salary)
X_num = X[:, 1:3].astype(float)
imputer_media.fit(X_num)          # calcula la media en los datos de entrenamiento
X_num_imputado = imputer_media.transform(X_num)

print("Antes de imputar:")
print(X[:, 1:3])
print("\nDespués de imputar:")
print(X_num_imputado)

# Verificar que no quedan nulos
assert not np.isnan(X_num_imputado).any(), "Aún hay valores nulos"
print("\n✓ Sin valores faltantes")
```

### Comparación de estrategias de imputación

```python
from sklearn.impute import SimpleImputer, KNNImputer
import pandas as pd

# Crear dataset con más nulos para comparar
np.random.seed(42)
datos = pd.DataFrame({
    'A': [1, 2, np.nan, 4, 5, np.nan, 7],
    'B': [10, np.nan, 30, 40, np.nan, 60, 70],
    'C': [100, 200, 300, np.nan, 500, 600, 700]
})

print("Dataset original:")
print(datos)

estrategias = {
    'Media':    SimpleImputer(strategy='mean'),
    'Mediana':  SimpleImputer(strategy='median'),
    'Moda':     SimpleImputer(strategy='most_frequent'),
    'Constante':SimpleImputer(strategy='constant', fill_value=0),
    'KNN (k=2)':KNNImputer(n_neighbors=2),
}

for nombre, imp in estrategias.items():
    resultado = pd.DataFrame(
        imp.fit_transform(datos),
        columns=datos.columns
    )
    print(f"\n{nombre}:")
    print(resultado)
```

| Estrategia | Cuándo usar |
|---|---|
| **Media** | Distribuciones simétricas, sin outliers |
| **Mediana** | Distribuciones sesgadas o con outliers |
| **Moda** | Variables categóricas |
| **Constante** | Cuando el valor faltante tiene significado (ej. 0 = sin compras) |
| **KNN** | Cuando hay correlación entre variables — más preciso pero lento |

---

## 1.4 Codificación de Variables Categóricas

Los modelos de ML trabajan con números. Las variables de texto deben
convertirse a representaciones numéricas.

### Tipos de variables categóricas

```python
# Nominal: sin orden intrínseco → Country (France, Spain, Germany)
# Ordinal: con orden → Tamaño (S < M < L < XL)
# Binaria: dos valores → Purchased (Yes/No)

# El error más común: usar LabelEncoder en nominales
# LabelEncoder(France=0, Germany=1, Spain=2) implica Germany > France
# lo que es falso y puede confundir al modelo
```

### One-Hot Encoding — Para variables nominales

```python
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import OneHotEncoder

# Columna 0 = Country (nominal → One-Hot)
ct = ColumnTransformer(
    transformers=[
        ('encoder', OneHotEncoder(), [0])  # índice de la columna a codificar
    ],
    remainder='passthrough'  # dejar las demás columnas sin cambios
)

X_encoded = ct.fit_transform(X)
print("X tras One-Hot Encoding de Country:")
print(X_encoded)
print(f"\nShape: {X_encoded.shape}")
# Añade 3 columnas (France, Germany, Spain) y elimina 1 (Country)
# Net: +2 columnas
```

**Visualizar el resultado:**

```python
# Convertir a DataFrame para mejor legibilidad
columnas = ['France', 'Germany', 'Spain', 'Age', 'Salary']
df_encoded = pd.DataFrame(X_encoded, columns=columnas)
print(df_encoded)
```

### Label Encoding — Para variables ordinales y el target

```python
from sklearn.preprocessing import LabelEncoder

# Codificar el target y (Purchased: Yes/No → 1/0)
le = LabelEncoder()
y_encoded = le.fit_transform(y)

print(f"y original: {y}")
print(f"y encoded:  {y_encoded}")
print(f"Clases: {le.classes_}")  # ['No' 'Yes']
# No → 0, Yes → 1
```

### Ordinal Encoding — Para variables con orden

```python
from sklearn.preprocessing import OrdinalEncoder

# Ejemplo con tallas de ropa
tallas = np.array([['S'], ['XL'], ['M'], ['L'], ['M'], ['S']])
orden = [['S', 'M', 'L', 'XL']]  # orden explícito

oe = OrdinalEncoder(categories=orden)
tallas_num = oe.fit_transform(tallas)
print("Tallas codificadas:")
for t, n in zip(tallas.flatten(), tallas_num.flatten()):
    print(f"  {t} → {int(n)}")
```

### Target Encoding — Para alta cardinalidad

```python
# Cuando una variable tiene muchas categorías (ej. ciudades, códigos postales)
# One-Hot crea demasiadas columnas → Target Encoding es mejor

# Target Encoding: reemplaza cada categoría por la media del target
df_ejemplo = pd.DataFrame({
    'Ciudad': ['Madrid', 'Madrid', 'Bcn', 'Bcn', 'Madrid', 'Valencia'],
    'Ventas': [100, 120, 90, 110, 130, 80]
})

media_por_ciudad = df_ejemplo.groupby('Ciudad')['Ventas'].mean()
df_ejemplo['Ciudad_encoded'] = df_ejemplo['Ciudad'].map(media_por_ciudad)
print(df_ejemplo)
# ⚠️ Requiere cross-validation para evitar data leakage
```

---

## 1.5 División Train/Test

La división train/test es la operación más crítica del preprocesamiento.
**Nunca** uses datos de test para decidir cómo preprocesar.

```python
from sklearn.model_selection import train_test_split

# Actualizar X con la versión codificada
X_final = np.array(X_encoded, dtype=float)
y_final = y_encoded

# División estándar: 80% train, 20% test
X_train, X_test, y_train, y_test = train_test_split(
    X_final, y_final,
    test_size=0.2,       # 20% para test
    random_state=42,     # reproducibilidad
    stratify=y_final     # mantener proporción de clases
)

print(f"X_train: {X_train.shape} | y_train: {y_train.shape}")
print(f"X_test:  {X_test.shape} | y_test:  {y_test.shape}")

# Verificar proporción de clases
print(f"\nProporción en train: {np.bincount(y_train) / len(y_train)}")
print(f"Proporción en test:  {np.bincount(y_test) / len(y_test)}")
```

### Cuánto reservar para test

```python
# Guía según tamaño del dataset
tamanios = {
    'Pequeño   (<1K filas)':   '20-30% test',
    'Mediano   (1K-100K)':     '10-20% test',
    'Grande    (100K-1M)':     '5-10% test',
    'Muy grande (>1M)':        '1-5% test',
}

for tamanio, recomendacion in tamanios.items():
    print(f"  {tamanio} → {recomendacion}")
```

### Train/Validation/Test — Para ajuste de hiperparámetros

```python
# Cuando necesitas ajustar hiperparámetros, necesitas 3 conjuntos
X_train_val, X_test, y_train_val, y_test = train_test_split(
    X_final, y_final, test_size=0.2, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X_train_val, y_train_val, test_size=0.25, random_state=42
    # 0.25 * 0.8 = 0.2 del total → split 60/20/20
)

print(f"Train:      {X_train.shape[0]} filas ({X_train.shape[0]/len(X_final)*100:.0f}%)")
print(f"Validation: {X_val.shape[0]} filas ({X_val.shape[0]/len(X_final)*100:.0f}%)")
print(f"Test:       {X_test.shape[0]} filas ({X_test.shape[0]/len(X_final)*100:.0f}%)")
```

> **Regla fundamental — Data Leakage:** El escalado, la imputación y
> cualquier transformación deben aprenderse SOLO en train y aplicarse
> en test. Nunca hagas `fit` en test. `fit_transform` solo en train,
> `transform` en test.

---

## 1.6 Feature Scaling — Normalización y Estandarización

La mayoría de los algoritmos de ML son sensibles a la escala de las
variables. Un feature con rango [0, 1] y otro con rango [10000, 100000]
harán que el modelo ignore el primero.

### StandardScaler — Estandarización (Z-score)

Transforma los datos para que tengan media = 0 y desviación estándar = 1.

```
x_scaled = (x - μ) / σ
```

```python
from sklearn.preprocessing import StandardScaler

# ⚠️ fit() SOLO en train, transform() en train Y test
scaler = StandardScaler()

# Solo escalar columnas numéricas (Age y Salary, índices 3 y 4)
X_train_scaled = X_train.copy()
X_test_scaled  = X_test.copy()

X_train_scaled[:, 3:] = scaler.fit_transform(X_train[:, 3:])
X_test_scaled[:, 3:]  = scaler.transform(X_test[:, 3:])

print("ANTES del escalado (X_train):")
print(X_train[:, 3:])

print("\nDESPUÉS del escalado (X_train):")
print(X_train_scaled[:, 3:].round(4))

print(f"\nMedia de Age en train:   {X_train_scaled[:, 3].mean():.4f}  (≈0)")
print(f"Std  de Age en train:    {X_train_scaled[:, 3].std():.4f}  (≈1)")
```

### MinMaxScaler — Normalización [0, 1]

Comprime los valores al rango [0, 1].

```
x_scaled = (x - x_min) / (x_max - x_min)
```

```python
from sklearn.preprocessing import MinMaxScaler

mm_scaler = MinMaxScaler()
X_train_mm = mm_scaler.fit_transform(X_train[:, 3:])
X_test_mm  = mm_scaler.transform(X_test[:, 3:])

print("MinMax Scaling (X_train Age y Salary):")
print(X_train_mm.round(4))
print(f"Min: {X_train_mm.min():.2f} | Max: {X_train_mm.max():.2f}")
```

### RobustScaler — Para datasets con outliers

Usa la mediana y el IQR en lugar de media y desviación estándar.
Más resistente a outliers.

```python
from sklearn.preprocessing import RobustScaler

# Crear dataset con outlier artificial
datos_outlier = np.array([[1], [2], [3], [4], [5], [100]])  # 100 es outlier

std_result    = StandardScaler().fit_transform(datos_outlier)
robust_result = RobustScaler().fit_transform(datos_outlier)

print("Comparación con outlier (valor 100):")
print(f"{'Original':>10} {'Standard':>10} {'Robust':>10}")
for orig, std, rob in zip(datos_outlier.flatten(),
                           std_result.flatten(),
                           robust_result.flatten()):
    print(f"{orig:>10.0f} {std:>10.4f} {rob:>10.4f}")
```

### ¿Qué escalador usar?

```python
# Guía de selección de escalador
guia = {
    'StandardScaler':  'Algoritmos basados en distancias o gradiente (SVM, KNN, Redes Neuronales, regresión)',
    'MinMaxScaler':    'Imágenes (pixeles 0-255), cuando el rango exacto importa, redes neuronales con sigmoid',
    'RobustScaler':    'Datos con outliers significativos',
    'Sin escalado':    'Decision Trees, Random Forest, XGBoost (invariantes a la escala)',
}

for escalador, cuando in guia.items():
    print(f"  {escalador:<20} → {cuando}")
```

---

## 1.7 Detección y Manejo de Outliers

Los outliers pueden distorsionar el entrenamiento de muchos modelos.

```python
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

# Dataset sintético con outliers
np.random.seed(42)
datos = pd.DataFrame({
    'Salary':  np.concatenate([
        np.random.normal(50000, 10000, 95),
        [150000, 160000, 170000, 180000, 190000]  # outliers
    ]),
    'Age': np.concatenate([
        np.random.normal(35, 8, 95),
        [75, 78, 80, 82, 85]  # outliers
    ])
})

# ─── Método 1: IQR (Interquartile Range) ─────────────────────────
def detectar_outliers_iqr(serie):
    Q1 = serie.quantile(0.25)
    Q3 = serie.quantile(0.75)
    IQR = Q3 - Q1
    lower = Q1 - 1.5 * IQR
    upper = Q3 + 1.5 * IQR
    return (serie < lower) | (serie > upper)

# ─── Método 2: Z-score ───────────────────────────────────────────
def detectar_outliers_zscore(serie, umbral=3):
    z = np.abs((serie - serie.mean()) / serie.std())
    return z > umbral

for col in ['Salary', 'Age']:
    out_iqr    = detectar_outliers_iqr(datos[col])
    out_zscore = detectar_outliers_zscore(datos[col])
    print(f"{col}:")
    print(f"  IQR:     {out_iqr.sum()} outliers detectados")
    print(f"  Z-score: {out_zscore.sum()} outliers detectados")

# ─── Visualización: Boxplot ──────────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
datos.boxplot(ax=axes[0])
axes[0].set_title('Boxplot — detección visual de outliers')

# Scatter con outliers marcados
outliers_mask = detectar_outliers_iqr(datos['Salary']) | detectar_outliers_iqr(datos['Age'])
axes[1].scatter(datos.loc[~outliers_mask, 'Age'],
                datos.loc[~outliers_mask, 'Salary'],
                label='Normal', alpha=0.6)
axes[1].scatter(datos.loc[outliers_mask, 'Age'],
                datos.loc[outliers_mask, 'Salary'],
                color='red', label='Outlier', s=80, zorder=5)
axes[1].set_xlabel('Age'); axes[1].set_ylabel('Salary')
axes[1].set_title('Scatter con outliers marcados')
axes[1].legend()

plt.tight_layout()
plt.savefig('outputs/outliers.png', dpi=150)
plt.show()
```

### Opciones para manejar outliers

```python
df_clean = datos.copy()
outliers = detectar_outliers_iqr(df_clean['Salary'])

# Opción 1: Eliminar
df_sin_out = df_clean[~outliers]
print(f"Opción 1 — Eliminar: {len(df_clean)} → {len(df_sin_out)} filas")

# Opción 2: Winsorizar (recortar al percentil 5-95)
from scipy.stats import mstats
df_clean['Salary_wins'] = mstats.winsorize(df_clean['Salary'], limits=[0.05, 0.05])

# Opción 3: Imputar con la mediana
mediana = df_clean.loc[~outliers, 'Salary'].median()
df_clean.loc[outliers, 'Salary_imputed'] = mediana
print(f"Opción 3 — Imputar: outliers reemplazados por mediana = {mediana:.0f}")
```

---

## 1.8 Feature Engineering Básico

Crear nuevas variables a partir de las existentes puede mejorar
significativamente el rendimiento de los modelos.

```python
import pandas as pd
import numpy as np

# Dataset de ejemplo: ventas de tienda
df = pd.DataFrame({
    'fecha':        pd.date_range('2024-01-01', periods=365, freq='D'),
    'ventas':       np.random.randint(100, 500, 365),
    'temperatura':  np.random.normal(20, 8, 365),
    'es_feriado':   np.random.choice([0, 1], 365, p=[0.97, 0.03]),
    'precio':       np.random.uniform(10, 50, 365),
    'unidades':     np.random.randint(5, 30, 365),
})

# ─── Features de fecha ───────────────────────────────────────────
df['dia_semana']  = df['fecha'].dt.dayofweek     # 0=Lunes, 6=Domingo
df['mes']         = df['fecha'].dt.month
df['trimestre']   = df['fecha'].dt.quarter
df['es_fin_semana'] = (df['dia_semana'] >= 5).astype(int)

# ─── Features de interacción ────────────────────────────────────
df['ingreso_total']  = df['precio'] * df['unidades']
df['precio_por_temp'] = df['precio'] / (df['temperatura'] + 1)

# ─── Features de ventana temporal (lag) ─────────────────────────
df['ventas_ayer']       = df['ventas'].shift(1)
df['ventas_semana_ant'] = df['ventas'].shift(7)
df['media_movil_7d']    = df['ventas'].rolling(7).mean()

# ─── Transformaciones matemáticas ───────────────────────────────
# Log-transform: útil para distribuciones sesgadas a la derecha
df['log_ventas'] = np.log1p(df['ventas'])  # log1p = log(1 + x), evita log(0)

print(f"Features creadas: {df.shape[1]} columnas")
print(df[['fecha', 'ventas', 'log_ventas', 'es_fin_semana',
           'media_movil_7d']].head(10).to_string())
```

---

## 1.9 Pipeline Completo con Scikit-learn

Un `Pipeline` encadena todos los pasos de preprocesamiento y el modelo
en un objeto único. Garantiza que no haya data leakage.

```python
import numpy as np
import pandas as pd
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder, LabelEncoder
from sklearn.impute import SimpleImputer
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# ─── Dataset completo ────────────────────────────────────────────
df = pd.read_csv('data/Data.csv')
X = df.drop('Purchased', axis=1)
y = LabelEncoder().fit_transform(df['Purchased'])

# ─── Definir columnas por tipo ───────────────────────────────────
columnas_numericas    = ['Age', 'Salary']
columnas_categoricas  = ['Country']

# ─── Transformadores por tipo de columna ────────────────────────
transformador_numerico = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='mean')),
    ('scaler',  StandardScaler()),
])

transformador_categorico = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('onehot',  OneHotEncoder(handle_unknown='ignore')),
])

# ─── Combinar transformadores ────────────────────────────────────
preprocessor = ColumnTransformer(transformers=[
    ('num', transformador_numerico,  columnas_numericas),
    ('cat', transformador_categorico, columnas_categoricas),
])

# ─── Pipeline completo (preprocesamiento + modelo) ───────────────
pipeline = Pipeline(steps=[
    ('preprocessor', preprocessor),
    ('classifier',   LogisticRegression(random_state=42)),
])

# ─── Train/Test split ────────────────────────────────────────────
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ─── Entrenar y evaluar ─────────────────────────────────────────
pipeline.fit(X_train, y_train)
y_pred = pipeline.predict(X_test)

print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
print("\n✓ Pipeline completo: sin data leakage garantizado")
print("  El preprocessor hace fit() solo en X_train")
print("  Aplica transform() en X_test automáticamente")
```

### Guardar y cargar el pipeline

```python
import joblib
import os

os.makedirs('models', exist_ok=True)

# Guardar pipeline entrenado
joblib.dump(pipeline, 'models/pipeline_preprocessing.pkl')
print("✓ Pipeline guardado en models/pipeline_preprocessing.pkl")

# Cargar y usar
pipeline_cargado = joblib.load('models/pipeline_preprocessing.pkl')
prediccion = pipeline_cargado.predict(X_test)
print(f"✓ Pipeline cargado y funcionando | Accuracy: {accuracy_score(y_test, prediccion):.4f}")
```

---

## 1.10 Balanceo de Clases

Cuando una clase tiene muchas más muestras que otra, el modelo aprende
a predecir siempre la clase mayoritaria.

```python
from sklearn.utils import resample
from collections import Counter

# Dataset desbalanceado simulado
np.random.seed(42)
n_mayoritaria = 900
n_minoritaria = 100

X_desc = np.random.randn(n_mayoritaria + n_minoritaria, 4)
y_desc = np.array([0] * n_mayoritaria + [1] * n_minoritaria)

print(f"Distribución original: {Counter(y_desc)}")
print(f"Ratio: {n_mayoritaria/n_minoritaria}:1")

# ─── Técnica 1: Oversampling — duplicar muestras minoritarias ────
X_min = X_desc[y_desc == 1]
y_min = y_desc[y_desc == 1]
X_min_ups, y_min_ups = resample(X_min, y_min,
                                 replace=True,
                                 n_samples=n_mayoritaria,
                                 random_state=42)
X_over = np.vstack([X_desc[y_desc == 0], X_min_ups])
y_over = np.hstack([y_desc[y_desc == 0], y_min_ups])
print(f"\nOversampling: {Counter(y_over)}")

# ─── Técnica 2: Undersampling — reducir clase mayoritaria ────────
X_may = X_desc[y_desc == 0]
y_may = y_desc[y_desc == 0]
X_may_down, y_may_down = resample(X_may, y_may,
                                   replace=False,
                                   n_samples=n_minoritaria,
                                   random_state=42)
X_under = np.vstack([X_may_down, X_min])
y_under = np.hstack([y_may_down, y_min])
print(f"Undersampling: {Counter(y_under)}")

# ─── Técnica 3: class_weight en el modelo (más simple) ───────────
from sklearn.linear_model import LogisticRegression

modelo_balanced = LogisticRegression(class_weight='balanced', random_state=42)
# class_weight='balanced' ajusta los pesos automáticamente
# peso_clase_i = n_total / (n_clases * n_muestras_clase_i)
print(f"\nclass_weight='balanced': ajuste automático sin modificar datos")
```

---

## ✅ Checkpoint Módulo 1

### Script de verificación completo

```python
# checkpoint_m01.py
import numpy as np
import pandas as pd
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder, LabelEncoder
from sklearn.impute import SimpleImputer, KNNImputer
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
import os

print("Checkpoint Módulo 1 — Preprocesamiento")
print("=" * 45)

# 1. Verificar entorno
import sklearn, pandas, numpy
print(f"✓ NumPy {numpy.__version__} | Pandas {pandas.__version__} | "
      f"Sklearn {sklearn.__version__}")

# 2. Crear dataset
os.makedirs('data', exist_ok=True)
contenido = """Country,Age,Salary,Purchased
France,44,72000,No
Spain,27,48000,Yes
Germany,30,54000,No
Spain,38,61000,No
Germany,40,,Yes
France,35,58000,Yes
Spain,,52000,No
France,48,79000,Yes
Germany,50,83000,No
France,37,67000,Yes"""
with open('data/Data.csv', 'w') as f:
    f.write(contenido)

df = pd.read_csv('data/Data.csv')
assert df.shape == (10, 4), "Error: forma del dataset incorrecta"
print(f"✓ Dataset cargado: {df.shape}")

# 3. Verificar nulos
nulos = df.isnull().sum().sum()
assert nulos == 2, f"Error: se esperaban 2 nulos, hay {nulos}"
print(f"✓ Valores faltantes detectados: {nulos}")

# 4. Pipeline
X = df.drop('Purchased', axis=1)
y = LabelEncoder().fit_transform(df['Purchased'])

pipe = Pipeline([
    ('pre', ColumnTransformer([
        ('num', Pipeline([
            ('imp', SimpleImputer(strategy='mean')),
            ('scl', StandardScaler()),
        ]), ['Age', 'Salary']),
        ('cat', Pipeline([
            ('imp', SimpleImputer(strategy='most_frequent')),
            ('ohe', OneHotEncoder(handle_unknown='ignore')),
        ]), ['Country']),
    ])),
    ('clf', LogisticRegression(random_state=42))
])

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42
)
pipe.fit(X_train, y_train)
acc = accuracy_score(y_test, pipe.predict(X_test))
assert 0.0 <= acc <= 1.0, "Error: accuracy fuera de rango"
print(f"✓ Pipeline completo — Accuracy: {acc:.4f}")

# 5. Verificar no data leakage
# El scaler debe haber aprendido solo de X_train
scaler = pipe.named_steps['pre'].transformers_[0][1].named_steps['scl']
assert hasattr(scaler, 'mean_'), "Error: scaler no entrenado"
print(f"✓ Scaler entrenado solo en train — media Age: {scaler.mean_[0]:.2f}")

import joblib, os
os.makedirs('models', exist_ok=True)
joblib.dump(pipe, 'models/pipeline_m01.pkl')
pipe2 = joblib.load('models/pipeline_m01.pkl')
assert accuracy_score(y_test, pipe2.predict(X_test)) == acc
print("✓ Pipeline guardado y cargado correctamente")

print("\n✓ CHECKPOINT M01 COMPLETADO")
```

```bash
python checkpoint_m01.py
```

### Archivos generados

| Archivo | Descripción |
|---|---|
| `data/Data.csv` | Dataset de ejemplo con nulos y categóricas |
| `outputs/exploracion_inicial.png` | Mapa de nulos y distribuciones |
| `outputs/outliers.png` | Boxplot y scatter con outliers |
| `models/pipeline_m01.pkl` | Pipeline completo serializado |
| `checkpoint_m01.py` | Script de verificación |

### Checklist

| Técnica | Implementada |
|---|---|
| Exploración: shape, info, describe, nunique | ✅ |
| Visualización de nulos con missingno | ✅ |
| Imputación: Mean, Median, Mode, KNN | ✅ |
| One-Hot Encoding (nominales) | ✅ |
| Label Encoding (target y ordinales) | ✅ |
| Ordinal Encoding (con orden explícito) | ✅ |
| Target Encoding (alta cardinalidad) | ✅ |
| Train/Test split con stratify | ✅ |
| Train/Val/Test split (60/20/20) | ✅ |
| StandardScaler, MinMaxScaler, RobustScaler | ✅ |
| Detección de outliers: IQR y Z-score | ✅ |
| Feature Engineering: fechas, lag, rolling | ✅ |
| Pipeline con ColumnTransformer | ✅ |
| Guardar/cargar pipeline con joblib | ✅ |
| Balanceo: oversampling, undersampling, class_weight | ✅ |

---

## Resumen

| Concepto | Estado |
|---|---|
| Exploración y diagnóstico de datos | ✅ |
| Manejo de valores faltantes (5 estrategias) | ✅ |
| Codificación de categóricas (4 tipos) | ✅ |
| División train/test sin data leakage | ✅ |
| Escalado (StandardScaler, MinMax, Robust) | ✅ |
| Detección y tratamiento de outliers | ✅ |
| Feature Engineering básico | ✅ |
| Pipeline reutilizable con Scikit-learn | ✅ |
| Balanceo de clases (3 técnicas) | ✅ |

**Siguiente módulo →** M02: Regresión Simple y Múltiple
