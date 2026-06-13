# Módulo 32 — Criptografía Cuántica: QKD y Protocolo BB84

**Objetivo:** Implementar distribución cuántica de claves (QKD) usando el protocolo BB84 y entender por qué la física cuántica garantiza seguridad incondicional. Simular ataques de intercepción y cuantificar la tasa de error que introduce un espía.

**Herramientas:** PennyLane 0.39+, NumPy, matplotlib
**Prerequisito:** M02 (Qubits), M03 (Puertas)

---

## Ejecución

```bash
conda activate quantum
mkdir -p outputs
python seccion_X.py
```

---

## 1. Fundamentos de QKD

```python
# seccion_01_fundamentos.py
"""
QKD (Quantum Key Distribution):
Permite a dos partes (Alice y Bob) establecer una clave secreta compartida
cuya seguridad está garantizada por las leyes de la física cuántica.

Principio fundamental:
  "Medir un estado cuántico desconocido lo perturba irreversiblemente"
  (Teorema de no-clonación + colapso de la función de onda)

Protocolo BB84 (Bennett & Brassard 1984):
  1. Alice prepara qubits en 4 estados posibles (2 bases)
  2. Bob mide en base aleatoria
  3. Sifteo: Alice y Bob revelan qué base usaron (canal público)
  4. Solo conservan los bits donde usaron la misma base
  5. Estimación de error: si hay espía → QBER > 0

Bases:
  Z (computacional): {|0⟩, |1⟩}           → bits {0, 1}
  X (Hadamard):      {|+⟩, |−⟩} = {H|0⟩, H|1⟩} → bits {0, 1}
"""

print("=== QKD — Protocolo BB84 ===\n")

print("Estados cuánticos del protocolo BB84:")
print("  Base Z (|): ")
print("    |0⟩ = [1,0]  → bit 0")
print("    |1⟩ = [0,1]  → bit 1")
print("  Base X (─): ")
print("    |+⟩ = (|0⟩+|1⟩)/√2  → bit 0")
print("    |−⟩ = (|0⟩-|1⟩)/√2  → bit 1")

print("\nCombinaciones base-medición:")
print("  Misma base   → resultado determinista (bit correcto)")
print("  Base diferente → resultado aleatorio  (50% correcto)")

print("\nFlujo BB84:")
pasos = [
    ("1. Alice prepara",  "N qubits, base y bit aleatorios"),
    ("2. Canal cuántico", "Qubits enviados a Bob"),
    ("3. Bob mide",       "En base aleatoria independiente"),
    ("4. Sifting",        "Revelar bases por canal público (sin bits)"),
    ("5. Clave raw",      "Conservar bits donde bases coincidieron (~N/2)"),
    ("6. QBER test",      "Comparar subconjunto para estimar errores"),
    ("7. Key distillation","Privacy amplification + reconciliación"),
    ("8. Clave final",    "Shared secret key"),
]
for paso, desc in pasos:
    print(f"  {paso}: {desc}")

print("\nSeguridad:")
print("  Sin espía: QBER ≈ 0% (solo ruido del canal)")
print("  Con espía: QBER ≈ 25% (por medir en base incorrecta ~50% y perturbar)")
print("  Umbral:    Si QBER > 11% → abortar protocolo")
```

---

## 2. Implementación BB84 con simulación cuántica

