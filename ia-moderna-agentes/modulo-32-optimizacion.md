# Módulo 32 — Optimización: Quantización, Pruning, Destilación

## Encabezado

- **Objetivo**: Implementar técnicas de optimización de modelos LLM: quantización (FP32 a INT4), pruning por magnitud y estructurado, destilación de conocimiento con KL divergence, y comparativa de técnicas.
- **Herramientas**: Python stdlib (math, json, random, hashlib, dataclasses, typing, pathlib, collections)
- **Prerequisito**: Módulos 6–10 (arquitectura LLM, finetuning, LoRA), Módulo 31

---

## Sección 1: Quantización (FP32 → FP16 → INT8 → INT4)

La quantización reduce la precisión numérica de los pesos del modelo para disminuir el consumo de memoria y acelerar la inferencia, con una pequeña pérdida de calidad.

```python
import math
import json
import random
import hashlib
from dataclasses import dataclass, field
from typing import List, Dict, Tuple, Optional
from pathlib import Path


# ── Representacion de tensores simulados ──────────────────────────────────────
@dataclass
class Tensor:
    """Tensor simulado con valores float como lista de floats."""
    nombre: str
    valores: List[float]
    dtype: str = "float32"

    @property
    def n_elementos(self) -> int:
        return len(self.valores)

    @property
    def bytes_total(self) -> int:
        bits_por_tipo = {
            "float32": 32, "fp32": 32,
            "float16": 16, "fp16": 16,
            "int8": 8, "uint8": 8,
            "int4": 4, "uint4": 4,
            "int2": 2,
        }
        bits = bits_por_tipo.get(self.dtype, 32)
        return (self.n_elementos * bits + 7) // 8

    @property
    def mb(self) -> float:
        return round(self.bytes_total / 1024 / 1024, 6)

    def media(self) -> float:
        return sum(self.valores) / len(self.valores) if self.valores else 0.0

    def varianza(self) -> float:
        if not self.valores: return 0.0
        m = self.media()
        return sum((v - m)**2 for v in self.valores) / len(self.valores)

    def __repr__(self):
        return "Tensor('{}', n={}, dtype={}, mb={})".format(
            self.nombre, self.n_elementos, self.dtype, self.mb
        )


def generar_tensor(nombre: str, n: int, escala: float = 1.0, seed: int = 42) -> Tensor:
    """Genera tensor con valores normales simulados."""
    random.seed(seed)
    # Box-Muller transform para distribucion normal
    valores = []
    for i in range(n):
        u1 = max(1e-10, random.random())
        u2 = random.random()
        z = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
        valores.append(round(z * escala, 6))
    return Tensor(nombre=nombre, valores=valores, dtype="float32")


# ── Cuantizador ────────────────────────────────────────────────────────────────
class Cuantizador:
    """
    Implementa quantizacion simetrica uniforme.
    FP32 -> FP16 -> INT8 -> INT4 con calculo de error.
    """

    BITS_CONFIG = {
        "float32": {"bits": 32, "dtype_out": "float32", "speedup_est": 1.0},
        "float16": {"bits": 16, "dtype_out": "float16", "speedup_est": 1.7},
        "int8":    {"bits": 8,  "dtype_out": "int8",    "speedup_est": 2.5},
        "int4":    {"bits": 4,  "dtype_out": "int4",    "speedup_est": 3.8},
        "int2":    {"bits": 2,  "dtype_out": "int2",    "speedup_est": 5.5},
    }

    def cuantizar_tensor(self, tensor: Tensor, bits: int) -> Tuple[Tensor, Dict]:
        """
        Quantiza un tensor a N bits usando quantizacion simetrica.
        Retorna (tensor_cuantizado, metricas).
        """
        if bits == 32:
            return tensor, {"error_mse": 0.0, "ratio_compresion": 1.0}

        # Rango de representacion
        n_niveles = 2 ** bits
        val_max = max(abs(v) for v in tensor.valores) if tensor.valores else 1.0
        if val_max == 0:
            val_max = 1.0

        escala = (2 * val_max) / (n_niveles - 1)

        # Quantizar: mapear a enteros y de-quantizar
        valores_cuant = []
        for v in tensor.valores:
            entero = round(v / escala)
            entero = max(-(n_niveles//2), min(n_niveles//2 - 1, entero))
            valor_reconstruido = entero * escala
            valores_cuant.append(round(valor_reconstruido, 6))

        # Calcular MSE
        mse = sum((orig - cuant)**2 for orig, cuant in zip(tensor.valores, valores_cuant))
        mse /= len(tensor.valores) if tensor.valores else 1
        rmse = math.sqrt(mse)

        # Ratio de compresion
        bytes_original = tensor.bytes_total
        bytes_cuant = (len(valores_cuant) * bits + 7) // 8
        ratio = round(bytes_original / bytes_cuant, 2) if bytes_cuant > 0 else 1.0

        nombre_dtype = {32: "float32", 16: "float16", 8: "int8", 4: "int4", 2: "int2"}
        dtype_salida = nombre_dtype.get(bits, "int" + str(bits))

        tensor_cuant = Tensor(
            nombre=tensor.nombre + "_q" + str(bits),
            valores=valores_cuant,
            dtype=dtype_salida,
        )

        metricas = {
            "bits": bits,
            "dtype": dtype_salida,
            "mse": round(mse, 8),
            "rmse": round(rmse, 8),
            "error_relativo": round(rmse / (val_max + 1e-10), 6),
            "bytes_original": bytes_original,
            "bytes_cuantizado": bytes_cuant,
            "ratio_compresion": ratio,
            "n_niveles": n_niveles,
            "escala": round(escala, 8),
        }

        return tensor_cuant, metricas

    def comparativa_completa(self, tensor: Tensor) -> Dict:
        """Genera tabla comparativa FP32 vs FP16 vs INT8 vs INT4."""
        tabla = []
        config_bits = [32, 16, 8, 4]

        for bits in config_bits:
            tensor_q, metricas = self.cuantizar_tensor(tensor, bits)
            nombre_dtype = {32: "float32", 16: "float16", 8: "int8", 4: "int4"}
            cfg = self.BITS_CONFIG.get(nombre_dtype[bits], {})

            tabla.append({
                "bits": bits,
                "dtype": nombre_dtype[bits],
                "mb": tensor_q.mb,
                "ratio_compresion": metricas["ratio_compresion"],
                "rmse": metricas["rmse"],
                "error_relativo_pct": round(metricas["error_relativo"] * 100, 4),
                "speedup_estimado": cfg.get("speedup_est", 1.0),
            })

        return {
            "tensor_original": tensor.nombre,
            "n_parametros": tensor.n_elementos,
            "tabla": tabla,
        }

    def imprimir_tabla(self, comparativa: Dict):
        print("\n{:=<72}".format(""))
        print("Comparativa de Quantizacion: " + comparativa["tensor_original"])
        print("N parametros: {:,}".format(comparativa["n_parametros"]))
        print("{:=<72}".format(""))
        print("{:<10} {:<10} {:<10} {:<12} {:<15} {:<10}".format(
            "Bits", "Dtype", "MB", "Compresion", "Error Rel (%)", "Speedup"
        ))
        print("{:-<72}".format(""))
        for fila in comparativa["tabla"]:
            print("{:<10} {:<10} {:<10} {:<12} {:<15} {:<10}".format(
                str(fila["bits"]),
                fila["dtype"],
                str(fila["mb"]),
                str(fila["ratio_compresion"]) + "x",
                str(fila["error_relativo_pct"]) + "%",
                str(fila["speedup_estimado"]) + "x",
            ))
        print("{:=<72}".format(""))


# ── Demo Sección 1 ─────────────────────────────────────────────────────────────
cuantizador = Cuantizador()

# Simular capa de un LLM (e.g., attention weight matrix 768x768 = 589824 params)
tensor_capa = generar_tensor("attention.weight", n=10000, escala=0.02, seed=42)

print("=== Quantizacion ===\n")
print("Tensor original:", tensor_capa)
print()

comparativa = cuantizador.comparativa_completa(tensor_capa)
cuantizador.imprimir_tabla(comparativa)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m32_quantizacion.json", "w", encoding="utf-8") as f:
    json.dump(comparativa, f, ensure_ascii=False, indent=2)
print("\nGuardado en outputs/m32_quantizacion.json")
```

