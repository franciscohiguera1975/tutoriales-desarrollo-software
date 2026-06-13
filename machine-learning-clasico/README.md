# Tutorial 1 — Machine Learning Clásico con Python

> Todo el currículo estándar de ML implementado en Python puro.
> Sin referencias a computación cuántica — este tutorial es autónomo.

---

## Setup del entorno

```bash
conda create -n ml-clasico python=3.11 -y
conda activate ml-clasico
pip install numpy pandas scikit-learn matplotlib seaborn torch torchvision
pip install xgboost nltk jupyter ipykernel
python -m ipykernel install --user --name ml-clasico --display-name "ML Clásico"
```

---

## Módulos

### Parte 1 — Preprocesamiento
| Módulo | Tema | Archivo |
|---|---|---|
| M01 | Preprocesamiento de Datos | [modulo-01-preprocessing.md](./modulo-01-preprocessing.md) |

### Parte 2 — Regresión
| Módulo | Tema | Archivo |
|---|---|---|
| M02 | Regresión Simple y Múltiple | [modulo-02-regresion.md](./modulo-02-regresion.md) |
| M03 | Regresión Polinomial, SVR, Decision Tree, Random Forest | [modulo-03-regresion-avanzada.md](./modulo-03-regresion-avanzada.md) |
| M04 | Evaluación y Selección de Modelos de Regresión | [modulo-04-evaluacion-regresion.md](./modulo-04-evaluacion-regresion.md) |

### Parte 3 — Clasificación
| Módulo | Tema | Archivo |
|---|---|---|
| M05 | Logistic Regression y K-Nearest Neighbors | [modulo-05-clasificacion-logistica-knn.md](./modulo-05-clasificacion-logistica-knn.md) |
| M06 | Support Vector Machine y Kernel SVM | [modulo-06-clasificacion-svm.md](./modulo-06-clasificacion-svm.md) |
| M07 | Naive Bayes, Decision Tree y Random Forest | [modulo-07-clasificacion-trees.md](./modulo-07-clasificacion-trees.md) |
| M08 | Evaluación de Modelos de Clasificación | [modulo-08-evaluacion-clasificacion.md](./modulo-08-evaluacion-clasificacion.md) |

### Parte 4 — Clustering
| Módulo | Tema | Archivo |
|---|---|---|
| M09 | K-Means y Hierarchical Clustering | [modulo-09-clustering.md](./modulo-09-clustering.md) |

### Parte 5 — Association Rules
| Módulo | Tema | Archivo |
|---|---|---|
| M10 | Apriori y Eclat | [modulo-10-association-rules.md](./modulo-10-association-rules.md) |

### Parte 6 — Reinforcement Learning
| Módulo | Tema | Archivo |
|---|---|---|
| M11 | Upper Confidence Bound y Thompson Sampling | [modulo-11-reinforcement-learning.md](./modulo-11-reinforcement-learning.md) |

### Parte 7 — NLP
| Módulo | Tema | Archivo |
|---|---|---|
| M12 | Procesamiento de Lenguaje Natural | [modulo-12-nlp.md](./modulo-12-nlp.md) |

### Parte 8 — Deep Learning
| Módulo | Tema | Archivo |
|---|---|---|
| M13 | Redes Neuronales Artificiales (ANN) | [modulo-13-deep-learning-ann.md](./modulo-13-deep-learning-ann.md) |
| M14 | Redes Neuronales Convolucionales (CNN) | [modulo-14-deep-learning-cnn.md](./modulo-14-deep-learning-cnn.md) |

### Parte 9 — Reducción de Dimensionalidad
| Módulo | Tema | Archivo |
|---|---|---|
| M15 | PCA, LDA y Kernel PCA | [modulo-15-dimensionality-reduction.md](./modulo-15-dimensionality-reduction.md) |

### Parte 10 — Model Selection y Boosting
| Módulo | Tema | Archivo |
|---|---|---|
| M16 | Model Selection y XGBoost | [modulo-16-model-selection-xgboost.md](./modulo-16-model-selection-xgboost.md) |

---

## Datasets utilizados

| Dataset | Módulos | Descripción |
|---|---|---|
| Salary | M02 | Regresión simple — años de experiencia vs salario |
| 50 Startups | M02 | Regresión múltiple — gastos vs ganancias |
| Position Salaries | M03 | Regresión polinomial |
| Social Network Ads | M05, M06 | Clasificación binaria |
| Breast Cancer (sklearn) | M07, M08 | Clasificación multiclase |
| Mall Customers | M09 | Clustering sin etiquetas |
| Market Basket | M10 | Reglas de asociación |
| IMDB Reviews | M12 | Análisis de sentimiento |
| MNIST | M13, M14 | Clasificación de dígitos |
| Iris | M15 | Reducción de dimensionalidad |

---

## Checklist de progreso

- [ ] M01 — Preprocessing
- [ ] M02 — Regresión Simple/Múltiple
- [ ] M03 — Regresión Avanzada
- [ ] M04 — Evaluación Regresión
- [ ] M05 — Logística + KNN
- [ ] M06 — SVM + Kernel SVM
- [ ] M07 — Naive Bayes + Trees
- [ ] M08 — Evaluación Clasificación
- [ ] M09 — Clustering
- [ ] M10 — Association Rules
- [ ] M11 — Reinforcement Learning
- [ ] M12 — NLP
- [ ] M13 — ANN
- [ ] M14 — CNN
- [ ] M15 — Dimensionality Reduction
- [ ] M16 — Model Selection + XGBoost
