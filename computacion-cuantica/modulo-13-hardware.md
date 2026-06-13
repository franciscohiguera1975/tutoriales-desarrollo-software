# Módulo 13 — Tipos de Hardware Cuántico

**Objetivo:** Comprender las principales tecnologías de hardware cuántico, sus principios físicos,
ventajas, limitaciones y el estado actual del arte en 2024-2025.

**Herramientas:** Python (sin dependencias cuánticas), matplotlib, numpy
**Prerequisito:** M01 (Mecánica Cuántica), M02 (Qubits)

---

## Cómo ejecutar este módulo

```bash
conda activate quantum
# Solo requiere numpy y matplotlib — sin frameworks cuánticos
python modulo-13-hardware.py
```

---

## 1. El reto del hardware cuántico

Un qubit físico requiere:
1. **Superposición coherente**: mantener |ψ⟩ = α|0⟩ + β|1⟩ sin colapsar
2. **Control preciso**: aplicar puertas con fidelidad > 99%
3. **Medición selectiva**: leer un qubit sin perturbar los demás
4. **Escalabilidad**: conectar miles de qubits con error bajo

El problema central es que cualquier interacción con el entorno destruye la coherencia.
Esto se llama **decoherencia** — el enemigo principal del hardware cuántico.

```python
# modulo-13-hardware.py
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
from matplotlib.gridspec import GridSpec
import os

os.makedirs("outputs", exist_ok=True)

# ─────────────────────────────────────────────
# SECCIÓN 1: Comparación de tecnologías
# ─────────────────────────────────────────────

TECNOLOGIAS = {
    "Superconductor\n(IBM, Google)": {
        "temperatura_K": 0.015,           # 15 mK
        "tiempo_coherencia_us": 100,      # T2 típico
        "fidelidad_1q_pct": 99.9,
        "fidelidad_2q_pct": 99.5,
        "frecuencia_puerta_MHz": 50,      # velocidad de operación
        "qubits_actuales": 1000,          # IBM Condor 2023
        "conectividad": "Vecino cercano (2D lattice)",
        "temperatura_relativa": "Más frío que el espacio interestelar",
        "color": "#1f77b4",
    },
    "Trampa de iones\n(IonQ, Quantinuum)": {
        "temperatura_K": 1e-7,            # doppler cooling → μK
        "tiempo_coherencia_us": 1e6,      # hasta segundos
        "fidelidad_1q_pct": 99.99,
        "fidelidad_2q_pct": 99.9,
        "frecuencia_puerta_MHz": 0.01,    # más lentas (kHz)
        "qubits_actuales": 32,            # IonQ Forte 2023
        "conectividad": "Todo-con-todo (all-to-all)",
        "temperatura_relativa": "Ultra-frío pero más cálido que supercond.",
        "color": "#ff7f0e",
    },
    "Fotónico\n(PsiQuantum, Xanadu)": {
        "temperatura_K": 300,             # temperatura ambiente!
        "tiempo_coherencia_us": float("inf"),  # fotones no decoheren térmicamente
        "fidelidad_1q_pct": 99.5,
        "fidelidad_2q_pct": 95.0,        # puertas 2-fotón probabilísticas
        "frecuencia_puerta_MHz": 1000,   # velocidad de luz
        "qubits_actuales": 216,          # Borealis (Xanadu 2022, modo Gaussian Boson Sampling)
        "conectividad": "Reconfigurable (modos ópticos)",
        "temperatura_relativa": "Temperatura ambiente — sin refrigeración criogénica",
        "color": "#2ca02c",
    },
    "Neutral atoms\n(QuEra, Pasqal)": {
        "temperatura_K": 1e-6,            # laser cooling → μK
        "tiempo_coherencia_us": 1000,
        "fidelidad_1q_pct": 99.5,
        "fidelidad_2q_pct": 99.5,
        "frecuencia_puerta_MHz": 0.1,
        "qubits_actuales": 280,          # QuEra Aquila 2022
        "conectividad": "Flexible — reposicionable con pinzas ópticas",
        "temperatura_relativa": "Ultra-frío, laser cooling",
        "color": "#d62728",
    },
    "Topológico\n(Microsoft)": {
        "temperatura_K": 0.02,
        "tiempo_coherencia_us": float("inf"),  # protegido topológicamente (teórico)
        "fidelidad_1q_pct": 99.0,        # aún en desarrollo
        "fidelidad_2q_pct": 95.0,
        "frecuencia_puerta_MHz": 1,
        "qubits_actuales": 8,            # demostraciones tempranas 2023
        "conectividad": "TBD — aún en investigación",
        "temperatura_relativa": "Criogénico",
        "color": "#9467bd",
    },
}

print("=" * 70)
print("COMPARACIÓN DE TECNOLOGÍAS DE HARDWARE CUÁNTICO (2024-2025)")
print("=" * 70)

for nombre, datos in TECNOLOGIAS.items():
    nombre_limpio = nombre.replace("\n", " ")
    print(f"\n{'─'*50}")
    print(f"  {nombre_limpio}")
    print(f"{'─'*50}")
    t_coh = datos['tiempo_coherencia_us']
    t_str = f"{t_coh:.0f} μs" if t_coh != float("inf") else "∞ (protección topológica)"
    print(f"  Temperatura:          {datos['temperatura_K']:.3g} K")
    print(f"  Tiempo de coherencia: {t_str}")
    print(f"  Fidelidad 1Q:         {datos['fidelidad_1q_pct']}%")
    print(f"  Fidelidad 2Q:         {datos['fidelidad_2q_pct']}%")
    print(f"  Qubits actuales:      {datos['qubits_actuales']}")
    print(f"  Conectividad:         {datos['conectividad']}")
```

