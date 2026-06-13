# Módulo 16 — Proyecto Integrador: Pipeline ML de Extremo a Extremo

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: Todos los módulos anteriores (M01–M15)

---

## Tabla de contenidos

1. [Descripción del proyecto](#descripción-del-proyecto)
2. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
3. [Fase 1: EDA y comprensión del negocio](#fase-1-eda)
4. [Fase 2: Preprocesamiento y Feature Engineering](#fase-2-preprocesamiento)
5. [Fase 3: Modelado](#fase-3-modelado)
6. [Fase 4: Evaluación y selección final](#fase-4-evaluación)
7. [Fase 5: Explicabilidad](#fase-5-explicabilidad)
8. [Fase 6: Despliegue (inferencia con joblib)](#fase-6-despliegue)
9. [Checklist del proyecto](#checklist)
10. [Checkpoint M16 — Evaluación final](#checkpoint-m16)

---

## Descripción del proyecto

**Problema**: Predecir si un cliente va a **abandonar (churn)** un servicio de telecomunicaciones.

```
Dataset: Telco Customer Churn (simulado — similar al de Kaggle)
  Muestras: 7,043 clientes
  Features: 20 variables (numéricas + categóricas + binarias)
  Target:   Churn (0=No abandona, 1=Abandona)  — desbalanceado ~26% positivos

Objetivo de negocio:
  Identificar clientes en riesgo de abandono ANTES de que ocurra
  para que el equipo de retención pueda actuar proactivamente.

Métricas prioritarias:
  AUC-ROC:  ranking de riesgo para priorizar contactos
  Recall:   capturar el mayor % de churners posibles
  F1:       balance entre precision y recall
  Precision: evitar molestar a clientes que no van a irse

El proyecto aplica TODOS los conceptos del tutorial:
  M01: Preprocesamiento, encoding, pipeline
  M02-M03: Modelos de baseline
  M05-M07: Clasificadores
  M08: Evaluación completa
  M09: (opcional) Segmentación de clientes en riesgo
  M11: Reducción de dimensionalidad para visualización
  M15: Selección de modelos, XGBoost, Optuna, SHAP
```

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install shap optuna lightgbm  # si no instalados
python modulo-16-proyecto-integrador.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
!pip install -q shap optuna lightgbm
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Fase 1: EDA

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import seaborn as sns
import warnings, os
warnings.filterwarnings('ignore')
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)

# ── Generación del dataset de telecomunicaciones ──────────────────────────────
np.random.seed(42)
n = 7043

def generar_telco(n=7043, seed=42):
    """Genera un dataset sintético de churn de telecomunicaciones."""
    np.random.seed(seed)

    df = pd.DataFrame({
        'customerID':       [f'C{i:05d}' for i in range(n)],
        'gender':           np.random.choice(['Male', 'Female'], n),
        'SeniorCitizen':    np.random.choice([0, 1], n, p=[0.84, 0.16]),
        'Partner':          np.random.choice(['Yes', 'No'], n, p=[0.48, 0.52]),
        'Dependents':       np.random.choice(['Yes', 'No'], n, p=[0.30, 0.70]),
        'tenure':           np.random.randint(1, 73, n).astype(float),
        'PhoneService':     np.random.choice(['Yes', 'No'], n, p=[0.90, 0.10]),
        'MultipleLines':    np.random.choice(['Yes', 'No', 'No phone service'], n, p=[0.42, 0.48, 0.10]),
        'InternetService':  np.random.choice(['DSL', 'Fiber optic', 'No'], n, p=[0.34, 0.44, 0.22]),
        'OnlineSecurity':   np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.28, 0.50, 0.22]),
        'OnlineBackup':     np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.34, 0.44, 0.22]),
        'DeviceProtection': np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.34, 0.44, 0.22]),
        'TechSupport':      np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.29, 0.49, 0.22]),
        'StreamingTV':      np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.38, 0.40, 0.22]),
        'StreamingMovies':  np.random.choice(['Yes', 'No', 'No internet service'], n, p=[0.39, 0.39, 0.22]),
        'Contract':         np.random.choice(['Month-to-month', 'One year', 'Two year'], n, p=[0.55, 0.21, 0.24]),
        'PaperlessBilling': np.random.choice(['Yes', 'No'], n, p=[0.59, 0.41]),
        'PaymentMethod':    np.random.choice(['Electronic check', 'Mailed check',
                                               'Bank transfer (automatic)',
                                               'Credit card (automatic)'], n,
                                              p=[0.34, 0.23, 0.22, 0.21]),
        'MonthlyCharges':   np.random.uniform(18.25, 118.75, n).round(2),
        'TotalCharges':     np.nan,  # se calculará
    })

    # TotalCharges = tenure × MonthlyCharges + ruido
    df['TotalCharges'] = (df['tenure'] * df['MonthlyCharges'] +
                          np.random.normal(0, 50, n)).round(2)
    # Algunos NaN en TotalCharges (11 en el dataset original)
    df.loc[df['tenure'] == 0, 'TotalCharges'] = np.nan

    # Construir churn con reglas de negocio realistas
    churn_prob = 0.10  # base
    churn_prob = np.where(df['Contract'] == 'Month-to-month',  churn_prob + 0.25, churn_prob)
    churn_prob = np.where(df['InternetService'] == 'Fiber optic', churn_prob + 0.15, churn_prob)
    churn_prob = np.where(df['tenure'] < 12,                    churn_prob + 0.20, churn_prob)
    churn_prob = np.where(df['tenure'] > 48,                    churn_prob - 0.15, churn_prob)
    churn_prob = np.where(df['OnlineSecurity'] == 'No',         churn_prob + 0.10, churn_prob)
    churn_prob = np.where(df['TechSupport'] == 'No',            churn_prob + 0.08, churn_prob)
    churn_prob = np.where(df['PaperlessBilling'] == 'Yes',      churn_prob + 0.05, churn_prob)
    churn_prob = np.where(df['SeniorCitizen'] == 1,             churn_prob + 0.05, churn_prob)
    churn_prob = np.clip(churn_prob, 0, 0.85)

    df['Churn'] = (np.random.random(n) < churn_prob).astype(int)
    return df

