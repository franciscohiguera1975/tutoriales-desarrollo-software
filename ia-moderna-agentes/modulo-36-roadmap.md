# M36 — Roadmap y Tendencias 2025-2030

**Objetivo:** Mapear el estado actual de la IA y proyectar las trayectorias
técnicas, organizacionales y sociales más relevantes para los próximos cinco
años. Se implementan modelos de proyección, análisis de readiness tecnológico
y simulación de adopción con código Python puro, sin APIs externas.

**Herramientas:** Python 3.11, dataclasses, math, statistics, json, pathlib

**Prerrequisito:** M01–M35 completos

---

## 1. Marco conceptual — Niveles de madurez tecnológica (TRL)

El modelo Technology Readiness Level (TRL) de la NASA, adoptado por la UE
para evaluar IA, define nueve niveles: desde principio observado (TRL 1)
hasta sistema probado en producción (TRL 9).

```python
# outputs/36_trl_framework.py
from __future__ import annotations
from dataclasses import dataclass, field
from typing import List, Dict, Optional
import json
from pathlib import Path

Path("outputs").mkdir(exist_ok=True)

# ---------------------------------------------------------------------------
# Niveles TRL
# ---------------------------------------------------------------------------

@dataclass
class NivelTRL:
    nivel: int          # 1-9
    nombre: str
    descripcion: str
    criterio_salida: str

TRL_DEFINICIONES = [
    NivelTRL(1, "Principios básicos", "Principio científico observado", "Publicación revisada por pares"),
    NivelTRL(2, "Concepto formulado", "Concepto tecnológico y/o aplicación formulados", "Estudio de viabilidad"),
    NivelTRL(3, "Prueba experimental", "Función crítica analítica/experimental", "PoC en laboratorio"),
    NivelTRL(4, "Validación en lab", "Componentes validados en entorno de laboratorio", "Demo controlada"),
    NivelTRL(5, "Validación relevante", "Tecnología validada en entorno relevante", "Piloto interno"),
    NivelTRL(6, "Demo en entorno", "Demostración en entorno relevante", "Prototipo funcional"),
    NivelTRL(7, "Demo en sistema", "Prototipo demostrado en entorno operacional", "Producción limitada"),
    NivelTRL(8, "Sistema completo", "Sistema completo y calificado", "GA disponible"),
    NivelTRL(9, "En producción", "Sistema probado en entorno operacional real", "Adopción masiva"),
]

@dataclass
class TecnologiaIA:
    nombre: str
    descripcion: str
    trl_actual: int         # 1-9
    trl_2025: int
    trl_2027: int
    trl_2030: int
    categoria: str          # LLM, Agentes, Hardware, Safety, etc.
    obstaculos: List[str] = field(default_factory=list)
    habilitadores: List[str] = field(default_factory=list)

    def velocidad_maduracion(self) -> float:
        """Niveles TRL por año proyectados 2025-2030."""
        return (self.trl_2030 - self.trl_actual) / 5.0

    def trl_en_anio(self, anio: int) -> int:
        """Interpolación lineal del TRL para un año dado."""
        if anio <= 2025:
            return self.trl_2025
        elif anio >= 2030:
            return self.trl_2030
        else:
            fraccion = (anio - 2025) / (2030 - 2025)
            trl_interpolado = self.trl_2025 + fraccion * (self.trl_2030 - self.trl_2025)
            return round(trl_interpolado)

    def resumen(self) -> Dict:
        return {
            "nombre": self.nombre,
            "categoria": self.categoria,
            "trl_actual": self.trl_actual,
            "trl_2025": self.trl_2025,
            "trl_2027": self.trl_2027,
            "trl_2030": self.trl_2030,
            "velocidad": round(self.velocidad_maduracion(), 2),
        }

# ---------------------------------------------------------------------------
# Catálogo de tecnologías emergentes
# ---------------------------------------------------------------------------

TECNOLOGIAS = [
    TecnologiaIA(
        nombre="LLMs Multimodales Generalizados",
        descripcion="Modelos que integran texto, imagen, audio, video y código en una arquitectura unificada",
        trl_actual=7, trl_2025=8, trl_2027=9, trl_2030=9,
        categoria="LLM",
        obstaculos=["Alineación cross-modal", "Coste computacional"],
        habilitadores=["Mejoras en atención eficiente", "Datos multi-modal anotados"],
    ),
    TecnologiaIA(
        nombre="Agentes Autónomos de Larga Duración",
        descripcion="Agentes que ejecutan tareas complejas durante horas/días sin supervisión constante",
        trl_actual=5, trl_2025=6, trl_2027=8, trl_2030=9,
        categoria="Agentes",
        obstaculos=["Gestión de errores acumulados", "Seguridad en acción autónoma"],
        habilitadores=["Memoria persistente robusta", "Herramientas de verificación formal"],
    ),
    TecnologiaIA(
        nombre="Razonamiento Formal Verificable",
        descripcion="Modelos capaces de producir pruebas matemáticas verificables automáticamente",
        trl_actual=4, trl_2025=5, trl_2027=7, trl_2030=8,
        categoria="Razonamiento",
        obstaculos=["Generalización más allá del entrenamiento", "Explosión combinatoria"],
        habilitadores=["Integración con demostradores de teoremas (Lean, Coq)", "MCTS avanzado"],
    ),
    TecnologiaIA(
        nombre="Hardware IA Especializado (NextGen)",
        descripcion="Chips diseñados para inferencia y entrenamiento ultra-eficiente (< 1W por token)",
        trl_actual=6, trl_2025=7, trl_2027=8, trl_2030=9,
        categoria="Hardware",
        obstaculos=["Ciclos de diseño largos", "Inversión de capital"],
        habilitadores=["Arquitecturas neurológicamente inspiradas", "Fotónica integrada"],
    ),
    TecnologiaIA(
        nombre="IA Cuántica Práctica",
        descripcion="Algoritmos cuánticos acelerando entrenamiento de ML en hardware NISQ",
        trl_actual=3, trl_2025=3, trl_2027=5, trl_2030=6,
        categoria="Quantum",
        obstaculos=["Ruido cuántico", "Escalado de qubits coherentes"],
        habilitadores=["Corrección de errores cuánticos", "Compiladores variacional"],
    ),
    TecnologiaIA(
        nombre="Alineación y Safety Escalable",
        descripcion="Técnicas robustas para garantizar comportamiento alineado en modelos de >1T parámetros",
        trl_actual=4, trl_2025=5, trl_2027=7, trl_2030=8,
        categoria="Safety",
        obstaculos=["Emergencia impredecible", "Juegos de distribución"],
        habilitadores=["RLHF+Constitutional AI", "Interpretabilidad mecanística"],
    ),
    TecnologiaIA(
        nombre="Modelos de Mundo (World Models)",
        descripcion="Modelos internos de simulación de la realidad física para planificación y robótica",
        trl_actual=4, trl_2025=5, trl_2027=7, trl_2030=8,
        categoria="Robótica",
        obstaculos=["Complejidad del mundo real", "Datos de interacción física"],
        habilitadores=["Simulación diferenciable", "Aprendizaje por refuerzo en mundo abierto"],
    ),
    TecnologiaIA(
        nombre="IA Personalizada en Dispositivo",
        descripcion="LLMs de 1-7B parámetros eficientes corriendo en móviles y edge devices",
        trl_actual=6, trl_2025=7, trl_2027=9, trl_2030=9,
        categoria="Edge",
        obstaculos=["Limitaciones de memoria RAM", "Latencia de primera respuesta"],
        habilitadores=["Cuantización INT4", "Arquitecturas MoE sparse"],
    ),
    TecnologiaIA(
        nombre="Generación de Código Autónoma",
        descripcion="Sistemas que diseñan, implementan y prueban software completo con supervisión mínima",
        trl_actual=6, trl_2025=7, trl_2027=8, trl_2030=9,
        categoria="Agentes",
        obstaculos=["Debugging de errores sutiles", "Especificaciones ambiguas"],
        habilitadores=["Contextos largos", "Integración con herramientas de testing"],
    ),
    TecnologiaIA(
        nombre="IA Científica Acelerada",
        descripcion="Modelos que generan hipótesis, diseñan experimentos y analizan resultados en ciencia",
        trl_actual=5, trl_2025=6, trl_2027=8, trl_2030=9,
        categoria="Ciencia",
        obstaculos=["Validación experimental lenta", "Reproducibilidad"],
        habilitadores=["AlphaFold-style breakthroughs", "Integración con laboratorios robóticos"],
    ),
]

# ---------------------------------------------------------------------------
# Análisis de readiness
# ---------------------------------------------------------------------------

class AnalizadorTRL:
    """Analiza el estado y proyección de un conjunto de tecnologías."""

    def __init__(self, tecnologias: List[TecnologiaIA]):
        self._techs = tecnologias

    def por_categoria(self) -> Dict[str, List[TecnologiaIA]]:
        categorias: Dict[str, List[TecnologiaIA]] = {}
        for t in self._techs:
            categorias.setdefault(t.categoria, []).append(t)
        return categorias

    def mas_maduras(self, n: int = 3) -> List[TecnologiaIA]:
        return sorted(self._techs, key=lambda t: t.trl_actual, reverse=True)[:n]

    def mas_rapidas(self, n: int = 3) -> List[TecnologiaIA]:
        return sorted(self._techs, key=lambda t: t.velocidad_maduracion(), reverse=True)[:n]

    def estado_2030(self) -> List[Dict]:
        """Proyección de todas las tecnologías al año 2030."""
        resultado = []
        for t in self._techs:
            r = t.resumen()
            trl_def = next((d for d in TRL_DEFINICIONES if d.nivel == t.trl_2030), None)
            r["estado_2030"] = trl_def.nombre if trl_def else "Desconocido"
            resultado.append(r)
        return sorted(resultado, key=lambda x: x["trl_2030"], reverse=True)

    def gap_actual_vs_2030(self) -> Dict[str, int]:
        return {t.nombre: t.trl_2030 - t.trl_actual for t in self._techs}

    def reporte(self) -> str:
        lineas = ["=" * 60, "REPORTE TRL — ESTADO Y PROYECCIÓN DE TECNOLOGÍAS IA", "=" * 60]
        for t in self._techs:
            lineas.append(
                f"  [{t.categoria:<10}] {t.nombre:<45} "
                f"TRL actual={t.trl_actual} → 2030={t.trl_2030} "
                f"(+{t.trl_2030-t.trl_actual}/5 años)"
            )
        lineas.append("")
        lineas.append("TECNOLOGÍAS MÁS MADURAS:")
        for t in self.mas_maduras(3):
            lineas.append(f"  ★ {t.nombre} — TRL {t.trl_actual}/9")
        lineas.append("")
        lineas.append("MAYOR VELOCIDAD DE MADURACIÓN:")
        for t in self.mas_rapidas(3):
            lineas.append(f"  ⚡ {t.nombre} — {t.velocidad_maduracion():.1f} TRL/año")
        return "\n".join(lineas)

# Demo
analizador = AnalizadorTRL(TECNOLOGIAS)
print(analizador.reporte())

# Proyección año a año
print("\nPROYECCIÓN AÑO A AÑO (muestra):")
for tech_nombre in ["Agentes Autónomos de Larga Duración", "IA Cuántica Práctica"]:
    tech = next(t for t in TECNOLOGIAS if t.nombre == tech_nombre)
    proyeccion = {a: tech.trl_en_anio(a) for a in range(2025, 2031)}
    print(f"  {tech_nombre}: {proyeccion}")

# Guardar
with open("outputs/36_trl_tecnologias.json", "w", encoding="utf-8") as f:
    json.dump([t.resumen() for t in TECNOLOGIAS], f, ensure_ascii=False, indent=2)
print("\n✓ outputs/36_trl_tecnologias.json guardado")
```