---

## 2. Qubits superconductores — El líder actual

Los qubits superconductores son circuitos LC cuánticos donde una unión Josephson
introduce la no-linealidad necesaria para que sea un sistema de dos niveles.

```python
# ─────────────────────────────────────────────
# SECCIÓN 2: Física de qubits superconductores
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("QUBITS SUPERCONDUCTORES — FÍSICA")
print("="*60)

# Parámetros del transmon (el tipo más común hoy)
hbar = 1.054e-34        # J·s
e = 1.602e-19           # C
kb = 1.381e-23          # J/K

# Energías típicas de un transmon
EJ_over_h = 15e9        # Hz (energía Josephson)
EC_over_h = 0.25e9      # Hz (energía de carga)
EJ_EC_ratio = EJ_over_h / EC_over_h

# Frecuencia del qubit (aprox): ω ≈ √(8·EJ·EC) - EC
omega_qubit = np.sqrt(8 * EJ_over_h * EC_over_h) - EC_over_h
freq_GHz = omega_qubit / 1e9

# Temperatura de operación
T_base = 0.015          # Kelvin (15 mK)
T_quantum = (hbar * omega_qubit * 1e9 * 2*np.pi) / kb  # temperatura cuántica

print(f"\nParámetros del transmon:")
print(f"  EJ/h = {EJ_over_h/1e9:.1f} GHz")
print(f"  EC/h = {EC_over_h/1e9:.3f} GHz")
print(f"  EJ/EC = {EJ_EC_ratio:.0f}  (debe ser >> 1 para protección de carga)")
print(f"  Frecuencia del qubit: {freq_GHz:.2f} GHz")
print(f"  Temperatura de operación: {T_base*1000:.0f} mK")
print(f"  'Temperatura cuántica' (hω/kb): {T_quantum*1e3:.1f} mK")
print(f"  Relación T_op/T_quantum: {T_base/T_quantum:.3f} ← debe ser << 1")
print(f"  → El sistema está en estado base con P ≈ {1-np.exp(-T_quantum/T_base):.6f}")

# Unión Josephson
print(f"\nUnión Josephson:")
print(f"  Corriente crítica Ic convierte energía en oscilación cuántica")
print(f"  I_s = Ic · sin(φ)  donde φ es la fase superconductora")
print(f"  No-linealidad: E_n ≠ n · E_1  → sistema de 2 niveles aislable")

# Tipos de transmon modernos
topologias = {
    "Transmon estándar": {
        "qubits": "1",
        "T1": "~100 μs",
        "T2": "~100 μs",
        "fabricante": "IBM, Google, Rigetti",
    },
    "Xmon (cruz)": {
        "qubits": "1",
        "T1": "~50 μs",
        "T2": "~20 μs",
        "fabricante": "Google",
    },
    "Transmon 3D": {
        "qubits": "1 en cavidad 3D",
        "T1": "~1 ms",
        "T2": "~500 μs",
        "fabricante": "Yale, IBM Research",
    },
    "Fluxonium": {
        "qubits": "1",
        "T1": "~1 ms",
        "T2": "~1 ms",
        "fabricante": "MIT, IBM Research",
    },
}

print(f"\nVariantes de qubits superconductores:")
for nombre, datos in topologias.items():
    print(f"  {nombre}:")
    print(f"    T1={datos['T1']}, T2={datos['T2']} | {datos['fabricante']}")
```

---

## 3. Trampas de iones — Fidelidad máxima