df = generar_telco(n)
df.to_csv('data/telco_churn.csv', index=False)
print(f"Dataset guardado: {df.shape}")
print(f"Churn rate: {df['Churn'].mean():.2%}")
print(f"\nPrimeras filas:")
print(df.head(3).to_string())

# ── EDA ───────────────────────────────────────────────────────────────────────
print(f"\n─── Información del dataset ───")
print(df.dtypes.value_counts())
print(f"\nValores nulos:\n{df.isnull().sum()[df.isnull().sum()>0]}")

# Distribución del target
fig, axes = plt.subplots(2, 3, figsize=(15, 9))

# Churn por contrato
churn_by_contract = df.groupby('Contract')['Churn'].mean()
axes[0,0].bar(churn_by_contract.index, churn_by_contract.values, color=['steelblue','orange','green'])
axes[0,0].set_title('Churn rate por tipo de contrato')
axes[0,0].set_ylabel('Tasa de abandono')

# Distribución de tenure
for churn_val, color, label in [(0,'blue','No Churn'), (1,'red','Churn')]:
    axes[0,1].hist(df[df['Churn']==churn_val]['tenure'], bins=30, alpha=0.6,
                   color=color, label=label, density=True)
axes[0,1].set_title('Distribución de tenure (meses)')
axes[0,1].legend()

# Monthly Charges vs Churn
axes[0,2].boxplot([df[df['Churn']==0]['MonthlyCharges'],
                    df[df['Churn']==1]['MonthlyCharges']],
                   labels=['No Churn', 'Churn'])
axes[0,2].set_title('MonthlyCharges por grupo')

# Churn por servicio de internet
churn_by_internet = df.groupby('InternetService')['Churn'].mean()
axes[1,0].bar(churn_by_internet.index, churn_by_internet.values,
              color=['coral','skyblue','lightgreen'])
axes[1,0].set_title('Churn rate por InternetService')

# Correlación de variables numéricas
num_cols = ['tenure', 'MonthlyCharges', 'TotalCharges', 'SeniorCitizen', 'Churn']
corr = df[num_cols].corr()
im = axes[1,1].imshow(corr, cmap='coolwarm', vmin=-1, vmax=1)
axes[1,1].set_xticks(range(len(num_cols)))
axes[1,1].set_yticks(range(len(num_cols)))
axes[1,1].set_xticklabels(num_cols, rotation=45, ha='right', fontsize=8)
axes[1,1].set_yticklabels(num_cols, fontsize=8)
plt.colorbar(im, ax=axes[1,1])
axes[1,1].set_title('Correlación variables numéricas')

# Distribución de clases (desbalance)
counts = df['Churn'].value_counts()
axes[1,2].pie(counts, labels=['No Churn', 'Churn'], autopct='%1.1f%%',
              colors=['steelblue','tomato'], startangle=90)
axes[1,2].set_title(f'Distribución de clases (Churn: {df["Churn"].mean():.1%})')

plt.suptitle('EDA — Telco Customer Churn', fontsize=14, fontweight='bold')
plt.tight_layout()
plt.savefig('outputs/m16_eda.png', dpi=120)
plt.show()

print("\n─── Insights del EDA ───")
print(f"  Contrato Month-to-month: {churn_by_contract['Month-to-month']:.1%} churn rate")
print(f"  Contrato Two year:       {churn_by_contract['Two year']:.1%} churn rate")
print(f"  Fiber Optic:             {churn_by_internet['Fiber optic']:.1%} churn rate")
print(f"  Tenure < 12 meses:       {df[df['tenure']<12]['Churn'].mean():.1%} churn rate")
print(f"  Tenure > 48 meses:       {df[df['tenure']>48]['Churn'].mean():.1%} churn rate")
```

---

## Fase 2: Preprocesamiento

```python
from sklearn.model_selection import train_test_split, StratifiedKFold
from sklearn.preprocessing import StandardScaler, OrdinalEncoder, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.base import BaseEstimator, TransformerMixin

# ── Definición de columnas por tipo ──────────────────────────────────────────
ID_COL       = 'customerID'
TARGET_COL   = 'Churn'