```python
# seccion_02_bb84.py
import numpy as np
import pennylane as qml

dev = qml.device("default.qubit", wires=1)

def preparar_qubit_bb84(bit, base):
    """
    Prepara qubit según protocolo BB84.
    bit:  0 o 1
    base: 'Z' o 'X'
    """
    @qml.qnode(dev)
    def estado():
        if bit == 1:
            qml.PauliX(wires=0)  # |1⟩
        if base == "X":
            qml.Hadamard(wires=0)  # Z→X
        return qml.state()
    return estado()

def medir_qubit_bb84(estado_in, base_bob):
    """
    Mide qubit en base dada. Retorna bit medido.
    """
    @qml.qnode(qml.device("default.qubit", wires=1))
    def medicion():
        qml.QubitStateVector(estado_in, wires=0)
        if base_bob == "X":
            qml.Hadamard(wires=0)  # cambiar a base X antes de medir
        return qml.probs(wires=0)
    probs = medicion()
    return int(np.random.choice([0, 1], p=[probs[0], probs[1]]))

def bb84_sin_espía(n_bits=200, seed=42):
    """
    Simulación BB84 sin espía.
    Retorna: clave sifted, QBER.
    """
    rng = np.random.RandomState(seed)
    bits_alice  = rng.randint(0, 2, n_bits)
    bases_alice = rng.choice(["Z", "X"], n_bits)
    bases_bob   = rng.choice(["Z", "X"], n_bits)

    bits_bob = np.zeros(n_bits, dtype=int)
    for i in range(n_bits):
        sv = preparar_qubit_bb84(bits_alice[i], bases_alice[i])
        bits_bob[i] = medir_qubit_bb84(sv, bases_bob[i])

    # Sifting: conservar solo donde bases coinciden
    mask_sifted = bases_alice == bases_bob
    key_alice = bits_alice[mask_sifted]
    key_bob   = bits_bob[mask_sifted]

    # QBER en subconjunto de comprobación
    n_test = min(20, len(key_alice))
    idx_test = rng.choice(len(key_alice), n_test, replace=False)
    errores  = np.sum(key_alice[idx_test] != key_bob[idx_test])
    qber     = errores / n_test

    # Clave final: bits no usados en test
    idx_final = [i for i in range(len(key_alice)) if i not in idx_test]
    clave_final = key_alice[idx_final]

    return {
        "n_bits_enviados":  n_bits,
        "n_sifted":         len(key_alice),
        "n_clave_final":    len(clave_final),
        "qber":             qber,
        "clave_alice":      clave_final,
        "coincidencias":    np.sum(key_alice[idx_final] == key_bob[idx_final]),
        "tasa_sifting":     len(key_alice)/n_bits,
    }

# Demostración sin espía
print("=== BB84 sin espía ===")
resultado = bb84_sin_espía(n_bits=100)
print(f"Bits enviados:         {resultado['n_bits_enviados']}")
print(f"Bits tras sifting:     {resultado['n_sifted']} "
      f"({resultado['tasa_sifting']:.1%} — esperado ~50%)")
print(f"QBER:                  {resultado['qber']:.2%} (esperado ~0%)")
print(f"Clave final (bits):    {resultado['n_clave_final']}")
print(f"Primeros 20 bits:      {''.join(map(str, resultado['clave_alice'][:20]))}")
```

---

## 3. Ataque de intercepción (Intercept-Resend)

```python
# seccion_03_ataque_eve.py
"""
Ataque intercept-resend (Eve):
  Eve intercepta cada qubit, mide en base aleatoria, reenvía a Bob.

  Análisis:
  - Eve escoge base correcta 50% de las veces
  - Cuando base incorrecta: introduce error con P=50%
  - Probabilidad de error por qubit: 0.5 × 0.5 = 0.25
  - QBER teórico con Eve: 25%
"""
import numpy as np
import pennylane as qml
from seccion_02_bb84 import preparar_qubit_bb84, medir_qubit_bb84

dev = qml.device("default.qubit", wires=1)

def bb84_con_eve(n_bits=200, seed=42, fraccion_interceptada=1.0):
    """
    BB84 con espía (Eve) que intercepta fracción de qubits.
    fraccion_interceptada: fracción de qubits que Eve intercepta
    """
    rng = np.random.RandomState(seed)
    bits_alice  = rng.randint(0, 2, n_bits)
    bases_alice = rng.choice(["Z", "X"], n_bits)
    bases_bob   = rng.choice(["Z", "X"], n_bits)
    bases_eve   = rng.choice(["Z", "X"], n_bits)

    bits_bob = np.zeros(n_bits, dtype=int)
    bits_eve = np.zeros(n_bits, dtype=int)

    for i in range(n_bits):
        sv_alice = preparar_qubit_bb84(bits_alice[i], bases_alice[i])

        if rng.rand() < fraccion_interceptada:
            # Eve intercepta
            bit_eve = medir_qubit_bb84(sv_alice, bases_eve[i])
            bits_eve[i] = bit_eve
            # Eve reenvía a Bob su propio qubit preparado
            sv_enviado = preparar_qubit_bb84(bit_eve, bases_eve[i])
        else:
            sv_enviado = sv_alice

        bits_bob[i] = medir_qubit_bb84(sv_enviado, bases_bob[i])

    # Sifting
    mask_sifted = bases_alice == bases_bob
    key_alice = bits_alice[mask_sifted]
    key_bob   = bits_bob[mask_sifted]

    # QBER
    n_test = min(30, len(key_alice))
    idx_test = rng.choice(len(key_alice), n_test, replace=False)
    errores  = np.sum(key_alice[idx_test] != key_bob[idx_test])
    qber     = errores / n_test

    return {
        "qber": qber,
        "n_sifted": len(key_alice),
        "n_clave_final": len(key_alice) - n_test,
    }

print("=== Impacto del Espía en QBER ===")
print(f"\n{'Fracción interceptada':>25} {'QBER (10 runs)':>15} {'Media':>8}")
print("-" * 55)

np.random.seed(0)
for frac in [0.0, 0.1, 0.25, 0.5, 0.75, 1.0]:
    qbers = [bb84_con_eve(n_bits=200, seed=i, fraccion_interceptada=frac)["qber"]
             for i in range(10)]
    media = np.mean(qbers)
    std   = np.std(qbers)
    deteccion = "⚠ DETECTADO" if media > 0.11 else "✓ seguro"
    print(f"{frac:>25.0%} {str([f'{q:.2f}' for q in qbers[:3]])+'...':>15} "
          f"{media:>6.3f} ± {std:.3f} {deteccion}")

print(f"\nUmbral BB84: QBER > 11% → presencia de espía detectada")
print(f"QBER teórico con Eve (100%): 25%")
```