Los iones atrapados son qubits naturales: cada ion idéntico a los demás.
Se usan los estados electrónicos del ion como |0⟩ y |1⟩.

```python
# ─────────────────────────────────────────────
# SECCIÓN 3: Trampas de iones
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("TRAMPAS DE IONES — FÍSICA")
print("="*60)

# El proceso: captura → enfriamiento → computación → medición
proceso = [
    ("1. Ionización", "Átomo neutro → ión por ablación láser o fuente de plasma"),
    ("2. Trampa de Paul", "Campo EM oscilante atrapa iones en punto de equilibrio"),
    ("3. Doppler cooling", "Láser contra-propagante enfría a μK (doppler limit)"),
    ("4. Sideband cooling", "Enfría el modo fonónico hasta estado base |n=0⟩"),
    ("5. Inicialización", "Optical pumping → todos los iones en |0⟩ = |↓⟩"),
    ("6. Puertas 1Q", "Pulsos láser/microondas: rotaciones en esfera de Bloch"),
    ("7. Puertas 2Q", "Mølmer-Sørensen gate via modo fonónico compartido"),
    ("8. Medición", "Fluorescencia: |1⟩ → fotón brillante, |0⟩ → oscuro"),
]

for paso, desc in proceso:
    print(f"  {paso}: {desc}")

print(f"\nIones más usados:")
iones = {
    "⁹⁰Sr⁺ (estroncio)":  {"qubit": "electrónico (óptico)", "empresa": "Oxford Ionics"},
    "¹⁷¹Yb⁺ (iterbio)":   {"qubit": "hiperfino (microondas)", "empresa": "IonQ, Quantinuum"},
    "⁴⁰Ca⁺ (calcio)":     {"qubit": "electrónico (óptico)", "empresa": "Innsbruck, AQT"},
    "¹³³Ba⁺ (bario)":     {"qubit": "hiperfino", "empresa": "University of Washington"},
}

for ion, datos in iones.items():
    print(f"  {ion}: {datos['qubit']} | {datos['empresa']}")

# La puerta Mølmer-Sørensen
print(f"\nPuerta Mølmer-Sørensen (MS gate):")
print(f"  |00⟩ → (|00⟩ + i|11⟩)/√2  [equivale a Bell + rotación]")
print(f"  Mecanismo: interacción ion-ion mediada por modo fonónico")
print(f"  Ventaja: all-to-all — cualquier par de iones puede interactuar")
print(f"  Fidelidad: >99.9% en sistemas de pocos qubits")

# Comparación velocidad vs fidelidad
print(f"\nComparación velocidad vs fidelidad:")
print(f"{'Tecnología':<20} {'Puerta 2Q':<15} {'Fidelidad 2Q':<15}")
print(f"{'─'*50}")
print(f"{'Superconductor':<20} {'~100 ns':<15} {'~99.5%':<15}")
print(f"{'Trampa de iones':<20} {'~10-100 μs':<15} {'~99.9%':<15}")
print(f"→ Iones son 100-1000x más lentos pero más fieles")
```

---

## 4. Fotónico y átomos neutros

```python
# ─────────────────────────────────────────────
# SECCIÓN 4: Fotónico y átomos neutros
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("FOTÓNICO Y ÁTOMOS NEUTROS")
print("="*60)

print("\n--- FOTÓNICO ---")
print("""
Qubit fotónico: un fotón codifica |0⟩ y |1⟩ en:
  • Polarización: |H⟩=|0⟩, |V⟩=|1⟩
  • Path encoding: camino óptico A vs B
  • Time-bin encoding: fotón temprano vs tardío
  • Number encoding (GBS): 0 o 1 fotón en un modo

Ventajas:
  ✓ Temperatura ambiente — sin dilución criogénica
  ✓ Velocidad de la luz — puertas a GHz
  ✓ Decoherencia mínima (fotones casi no interactúan con entorno)
  ✓ Natural para comunicación cuántica

Desventajas:
  ✗ Puertas 2Q entre fotones son difíciles/probabilísticas
  ✗ Pérdidas en fibras ópticas y componentes
  ✗ Detección de fotón único requiere SNSPD (criogénico)
  ✗ Gaussian Boson Sampling ≠ computación cuántica universal directa

Enfoque PsiQuantum: millones de qubits fotónicos vía error correction
  → Target: computación tolerante a fallos con fotones
""")

print("\n--- ÁTOMOS NEUTROS ---")
print("""
Qubits: estados hiperfinos de átomos (Rb, Cs, Sr) atrapados en
        pinzas ópticas (optical tweezers) o redes ópticas.

Proceso:
  1. Enfriamiento láser a μK
  2. Trampa en foco de láser (pinza óptica)
  3. Arrays 2D/3D configurables con pinzas individuales
  4. Puertas 2Q via interacciones Rydberg (átomo altamente excitado)
  5. Medición por fluorescencia

Ventajas:
  ✓ Arrays grandes y reconfigurables (QuEra: 256 qubits en 2D)
  ✓ Conectividad flexible: reposicionar átomos
  ✓ Fidelidades de dos qubits ~99.5% (Rydberg gates)
  ✓ Tiempo de coherencia ~1-10 segundos

Desventajas:
  ✗ Velocidad moderada (~μs por puerta)
  ✗ Pérdidas por imagen de fluorescencia
  ✗ Temperatura criogénica (μK) aún requerida

Hito 2024: Harvard/QuEra — 48 qubits lógicos con QEC en hardware neutral atoms
""")
```