NUM_COLS     = ['tenure', 'MonthlyCharges', 'TotalCharges']
BIN_COLS     = ['SeniorCitizen']
CAT_NOM_COLS = ['gender', 'Partner', 'Dependents', 'PhoneService',
                 'MultipleLines', 'InternetService', 'OnlineSecurity',
                 'OnlineBackup', 'DeviceProtection', 'TechSupport',
                 'StreamingTV', 'StreamingMovies', 'PaperlessBilling',
                 'PaymentMethod']
CAT_ORD_COLS = ['Contract']
CONTRACT_ORD = [['Month-to-month', 'One year', 'Two year']]

# ── Feature Engineering ───────────────────────────────────────────────────────
class TelcoFeatureEngineer(BaseEstimator, TransformerMixin):
    """
    Transformador custom que agrega features derivadas de negocio.
    Compatible con sklearn Pipeline.
    """
    def fit(self, X, y=None):
        return self

    def transform(self, X):
        df = X.copy()

        # 1. Segmento de tenure
        df['tenure_segment'] = pd.cut(df['tenure'], bins=[0, 12, 24, 48, 72],
                                       labels=['nuevo', 'medio', 'maduro', 'leal'],
                                       include_lowest=True)

        # 2. Servicios contratados (cuenta)
        servicios = ['PhoneService', 'InternetService', 'OnlineSecurity',
                     'OnlineBackup', 'DeviceProtection', 'TechSupport',
                     'StreamingTV', 'StreamingMovies']
        df['n_servicios'] = sum(
            (df[col].isin(['Yes', 'DSL', 'Fiber optic'])).astype(int)
            for col in servicios
        )

        # 3. Costo por mes normalizado por tenure
        df['avg_monthly_cost'] = df['MonthlyCharges']  # directo
        df['cost_per_tenure']  = df['MonthlyCharges'] / (df['tenure'] + 1)

        # 4. ¿Tiene internet de alta velocidad?
        df['is_fiber'] = (df['InternetService'] == 'Fiber optic').astype(int)

        # 5. ¿Sin servicios de soporte?
        df['sin_soporte'] = (
            (df['OnlineSecurity'].isin(['No', 'No internet service'])) &
            (df['TechSupport'].isin(['No', 'No internet service']))
        ).astype(int)

        return df

    def get_feature_names_out(self, input_features=None):
        return input_features  # pass-through

# Preparar datos
X_raw = df.drop(columns=[ID_COL, TARGET_COL])
y     = df[TARGET_COL].values

X_eng = TelcoFeatureEngineer().fit_transform(X_raw)

# Actualizar listas de columnas después del FE
NEW_NUM_COLS   = NUM_COLS + ['n_servicios', 'avg_monthly_cost', 'cost_per_tenure']
NEW_BIN_COLS   = BIN_COLS + ['is_fiber', 'sin_soporte']
NEW_CAT_NOM    = CAT_NOM_COLS
NEW_CAT_ORD    = CAT_ORD_COLS + ['tenure_segment']
TENURE_SEG_ORD = [['nuevo', 'medio', 'maduro', 'leal']]
CONTRACT_ORD2  = CONTRACT_ORD

print(f"Dataset después de FE: {X_eng.shape}")
print(f"Nuevas features: n_servicios, cost_per_tenure, is_fiber, sin_soporte, tenure_segment")

# ── Preprocesador ─────────────────────────────────────────────────────────────
preprocesador = ColumnTransformer(transformers=[
    ('num', Pipeline([
        ('imputer', SimpleImputer(strategy='median')),
        ('scaler',  StandardScaler()),
    ]), NEW_NUM_COLS),

    ('bin', 'passthrough', NEW_BIN_COLS),

    ('nom', Pipeline([
        ('imputer', SimpleImputer(strategy='most_frequent')),
        ('ohe',     OneHotEncoder(handle_unknown='ignore', sparse_output=False)),
    ]), NEW_CAT_NOM),

    ('ord', Pipeline([
        ('imputer', SimpleImputer(strategy='most_frequent')),
        ('oe',      OrdinalEncoder(
            categories=CONTRACT_ORD2 + TENURE_SEG_ORD,
            handle_unknown='use_encoded_value', unknown_value=-1
        )),
    ]), NEW_CAT_ORD),

], remainder='drop')

# Split
X_tr, X_te, y_tr, y_te = train_test_split(
    X_eng, y, test_size=0.15, stratify=y, random_state=42
)
X_tr, X_val, y_tr, y_val = train_test_split(
    X_tr, y_tr, test_size=0.12, stratify=y_tr, random_state=42
)

print(f"\nSplit: Train={len(y_tr)} | Val={len(y_val)} | Test={len(y_te)}")
print(f"Churn en test: {y_te.mean():.2%}")
```

---

## Fase 3: Modelado

```python
from sklearn.metrics import (roc_auc_score, f1_score, recall_score,
                              accuracy_score, classification_report,
                              ConfusionMatrixDisplay)
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
import xgboost as xgb
import optuna, time
optuna.logging.set_verbosity(optuna.logging.WARNING)

cv_strat = StratifiedKFold(5, shuffle=True, random_state=42)