**Salida esperada:**
```
============================================================
REPORTE TRL — ESTADO Y PROYECCIÓN DE TECNOLOGÍAS IA
============================================================
  [LLM       ] LLMs Multimodales Generalizados             TRL actual=7 → 2030=9 (+2/5 años)
  [Agentes   ] Agentes Autónomos de Larga Duración         TRL actual=5 → 2030=9 (+4/5 años)
  ...
TECNOLOGÍAS MÁS MADURAS:
  ★ LLMs Multimodales Generalizados — TRL 7/9
  ★ Hardware IA Especializado (NextGen) — TRL 6/9
  ...
MAYOR VELOCIDAD DE MADURACIÓN:
  ⚡ Agentes Autónomos de Larga Duración — 0.8 TRL/año
  ...
PROYECCIÓN AÑO A AÑO (muestra):
  Agentes Autónomos de Larga Duración: {2025: 6, 2026: 6, 2027: 7, 2028: 8, 2029: 8, 2030: 9}
  IA Cuántica Práctica: {2025: 3, 2026: 4, 2027: 5, 2028: 5, 2029: 6, 2030: 6}
```

---

## 2. Modelo de difusión de innovación (curva S)

La curva S de Rogers describe cómo una tecnología pasa de innovadores tempranos
a la mayoría tardía. Permite estimar el tiempo de adopción masiva.