---

## Sección 2: Pruning (Poda de Pesos)

El pruning elimina pesos poco importantes del modelo, reduciendo el tamaño y la cantidad de operaciones, especialmente útil cuando combinado con cuantización.

```python
import math
import json
import random
from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass, field
from pathlib import Path


def generar_pesos(n: int, seed: int = 0) -> List[float]:
    """Genera lista de pesos con distribucion normal."""
    random.seed(seed)
    pesos = []
    for i in range(n):
        u1 = max(1e-10, random.random())
        u2 = random.random()
        z = math.sqrt(-2 * math.log(u1)) * math.cos(2 * math.pi * u2)
        pesos.append(round(z * 0.02, 6))
    return pesos


# ── Magnitude Pruning ──────────────────────────────────────────────────────────
def magnitude_pruning(pesos: List[float], sparsity: float) -> Tuple[List[float], Dict]:
    """
    Poda por magnitud: pone a cero los pesos con menor valor absoluto.
    sparsity: fraccion de pesos a podar (0.0 = sin poda, 1.0 = todo cero).
    """
    assert 0.0 <= sparsity <= 1.0, "Sparsity debe estar en [0, 1]"

    n = len(pesos)
    n_podar = int(n * sparsity)

    if n_podar == 0:
        return pesos[:], {"sparsity_real": 0.0, "pesos_podados": 0, "pesos_restantes": n}

    # Ordenar por magnitud ascendente
    indices_por_magnitud = sorted(range(n), key=lambda i: abs(pesos[i]))
    indices_podar = set(indices_por_magnitud[:n_podar])

    pesos_podados = [
        0.0 if i in indices_podar else pesos[i]
        for i in range(n)
    ]

    # Umbral aplicado
    umbral = abs(pesos[indices_por_magnitud[n_podar - 1]]) if n_podar > 0 else 0.0

    # Calcular densidad y compresion estimada
    n_nonzero = sum(1 for p in pesos_podados if p != 0.0)
    sparsity_real = round(1 - n_nonzero / n, 4)

    # MSE entre original y podado
    mse = sum((o - p)**2 for o, p in zip(pesos, pesos_podados)) / n

    return pesos_podados, {
        "sparsity_objetivo": round(sparsity, 4),
        "sparsity_real": sparsity_real,
        "pesos_podados": n_podar,
        "pesos_restantes": n - n_podar,
        "n_total": n,
        "umbral_magnitud": round(umbral, 8),
        "mse": round(mse, 10),
        "rmse": round(math.sqrt(mse), 8),
        "ratio_compresion_estimado": round(n / max(n_nonzero, 1), 2),
    }


# ── Structured Pruning ─────────────────────────────────────────────────────────
def structured_pruning(matriz: List[List[float]], ratio: float, modo: str = "filas") -> Tuple[List[List[float]], Dict]:
    """
    Poda estructurada: elimina filas o columnas enteras con menor norma L2.
    modo: "filas" | "columnas"
    ratio: fraccion de filas/columnas a eliminar
    """
    assert 0.0 <= ratio < 1.0, "Ratio debe estar en [0, 1)"
    assert modo in ("filas", "columnas"), "Modo debe ser 'filas' o 'columnas'"

    n_filas = len(matriz)
    n_cols = len(matriz[0]) if n_filas > 0 else 0

    if modo == "filas":
        # Calcular norma L2 de cada fila
        normas = [math.sqrt(sum(v**2 for v in fila)) for fila in matriz]
        n_eliminar = int(n_filas * ratio)
        indices_ord = sorted(range(n_filas), key=lambda i: normas[i])
        indices_eliminar = set(indices_ord[:n_eliminar])

        matriz_podada = [fila for i, fila in enumerate(matriz) if i not in indices_eliminar]
        eliminados = n_eliminar
        total_original = n_filas
        total_nuevo = n_filas - n_eliminar
        dim_desc = "{} filas -> {} filas".format(n_filas, n_filas - n_eliminar)
    else:
        # Calcular norma L2 de cada columna
        normas = [
            math.sqrt(sum(fila[j]**2 for fila in matriz))
            for j in range(n_cols)
        ]
        n_eliminar = int(n_cols * ratio)
        indices_ord = sorted(range(n_cols), key=lambda j: normas[j])
        indices_eliminar = set(indices_ord[:n_eliminar])

        matriz_podada = [
            [v for j, v in enumerate(fila) if j not in indices_eliminar]
            for fila in matriz
        ]
        eliminados = n_eliminar
        total_original = n_cols
        total_nuevo = n_cols - n_eliminar
        dim_desc = "{} cols -> {} cols".format(n_cols, n_cols - n_eliminar)

    params_originales = n_filas * n_cols
    params_nuevo = len(matriz_podada) * (len(matriz_podada[0]) if matriz_podada else 0)
    ratio_compresion = round(params_originales / max(params_nuevo, 1), 2)

    return matriz_podada, {
        "modo": modo,
        "ratio": ratio,
        "eliminados": eliminados,
        "dimension": dim_desc,
        "params_originales": params_originales,
        "params_resultado": params_nuevo,
        "ratio_compresion": ratio_compresion,
    }


# ── Pruning Schedule ───────────────────────────────────────────────────────────
@dataclass
class PruningSchedule:
    """
    Programa de sparsidad gradual: comienza en 0% y llega a sparsity_target%.
    Implementa el schedule de Zhu & Gupta (2017): cubica desde inicio hasta fin.
    """
    sparsity_inicial: float = 0.0
    sparsity_target: float = 0.8
    n_pasos_inicio: int = 0
    n_pasos_total: int = 10
    frecuencia: int = 1           # Cada cuantos pasos aplicar poda

    def sparsity_en_paso(self, paso: int) -> float:
        """
        Calcula sparsity objetivo en un paso dado.
        Sigue schedule cubico: s(t) = s_f + (s_i - s_f)(1 - (t-t0)/(T-t0))^3
        """
        if paso < self.n_pasos_inicio:
            return self.sparsity_inicial

        t = paso - self.n_pasos_inicio
        T = self.n_pasos_total - self.n_pasos_inicio

        if T <= 0 or t >= T:
            return self.sparsity_target

        fraccion = t / T
        s = self.sparsity_target + (self.sparsity_inicial - self.sparsity_target) * (1 - fraccion) ** 3
        return round(s, 4)

    def debe_podar_en_paso(self, paso: int) -> bool:
        return paso >= self.n_pasos_inicio and (paso - self.n_pasos_inicio) % self.frecuencia == 0

    def secuencia_completa(self) -> List[Dict]:
        """Retorna la secuencia de sparsidades para todos los pasos."""
        return [
            {
                "paso": p,
                "sparsity": self.sparsity_en_paso(p),
                "activo": self.debe_podar_en_paso(p),
            }
            for p in range(self.n_pasos_total + 1)
        ]


# ── Demo Sección 2 ─────────────────────────────────────────────────────────────
print("=== Pruning ===\n")

pesos_orig = generar_pesos(1000, seed=10)

print("--- Magnitude Pruning ---")
for sparsity in [0.0, 0.3, 0.5, 0.7, 0.9]:
    pesos_p, metricas = magnitude_pruning(pesos_orig, sparsity)
    print("  Sparsity {:.0%}: {} pesos eliminados, RMSE={}, compresion={}x".format(
        sparsity, metricas["pesos_podados"], metricas["rmse"], metricas["ratio_compresion_estimado"]
    ))

print("\n--- Structured Pruning ---")
# Matriz 10x8
random.seed(20)
matriz = [[random.gauss(0, 0.1) for _ in range(8)] for _ in range(10)]
matriz_p, metricas_s = structured_pruning(matriz, ratio=0.3, modo="filas")
print("  Poda filas: " + metricas_s["dimension"])
print("  Params: {} -> {} (compresion: {}x)".format(
    metricas_s["params_originales"], metricas_s["params_resultado"], metricas_s["ratio_compresion"]
))

print("\n--- Pruning Schedule ---")
schedule = PruningSchedule(sparsity_target=0.7, n_pasos_total=10, frecuencia=2)
secuencia = schedule.secuencia_completa()
print("  Paso | Sparsity | Activo")
for item in secuencia:
    activo_txt = "[PODA]" if item["activo"] else "      "
    print("  {:4d} | {:.4f}   | {}".format(item["paso"], item["sparsity"], activo_txt))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m32_pruning.json", "w", encoding="utf-8") as f:
    json.dump({
        "magnitude_pruning_demo": {"sparsidades": [0.0, 0.3, 0.5, 0.7, 0.9]},
        "structured_pruning_demo": metricas_s,
        "schedule": secuencia,
    }, f, ensure_ascii=False, indent=2)
print("\nGuardado en outputs/m32_pruning.json")
```

