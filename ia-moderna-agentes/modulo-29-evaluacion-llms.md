# Módulo 29 — Evaluación de LLMs: Benchmarks, RAGAS, LLM-as-Judge

## Encabezado

- **Objetivo**: Implementar métricas de evaluación de LLMs (BLEU, ROUGE, BERTScore simulado), frameworks RAGAS para RAG, evaluación LLM-as-Judge y benchmarks categorizados.
- **Herramientas**: Python stdlib (math, re, json, collections, hashlib, typing, dataclasses, pathlib, random)
- **Prerequisito**: Módulos 13–15 (RAG básico y avanzado), Módulo 28

---

## Sección 1: Métricas de Evaluación (BLEU, ROUGE, BERTScore simulado)

Las métricas automáticas permiten evaluar a escala sin intervención humana. BLEU mide precisión de n-gramas, ROUGE mide recall, BERTScore usa similitud semántica.

```python
import math
import re
import json
import hashlib
from collections import Counter
from typing import List, Dict, Tuple, Optional
from pathlib import Path


# ── Tokenizador simple ─────────────────────────────────────────────────────────
def tokenizar(texto: str) -> List[str]:
    """Tokeniza texto en palabras, normalizando a minúsculas."""
    texto = texto.lower()
    tokens = re.findall(r"\b\w+\b", texto)
    return tokens


def obtener_ngramas(tokens: List[str], n: int) -> List[Tuple]:
    """Genera n-gramas de una lista de tokens."""
    return [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]


# ── BLEU Score ─────────────────────────────────────────────────────────────────
def bleu_score(referencia: str, hipotesis: str, max_n: int = 4) -> Dict:
    """
    Calcula BLEU score con n-gramas del 1 al max_n.
    Incluye brevity penalty (BP) para penalizar respuestas muy cortas.
    """
    ref_tokens = tokenizar(referencia)
    hip_tokens = tokenizar(hipotesis)

    if len(hip_tokens) == 0:
        return {"bleu": 0.0, "precisions": [], "bp": 0.0}

    precisions = []
    for n in range(1, max_n + 1):
        ref_ngramas = Counter(obtener_ngramas(ref_tokens, n))
        hip_ngramas = Counter(obtener_ngramas(hip_tokens, n))

        if not hip_ngramas:
            precisions.append(0.0)
            continue

        # Precision con clipping
        matches = sum(
            min(cnt, ref_ngramas[ngram])
            for ngram, cnt in hip_ngramas.items()
        )
        precision = matches / sum(hip_ngramas.values())
        precisions.append(precision)

    # Brevity Penalty
    r = len(ref_tokens)
    c = len(hip_tokens)
    bp = 1.0 if c >= r else math.exp(1 - r / c)

    # BLEU como media geometrica de precisiones
    validas = [p for p in precisions if p > 0]
    if not validas:
        bleu = 0.0
    else:
        log_avg = sum(math.log(p) for p in validas) / len(validas)
        bleu = bp * math.exp(log_avg)

    return {
        "bleu": round(bleu, 4),
        "precisions": [round(p, 4) for p in precisions],
        "bp": round(bp, 4),
        "len_referencia": r,
        "len_hipotesis": c,
    }


# ── ROUGE-L ────────────────────────────────────────────────────────────────────
def _lcs_length(a: List, b: List) -> int:
    """Calcula longitud de la subsecuencia comun mas larga (LCS)."""
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]


def rouge_l(referencia: str, hipotesis: str) -> Dict:
    """
    Calcula ROUGE-L basado en la subsecuencia comun mas larga.
    Retorna precision, recall y F1.
    """
    ref_tokens = tokenizar(referencia)
    hip_tokens = tokenizar(hipotesis)

    if not ref_tokens or not hip_tokens:
        return {"rouge_l_f1": 0.0, "precision": 0.0, "recall": 0.0}

    lcs = _lcs_length(ref_tokens, hip_tokens)
    precision = lcs / len(hip_tokens) if hip_tokens else 0.0
    recall = lcs / len(ref_tokens) if ref_tokens else 0.0

    if precision + recall > 0:
        f1 = 2 * precision * recall / (precision + recall)
    else:
        f1 = 0.0

    return {
        "rouge_l_f1": round(f1, 4),
        "precision": round(precision, 4),
        "recall": round(recall, 4),
        "lcs_length": lcs,
    }


def rouge_1(referencia: str, hipotesis: str) -> Dict:
    """ROUGE-1: overlap de unigramas."""
    ref_tokens = tokenizar(referencia)
    hip_tokens = tokenizar(hipotesis)
    ref_cnt = Counter(ref_tokens)
    hip_cnt = Counter(hip_tokens)

    matches = sum(min(cnt, ref_cnt[tok]) for tok, cnt in hip_cnt.items())
    precision = matches / len(hip_tokens) if hip_tokens else 0.0
    recall = matches / len(ref_tokens) if ref_tokens else 0.0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0.0

    return {
        "rouge_1_f1": round(f1, 4),
        "precision": round(precision, 4),
        "recall": round(recall, 4),
    }


# ── BERTScore Simulado ─────────────────────────────────────────────────────────
def bertscore_simulado(texto_a: str, texto_b: str) -> Dict:
    """
    BERTScore simulado usando similitud de tokens con peso por frecuencia inversa.
    En produccion usaria embeddings BERT reales.
    """
    tokens_a = set(tokenizar(texto_a))
    tokens_b = set(tokenizar(texto_b))

    if not tokens_a or not tokens_b:
        return {"bertscore_f1": 0.0, "precision": 0.0, "recall": 0.0}

    # Similitud Jaccard base
    interseccion = tokens_a & tokens_b
    union = tokens_a | tokens_b
    jaccard = len(interseccion) / len(union) if union else 0.0

    # Ajuste por longitud (penalizar diferencias grandes)
    len_ratio = min(len(tokens_a), len(tokens_b)) / max(len(tokens_a), len(tokens_b))

    # Hash deterministico como componente "semantico"
    combined = texto_a[:50] + texto_b[:50]
    h = int(hashlib.md5(combined.encode()).hexdigest(), 16)
    bonus_semantico = (h % 200) / 2000.0  # 0.0 - 0.1

    f1 = round(min(1.0, jaccard * 0.7 + len_ratio * 0.2 + bonus_semantico), 4)
    precision = round(len(interseccion) / len(tokens_b), 4) if tokens_b else 0.0
    recall = round(len(interseccion) / len(tokens_a), 4) if tokens_a else 0.0

    return {
        "bertscore_f1": f1,
        "precision": precision,
        "recall": recall,
        "similitud_jaccard": round(jaccard, 4),
    }


# ── Demo Sección 1 ─────────────────────────────────────────────────────────────
referencia = "El aprendizaje automatico es una rama de la inteligencia artificial que permite a las maquinas aprender de datos."
hipotesis_buena = "El machine learning es una parte de la IA que permite que las computadoras aprendan de datos."
hipotesis_mala = "El futbol es un deporte popular en muchos paises del mundo."

print("=== Métricas de Evaluación ===\n")
print("Referencia:", referencia[:60] + "...")
print()

for nombre, hip in [("Hipotesis buena", hipotesis_buena), ("Hipotesis mala", hipotesis_mala)]:
    b = bleu_score(referencia, hip)
    r = rouge_l(referencia, hip)
    bs = bertscore_simulado(referencia, hip)
    print(nombre + ":")
    print("  BLEU:      {:.4f}".format(b["bleu"]))
    print("  ROUGE-L:   {:.4f}".format(r["rouge_l_f1"]))
    print("  BERTScore: {:.4f}".format(bs["bertscore_f1"]))
    print()
```