---

## 4. Protocolo E91 (entrelazamiento)

```python
# seccion_04_e91.py
"""
Protocolo E91 (Ekert 1991):
Usa pares entrelazados Bell en lugar de qubits individuales.

Fuente emite: |Φ+⟩ = (|00⟩ + |11⟩)/√2
Alice y Bob miden en ángulos aleatorios {0°, 45°, 90°} y {45°, 90°, 135°}
Verifican desigualdad de Bell (CHSH): S > 2 → canales no comprometidos

Ventaja sobre BB84:
- No requiere fuente confiable del lado de Alice
- La fuente puede estar en manos de tercero no confiable
- Seguridad basada en desigualdad de Bell (violación = no hay espía)
"""
import numpy as np
import pennylane as qml

dev_2 = qml.device("default.qubit", wires=2)

def par_bell():
    """Estado de Bell |Φ+⟩ = (|00⟩+|11⟩)/√2."""
    @qml.qnode(dev_2)
    def estado():
        qml.Hadamard(wires=0)
        qml.CNOT(wires=[0, 1])
        return qml.state()
    return estado()

def medir_en_angulo(sv, qubit, theta):
    """
    Mide qubit en ángulo θ en el plano XZ.
    Equivalent a: Ry(-θ)|ψ⟩ → medir en Z
    """
    @qml.qnode(qml.device("default.qubit", wires=2))
    def circuit():
        qml.QubitStateVector(sv, wires=[0, 1])
        qml.RY(-theta, wires=qubit)
        return qml.probs(wires=qubit)
    probs = circuit()
    return int(np.random.choice([0, 1], p=[probs[0], probs[1]]))

def correlacion_e91(theta_a, theta_b, n=1000):
    """
    Correlación ⟨AB⟩ entre mediciones de Alice (θ_a) y Bob (θ_b).
    Para |Φ+⟩: ⟨AB⟩ = cos(θ_a - θ_b)
    """
    sv = par_bell()
    pares = 0
    for _ in range(n):
        sv_fresh = par_bell()
        a = 1 - 2*medir_en_angulo(sv_fresh, 0, theta_a)  # ±1
        b = 1 - 2*medir_en_angulo(sv_fresh, 1, theta_b)  # ±1
        pares += a * b
    return pares / n

# Ángulos del protocolo E91
angulos_alice = [0, np.pi/4, np.pi/2]        # 0°, 45°, 90°
angulos_bob   = [np.pi/4, np.pi/2, 3*np.pi/4]  # 45°, 90°, 135°

print("=== Protocolo E91 — Correlaciones Bell ===\n")
print("Correlaciones teóricas para |Φ+⟩: ⟨AB⟩ = cos(θ_A - θ_B)")
print(f"\n{'θ_A':>8} {'θ_B':>8} {'⟨AB⟩ teórico':>15} {'⟨AB⟩ simulado':>16}")
print("-" * 55)

correlaciones = {}
for ta in angulos_alice:
    for tb in angulos_bob:
        teorico   = np.cos(ta - tb)
        simulado  = correlacion_e91(ta, tb, n=200)
        key = f"({ta*180/np.pi:.0f},{tb*180/np.pi:.0f})"
        correlaciones[key] = simulado
        print(f"{ta*180/np.pi:>8.0f}° {tb*180/np.pi:>8.0f}° "
              f"{teorico:>15.4f} {simulado:>16.4f}")

# Parámetro CHSH S = |E(a,b) - E(a,b') + E(a',b) + E(a',b')|
# Para a=0°,a'=90°, b=45°,b'=135°
ta1, ta2 = 0, np.pi/2  # Alice: 0° y 90°
tb1, tb2 = np.pi/4, 3*np.pi/4  # Bob: 45° y 135°

E_ab   = correlacion_e91(ta1, tb1, n=500)
E_ab2  = correlacion_e91(ta1, tb2, n=500)
E_a2b  = correlacion_e91(ta2, tb1, n=500)
E_a2b2 = correlacion_e91(ta2, tb2, n=500)

S_chsh = abs(E_ab - E_ab2 + E_a2b + E_a2b2)

print(f"\nParámetro CHSH S = |E(a,b) - E(a,b') + E(a',b) + E(a',b')|")
print(f"  S = {S_chsh:.4f}")
print(f"  Límite clásico (HV): S ≤ 2")
print(f"  Límite cuántico:     S ≤ 2√2 ≈ {2*np.sqrt(2):.4f}")
print(f"  Violación de Bell: {'SÍ ✓' if S_chsh > 2 else 'NO ✗'}")
print(f"  Canal no comprometido: {'SÍ — comunicación segura' if S_chsh > 2 else 'Posible espía'}")
```

