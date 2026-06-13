# Módulo 12 — NLP Clásico: Bag of Words, TF-IDF y Análisis de Sentimiento

> **Tutorial 1 · Machine Learning Clásico con Python**
> Prerequisito: M08 (Evaluación de Clasificación)

---

## Tabla de contenidos

1. [Cómo ejecutar este módulo](#cómo-ejecutar-este-módulo)
2. [Preprocesamiento de texto](#preprocesamiento-de-texto)
3. [Bag of Words (BoW)](#bag-of-words)
4. [TF-IDF](#tf-idf)
5. [N-gramas](#n-gramas)
6. [Análisis de sentimiento](#análisis-de-sentimiento)
7. [Clasificación de texto — pipeline completo](#clasificación-de-texto)
8. [Word Embeddings básicos (Word2Vec, GloVe)](#word-embeddings)
9. [Checkpoint M12](#checkpoint-m12)

---

## Cómo ejecutar este módulo

### Opción A — Script local

```bash
conda activate ml-clasico
pip install nltk textblob gensim
python -c "import nltk; nltk.download(['punkt','stopwords','wordnet','averaged_perceptron_tagger','punkt_tab'])"
python modulo-12-nlp-clasico.py
```

### Opción B — Jupyter Notebook

```python
%matplotlib inline
import warnings; warnings.filterwarnings('ignore')
import os, nltk
for d in ['data', 'outputs', 'models']: os.makedirs(d, exist_ok=True)
nltk.download(['punkt','stopwords','wordnet','punkt_tab'], quiet=True)
```

### Opción C — Google Colab

```python
!pip install -q nltk textblob gensim
import nltk
nltk.download(['punkt','stopwords','wordnet','punkt_tab'], quiet=True)
from google.colab import drive
drive.mount('/content/drive')
import os
BASE = '/content/drive/MyDrive/tutoriales-ia/ml-clasico'
os.makedirs(BASE, exist_ok=True)
os.chdir(BASE)
```

---

## Preprocesamiento de texto

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings, os
warnings.filterwarnings('ignore')
for d in ['outputs', 'models']: os.makedirs(d, exist_ok=True)

import re
import string
import nltk
nltk.download(['punkt', 'stopwords', 'wordnet', 'punkt_tab'], quiet=True)

from nltk.corpus import stopwords
from nltk.stem import PorterStemmer, WordNetLemmatizer
from nltk.tokenize import word_tokenize, sent_tokenize

# ── Pipeline de preprocesamiento de texto ─────────────────────────────────────
STOP_WORDS = set(stopwords.words('english'))
stemmer     = PorterStemmer()
lemmatizer  = WordNetLemmatizer()

def limpiar_texto(texto, metodo='lemmatize'):
    """
    Limpieza completa de texto para NLP.

    Pasos:
      1. Minúsculas
      2. Eliminar HTML / URLs / menciones
      3. Eliminar puntuación y números
      4. Tokenizar
      5. Eliminar stopwords
      6. Stemming o Lemmatización
    """
    # 1. Minúsculas
    texto = texto.lower()

    # 2. Eliminar HTML
    texto = re.sub(r'<[^>]+>', ' ', texto)

    # 3. Eliminar URLs
    texto = re.sub(r'http\S+|www\S+', ' ', texto)

    # 4. Eliminar menciones y hashtags (@user, #tag)
    texto = re.sub(r'@\w+|#\w+', ' ', texto)

    # 5. Eliminar caracteres especiales y números
    texto = re.sub(r'[^a-z\s]', ' ', texto)

    # 6. Tokenizar
    tokens = word_tokenize(texto)

    # 7. Eliminar stopwords y tokens muy cortos
    tokens = [t for t in tokens if t not in STOP_WORDS and len(t) > 2]

    # 8. Normalización
    if metodo == 'stem':
        tokens = [stemmer.stem(t) for t in tokens]
    elif metodo == 'lemmatize':
        tokens = [lemmatizer.lemmatize(t) for t in tokens]

    return ' '.join(tokens)


# Ejemplos
textos_ejemplo = [
    "I LOVED this product! It's amazing and worth every penny!! 😊",
    "Terrible quality. Broke after 2 days... very disappointed.",
    "The movie was <b>great</b>! Visit http://reviews.com for more.",
    "Running runs runner ran — stemming/lemmatizing these words",
]

print("─── Preprocesamiento de texto ───")
for txt in textos_ejemplo:
    clean_stem = limpiar_texto(txt, 'stem')
    clean_lem  = limpiar_texto(txt, 'lemmatize')
    print(f"\nOriginal:   {txt[:60]}")
    print(f"Stemming:   {clean_stem}")
    print(f"Lemmatize:  {clean_lem}")

# Diferencia Stemming vs Lemmatización
print("\n─── Stemming vs Lemmatización ───")
words = ['running', 'runner', 'runs', 'easily', 'fairly', 'studies', 'better']
for w in words:
    stem = stemmer.stem(w)
    lemm = lemmatizer.lemmatize(w)
    print(f"  {w:<12} → stem: {stem:<12} lem: {lemm}")

print("""
Stemming:      corta palabras con reglas heurísticas (más rápido, resultado no siempre válido)
Lemmatización: usa el diccionario para obtener la forma base (lema) real de la palabra
→ Para NLP moderno: lemmatización es más precisa; spaCy simplifica esto
""")
```

---

## Bag of Words

```
Bag of Words (BoW):
  Representa cada documento como un vector de conteos de palabras.
  Ignora el ORDEN de las palabras — solo cuenta la frecuencia.

  Vocabulario: todas las palabras únicas del corpus
  Vector: [count(word1), count(word2), ..., count(wordN)]

  Ejemplo:
    Corpus: ["I love cats", "I love dogs", "Cats and dogs"]
    Vocab:  [I, love, cats, dogs, and]
    "I love cats" → [1, 1, 1, 0, 0]
    "I love dogs" → [1, 1, 0, 1, 0]
    "Cats and dogs" → [0, 0, 1, 1, 1]

  Problemas:
    ✗ Vocabulario gigante → matriz muy dispersa (sparse)
    ✗ Palabras comunes ('the', 'is') dominan → usar TF-IDF
    ✗ Sin semántica ni orden
```

```python
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.datasets import fetch_20newsgroups

# ── CountVectorizer básico ────────────────────────────────────────────────────
corpus_simple = [
    "I love machine learning and deep learning",
    "Machine learning is great for data science",
    "Deep learning uses neural networks",
    "Data science requires statistics and programming",
]

cv = CountVectorizer()
X_bow = cv.fit_transform(corpus_simple)

print("─── Bag of Words ───")
print(f"Vocabulario: {len(cv.vocabulary_)} palabras")
print(f"Forma matriz: {X_bow.shape} (documentos × palabras)")
print(f"\nVocabulario: {sorted(cv.vocabulary_.keys())}")
print(f"\nMatriz BoW:")
print(pd.DataFrame(X_bow.toarray(), columns=cv.get_feature_names_out()).to_string())

# ── Parámetros importantes de CountVectorizer ─────────────────────────────────
cv_params = CountVectorizer(
    min_df=2,          # ignorar palabras que aparecen en < 2 docs
    max_df=0.90,       # ignorar palabras en > 90% de docs (muy comunes)
    max_features=5000, # solo las 5000 palabras más frecuentes
    stop_words='english',
    ngram_range=(1, 2) # unigramas + bigramas
)
print(f"\nParámetros adicionales: min_df, max_df, max_features, ngram_range")
```

---

## TF-IDF

```
TF-IDF = Term Frequency × Inverse Document Frequency

TF(t, d) = frecuencia del término t en documento d
           (varias variantes: cruda, logarítmica, normalizada)

IDF(t)   = log(N / df(t)) + 1
           N  = número total de documentos
           df = número de documentos que contienen t

TF-IDF(t, d) = TF(t, d) × IDF(t)

Intuición:
  • Palabras frecuentes en un doc pero raras en el corpus → alta TF-IDF
    (palabras discriminativas, específicas de ese documento)
  • Palabras comunes en todos los docs → IDF bajo → TF-IDF bajo
    (palabras como "the", "is" ya filtradas por stopwords o IDF bajo)

Normalización: cosine normalization → cada vector tiene norma 1
```

```python
# ── TfidfVectorizer ───────────────────────────────────────────────────────────
tfidf = TfidfVectorizer(stop_words='english', max_features=1000)
X_tfidf = tfidf.fit_transform(corpus_simple)

print("\n─── TF-IDF ───")
print(f"Forma: {X_tfidf.shape}")

df_tfidf = pd.DataFrame(
    X_tfidf.toarray().round(3),
    columns=tfidf.get_feature_names_out()
)
print(f"\nMatriz TF-IDF (columnas con >0):")
cols_con_valor = (df_tfidf > 0).any()
print(df_tfidf.loc[:, cols_con_valor].to_string())

# ── 20 Newsgroups — dataset real ──────────────────────────────────────────────
categorias = ['sci.space', 'sci.med', 'rec.sport.hockey', 'talk.politics.guns']
news_train = fetch_20newsgroups(subset='train', categories=categorias,
                                 remove=('headers', 'footers', 'quotes'))
news_test  = fetch_20newsgroups(subset='test',  categories=categorias,
                                 remove=('headers', 'footers', 'quotes'))

print(f"\n─── 20 Newsgroups (4 categorías) ───")
print(f"Train: {len(news_train.data)} docs | Test: {len(news_test.data)} docs")
print(f"Clases: {news_train.target_names}")

# Vectorización con TF-IDF
tfidf_news = TfidfVectorizer(
    max_features=10000,
    stop_words='english',
    ngram_range=(1, 2),
    min_df=2,
    max_df=0.95,
    sublinear_tf=True  # aplica log(1+tf) en vez de tf crudo
)
X_tr_news = tfidf_news.fit_transform(news_train.data)
X_te_news = tfidf_news.transform(news_test.data)

print(f"Forma TF-IDF train: {X_tr_news.shape}")
print(f"Densidad de la matriz: {X_tr_news.nnz / (X_tr_news.shape[0] * X_tr_news.shape[1]):.4%}")
print("(matrices de texto son muy dispersas — sparse matrices son eficientes en memoria)")

# Palabras con mayor TF-IDF promedio por categoría
print("\nTop palabras por categoría:")
feature_names = tfidf_news.get_feature_names_out()
for cat_idx, cat_name in enumerate(categorias):
    mask = news_train.target == cat_idx
    media_tfidf = X_tr_news[mask].toarray().mean(axis=0)
    top_idx = media_tfidf.argsort()[::-1][:8]
    print(f"  {cat_name}: {', '.join(feature_names[top_idx])}")
```

---

## N-gramas

```python
# ── Unigramas vs Bigramas vs Trigramas ───────────────────────────────────────
texto_ng = "machine learning is not magic but it can feel magical"

for ngram_range, nombre in [((1,1), 'Unigrama'), ((2,2), 'Bigrama'), ((3,3), 'Trigrama')]:
    cv_ng = CountVectorizer(ngram_range=ngram_range)
    cv_ng.fit([texto_ng])
    print(f"\n{nombre}: {list(cv_ng.vocabulary_.keys())[:10]}")

# ── Efecto de los n-gramas en clasificación ───────────────────────────────────
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, f1_score
from sklearn.pipeline import Pipeline

print("\n─── Efecto de n-gramas en 20 Newsgroups ───")
for ngram, nombre in [((1,1),'Unigrama'), ((1,2),'Uni+Bigrama'), ((1,3),'Uni+Bi+Trigrama')]:
    pipe_ng = Pipeline([
        ('tfidf', TfidfVectorizer(max_features=15000, stop_words='english',
                                   ngram_range=ngram, sublinear_tf=True)),
        ('clf',   LogisticRegression(C=5, max_iter=1000, random_state=42, n_jobs=-1))
    ])
    pipe_ng.fit(news_train.data, news_train.target)
    acc = accuracy_score(news_test.target, pipe_ng.predict(news_test.data))
    f1  = f1_score(news_test.target, pipe_ng.predict(news_test.data), average='macro')
    print(f"  {nombre:<20}: Acc={acc:.4f} | F1={f1:.4f}")
```

---

## Análisis de sentimiento

```python
# ── Dataset de sentimiento: críticas de películas ─────────────────────────────
# Generamos reviews sintéticas con sentimiento conocido
reseñas_positivas = [
    "This movie is absolutely fantastic! Great performances by all actors.",
    "Wonderful film! I loved every minute. Highly recommended.",
    "A masterpiece of cinema. The storyline is captivating and emotional.",
    "Best movie I've seen this year. Amazing cinematography and music.",
    "Brilliant acting and stunning visuals. Will watch again for sure!",
    "The plot was engaging and the characters were well-developed. Loved it.",
    "Outstanding film with exceptional dialogue. Truly a gem.",
    "Superb storytelling. The director did an incredible job.",
]
reseñas_negativas = [
    "Awful movie, total waste of time. The plot makes no sense.",
    "Terrible acting and boring storyline. I fell asleep halfway through.",
    "Worst movie I've seen. Poor production quality and horrible script.",
    "Extremely disappointing. The trailer was much better than the film.",
    "Bad direction, bad acting, bad everything. Avoid this film.",
    "Completely nonsensical plot. Couldn't follow what was happening.",
    "Dreadful experience. The characters were flat and unconvincing.",
    "Boring and predictable. A waste of good talent.",
]
reseñas_neutras = [
    "The movie was okay, nothing special but not terrible either.",
    "Average film. Some parts were good, others felt slow.",
    "It had its moments but overall it was just mediocre.",
    "Decent enough for a weekend watch but won't stick with you.",
]

textos_sent = reseñas_positivas + reseñas_negativas + reseñas_neutras * 2
etiquetas_sent = ([1] * len(reseñas_positivas) +
                  [0] * len(reseñas_negativas) +
                  [2] * len(reseñas_neutras) * 2)  # 0=neg, 1=pos, 2=neutro

print(f"\n─── Dataset de Sentimiento ───")
print(f"Total reviews: {len(textos_sent)} | Positivas={etiquetas_sent.count(1)}, "
      f"Negativas={etiquetas_sent.count(0)}, Neutras={etiquetas_sent.count(2)}")

# Ampliamos con más datos para demostración
np.random.seed(42)
from sklearn.model_selection import train_test_split

X_sent, y_sent = textos_sent, etiquetas_sent
X_tr_sent, X_te_sent, y_tr_sent, y_te_sent = train_test_split(
    X_sent, y_sent, test_size=0.3, stratify=y_sent, random_state=42
)

# Pipeline de clasificación de sentimiento
pipe_sent = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=5000, ngram_range=(1,2),
                               stop_words='english', sublinear_tf=True)),
    ('clf',   LogisticRegression(C=1, max_iter=1000, multi_class='multinomial',
                                  random_state=42))
])
pipe_sent.fit(X_tr_sent, y_tr_sent)
y_pred_sent = pipe_sent.predict(X_te_sent)

from sklearn.metrics import classification_report
clases = ['Negativo', 'Positivo', 'Neutro']
print(f"\nClasificación de sentimiento:")
print(classification_report(y_te_sent, y_pred_sent,
                             target_names=clases, zero_division=0))

# Predicciones con probabilidades
nuevas_reviews = [
    "This was absolutely terrible! Never watching again.",
    "I really enjoyed this movie. Fantastic experience!",
    "It was fine, nothing memorable.",
]
proba_sent = pipe_sent.predict_proba(nuevas_reviews)
for review, proba in zip(nuevas_reviews, proba_sent):
    pred = clases[np.argmax(proba)]
    print(f"\n  '{review[:50]}...'")
    print(f"  → {pred} (neg={proba[0]:.2f}, pos={proba[1]:.2f}, neutro={proba[2]:.2f})")

# ── TextBlob: análisis de sentimiento basado en lexicón ───────────────────────
try:
    from textblob import TextBlob

    print("\n─── TextBlob (lexicón, sin entrenamiento) ───")
    for txt in nuevas_reviews:
        blob = TextBlob(txt)
        print(f"  '{txt[:50]}...'")
        print(f"  → Polaridad={blob.sentiment.polarity:.2f}, "
              f"Subjetividad={blob.sentiment.subjectivity:.2f}")
    print("\nPolaridad ∈ [-1, 1]: -1=muy negativo, 0=neutro, 1=muy positivo")
    print("Subjetividad ∈ [0, 1]: 0=objetivo, 1=muy subjetivo")
except ImportError:
    print("(textblob no instalado — pip install textblob)")
```

---

## Clasificación de texto

```python
# ── Comparativa de clasificadores para texto ──────────────────────────────────
from sklearn.naive_bayes import MultinomialNB, ComplementNB
from sklearn.svm import LinearSVC
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import SGDClassifier
from sklearn.model_selection import StratifiedKFold, cross_validate

print("\n─── Comparativa de clasificadores de texto (20 Newsgroups) ───\n")

clasificadores_texto = {
    'MultinomialNB':      MultinomialNB(alpha=0.1),
    'ComplementNB':       ComplementNB(alpha=0.1),
    'LinearSVC':          LinearSVC(C=1, max_iter=2000, random_state=42),
    'LogReg (L2)':        LogisticRegression(C=5, max_iter=1000, n_jobs=-1, random_state=42),
    'SGD (hinge)':        SGDClassifier(loss='hinge', alpha=1e-4, max_iter=100,
                                         random_state=42, n_jobs=-1),
    'SGD (log_loss)':     SGDClassifier(loss='log_loss', alpha=1e-4, max_iter=100,
                                         random_state=42, n_jobs=-1),
}

# Vectorización fija para comparar solo el clasificador
X_news_full = tfidf_news.fit_transform(news_train.data)
y_news_full  = news_train.target

cv_strat = StratifiedKFold(5, shuffle=True, random_state=42)
resultados_texto = {}

for nombre, clf in clasificadores_texto.items():
    scores = cross_validate(clf, X_news_full, y_news_full, cv=cv_strat,
                             scoring=['accuracy', 'f1_macro'], n_jobs=-1)
    resultados_texto[nombre] = {
        'Acc':    f"{scores['test_accuracy'].mean():.4f} ± {scores['test_accuracy'].std():.4f}",
        'F1':     f"{scores['test_f1_macro'].mean():.4f} ± {scores['test_f1_macro'].std():.4f}",
    }
    print(f"  {nombre:<20} Acc={scores['test_accuracy'].mean():.4f} "
          f"| F1={scores['test_f1_macro'].mean():.4f}")

print("""
Recomendaciones:
  • MultinomialNB / ComplementNB: muy rápido, excelente baseline, bueno con desbalance
  • LinearSVC:  mejor clasificador de texto en datasets medianos
  • LogReg L2:  similar a LinearSVC, da probabilidades
  • SGD:        para datasets enormes (online learning)
  • RF/GB:      generalmente peores en texto (no aprovechan la dispersión)
""")
```

---

## Word Embeddings

```
Bag of Words: "gato" y "felino" son palabras DIFERENTES (sin relación)
Word2Vec:     "gato" y "felino" están CERCA en el espacio vectorial

Word Embeddings codifican el SIGNIFICADO semántico en vectores densos (50-300 dims).

Word2Vec (Mikolov 2013):
  Skip-Gram: predice el contexto dado el word central
  CBOW:      predice el word central dado el contexto

Propiedades:
  vec("king") - vec("man") + vec("woman") ≈ vec("queen")
  vec("Paris") - vec("France") + vec("Italy") ≈ vec("Rome")
```

```python
# ── Word2Vec con Gensim ───────────────────────────────────────────────────────
try:
    from gensim.models import Word2Vec
    from gensim.utils import simple_preprocess

    # Preprocesar corpus
    corpus_w2v = [simple_preprocess(doc) for doc in news_train.data]

    # Entrenar Word2Vec
    w2v_model = Word2Vec(
        sentences=corpus_w2v,
        vector_size=100,   # dimensión de los embeddings
        window=5,          # contexto: 5 palabras a cada lado
        min_count=5,       # ignorar palabras con frecuencia < 5
        workers=4,
        sg=1,              # 1=Skip-Gram, 0=CBOW
        epochs=10,
        seed=42
    )

    vocab_size = len(w2v_model.wv)
    print(f"\n─── Word2Vec (20 Newsgroups) ───")
    print(f"Vocabulario: {vocab_size:,} palabras")
    print(f"Dimensión embeddings: {w2v_model.vector_size}")

    # Palabras más similares
    for word in ['space', 'doctor', 'hockey', 'gun']:
        if word in w2v_model.wv:
            similares = w2v_model.wv.most_similar(word, topn=5)
            sim_str = ', '.join([f"{w}({s:.2f})" for w, s in similares])
            print(f"\n  Similares a '{word}': {sim_str}")

    # ── Promedio de embeddings como representación de documento ───────────────
    def doc_to_vec(doc, model, size=100):
        """Representa un documento como el promedio de sus word embeddings."""
        tokens = simple_preprocess(doc)
        vecs   = [model.wv[w] for w in tokens if w in model.wv]
        return np.mean(vecs, axis=0) if vecs else np.zeros(size)

    # Representar el corpus
    X_w2v = np.array([doc_to_vec(doc, w2v_model) for doc in news_train.data])
    X_w2v_te = np.array([doc_to_vec(doc, w2v_model) for doc in news_test.data])

    print(f"\nRepresentación documental con W2V: {X_w2v.shape}")

    # Clasificación con embeddings de W2V
    from sklearn.preprocessing import StandardScaler
    from sklearn.linear_model import LogisticRegression

    sc = StandardScaler()
    X_w2v_sc = sc.fit_transform(X_w2v)
    X_w2v_te_sc = sc.transform(X_w2v_te)

    clf_w2v = LogisticRegression(C=5, max_iter=1000, random_state=42, n_jobs=-1)
    clf_w2v.fit(X_w2v_sc, news_train.target)
    acc_w2v = accuracy_score(news_test.target, clf_w2v.predict(X_w2v_te_sc))
    print(f"Clasificación con W2V embeddings: Acc={acc_w2v:.4f}")

    # Comparativa vs TF-IDF
    pipe_tfidf = Pipeline([
        ('tfidf', TfidfVectorizer(max_features=10000, sublinear_tf=True,
                                   stop_words='english', ngram_range=(1,2))),
        ('clf',   LogisticRegression(C=5, max_iter=1000, n_jobs=-1))
    ])
    pipe_tfidf.fit(news_train.data, news_train.target)
    acc_tfidf = accuracy_score(news_test.target, pipe_tfidf.predict(news_test.data))

    print(f"\nComparativa:")
    print(f"  TF-IDF + LogReg: Acc={acc_tfidf:.4f}  ← generalmente gana en texto corto")
    print(f"  W2V + LogReg:    Acc={acc_w2v:.4f}")
    print("  W2V brilla con texto corto (tweets) o cuando hay poca data de entrenamiento")

    # Visualización de embeddings con PCA
    from sklearn.decomposition import PCA

    palabras_interesantes = ['space', 'nasa', 'orbit', 'doctor', 'patient',
                              'hockey', 'gun', 'weapon', 'science', 'research']
    palabras_disponibles  = [p for p in palabras_interesantes if p in w2v_model.wv]

    if len(palabras_disponibles) >= 5:
        vecs_palabras = np.array([w2v_model.wv[p] for p in palabras_disponibles])
        pca_w2v = PCA(n_components=2, random_state=42)
        vecs_2d = pca_w2v.fit_transform(vecs_palabras)

        fig, ax = plt.subplots(figsize=(8, 6))
        ax.scatter(vecs_2d[:, 0], vecs_2d[:, 1], s=50)
        for i, palabra in enumerate(palabras_disponibles):
            ax.annotate(palabra, (vecs_2d[i, 0], vecs_2d[i, 1]),
                        fontsize=11, textcoords='offset points', xytext=(5, 5))
        ax.set_title('Embeddings Word2Vec en 2D (PCA)')
        ax.grid(alpha=0.3)
        plt.tight_layout()
        plt.savefig('outputs/m12_word2vec.png', dpi=120)
        plt.show()

except ImportError:
    print("\n(gensim no instalado — pip install gensim)")
    print("Word2Vec requiere gensim. TF-IDF + sklearn es suficiente para clasificación básica.")
```

---

## Guardar y desplegar el modelo NLP

```python
# ── Guardar pipeline completo de NLP ─────────────────────────────────────────
import joblib

pipe_final = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=15000, stop_words='english',
                               ngram_range=(1,2), sublinear_tf=True, min_df=2)),
    ('clf',   LinearSVC(C=1, max_iter=2000, random_state=42))
])
pipe_final.fit(news_train.data, news_train.target)

os.makedirs('models', exist_ok=True)
joblib.dump(pipe_final, 'models/nlp_tfidf_linearsvc.pkl')
print("Modelo guardado en models/nlp_tfidf_linearsvc.pkl")

# Uso del modelo cargado
modelo_cargado = joblib.load('models/nlp_tfidf_linearsvc.pkl')
nuevos_textos = [
    "The astronauts completed their spacewalk successfully",
    "The patient was diagnosed with pneumonia after blood tests",
    "He scored the winning goal in overtime",
]
predicciones = modelo_cargado.predict(nuevos_textos)
for txt, pred in zip(nuevos_textos, predicciones):
    print(f"  '{txt[:50]}...' → {news_train.target_names[pred]}")

# Verificación de rendimiento
acc_final = accuracy_score(news_test.target, modelo_cargado.predict(news_test.data))
print(f"\nTest Accuracy del modelo final: {acc_final:.4f}")
```

---

## Resumen del módulo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESUMEN M12 — NLP Clásico                                │
├──────────────────┬──────────────────────────────────────────────────────────┤
│ Técnica          │ Descripción                                               │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Tokenización     │ Divide texto en tokens (palabras, subpalabras)            │
│ Stopwords        │ Elimina palabras sin valor informativo (el, es, la, ...)  │
│ Stemming         │ Corta morfemas (running → run) — rápido, no siempre real │
│ Lemmatización    │ Forma canónica (better → good) — más preciso, más lento  │
│ BoW              │ Vector de conteos — simple, ignora orden                  │
│ TF-IDF           │ Penaliza palabras comunes — mejor que BoW                │
│ N-gramas         │ Captura contexto local (n=2: bigramas)                   │
│ Word2Vec         │ Embeddings densos con significado semántico               │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Mejor clf texto  │ LinearSVC > ComplementNB > LogReg >> RF                  │
│ Parámetros clave │ max_features, ngram_range, min_df, sublinear_tf          │
└──────────────────┴──────────────────────────────────────────────────────────┘

Para NLP moderno (LLMs, transformers) → ver Tutorial 3
```

---

## Checkpoint M12

```python
# checkpoint_m12.py
"""
Checkpoint M12 — NLP Clásico
Ejecutar: python checkpoint_m12.py
"""
import numpy as np
import warnings
warnings.filterwarnings('ignore')
import nltk
nltk.download(['punkt', 'stopwords', 'wordnet', 'punkt_tab'], quiet=True)

from sklearn.datasets import fetch_20newsgroups
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer
from sklearn.pipeline import Pipeline
from sklearn.svm import LinearSVC
from sklearn.naive_bayes import ComplementNB
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, f1_score
from sklearn.model_selection import StratifiedKFold, cross_val_score
import joblib, os

print("=" * 60)
print("CHECKPOINT M12 — NLP Clásico")
print("=" * 60)

# 1. Datos
cats = ['sci.space', 'rec.sport.hockey']
train = fetch_20newsgroups(subset='train', categories=cats,
                            remove=('headers', 'footers', 'quotes'))
test  = fetch_20newsgroups(subset='test', categories=cats,
                            remove=('headers', 'footers', 'quotes'))
print(f"\n✓ Dataset: {len(train.data)} train | {len(test.data)} test | {len(cats)} clases")

# 2. TF-IDF
tfidf = TfidfVectorizer(max_features=5000, stop_words='english', sublinear_tf=True)
X_tr = tfidf.fit_transform(train.data)
X_te = tfidf.transform(test.data)
print(f"✓ TF-IDF: {X_tr.shape} | densidad={X_tr.nnz/(X_tr.shape[0]*X_tr.shape[1]):.4%}")
assert X_tr.shape[1] <= 5000

# 3. Pipeline LinearSVC
pipe = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=5000, stop_words='english',
                               ngram_range=(1,2), sublinear_tf=True)),
    ('clf',   LinearSVC(C=1, max_iter=2000, random_state=42))
])
pipe.fit(train.data, train.target)
acc  = accuracy_score(test.target, pipe.predict(test.data))
f1   = f1_score(test.target, pipe.predict(test.data), average='macro')
print(f"✓ LinearSVC: Acc={acc:.4f} | F1={f1:.4f}")
assert acc > 0.93, f"Accuracy baja: {acc:.4f}"

# 4. ComplementNB
pipe_nb = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=5000, stop_words='english',
                               sublinear_tf=True)),
    ('clf',   ComplementNB(alpha=0.1))
])
pipe_nb.fit(train.data, train.target)
acc_nb = accuracy_score(test.target, pipe_nb.predict(test.data))
print(f"✓ ComplementNB: Acc={acc_nb:.4f}")
assert acc_nb > 0.90

# 5. Predicción nueva
nuevos = ["Houston, we have a liftoff! The shuttle is heading to orbit.",
          "The player scored a hat-trick in the NHL playoffs."]
preds = pipe.predict(nuevos)
for txt, pred in zip(nuevos, preds):
    print(f"✓ '{txt[:40]}...' → {train.target_names[pred]}")
assert train.target_names[preds[0]] == 'sci.space'
assert train.target_names[preds[1]] == 'rec.sport.hockey'

# 6. Serialización
os.makedirs('models', exist_ok=True)
joblib.dump(pipe, 'models/checkpoint_m12_nlp.pkl')
loaded = joblib.load('models/checkpoint_m12_nlp.pkl')
assert np.array_equal(loaded.predict(test.data), pipe.predict(test.data))
print("✓ Serialización del pipeline NLP: OK")

print("\n" + "=" * 60)
print("CHECKPOINT M12 COMPLETADO ✓")
print("Conceptos verificados:")
print("  • TF-IDF vectorización con parámetros clave")
print("  • LinearSVC para clasificación de texto")
print("  • ComplementNB baseline")
print("  • Pipeline sklearn completo para NLP")
print("  • Predicción de nuevos textos")
print("=" * 60)
```

---

*Siguiente módulo → M13: Redes Neuronales Artificiales (ANN) con PyTorch*