---

## Sección 2: RAGAS para RAG

RAGAS (Retrieval Augmented Generation Assessment) evalúa sistemas RAG con métricas específicas: faithfulness (la respuesta está sustentada en el contexto), answer relevance (relevancia respecto a la pregunta), context precision y context recall.

```python
import math
import re
import json
import hashlib
from collections import Counter
from typing import List, Dict
from pathlib import Path


def tokenizar(texto: str) -> List[str]:
    return re.findall(r"\b\w+\b", texto.lower())


def jaccard(set_a: set, set_b: set) -> float:
    if not set_a and not set_b:
        return 1.0
    union = set_a | set_b
    return len(set_a & set_b) / len(union) if union else 0.0


class RAGASEvaluador:
    """
    Implementa las 4 metricas principales de RAGAS:
    1. Faithfulness: respuesta apoyada en contexto
    2. Answer Relevance: respuesta relevante a la pregunta
    3. Context Precision: contexto relevante para responder
    4. Context Recall: contexto cubre la respuesta de referencia
    """

    def __init__(self):
        pass

    def faithfulness(self, respuesta: str, contexto: str) -> float:
        """
        Mide si los hechos de la respuesta estan en el contexto.
        Aproximacion: overlap de tokens significativos.
        """
        tokens_resp = set(tokenizar(respuesta))
        tokens_ctx = set(tokenizar(contexto))

        # Filtrar stopwords basicas
        stopwords = {"el", "la", "los", "las", "un", "una", "de", "en", "y", "a", "que", "es"}
        tokens_resp -= stopwords
        tokens_ctx -= stopwords

        if not tokens_resp:
            return 1.0

        overlap = len(tokens_resp & tokens_ctx) / len(tokens_resp)
        return round(min(1.0, overlap * 1.2), 4)  # leve boost

    def answer_relevance(self, pregunta: str, respuesta: str) -> float:
        """
        Mide si la respuesta es relevante a la pregunta.
        Usa similitud Jaccard entre tokens de pregunta y respuesta.
        """
        tokens_q = set(tokenizar(pregunta))
        tokens_r = set(tokenizar(respuesta))
        stopwords = {"el", "la", "los", "las", "un", "una", "de", "en", "y", "a", "que", "es",
                     "como", "cual", "cuando", "donde", "quien"}
        tokens_q -= stopwords
        tokens_r -= stopwords

        if not tokens_q:
            return 0.5
        overlap = len(tokens_q & tokens_r) / len(tokens_q)
        return round(min(1.0, overlap * 1.5), 4)

    def context_precision(self, pregunta: str, contexto: str) -> float:
        """
        Mide que proporcion del contexto es relevante para la pregunta.
        """
        tokens_q = set(tokenizar(pregunta))
        tokens_c = set(tokenizar(contexto))
        stopwords = {"el", "la", "los", "las", "de", "en", "y", "a", "que"}
        tokens_q -= stopwords
        tokens_c -= stopwords

        if not tokens_c:
            return 0.0
        relevantes = len(tokens_q & tokens_c)
        return round(min(1.0, relevantes / len(tokens_q)), 4) if tokens_q else 0.0

    def context_recall(self, referencia: str, contexto: str) -> float:
        """
        Mide que proporcion de la respuesta de referencia esta cubierta por el contexto.
        """
        tokens_ref = set(tokenizar(referencia))
        tokens_ctx = set(tokenizar(contexto))
        stopwords = {"el", "la", "los", "las", "de", "en", "y", "a", "que", "es", "son"}
        tokens_ref -= stopwords
        tokens_ctx -= stopwords

        if not tokens_ref:
            return 1.0
        cobertura = len(tokens_ref & tokens_ctx) / len(tokens_ref)
        return round(min(1.0, cobertura), 4)

    def score_global(self, f: float, ar: float, cp: float, cr: float) -> float:
        """Score RAGAS global como media armonica de las 4 metricas."""
        valores = [f, ar, cp, cr]
        if any(v == 0 for v in valores):
            return 0.0
        return round(4 / sum(1/v for v in valores), 4)

    def evaluar(self, pregunta: str, respuesta: str, contexto: str, referencia: str) -> Dict:
        """Evalua un unico ejemplo RAG."""
        f = self.faithfulness(respuesta, contexto)
        ar = self.answer_relevance(pregunta, respuesta)
        cp = self.context_precision(pregunta, contexto)
        cr = self.context_recall(referencia, contexto)
        sg = self.score_global(f, ar, cp, cr)

        return {
            "faithfulness": f,
            "answer_relevance": ar,
            "context_precision": cp,
            "context_recall": cr,
            "ragas_score": sg,
            "calificacion": "EXCELENTE" if sg >= 0.8 else "BUENO" if sg >= 0.6 else "REGULAR" if sg >= 0.4 else "MALO",
        }

    def evaluar_batch(
        self,
        preguntas: List[str],
        respuestas: List[str],
        contextos: List[str],
        referencias: List[str],
    ) -> Dict:
        """Evalua un batch de ejemplos RAG."""
        assert len(preguntas) == len(respuestas) == len(contextos) == len(referencias), \
            "Todas las listas deben tener la misma longitud"

        resultados = []
        for i, (q, r, c, ref) in enumerate(zip(preguntas, respuestas, contextos, referencias)):
            res = self.evaluar(q, r, c, ref)
            res["indice"] = i
            res["pregunta"] = q[:60]
            resultados.append(res)

        # Promedios
        metricas = ["faithfulness", "answer_relevance", "context_precision", "context_recall", "ragas_score"]
        promedios = {
            m: round(sum(r[m] for r in resultados) / len(resultados), 4)
            for m in metricas
        }

        return {
            "n_ejemplos": len(resultados),
            "resultados_individuales": resultados,
            "promedios": promedios,
            "distribucion_calificaciones": Counter(r["calificacion"] for r in resultados),
        }


# ── Demo Sección 2 ─────────────────────────────────────────────────────────────
evaluador = RAGASEvaluador()

preguntas = [
    "Que es el transformer en deep learning?",
    "Como funciona el mecanismo de atencion?",
    "Que ventajas tiene BERT sobre GPT?",
]
contextos = [
    "El transformer es una arquitectura de red neuronal basada en atencion. Fue introducido en 2017 por Vaswani et al. Usa self-attention para procesar secuencias en paralelo.",
    "El mecanismo de atencion calcula pesos de importancia entre tokens. Cada token atiende a todos los demas usando Query, Key y Value matrices.",
    "BERT es un modelo bidireccional pre-entrenado con masked language modeling. GPT es autoregresivo y unidireccional. BERT es mejor para tareas de comprension.",
]
respuestas = [
    "El transformer es una arquitectura neuronal basada en self-attention que procesa secuencias en paralelo.",
    "La atencion calcula pesos entre tokens usando matrices Query Key Value para capturar relaciones contextuales.",
    "BERT tiene ventajas en comprension de lenguaje al ser bidireccional, mientras GPT es mejor en generacion.",
]
referencias = [
    "El transformer usa atencion para procesar secuencias en paralelo sin recurrencia.",
    "El mecanismo de atencion pondera la importancia de cada token en el contexto.",
    "BERT es bidireccional y usa MLM, ideal para clasificacion y comprension.",
]

batch_result = evaluador.evaluar_batch(preguntas, respuestas, contextos, referencias)

print("=== RAGAS Evaluacion ===")
print("Ejemplos evaluados:", batch_result["n_ejemplos"])
print("\nPromedios:")
for metrica, valor in batch_result["promedios"].items():
    print("  {}: {}".format(metrica, valor))
print("\nDistribucion:", dict(batch_result["distribucion_calificaciones"]))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m29_ragas.json", "w", encoding="utf-8") as f:
    # Convertir Counter a dict para JSON
    batch_result["distribucion_calificaciones"] = dict(batch_result["distribucion_calificaciones"])
    json.dump(batch_result, f, ensure_ascii=False, indent=2)
print("\nGuardado en outputs/m29_ragas.json")
```