---

## 5. Privacy Amplification y reconciliación

```python
# seccion_05_privacy_amplification.py
"""
Después del sifting, la clave raw puede tener:
1. Errores de canal (QBER pequeño)
2. Información parcial del espía

Pasos de destilación de clave:
1. Error Correction: reconciliación para hacer las claves iguales
   - Cascade protocol: O(QBER·n) bits de comunicación pública
   - LDPC codes: más eficiente

2. Privacy Amplification: reducir información del espía
   - Universal hash function: h: {0,1}^n → {0,1}^m
   - m = n - n·h(QBER) - n·h(e) donde h es la entropía binaria
   - Resultado: espía tiene información ≈ 0 sobre la clave final
"""
import numpy as np
import hashlib

def entropia_binaria(p):
    """h(p) = -p·log2(p) - (1-p)·log2(1-p)."""
    if p <= 0 or p >= 1:
        return 0
    return -p * np.log2(p) - (1-p) * np.log2(1-p)

def longitud_clave_segura(n_bits_raw, qber, epsilon=0.01):
    """
    Longitud de clave segura después de privacy amplification.
    Fórmula simplificada (composición universal):
    m ≈ n(1 - h(QBER) - h(QBER)) - log2(2/ε)
    """
    # Tasa de clave (Shor-Preskill):
    # r = 1 - h(QBER) - h(QBER) para BB84 con corrección de paridad
    r = 1 - 2 * entropia_binaria(qber)
    m = max(0, int(n_bits_raw * r) - int(np.log2(2/epsilon) + 1))
    return m

def privacy_amplification(clave_raw, m):
    """
    Privacy amplification usando hash criptográfico.
    Implementación simplificada (no criptográficamente rigurosa).
    """
    if m <= 0 or len(clave_raw) == 0:
        return np.array([], dtype=int)

    # Convertir bits a bytes
    n = len(clave_raw)
    padding = (8 - n % 8) % 8
    bits_padded = np.concatenate([clave_raw, np.zeros(padding, dtype=int)])
    bytes_clave = np.packbits(bits_padded)

    # Hash SHA-256 y tomar primeros m bits
    h = hashlib.sha256(bytes_clave.tobytes()).digest()
    bits_hash = np.unpackbits(np.frombuffer(h, dtype=np.uint8))
    return bits_hash[:m]

# Demostración
print("=== Privacy Amplification ===\n")

qber_values = [0.0, 0.05, 0.10, 0.11, 0.15, 0.20, 0.25]
n_raw = 1000

print(f"{'QBER':>8} {'h(QBER)':>10} {'Tasa clave':>12} {'Bits finales':>14} {'Estado':>12}")
print("-" * 60)
for qber in qber_values:
    h = entropia_binaria(qber)
    r = max(0, 1 - 2*h)
    m = longitud_clave_segura(n_raw, qber)
    estado = "SEGURO" if qber <= 0.11 else "ABORTAR"
    print(f"{qber:>8.2%} {h:>10.4f} {r:>12.4f} {m:>14d} {estado:>12}")

print(f"\nDemostración privacy amplification con n=50 bits, QBER=5%:")
np.random.seed(42)
clave_raw = np.random.randint(0, 2, 50)
m_seguro  = longitud_clave_segura(50, 0.05)
print(f"  Clave raw ({len(clave_raw)} bits): {''.join(map(str, clave_raw))}")
clave_final = privacy_amplification(clave_raw, m_seguro)
print(f"  Clave final ({m_seguro} bits): {''.join(map(str, clave_final))}")
print(f"  Reducción: {len(clave_raw)} → {len(clave_final)} bits (extracción: {m_seguro/50:.1%})")
```