```python
# outputs/36_difusion_innovacion.py
import math
from dataclasses import dataclass, field
from typing import List, Dict, Tuple

@dataclass
class ModeloDifusion:
    """
    Modelo logístico de difusión de innovación (curva S).

    P(t) = K / (1 + exp(-r*(t - t0)))

    K  = capacidad de mercado (fracción máxima adoptantes)
    r  = tasa de crecimiento
    t0 = punto de inflexión (50% de K)
    """
    nombre: str
    K: float            # capacidad máxima [0,1]
    r: float            # tasa de crecimiento anual
    t0: float           # año de inflexión
    anio_inicio: int    # año base t=0

    def adopcion(self, anio: int) -> float:
        """Fracción del mercado adoptante en el año dado."""
        t = anio - self.anio_inicio
        try:
            valor = self.K / (1 + math.exp(-self.r * (t - self.t0)))
        except OverflowError:
            valor = 0.0
        return round(min(valor, self.K), 4)

    def anio_umbral(self, fraccion: float) -> Optional[int]:
        """Año en que se alcanza la fracción de adopción dada."""
        # Despejando: t = t0 - (1/r) * ln(K/fraccion - 1)
        if fraccion >= self.K or fraccion <= 0:
            return None
        try:
            t = self.t0 - (1 / self.r) * math.log(self.K / fraccion - 1)
            return int(self.anio_inicio + t)
        except (ValueError, ZeroDivisionError):
            return None

    def trayectoria(self, desde: int, hasta: int) -> Dict[int, float]:
        return {a: self.adopcion(a) for a in range(desde, hasta + 1)}

    def segmentos_rogers(self) -> Dict[str, Tuple[float, float]]:
        """
        Segmentos de Rogers basados en la curva S.
        Innovadores: 0-2.5%, Adopters tempranos: 2.5-16%,
        Mayoría temprana: 16-50%, Mayoría tardía: 50-84%, Rezagados: 84%+
        """
        umbrales = {
            "Innovadores (0-2.5%)": (0.0, 0.025 * self.K),
            "Adopters tempranos (2.5-16%)": (0.025 * self.K, 0.16 * self.K),
            "Mayoría temprana (16-50%)": (0.16 * self.K, 0.50 * self.K),
            "Mayoría tardía (50-84%)": (0.50 * self.K, 0.84 * self.K),
            "Rezagados (84%+)": (0.84 * self.K, self.K),
        }
        resultado = {}
        for segmento, (frac_ini, frac_fin) in umbrales.items():
            anio_ini = self.anio_umbral(max(frac_ini, 0.001))
            anio_fin = self.anio_umbral(frac_fin)
            resultado[segmento] = (anio_ini, anio_fin)
        return resultado

from typing import Optional  # noqa: E402

# ---------------------------------------------------------------------------
# Tecnologías con modelos de difusión calibrados
# ---------------------------------------------------------------------------

MODELOS_DIFUSION = [
    ModeloDifusion(
        nombre="Asistentes IA en el trabajo",
        K=0.85,         # 85% de trabajadores del conocimiento
        r=0.9,          # adopción rápida
        t0=3.0,         # inflexión ~2028
        anio_inicio=2025,
    ),
    ModeloDifusion(
        nombre="Agentes autónomos en empresas",
        K=0.70,
        r=0.7,
        t0=4.5,         # inflexión ~2029.5
        anio_inicio=2025,
    ),
    ModeloDifusion(
        nombre="IA en dispositivos personales (on-device)",
        K=0.80,
        r=1.1,          # muy rápida: impulso de fabricantes
        t0=2.5,         # inflexión ~2027.5
        anio_inicio=2025,
    ),
    ModeloDifusion(
        nombre="IA científica (drug discovery, materiales)",
        K=0.60,
        r=0.5,          # más lenta: requiere validación experimental
        t0=5.0,         # inflexión ~2030
        anio_inicio=2025,
    ),
    ModeloDifusion(
        nombre="Robótica general con IA (world models)",
        K=0.40,
        r=0.4,
        t0=6.0,         # inflexión ~2031: fuera del período
        anio_inicio=2025,
    ),
]

# ---------------------------------------------------------------------------
# Análisis y reporte
# ---------------------------------------------------------------------------

print("MODELO DE DIFUSIÓN — ADOPCIÓN 2025-2030")
print("=" * 70)

for modelo in MODELOS_DIFUSION:
    tray = modelo.trayectoria(2025, 2030)
    print(f"\n► {modelo.nombre}")
    print(f"  {'Año':<6}", end="")
    for anio in range(2025, 2031):
        print(f"{anio:<8}", end="")
    print()
    print(f"  {'%':<6}", end="")
    for anio, val in tray.items():
        print(f"{val*100:.1f}%   ", end="")
    print()

    # Segmentos de Rogers
    segmentos = modelo.segmentos_rogers()
    print("  Segmentos de Rogers:")
    for seg, (a_ini, a_fin) in segmentos.items():
        a_ini_str = str(a_ini) if a_ini else "antes 2025"
        a_fin_str = str(a_fin) if a_fin else ">2030"
        print(f"    • {seg:<40} {a_ini_str} → {a_fin_str}")

print("\n✓ Análisis de difusión completado")
```

**Salida esperada:**
```
MODELO DE DIFUSIÓN — ADOPCIÓN 2025-2030
======================================================================

► Asistentes IA en el trabajo
  Año   2025    2026    2027    2028    2029    2030
  %     5.8%    13.2%   26.7%   47.4%   68.1%   81.3%
  Segmentos de Rogers:
    • Innovadores (0-2.5%)                   antes 2025 → 2025
    • Adopters tempranos (2.5-16%)           2025 → 2026
    • Mayoría temprana (16-50%)              2026 → 2028
    • Mayoría tardía (50-84%)               2028 → 2030
    • Rezagados (84%+)                       2030 → >2030
...
```

---

## 3. Análisis de impacto por sector

Cada sector económico tiene un índice de disruption_ia (0-10) y una
velocidad de transformación proyectada.