# ── BASELINE: Logistic Regression ────────────────────────────────────────────
pipe_lr = Pipeline([
    ('prep', preprocesador),
    ('clf', LogisticRegression(C=1, max_iter=2000, class_weight='balanced',
                                random_state=42))
])

scores_lr = cross_validate(pipe_lr, X_tr, y_tr, cv=cv_strat,
                             scoring=['roc_auc', 'f1', 'recall'],
                             return_train_score=False, n_jobs=-1)

print(f"\n─── Baseline: Logistic Regression ───")
print(f"  AUC={scores_lr['test_roc_auc'].mean():.4f} | "
      f"F1={scores_lr['test_f1'].mean():.4f} | "
      f"Recall={scores_lr['test_recall'].mean():.4f}")

# ── Random Forest con balanceo ────────────────────────────────────────────────
pipe_rf = Pipeline([
    ('prep', preprocesador),
    ('clf', RandomForestClassifier(n_estimators=100, class_weight='balanced',
                                    random_state=42, n_jobs=-1))
])

scores_rf = cross_validate(pipe_rf, X_tr, y_tr, cv=cv_strat,
                             scoring=['roc_auc', 'f1', 'recall'], n_jobs=1)

print(f"\nRandom Forest (class_weight=balanced):")
print(f"  AUC={scores_rf['test_roc_auc'].mean():.4f} | "
      f"F1={scores_rf['test_f1'].mean():.4f} | "
      f"Recall={scores_rf['test_recall'].mean():.4f}")

# ── XGBoost con Optuna ────────────────────────────────────────────────────────
# Preprocesar una vez para usar en Optuna directamente (más rápido)
X_tr_pp  = preprocesador.fit_transform(X_tr, y_tr)
X_val_pp = preprocesador.transform(X_val)

scale_pos_weight = (y_tr == 0).sum() / (y_tr == 1).sum()
print(f"\nscale_pos_weight (desbalance): {scale_pos_weight:.2f}")

def objective_churn(trial):
    params = {
        'n_estimators':     trial.suggest_int('n_estimators', 100, 400),
        'max_depth':        trial.suggest_int('max_depth', 3, 7),
        'learning_rate':    trial.suggest_float('learning_rate', 0.01, 0.2, log=True),
        'subsample':        trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'reg_alpha':        trial.suggest_float('reg_alpha', 1e-6, 5.0, log=True),
        'reg_lambda':       trial.suggest_float('reg_lambda', 1e-6, 5.0, log=True),
        'min_child_weight': trial.suggest_int('min_child_weight', 1, 8),
        'scale_pos_weight': scale_pos_weight,  # manejo de desbalance en XGB
        'eval_metric':      'auc',
        'random_state':     42,
        'n_jobs':           -1,
        'verbosity':        0,
    }
    clf = xgb.XGBClassifier(**params)
    scores = cross_val_score(clf, X_tr_pp, y_tr,
                              cv=StratifiedKFold(3, shuffle=True, random_state=42),
                              scoring='roc_auc', n_jobs=-1)
    return scores.mean()

print("\nOptimizando XGBoost (30 trials)...")
t0 = time.time()
study = optuna.create_study(direction='maximize',
                             sampler=optuna.samplers.TPESampler(seed=42))
study.optimize(objective_churn, n_trials=30)
print(f"  Completado en {time.time()-t0:.1f}s")
print(f"  Mejor AUC CV: {study.best_value:.4f}")

# Modelo final XGBoost
best_params = study.best_params
best_params['scale_pos_weight'] = scale_pos_weight
best_params.update({'eval_metric': 'auc', 'random_state': 42, 'n_jobs': -1, 'verbosity': 0})

xgb_best = xgb.XGBClassifier(**best_params)
xgb_best.fit(X_tr_pp, y_tr,
             eval_set=[(X_val_pp, y_val)],
             early_stopping_rounds=20,
             verbose=False)

print(f"  Árboles finales: {xgb_best.best_iteration}")

from sklearn.model_selection import cross_val_score
```

---

## Fase 4: Evaluación

```python
# ── Evaluación comparativa en test ────────────────────────────────────────────
X_te_pp = preprocesador.transform(X_te)

modelos_finales = {
    'Logistic Regression':  (pipe_lr, X_te, True),
    'Random Forest':         (pipe_rf, X_te, True),
    'XGBoost (Optuna)':      (xgb_best, X_te_pp, False),
}

resultados_test = {}

print("\n─── Evaluación en Test Set ───\n")
print(f"{'Modelo':<25} {'AUC':>7} {'F1':>7} {'Recall':>8} {'Prec':>7} {'Acc':>7}")
print('─' * 60)

for nombre, (modelo, X_ev, usa_pipe) in modelos_finales.items():
    if usa_pipe:
        modelo.fit(X_tr, y_tr)
    y_proba = modelo.predict_proba(X_ev)[:, 1]
    y_pred  = modelo.predict(X_ev)

    resultados_test[nombre] = {
        'AUC':     roc_auc_score(y_te, y_proba),
        'F1':      f1_score(y_te, y_pred),
        'Recall':  recall_score(y_te, y_pred),
        'Prec':    f1_score(y_te, y_pred, pos_label=0),
        'Acc':     accuracy_score(y_te, y_pred),
    }
    r = resultados_test[nombre]
    print(f"  {nombre:<23} {r['AUC']:.4f}  {r['F1']:.4f}  {r['Recall']:.4f}  "
          f"{r['Prec']:.4f}  {r['Acc']:.4f}")