---

## 6. Visualización completa BB84

```python
# seccion_06_visualizacion.py
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from seccion_02_bb84 import bb84_sin_espía
from seccion_03_ataque_eve import bb84_con_eve
from seccion_05_privacy_amplification import (
    entropia_binaria, longitud_clave_segura, privacy_amplification
)

fig, axes = plt.subplots(2, 3, figsize=(15, 10))
fig.suptitle("BB84 — Distribución Cuántica de Claves", fontsize=14)

# Panel 1: Diagrama del protocolo BB84
ax = axes[0, 0]
ax.axis("off")
ax.set_title("Protocolo BB84", fontsize=11)
texto_protocolo = """
ALICE                    BOB
──────                   ───
Base:   Z X Z Z X X Z Z
Bit:    0 0 1 1 0 1 0 0
         ↓↓↓↓↓↓↓↓ (canal cuántico)
Base:   X X Z Z X Z X Z
Mide:   ? 0 1 1 0 ? ? 0
         ↕↕ sifting (canal público)
Base A: X   Z Z X       Z
Base B:   X Z Z X     X Z
Iguales:  ✗ ✓ ✓ ✓     ✗ ✓
Key:        1 1 0         0
"""
ax.text(0.05, 0.95, texto_protocolo, transform=ax.transAxes,
        fontfamily="monospace", fontsize=9, va="top")

# Panel 2: Impacto de Eve en QBER
ax = axes[0, 1]
fracciones = np.linspace(0, 1, 11)
qbers_sim = []
qbers_teo = [0.25 * f for f in fracciones]

np.random.seed(42)
for f in fracciones:
    qs = [bb84_con_eve(n_bits=100, seed=s, fraccion_interceptada=f)["qber"] for s in range(5)]
    qbers_sim.append(np.mean(qs))

ax.plot(fracciones*100, qbers_sim, "b.-", linewidth=2, markersize=8, label="Simulado")
ax.plot(fracciones*100, qbers_teo, "r--",  linewidth=2, label="Teórico (25%·f)")
ax.axhline(0.11, color="orange", linestyle="--", linewidth=1.5, label="Umbral 11%")
ax.fill_between(fracciones*100, 0.11, 0.3, alpha=0.1, color="red",
                label="Zona de detección")
ax.set_xlabel("% Qubits interceptados por Eve")
ax.set_ylabel("QBER")
ax.set_title("QBER vs Fracción Interceptada")
ax.legend(fontsize=8); ax.grid(alpha=0.3)
ax.set_ylim([0, 0.32])

# Panel 3: Tasa de generación de clave segura
ax = axes[0, 2]
qbers = np.linspace(0, 0.25, 50)
tasas = [max(0, 1 - 2*entropia_binaria(q)) for q in qbers]
ax.plot(qbers*100, tasas, "b-", linewidth=2)
ax.axvline(11, color="red", linestyle="--", linewidth=1.5, label="Umbral 11%")
ax.fill_between(qbers*100, tasas, 0, alpha=0.2, color="blue", label="Clave útil")
ax.set_xlabel("QBER (%)")
ax.set_ylabel("Tasa de clave segura")
ax.set_title("Tasa de Clave vs QBER (Shor-Preskill)")
ax.legend(fontsize=8); ax.grid(alpha=0.3)
ax.set_ylim([0, 1.05])

# Panel 4: Distribución de bases y bits (100 runs)
ax = axes[1, 0]
res = bb84_sin_espía(n_bits=1000)
ax.bar(["Enviados", "Sifted", "Clave final"],
       [1000, res["n_sifted"], res["n_clave_final"]],
       color=["steelblue", "darkorange", "green"],
       edgecolor="black")
for v, h in zip([1000, res["n_sifted"], res["n_clave_final"]],
                ["", f"({res['n_sifted']/10:.0f}%)", f"({res['n_clave_final']/10:.0f}%)"]):
    ax.text(list([0,1,2])[[v==1000, v==res["n_sifted"], v==res["n_clave_final"]].index(True)],
            v + 10, f"{v}\n{h}", ha="center", fontsize=9)
ax.set_ylabel("Número de bits")
ax.set_title("Reducción de Bits (sin espía)")
ax.grid(axis="y", alpha=0.3)

# Panel 5: Privacy amplification - tasa vs QBER
ax = axes[1, 1]
qbers_pa = np.linspace(0, 0.12, 50)
n_bits_pa = 1000
clave_finals = [longitud_clave_segura(n_bits_pa, q) for q in qbers_pa]
ax.plot(qbers_pa*100, [c/n_bits_pa for c in clave_finals], "b-", linewidth=2)
ax.axvline(11, color="red", linestyle="--", linewidth=1.5, label="Umbral")
ax.set_xlabel("QBER (%)")
ax.set_ylabel("Eficiencia (bits finales / bits raw)")
ax.set_title("Eficiencia Privacy Amplification")
ax.legend(fontsize=8); ax.grid(alpha=0.3)
ax.set_ylim([0, 1.05])

# Panel 6: Comparación de protocolos QKD
ax = axes[1, 2]
protocolos = ["BB84\n(1984)", "E91\n(1991)", "B92\n(1992)", "SARG04\n(2004)"]
eficiencias = [0.50, 0.25, 0.25, 0.50]  # fracción de bits usados
seguridad  = [8, 9, 8, 9]  # escala 1-10
colores    = ["steelblue", "darkorange", "green", "purple"]

ax2_twin = ax.twinx()
bars = ax.bar(protocolos, eficiencias, color=colores, alpha=0.7, edgecolor="black")
ax2_twin.plot(protocolos, seguridad, "k.-", linewidth=2, markersize=12, label="Nivel seguridad")
ax.set_ylabel("Eficiencia de clave")
ax2_twin.set_ylabel("Nivel de seguridad (1-10)")
ax2_twin.set_ylim([0, 12])
ax.set_title("Protocolos QKD — Comparativa")
ax.grid(axis="y", alpha=0.3)
ax2_twin.legend(loc="upper right", fontsize=8)

plt.tight_layout()
plt.savefig("outputs/m32_qkd.png", dpi=120, bbox_inches="tight")
print("Figura guardada: outputs/m32_qkd.png")
plt.close()
```

