# Módulo 15 — Selección de Modelos, Pipelines Avanzados y XGBoost Avanzado

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M08 (Evaluación de Clasificación), M07 (Random Forest, XGBoost)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Selección de features](#selección-de-features)
3. [Pipelines avanzados con ColumnTransformer](#pipelines-avanzados)
4. [Búsqueda de hiperparámetros avanzada](#búsqueda-de-hiperparámetros)
5. [XGBoost avanzado — todos los hiperparámetros](#xgboost-avanzado)
6. [LightGBM y CatBoost](#lightgbm-y-catboost)
7. [Stacking y Blending](#stacking-y-blending)
8. [SHAP — explicabilidad de modelos](#shap)
9. [Checkpoint M15](#checkpoint-m15)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install lightgbm catboost shap optuna
python modulo-15-seleccion-modelos-xgboost.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)
```

### Opción C — Google Colab

```python
!pip install -q lightgbm catboost shap optuna
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Selección de features

```
Problema: demasiadas features pueden:
  → Sobreajustar (overfitting)
  → Ralentizar el entrenamiento
  → Dificultar la interpretación
  → Incluir ruido o features irrelevantes

Métodos de selección:
  1. Filter methods:   estadísticas univariadas (independientes del modelo)
  2. Wrapper methods:  búsqueda basada en rendimiento del modelo (costoso)
  3. Embedded:         el modelo mismo selecciona (Lasso L1, RF importance, XGB)
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings, os
warnings.filterwarnings('ignore')
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

from sklearn.datasets import load_breast_cancer, make_classification
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_validate, cross_val_score)
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.metrics import accuracy_score, roc_auc_score
from sklearn.feature_selection import (
    SelectKBest, SelectPercentile,
    f_classif, mutual_info_classif,
    RFE, RFECV,
    SelectFromModel, VarianceThreshold
)
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression, Lasso

cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
feature_names = cancer.feature_names

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

print(f"Dataset: {X.shape} | Features: {feature_names[:5].tolist()}...")

# ── 1. Filter methods — estadísticos univariados ──────────────────────────────
# f_classif: ANOVA F-value (asume normalidad)
# mutual_info_classif: información mutua (no paramétrico)

selector_f    = SelectKBest(score_func=f_classif, k=10)
selector_mi   = SelectKBest(score_func=mutual_info_classif, k=10)

selector_f.fit(X_train, y_train)
selector_mi.fit(X_train, y_train)

df_scores = pd.DataFrame({
    'Feature':         feature_names,
    'F-score':         selector_f.scores_,
    'MI-score':        selector_mi.scores_,
    'F-rank':          np.argsort(np.argsort(-selector_f.scores_)) + 1,
    'MI-rank':         np.argsort(np.argsort(-selector_mi.scores_)) + 1,
}).sort_values('F-score', ascending=False)

print("\n─── Top 10 features por F-score y Mutual Information ───")
print(df_scores.head(10)[['Feature', 'F-score', 'F-rank', 'MI-score', 'MI-rank']].to_string())

# Correlación entre rankings
from scipy.stats import spearmanr
corr, _ = spearmanr(df_scores['F-rank'], df_scores['MI-rank'])
print(f"\nCorrelación Spearman entre rankings: {corr:.4f}")

# ── 2. Wrapper: RFE (Recursive Feature Elimination) ──────────────────────────
rfe = RFE(
    estimator=LogisticRegression(max_iter=2000, C=1),
    n_features_to_select=10,
    step=1
)
rfe.fit(StandardScaler().fit_transform(X_train), y_train)

features_rfe = feature_names[rfe.support_]
print(f"\nRFE — Top 10 features: {list(features_rfe)}")

# RFECV — encuentra automáticamente el número óptimo de features
rfecv = RFECV(
    estimator=LogisticRegression(max_iter=2000, C=1),
    step=1,
    cv=StratifiedKFold(5, shuffle=True, random_state=42),
    scoring='roc_auc',
    min_features_to_select=3,
    n_jobs=-1
)
sc_rfecv = StandardScaler()
X_tr_rfecv = sc_rfecv.fit_transform(X_train)
rfecv.fit(X_tr_rfecv, y_train)

print(f"\nRFECV — número óptimo de features: {rfecv.n_features_}")

fig, ax = plt.subplots(figsize=(10, 4))
ax.plot(range(1, len(rfecv.cv_results_['mean_test_score'])+1),
        rfecv.cv_results_['mean_test_score'], 'b-o', ms=4)
ax.fill_between(
    range(1, len(rfecv.cv_results_['mean_test_score'])+1),
    rfecv.cv_results_['mean_test_score'] - rfecv.cv_results_['std_test_score'],
    rfecv.cv_results_['mean_test_score'] + rfecv.cv_results_['std_test_score'],
    alpha=0.15
)
ax.axvline(rfecv.n_features_, color='red', ls='--',
           label=f'Óptimo: {rfecv.n_features_} features')
ax.set_xlabel('Número de features')
ax.set_ylabel('AUC-ROC (CV-5)')
ax.set_title('RFECV — número óptimo de features')
ax.legend()
plt.tight_layout()
plt.savefig('outputs/m15_rfecv.png', dpi=120)
plt.show()

# ── 3. Embedded: SelectFromModel (L1/RF importance) ──────────────────────────
# Lasso L1 — coeficientes a cero
sc_emb = StandardScaler()
X_tr_emb = sc_emb.fit_transform(X_train)

lasso = Lasso(alpha=0.01, max_iter=5000)
lasso.fit(X_tr_emb, y_train)
lasso_mask = lasso.coef_ != 0
print(f"\nLasso (alpha=0.01): {lasso_mask.sum()} features seleccionadas")
if lasso_mask.any():
    print(f"  Features: {feature_names[lasso_mask].tolist()}")

# Random Forest importance
sfm_rf = SelectFromModel(
    RandomForestClassifier(n_estimators=100, random_state=42),
    threshold='median'   # selecciona features con importance > mediana
)
sfm_rf.fit(X_train, y_train)
features_rf_sfm = feature_names[sfm_rf.get_support()]
print(f"\nSelectFromModel(RF): {len(features_rf_sfm)} features")
print(f"  {list(features_rf_sfm)[:8]}")

# ── Comparativa de métodos de selección ──────────────────────────────────────
cv_strat = StratifiedKFold(5, shuffle=True, random_state=42)

print("\n─── Comparativa de selección de features (AUC-ROC, CV-5) ───")
configs = [
    ('Todas las features (30)',   Pipeline([('sc', StandardScaler()),
                                            ('clf', LogisticRegression(C=1, max_iter=2000))])),
    ('SelectKBest-F (k=10)',      Pipeline([('sc', StandardScaler()),
                                            ('sel', SelectKBest(f_classif, k=10)),
                                            ('clf', LogisticRegression(C=1, max_iter=2000))])),
    ('SelectKBest-MI (k=10)',     Pipeline([('sc', StandardScaler()),
                                            ('sel', SelectKBest(mutual_info_classif, k=10)),
                                            ('clf', LogisticRegression(C=1, max_iter=2000))])),
    (f'RFECV (k={rfecv.n_features_})', Pipeline([('sc', StandardScaler()),
                                                   ('sel', rfecv),
                                                   ('clf', LogisticRegression(C=1, max_iter=2000))])),
    ('SelectFromModel RF',         Pipeline([('sc', StandardScaler()),
                                             ('sel', SelectFromModel(
                                                 RandomForestClassifier(n_estimators=50, random_state=42),
                                                 threshold='median')),
                                             ('clf', LogisticRegression(C=1, max_iter=2000))])),
]

for nombre, pipe in configs:
    scores = cross_val_score(pipe, X, y, cv=cv_strat, scoring='roc_auc', n_jobs=-1)
    print(f"  {nombre:<35} AUC={scores.mean():.4f} ± {scores.std():.4f}")
```

---

## Pipelines avanzados

```python
# ── ColumnTransformer — diferentes transformaciones por tipo de feature ────────
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import (StandardScaler, OrdinalEncoder,
                                    OneHotEncoder, PowerTransformer)
from sklearn.impute import SimpleImputer, KNNImputer

# Dataset mixto: numérico + categórico
np.random.seed(42)
n = 500

df_mixto = pd.DataFrame({
    # Numéricas
    'edad':       np.random.randint(18, 70, n).astype(float),
    'salario':    np.random.exponential(40000, n) + 20000,
    'antiguedad': np.random.randint(0, 30, n).astype(float),
    # Categóricas nominales
    'departamento': np.random.choice(['IT', 'Ventas', 'RRHH', 'Finanzas'], n),
    'ciudad':       np.random.choice(['Madrid', 'Barcelona', 'Valencia', 'Sevilla'], n),
    # Categóricas ordinales
    'nivel_edu':    np.random.choice(['secundaria', 'grado', 'master', 'doctorado'], n),
    # Con NaN
    'experiencia':  np.where(np.random.random(n) < 0.15, np.nan,
                             np.random.randint(0, 20, n).astype(float)),
})

# Target
y_mixto = (df_mixto['salario'] > df_mixto['salario'].median()).astype(int)
df_mixto.drop(columns=['salario'], inplace=True)

X_tr_mx, X_te_mx, y_tr_mx, y_te_mx = train_test_split(
    df_mixto, y_mixto, test_size=0.2, random_state=42
)

print(f"Dataset mixto: {df_mixto.shape}")
print(df_mixto.dtypes)
print(f"\nNaN: {df_mixto.isnull().sum().sum()}")

# Definir columnas por tipo
num_cols    = ['edad', 'antiguedad', 'experiencia']
nom_cols    = ['departamento', 'ciudad']
ord_cols    = ['nivel_edu']
edu_orden   = ['secundaria', 'grado', 'master', 'doctorado']

# ColumnTransformer
preprocesador = ColumnTransformer(transformers=[
    ('num',  Pipeline([
        ('imputer', KNNImputer(n_neighbors=5)),
        ('scaler',  StandardScaler()),
    ]), num_cols),

    ('nom',  Pipeline([
        ('imputer', SimpleImputer(strategy='most_frequent')),
        ('onehot',  OneHotEncoder(handle_unknown='ignore', sparse_output=False)),
    ]), nom_cols),

    ('ord',  Pipeline([
        ('imputer', SimpleImputer(strategy='most_frequent')),
        ('ordinal', OrdinalEncoder(categories=[edu_orden])),
    ]), ord_cols),
], remainder='drop')

# Pipeline completo
pipe_mixto = Pipeline([
    ('prep', preprocesador),
    ('clf',  RandomForestClassifier(n_estimators=100, random_state=42))
])

pipe_mixto.fit(X_tr_mx, y_tr_mx)
acc_mixto = accuracy_score(y_te_mx, pipe_mixto.predict(X_te_mx))
print(f"\nPipeline mixto — Test Accuracy: {acc_mixto:.4f}")

# Guardar pipeline completo
import joblib
joblib.dump(pipe_mixto, 'models/pipeline_mixto.pkl')
print("Pipeline guardado en models/pipeline_mixto.pkl")

# ── FunctionTransformer — transformaciones personalizadas ─────────────────────
from sklearn.preprocessing import FunctionTransformer

log_transform = FunctionTransformer(
    func=np.log1p,
    inverse_func=np.expm1,
    validate=True
)

print("""
Transformaciones útiles para features sesgadas:
  np.log1p (log(x+1)):  para variables con sesgo positivo (salarios, precios)
  PowerTransformer:     Box-Cox o Yeo-Johnson → hace más gaussiana la distribución
  QuantileTransformer:  transforma a distribución uniforme o normal
""")
```

---

## Búsqueda de hiperparámetros

```python
# ── Optuna — Bayesian Optimization ───────────────────────────────────────────
import optuna
optuna.logging.set_verbosity(optuna.logging.WARNING)

import xgboost as xgb
from sklearn.metrics import roc_auc_score

def objective_xgb(trial):
    """Función objetivo para Optuna — maximizar AUC-ROC."""
    params = {
        'n_estimators':       trial.suggest_int('n_estimators', 50, 500),
        'max_depth':          trial.suggest_int('max_depth', 2, 8),
        'learning_rate':      trial.suggest_float('learning_rate', 1e-3, 0.3, log=True),
        'subsample':          trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree':   trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'reg_alpha':          trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
        'reg_lambda':         trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
        'min_child_weight':   trial.suggest_int('min_child_weight', 1, 10),
        'gamma':              trial.suggest_float('gamma', 0, 5),
        'eval_metric':        'logloss',
        'random_state':       42,
        'n_jobs':             -1,
        'verbosity':          0,
    }

    clf = xgb.XGBClassifier(**params)
    cv_scores = cross_val_score(clf, X_train, y_train,
                                 cv=StratifiedKFold(3, shuffle=True, random_state=42),
                                 scoring='roc_auc', n_jobs=-1)
    return cv_scores.mean()


study = optuna.create_study(direction='maximize',
                             sampler=optuna.samplers.TPESampler(seed=42))
study.optimize(objective_xgb, n_trials=30, show_progress_bar=False)

print(f"\nOptuna XGBoost — Mejor AUC CV: {study.best_value:.4f}")
print(f"Mejores parámetros:")
for k, v in study.best_params.items():
    print(f"  {k}: {v}")

# Entrenar con los mejores parámetros
best_xgb = xgb.XGBClassifier(**study.best_params, eval_metric='logloss',
                                random_state=42, n_jobs=-1, verbosity=0)
best_xgb.fit(X_train, y_train)
y_proba_opt = best_xgb.predict_proba(X_test)[:, 1]
print(f"\nXGBoost Optuna — Test AUC: {roc_auc_score(y_test, y_proba_opt):.4f}")

# ── Visualización de la optimización Optuna ───────────────────────────────────
try:
    from optuna.visualization import plot_optimization_history, plot_param_importances
    import plotly

    fig_hist = plot_optimization_history(study)
    fig_hist.write_image('outputs/m15_optuna_history.png')
    print("Historial de optimización guardado")
except Exception:
    # Visualización manual
    valores_trial = [t.value for t in study.trials]
    mejor_acum    = np.maximum.accumulate(valores_trial)
    fig, ax = plt.subplots(figsize=(10, 4))
    ax.plot(valores_trial, 'b.', alpha=0.5, ms=4, label='Trial')
    ax.plot(mejor_acum, 'r-', lw=2, label='Mejor acumulado')
    ax.set_xlabel('Trial'); ax.set_ylabel('AUC-ROC (CV-3)')
    ax.set_title('Optuna — Historial de optimización')
    ax.legend()
    plt.tight_layout()
    plt.savefig('outputs/m15_optuna_history.png', dpi=120)
    plt.show()
```

---

## XGBoost avanzado

```python
# ── Hiperparámetros clave de XGBoost ─────────────────────────────────────────
print("""
Hiperparámetros principales de XGBoost:
────────────────────────────────────────────────────────────────────────────────
Grupo       Parámetro              Descripción
────────────────────────────────────────────────────────────────────────────────
Boosting    n_estimators           Número de árboles (más = más expresivo pero lento)
            learning_rate (eta)    Paso de aprendizaje — disminuir con más árboles
            max_depth              Profundidad máxima (3-8 típico)

Aleatorio   subsample              Fracción de muestras por árbol (0.6-1.0)
            colsample_bytree       Fracción de features por árbol (0.5-1.0)
            colsample_bylevel      Fracción de features por nivel
            colsample_bynode       Fracción de features por split

Regularz.   reg_alpha (L1)         Regularización L1 en los pesos de las hojas
            reg_lambda (L2)        Regularización L2 en los pesos de las hojas
            gamma (min_split_loss) Mínima ganancia para hacer un split
            min_child_weight       Mínimo de muestras en un nodo hoja

Objetivo    objective              'binary:logistic', 'multi:softprob', 'reg:squarederror'
            eval_metric            'logloss', 'auc', 'rmse', 'mlogloss'

Early stop  early_stopping_rounds  Parar si no mejora en N rondas (requiere eval_set)

Velocidad   tree_method            'hist' (más rápido), 'auto', 'gpu_hist' (GPU)
            n_jobs                 Paralelismo (-1 = todos los cores)
────────────────────────────────────────────────────────────────────────────────
""")

# ── XGBoost con early stopping y eval_set ────────────────────────────────────
X_tr, X_val, y_tr, y_val = train_test_split(
    X_train, y_train, test_size=0.2, stratify=y_train, random_state=42
)

xgb_es = xgb.XGBClassifier(
    n_estimators=1000,
    learning_rate=0.05,
    max_depth=4,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,
    reg_lambda=1.0,
    gamma=0.1,
    min_child_weight=3,
    tree_method='hist',
    eval_metric='logloss',
    random_state=42,
    n_jobs=-1,
    verbosity=0,
)

xgb_es.fit(
    X_tr, y_tr,
    eval_set=[(X_tr, y_tr), (X_val, y_val)],
    early_stopping_rounds=30,
    verbose=False,
)

print(f"XGBoost Early Stopping:")
print(f"  Árboles entrenados:  {xgb_es.best_iteration}")
print(f"  Mejor val log-loss:  {xgb_es.best_score:.4f}")
print(f"  Test AUC-ROC:        {roc_auc_score(y_test, xgb_es.predict_proba(X_test)[:,1]):.4f}")

# Curva de aprendizaje XGBoost
evals_result = xgb_es.evals_result()
fig, ax = plt.subplots(figsize=(10, 4))
ax.plot(evals_result['validation_0']['logloss'], 'b-', lw=1.5, alpha=0.8, label='Train')
ax.plot(evals_result['validation_1']['logloss'], 'r-', lw=1.5, alpha=0.8, label='Val')
ax.axvline(xgb_es.best_iteration, color='green', ls='--',
           label=f'Early stop (iter={xgb_es.best_iteration})')
ax.set_xlabel('Ronda (árbol)'); ax.set_ylabel('Log-loss')
ax.set_title('XGBoost — Curva de aprendizaje')
ax.legend()
plt.tight_layout()
plt.savefig('outputs/m15_xgb_learning_curve.png', dpi=120)
plt.show()
```

---

## LightGBM y CatBoost

```python
# ── LightGBM ──────────────────────────────────────────────────────────────────
try:
    import lightgbm as lgb

    lgb_clf = lgb.LGBMClassifier(
        n_estimators=500,
        learning_rate=0.05,
        num_leaves=31,      # max hojas por árbol (más = más expresivo)
        max_depth=-1,       # sin límite (controlado por num_leaves)
        subsample=0.8,
        colsample_bytree=0.8,
        reg_alpha=0.1,
        reg_lambda=0.1,
        min_child_samples=20,  # mínimo muestras por hoja
        random_state=42,
        n_jobs=-1,
        verbosity=-1,
    )

    lgb_clf.fit(
        X_tr, y_tr,
        eval_set=[(X_val, y_val)],
        callbacks=[lgb.early_stopping(30, verbose=False),
                   lgb.log_evaluation(-1)]
    )

    auc_lgb = roc_auc_score(y_test, lgb_clf.predict_proba(X_test)[:,1])
    print(f"\nLightGBM:")
    print(f"  Árboles: {lgb_clf.best_iteration_}")
    print(f"  Test AUC-ROC: {auc_lgb:.4f}")
    print("  LightGBM es típicamente 2-10x más rápido que XGBoost")

except ImportError:
    print("\n(lightgbm no instalado — pip install lightgbm)")

# ── CatBoost ──────────────────────────────────────────────────────────────────
try:
    from catboost import CatBoostClassifier

    cb_clf = CatBoostClassifier(
        iterations=500,
        learning_rate=0.05,
        depth=6,
        l2_leaf_reg=3,
        random_seed=42,
        verbose=0,
        eval_metric='AUC',
    )

    cb_clf.fit(X_tr, y_tr, eval_set=(X_val, y_val),
               early_stopping_rounds=30, verbose=False)

    auc_cb = roc_auc_score(y_test, cb_clf.predict_proba(X_test)[:,1])
    print(f"\nCatBoost:")
    print(f"  Test AUC-ROC: {auc_cb:.4f}")
    print("  CatBoost maneja variables categóricas nativamente (sin encoding)")

except ImportError:
    print("\n(catboost no instalado — pip install catboost)")

# ── Comparativa final de boosting ─────────────────────────────────────────────
print("\n─── Comparativa Boosting (Breast Cancer) ───")
boosting_models = {
    'XGBoost (básico)':     xgb.XGBClassifier(n_estimators=100, random_state=42,
                                                verbosity=0, n_jobs=-1),
    'XGBoost (optimizado)': best_xgb,
    'XGBoost (ES)':         xgb_es,
}

for nombre, clf in boosting_models.items():
    scores = cross_val_score(clf, X, y, cv=StratifiedKFold(5, shuffle=True, random_state=42),
                              scoring='roc_auc', n_jobs=-1)
    print(f"  {nombre:<25}: AUC={scores.mean():.4f} ± {scores.std():.4f}")
```

---

## Stacking y Blending

```python
# ── Stacking — ensemble de nivel 2 ───────────────────────────────────────────
from sklearn.ensemble import StackingClassifier, VotingClassifier
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.linear_model import LogisticRegression as LR

# Base learners (nivel 1)
base_learners = [
    ('rf',  RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1)),
    ('xgb', xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                                random_state=42, verbosity=0, n_jobs=-1)),
    ('svm', Pipeline([('sc', StandardScaler()),
                      ('svc', SVC(probability=True, kernel='rbf', C=10, random_state=42))])),
    ('knn', Pipeline([('sc', StandardScaler()),
                      ('knn', KNeighborsClassifier(n_neighbors=7, n_jobs=-1))])),
]

# Meta-learner (nivel 2)
stacking = StackingClassifier(
    estimators=base_learners,
    final_estimator=LR(C=1, max_iter=1000),  # regresión logística simple
    cv=5,
    stack_method='predict_proba',
    passthrough=False,  # True: incluye features originales en nivel 2
    n_jobs=-1,
)

scores_stack = cross_validate(stacking, X, y,
                               cv=StratifiedKFold(5, shuffle=True, random_state=42),
                               scoring=['accuracy', 'roc_auc'], n_jobs=1)

print(f"\nStacking (RF + XGB + SVM + KNN → LR):")
print(f"  CV Accuracy: {scores_stack['test_accuracy'].mean():.4f} ± {scores_stack['test_accuracy'].std():.4f}")
print(f"  CV AUC-ROC:  {scores_stack['test_roc_auc'].mean():.4f} ± {scores_stack['test_roc_auc'].std():.4f}")

# ── Voting Classifier ──────────────────────────────────────────────────────────
voting = VotingClassifier(
    estimators=[
        ('rf',  RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1)),
        ('xgb', xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                                    random_state=42, verbosity=0, n_jobs=-1)),
        ('lr',  Pipeline([('sc', StandardScaler()),
                           ('clf', LR(C=1, max_iter=2000))])),
    ],
    voting='soft',  # 'soft' usa probabilidades (mejor); 'hard' usa votos
    weights=[2, 2, 1],  # XGB y RF pesan más
    n_jobs=-1,
)

scores_vote = cross_validate(voting, X, y,
                              cv=StratifiedKFold(5, shuffle=True, random_state=42),
                              scoring=['accuracy', 'roc_auc'], n_jobs=1)

print(f"\nVoting (RF + XGB + LR, soft, weights=[2,2,1]):")
print(f"  CV Accuracy: {scores_vote['test_accuracy'].mean():.4f}")
print(f"  CV AUC-ROC:  {scores_vote['test_roc_auc'].mean():.4f}")

print("""
Stacking vs Voting:
  Voting:   promedio ponderado de probabilidades — simple, menos overhead
  Stacking: el meta-learner aprende CÓMO combinar los base learners
            → generalmente supera a Voting, más costoso computacionalmente
""")
```

---

## SHAP

```python
# ── SHAP — SHapley Additive exPlanations ─────────────────────────────────────
try:
    import shap

    # Entrenar modelo para explicar
    xgb_shap = xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                                   random_state=42, verbosity=0)
    xgb_shap.fit(X_train, y_train)

    # Calcular SHAP values
    explainer = shap.TreeExplainer(xgb_shap)
    shap_values = explainer.shap_values(X_test)

    print(f"\n─── SHAP Values ───")
    print(f"Forma SHAP values: {np.array(shap_values).shape}")  # (n_test, n_features)

    # ── Beeswarm plot — importancia global ────────────────────────────────────
    fig, ax = plt.subplots(figsize=(10, 8))
    shap.summary_plot(shap_values, X_test, feature_names=feature_names,
                      show=False, plot_type='dot')
    plt.title('SHAP Beeswarm — Importancia global de features')
    plt.tight_layout()
    plt.savefig('outputs/m15_shap_beeswarm.png', dpi=120, bbox_inches='tight')
    plt.show()

    # ── Bar plot — importancia media ──────────────────────────────────────────
    fig, ax = plt.subplots(figsize=(10, 6))
    shap.summary_plot(shap_values, X_test, feature_names=feature_names,
                      show=False, plot_type='bar')
    plt.title('SHAP Bar — Importancia media de features')
    plt.tight_layout()
    plt.savefig('outputs/m15_shap_bar.png', dpi=120, bbox_inches='tight')
    plt.show()

    # ── Force plot — explicación individual ───────────────────────────────────
    print("\nExplicación individual (primera muestra de test):")
    print(f"  Predicción: {xgb_shap.predict_proba(X_test[:1])[0, 1]:.4f}")
    print(f"  SHAP base value: {explainer.expected_value:.4f}")
    shap_vals_i = shap_values[0]
    top_pos = np.argsort(shap_vals_i)[-5:][::-1]  # 5 más positivos
    top_neg = np.argsort(shap_vals_i)[:5]          # 5 más negativos

    print(f"\n  Top 5 features que AUMENTAN la predicción (+):")
    for i in top_pos:
        print(f"    {feature_names[i]:<35} SHAP={shap_vals_i[i]:+.4f}  val={X_test[0,i]:.4f}")

    print(f"\n  Top 5 features que DISMINUYEN la predicción (-):")
    for i in top_neg:
        print(f"    {feature_names[i]:<35} SHAP={shap_vals_i[i]:+.4f}  val={X_test[0,i]:.4f}")

    # ── Dependence plot ───────────────────────────────────────────────────────
    top_feature_idx = np.argmax(np.abs(shap_values).mean(0))
    fig, ax = plt.subplots(figsize=(8, 5))
    shap.dependence_plot(top_feature_idx, shap_values, X_test,
                          feature_names=feature_names, show=False, ax=ax)
    plt.title(f'SHAP Dependence — {feature_names[top_feature_idx]}')
    plt.tight_layout()
    plt.savefig('outputs/m15_shap_dependence.png', dpi=120)
    plt.show()

except ImportError:
    print("\n(shap no instalado — pip install shap)")
    print("SHAP es la herramienta más importante para explicabilidad de ML.")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           RESUMEN M15 — Selección de Modelos y Técnicas Avanzadas           │
├───────────────────────┬─────────────────────────────────────────────────────┤
│ Técnica               │ Descripción                                          │
├───────────────────────┼─────────────────────────────────────────────────────┤
│ SelectKBest           │ Selección por estadístico univariado (F, MI)         │
│ RFECV                 │ RFE con CV → encuentra automáticamente k óptimo     │
│ SelectFromModel       │ Selección por importancia del modelo (RF, Lasso)     │
│ ColumnTransformer     │ Transformaciones diferenciadas por tipo de columna   │
│ Optuna                │ Optimización bayesiana de hiperparámetros           │
│ XGBoost ES            │ Early stopping para parar en el número óptimo       │
│ LightGBM              │ Más rápido que XGBoost (leaf-wise growth)           │
│ CatBoost              │ Nativo para categóricas, sin encoding manual         │
│ StackingClassifier    │ Meta-learner aprende a combinar base learners        │
│ VotingClassifier      │ Promedio ponderado de probabilidades                 │
│ SHAP                  │ Explicabilidad: contribución de cada feature         │
└───────────────────────┴─────────────────────────────────────────────────────┘

Flujo recomendado para una competición/proyecto ML:
  1. Baseline: LogReg o RF rápido → establecer métrica objetivo
  2. EDA + Feature Engineering (M01)
  3. Selección de features (RFECV o SelectFromModel)
  4. Ajuste de XGBoost/LGBM con Optuna (30-100 trials)
  5. Ensemble: Voting o Stacking
  6. Interpretación con SHAP
```

---

## Checkpoint M15

```python
# checkpoint_m15.py
"""
Checkpoint M15 — Selección de Modelos y XGBoost Avanzado
Ejecutar: python checkpoint_m15.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')

from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import (train_test_split, StratifiedKFold,
                                      cross_val_score)
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.feature_selection import SelectKBest, f_classif, SelectFromModel
from sklearn.ensemble import RandomForestClassifier, StackingClassifier, VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
import xgboost as xgb
import optuna, joblib, os
optuna.logging.set_verbosity(optuna.logging.ERROR)

print("=" * 60)
print("CHECKPOINT M15 — Selección de Modelos y Técnicas Avanzadas")
print("=" * 60)

cancer = load_breast_cancer()
X, y = cancer.data, cancer.target
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2,
                                            stratify=y, random_state=42)

cv = StratifiedKFold(5, shuffle=True, random_state=42)

# 1. Selección de features
sel = SelectKBest(f_classif, k=10)
X_sel = sel.fit_transform(X_tr, y_tr)
assert X_sel.shape[1] == 10
print(f"\n✓ SelectKBest (k=10): {X_sel.shape}")

# 2. SelectFromModel
sfm = SelectFromModel(RandomForestClassifier(n_estimators=50, random_state=42),
                       threshold='median')
sfm.fit(X_tr, y_tr)
n_sel = sfm.get_support().sum()
print(f"✓ SelectFromModel RF: {n_sel} features seleccionadas")
assert 5 <= n_sel <= 25

# 3. XGBoost básico
xgb_clf = xgb.XGBClassifier(n_estimators=100, eval_metric='logloss',
                               random_state=42, verbosity=0, n_jobs=-1)
scores_xgb = cross_val_score(xgb_clf, X, y, cv=cv, scoring='roc_auc', n_jobs=-1)
print(f"✓ XGBoost CV AUC: {scores_xgb.mean():.4f} ± {scores_xgb.std():.4f}")
assert scores_xgb.mean() > 0.96

# 4. Optuna mini
def objective(trial):
    c = trial.suggest_float('C', 0.01, 10, log=True)
    clf = Pipeline([('sc', StandardScaler()),
                    ('clf', LogisticRegression(C=c, max_iter=1000))])
    return cross_val_score(clf, X_tr, y_tr, cv=3, scoring='roc_auc').mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=10)
print(f"✓ Optuna (LogReg): Mejor AUC={study.best_value:.4f}, C={study.best_params['C']:.4f}")
assert study.best_value > 0.95

# 5. Voting
voting = VotingClassifier(
    estimators=[
        ('rf',  RandomForestClassifier(n_estimators=50, random_state=42, n_jobs=-1)),
        ('xgb', xgb.XGBClassifier(n_estimators=50, eval_metric='logloss',
                                    random_state=42, verbosity=0, n_jobs=-1)),
        ('lr',  Pipeline([('sc', StandardScaler()),
                           ('clf', LogisticRegression(C=1, max_iter=2000))])),
    ],
    voting='soft'
)
scores_vote = cross_val_score(voting, X, y, cv=cv, scoring='roc_auc', n_jobs=1)
print(f"✓ VotingClassifier CV AUC: {scores_vote.mean():.4f}")
assert scores_vote.mean() > 0.96

# 6. Serialización del mejor modelo
os.makedirs('models', exist_ok=True)
xgb_clf.fit(X_tr, y_tr)
joblib.dump(xgb_clf, 'models/checkpoint_m15_xgb.pkl')
loaded = joblib.load('models/checkpoint_m15_xgb.pkl')
auc_loaded = roc_auc_score(y_te, loaded.predict_proba(X_te)[:,1])
print(f"✓ Modelo cargado — Test AUC: {auc_loaded:.4f}")
assert auc_loaded > 0.97

print("\n" + "=" * 60)
print("CHECKPOINT M15 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • SelectKBest y SelectFromModel")
print("  • XGBoost con CV")
print("  • Optuna bayesian optimization")
print("  • VotingClassifier soft")
print("  • Serialización y carga")
print("=" * 60)
```

---

*Siguiente módulo → M16: Proyecto Integrador — Pipeline ML de extremo a extremo*