# Selección del mejor modelo
mejor_nombre = max(resultados_test, key=lambda k: resultados_test[k]['AUC'])
print(f"\n✓ Mejor modelo por AUC-ROC: {mejor_nombre}")

# ── Evaluación detallada del mejor modelo ─────────────────────────────────────
y_proba_best = modelos_finales[mejor_nombre][0].predict_proba(
    modelos_finales[mejor_nombre][1]
)[:, 1]
y_pred_best = modelos_finales[mejor_nombre][0].predict(modelos_finales[mejor_nombre][1])

print(f"\nReporte del mejor modelo ({mejor_nombre}):")
print(classification_report(y_te, y_pred_best, target_names=['No Churn', 'Churn']))

# ── Análisis de threshold ─────────────────────────────────────────────────────
from sklearn.metrics import precision_recall_curve, roc_curve

thresholds = np.linspace(0.1, 0.9, 81)
recalls, precisions, f1s = [], [], []
for t in thresholds:
    y_t = (y_proba_best >= t).astype(int)
    recalls.append(recall_score(y_te, y_t, zero_division=0))
    precisions.append(f1_score(y_te, y_t, pos_label=0, zero_division=0))  # precision inversores
    f1s.append(f1_score(y_te, y_t, zero_division=0))

opt_threshold = thresholds[np.argmax(f1s)]

fig, axes = plt.subplots(1, 3, figsize=(16, 5))

axes[0].plot(thresholds, recalls, 'r-', lw=2, label='Recall (Churn)')
axes[0].plot(thresholds, f1s, 'g-', lw=2, label='F1')
axes[0].axvline(opt_threshold, color='black', ls='--',
                 label=f'Óptimo t={opt_threshold:.2f}')
axes[0].set_title('Threshold vs Métricas')
axes[0].legend()

# ROC
fpr, tpr, _ = roc_curve(y_te, y_proba_best)
auc_val = roc_auc_score(y_te, y_proba_best)
axes[1].plot(fpr, tpr, 'b-', lw=2, label=f'AUC={auc_val:.4f}')
axes[1].plot([0,1],[0,1],'k--')
axes[1].set_xlabel('FPR'); axes[1].set_ylabel('TPR')
axes[1].set_title('Curva ROC')
axes[1].legend()

# Matriz de confusión con threshold óptimo
y_pred_opt = (y_proba_best >= opt_threshold).astype(int)
ConfusionMatrixDisplay.from_predictions(
    y_te, y_pred_opt,
    display_labels=['No Churn', 'Churn'],
    cmap='Blues', ax=axes[2]
)
axes[2].set_title(f'Confusion Matrix (t={opt_threshold:.2f})')

plt.suptitle(f'Evaluación final — {mejor_nombre}', fontsize=13)
plt.tight_layout()
plt.savefig('outputs/m16_evaluacion_final.png', dpi=120)
plt.show()

print(f"\nThreshold óptimo: {opt_threshold:.2f}")
print(f"  Recall:    {recall_score(y_te, y_pred_opt):.4f}")
print(f"  F1:        {f1_score(y_te, y_pred_opt):.4f}")
print(f"  Accuracy:  {accuracy_score(y_te, y_pred_opt):.4f}")
```

---

## Fase 5: Explicabilidad

```python
# ── SHAP para XGBoost ─────────────────────────────────────────────────────────
try:
    import shap

    # Obtener nombres de features del preprocesador
    feature_names_out = []
    for name, trans, cols in preprocesador.transformers_:
        if name == 'num':
            feature_names_out.extend(cols)
        elif name == 'bin':
            feature_names_out.extend(cols)
        elif name == 'nom':
            try:
                ohe = trans.named_steps['ohe']
                for col, cats in zip(cols, ohe.categories_):
                    feature_names_out.extend([f'{col}_{c}' for c in cats])
            except Exception:
                feature_names_out.extend([f'{c}_enc' for c in cols])
        elif name == 'ord':
            feature_names_out.extend(cols)

    # Calcular SHAP values
    explainer = shap.TreeExplainer(xgb_best)
    shap_values = explainer.shap_values(X_te_pp[:200])  # submuestra

    # Feature names ajustados al número de columnas
    n_feats = X_te_pp.shape[1]
    if len(feature_names_out) > n_feats:
        feature_names_out = feature_names_out[:n_feats]
    elif len(feature_names_out) < n_feats:
        feature_names_out += [f'f{i}' for i in range(len(feature_names_out), n_feats)]

    fig, axes = plt.subplots(1, 2, figsize=(16, 6))

    plt.sca(axes[0])
    shap.summary_plot(shap_values, X_te_pp[:200], plot_type='bar',
                      feature_names=feature_names_out[:n_feats],
                      show=False, max_display=15)
    axes[0].set_title('SHAP — Importancia global')

    plt.sca(axes[1])
    shap.summary_plot(shap_values, X_te_pp[:200],
                      feature_names=feature_names_out[:n_feats],
                      show=False, max_display=15)
    axes[1].set_title('SHAP — Distribución por muestra')

    plt.tight_layout()
    plt.savefig('outputs/m16_shap.png', dpi=120, bbox_inches='tight')
    plt.show()

    # Explicación de clientes de alto riesgo
    idx_alto_riesgo = np.where(y_proba_best > 0.7)[0]
    print(f"\nClientes de alto riesgo (prob > 70%): {len(idx_alto_riesgo)}")
    if len(idx_alto_riesgo) > 0:
        i = idx_alto_riesgo[0]
        print(f"\nExplicación del cliente más en riesgo:")
        print(f"  Probabilidad de churn: {y_proba_best[i]:.2%}")
        shap_i = shap_values[i]
        top5 = np.argsort(np.abs(shap_i))[::-1][:5]
        for j in top5:
            fname = feature_names_out[j] if j < len(feature_names_out) else f'f{j}'
            print(f"  {fname:<35} SHAP={shap_i[j]:+.4f}")