---

## Sección 3: LLM-as-Judge

LLM-as-Judge usa un LLM para evaluar respuestas de otro LLM, permitiendo evaluación a escala con criterios complejos imposibles de capturar con métricas automáticas.

```python
import hashlib
import json
import re
from typing import Dict, List, Tuple
from pathlib import Path


class LLMSimulado:
    """LLM determinista para simular el juez evaluador."""

    def __init__(self, nombre: str = "judge-llm"):
        self.nombre = nombre

    def _hash_score(self, texto: str, criterio: str, offset: float = 0.0) -> float:
        combinado = texto + criterio + self.nombre
        h = int(hashlib.md5(combinado.encode()).hexdigest(), 16)
        base = 0.3 + (h % 700) / 1000.0
        return round(min(1.0, max(0.0, base + offset)), 3)

    def evaluar_criterio(self, texto: str, criterio: str, contexto: str = "") -> Dict:
        score = self._hash_score(texto + contexto, criterio)
        justificaciones = {
            "precision": [
                "La respuesta contiene informacion correcta y verificable.",
                "Algunos datos podrian ser mas precisos.",
                "La precision es limitada, faltan detalles clave.",
            ],
            "claridad": [
                "La explicacion es clara y bien estructurada.",
                "La estructura es aceptable pero podria mejorar.",
                "La respuesta es confusa y dificil de seguir.",
            ],
            "utilidad": [
                "La respuesta es muy util y accionable.",
                "Tiene valor pero podria ser mas practica.",
                "No resuelve directamente el problema planteado.",
            ],
        }

        lista_just = justificaciones.get(criterio, ["Evaluacion completada."])
        idx = int(score * len(lista_just))
        justificacion = lista_just[min(idx, len(lista_just) - 1)]

        return {"score": score, "justificacion": justificacion}


class LLMJudge:
    """
    Juez LLM que compara dos respuestas en multiples criterios.
    Implementa el patron pairwise comparison de Chatbot Arena / MT-Bench.
    """

    CRITERIOS = {
        "precision": "Exactitud factual y correctitud de la informacion proporcionada.",
        "claridad": "Claridad, estructura y facilidad de comprension de la respuesta.",
        "utilidad": "Valor practico y utilidad para resolver el problema del usuario.",
    }

    PESOS = {
        "precision": 0.4,
        "claridad": 0.3,
        "utilidad": 0.3,
    }

    def __init__(self):
        self.juez = LLMSimulado(nombre="llm-judge-v1")

    def juzgar_respuesta(self, pregunta: str, respuesta: str) -> Dict:
        """Evalua una respuesta individual en todos los criterios."""
        evaluaciones = {}
        for criterio, descripcion in self.CRITERIOS.items():
            evaluaciones[criterio] = self.juez.evaluar_criterio(
                respuesta, criterio, contexto=pregunta
            )

        # Score ponderado
        score_total = sum(
            evaluaciones[c]["score"] * self.PESOS[c]
            for c in self.CRITERIOS
        )

        return {
            "evaluaciones": evaluaciones,
            "score_total": round(score_total, 3),
        }

    def juzgar_respuestas(self, pregunta: str, resp_a: str, resp_b: str) -> Dict:
        """
        Compara dos respuestas (A vs B) y determina la preferida.
        Retorna: preferencia, scores, justificacion por criterio, veredicto.
        """
        eval_a = self.juzgar_respuesta(pregunta, resp_a)
        eval_b = self.juzgar_respuesta(pregunta, resp_b)

        score_a = eval_a["score_total"]
        score_b = eval_b["score_total"]
        margen = abs(score_a - score_b)

        if margen < 0.05:
            preferencia = "EMPATE"
            veredicto = "Ambas respuestas son de calidad similar."
        elif score_a > score_b:
            preferencia = "A"
            veredicto = "Respuesta A es superior en {} de {} criterios.".format(
                sum(
                    1 for c in self.CRITERIOS
                    if eval_a["evaluaciones"][c]["score"] > eval_b["evaluaciones"][c]["score"]
                ),
                len(self.CRITERIOS),
            )
        else:
            preferencia = "B"
            veredicto = "Respuesta B es superior en {} de {} criterios.".format(
                sum(
                    1 for c in self.CRITERIOS
                    if eval_b["evaluaciones"][c]["score"] > eval_a["evaluaciones"][c]["score"]
                ),
                len(self.CRITERIOS),
            )

        comparacion_criterios = {}
        for criterio in self.CRITERIOS:
            sa = eval_a["evaluaciones"][criterio]["score"]
            sb = eval_b["evaluaciones"][criterio]["score"]
            comparacion_criterios[criterio] = {
                "score_a": sa,
                "score_b": sb,
                "mejor": "A" if sa > sb else "B" if sb > sa else "EMPATE",
                "justificacion_a": eval_a["evaluaciones"][criterio]["justificacion"],
                "justificacion_b": eval_b["evaluaciones"][criterio]["justificacion"],
            }

        return {
            "preferencia": preferencia,
            "score_a": score_a,
            "score_b": score_b,
            "margen": round(margen, 3),
            "veredicto": veredicto,
            "comparacion_por_criterio": comparacion_criterios,
            "confianza": "ALTA" if margen > 0.15 else "MEDIA" if margen > 0.05 else "BAJA",
        }

    def juzgar_batch(self, comparaciones: List[Dict]) -> Dict:
        """Evalua multiples pares de respuestas."""
        resultados = []
        victorias = {"A": 0, "B": 0, "EMPATE": 0}

        for comp in comparaciones:
            r = self.juzgar_respuestas(
                comp["pregunta"], comp["resp_a"], comp["resp_b"]
            )
            r["id"] = comp.get("id", len(resultados))
            resultados.append(r)
            victorias[r["preferencia"]] += 1

        return {
            "total_comparaciones": len(resultados),
            "victorias": victorias,
            "tasa_victoria_a": round(victorias["A"] / len(resultados), 3),
            "tasa_victoria_b": round(victorias["B"] / len(resultados), 3),
            "resultados": resultados,
        }


# ── Demo Sección 3 ─────────────────────────────────────────────────────────────
juez = LLMJudge()

pregunta = "Explica como funciona el gradient descent en el entrenamiento de redes neuronales."
resp_a = "El gradient descent es un algoritmo de optimizacion. Calcula el gradiente de la funcion de perdida y actualiza los pesos en la direccion contraria al gradiente con un learning rate. Iterativamente minimiza la perdida."
resp_b = "Es un metodo de aprendizaje."

resultado = juez.juzgar_respuestas(pregunta, resp_a, resp_b)

print("=== LLM-as-Judge ===")
print("Pregunta:", pregunta[:60] + "...")
print("\nScore A:", resultado["score_a"])
print("Score B:", resultado["score_b"])
print("Preferencia:", resultado["preferencia"])
print("Confianza:", resultado["confianza"])
print("Veredicto:", resultado["veredicto"])
print("\nPor criterio:")
for criterio, detalle in resultado["comparacion_por_criterio"].items():
    print("  {}: A={} vs B={} -> {}".format(
        criterio, detalle["score_a"], detalle["score_b"], detalle["mejor"]
    ))
```