---

## Sección 3: Destilación de Conocimiento

La destilación (Knowledge Distillation) entrena un modelo pequeño (student) para imitar las distribuciones de probabilidad de un modelo grande (teacher), transfiriendo más información que solo los labels.

```python
import math
import json
import random
from typing import List, Dict, Tuple
from dataclasses import dataclass, field
from pathlib import Path


# ── Funciones de activacion y softmax ─────────────────────────────────────────
def softmax(logits: List[float]) -> List[float]:
    """Softmax numericamente estable."""
    max_val = max(logits)
    exp_vals = [math.exp(v - max_val) for v in logits]
    suma = sum(exp_vals)
    return [v / suma for v in exp_vals]


def softmax_temperatura(logits: List[float], temperatura: float = 1.0) -> List[float]:
    """Softmax con temperatura para suavizar/agudizar distribuciones."""
    assert temperatura > 0, "Temperatura debe ser > 0"
    logits_temp = [v / temperatura for v in logits]
    return softmax(logits_temp)


def kl_divergencia(p: List[float], q: List[float]) -> float:
    """
    KL divergencia KL(P || Q) = sum(P * log(P / Q)).
    Medida de cuanto difiere Q de P.
    P = distribucion de referencia (teacher suavizado).
    Q = distribucion del estudiante.
    """
    eps = 1e-10
    return sum(
        pi * math.log((pi + eps) / (qi + eps))
        for pi, qi in zip(p, q)
    )


def destilacion_loss(
    logits_teacher: List[float],
    logits_student: List[float],
    labels_verdaderos: List[float],
    temperatura: float = 3.0,
    alpha: float = 0.5,
) -> Dict:
    """
    Calcula la perdida de destilacion combinada:
    L = alpha * L_KL(soft targets) + (1-alpha) * L_CE(hard labels)

    - L_KL: KL divergencia entre distribuciones suavizadas por temperatura
    - L_CE: Cross-entropy con labels verdaderos (hard targets)
    - alpha: balance entre soft y hard loss
    - temperatura^2 escala el gradiente del termino soft (Hinton et al. 2015)
    """
    # Soft targets (con temperatura)
    soft_teacher = softmax_temperatura(logits_teacher, temperatura)
    soft_student = softmax_temperatura(logits_student, temperatura)

    # KL divergencia (loss soft)
    kl = kl_divergencia(soft_teacher, soft_student)
    loss_soft = (temperatura ** 2) * kl

    # Cross-entropy con hard labels
    prob_student = softmax(logits_student)
    eps = 1e-10
    loss_hard = -sum(
        lv * math.log(ps + eps)
        for lv, ps in zip(labels_verdaderos, prob_student)
    )

    # Loss combinada
    loss_total = alpha * loss_soft + (1 - alpha) * loss_hard

    return {
        "loss_total": round(loss_total, 6),
        "loss_soft_kl": round(loss_soft, 6),
        "loss_hard_ce": round(loss_hard, 6),
        "kl_divergencia": round(kl, 6),
        "temperatura": temperatura,
        "alpha": alpha,
    }


# ── Teacher y Student simulados ────────────────────────────────────────────────
class TeacherModel:
    """
    Modelo grande y preciso (e.g., 70B).
    Genera logits con distribuciones bien calibradas.
    """

    def __init__(self, n_clases: int, precision: float = 0.92, seed: int = 1):
        self.n_clases = n_clases
        self.precision = precision
        self.seed = seed

    def predecir_logits(self, entrada_id: int) -> List[float]:
        """Genera logits del teacher para un ejemplo."""
        random.seed(self.seed + entrada_id * 13)
        # El teacher tiene logits mas concentrados (mas confiado y preciso)
        logits = [random.gauss(0, 0.5) for _ in range(self.n_clases)]
        # Clase "correcta" tiene logit alto
        clase_correcta = entrada_id % self.n_clases
        logits[clase_correcta] += random.gauss(3.0, 0.3)
        return [round(v, 4) for v in logits]

    def obtener_label(self, entrada_id: int) -> int:
        return entrada_id % self.n_clases


class StudentModel:
    """
    Modelo pequenio y rapido (e.g., 7B).
    Comienza con logits ruidosos y mejora via destilacion.
    """

    def __init__(self, n_clases: int, seed: int = 99):
        self.n_clases = n_clases
        self.seed = seed
        self.pesos_simulados = [random.gauss(0, 0.1) for _ in range(n_clases * 16)]
        self.paso_entrenamiento = 0

    def predecir_logits(self, entrada_id: int) -> List[float]:
        """Genera logits del student (menos precisos inicialmente)."""
        random.seed(self.seed + entrada_id * 7 + self.paso_entrenamiento)
        # Al inicio: muy ruidoso. Con entrenamiento: mejora gradualmente.
        ruido_base = max(0.1, 2.0 - self.paso_entrenamiento * 0.08)
        logits = [random.gauss(0, ruido_base) for _ in range(self.n_clases)]
        # Gradualmente aprende la clase correcta
        clase_correcta = entrada_id % self.n_clases
        boost = min(2.5, self.paso_entrenamiento * 0.12)
        logits[clase_correcta] += boost
        return [round(v, 4) for v in logits]

    def actualizar(self, loss: float, lr: float = 0.01):
        """Simula actualizacion de pesos (simplificada)."""
        self.paso_entrenamiento += 1
        # Perturbar pesos simulados en la direccion que reduce loss
        for i in range(len(self.pesos_simulados)):
            random.seed(self.seed + i + self.paso_entrenamiento)
            self.pesos_simulados[i] -= lr * loss * random.gauss(0, 0.01)


def ciclo_destilacion(
    teacher: TeacherModel,
    student: StudentModel,
    n_ejemplos: int = 20,
    n_epocas: int = 15,
    temperatura: float = 4.0,
    alpha: float = 0.7,
    lr: float = 0.05,
) -> List[Dict]:
    """
    Ciclo completo de entrenamiento por destilacion.
    Retorna historial de perdidas por epoca.
    """
    n_clases = teacher.n_clases
    historial = []

    for epoca in range(n_epocas):
        losses_epoca = []

        for ej_id in range(n_ejemplos):
            # Obtener predicciones de teacher y student
            logits_t = teacher.predecir_logits(ej_id)
            logits_s = student.predecir_logits(ej_id)

            # Construir one-hot del label verdadero
            label_idx = teacher.obtener_label(ej_id)
            labels = [0.0] * n_clases
            labels[label_idx] = 1.0

            # Calcular loss de destilacion
            resultado = destilacion_loss(logits_t, logits_s, labels, temperatura, alpha)
            loss = resultado["loss_total"]
            losses_epoca.append(loss)

            # Actualizar student
            student.actualizar(loss, lr)

        loss_prom = round(sum(losses_epoca) / len(losses_epoca), 6)
        historial.append({
            "epoca": epoca + 1,
            "loss_promedio": loss_prom,
            "loss_min": round(min(losses_epoca), 6),
            "loss_max": round(max(losses_epoca), 6),
        })

        if (epoca + 1) % 5 == 0:
            print("  Epoca {}/{}: loss={:.4f}".format(epoca + 1, n_epocas, loss_prom))

    return historial


# ── Demo Sección 3 ─────────────────────────────────────────────────────────────
print("=== Destilacion de Conocimiento ===\n")

N_CLASES = 10

teacher = TeacherModel(n_clases=N_CLASES, precision=0.92)
student = StudentModel(n_clases=N_CLASES)

print("Teacher (modelo grande):", N_CLASES, "clases, precision=0.92")
print("Student (modelo pequenio): comenzando sin conocimiento")
print("\nIniciando ciclo de destilacion (T=4.0, alpha=0.7)...")

historial = ciclo_destilacion(teacher, student, n_ejemplos=20, n_epocas=15, temperatura=4.0)

# Verificar que la perdida decrece
primer_loss = historial[0]["loss_promedio"]
ultimo_loss = historial[-1]["loss_promedio"]
print("\nLoss inicial: {:.4f}".format(primer_loss))
print("Loss final:   {:.4f}".format(ultimo_loss))
print("Mejora:       {:.1f}%".format((1 - ultimo_loss/primer_loss) * 100))

# Demo de KL divergencia directa
logits_t = teacher.predecir_logits(0)
logits_s_inicial = StudentModel(N_CLASES, seed=50).predecir_logits(0)
logits_s_final = student.predecir_logits(0)

kl_inicial = kl_divergencia(softmax_temperatura(logits_t, 4.0), softmax_temperatura(logits_s_inicial, 4.0))
kl_final = kl_divergencia(softmax_temperatura(logits_t, 4.0), softmax_temperatura(logits_s_final, 4.0))
print("\nKL divergencia inicial: {:.4f}".format(kl_inicial))
print("KL divergencia final:   {:.4f}".format(kl_final))

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m32_destilacion.json", "w", encoding="utf-8") as f:
    json.dump({"historial": historial, "kl_inicial": round(kl_inicial, 4), "kl_final": round(kl_final, 4)}, f, indent=2)
print("Guardado en outputs/m32_destilacion.json")
```