except ImportError:
    print("\n(shap no instalado — pip install shap)")

print("""
Insights de negocio a comunicar al equipo:
  1. Clientes con contrato mensual → mayor riesgo (sin compromiso a largo plazo)
  2. Fibra óptica → paradójicamente mayor churn (expectativas altas, más quejas)
  3. Tenure bajo (< 12 meses) → período crítico de retención
  4. Sin soporte técnico online → mayor insatisfacción
  5. Clientes leales (> 4 años) → mínimo riesgo
""")
```

---

## Fase 6: Despliegue

```python
# ── Pipeline completo para producción ────────────────────────────────────────
import joblib

# Pipeline de producción: FE + preprocesador + modelo
# Nota: XGBoost necesita preprocesador separado (no sklearn native)
# Para producción, empaquetamos como dict con los objetos necesarios

pipeline_produccion = {
    'feature_engineer': TelcoFeatureEngineer(),
    'preprocessor':     preprocesador,
    'model':            xgb_best,
    'threshold':        opt_threshold,
    'metadata': {
        'version':         '1.0.0',
        'target':          'Churn',
        'features_input':  list(X_raw.columns),
        'auc_test':        resultados_test[mejor_nombre]['AUC'],
        'recall_test':     resultados_test[mejor_nombre]['Recall'],
        'churn_rate_train': float(y_tr.mean()),
    }
}

joblib.dump(pipeline_produccion, 'models/churn_pipeline_v1.pkl')
print("Pipeline de producción guardado: models/churn_pipeline_v1.pkl")

# ── Función de inferencia ─────────────────────────────────────────────────────
def predecir_churn(datos_nuevos: pd.DataFrame,
                    pipeline_path: str = 'models/churn_pipeline_v1.pkl') -> pd.DataFrame:
    """
    Predice la probabilidad de churn para nuevos clientes.

    Parámetros:
        datos_nuevos: DataFrame con las mismas columnas que el dataset original
                      (sin customerID y sin Churn)
        pipeline_path: ruta al pipeline serializado

    Retorna:
        DataFrame con columnas: customerID (si existe), churn_prob, churn_pred, riesgo
    """
    pipeline = joblib.load(pipeline_path)
    fe       = pipeline['feature_engineer']
    prep     = pipeline['preprocessor']
    model    = pipeline['model']
    threshold = pipeline['threshold']

    # Feature engineering
    df_eng = fe.transform(datos_nuevos)

    # Preprocesamiento
    X_pp = prep.transform(df_eng)

    # Predicción
    proba = model.predict_proba(X_pp)[:, 1]
    pred  = (proba >= threshold).astype(int)

    resultado = pd.DataFrame({
        'churn_prob': proba.round(4),
        'churn_pred': pred,
        'riesgo':     pd.cut(proba, bins=[0, 0.3, 0.6, 1.0],
                              labels=['Bajo', 'Medio', 'Alto'],
                              include_lowest=True)
    })

    if 'customerID' in datos_nuevos.columns:
        resultado.insert(0, 'customerID', datos_nuevos['customerID'].values)

    return resultado


# ── Test de la función de inferencia ─────────────────────────────────────────
nuevos_clientes = df.sample(5, random_state=99)
X_nuevos = nuevos_clientes.drop(columns=['Churn'])
y_real   = nuevos_clientes['Churn'].values

predicciones = predecir_churn(X_nuevos)
print("\n─── Predicciones en nuevos clientes ───")
print(predicciones.to_string())
print(f"\nChurn real:     {y_real}")
print(f"Churn predicho: {predicciones['churn_pred'].values}")

# ── Reporte de negocio ────────────────────────────────────────────────────────
# Segmentación por nivel de riesgo para acciones diferenciadas
proba_todos = predecir_churn(df.drop(columns=['Churn']))
segmentos = proba_todos.groupby('riesgo').size()

print("\n─── Segmentación de clientes por riesgo ───")
for riesgo, count in segmentos.items():
    pct = count / len(df) * 100
    print(f"  {riesgo:<6}: {count:4d} clientes ({pct:.1f}%)")