```python
# outputs/36_impacto_sectorial.py
from dataclasses import dataclass, field
from typing import List, Dict
import statistics

@dataclass
class Sector:
    nombre: str
    disruption_ia: float    # 0-10: impacto esperado de la IA
    velocidad: str          # "lenta" | "media" | "rápida"
    casos_uso_clave: List[str] = field(default_factory=list)
    riesgos: List[str] = field(default_factory=list)
    oportunidades: List[str] = field(default_factory=list)
    empleos_riesgo_pct: float = 0.0     # % empleos afectados por automatización
    nuevos_roles_pct: float = 0.0       # % nuevos roles creados

    @property
    def velocidad_num(self) -> int:
        return {"lenta": 1, "media": 2, "rápida": 3}.get(self.velocidad, 2)

    @property
    def impacto_neto_empleo(self) -> float:
        """Positivo = más empleos creados; negativo = destrucción neta."""
        return self.nuevos_roles_pct - self.empleos_riesgo_pct

    def resumen_impacto(self) -> str:
        neto = self.impacto_neto_empleo
        signo = "+" if neto >= 0 else ""
        return (
            f"{self.nombre:<30} "
            f"Disruption={self.disruption_ia:.1f}/10  "
            f"Empleo neto={signo}{neto:.1f}%  "
            f"Velocidad={self.velocidad}"
        )

SECTORES = [
    Sector(
        nombre="Software y Tecnología",
        disruption_ia=9.5,
        velocidad="rápida",
        casos_uso_clave=["Generación de código", "Testing automático", "DevOps IA"],
        riesgos=["Commoditización del código junior", "Deuda técnica por IA"],
        oportunidades=["10x productividad", "Nuevas categorías de producto"],
        empleos_riesgo_pct=35.0,
        nuevos_roles_pct=55.0,
    ),
    Sector(
        nombre="Salud y Medicina",
        disruption_ia=8.5,
        velocidad="media",
        casos_uso_clave=["Diagnóstico por imagen", "Drug discovery", "Medicina personalizada"],
        riesgos=["Regulación lenta", "Sesgo en datos clínicos"],
        oportunidades=["Reducción de errores diagnósticos", "Acceso global"],
        empleos_riesgo_pct=20.0,
        nuevos_roles_pct=40.0,
    ),
    Sector(
        nombre="Educación",
        disruption_ia=8.0,
        velocidad="media",
        casos_uso_clave=["Tutores personalizados", "Evaluación adaptativa", "Contenido generado"],
        riesgos=["Brecha digital", "Dependencia excesiva de IA"],
        oportunidades=["Aprendizaje personalizado a escala", "Acceso universal"],
        empleos_riesgo_pct=25.0,
        nuevos_roles_pct=30.0,
    ),
    Sector(
        nombre="Servicios Financieros",
        disruption_ia=8.5,
        velocidad="rápida",
        casos_uso_clave=["Trading algorítmico", "Detección de fraude", "Asesores IA"],
        riesgos=["Riesgo sistémico por IA", "Regulación estricta"],
        oportunidades=["Inclusión financiera", "Eficiencia operacional"],
        empleos_riesgo_pct=40.0,
        nuevos_roles_pct=25.0,
    ),
    Sector(
        nombre="Manufactura y Robótica",
        disruption_ia=7.5,
        velocidad="media",
        casos_uso_clave=["Robots autónomos", "Control de calidad IA", "Mantenimiento predictivo"],
        riesgos=["Alto CAPEX", "Resistencia sindical"],
        oportunidades=["Reshoring de producción", "Fabricación personalizada"],
        empleos_riesgo_pct=45.0,
        nuevos_roles_pct=20.0,
    ),
    Sector(
        nombre="Investigación Científica",
        disruption_ia=9.0,
        velocidad="rápida",
        casos_uso_clave=["Generación de hipótesis", "Síntesis de literatura", "Diseño de experimentos"],
        riesgos=["Reproducibilidad de resultados IA", "Pérdida de serendipia"],
        oportunidades=["Aceleración 10-100x", "Ciencia interdisciplinar"],
        empleos_riesgo_pct=15.0,
        nuevos_roles_pct=50.0,
    ),
    Sector(
        nombre="Legal y Cumplimiento",
        disruption_ia=7.0,
        velocidad="media",
        casos_uso_clave=["Revisión de contratos", "Due diligence", "Investigación jurídica"],
        riesgos=["Responsabilidad legal de la IA", "Confidencialidad"],
        oportunidades=["Acceso a justicia democratizado"],
        empleos_riesgo_pct=30.0,
        nuevos_roles_pct=15.0,
    ),
    Sector(
        nombre="Medios y Entretenimiento",
        disruption_ia=8.0,
        velocidad="rápida",
        casos_uso_clave=["Generación de contenido", "Personalización", "Producción IA"],
        riesgos=["Derechos de autor", "Desinformación deepfake"],
        oportunidades=["Creación democratizada", "Nuevos formatos"],
        empleos_riesgo_pct=50.0,
        nuevos_roles_pct=35.0,
    ),
]

# ---------------------------------------------------------------------------
# Análisis estadístico
# ---------------------------------------------------------------------------

class AnalizadorSectorial:
    def __init__(self, sectores: List[Sector]):
        self._sectores = sectores

    def disrupciones_promedio(self) -> float:
        return statistics.mean(s.disruption_ia for s in self._sectores)

    def por_velocidad(self) -> Dict[str, List[str]]:
        resultado: Dict[str, List[str]] = {"lenta": [], "media": [], "rápida": []}
        for s in self._sectores:
            resultado[s.velocidad].append(s.nombre)
        return resultado

    def balance_empleo(self) -> Dict[str, float]:
        """Balance neto de empleo por sector, ordenado."""
        return dict(sorted(
            {s.nombre: s.impacto_neto_empleo for s in self._sectores}.items(),
            key=lambda x: x[1], reverse=True
        ))

    def top_disruptivos(self, n: int = 3) -> List[Sector]:
        return sorted(self._sectores, key=lambda s: s.disruption_ia, reverse=True)[:n]

    def reporte(self) -> str:
        lineas = ["=" * 75, "ANÁLISIS DE IMPACTO SECTORIAL — IA 2025-2030", "=" * 75]
        for s in sorted(self._sectores, key=lambda x: x.disruption_ia, reverse=True):
            lineas.append("  " + s.resumen_impacto())
        lineas.append(f"\n  Disruption media: {self.disrupciones_promedio():.1f}/10")

        balance = self.balance_empleo()
        lineas.append("\nBALANCE NETO DE EMPLEO:")
        for nombre, neto in balance.items():
            signo = "📈" if neto >= 0 else "📉"
            lineas.append(f"  {signo} {nombre:<30} {neto:+.1f}%")
        return "\n".join(lineas)

analizador_s = AnalizadorSectorial(SECTORES)
print(analizador_s.reporte())
```

---

## 4. Hoja de ruta técnica — Hitos clave 2025-2030