---

## Checkpoint

```python
# checkpoint_m32.py
"""Tests para Módulo 32 — QKD y BB84."""
import numpy as np
import pennylane as qml

dev = qml.device("default.qubit", wires=1)

def preparar_bb84(bit, base):
    @qml.qnode(dev)
    def estado():
        if bit == 1: qml.PauliX(wires=0)
        if base == "X": qml.Hadamard(wires=0)
        return qml.state()
    return estado()

def medir_bb84(sv, base):
    @qml.qnode(qml.device("default.qubit", wires=1))
    def circ():
        qml.QubitStateVector(sv, wires=0)
        if base == "X": qml.Hadamard(wires=0)
        return qml.probs(wires=0)
    p = circ()
    return int(np.random.choice([0, 1], p=p))

# T1: Misma base → medición determinista
def test_misma_base_determinista():
    for bit in [0, 1]:
        for base in ["Z", "X"]:
            sv = preparar_bb84(bit, base)
            resultados = [medir_bb84(sv, base) for _ in range(20)]
            assert all(r == bit for r in resultados), \
                f"Medir en misma base da resultado estocástico: bit={bit}, base={base}"
    print("T1 PASS: misma base produce resultado determinista")

# T2: Bases distintas → resultado aleatorio (50%)
def test_bases_distintas_aleatorio():
    np.random.seed(42)
    n = 200
    sv_Z0 = preparar_bb84(0, "Z")
    resultados = [medir_bb84(sv_Z0, "X") for _ in range(n)]
    p_0 = resultados.count(0) / n
    assert 0.35 < p_0 < 0.65, f"P(0) con base X en |0⟩_Z = {p_0:.2f} (esperado ~0.5)"
    print(f"T2 PASS: bases distintas → P(0) = {p_0:.3f} ≈ 0.5")

# T3: Protocolo BB84 sin espía → QBER ≈ 0
def test_qber_sin_espia():
    from seccion_02_bb84 import bb84_sin_espía
    res = bb84_sin_espía(n_bits=200, seed=99)
    assert res["qber"] < 0.15, f"QBER sin espía = {res['qber']:.2%} (esperado ~0%)"
    print(f"T3 PASS: QBER sin espía = {res['qber']:.2%} < 15%")

# T4: Protocolo BB84 con Eve → QBER > umbral
def test_qber_con_eve():
    from seccion_03_ataque_eve import bb84_con_eve
    qbers = [bb84_con_eve(n_bits=100, seed=s, fraccion_interceptada=1.0)["qber"]
             for s in range(5)]
    qber_medio = np.mean(qbers)
    assert qber_medio > 0.10, f"QBER con Eve = {qber_medio:.2%} (esperado ~25%)"
    print(f"T4 PASS: QBER con Eve = {qber_medio:.2%} > 10%")

# T5: Entropía binaria h(0)=0, h(0.5)=1, h(0.11)≈0.5
def test_entropia_binaria():
    from seccion_05_privacy_amplification import entropia_binaria
    assert abs(entropia_binaria(0.0) - 0.0) < 1e-6
    assert abs(entropia_binaria(0.5) - 1.0) < 1e-6
    h11 = entropia_binaria(0.11)
    assert 0.4 < h11 < 0.6, f"h(0.11) = {h11:.4f}"
    print(f"T5 PASS: h(0)={entropia_binaria(0)}, h(0.5)={entropia_binaria(0.5)}, h(0.11)={h11:.4f}")

# T6: Privacy amplification produce clave más corta
def test_pa_reduce_longitud():
    from seccion_05_privacy_amplification import privacy_amplification, longitud_clave_segura
    clave_raw = np.random.randint(0, 2, 100)
    m = longitud_clave_segura(100, 0.05)
    clave_final = privacy_amplification(clave_raw, m)
    assert len(clave_final) == m
    assert m < len(clave_raw), f"PA no redujo longitud: {len(clave_raw)} → {m}"
    print(f"T6 PASS: PA reduce {len(clave_raw)} bits → {m} bits")

# T7: Violación de Bell para estado |Φ+⟩
def test_violacion_bell():
    dev2 = qml.device("default.qubit", wires=2)

    @qml.qnode(dev2)
    def correlacion(ta, tb):
        qml.Hadamard(0)
        qml.CNOT(wires=[0, 1])
        qml.RY(-ta, wires=0)
        qml.RY(-tb, wires=1)
        return qml.expval(qml.PauliZ(0) @ qml.PauliZ(1))

    # Ángulos CHSH óptimos
    ta1, ta2 = 0, np.pi/2
    tb1, tb2 = np.pi/4, 3*np.pi/4
    S = abs(correlacion(ta1, tb1) - correlacion(ta1, tb2) +
            correlacion(ta2, tb1) + correlacion(ta2, tb2))
    assert S > 2.0, f"CHSH S = {S:.4f} ≤ 2 (no hay violación de Bell)"
    print(f"T7 PASS: CHSH S = {S:.4f} > 2 (violación de Bell confirmada)")

if __name__ == "__main__":
    print("=== Checkpoint M32: QKD y BB84 ===\n")
    test_misma_base_determinista()
    test_bases_distintas_aleatorio()
    test_qber_sin_espia()
    test_qber_con_eve()
    test_entropia_binaria()
    test_pa_reduce_longitud()
    test_violacion_bell()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M33 — Criptografía Post-Cuántica — Estándares NIST 2024](./modulo-33-post-cuantica.md)