---

## 5. Qubits topológicos — El Santo Grial

```python
# ─────────────────────────────────────────────
# SECCIÓN 5: Qubits topológicos
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("QUBITS TOPOLÓGICOS — MICROSOFT")
print("="*60)

print("""
Idea central: codificar información cuántica de forma NO-LOCAL
→ no hay un lugar físico donde "vive" el qubit
→ perturbaciones locales no pueden destruir la información

Fundamento: Fermiones de Majorana
  • Partículas que son su propia antipartícula: γ = γ†
  • En física de materia condensada: cuasipartículas en nanowires
    superconductores bajo campo magnético
  • Par Majorana = 1 qubit topológico protegido

Ventajas teóricas:
  ✓ Decoherencia exponencialmente suprimida
  ✓ Puertas cuánticas por trenzado (braiding) = operaciones topológicas
  ✓ Tolerancia a fallos intrínseca
  ✓ Escalabilidad potencial hacia el millón de qubits

Estado actual (2024):
  Microsoft Azure Quantum:
  • 2023: primer par Majorana verificado en nanowire InAs/Al
  • Roadmap: Logical Qubit → Resilient Qubit → Scale
  • aún en etapas tempranas de demostración

Desafíos:
  ✗ Fabricación extremadamente difícil
  ✗ Interferencia de otros estados del gap
  ✗ Braiding lento
  ✗ Resultados cuestionados y retirados (2021) — ciencia cuidadosa requerida
""")
```

---

## 6. Mapa del ecosistema actual (2024-2025)

```python
# ─────────────────────────────────────────────
# SECCIÓN 6: Ecosistema y roadmaps
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("ECOSISTEMA CUÁNTICO 2024-2025")
print("="*60)

empresas = {
    "IBM Quantum": {
        "tecnología": "Superconductor (transmon)",
        "hitos": [
            "2019: 53 qubits (Rochester)",
            "2021: 127 qubits (Eagle)",
            "2022: 433 qubits (Osprey)",
            "2023: 1121 qubits (Condor)",
            "2024: 133q Heron R2 (mejor calidad)",
        ],
        "roadmap": "2025: Kookaburra (1386q modulares); 2033: Quantum-centric supercomputing",
        "acceso": "IBM Quantum Platform (gratuito con límites)",
    },
    "Google Quantum AI": {
        "tecnología": "Superconductor (Sycamore/Willow)",
        "hitos": [
            "2019: 53q Sycamore — supremacía cuántica (200s vs 10000 años)",
            "2023: 70q Sycamore mejorado",
            "2024: Willow — 105q, error por debajo del umbral QEC",
        ],
        "roadmap": "Error correction, Willow successors",
        "acceso": "Google Cloud Quantum Computing Service (beta/enterprise)",
    },
    "IonQ": {
        "tecnología": "Trampa de iones (Yb⁺)",
        "hitos": [
            "2020: 32 qubits #AQ (métrica propia)",
            "2023: Forte — 35 #AQ",
            "2024: Forte Enterprise",
        ],
        "roadmap": "2025: 64 #AQ; barcoded ions, photonic interconnects",
        "acceso": "AWS Braket, Azure Quantum, Google Cloud",
    },
    "Quantinuum": {
        "tecnología": "Trampa de iones (Yb⁺, Ba⁺)",
        "hitos": [
            "2022: H1 — 20 qubits, fidelidad 99.9%",
            "2023: H2 — 32 qubits",
            "2024: H2-1 — mejores fidelidades del mercado",
        ],
        "roadmap": "2025: 56 qubits, QEC demostraciones",
        "acceso": "Azure Quantum, Quantinuum Direct",
    },
    "QuEra Computing": {
        "tecnología": "Átomos neutros (Rb)",
        "hitos": [
            "2022: Aquila — 256 neutral atoms (analog)",
            "2024: 48 qubits lógicos con QEC (Harvard collab)",
        ],
        "roadmap": "2025: 100 qubits lógicos; digital+analog",
        "acceso": "AWS Braket",
    },
    "PsiQuantum": {
        "tecnología": "Fotónico",
        "hitos": [
            "Stealth hasta 2023 — millones de qubits fotónicos target",
            "2024: Partnership con GlobalFoundries para chips fotónicos",
        ],
        "roadmap": "Fault-tolerant at scale via photonics + silicon photonics fab",
        "acceso": "No público aún",
    },
}

for empresa, datos in empresas.items():
    print(f"\n{'─'*55}")
    print(f"  {empresa} [{datos['tecnología']}]")
    print(f"{'─'*55}")
    print(f"  Hitos clave:")
    for h in datos["hitos"][-2:]:  # últimos 2
        print(f"    • {h}")
    print(f"  Roadmap: {datos['roadmap']}")
    print(f"  Acceso: {datos['acceso']}")
```