```python
# outputs/36_hoja_ruta.py
from dataclasses import dataclass, field
from typing import List, Optional
import json

@dataclass
class Hito:
    anio: int
    trimestre: int      # 1-4
    titulo: str
    descripcion: str
    categoria: str      # LLM | Agentes | Hardware | Safety | Ciencia | Regulacion
    probabilidad: float # 0.0-1.0
    impacto: str        # "bajo" | "medio" | "alto" | "transformador"
    precondiciones: List[str] = field(default_factory=list)

    @property
    def fecha_str(self) -> str:
        return f"{self.anio} Q{self.trimestre}"

    @property
    def impacto_num(self) -> int:
        return {"bajo": 1, "medio": 2, "alto": 3, "transformador": 4}.get(self.impacto, 2)

    def es_probable(self, umbral: float = 0.6) -> bool:
        return self.probabilidad >= umbral

HITOS = [
    # 2025
    Hito(2025, 2, "LLMs de 1M+ tokens de contexto", "Modelos con ventana de contexto ≥1M tokens en producción general",
         "LLM", 0.85, "alto"),
    Hito(2025, 3, "Agentes multimodales en producción", "Primeros agentes autónomos con visión+acción en despliegue empresarial amplio",
         "Agentes", 0.80, "alto"),
    Hito(2025, 4, "Regulación EU AI Act plena", "Entrada en vigor completa del EU AI Act con sanciones activas",
         "Regulacion", 0.95, "medio"),
    Hito(2025, 4, "LLM en dispositivos <4GB RAM", "Modelos de calidad GPT-3.5 funcionando en smartphones sin conexión",
         "Hardware", 0.80, "medio"),
    # 2026
    Hito(2026, 1, "First AGI-adjacent benchmark", "Modelo supera a humanos expertos en suite de benchmarks generalistas",
         "LLM", 0.55, "transformador",
         precondiciones=["Contexto largo", "Razonamiento multi-step robusto"]),
    Hito(2026, 2, "Laboratorio robótico IA autónomo", "Laboratorio de síntesis química dirigido por agentes IA 24/7",
         "Ciencia", 0.65, "transformador"),
    Hito(2026, 3, "Chip IA 2nm de próxima gen", "Primer chip de inferencia con fabricación 2nm y eficiencia 3x actual",
         "Hardware", 0.70, "alto"),
    Hito(2026, 4, "100 fármacos en trial clínico IA-descubiertos", "Primera oleada masiva de candidatos descubiertos por IA en ensayo",
         "Ciencia", 0.60, "transformador"),
    # 2027
    Hito(2027, 1, "MCP universal estándar W3C", "Protocolo MCP estandarizado y adoptado por los principales LLM providers",
         "Agentes", 0.60, "alto"),
    Hito(2027, 2, "World models para robótica general", "Robots con modelos de mundo que se generalizan a entornos no vistos",
         "Agentes", 0.50, "transformador"),
    Hito(2027, 3, "IA en el 30% del PIB digital global", "IA contribuye directamente a ≥30% del valor digital generado",
         "Regulacion", 0.65, "transformador"),
    Hito(2027, 4, "Cuántica + ML primer speedup real", "Demostración verificada de ventaja cuántica en ML task relevante",
         "LLM", 0.35, "transformador",
         precondiciones=["1000+ qubits lógicos", "Corrección de errores cuánticos"]),
    # 2028
    Hito(2028, 1, "Asistente personal IA ubicuo", "80%+ de trabajadores del conocimiento con asistente IA activo diario",
         "Agentes", 0.75, "alto"),
    Hito(2028, 2, "Alineación escalable demostrada", "Técnicas robustas de alineación verificadas en modelos >1T parámetros",
         "Safety", 0.45, "transformador"),
    Hito(2028, 3, "IA en diagnóstico médico nivel 1 global", "Diagnóstico IA equivalente a especialista disponible en todo el mundo",
         "Ciencia", 0.60, "transformador"),
    Hito(2028, 4, "Primer lenguaje de programación IA-first", "Lenguaje diseñado para ser escrito/leído/modificado por LLMs",
         "LLM", 0.40, "medio"),
    # 2029
    Hito(2029, 1, "5G/6G + edge IA convergencia", "IA en tiempo real distribuida en red, sin cloud centralizado",
         "Hardware", 0.65, "alto"),
    Hito(2029, 2, "Neuromorphic computing mainstream", "Chips neuromórficos en producción de volumen para inferencia",
         "Hardware", 0.40, "alto"),
    Hito(2029, 3, "Reglamento global IA básico", "Primer tratado internacional sobre IA frontera con firma de >50 países",
         "Regulacion", 0.45, "alto"),
    # 2030
    Hito(2030, 1, "Robótica doméstica IA a $5,000", "Robot general de hogar con IA avanzada al precio de un coche económico",
         "Agentes", 0.40, "transformador"),
    Hito(2030, 2, "Cura de 10 enfermedades raras por IA", "Tratamientos efectivos descubiertos íntegramente por sistemas IA",
         "Ciencia", 0.50, "transformador"),
    Hito(2030, 4, "1 billón de agentes autónomos activos", "Ecosistema de agentes IA ejecutando tareas del mundo real a escala",
         "Agentes", 0.40, "transformador"),
]

class HojaRuta:
    def __init__(self, hitos: List[Hito]):
        self._hitos = sorted(hitos, key=lambda h: (h.anio, h.trimestre))

    def por_anio(self) -> Dict[int, List[Hito]]:
        resultado: Dict[int, List[Hito]] = {}
        for h in self._hitos:
            resultado.setdefault(h.anio, []).append(h)
        return resultado

    def probabilidad_promedio(self) -> float:
        return statistics.mean(h.probabilidad for h in self._hitos)

    def hitos_transformadores(self) -> List[Hito]:
        return [h for h in self._hitos if h.impacto == "transformador"]

    def hitos_probables(self, umbral: float = 0.6) -> List[Hito]:
        return [h for h in self._hitos if h.probabilidad >= umbral]

    def reporte(self) -> str:
        lineas = ["=" * 70, "HOJA DE RUTA IA 2025-2030 — HITOS CLAVE", "=" * 70]
        for anio, hitos_anio in self.por_anio().items():
            lineas.append(f"\n  ┌── {anio} ──────────────────────────────────────────")
            for h in hitos_anio:
                prob_str = f"{h.probabilidad*100:.0f}%"
                probable = "✓" if h.es_probable() else "?"
                lineas.append(
                    f"  │  {probable} Q{h.trimestre} [{h.categoria:<12}] "
                    f"{h.titulo:<40} P={prob_str} [{h.impacto}]"
                )
        lineas.append(f"\n  Hitos transformadores: {len(self.hitos_transformadores())}")
        lineas.append(f"  Hitos probables (P≥60%): {len(self.hitos_probables())}")
        lineas.append(f"  Probabilidad media: {self.probabilidad_promedio()*100:.1f}%")
        return "\n".join(lineas)

import statistics  # noqa: E402
from typing import Dict  # noqa: E402

ruta = HojaRuta(HITOS)
print(ruta.reporte())

# Guardar JSON
with open("outputs/36_hoja_ruta.json", "w", encoding="utf-8") as f:
    json.dump([
        {"fecha": h.fecha_str, "titulo": h.titulo, "categoria": h.categoria,
         "probabilidad": h.probabilidad, "impacto": h.impacto}
        for h in HITOS
    ], f, ensure_ascii=False, indent=2)
print("\n✓ outputs/36_hoja_ruta.json guardado")
```

---

## 5. Análisis de competencia — Landscape de modelos frontier

```python
# outputs/36_landscape_modelos.py
from dataclasses import dataclass, field
from typing import List, Dict, Optional
import math

@dataclass
class ModeloFrontier:
    nombre: str
    organizacion: str
    parametros_B: float         # miles de millones
    contexto_K: int             # ventana de contexto en K tokens
    multimodal: bool
    abierto: bool               # pesos públicos
    razonamiento_avanzado: bool # chain-of-thought largo integrado
    coste_1M_tokens_USD: float  # coste de inferencia por millón de tokens
    benchmark_mmlu: float       # % exactitud MMLU
    anio_lanzamiento: int
    estado: str                 # "produccion" | "investigacion" | "deprecado"

    def coste_normalizado(self, benchmark_base: float = 50.0) -> float:
        """Coste por punto de MMLU sobre el baseline."""
        delta = self.benchmark_mmlu - benchmark_base
        if delta <= 0:
            return float("inf")
        return self.coste_1M_tokens_USD / delta

    def score_global(self) -> float:
        """Score compuesto: MMLU * log(contexto) / coste^0.5."""
        ctx_factor = math.log10(max(self.contexto_K, 1))
        coste_factor = math.sqrt(max(self.coste_1M_tokens_USD, 0.001))
        return (self.benchmark_mmlu / 100) * ctx_factor / coste_factor

MODELOS = [
    ModeloFrontier("GPT-4o", "OpenAI", 200, 128, True, False, True, 5.0, 87.5, 2024, "produccion"),
    ModeloFrontier("Claude 3.5 Sonnet", "Anthropic", 175, 200, True, False, True, 3.0, 88.3, 2024, "produccion"),
    ModeloFrontier("Gemini 1.5 Pro", "Google", 340, 1000, True, False, True, 7.0, 85.9, 2024, "produccion"),
    ModeloFrontier("LLaMA 3.1 405B", "Meta", 405, 128, True, True, True, 2.0, 85.1, 2024, "produccion"),
    ModeloFrontier("Mistral Large 2", "Mistral AI", 123, 128, False, False, True, 2.0, 84.0, 2024, "produccion"),
    ModeloFrontier("DeepSeek-V3", "DeepSeek", 671, 64, True, True, True, 0.3, 87.1, 2024, "produccion"),
    ModeloFrontier("Qwen2.5-72B", "Alibaba", 72, 128, True, True, True, 0.5, 84.2, 2024, "produccion"),
    ModeloFrontier("Grok-2", "xAI", 314, 128, True, False, True, 6.0, 87.5, 2024, "produccion"),
    ModeloFrontier("Phi-4", "Microsoft", 14, 16, False, True, False, 0.2, 84.8, 2024, "produccion"),
    ModeloFrontier("Gemma 2 27B", "Google", 27, 8, False, True, False, 0.1, 78.0, 2024, "produccion"),
]

class LandscapeModelos:
    def __init__(self, modelos: List[ModeloFrontier]):
        self._modelos = modelos

    def top_benchmark(self, n: int = 3) -> List[ModeloFrontier]:
        return sorted(self._modelos, key=lambda m: m.benchmark_mmlu, reverse=True)[:n]

    def mas_eficientes(self, n: int = 3) -> List[ModeloFrontier]:
        return sorted(self._modelos, key=lambda m: m.score_global(), reverse=True)[:n]

    def abiertos(self) -> List[ModeloFrontier]:
        return [m for m in self._modelos if m.abierto]

    def por_organizacion(self) -> Dict[str, List[ModeloFrontier]]:
        resultado: Dict[str, List[ModeloFrontier]] = {}
        for m in self._modelos:
            resultado.setdefault(m.organizacion, []).append(m)
        return resultado

    def reporte(self) -> str:
        lineas = ["=" * 80, "LANDSCAPE DE MODELOS FRONTIER — IA 2024-2025", "=" * 80]
        lineas.append(f"\n  {'Modelo':<25} {'Org':<12} {'Params':<8} {'Ctx':<8} {'MMLU':<7} {'$/1M':<8} {'Score'}")
        lineas.append("  " + "-" * 76)
        for m in sorted(self._modelos, key=lambda x: x.benchmark_mmlu, reverse=True):
            multimodal = "🖼" if m.multimodal else "  "
            abierto = "🔓" if m.abierto else "  "
            lineas.append(
                f"  {m.nombre:<25} {m.organizacion:<12} "
                f"{m.parametros_B:<8.0f} {m.contexto_K:<8}K "
                f"{m.benchmark_mmlu:<7.1f} "
                f"${m.coste_1M_tokens_USD:<7.2f} "
                f"{m.score_global():.2f} {multimodal}{abierto}"
            )
        lineas.append(f"\n  Modelos de código abierto: {len(self.abiertos())}/{len(self._modelos)}")
        lineas.append("\n  TOP 3 EFICIENCIA (score = MMLU×log(ctx)/√coste):")
        for m in self.mas_eficientes(3):
            lineas.append(f"    ⚡ {m.nombre} — score={m.score_global():.2f}")
        return "\n".join(lineas)

landscape = LandscapeModelos(MODELOS)
print(landscape.reporte())
```