print("""
Acciones recomendadas por segmento:
  Bajo (<30%):   Sin acción proactiva — monitoreo estándar
  Medio (30-60%): Email/SMS con oferta de retención moderada
  Alto (>60%):   Llamada proactiva + oferta de descuento personalizado
""")
```

---

## Checklist

```
╔═════════════════════════════════════════════════════════════════════════╗
║           CHECKLIST DEL PROYECTO ML — TELCO CHURN                      ║
╠═════════════════════════════════════════════════════════════════════════╣
║ ✅ EDA: distribución del target, correlaciones, insights de negocio     ║
║ ✅ Manejo de NaN: SimpleImputer / KNNImputer                            ║
║ ✅ Encoding: OneHotEncoder (nominal) + OrdinalEncoder (ordinal)         ║
║ ✅ Scaling: StandardScaler para variables numéricas                     ║
║ ✅ Feature Engineering: tenure_segment, n_servicios, cost_per_tenure    ║
║ ✅ Manejo de desbalance: class_weight / scale_pos_weight                ║
║ ✅ Pipeline completo con ColumnTransformer                              ║
║ ✅ Baseline: Logistic Regression                                        ║
║ ✅ Modelos: Random Forest, XGBoost                                      ║
║ ✅ Optimización: Optuna (30 trials)                                     ║
║ ✅ Evaluación: AUC-ROC, F1, Recall, Precision (train/val/test)         ║
║ ✅ Análisis de threshold: maximizar F1 / Recall según el negocio        ║
║ ✅ Curvas ROC y PR, matriz de confusión                                 ║
║ ✅ Explicabilidad: SHAP (importancia global + explicación individual)   ║
║ ✅ Insights de negocio comunicados en lenguaje no técnico               ║
║ ✅ Pipeline serializado con joblib                                      ║
║ ✅ Función de inferencia para producción                                ║
║ ✅ Segmentación de clientes por nivel de riesgo                         ║
╚═════════════════════════════════════════════════════════════════════════╝
```

---

## Resumen del tutorial T1

```
┌─────────────────────────────────────────────────────────────────────────────┐
│         TUTORIAL 1 COMPLETADO — Machine Learning Clásico con Python         │
├─────────────────────────────────────────────────────────────────────────────┤
│  M01: Preprocesamiento completo (imputation, encoding, scaling, FE)         │
│  M02: Regresión lineal (OLS, Ridge, Lasso, ElasticNet)                      │
│  M03: Regresión avanzada (Poly, SVR, DT, RF) + benchmark                    │
│  M04: Evaluación de regresión (RMSE, R², curvas, Optuna)                    │
│  M05: Clasificación: Logistic Regression + KNN                              │
│  M06: SVM y Kernel SVM (RBF, poly, GridSearch)                              │
│  M07: Naive Bayes, Decision Tree, Random Forest, XGBoost                    │
│  M08: Evaluación de clasificación (ROC, PR, desbalance, CV)                 │
│  M09: Clustering (K-Means, DBSCAN, Jerárquico, GMM)                        │
│  M10: Reglas de asociación (Apriori, FP-Growth, mlxtend)                   │
│  M11: Reducción de dimensionalidad (PCA, LDA, t-SNE, UMAP)                 │
│  M12: NLP clásico (TF-IDF, clasificación de texto, Word2Vec)               │
│  M13: Redes Neuronales ANN con PyTorch (MLP, BCELoss, Adam)                │
│  M14: CNN con PyTorch (CIFAR-10, Transfer Learning, TextCNN)               │
│  M15: Selección de features, Optuna, Stacking, SHAP                        │
│  M16: Proyecto integrador end-to-end (Telco Churn)                         │
│                                                                             │
│  → Siguiente: Tutorial 2 — Computación Cuántica y QML                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Checkpoint M16