---

## Sección 4: Comparativa de Técnicas y Recomendación

Una tabla consolidada permite seleccionar la técnica adecuada según las restricciones de VRAM, velocidad y calidad aceptable.

```python
import json
from typing import Dict, List
from pathlib import Path


# ── Tabla comparativa ──────────────────────────────────────────────────────────
TECNICAS_OPTIMIZACION = [
    {
        "tecnica": "FP32 (baseline)",
        "categoria": "quantizacion",
        "reduccion_tamano_pct": 0,
        "speedup": 1.0,
        "calidad_relativa": 1.00,
        "vram_relativa": 1.00,
        "caso_de_uso": "Entrenamiento / investigacion",
        "pros": "Maxima precision numerica",
        "contras": "Mayor VRAM, mas lento",
        "requisito_hardware": "GPU con suficiente VRAM",
    },
    {
        "tecnica": "FP16 / BF16",
        "categoria": "quantizacion",
        "reduccion_tamano_pct": 50,
        "speedup": 1.7,
        "calidad_relativa": 0.99,
        "vram_relativa": 0.50,
        "caso_de_uso": "Entrenamiento mixto / inferencia GPU",
        "pros": "Casi identica precision, 2x menos VRAM",
        "contras": "Requiere GPU compatible con tensor cores",
        "requisito_hardware": "GPU Ampere+ (RTX 30xx, A100)",
    },
    {
        "tecnica": "INT8 (GPTQ/bitsandbytes)",
        "categoria": "quantizacion",
        "reduccion_tamano_pct": 75,
        "speedup": 2.5,
        "calidad_relativa": 0.97,
        "vram_relativa": 0.25,
        "caso_de_uso": "Inferencia GPU con VRAM limitada",
        "pros": "Buen balance calidad/VRAM",
        "contras": "Pequenia degradacion en tareas exigentes",
        "requisito_hardware": "GPU 16GB+",
    },
    {
        "tecnica": "INT4 (GGUF Q4_K_M)",
        "categoria": "quantizacion",
        "reduccion_tamano_pct": 87,
        "speedup": 3.8,
        "calidad_relativa": 0.93,
        "vram_relativa": 0.13,
        "caso_de_uso": "CPU / GPU consumidor / edge",
        "pros": "Ejecutable en CPU, VRAM minima",
        "contras": "Degradacion notable en precision numerica",
        "requisito_hardware": "CPU moderna / GPU 8GB",
    },
    {
        "tecnica": "Magnitude Pruning 50%",
        "categoria": "pruning",
        "reduccion_tamano_pct": 50,
        "speedup": 1.5,
        "calidad_relativa": 0.95,
        "vram_relativa": 0.60,
        "caso_de_uso": "Reduccion de pesos con sparse matrices",
        "pros": "Complementario con quantizacion",
        "contras": "Requiere hardware con soporte sparse",
        "requisito_hardware": "GPU con sparse tensor support",
    },
    {
        "tecnica": "Structured Pruning 30%",
        "categoria": "pruning",
        "reduccion_tamano_pct": 30,
        "speedup": 1.4,
        "calidad_relativa": 0.92,
        "vram_relativa": 0.70,
        "caso_de_uso": "Reduccion de arquitectura permanente",
        "pros": "Compatible con cualquier hardware",
        "contras": "Perdida de calidad mas visible",
        "requisito_hardware": "Cualquier GPU",
    },
    {
        "tecnica": "Destilacion 70B->7B",
        "categoria": "destilacion",
        "reduccion_tamano_pct": 90,
        "speedup": 10.0,
        "calidad_relativa": 0.88,
        "vram_relativa": 0.10,
        "caso_de_uso": "Produccion a escala, bajo costo",
        "pros": "Gran reduccion de costo operativo",
        "contras": "Requiere entrenamiento adicional costoso",
        "requisito_hardware": "Cluster GPU para entrenamiento",
    },
    {
        "tecnica": "Destilacion + INT4",
        "categoria": "combinado",
        "reduccion_tamano_pct": 97,
        "speedup": 15.0,
        "calidad_relativa": 0.82,
        "vram_relativa": 0.03,
        "caso_de_uso": "Edge / mobile / CPU constrained",
        "pros": "Maxima compresion, deployable en edge",
        "contras": "Proceso complejo, calidad reducida",
        "requisito_hardware": "CPU o GPU 4GB",
    },
]


def imprimir_comparativa(tecnicas: List[Dict]):
    """Imprime tabla comparativa de tecnicas."""
    print("\n" + "="*90)
    print("COMPARATIVA DE TECNICAS DE OPTIMIZACION")
    print("="*90)
    print("{:<28} {:<14} {:<10} {:<8} {:<12} {:<10}".format(
        "Tecnica", "Categoria", "Reduc.(%)", "Speedup", "Calidad", "VRAM rel."
    ))
    print("-"*90)

    for t in tecnicas:
        print("{:<28} {:<14} {:<10} {:<8} {:<12} {:<10}".format(
            t["tecnica"][:27],
            t["categoria"],
            str(t["reduccion_tamano_pct"]) + "%",
            str(t["speedup"]) + "x",
            str(round(t["calidad_relativa"]*100, 0)) + "%",
            str(round(t["vram_relativa"]*100, 0)) + "%",
        ))

    print("="*90)


def recomendar_tecnica(restricciones: Dict) -> Dict:
    """
    Recomienda la mejor tecnica segun restricciones del usuario.

    Restricciones posibles:
    - vram_gb_disponible: VRAM disponible en GB (ej: 8, 16, 24, 80)
    - calidad_minima: fraccion de calidad aceptable (0.8, 0.9, 0.95)
    - prioridad: "calidad" | "velocidad" | "memoria"
    - puede_reentrenar: bool (si se puede hacer destilacion)
    """
    vram_disponible = restricciones.get("vram_gb_disponible", 8)
    calidad_min = restricciones.get("calidad_minima", 0.85)
    prioridad = restricciones.get("prioridad", "memoria")
    puede_reentrenar = restricciones.get("puede_reentrenar", False)

    # Tamano base del modelo en GB (asumimos 7B en FP32 = ~28GB)
    tamano_base_gb = 28.0

    candidatas = []
    for t in TECNICAS_OPTIMIZACION:
        # Filtrar por calidad minima
        if t["calidad_relativa"] < calidad_min:
            continue

        # Filtrar por destilacion si no puede reentrenar
        if "destil" in t["tecnica"].lower() and not puede_reentrenar:
            continue

        # Calcular VRAM estimada necesaria
        vram_necesaria = tamano_base_gb * t["vram_relativa"]

        if vram_necesaria > vram_disponible:
            continue

        candidatas.append({
            **t,
            "vram_necesaria_gb": round(vram_necesaria, 1),
        })

    if not candidatas:
        return {
            "recomendacion": None,
            "razon": "Ninguna tecnica cumple todas las restricciones. Considera relajar calidad_minima o aumentar VRAM.",
            "restricciones": restricciones,
        }

    # Seleccionar segun prioridad
    if prioridad == "calidad":
        mejor = max(candidatas, key=lambda t: (t["calidad_relativa"], t["speedup"]))
    elif prioridad == "velocidad":
        mejor = max(candidatas, key=lambda t: (t["speedup"], t["calidad_relativa"]))
    else:  # memoria
        mejor = min(candidatas, key=lambda t: (t["vram_necesaria_gb"], -t["calidad_relativa"]))

    return {
        "recomendacion": mejor["tecnica"],
        "categoria": mejor["categoria"],
        "calidad_esperada": round(mejor["calidad_relativa"] * 100, 1),
        "speedup_esperado": mejor["speedup"],
        "vram_necesaria_gb": mejor["vram_necesaria_gb"],
        "caso_de_uso": mejor["caso_de_uso"],
        "restricciones_aplicadas": restricciones,
        "n_candidatas": len(candidatas),
        "pros": mejor["pros"],
        "contras": mejor["contras"],
    }


# ── Demo Sección 4 ─────────────────────────────────────────────────────────────
imprimir_comparativa(TECNICAS_OPTIMIZACION)

print("\n=== Recomendaciones segun restricciones ===\n")

casos = [
    {"vram_gb_disponible": 8,  "calidad_minima": 0.85, "prioridad": "memoria", "puede_reentrenar": False},
    {"vram_gb_disponible": 24, "calidad_minima": 0.95, "prioridad": "calidad", "puede_reentrenar": False},
    {"vram_gb_disponible": 80, "calidad_minima": 0.90, "prioridad": "velocidad", "puede_reentrenar": True},
    {"vram_gb_disponible": 4,  "calidad_minima": 0.80, "prioridad": "memoria", "puede_reentrenar": False},
]

recomendaciones = []
for caso in casos:
    rec = recomendar_tecnica(caso)
    recomendaciones.append(rec)
    print("VRAM: {}GB | Calidad min: {}% | Prioridad: {}".format(
        caso["vram_gb_disponible"],
        int(caso["calidad_minima"]*100),
        caso["prioridad"],
    ))
    if rec["recomendacion"]:
        print("  -> {} (calidad: {}%, speedup: {}x, VRAM: {}GB)".format(
            rec["recomendacion"], rec["calidad_esperada"],
            rec["speedup_esperado"], rec["vram_necesaria_gb"]
        ))
    else:
        print("  -> " + rec["razon"])
    print()

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m32_comparativa.json", "w", encoding="utf-8") as f:
    json.dump({
        "tabla_tecnicas": TECNICAS_OPTIMIZACION,
        "recomendaciones": recomendaciones,
    }, f, ensure_ascii=False, indent=2)
print("Guardado en outputs/m32_comparativa.json")
```