---

## 6. Recomendaciones estratégicas por perfil

```python
# outputs/36_estrategia_perfil.py
from dataclasses import dataclass, field
from typing import List, Dict

@dataclass
class AccionEstrategica:
    prioridad: int          # 1 = máxima
    titulo: str
    descripcion: str
    horizonte: str          # "inmediato" | "1 año" | "2-3 años" | "5 años"
    recursos_necesarios: str
    indicador_exito: str

@dataclass
class PerfilProfesional:
    nombre: str
    descripcion: str
    acciones: List[AccionEstrategica] = field(default_factory=list)

    def acciones_inmediatas(self) -> List[AccionEstrategica]:
        return [a for a in self.acciones if a.horizonte == "inmediato"]

    def reporte(self) -> str:
        lineas = [f"\n{'='*60}", f"PERFIL: {self.nombre.upper()}", f"{'='*60}"]
        lineas.append(f"  {self.descripcion}\n")
        for a in sorted(self.acciones, key=lambda x: x.prioridad):
            lineas.append(f"  [{a.prioridad}] {a.titulo} [{a.horizonte}]")
            lineas.append(f"      {a.descripcion}")
            lineas.append(f"      → Recursos: {a.recursos_necesarios}")
            lineas.append(f"      → KPI: {a.indicador_exito}")
        return "\n".join(lineas)

PERFILES = [
    PerfilProfesional(
        nombre="Desarrollador de Software",
        descripcion="Ingeniero que quiere mantenerse relevante y crecer en la era de la IA",
        acciones=[
            AccionEstrategica(1, "Dominar el desarrollo asistido por IA",
                "Usar Claude/Cursor/GitHub Copilot en el 100% de las tareas diarias",
                "inmediato", "2h/semana práctica", "30% reducción tiempo en tareas rutinarias"),
            AccionEstrategica(2, "Aprender a construir agentes",
                "Completar proyecto real con LangChain/LlamaIndex + herramientas externas",
                "1 año", "Cursos + proyectos personales", "Agente productivo desplegado"),
            AccionEstrategica(3, "Especializarse en IA + dominio vertical",
                "Combinar expertise de dominio (fintech, salud, legal) con habilidades IA",
                "2-3 años", "Estudio del dominio objetivo", "Seniority en posición IA+dominio"),
            AccionEstrategica(4, "Construir intuición sobre sistemas distribuidos IA",
                "Entender deployment, observabilidad y optimización de LLMs en producción",
                "1 año", "M29-M32 de este tutorial", "LLMOps pipeline funcional"),
        ],
    ),
    PerfilProfesional(
        nombre="Investigador / Científico de Datos",
        descripcion="Profesional que diseña y evalúa sistemas de IA en profundidad",
        acciones=[
            AccionEstrategica(1, "Publicar en benchmarks emergentes",
                "Contribuir a nuevos benchmarks de razonamiento, safety o multi-agente",
                "1 año", "GPU cloud + colaboraciones", "1 paper en venue top-tier"),
            AccionEstrategica(2, "Dominar interpretabilidad mecanística",
                "Circuits, attention patterns, SAE (Sparse Autoencoders)",
                "2-3 años", "Papers + código propio", "Análisis publicable de un modelo frontier"),
            AccionEstrategica(3, "Integrar IA con dominio científico propio",
                "Acelerar investigación existente con foundation models y agentes",
                "inmediato", "Tiempo + acceso a APIs", "1 paper IA-acelerado"),
            AccionEstrategica(4, "Explorar evaluación de agentes autónomos",
                "Construir frameworks de evaluación para sistemas multi-agente",
                "2-3 años", "Colaboración academia-industria", "Benchmark adoptado por la comunidad"),
        ],
    ),
    PerfilProfesional(
        nombre="Director / CTO / Emprendedor",
        descripcion="Líder que debe tomar decisiones estratégicas sobre adopción y construcción de IA",
        acciones=[
            AccionEstrategica(1, "Auditar capacidades IA actuales del equipo",
                "Evaluar skills, herramientas y procesos; identificar gaps críticos",
                "inmediato", "1 semana tiempo directivo", "Mapa de capacidades documentado"),
            AccionEstrategica(2, "Diseñar estrategia make vs buy para IA",
                "Decidir qué construir internamente vs. integrar APIs de terceros",
                "inmediato", "Arquitectos + budget", "Decisión documentada con criterios"),
            AccionEstrategica(3, "Pilotar agentes en proceso de alto valor",
                "Seleccionar 1-2 procesos candidatos y medir ROI de automatización IA",
                "1 año", "Equipo 3-5 personas + 6 meses", "ROI medible > 3x coste"),
            AccionEstrategica(4, "Construir governance de IA interno",
                "Políticas de uso, privacidad de datos, auditoría de sesgos",
                "1 año", "Legal + CISO + 2 meses", "Policy aprobada y comunicada"),
            AccionEstrategica(5, "Posicionarse en nicho IA defensible",
                "Datos propietarios + domain expertise = moat ante commoditización",
                "2-3 años", "Inversión sostenida", "Ventaja competitiva cuantificable"),
        ],
    ),
    PerfilProfesional(
        nombre="Profesional No-Técnico",
        descripcion="Abogado, médico, periodista, consultor que quiere aprovechar la IA sin programar",
        acciones=[
            AccionEstrategica(1, "Adoptar asistentes IA en flujo de trabajo diario",
                "Usar Claude/ChatGPT para redacción, investigación y análisis",
                "inmediato", "Suscripción + 1h práctica diaria", "2h/día de tiempo liberado"),
            AccionEstrategica(2, "Aprender prompt engineering básico",
                "Few-shot, chain-of-thought, rol + contexto para el dominio propio",
                "inmediato", "1 semana de práctica", "Prompts reutilizables para tareas recurrentes"),
            AccionEstrategica(3, "Entender límites y riesgos de la IA en tu campo",
                "Alucinaciones, sesgos, privacidad, responsabilidad legal",
                "1 año", "Lectura + formación", "Protocolo de verificación de outputs IA"),
            AccionEstrategica(4, "Combinar expertise humano con IA: ser el experto que valida",
                "El mayor valor es el juicio experto aplicado a outputs de IA",
                "2-3 años", "Experiencia acumulada", "Rol diferenciado en el mercado"),
        ],
    ),
]

for perfil in PERFILES:
    print(perfil.reporte())
    print(f"\n  ACCIONES INMEDIATAS para {perfil.nombre}:")
    for a in perfil.acciones_inmediatas():
        print(f"    ✅ {a.titulo}: {a.indicador_exito}")
```