```python
# checkpoint_m16.py
"""
Checkpoint M16 — Proyecto Integrador
Ejecutar: python checkpoint_m16.py
Verifica que todo el pipeline de extremo a extremo funciona correctamente.
"""
import numpy as np
import pandas as pd
import warnings
warnings.filterwarnings('ignore')

from sklearn.model_selection import train_test_split, StratifiedKFold, cross_val_score
from sklearn.preprocessing import StandardScaler, OneHotEncoder, OrdinalEncoder
from sklearn.impute import SimpleImputer
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import roc_auc_score, f1_score, recall_score, accuracy_score
import xgboost as xgb
import joblib, os

print("=" * 60)
print("CHECKPOINT M16 — Proyecto Integrador")
print("=" * 60)

# 1. Cargar o generar dataset
try:
    df = pd.read_csv('data/telco_churn.csv')
    print(f"\n✓ Dataset cargado: {df.shape}")
except FileNotFoundError:
    # Generación rápida si no existe
    np.random.seed(42)
    n = 1000
    df = pd.DataFrame({
        'tenure':           np.random.randint(1, 73, n).astype(float),
        'MonthlyCharges':   np.random.uniform(18, 118, n),
        'TotalCharges':     np.random.uniform(100, 8000, n),
        'SeniorCitizen':    np.random.choice([0, 1], n),
        'Contract':         np.random.choice(['Month-to-month','One year','Two year'], n),
        'InternetService':  np.random.choice(['DSL','Fiber optic','No'], n),
        'PaperlessBilling': np.random.choice(['Yes','No'], n),
        'Churn':            np.random.choice([0,1], n, p=[0.74, 0.26])
    })
    print(f"\n✓ Dataset generado: {df.shape}")

assert len(df) > 100, "Dataset demasiado pequeño"
assert 'Churn' in df.columns

# 2. Preprocesamiento básico
NUM_COLS = ['tenure', 'MonthlyCharges', 'TotalCharges']
BIN_COLS = ['SeniorCitizen']
CAT_COLS = ['Contract', 'InternetService', 'PaperlessBilling'] if all(
    c in df.columns for c in ['Contract', 'InternetService', 'PaperlessBilling']
) else []

feat_cols = NUM_COLS + BIN_COLS + CAT_COLS
X = df[feat_cols]
y = df['Churn'].values

X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2,
                                            stratify=y, random_state=42)

transformers = [
    ('num', StandardScaler(), NUM_COLS),
    ('bin', 'passthrough', BIN_COLS),
]
if CAT_COLS:
    transformers.append(
        ('cat', OneHotEncoder(handle_unknown='ignore', sparse_output=False), CAT_COLS)
    )

prep = ColumnTransformer(transformers=transformers)

# 3. Pipeline completo
pipe = Pipeline([
    ('prep', prep),
    ('clf', RandomForestClassifier(n_estimators=50, class_weight='balanced',
                                    random_state=42, n_jobs=-1))
])

# Cross-validation
cv_scores = cross_val_score(pipe, X_tr, y_tr,
                             cv=StratifiedKFold(5, shuffle=True, random_state=42),
                             scoring='roc_auc', n_jobs=-1)
print(f"✓ Pipeline RF — CV AUC: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
assert cv_scores.mean() > 0.65, f"AUC muy baja: {cv_scores.mean():.4f}"

# 4. Evaluación en test
pipe.fit(X_tr, y_tr)
y_proba = pipe.predict_proba(X_te)[:, 1]
y_pred  = pipe.predict(X_te)

auc  = roc_auc_score(y_te, y_proba)
f1   = f1_score(y_te, y_pred)
rec  = recall_score(y_te, y_pred)
acc  = accuracy_score(y_te, y_pred)

print(f"✓ Test: AUC={auc:.4f} | F1={f1:.4f} | Recall={rec:.4f} | Acc={acc:.4f}")
assert auc > 0.60

# 5. Análisis de threshold
thresholds = np.linspace(0.1, 0.9, 81)
f1s = [f1_score(y_te, (y_proba >= t).astype(int), zero_division=0) for t in thresholds]
opt_t = thresholds[np.argmax(f1s)]
print(f"✓ Threshold óptimo: {opt_t:.2f} | F1 óptimo: {max(f1s):.4f}")
assert 0.1 <= opt_t <= 0.9

# 6. XGBoost
xgb_clf = xgb.XGBClassifier(n_estimators=50, scale_pos_weight=(y_tr==0).sum()/(y_tr==1).sum(),
                               eval_metric='auc', random_state=42, verbosity=0, n_jobs=-1)
pipe_xgb = Pipeline([('prep', ColumnTransformer(transformers=transformers)), ('clf', xgb_clf)])
pipe_xgb.fit(X_tr, y_tr)
auc_xgb = roc_auc_score(y_te, pipe_xgb.predict_proba(X_te)[:,1])
print(f"✓ XGBoost Test AUC: {auc_xgb:.4f}")

# 7. Serialización
os.makedirs('models', exist_ok=True)
artefacto = {'pipeline': pipe, 'threshold': opt_t,
             'auc_test': auc, 'recall_test': rec}
joblib.dump(artefacto, 'models/checkpoint_m16_churn.pkl')

cargado = joblib.load('models/checkpoint_m16_churn.pkl')
y_pred_check = (cargado['pipeline'].predict_proba(X_te)[:,1] >= cargado['threshold']).astype(int)
assert np.array_equal(y_pred_check, (y_proba >= opt_t).astype(int))
print("✓ Serialización del pipeline completo: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M16 COMPLETADO ✓")
print("\nTUTORIAL 1 COMPLETADO — Machine Learning Clásico con Python")
print("=" * 60)
print("\nTodos los módulos superados:")
print("  M01: Preprocesamiento    M05: Logistic + KNN")
print("  M02: Regresión           M06: SVM + Kernel SVM")
print("  M03: Regresión avanzada  M07: NB, DT, RF, XGBoost")
print("  M04: Evaluación regres.  M08: Evaluación clasificación")
print("                           M09: Clustering")
print("                           M10: Reglas de asociación")
print("                           M11: Dimensionalidad")
print("                           M12: NLP Clásico")
print("                           M13: ANN / PyTorch")
print("                           M14: CNN / PyTorch")
print("                           M15: Selección modelos")
print("                           M16: Proyecto integrador ✓")
print("\n→ Continúa con Tutorial 2: Computación Cuántica")
print("=" * 60)
```

---

*¡Tutorial 1 completado! → Continúa con Tutorial 2: Computación Cuántica y QML*