---

## 7. Métricas de calidad: más allá del número de qubits

```python
# ─────────────────────────────────────────────
# SECCIÓN 7: Métricas de calidad
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("MÉTRICAS DE CALIDAD — MÁS ALLÁ DEL NÚMERO DE QUBITS")
print("="*60)

print("""
El número de qubits físicos es una métrica engañosa.
Las métricas importantes son:

1. QUANTUM VOLUME (QV) — IBM
   QV = 2^n donde n es el mayor circuito cuadrado (n×n profundidad)
   que el dispositivo puede ejecutar con heavy-output probability > 2/3.
   QV_IBM_Eagle: 512 | QV_Quantinuum_H1: 8192

2. ALGORITHMIC QUBITS (#AQ) — IonQ
   Número de qubits que pueden ejecutar algoritmos reales con resultados correctos.
   Captura: conectividad + fidelidad + profundidad conjuntamente.

3. CIRCUIT LAYER OPERATIONS PER SECOND (CLOPS) — IBM
   Velocidad de ejecución de capas de circuito.
   IBM Eagle: 1400 CLOPS | Cuánto más alto, más rápido.

4. FIDELIDAD DE PUERTA (Gate Fidelity)
   1Q: IBM ~99.9%, IonQ/Quantinuum ~99.99%
   2Q: IBM ~99.5%, IonQ ~99.5%, Quantinuum ~99.9%

5. T1 (relajación) y T2 (decoherencia)
   T1: tiempo hasta que |1⟩ decae a |0⟩
   T2: tiempo hasta perder coherencia (fase)
   Generalmente T2 ≤ 2·T1 (Hahn echo: T2 ≤ 2·T1)

6. QUBITS LÓGICOS vs FÍSICOS
   Para computación tolerante a fallos:
   1 qubit lógico = ~1000 qubits físicos (código de superficie, p=0.1%)
   IBM Condor (1121 físicos) ≈ ~1 qubit lógico real
   Objetivo industria 2030: 1M+ qubits físicos → 1000+ qubits lógicos
""")

# Simulación de cómo el ruido limita la profundidad útil
print("\n--- PROFUNDIDAD ÚTIL SEGÚN FIDELIDAD ---")
fidelidades_2q = [0.99, 0.995, 0.999, 0.9999]
profundidades = np.arange(1, 200)

print(f"\n{'Fidelidad 2Q':<15} {'Profund. útil (P>50%)':<25} {'Profund. útil (P>90%)'}")
print(f"{'─'*65}")
for f in fidelidades_2q:
    # P_exitosa = f^(n_puertas_2q) — simplificado
    # Asumir n_puertas_2q = profundidad * 0.5 (mezcla de 1Q y 2Q)
    probs = [f**(d * 0.5) for d in profundidades]
    prof_50 = next((d for d, p in zip(profundidades, probs) if p < 0.5), ">200")
    prof_90 = next((d for d, p in zip(profundidades, probs) if p < 0.9), ">200")
    print(f"  {f*100:.2f}%         {str(prof_50):<25} {str(prof_90)}")
```

---

## 8. Visualización comparativa