---

## Sección 4: Benchmarks Simulados

Los benchmarks estandarizados permiten comparar modelos de forma reproducible en categorías diversas.

```python
import json
import hashlib
import random
from typing import Dict, List, Callable, Optional
from pathlib import Path
from collections import defaultdict


class LLMSimulado:
    def __init__(self, nombre: str, sesgo: float = 0.0):
        self.nombre = nombre
        self.sesgo = sesgo  # Ajuste de rendimiento del modelo

    def responder(self, pregunta: str, categoria: str) -> str:
        h = int(hashlib.md5((pregunta + self.nombre).encode()).hexdigest(), 16)
        respuestas = [
            "Respuesta correcta y detallada sobre: " + pregunta[:30],
            "La respuesta es: " + str(h % 1000),
            "Segun mi conocimiento, " + pregunta[:20] + " se explica asi.",
            "No estoy seguro de la respuesta.",
            "La solucion correcta implica varios pasos.",
        ]
        return respuestas[h % len(respuestas)]

    def puntuar_respuesta(self, pregunta: str, respuesta: str, categoria: str) -> float:
        h = int(hashlib.md5((pregunta + respuesta + categoria).encode()).hexdigest(), 16)
        base = 0.3 + (h % 700) / 1000.0
        return round(min(1.0, max(0.0, base + self.sesgo)), 3)


# ── Preguntas de benchmark por categoria ──────────────────────────────────────
PREGUNTAS_BENCHMARK = {
    "razonamiento": [
        {"id": "r1", "pregunta": "Si A implica B y B implica C, que implica A?", "referencia": "A implica C"},
        {"id": "r2", "pregunta": "Todos los gatos son mamiferos. Felix es un gato. Es Felix mamifero?", "referencia": "Si, Felix es mamifero"},
        {"id": "r3", "pregunta": "Un tren sale a 100km/h. Cuanto tarda en recorrer 250km?", "referencia": "2.5 horas"},
        {"id": "r4", "pregunta": "Ordena estos numeros: 7, 2, 9, 1, 5.", "referencia": "1, 2, 5, 7, 9"},
        {"id": "r5", "pregunta": "Si el doble de X es 14, cuanto es X?", "referencia": "7"},
    ],
    "conocimiento": [
        {"id": "c1", "pregunta": "En que anio se publico el paper 'Attention is All You Need'?", "referencia": "2017"},
        {"id": "c2", "pregunta": "Que significa GPU en computacion?", "referencia": "Graphics Processing Unit"},
        {"id": "c3", "pregunta": "Quien desarrollo el lenguaje Python?", "referencia": "Guido van Rossum"},
        {"id": "c4", "pregunta": "Cuantas capas tiene GPT-3?", "referencia": "96 capas"},
        {"id": "c5", "pregunta": "Que es el gradiente en redes neuronales?", "referencia": "Derivada parcial de la funcion de perdida"},
    ],
    "codigo": [
        {"id": "cd1", "pregunta": "Escribe una funcion Python que calcule el factorial de n.", "referencia": "def factorial(n): return 1 if n<=1 else n*factorial(n-1)"},
        {"id": "cd2", "pregunta": "Como se declara una lista vacia en Python?", "referencia": "lista = [] o lista = list()"},
        {"id": "cd3", "pregunta": "Que hace el metodo .strip() en Python?", "referencia": "Elimina espacios al inicio y final del string"},
        {"id": "cd4", "pregunta": "Como iterar sobre un diccionario en Python?", "referencia": "for key, value in dict.items():"},
        {"id": "cd5", "pregunta": "Que es un decorador en Python?", "referencia": "Funcion que modifica el comportamiento de otra funcion"},
    ],
    "matematicas": [
        {"id": "m1", "pregunta": "Calcula la derivada de x^3.", "referencia": "3x^2"},
        {"id": "m2", "pregunta": "Cual es la integral de 2x dx?", "referencia": "x^2 + C"},
        {"id": "m3", "pregunta": "Resuelve: 2x + 5 = 15.", "referencia": "x = 5"},
        {"id": "m4", "pregunta": "Que es la desviacion estandar?", "referencia": "Raiz cuadrada de la varianza"},
        {"id": "m5", "pregunta": "Calcula log2(8).", "referencia": "3"},
    ],
}


class Benchmark:
    """
    Sistema de benchmark con categorias: razonamiento, conocimiento, codigo, matematicas.
    """

    CATEGORIAS = list(PREGUNTAS_BENCHMARK.keys())
    PESOS_CATEGORIA = {
        "razonamiento": 0.3,
        "conocimiento": 0.25,
        "codigo": 0.25,
        "matematicas": 0.2,
    }

    def __init__(self, nombre: str = "BenchmarkIA-v1"):
        self.nombre = nombre
        self.preguntas = PREGUNTAS_BENCHMARK
        self.resultados_historicos: List[Dict] = []

    def ejecutar_benchmark(self, modelo: LLMSimulado) -> Dict:
        """
        Ejecuta el benchmark completo para un modelo.
        Retorna tabla de resultados por categoria y score global.
        """
        resultados_por_categoria = {}
        scores_individuales = []

        for categoria, preguntas in self.preguntas.items():
            scores_categoria = []

            for item in preguntas:
                respuesta = modelo.responder(item["pregunta"], categoria)
                score = modelo.puntuar_respuesta(
                    item["pregunta"], respuesta, categoria
                )
                scores_categoria.append({
                    "id": item["id"],
                    "score": score,
                    "respuesta": respuesta[:50] + "..." if len(respuesta) > 50 else respuesta,
                })
                scores_individuales.append(score)

            promedio_cat = round(sum(s["score"] for s in scores_categoria) / len(scores_categoria), 4)
            resultados_por_categoria[categoria] = {
                "promedio": promedio_cat,
                "min": round(min(s["score"] for s in scores_categoria), 4),
                "max": round(max(s["score"] for s in scores_categoria), 4),
                "n_preguntas": len(preguntas),
                "scores": scores_categoria,
            }

        # Score global ponderado
        score_global = round(sum(
            resultados_por_categoria[cat]["promedio"] * self.PESOS_CATEGORIA[cat]
            for cat in self.CATEGORIAS
        ), 4)

        resultado = {
            "modelo": modelo.nombre,
            "benchmark": self.nombre,
            "score_global": score_global,
            "calificacion": "A" if score_global >= 0.8 else "B" if score_global >= 0.65 else "C" if score_global >= 0.5 else "D",
            "resultados_por_categoria": resultados_por_categoria,
            "ranking_categorias": sorted(
                [(cat, resultados_por_categoria[cat]["promedio"]) for cat in self.CATEGORIAS],
                key=lambda x: -x[1],
            ),
        }

        self.resultados_historicos.append(resultado)
        return resultado

    def comparar_modelos(self, modelos: List[LLMSimulado]) -> Dict:
        """Compara multiples modelos y genera ranking."""
        resultados = [self.ejecutar_benchmark(m) for m in modelos]

        ranking = sorted(resultados, key=lambda x: -x["score_global"])

        tabla = []
        for pos, r in enumerate(ranking, 1):
            fila = {"posicion": pos, "modelo": r["modelo"], "score_global": r["score_global"], "calificacion": r["calificacion"]}
            for cat in self.CATEGORIAS:
                fila[cat] = r["resultados_por_categoria"][cat]["promedio"]
            tabla.append(fila)

        return {
            "ranking": tabla,
            "mejor_modelo": ranking[0]["modelo"] if ranking else None,
            "n_modelos": len(modelos),
        }

    def imprimir_tabla(self, resultado_comparacion: Dict):
        """Imprime tabla de resultados formateada."""
        print("\n{:=<70}".format(""))
        print("BENCHMARK: " + self.nombre)
        print("{:=<70}".format(""))
        print("{:<5} {:<20} {:<8} {:<5} {:<12} {:<12} {:<10} {:<12}".format(
            "Pos", "Modelo", "Global", "Cal", "Razonamiento", "Conocimiento", "Codigo", "Matematicas"
        ))
        print("{:-<70}".format(""))

        for fila in resultado_comparacion["ranking"]:
            print("{:<5} {:<20} {:<8} {:<5} {:<12} {:<12} {:<10} {:<12}".format(
                fila["posicion"],
                fila["modelo"][:18],
                str(fila["score_global"]),
                fila["calificacion"],
                str(fila.get("razonamiento", "-")),
                str(fila.get("conocimiento", "-")),
                str(fila.get("codigo", "-")),
                str(fila.get("matematicas", "-")),
            ))
        print("{:=<70}".format(""))
        print("Mejor modelo:", resultado_comparacion["mejor_modelo"])


# ── Demo Sección 4 ─────────────────────────────────────────────────────────────
benchmark = Benchmark()

modelos = [
    LLMSimulado("ModeloGrande-70B", sesgo=0.15),
    LLMSimulado("ModeloMedio-13B", sesgo=0.05),
    LLMSimulado("ModeloPequenio-7B", sesgo=-0.05),
    LLMSimulado("ModeloEspecializado", sesgo=0.10),
]

comparacion = benchmark.comparar_modelos(modelos)
benchmark.imprimir_tabla(comparacion)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m29_benchmark.json", "w", encoding="utf-8") as f:
    json.dump(comparacion, f, ensure_ascii=False, indent=2)
print("\nResultados guardados en outputs/m29_benchmark.json")
```