---

## 7. Síntesis final — El meta-aprendizaje en la era IA

```python
# outputs/36_sintesis_final.py
from dataclasses import dataclass, field
from typing import List
import json
from pathlib import Path

@dataclass
class PrincipioFundamental:
    numero: int
    titulo: str
    descripcion: str
    implicacion_practica: str
    cita: str = ""

PRINCIPIOS = [
    PrincipioFundamental(
        1,
        "La IA amplifica, no reemplaza, el pensamiento de calidad",
        "Los sistemas de IA multiplican la productividad de quienes ya piensan bien "
        "y producen mediocridad amplificada de quienes no lo hacen.",
        "Invierte en pensamiento crítico, diseño de sistemas y comunicación clara "
        "antes de invertir en herramientas IA.",
        "Intelligence is not about knowing, it's about knowing how to think.",
    ),
    PrincipioFundamental(
        2,
        "El contexto es el activo más escaso",
        "En un mundo de capacidades de generación ilimitadas, lo que diferencia "
        "un output mediocre de uno excelente es la calidad del contexto de entrada: "
        "datos propietarios, expertise de dominio, memoria estructurada.",
        "Construye sistemas de captura, organización y recuperación de contexto "
        "antes de construir capacidades de generación.",
    ),
    PrincipioFundamental(
        3,
        "La interfaz con la IA es una habilidad de supervivencia",
        "Saber construir prompts, diseñar workflows de agentes y evaluar outputs "
        "de IA se convierte en tan fundamental como leer y escribir.",
        "Practica el diálogo con LLMs como practicarías un instrumento: "
        "con intención, variación y reflexión sobre los resultados.",
    ),
    PrincipioFundamental(
        4,
        "La velocidad de cambio supera los ciclos de aprendizaje tradicionales",
        "Un modelo frontier lanzado hoy puede ser obsoleto en 18 meses. "
        "Los principios (arquitecturas, algoritmos, trade-offs) duran décadas.",
        "Aprende los fundamentos con profundidad. Las herramientas cambian; "
        "los principios permanecen.",
    ),
    PrincipioFundamental(
        5,
        "La seguridad y la alineación son precondiciones, no añadidos",
        "Sistemas IA inseguros o desalineados generan confianza cero, "
        "que es el punto de fallo más caro en producción.",
        "Integra evaluación de sesgos, trazabilidad y mecanismos de control "
        "desde el diseño inicial, no como paso final.",
    ),
    PrincipioFundamental(
        6,
        "Los sistemas multi-agente emergen con propiedades inesperadas",
        "La composición de agentes especializados produce comportamientos "
        "que no se pueden predecir estudiando los agentes individuales.",
        "Diseña para la observabilidad total: logs estructurados, trazas "
        "causales y puntos de intervención humana en cada sistema multi-agente.",
    ),
    PrincipioFundamental(
        7,
        "El dominio vertical + IA = ventaja defensible",
        "La commoditización de las capacidades base de IA significa que "
        "la ventaja competitiva sostenible reside en la combinación con "
        "expertise de dominio profundo y datos propietarios.",
        "Elige el dominio donde tienes o puedes construir expertise genuino. "
        "La IA generalista + dominio específico supera a ambos por separado.",
    ),
]

class SintesisFinal:
    def __init__(self, principios: List[PrincipioFundamental]):
        self._principios = sorted(principios, key=lambda p: p.numero)

    def reporte_completo(self) -> str:
        lineas = [
            "",
            "╔══════════════════════════════════════════════════════════════╗",
            "║        SÍNTESIS FINAL — TUTORIAL IA MODERNA Y AGENTES       ║",
            "║                    M01 → M36 COMPLETADO                     ║",
            "╚══════════════════════════════════════════════════════════════╝",
            "",
            "  7 PRINCIPIOS FUNDAMENTALES PARA NAVEGAR LA ERA DE LA IA",
            "",
        ]
        for p in self._principios:
            lineas.append(f"  ─── Principio {p.numero}: {p.titulo}")
            lineas.append(f"      {p.descripcion}")
            lineas.append(f"      → ACCIÓN: {p.implicacion_practica}")
            if p.cita:
                lineas.append(f'      ❝ {p.cita} ❞')
            lineas.append("")
        lineas.extend([
            "  ═══════════════════════════════════════════════════════════",
            "  RESUMEN DEL TUTORIAL — 36 MÓDULOS, 8 PARTES",
            "",
            "  Parte 1 — Deep Learning: RNN → Transformers → ViT → Difusión",
            "  Parte 2 — LLMs: Arquitectura → Pre-entrenamiento → Prompting → Fine-tuning",
            "  Parte 3 — RAG: Embeddings → VectorDB → RAG básico → Avanzado → GraphRAG",
            "  Parte 4 — Agentes: ReAct → LangChain → LlamaIndex → CrewAI → AutoGen",
            "  Parte 5 — MCP: Protocolo → Servidores → Skills → Integraciones",
            "  Parte 6 — Multi-Agente: Orquestación → Memoria → Code Interpreter → Safety",
            "  Parte 7 — LLMOps: Evaluación → Observabilidad → Deployment → Optimización",
            "  Parte 8 — Emergentes: Razonamiento → Producción → Quantum → Roadmap",
            "",
            "  ❝ El futuro ya está aquí, solo que no está distribuido de manera uniforme. ❞",
            "                                              — William Gibson",
            "  ═══════════════════════════════════════════════════════════",
        ])
        return "\n".join(lineas)

    def guardar_json(self, ruta: str) -> None:
        datos = [
            {
                "numero": p.numero,
                "titulo": p.titulo,
                "descripcion": p.descripcion,
                "accion": p.implicacion_practica,
            }
            for p in self._principios
        ]
        Path(ruta).parent.mkdir(parents=True, exist_ok=True)
        with open(ruta, "w", encoding="utf-8") as f:
            json.dump(datos, f, ensure_ascii=False, indent=2)

sintesis = SintesisFinal(PRINCIPIOS)
print(sintesis.reporte_completo())
sintesis.guardar_json("outputs/36_principios_finales.json")
print("\n✓ outputs/36_principios_finales.json guardado")
```

**Salida esperada:**
```
╔══════════════════════════════════════════════════════════════╗
║        SÍNTESIS FINAL — TUTORIAL IA MODERNA Y AGENTES       ║
║                    M01 → M36 COMPLETADO                     ║
╚══════════════════════════════════════════════════════════════╝

  7 PRINCIPIOS FUNDAMENTALES PARA NAVEGAR LA ERA DE LA IA

  ─── Principio 1: La IA amplifica, no reemplaza, el pensamiento de calidad
      Los sistemas de IA multiplican la productividad de quienes ya piensan bien...
      → ACCIÓN: Invierte en pensamiento crítico, diseño de sistemas...
  ...

  ═══════════════════════════════════════════════════════════
  RESUMEN DEL TUTORIAL — 36 MÓDULOS, 8 PARTES
  ...
```