---

## Checkpoint — 6 Tests de Verificación

```python
import math
import json
import random
from typing import List, Dict, Tuple
from pathlib import Path


# ── Funciones y clases compactas para el checkpoint ───────────────────────────

def generar_pesos(n, seed=0):
    random.seed(seed)
    pesos = []
    for _ in range(n):
        u1 = max(1e-10, random.random())
        u2 = random.random()
        pesos.append(round(math.sqrt(-2*math.log(u1))*math.cos(2*math.pi*u2)*0.02, 6))
    return pesos


class Cuantizador:
    def cuantizar_tensor(self, valores, bits):
        n_niveles = 2 ** bits
        val_max = max(abs(v) for v in valores) if valores else 1.0
        if val_max == 0: val_max = 1.0
        escala = (2 * val_max) / (n_niveles - 1)
        valores_q = []
        for v in valores:
            entero = round(v / escala)
            entero = max(-(n_niveles//2), min(n_niveles//2 - 1, entero))
            valores_q.append(round(entero * escala, 6))
        mse = sum((o-q)**2 for o,q in zip(valores, valores_q)) / len(valores)
        bytes_orig = len(valores) * 4  # float32
        bytes_q = (len(valores) * bits + 7) // 8
        return valores_q, {
            "bits": bits, "mse": round(mse, 10), "rmse": round(math.sqrt(mse), 8),
            "ratio_compresion": round(bytes_orig / max(bytes_q, 1), 2),
            "bytes_original": bytes_orig, "bytes_cuantizado": bytes_q,
        }


def magnitude_pruning(pesos, sparsity):
    n = len(pesos)
    n_podar = int(n * sparsity)
    if n_podar == 0:
        return pesos[:], {"sparsity_real": 0.0, "pesos_podados": 0}
    indices_ord = sorted(range(n), key=lambda i: abs(pesos[i]))
    a_podar = set(indices_ord[:n_podar])
    resultado = [0.0 if i in a_podar else pesos[i] for i in range(n)]
    n_nonzero = sum(1 for p in resultado if p != 0.0)
    mse = sum((o-p)**2 for o,p in zip(pesos, resultado)) / n
    return resultado, {
        "sparsity_real": round(1 - n_nonzero/n, 4),
        "pesos_podados": n_podar,
        "rmse": round(math.sqrt(mse), 8),
    }


def softmax(logits):
    mx = max(logits)
    exp_v = [math.exp(v - mx) for v in logits]
    s = sum(exp_v)
    return [v/s for v in exp_v]

def softmax_temp(logits, t):
    return softmax([v/t for v in logits])

def kl_divergencia(p, q):
    eps = 1e-10
    return sum(pi * math.log((pi+eps)/(qi+eps)) for pi,qi in zip(p,q))

def destilacion_loss(logits_t, logits_s, labels, temperatura=3.0, alpha=0.5):
    soft_t = softmax_temp(logits_t, temperatura)
    soft_s = softmax_temp(logits_s, temperatura)
    kl = kl_divergencia(soft_t, soft_s)
    loss_soft = (temperatura**2) * kl
    prob_s = softmax(logits_s)
    eps = 1e-10
    loss_hard = -sum(lv * math.log(ps+eps) for lv,ps in zip(labels, prob_s))
    return {"loss_total": round(alpha*loss_soft + (1-alpha)*loss_hard, 6), "kl_divergencia": round(kl, 6)}


BITS_POR_CUANTIZACION = {"fp32": 32, "fp16": 16, "int8": 8, "q4_0": 4, "int4": 4}

def calcular_vram(n_params_b, cuantizacion="fp16"):
    bits = BITS_POR_CUANTIZACION.get(cuantizacion, 16)
    return round(n_params_b * 1e9 * bits / 8 / 1e9, 2)


TECNICAS = [
    {"tecnica": "FP32", "calidad_relativa": 1.00, "vram_relativa": 1.00, "speedup": 1.0},
    {"tecnica": "FP16", "calidad_relativa": 0.99, "vram_relativa": 0.50, "speedup": 1.7},
    {"tecnica": "INT8", "calidad_relativa": 0.97, "vram_relativa": 0.25, "speedup": 2.5},
    {"tecnica": "INT4", "calidad_relativa": 0.93, "vram_relativa": 0.13, "speedup": 3.8},
    {"tecnica": "Destilacion 70B->7B", "calidad_relativa": 0.88, "vram_relativa": 0.10, "speedup": 10.0},
]

def recomendar_tecnica(vram_gb, calidad_min, tamano_base_gb=28.0):
    candidatas = [t for t in TECNICAS
                  if t["calidad_relativa"] >= calidad_min
                  and tamano_base_gb * t["vram_relativa"] <= vram_gb]
    if not candidatas: return None
    return min(candidatas, key=lambda t: t["vram_relativa"])


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


def test_1_cuantizacion_bits():
    """Cuantizacion a menos bits debe aumentar el error MSE."""
    cuant = Cuantizador()
    vals = generar_pesos(500, seed=1)

    _, m32 = cuant.cuantizar_tensor(vals, 32)
    _, m16 = cuant.cuantizar_tensor(vals, 16)
    _, m8  = cuant.cuantizar_tensor(vals, 8)
    _, m4  = cuant.cuantizar_tensor(vals, 4)

    assert m32["mse"] <= m16["mse"], "FP32 debe tener MSE <= FP16"
    assert m16["mse"] <= m8["mse"],  "FP16 debe tener MSE <= INT8"
    assert m8["mse"]  <= m4["mse"],  "INT8 debe tener MSE <= INT4"
    assert m4["mse"]  > 0,           "INT4 debe tener MSE > 0"


def test_2_cuantizacion_compresion():
    """Ratio de compresion debe escalar con la reduccion de bits."""
    cuant = Cuantizador()
    vals = generar_pesos(1000, seed=2)

    _, m16 = cuant.cuantizar_tensor(vals, 16)
    _, m4  = cuant.cuantizar_tensor(vals, 4)

    assert m16["ratio_compresion"] >= 1.9, \
        "FP16 debe comprimir al menos 2x (float32->float16), got: " + str(m16["ratio_compresion"])
    assert m4["ratio_compresion"] >= 7.0, \
        "INT4 debe comprimir al menos 8x (float32->int4), got: " + str(m4["ratio_compresion"])
    assert m4["ratio_compresion"] > m16["ratio_compresion"], \
        "INT4 debe comprimir mas que FP16"


def test_3_pruning_sparsity():
    """Pruning al 70% debe resultar en ~70% de pesos en cero."""
    pesos = generar_pesos(1000, seed=3)
    pesos_p, metricas = magnitude_pruning(pesos, sparsity=0.7)

    assert abs(metricas["sparsity_real"] - 0.7) < 0.02, \
        "Sparsity real debe ser ~0.70, got: " + str(metricas["sparsity_real"])
    assert metricas["pesos_podados"] == 700, \
        "Deben podarse 700 pesos, got: " + str(metricas["pesos_podados"])

    n_ceros = sum(1 for p in pesos_p if p == 0.0)
    assert n_ceros == 700, "Deben haber 700 ceros en el resultado, got: " + str(n_ceros)


def test_4_pruning_sin_poda():
    """Pruning con sparsity=0 no debe modificar pesos."""
    pesos = generar_pesos(200, seed=4)
    pesos_p, metricas = magnitude_pruning(pesos, sparsity=0.0)

    assert metricas["pesos_podados"] == 0, "Con sparsity=0 no debe haber poda"
    assert metricas["sparsity_real"] == 0.0, "Sparsity real debe ser 0"
    assert pesos_p == pesos[:], "Pesos no deben modificarse con sparsity=0"


def test_5_destilacion_loss():
    """Loss de destilacion debe ser positiva y KL divergencia correcta."""
    random.seed(5)
    n_clases = 5
    logits_t = [2.0, 0.5, -0.5, 1.0, -1.0]   # Teacher: clase 0 preferida
    logits_s = [0.1, 0.2, 0.1, 0.05, 0.0]      # Student: distribucion plana
    labels = [1.0, 0.0, 0.0, 0.0, 0.0]          # Label verdadero: clase 0

    resultado = destilacion_loss(logits_t, logits_s, labels, temperatura=3.0, alpha=0.5)

    assert resultado["loss_total"] > 0, "Loss debe ser positiva, got: " + str(resultado["loss_total"])
    assert resultado["kl_divergencia"] >= 0, "KL divergencia debe ser >= 0"

    # KL debe ser menor cuando student imita al teacher
    logits_s_bueno = logits_t[:]   # Student = Teacher
    resultado_bueno = destilacion_loss(logits_t, logits_s_bueno, labels)
    assert resultado_bueno["kl_divergencia"] < resultado["kl_divergencia"], \
        "KL debe ser 0 cuando student = teacher"


def test_6_recomendacion_tecnica():
    """Sistema de recomendacion debe seleccionar tecnica valida segun restricciones."""
    # Con poca VRAM y calidad media -> debe recomendar algo comprimido
    rec = recomendar_tecnica(vram_gb=4.0, calidad_min=0.85)
    assert rec is not None, "Debe haber recomendacion para 4GB y calidad 0.85"
    assert rec["calidad_relativa"] >= 0.85, "Calidad debe cumplir minimo"
    vram_necesaria = 28.0 * rec["vram_relativa"]
    assert vram_necesaria <= 4.0, "VRAM de la recomendacion debe caber en 4GB"

    # Con mucha VRAM y alta calidad -> puede ser FP32 o FP16
    rec_alto = recomendar_tecnica(vram_gb=80.0, calidad_min=0.98)
    assert rec_alto is not None, "Debe haber recomendacion para 80GB y calidad 0.98"
    assert rec_alto["calidad_relativa"] >= 0.98, "Calidad minima 0.98 debe cumplirse"

    # Con VRAM imposible y calidad muy alta -> sin candidata
    rec_imposible = recomendar_tecnica(vram_gb=0.1, calidad_min=0.99)
    assert rec_imposible is None, "Sin candidatas para VRAM=0.1GB y calidad 0.99"


# ── Ejecutar todos los tests ───────────────────────────────────────────────────
print("\n" + "="*60)
print("CHECKPOINT M32 — Optimizacion")
print("="*60)

run_test("T1: Cuantizacion menor bits = mayor error MSE", test_1_cuantizacion_bits)
run_test("T2: Ratio compresion escala con reduccion de bits", test_2_cuantizacion_compresion)
run_test("T3: Pruning 70% resulta en 70% pesos en cero", test_3_pruning_sparsity)
run_test("T4: Pruning sparsity=0 no modifica pesos", test_4_pruning_sin_poda)
run_test("T5: Destilacion loss positiva y KL correcta", test_5_destilacion_loss)
run_test("T6: Recomendacion tecnica respeta restricciones VRAM", test_6_recomendacion_tecnica)

total = len(tests_resultados)
pasados = sum(1 for _, s, _ in tests_resultados if s == "PASS")
print("\n" + "="*60)
print("Resultado: {}/{} tests pasados".format(pasados, total))
if pasados == total:
    print("TODOS LOS TESTS PASARON - Modulo 32 completado")
else:
    for nombre, estado, msg in tests_resultados:
        if estado != "PASS":
            print("  FALLO: " + nombre + " -> " + str(msg))
print("="*60)

Path("outputs").mkdir(exist_ok=True)
with open("outputs/m32_checkpoint.json", "w", encoding="utf-8") as f:
    json.dump({
        "modulo": "M32",
        "total": total,
        "pasados": pasados,
        "tests": [{"nombre": n, "estado": s, "mensaje": m} for n, s, m in tests_resultados],
    }, f, ensure_ascii=False, indent=2)
```