---

## Checkpoint — 6 Tests de Verificación

```python
import math
import re
import json
import hashlib
from collections import Counter
from typing import List, Dict
from pathlib import Path


# ── Re-definir funciones y clases para el checkpoint ──────────────────────────

def tokenizar(texto):
    return re.findall(r"\b\w+\b", texto.lower())

def obtener_ngramas(tokens, n):
    return [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]

def bleu_score(referencia, hipotesis, max_n=4):
    ref_tokens = tokenizar(referencia)
    hip_tokens = tokenizar(hipotesis)
    if not hip_tokens:
        return {"bleu": 0.0, "precisions": [], "bp": 0.0}
    precisions = []
    for n in range(1, max_n + 1):
        ref_ng = Counter(obtener_ngramas(ref_tokens, n))
        hip_ng = Counter(obtener_ngramas(hip_tokens, n))
        if not hip_ng:
            precisions.append(0.0)
            continue
        matches = sum(min(cnt, ref_ng[ng]) for ng, cnt in hip_ng.items())
        precisions.append(matches / sum(hip_ng.values()))
    r, c = len(ref_tokens), len(hip_tokens)
    bp = 1.0 if c >= r else math.exp(1 - r/c)
    validas = [p for p in precisions if p > 0]
    bleu = bp * math.exp(sum(math.log(p) for p in validas) / len(validas)) if validas else 0.0
    return {"bleu": round(bleu, 4), "precisions": [round(p, 4) for p in precisions], "bp": round(bp, 4)}

def _lcs_length(a, b):
    m, n = len(a), len(b)
    dp = [[0]*(n+1) for _ in range(m+1)]
    for i in range(1, m+1):
        for j in range(1, n+1):
            dp[i][j] = dp[i-1][j-1]+1 if a[i-1]==b[j-1] else max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]

def rouge_l(referencia, hipotesis):
    ref_t = tokenizar(referencia)
    hip_t = tokenizar(hipotesis)
    if not ref_t or not hip_t:
        return {"rouge_l_f1": 0.0, "precision": 0.0, "recall": 0.0}
    lcs = _lcs_length(ref_t, hip_t)
    p = lcs / len(hip_t)
    r = lcs / len(ref_t)
    f1 = 2*p*r/(p+r) if (p+r) > 0 else 0.0
    return {"rouge_l_f1": round(f1, 4), "precision": round(p, 4), "recall": round(r, 4)}

def bertscore_simulado(a, b):
    ta = set(tokenizar(a))
    tb = set(tokenizar(b))
    if not ta or not tb:
        return {"bertscore_f1": 0.0}
    inter = ta & tb
    union = ta | tb
    jaccard = len(inter)/len(union)
    lr = min(len(ta), len(tb)) / max(len(ta), len(tb))
    h = int(hashlib.md5((a[:50]+b[:50]).encode()).hexdigest(), 16)
    bonus = (h % 200) / 2000.0
    f1 = round(min(1.0, jaccard*0.7 + lr*0.2 + bonus), 4)
    return {"bertscore_f1": f1}

class RAGASEvaluador:
    def faithfulness(self, respuesta, contexto):
        tr = set(tokenizar(respuesta)) - {"el","la","de","en","y","a","que","es"}
        tc = set(tokenizar(contexto)) - {"el","la","de","en","y","a","que","es"}
        if not tr: return 1.0
        return round(min(1.0, len(tr&tc)/len(tr)*1.2), 4)
    def answer_relevance(self, pregunta, respuesta):
        tq = set(tokenizar(pregunta)) - {"como","cual","cuando","donde","que"}
        tr = set(tokenizar(respuesta))
        if not tq: return 0.5
        return round(min(1.0, len(tq&tr)/len(tq)*1.5), 4)
    def context_precision(self, pregunta, contexto):
        tq = set(tokenizar(pregunta)) - {"el","la","de","en","y"}
        tc = set(tokenizar(contexto)) - {"el","la","de","en","y"}
        if not tc or not tq: return 0.0
        return round(min(1.0, len(tq&tc)/len(tq)), 4)
    def context_recall(self, referencia, contexto):
        tr = set(tokenizar(referencia)) - {"el","la","de","en","y","que","es","son"}
        tc = set(tokenizar(contexto))
        if not tr: return 1.0
        return round(min(1.0, len(tr&tc)/len(tr)), 4)
    def evaluar(self, pregunta, respuesta, contexto, referencia):
        f = self.faithfulness(respuesta, contexto)
        ar = self.answer_relevance(pregunta, respuesta)
        cp = self.context_precision(pregunta, contexto)
        cr = self.context_recall(referencia, contexto)
        vals = [f, ar, cp, cr]
        sg = round(4/sum(1/v for v in vals), 4) if all(v>0 for v in vals) else 0.0
        return {"faithfulness": f, "answer_relevance": ar, "context_precision": cp,
                "context_recall": cr, "ragas_score": sg}

class LLMSimuladoJuez:
    def __init__(self, nombre="juez"):
        self.nombre = nombre
    def puntuar(self, texto, criterio):
        h = int(hashlib.md5((texto+criterio+self.nombre).encode()).hexdigest(), 16)
        return round(0.3 + (h % 700) / 1000.0, 3)

class LLMJudge:
    CRITERIOS = {"precision": 0.4, "claridad": 0.3, "utilidad": 0.3}
    def __init__(self):
        self.juez = LLMSimuladoJuez()
    def juzgar_respuestas(self, pregunta, resp_a, resp_b):
        sa = sum(self.juez.puntuar(resp_a+pregunta, c)*w for c,w in self.CRITERIOS.items())
        sb = sum(self.juez.puntuar(resp_b+pregunta, c)*w for c,w in self.CRITERIOS.items())
        pref = "A" if abs(sa-sb) < 0.05 and sa >= sb else ("A" if sa > sb else "B")
        return {"preferencia": pref, "score_a": round(sa,3), "score_b": round(sb,3), "margen": round(abs(sa-sb),3)}

class LLMBenchmark:
    def __init__(self, nombre="juez"):
        self.nombre = nombre
        self.sesgo = 0.0
    def puntuar(self, p, c):
        h = int(hashlib.md5((p+c+self.nombre).encode()).hexdigest(), 16)
        return round(min(1.0, max(0.0, 0.3+(h%700)/1000.0+self.sesgo)), 3)

def ejecutar_benchmark(modelo):
    categorias = ["razonamiento", "conocimiento", "codigo", "matematicas"]
    preguntas = ["p1", "p2", "p3"]
    scores = {}
    for cat in categorias:
        sc = [modelo.puntuar(p, cat) for p in preguntas]
        scores[cat] = round(sum(sc)/len(sc), 4)
    global_score = round(sum(scores.values())/len(scores), 4)
    return {"modelo": modelo.nombre, "score_global": global_score, "por_categoria": scores}


# ── TESTS ──────────────────────────────────────────────────────────────────────
tests_resultados = []

def run_test(nombre, fn):
    try:
        fn()
        tests_resultados.append((nombre, "PASS", None))
        print("[PASS] " + nombre)
    except AssertionError as e:
        tests_resultados.append((nombre, "FAIL", str(e)))
        print("[FAIL] " + nombre + " -> " + str(e))
    except Exception as e:
        tests_resultados.append((nombre, "ERROR", str(e)))
        print("[ERROR] " + nombre + " -> " + str(e))


def test_1_bleu_identico():
    """BLEU de texto identico debe ser 1.0."""
    texto = "el gato negro camina por la calle oscura"
    result = bleu_score(texto, texto)
    assert result["bleu"] >= 0.99, "BLEU identico debe ser ~1.0, got: " + str(result["bleu"])
    assert result["bp"] == 1.0, "BP debe ser 1.0 para misma longitud"


def test_2_bleu_diferente():
    """BLEU de textos completamente distintos debe ser muy bajo."""
    ref = "el aprendizaje automatico usa algoritmos de optimizacion"
    hip = "futbol deporte popular entretenimiento mundial estadio"
    result = bleu_score(ref, hip)
    assert result["bleu"] < 0.1, "BLEU textos distintos debe ser < 0.1, got: " + str(result["bleu"])


def test_3_rouge_l_parcial():
    """ROUGE-L debe ser > 0 cuando hay subsecuencia comun."""
    ref = "el modelo aprende de los datos de entrenamiento"
    hip = "el modelo aprende patrones de datos"
    result = rouge_l(ref, hip)
    assert result["rouge_l_f1"] > 0.3, "ROUGE-L con subsecuencia comun debe ser > 0.3, got: " + str(result["rouge_l_f1"])
    assert 0 <= result["rouge_l_f1"] <= 1.0, "ROUGE-L debe estar en [0,1]"


def test_4_ragas_faithfulness():
    """Faithfulness alta cuando la respuesta esta en el contexto."""
    evaluador = RAGASEvaluador()
    contexto = "Python es un lenguaje de programacion interpretado y de alto nivel creado por Guido van Rossum."
    respuesta = "Python es un lenguaje interpretado de alto nivel."
    f = evaluador.faithfulness(respuesta, contexto)
    assert f >= 0.5, "Faithfulness debe ser alta cuando respuesta esta en contexto, got: " + str(f)


def test_5_llm_judge_preferencia():
    """LLM Judge debe retornar preferencia valida."""
    juez = LLMJudge()
    pregunta = "Que es una red neuronal?"
    resp_a = "Una red neuronal es un modelo computacional inspirado en el cerebro humano con capas de neuronas artificiales."
    resp_b = "Es un algoritmo."
    resultado = juez.juzgar_respuestas(pregunta, resp_a, resp_b)
    assert resultado["preferencia"] in ["A", "B", "EMPATE"], "Preferencia debe ser A, B o EMPATE"
    assert 0 <= resultado["score_a"] <= 1.0, "Score A debe estar en [0,1]"
    assert 0 <= resultado["score_b"] <= 1.0, "Score B debe estar en [0,1]"
    assert resultado["margen"] >= 0, "Margen debe ser no negativo"


def test_6_benchmark_categorias():
    """Benchmark debe retornar scores para todas las categorias."""
    modelo = LLMBenchmark("test-model")
    resultado = ejecutar_benchmark(modelo)
    assert "score_global" in resultado, "Debe haber score_global"
    assert 0 <= resultado["score_global"] <= 1.0, "Score global debe estar en [0,1]"
    categorias_esperadas = {"razonamiento", "conocimiento", "codigo", "matematicas"}
    assert set(resultado["por_categoria"].keys()) == categorias_esperadas, \
        "Deben estar todas las categorias: " + str(set(resultado["por_categoria"].keys()))
    for cat, score in resultado["por_categoria"].items():
        assert 0 <= score <= 1.0, "Score de " + cat + " debe estar en [0,1], got: " + str(score)


# ── Ejecutar todos los tests ───────────────────────────────────────────────────
print("\n" + "="*60)
print("CHECKPOINT M29 — Evaluacion de LLMs")
print("="*60)

run_test("T1: BLEU texto identico = 1.0", test_1_bleu_identico)
run_test("T2: BLEU textos distintos < 0.1", test_2_bleu_diferente)
run_test("T3: ROUGE-L con subsecuencia comun > 0.3", test_3_rouge_l_parcial)
run_test("T4: RAGAS faithfulness alta en contexto relevante", test_4_ragas_faithfulness)
run_test("T5: LLM Judge retorna preferencia valida", test_5_llm_judge_preferencia)
run_test("T6: Benchmark cubre todas las categorias", test_6_benchmark_categorias)

total = len(tests_resultados)
pasados = sum(1 for _, s, _ in tests_resultados if s == "PASS")
print("\n" + "="*60)
print("Resultado: {}/{} tests pasados".format(pasados, total))
if pasados == total:
    print("TODOS LOS TESTS PASARON - Modulo 29 completado")
else:
    for nombre, estado, msg in tests_resultados:
        if estado != "PASS":
            print("  FALLO: " + nombre + " -> " + str(msg))
print("="*60)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m29_checkpoint.json", "w", encoding="utf-8") as f:
    json.dump({
        "modulo": "M29",
        "total": total,
        "pasados": pasados,
        "tests": [{"nombre": n, "estado": s, "mensaje": m} for n, s, m in tests_resultados],
    }, f, ensure_ascii=False, indent=2)
```