```python
# ─────────────────────────────────────────────
# SECCIÓN 8: Visualización
# ─────────────────────────────────────────────

fig = plt.figure(figsize=(18, 12))
gs = GridSpec(2, 3, figure=fig, hspace=0.4, wspace=0.4)

nombres = list(TECNOLOGIAS.keys())
colores = [d["color"] for d in TECNOLOGIAS.values()]

# --- Plot 1: Tiempo de coherencia (log scale) ---
ax1 = fig.add_subplot(gs[0, 0])
tiempos = []
for d in TECNOLOGIAS.values():
    t = d["tiempo_coherencia_us"]
    tiempos.append(t if t != float("inf") else 1e9)  # placeholder para ∞

bars = ax1.bar(range(len(nombres)), tiempos, color=colores, alpha=0.85)
ax1.set_yscale("log")
ax1.set_xticks(range(len(nombres)))
ax1.set_xticklabels([n.replace("\n", "\n") for n in nombres], fontsize=7)
ax1.set_ylabel("Tiempo de coherencia (μs)")
ax1.set_title("Tiempo de coherencia\n(escala logarítmica)", fontsize=10)
ax1.axhline(y=100, color="red", linestyle="--", alpha=0.5, label="100 μs threshold")
ax1.legend(fontsize=7)

# --- Plot 2: Fidelidad 2Q ---
ax2 = fig.add_subplot(gs[0, 1])
fids_2q = [d["fidelidad_2q_pct"] for d in TECNOLOGIAS.values()]
bars2 = ax2.bar(range(len(nombres)), fids_2q, color=colores, alpha=0.85)
ax2.set_xticks(range(len(nombres)))
ax2.set_xticklabels([n for n in nombres], fontsize=7)
ax2.set_ylabel("Fidelidad 2Q (%)")
ax2.set_title("Fidelidad de puertas\nde 2 qubits", fontsize=10)
ax2.set_ylim(94, 100.2)
ax2.axhline(y=99.0, color="red", linestyle="--", alpha=0.5, label="99% threshold")
ax2.legend(fontsize=7)
for bar, val in zip(bars2, fids_2q):
    ax2.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.02,
             f"{val}%", ha="center", va="bottom", fontsize=7)

# --- Plot 3: Qubits actuales ---
ax3 = fig.add_subplot(gs[0, 2])
qubits_n = [d["qubits_actuales"] for d in TECNOLOGIAS.values()]
bars3 = ax3.bar(range(len(nombres)), qubits_n, color=colores, alpha=0.85)
ax3.set_xticks(range(len(nombres)))
ax3.set_xticklabels([n for n in nombres], fontsize=7)
ax3.set_ylabel("Número de qubits")
ax3.set_title("Qubits físicos\n(estado arte 2024)", fontsize=10)
ax3.set_yscale("log")
for bar, val in zip(bars3, qubits_n):
    ax3.text(bar.get_x() + bar.get_width()/2, bar.get_height() * 1.1,
             str(val), ha="center", va="bottom", fontsize=8, fontweight="bold")

# --- Plot 4: Temperatura de operación ---
ax4 = fig.add_subplot(gs[1, 0])
temps = [d["temperatura_K"] for d in TECNOLOGIAS.values()]
temps_plot = [t if t != float("inf") else 1e3 for t in temps]
bars4 = ax4.bar(range(len(nombres)), temps_plot, color=colores, alpha=0.85)
ax4.set_yscale("log")
ax4.set_xticks(range(len(nombres)))
ax4.set_xticklabels([n for n in nombres], fontsize=7)
ax4.set_ylabel("Temperatura (K)")
ax4.set_title("Temperatura de operación\n(300K = ambiente)", fontsize=10)
ax4.axhline(y=300, color="orange", linestyle="--", alpha=0.7, label="Temp. ambiente")
ax4.axhline(y=0.015, color="blue", linestyle="--", alpha=0.7, label="15 mK (dilution)")
ax4.legend(fontsize=7)

# --- Plot 5: Velocidad de puertas ---
ax5 = fig.add_subplot(gs[1, 1])
freqs = [d["frecuencia_puerta_MHz"] for d in TECNOLOGIAS.values()]
bars5 = ax5.bar(range(len(nombres)), freqs, color=colores, alpha=0.85)
ax5.set_yscale("log")
ax5.set_xticks(range(len(nombres)))
ax5.set_xticklabels([n for n in nombres], fontsize=7)
ax5.set_ylabel("Frecuencia de puertas (MHz)")
ax5.set_title("Velocidad de operación\n(puertas por segundo)", fontsize=10)

# --- Plot 6: Radar chart - profundidad útil vs fidelidad ---
ax6 = fig.add_subplot(gs[1, 2])
fids_range = np.linspace(0.99, 0.9999, 100)
max_depth_50 = np.log(0.5) / (np.log(fids_range) * 0.5)
max_depth_90 = np.log(0.9) / (np.log(fids_range) * 0.5)

ax6.plot(fids_range * 100, max_depth_50, "b-", label="P>50% (viable)", linewidth=2)
ax6.plot(fids_range * 100, max_depth_90, "g-", label="P>90% (alto éxito)", linewidth=2)
ax6.set_xlabel("Fidelidad 2Q (%)")
ax6.set_ylabel("Profundidad máxima útil")
ax6.set_title("Profundidad útil\nvs fidelidad", fontsize=10)
ax6.set_yscale("log")
ax6.legend(fontsize=8)
ax6.grid(True, alpha=0.3)

# Marcar tecnologías
tech_fids = {"SC": 99.5, "Iones": 99.9, "Foton": 95.0, "NA": 99.5}
tech_colors_map = {"SC": "#1f77b4", "Iones": "#ff7f0e", "Foton": "#2ca02c", "NA": "#d62728"}
for tname, fid in tech_fids.items():
    d50 = np.log(0.5) / (np.log(fid/100) * 0.5)
    ax6.scatter([fid], [d50], s=80, zorder=5, color=tech_colors_map[tname])
    ax6.annotate(tname, (fid, d50), textcoords="offset points",
                xytext=(5, 5), fontsize=7, color=tech_colors_map[tname])

fig.suptitle("Comparación de Tecnologías de Hardware Cuántico (2024-2025)",
             fontsize=14, fontweight="bold", y=1.01)

plt.savefig("outputs/m13_hardware_comparacion.png", dpi=150, bbox_inches="tight")
plt.close()
print("\n→ Guardado: outputs/m13_hardware_comparacion.png")

# ─────────────────────────────────────────────
# SECCIÓN 9: Guía de selección
# ─────────────────────────────────────────────

print("\n" + "="*60)
print("GUÍA DE SELECCIÓN DE PLATAFORMA")
print("="*60)

print("""
¿Qué hardware usar para mi problema?

ALGORITMOS DE ALTA FIDELIDAD (pocos qubits, circuitos profundos):
  → Trampa de iones (IonQ, Quantinuum)
  → Ej: VQE para química, algoritmos de pocos qubits

ALGORITMOS DE MUCHOS QUBITS (circuitos poco profundos):
  → Superconductor (IBM, Google)
  → Ej: QAOA, QML en datos de alta dimensión, Grover a escala

SIMULACIÓN ANALÓGICA (muchos qubits, dinámica Hamiltoniana):
  → Átomos neutros (QuEra Aquila)
  → Ej: optimización combinatoria, sistemas de spines

COMUNICACIÓN CUÁNTICA (redes, criptografía):
  → Fotónico
  → Ej: QKD, quantum repeaters, redes cuánticas

INVESTIGACIÓN (topología, física exótica):
  → Topológico (cuando esté disponible)

ACCESO GRATUITO PARA APRENDIZAJE:
  → IBM Quantum (ibm_brisbane, ibm_kyoto, etc.)
  → simuladores locales: AerSimulator, Pennylane, Cirq
""")
```