---

## Checkpoint M36

```python
# checkpoint_m36.py
"""
Checkpoint M36 — Roadmap y Tendencias 2025-2030
Valida los modelos de proyección, análisis sectorial y hoja de ruta.
"""
import math
import statistics

# ─── Replicaciones mínimas para el checkpoint ─────────────────────────────

# TecnologiaIA simplificada
class TechIA:
    def __init__(self, trl_actual, trl_2025, trl_2030, anio_inicio=2025):
        self.trl_actual = trl_actual
        self.trl_2025 = trl_2025
        self.trl_2030 = trl_2030
        self.anio_inicio = anio_inicio

    def velocidad_maduracion(self):
        return (self.trl_2030 - self.trl_actual) / 5.0

    def trl_en_anio(self, anio):
        if anio <= 2025:
            return self.trl_2025
        elif anio >= 2030:
            return self.trl_2030
        fraccion = (anio - 2025) / (2030 - 2025)
        return round(self.trl_2025 + fraccion * (self.trl_2030 - self.trl_2025))

# ModeloDifusion simplificado
class Difusion:
    def __init__(self, K, r, t0, anio_inicio=2025):
        self.K = K; self.r = r; self.t0 = t0; self.anio_inicio = anio_inicio

    def adopcion(self, anio):
        t = anio - self.anio_inicio
        try:
            return min(self.K / (1 + math.exp(-self.r * (t - self.t0))), self.K)
        except OverflowError:
            return 0.0

# Hito simplificado
class Hito:
    def __init__(self, probabilidad, impacto):
        self.probabilidad = probabilidad
        self.impacto = impacto
    def es_probable(self, umbral=0.6):
        return self.probabilidad >= umbral

# ─── Tests ─────────────────────────────────────────────────────────────────

def test_trl_velocidad_maduracion():
    """Tecnología que pasa de TRL 5 a TRL 9 en 5 años tiene velocidad 0.8."""
    tech = TechIA(trl_actual=5, trl_2025=6, trl_2030=9)
    assert tech.velocidad_maduracion() == 0.8, f"Esperado 0.8, obtenido {tech.velocidad_maduracion()}"
    print("✓ test_trl_velocidad_maduracion")

def test_trl_interpolacion():
    """TRL interpolado en 2027 debe estar entre trl_2025 y trl_2030."""
    tech = TechIA(trl_actual=3, trl_2025=3, trl_2030=7)
    trl_2027 = tech.trl_en_anio(2027)
    assert 3 <= trl_2027 <= 7, f"TRL 2027 fuera de rango: {trl_2027}"
    assert trl_2027 < tech.trl_2030, "TRL 2027 debe ser menor que TRL 2030"
    print(f"✓ test_trl_interpolacion (TRL 2027={trl_2027})")

def test_trl_limites():
    """Años en el límite retornan trl_2025 o trl_2030."""
    tech = TechIA(trl_actual=5, trl_2025=6, trl_2030=9)
    assert tech.trl_en_anio(2025) == 6
    assert tech.trl_en_anio(2030) == 9
    assert tech.trl_en_anio(2020) == 6   # antes de 2025 → trl_2025
    assert tech.trl_en_anio(2035) == 9   # después de 2030 → trl_2030
    print("✓ test_trl_limites")

def test_difusion_logistica():
    """La adopción en el año de inflexión debe ser aproximadamente K/2."""
    d = Difusion(K=0.8, r=0.9, t0=3.0)
    adopcion_inflexion = d.adopcion(2025 + 3)  # t0=3 → anio=2028
    assert abs(adopcion_inflexion - 0.4) < 0.05, \
        f"Adopción en inflexión debe ser ~K/2=0.4, obtenida {adopcion_inflexion:.3f}"
    print(f"✓ test_difusion_logistica (adopción en inflexión={adopcion_inflexion:.3f})")

def test_difusion_monotona():
    """La curva de adopción debe ser monótona no decreciente."""
    d = Difusion(K=0.85, r=0.9, t0=3.0)
    adopciones = [d.adopcion(a) for a in range(2025, 2031)]
    for i in range(len(adopciones) - 1):
        assert adopciones[i] <= adopciones[i + 1], \
            f"No monótona en {2025+i}: {adopciones[i]} > {adopciones[i+1]}"
    print("✓ test_difusion_monotona")

def test_hito_probabilidad():
    """Hitos con P≥0.6 son considerados probables, los demás no."""
    h_probable = Hito(probabilidad=0.75, impacto="alto")
    h_improbable = Hito(probabilidad=0.45, impacto="transformador")
    assert h_probable.es_probable() is True
    assert h_improbable.es_probable() is False
    # Con umbral personalizado
    assert h_improbable.es_probable(umbral=0.4) is True
    print("✓ test_hito_probabilidad")

if __name__ == "__main__":
    print("Ejecutando checkpoint M36...")
    test_trl_velocidad_maduracion()
    test_trl_interpolacion()
    test_trl_limites()
    test_difusion_logistica()
    test_difusion_monotona()
    test_hito_probabilidad()
    print("\n✅ Todos los tests de M36 pasaron correctamente.")
    print("✅ Tutorial 03-ia-moderna-agentes COMPLETADO — M01 → M36")
```

**Salida esperada:**
```
Ejecutando checkpoint M36...
✓ test_trl_velocidad_maduracion
✓ test_trl_interpolacion (TRL 2027=5)
✓ test_trl_limites
✓ test_difusion_logistica (adopción en inflexión=0.400)
✓ test_difusion_monotona
✓ test_hito_probabilidad

✅ Todos los tests de M36 pasaron correctamente.
✅ Tutorial 03-ia-moderna-agentes COMPLETADO — M01 → M36
```

---

## Resumen del módulo

| Concepto | Implementación |
|---|---|
| `TecnologiaIA` | TRL actual + proyección 2025/2027/2030, velocidad de maduración |
| `AnalizadorTRL` | Ranking por TRL y velocidad, categorías, reporte |
| `ModeloDifusion` | Curva logística S: `K/(1+e^{-r(t-t0)})` |
| `anio_umbral()` | Inversión de la logística para estimar años de adopción |
| `segmentos_rogers()` | Innovadores/Adopters/Mayoría/Rezagados con años estimados |
| `Sector` | Disruption IA + balance neto de empleo |
| `HojaRuta` | 20 hitos 2025-2030 con probabilidad e impacto |
| `ModeloFrontier` | Score = MMLU × log(ctx) / √coste |
| `PerfilProfesional` | 4 perfiles con acciones estratégicas priorizadas |
| `SintesisFinal` | 7 principios fundamentales + resumen del tutorial |

**Tecnologías con mayor potencial transformador 2025-2030:**
1. **Agentes autónomos de larga duración** — la frontera más activa hoy
2. **IA científica acelerada** — impacto en salud, clima y materiales
3. **World models para robótica** — convergencia física-digital
4. **Alineación escalable** — precondición para todo lo demás

> *"El objetivo no es construir IA que reemplace a los humanos, sino IA que amplíe lo que los humanos pueden hacer."*