---

## 9. La frontera cuántica — ¿Cuándo la ventaja cuántica práctica?

```python
# ─────────────────────────────────────────────
# SECCIÓN 9: Quantum advantage timeline
# ─────────────────────────────────────────────

print("=" * 60)
print("LA FRONTERA CUÁNTICA — TIMELINE ESTIMADO")
print("=" * 60)

timeline = {
    "2019": "Google Sycamore — supremacía cuántica (tarea artificial)",
    "2023": "Google Willow — QEC por debajo del umbral de error",
    "2024": "QuEra/Harvard — 48 qubits lógicos (neutral atoms)",
    "2025 (est.)": "IBM — 100+ qubits lógicos; demostraciones QEC estables",
    "2026-2028 (est.)": "Ventaja cuántica en química/materiales (VQE industrial)",
    "2030 (est.)": "1M qubits físicos → 1000 qubits lógicos; QEC a escala",
    "2033 (est.)": "IBM 'Quantum-centric supercomputing' vision",
    "2035+ (est.)": "Shor a RSA-2048: requiere ~4M qubits físicos",
}

for año, evento in timeline.items():
    print(f"  {año:<20} {evento}")

print(f"""
Estado actual (2025):
  • NISQ era (Noisy Intermediate-Scale Quantum) — en curso
  • Fault-tolerant era — comenzando (demostraciones en laboratorio)
  • Ventaja cuántica práctica en aplicaciones reales — próximos 5-10 años

NISQ vs Fault-Tolerant:
  NISQ: qubits físicos ruidosos, sin corrección de errores,
        circuitos cortos (<1000 puertas)
  FT: qubits lógicos protegidos, circuitos de millones de puertas,
      requiere ~1000x más qubits físicos
""")

print("\n✓ Módulo 13 completado — outputs/m13_hardware_comparacion.png generado")
```

---

## Checkpoint M13

```python
# checkpoint_m13.py
"""Verificaciones del Módulo 13 — Hardware Cuántico"""

def test_t2_coherencia():
    """T2 <= 2*T1 (principio físico fundamental)."""
    datos = {
        "superconductor_ibm": {"T1": 100, "T2": 100},
        "trampa_iones": {"T1": 1e6, "T2": 1e6},
    }
    for nombre, vals in datos.items():
        assert vals["T2"] <= 2 * vals["T1"], f"Violación T2<=2T1 en {nombre}"
    print("✓ test_t2_coherencia")

def test_fidelidades_razonables():
    """Fidelidades conocidas del hardware actual."""
    fidelidades = {
        "IBM_superconductor_1Q": 99.9,
        "IBM_superconductor_2Q": 99.5,
        "IonQ_2Q": 99.5,
        "Quantinuum_2Q": 99.9,
    }
    for nombre, fid in fidelidades.items():
        assert 90 <= fid <= 100, f"Fidelidad fuera de rango: {nombre}={fid}"
        assert fid >= 90, f"Fidelidad demasiado baja: {nombre}"
    print("✓ test_fidelidades_razonables")

def test_temperatura_operacion():
    """Superconductores más fríos que átomos neutros."""
    T_superconductor = 0.015   # K
    T_neutros = 1e-6           # K (doppler: μK)
    T_ambiente = 300           # K (fotónico)
    # Fotónico es más caliente que superconductor
    assert T_ambiente > T_superconductor
    # Átomos neutros (μK) son más fríos que superconductores (mK)
    assert T_neutros < T_superconductor
    print("✓ test_temperatura_operacion")

def test_profundidad_util_fidelidad():
    """Mayor fidelidad → mayor profundidad útil del circuito."""
    import numpy as np
    def max_depth_50pct(fid):
        return np.log(0.5) / (np.log(fid) * 0.5)

    f_baja = 0.99
    f_alta = 0.999
    assert max_depth_50pct(f_alta) > max_depth_50pct(f_baja), \
        "Mayor fidelidad debe permitir mayor profundidad"
    print("✓ test_profundidad_util_fidelidad")

def test_quantum_volume_concepto():
    """QV mide circuitos cuadrados nxn exitosos."""
    import math
    def quantum_volume(n):
        return 2**n
    # QV crece exponencialmente con n
    assert quantum_volume(10) == 1024
    assert quantum_volume(13) == 8192   # Quantinuum H1
    assert quantum_volume(9) == 512     # IBM Eagle
    print("✓ test_quantum_volume_concepto")

def test_qubits_logicos_vs_fisicos():
    """Para QEC, 1 qubit lógico requiere muchos físicos."""
    # Código de superficie: overhead ~1000x para p=0.1%
    qubits_logicos_objetivo = 1000
    overhead_qec = 1000   # qubits físicos por lógico (conservador)
    qubits_fisicos_necesarios = qubits_logicos_objetivo * overhead_qec
    assert qubits_fisicos_necesarios == 1_000_000, \
        f"Se necesitan {qubits_fisicos_necesarios} qubits físicos"
    print(f"✓ test_qubits_logicos_vs_fisicos ({qubits_fisicos_necesarios:,} físicos para {qubits_logicos_objetivo} lógicos)")

def test_all_to_all_vs_nearest_neighbor():
    """Iones tienen conectividad all-to-all, superconductores solo vecinos."""
    conectividad = {
        "superconductor": "nearest_neighbor",
        "trampa_iones": "all_to_all",
        "neutral_atoms": "flexible",
        "fotonico": "reconfigurable",
    }
    assert conectividad["trampa_iones"] == "all_to_all"
    assert conectividad["superconductor"] == "nearest_neighbor"
    print("✓ test_all_to_all_vs_nearest_neighbor")

if __name__ == "__main__":
    print("Ejecutando checkpoint M13 — Hardware Cuántico\n")
    test_t2_coherencia()
    test_fidelidades_razonables()
    test_temperatura_operacion()
    test_profundidad_util_fidelidad()
    test_quantum_volume_concepto()
    test_qubits_logicos_vs_fisicos()
    test_all_to_all_vs_nearest_neighbor()
    print("\n✓ Todos los tests de M13 pasaron")
```
