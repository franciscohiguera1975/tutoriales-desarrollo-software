# Módulo 33 — Criptografía Post-Cuántica: Estándares NIST 2024

**Objetivo:** Entender por qué los algoritmos criptográficos actuales (RSA, ECDH, DSA) son vulnerables a computadoras cuánticas, y explorar los estándares post-cuánticos finalizados por NIST en 2024: ML-KEM (CRYSTALS-Kyber), ML-DSA (CRYSTALS-Dilithium) y SLH-DSA (SPHINCS+).

**Herramientas:** Python puro, NumPy, matplotlib, cryptography (PyPI)
**Prerequisito:** M11 (Shor), M32 (QKD)

---

## Setup

```bash
conda activate quantum
pip install cryptography  # Para RSA/ECDH clásico de referencia
```

---

## 1. Por qué la computación cuántica rompe la criptografía actual

```python
# seccion_01_amenaza_cuantica.py
"""
Impacto de las computadoras cuánticas en criptografía:

VULNERABLES:
  RSA: basado en factorización (Shor 1994) → O(log³n) cuántico vs O(e^(n^1/3)) clásico
  ECC: basado en logaritmo discreto (Shor) → misma vulnerabilidad
  DH:  Diffie-Hellman → logaritmo discreto → vulnerable

NO VULNERABLES (por ahora):
  AES-256:   búsqueda exhaustiva → Grover reduce a 2^128 (aún seguro)
  SHA-3:     resistencia de colisión → Grover afecta pero con doble tamaño es seguro
  One-time pad: matemáticamente perfecto

Horizonte temporal:
  ~2035-2040: computadoras cuánticas tolerantes a errores con >1M qubits
  Ya hoy: "harvest now, decrypt later" — adversarios guardan tráfico cifrado
"""

import numpy as np
import time

print("=== Amenaza Cuántica a la Criptografía Actual ===\n")

# Estimación del impacto del algoritmo de Shor
print("Impacto del Algoritmo de Shor:")
print(f"{'Tamaño de clave':>20} {'Bits seg. clásica':>20} {'Qubits para Shor':>18}")
print("-" * 62)

clave_data = [
    ("RSA-1024",  1024, 80,  "~2048 qubits lógicos"),
    ("RSA-2048",  2048, 112, "~4096 qubits lógicos"),
    ("RSA-3072",  3072, 128, "~6144 qubits lógicos"),
    ("ECC-256",   256,  128, "~2330 qubits lógicos"),
    ("ECC-384",   384,  192, "~3484 qubits lógicos"),
]
for nombre, n_bits, seg_cl, qubits in clave_data:
    print(f"{nombre:>20} {seg_cl:>20} bits {qubits:>18}")

print("\nNota: 'qubits lógicos' = qubits tolerantes a errores")
print("Con Surface Code (overhead ×1000): RSA-2048 necesita ~4M qubits físicos")
print("Hardware actual: IBM (1000+ qubits físicos, 2023), Microsoft (12 qubits topológicos)")
print("Horizonte estimado: 2035-2040 para romper RSA-2048")

# Impacto del algoritmo de Grover
print("\nImpacto del Algoritmo de Grover (búsqueda):")
print(f"{'Algoritmo':>20} {'Seg. clásica':>15} {'Seg. cuántica':>15} {'Defensa':>25}")
print("-" * 80)
grover_data = [
    ("AES-128",  "128 bits", "64 bits",  "Usar AES-256"),
    ("AES-192",  "192 bits", "96 bits",  "Usar AES-256"),
    ("AES-256",  "256 bits", "128 bits", "Seguro post-cuántico"),
    ("SHA-256",  "128 bits", "85 bits",  "Usar SHA-384/SHA-3-256"),
    ("SHA-512",  "256 bits", "128 bits", "Seguro post-cuántico"),
]
for alg, cl, q, def_ in grover_data:
    print(f"{alg:>20} {cl:>15} {q:>15} {def_:>25}")

print("\nResumen:")
print("  AES y SHA con tamaño suficiente: seguros contra Grover")
print("  RSA y ECC: COMPLETAMENTE ROTOS por Shor")
print("  Solución: algoritmos post-cuánticos (PQC)")
```

---

## 2. Criptografía basada en retículos (Lattice-based)

```python
# seccion_02_lattice.py
"""
Criptografía basada en retículos — fundamento de ML-KEM y ML-DSA.

Problema LWE (Learning With Errors):
  Dado: A (matriz aleatoria), b = As + e (mod q)
  Difícil: encontrar s dado (A, b) incluso con computadoras cuánticas
  Fácil con s: verificar b - As ≈ 0 (mod q)

Problema MLWE (Module LWE) — Kyber:
  Más eficiente: trabaja en anillo de polinomios R_q = Z_q[x]/(x^n + 1)
  n=256, q=3329 para ML-KEM-768

Intuición geométrica:
  Un retículo es una rejilla de puntos en Z^n.
  Los problemas duros son: encontrar el vector más corto (SVP),
  o encontrar el punto del retículo más cercano a un punto dado (CVP).
"""
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

print("=== Retículos y LWE ===\n")

# === Demostración de LWE en dimensión pequeña ===
n = 4    # dimensión del secreto (pequeño para demo)
q = 97   # módulo primo

np.random.seed(42)

# Clave secreta s
s = np.array([3, 1, 4, 1])  # secreto

# Generar muestras LWE: (a_i, b_i = a_i·s + e_i mod q)
def generar_lwe(s, q, n_muestras=8, sigma=2):
    """Genera n_muestras de LWE con error Gaussiano."""
    n = len(s)
    A = np.random.randint(0, q, size=(n_muestras, n))
    e = np.round(np.random.normal(0, sigma, n_muestras)).astype(int)
    b = (A @ s + e) % q
    return A, b, e

A, b, e_verdadero = generar_lwe(s, q)
print("Instancia LWE (n=4, q=97):")
print(f"Secreto s = {s}")
print(f"\nMatriz A (pública):")
print(A)
print(f"\nVector b = As + e (mod {q}):")
print(b)
print(f"\nVerificación: A·s mod q = {(A @ s) % q}")
print(f"Error e (oculto) = {e_verdadero}")
print(f"b - A·s mod q = {(b - A @ s) % q}")

# Mostrar que b≈A·s (con error pequeño)
diferencias = (b - A @ s) % q
diferencias_centradas = np.where(diferencias > q//2, diferencias - q, diferencias)
print(f"\n|b - A·s| (errores): {diferencias_centradas}")
print(f"Norma del error: {np.linalg.norm(diferencias_centradas):.2f}")
print(f"Sin el secreto, b parece aleatorio módulo {q}")

# Visualización: retículo 2D y el problema CVP
fig, axes = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle("Criptografía de Retículos — Fundamentos", fontsize=13)

# Retículo 2D: puntos de la forma m1*v1 + m2*v2
ax = axes[0]
v1 = np.array([3, 0])
v2 = np.array([1, 4])
puntos = []
for m1 in range(-4, 5):
    for m2 in range(-4, 5):
        p = m1*v1 + m2*v2
        if -12 <= p[0] <= 12 and -12 <= p[1] <= 12:
            puntos.append(p)
puntos = np.array(puntos)
ax.scatter(puntos[:, 0], puntos[:, 1], c="steelblue", s=30, zorder=3)

# Punto objetivo (no en el retículo) y vector más cercano
target = np.array([5.5, 6.2])
distancias = np.linalg.norm(puntos - target, axis=1)
mas_cercano = puntos[np.argmin(distancias)]
ax.scatter(*target, c="red", s=200, marker="*", zorder=5, label="Objetivo t")
ax.scatter(*mas_cercano, c="orange", s=200, marker="D", zorder=5, label="CVP(t)")
ax.annotate("", xy=mas_cercano, xytext=target,
            arrowprops=dict(arrowstyle="->", color="orange", lw=2))
ax.set_xlim(-8, 8); ax.set_ylim(-8, 8)
ax.set_aspect("equal"); ax.grid(alpha=0.3)
ax.set_title("Problema CVP (Closest Vector Problem)")
ax.set_xlabel("x"); ax.set_ylabel("y")
ax.legend(fontsize=9)

# LWE: distribución de b (aleatoria vs estructurada)
ax = axes[1]
np.random.seed(0)
n_samples = 1000
b_real = np.array([(np.random.randint(0, q, 4) @ s) % q for _ in range(n_samples)])
b_rand = np.random.randint(0, q, n_samples)

ax.hist(b_real, bins=30, alpha=0.6, color="steelblue", density=True, label="A·s mod q")
ax.hist(b_rand, bins=30, alpha=0.6, color="coral",    density=True, label="Aleatorio")
ax.set_xlabel("Valor b")
ax.set_ylabel("Densidad")
ax.set_title("LWE: b vs. Aleatorio\n(sin error: b=A·s parece aleatorio)")
ax.legend(fontsize=9); ax.grid(alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m33_lattice.png", dpi=120, bbox_inches="tight")
print("\nFigura guardada: outputs/m33_lattice.png")
plt.close()
```

---

## 3. ML-KEM (CRYSTALS-Kyber) — Esquema de encapsulamiento

```python
# seccion_03_mlkem.py
"""
ML-KEM (FIPS 203, 2024) = CRYSTALS-Kyber estandarizado.
Mecanismo de encapsulación de clave (KEM):
  1. KeyGen() → (pk, sk)
  2. Encaps(pk) → (ciphertext, shared_secret)
  3. Decaps(sk, ciphertext) → shared_secret

Parámetros ML-KEM-768 (nivel de seguridad NIST 3 ≈ AES-192):
  n = 256, k = 3, q = 3329
  Seguridad clásica: ~180 bits
  Seguridad cuántica: ~164 bits

Implementación simplificada (conceptual, no segura):
"""
import numpy as np
import hashlib
import os

class MLKEMSimplificado:
    """
    Implementación CONCEPTUAL de Kyber/ML-KEM.
    Solo para fines educativos — NO usar en producción.
    """
    def __init__(self, n=16, q=97, k=2, eta=2):
        """
        n: dimensión del polinomio (simplificado: 16 vs 256 real)
        q: módulo primo
        k: módulo de la dimensión del módulo (2 vs 3 o 4 real)
        eta: parámetro de ruido (distribución CBD)
        """
        self.n = n
        self.q = q
        self.k = k
        self.eta = eta

    def _cbd(self, seed, n_coefs):
        """Distribución Centered Binomial (CBD): simula ruido pequeño."""
        rng = np.random.RandomState(int.from_bytes(seed[:4], "big"))
        a = sum(rng.randint(0, 2, n_coefs) for _ in range(self.eta))
        b = sum(rng.randint(0, 2, n_coefs) for _ in range(self.eta))
        return (a - b) % self.q

    def _polimulmod(self, a, b):
        """Multiplicación de polinomios mod (x^n + 1) y mod q."""
        n, q = self.n, self.q
        result = np.zeros(n, dtype=int)
        for i in range(n):
            for j in range(n):
                sign = 1 if (i + j) < n else -1
                result[(i + j) % n] = (result[(i + j) % n] + sign * a[i] * b[j]) % q
        return result

    def keygen(self):
        """Genera par de claves (pk, sk)."""
        rng = np.random.RandomState(42)
        # Clave pública: (A, t=As+e)
        A = [[rng.randint(0, self.q, self.n) for _ in range(self.k)]
             for _ in range(self.k)]
        # Secreto s y error e
        s = [rng.randint(-self.eta, self.eta+1, self.n) % self.q
             for _ in range(self.k)]
        e = [rng.randint(-2, 3, self.n) % self.q for _ in range(self.k)]

        # t = As + e
        t = []
        for i in range(self.k):
            ti = np.zeros(self.n, dtype=int)
            for j in range(self.k):
                ti = (ti + self._polimulmod(A[i][j], s[j])) % self.q
            t.append((ti + e[i]) % self.q)

        pk = (A, t)
        sk = s
        return pk, sk

    def encaps(self, pk):
        """Encapsula clave compartida. Retorna (ciphertext, shared_secret)."""
        A, t = pk
        rng = np.random.RandomState(99)

        # Efímero r, e1, e2
        r  = [rng.randint(-self.eta, self.eta+1, self.n) % self.q for _ in range(self.k)]
        e1 = [rng.randint(-2, 3, self.n) % self.q for _ in range(self.k)]
        e2 = rng.randint(-2, 3, self.n) % self.q

        # Mensaje (aleatório)
        m = rng.randint(0, 2, self.n)
        m_scaled = (m * ((self.q + 1) // 2)) % self.q  # escalar a mod q

        # u = A^T r + e1
        u = []
        for j in range(self.k):
            uj = np.zeros(self.n, dtype=int)
            for i in range(self.k):
                uj = (uj + self._polimulmod(A[i][j], r[i])) % self.q
            u.append((uj + e1[j]) % self.q)

        # v = t^T r + e2 + m_scaled
        v = np.zeros(self.n, dtype=int)
        for i in range(self.k):
            v = (v + self._polimulmod(t[i], r[i])) % self.q
        v = (v + e2 + m_scaled) % self.q

        # Shared secret: hash de m
        shared_secret = hashlib.sha256(m.tobytes()).digest()[:32]
        return (u, v), shared_secret

    def decaps(self, sk, ciphertext):
        """Desencapsula clave compartida."""
        u, v = ciphertext
        s = sk

        # m' = v - s^T u
        m_prime = v.copy()
        for j in range(self.k):
            m_prime = (m_prime - self._polimulmod(s[j], u[j])) % self.q

        # Decodificar: si coeficiente cerca de q/2 → 1, cerca de 0 → 0
        threshold = self.q // 4
        m_decoded = np.where(
            np.abs(m_prime - self.q//2) < threshold,
            1, 0
        )
        return hashlib.sha256(m_decoded.tobytes()).digest()[:32]

# Demostración
kem = MLKEMSimplificado(n=16, q=97, k=2)

print("=== ML-KEM (Kyber simplificado) ===\n")

# KeyGen
pk, sk = kem.keygen()
print("KeyGen:")
print(f"  Clave pública pk = (A, t)  — tamaño A: {len(pk[0])}×{len(pk[0][0])}×{len(pk[0][0][0])}")
print(f"  Clave privada sk = s       — tamaño: {len(sk)}×{len(sk[0])}")

# Encaps
ct, ss_alice = kem.encaps(pk)
print(f"\nEncaps:")
print(f"  Ciphertext (u,v)  — tamaño u: {len(ct[0])}×{len(ct[0][0])}, v: {len(ct[1])}")
print(f"  Shared secret Alice: {ss_alice.hex()[:32]}...")

# Decaps
ss_bob = kem.decaps(sk, ct)
print(f"\nDecaps:")
print(f"  Shared secret Bob:  {ss_bob.hex()[:32]}...")
print(f"\n  Claves iguales: {ss_alice == ss_bob}")

# Parámetros reales de ML-KEM
print("\nParámetros oficiales ML-KEM (FIPS 203):")
print(f"{'Variante':>15} {'k':>4} {'n':>6} {'q':>6} {'Pk (bytes)':>12} {'Ct (bytes)':>12} {'Seg. cuántica':>15}")
print("-" * 70)
variantes = [
    ("ML-KEM-512",  2, 256, 3329, 800,  768,  128),
    ("ML-KEM-768",  3, 256, 3329, 1184, 1088, 164),
    ("ML-KEM-1024", 4, 256, 3329, 1568, 1568, 200),
]
for v, k, n, q, pk_bytes, ct_bytes, seg_q in variantes:
    print(f"{v:>15} {k:>4} {n:>6} {q:>6} {pk_bytes:>12} {ct_bytes:>12} {seg_q:>15}")
```

---

## 4. ML-DSA (CRYSTALS-Dilithium) — Firma digital

```python
# seccion_04_mldsa.py
"""
ML-DSA (FIPS 204, 2024) = CRYSTALS-Dilithium estandarizado.
Esquema de firma digital post-cuántico.

Operaciones:
  KeyGen() → (pk=t, sk=(rho, K, t0, s1, s2))
  Sign(sk, message) → sigma
  Verify(pk, message, sigma) → True/False

Seguridad: basada en MLWE + MSIS (Short Integer Solution)
"""
import numpy as np
import hashlib

print("=== ML-DSA (Dilithium simplificado) ===\n")

class MLDSASimplificado:
    """Implementación conceptual de Dilithium/ML-DSA."""
    def __init__(self, n=16, q=8380417, k=4, l=4, eta=2, gamma1=2**17, gamma2=95232):
        self.n = n
        self.q = q
        self.k = k
        self.l = l
        self.eta = eta
        self.gamma1 = gamma1
        self.gamma2 = gamma2

    def keygen(self, seed=None):
        rng = np.random.RandomState(seed or 42)
        # Clave: (pk_t, sk_s)
        s1 = rng.randint(-self.eta, self.eta+1, (self.l, self.n)) % self.q
        s2 = rng.randint(-self.eta, self.eta+1, (self.k, self.n)) % self.q
        # t = A·s1 + s2 (simplificado: A=I para demo)
        t  = (s1[:self.k] + s2) % self.q
        return t, (s1, s2)

    def sign(self, sk, mensaje):
        """Firma mensaje. Retorna sigma."""
        s1, s2 = sk
        # Hash del mensaje como "compromiso"
        m_hash = hashlib.sha3_256(mensaje).digest()
        # Firma simplificada: hash + s1[0] mod q
        z = (int.from_bytes(m_hash[:4], "big") + int(s1[0, 0])) % self.q
        return (m_hash, z)

    def verify(self, pk, mensaje, sigma):
        """Verifica firma."""
        m_hash, z = sigma
        m_hash_exp = hashlib.sha3_256(mensaje).digest()
        # Verificar que hash coincide (simplificado)
        return m_hash == m_hash_exp

# Demostración
dsa = MLDSASimplificado()
pk, sk = dsa.keygen()
mensaje = b"Hola mundo post-cuantico"

sigma = dsa.sign(sk, mensaje)
valido = dsa.verify(pk, mensaje, sigma)
alterado = dsa.verify(pk, b"Mensaje alterado", sigma)

print(f"Clave pública pk: shape={pk.shape}")
print(f"Mensaje: '{mensaje.decode()}'")
print(f"Firma generada: ({sigma[0].hex()[:16]}..., z={sigma[1]})")
print(f"Verificación (mensaje correcto): {valido}")
print(f"Verificación (mensaje alterado): {alterado}")

# Parámetros reales
print("\nParámetros oficiales ML-DSA (FIPS 204):")
print(f"{'Variante':>15} {'k,l':>6} {'n':>6} {'q':>10} {'Pk (bytes)':>12} {'Sig (bytes)':>13} {'Seg. cuántica':>15}")
print("-" * 78)
variantes_dsa = [
    ("ML-DSA-44",  "4,4", 256, 8380417, 1312,  2420, 128),
    ("ML-DSA-65",  "6,5", 256, 8380417, 1952,  3309, 192),
    ("ML-DSA-87",  "8,7", 256, 8380417, 2592,  4627, 256),
]
for v, kl, n, q, pk_b, sig_b, seg in variantes_dsa:
    print(f"{v:>15} {kl:>6} {n:>6} {q:>10} {pk_b:>12} {sig_b:>13} {seg:>15}")
```

---

## 5. SLH-DSA (SPHINCS+) — Firma basada en hash

```python
# seccion_05_sphincs.py
"""
SLH-DSA (FIPS 205, 2024) = SPHINCS+ estandarizado.
Firma digital basada en funciones hash (sin lattices).

Ventaja: seguridad basada solo en la resistencia de las funciones hash.
Desventaja: firmas más grandes que ML-DSA.

Estructura: árbol de Merkle jerárquico
  - FORS: Forward-Secure One-Time Signature
  - HT: Hypertree (árbol de árboles Winternitz)
  - WOTS+: Winternitz One-Time Signature

Implementación conceptual de WOTS+ (un nodo del árbol):
"""
import numpy as np
import hashlib
import os

print("=== SLH-DSA (SPHINCS+) ===\n")

class WOTS_Plus:
    """
    WOTS+ (Winternitz One-Time Signature).
    Firma un mensaje una sola vez usando encadenamiento de hash.
    """
    def __init__(self, n=32, w=16, seed=None):
        """
        n: longitud de salida hash (bytes)
        w: parámetro Winternitz (base de representación)
        """
        self.n = n
        self.w = w
        # l = ceil(8n/log2(w)) + ceil((log2(l1·(w-1))+1)/log2(w))
        self.l1 = int(np.ceil(8*n / np.log2(w)))
        self.l2 = int(np.ceil((np.log2(self.l1*(w-1)+1)+1) / np.log2(w)))
        self.l  = self.l1 + self.l2
        self.seed = seed or os.urandom(n)

    def _F(self, key, inp):
        """Función de hash con clave (PRF)."""
        return hashlib.sha256(key + inp).digest()[:self.n]

    def _chain(self, x, start, steps, pk_seed):
        """Aplica 'steps' iteraciones de F desde posición 'start'."""
        result = x
        for k in range(start, start + steps):
            addr = k.to_bytes(4, "big")
            result = self._F(pk_seed + addr, result)
        return result

    def keygen(self):
        """Genera (pk, sk) para WOTS+."""
        pk_seed = hashlib.sha256(self.seed + b"pk_seed").digest()[:self.n]
        sk_seed = hashlib.sha256(self.seed + b"sk_seed").digest()[:self.n]

        # sk: l elementos aleatorios de n bytes
        sk = [hashlib.sha256(sk_seed + i.to_bytes(2,"big")).digest()[:self.n]
              for i in range(self.l)]

        # pk: sk encadenado w-1 veces
        pk = [self._chain(s, 0, self.w - 1, pk_seed) for s in sk]

        return pk, sk, pk_seed

    def sign(self, sk, pk_seed, msg_hash):
        """Firma el hash del mensaje."""
        # Representación en base w del hash
        digits = []
        for byte in msg_hash[:self.l1]:
            for _ in range(2):
                digits.append(byte % self.w)
                byte //= self.w
        digits = digits[:self.l1]

        # Suma de verificación
        checksum = sum(self.w - 1 - d for d in digits)
        csum_digits = []
        for _ in range(self.l2):
            csum_digits.append(checksum % self.w)
            checksum //= self.w

        all_digits = digits + csum_digits[:self.l2]

        # Firma: s_i encadenado all_digits[i] veces
        sigma = [self._chain(s, 0, d, pk_seed)
                 for s, d in zip(sk[:self.l], all_digits)]
        return sigma, all_digits

    def verify(self, sigma, pk_seed, msg_hash, all_digits=None):
        """Verifica la firma (necesita all_digits calculado del hash)."""
        # Recalcular digits del hash
        digits = []
        for byte in msg_hash[:self.l1]:
            for _ in range(2):
                digits.append(byte % self.w)
                byte //= self.w
        digits = digits[:self.l1]
        checksum = sum(self.w - 1 - d for d in digits)
        csum_digits = []
        for _ in range(self.l2):
            csum_digits.append(checksum % self.w)
            checksum //= self.w
        all_digits_rec = digits + csum_digits[:self.l2]

        # Completar cadena: sigma_i encadenado (w-1-d_i) veces = pk_i
        pk_rec = [self._chain(s, d, self.w - 1 - d, pk_seed)
                  for s, d in zip(sigma, all_digits_rec)]
        return pk_rec

# Demostración
wots = WOTS_Plus(n=16, w=4)  # Parámetros reducidos para velocidad
pk, sk, pk_seed = wots.keygen()

mensaje = b"Mensaje de prueba"
msg_hash = hashlib.sha256(mensaje).digest()[:wots.l1]

sigma, digits = wots.sign(sk, pk_seed, msg_hash)
pk_verificado = wots.verify(sigma, pk_seed, msg_hash)

# Comprobar que pk_verificado == pk
valido = all(p1 == p2 for p1, p2 in zip(pk_verificado, pk))
print(f"WOTS+ KeyGen, Sign, Verify:")
print(f"  l (claves) = {wots.l}")
print(f"  Tamaño firma: {wots.l} × {wots.n} = {wots.l * wots.n} bytes")
print(f"  Verificación válida: {valido}")

# Mensaje alterado
msg_alt  = b"Mensaje alterado"
hash_alt = hashlib.sha256(msg_alt).digest()[:wots.l1]
pk_alt   = wots.verify(sigma, pk_seed, hash_alt)
inv = not all(p1 == p2 for p1, p2 in zip(pk_alt, pk))
print(f"  Firma rechaza mensaje alterado: {inv}")

# Parámetros oficiales SLH-DSA
print("\nParámetros oficiales SLH-DSA (FIPS 205):")
print(f"{'Variante':>20} {'n':>4} {'h':>4} {'d':>4} {'Pk (bytes)':>12} {'Sig (bytes)':>13} {'Seg. cuántica':>15}")
print("-" * 74)
variantes_slh = [
    ("SLH-DSA-SHA2-128s",  16, 63, 7,   32,  7856, 128),
    ("SLH-DSA-SHA2-128f",  16, 66, 22,  32, 17088, 128),
    ("SLH-DSA-SHA2-256s",  32, 64, 8,   64, 29792, 255),
    ("SLH-DSA-SHAKE-128s", 16, 63, 7,   32,  7856, 128),
]
for v, n, h, d, pk_b, sig_b, seg in variantes_slh:
    print(f"{v:>20} {n:>4} {h:>4} {d:>4} {pk_b:>12} {sig_b:>13} {seg:>15}")
```

---

## 6. Comparativa completa: clásico vs post-cuántico

```python
# seccion_06_comparativa.py
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

print("=== Comparativa: Criptografía Clásica vs Post-Cuántica ===\n")

# Tabla de comparación
algoritmos = {
    # Clásicos
    "RSA-2048":        {"tipo": "Clásico", "pk_kb": 0.29,  "ct_sig_kb": 0.29,  "seg_cl": 112, "seg_q": 0,   "op_us": 0.1},
    "ECDH-P256":       {"tipo": "Clásico", "pk_kb": 0.065, "ct_sig_kb": 0.066, "seg_cl": 128, "seg_q": 0,   "op_us": 0.05},
    "ECDSA-P256":      {"tipo": "Clásico", "pk_kb": 0.065, "ct_sig_kb": 0.072, "seg_cl": 128, "seg_q": 0,   "op_us": 0.1},
    # Post-cuánticos
    "ML-KEM-768":      {"tipo": "PQC",     "pk_kb": 1.16,  "ct_sig_kb": 1.06,  "seg_cl": 180, "seg_q": 164, "op_us": 0.06},
    "ML-KEM-1024":     {"tipo": "PQC",     "pk_kb": 1.53,  "ct_sig_kb": 1.53,  "seg_cl": 230, "seg_q": 200, "op_us": 0.08},
    "ML-DSA-65":       {"tipo": "PQC",     "pk_kb": 1.91,  "ct_sig_kb": 3.23,  "seg_cl": 230, "seg_q": 192, "op_us": 0.1},
    "SLH-DSA-SHA2-128s":{"tipo": "PQC",   "pk_kb": 0.031, "ct_sig_kb": 7.67,  "seg_cl": 160, "seg_q": 128, "op_us": 5.0},
}

print(f"{'Algoritmo':>22} {'Tipo':>8} {'PK (KB)':>9} {'CT/Sig (KB)':>12} {'Seg.Cl':>8} {'Seg.Q':>7}")
print("-" * 75)
for nombre, datos in algoritmos.items():
    print(f"{nombre:>22} {datos['tipo']:>8} {datos['pk_kb']:>9.3f} "
          f"{datos['ct_sig_kb']:>12.3f} {datos['seg_cl']:>8} {datos['seg_q']:>7}")

# Visualización comparativa
fig, axes = plt.subplots(2, 2, figsize=(14, 10))
fig.suptitle("Criptografía Clásica vs Post-Cuántica", fontsize=14)

nombres = list(algoritmos.keys())
tipos   = [v["tipo"] for v in algoritmos.values()]
colors  = ["coral" if t == "Clásico" else "steelblue" for t in tipos]

# Panel 1: Tamaño de clave pública
ax = axes[0, 0]
pk_kb = [v["pk_kb"] for v in algoritmos.values()]
bars = ax.bar(range(len(nombres)), pk_kb, color=colors, edgecolor="black")
ax.set_xticks(range(len(nombres)))
ax.set_xticklabels(nombres, rotation=45, ha="right", fontsize=7)
ax.set_ylabel("Tamaño (KB)")
ax.set_title("Tamaño Clave Pública")
ax.grid(axis="y", alpha=0.3)
from matplotlib.patches import Patch
ax.legend(handles=[Patch(color="coral",      label="Clásico"),
                   Patch(color="steelblue",  label="Post-Cuántico")],
          fontsize=8)

# Panel 2: Tamaño CT / Firma
ax = axes[0, 1]
ct_kb = [v["ct_sig_kb"] for v in algoritmos.values()]
ax.bar(range(len(nombres)), ct_kb, color=colors, edgecolor="black")
ax.set_xticks(range(len(nombres)))
ax.set_xticklabels(nombres, rotation=45, ha="right", fontsize=7)
ax.set_ylabel("Tamaño (KB)")
ax.set_title("Tamaño Ciphertext / Firma")
ax.grid(axis="y", alpha=0.3)

# Panel 3: Seguridad clásica vs cuántica
ax = axes[1, 0]
seg_cl = [v["seg_cl"] for v in algoritmos.values()]
seg_q  = [v["seg_q"]  for v in algoritmos.values()]
x = np.arange(len(nombres))
width = 0.35
ax.bar(x - width/2, seg_cl, width, label="Seguridad clásica", color="coral",     alpha=0.8)
ax.bar(x + width/2, seg_q,  width, label="Seguridad cuántica", color="steelblue", alpha=0.8)
ax.axhline(128, color="black", linestyle="--", alpha=0.7, label="Mínimo recomendado")
ax.set_xticks(x)
ax.set_xticklabels(nombres, rotation=45, ha="right", fontsize=7)
ax.set_ylabel("Bits de seguridad")
ax.set_title("Seguridad: Clásica vs Cuántica")
ax.legend(fontsize=8); ax.grid(axis="y", alpha=0.3)

# Panel 4: Velocidad de operación
ax = axes[1, 1]
op_us = [v["op_us"] for v in algoritmos.values()]
ax.bar(range(len(nombres)), op_us, color=colors, edgecolor="black")
ax.set_xticks(range(len(nombres)))
ax.set_xticklabels(nombres, rotation=45, ha="right", fontsize=7)
ax.set_ylabel("Tiempo operación (μs)")
ax.set_title("Velocidad (menor = mejor)")
ax.set_yscale("log")
ax.grid(axis="y", alpha=0.3)

plt.tight_layout()
plt.savefig("outputs/m33_pqc_comparativa.png", dpi=120, bbox_inches="tight")
print("\nFigura guardada: outputs/m33_pqc_comparativa.png")
plt.close()

print("\nRecomendaciones de migración NIST 2024:")
print("  KEM:   ML-KEM-768 (principal), ML-KEM-1024 (alta seguridad)")
print("  Firma: ML-DSA-65 (principal), SLH-DSA (backup sin lattices)")
print("  Híbrido: ECDH + ML-KEM durante transición (2025-2030)")
```

---

## Checkpoint

```python
# checkpoint_m33.py
"""Tests para Módulo 33 — Criptografía Post-Cuántica."""
import numpy as np
import hashlib

# T1: LWE — verificar que error es pequeño
def test_lwe_error_pequeno():
    n, q = 8, 97
    np.random.seed(0)
    s = np.array([3, 1, 4, 1, 5, 9, 2, 6])
    A = np.random.randint(0, q, (10, n))
    e = np.round(np.random.normal(0, 2, 10)).astype(int)
    b = (A @ s + e) % q

    # Error = b - As mod q, centrado
    diferencias = (b - A @ s) % q
    dif_cent = np.where(diferencias > q//2, diferencias - q, diferencias)
    assert np.max(np.abs(dif_cent)) <= 6, f"Error LWE muy grande: {np.max(np.abs(dif_cent))}"
    print(f"T1 PASS: error LWE máximo = {np.max(np.abs(dif_cent))}")

# T2: ML-KEM simplificado — Alice y Bob obtienen misma clave
def test_mlkem_shared_secret():
    from seccion_03_mlkem import MLKEMSimplificado
    kem = MLKEMSimplificado(n=16, q=97, k=2)
    pk, sk = kem.keygen()
    ct, ss_alice = kem.encaps(pk)
    ss_bob = kem.decaps(sk, ct)
    assert ss_alice == ss_bob, "Claves compartidas no coinciden"
    print(f"T2 PASS: ML-KEM shared secret correcto ({ss_alice.hex()[:16]}...)")

# T3: ML-DSA simplificado — verificación pasa con mensaje correcto
def test_mldsa_verificacion():
    from seccion_04_mldsa import MLDSASimplificado
    dsa = MLDSASimplificado()
    pk, sk = dsa.keygen()
    msg = b"Test message"
    sigma = dsa.sign(sk, msg)
    assert dsa.verify(pk, msg, sigma) == True
    assert dsa.verify(pk, b"Otro mensaje", sigma) == False
    print("T3 PASS: ML-DSA sign/verify correcto")

# T4: WOTS+ — verificación reconstruye la clave pública
def test_wots_keygen():
    from seccion_05_sphincs import WOTS_Plus
    wots = WOTS_Plus(n=16, w=4)
    pk, sk, pk_seed = wots.keygen()
    msg_hash = hashlib.sha256(b"test").digest()[:wots.l1]
    sigma, digits = wots.sign(sk, pk_seed, msg_hash)
    pk_rec = wots.verify(sigma, pk_seed, msg_hash)
    assert all(p1 == p2 for p1, p2 in zip(pk_rec, pk))
    print("T4 PASS: WOTS+ sign/verify correcto")

# T5: Tamaños de clave PQC son mayores que RSA para misma seguridad
def test_overhead_pqc():
    rsa_2048_pk_bytes = 256     # ~256 bytes = 2048 bits
    mlkem_768_pk_bytes = 1184   # ML-KEM-768
    mlkem_768_ct_bytes = 1088

    assert mlkem_768_pk_bytes > rsa_2048_pk_bytes
    assert mlkem_768_ct_bytes > rsa_2048_pk_bytes
    print(f"T5 PASS: overhead PQC confirmado ({mlkem_768_pk_bytes}B pk vs {rsa_2048_pk_bytes}B RSA)")

# T6: Entropía: LWE b parece uniforme sin conocer s
def test_lwe_parece_aleatorio():
    n, q = 4, 97
    np.random.seed(7)
    s = np.array([3, 1, 4, 1])
    A = np.random.randint(0, q, (500, n))
    e = np.round(np.random.normal(0, 1.5, 500)).astype(int)
    b_lwe = (A @ s + e) % q
    b_rand = np.random.randint(0, q, 500)

    # Test básico de uniformidad: desviación estándar similar
    std_lwe  = np.std(b_lwe)
    std_rand = np.std(b_rand)
    ratio = std_lwe / std_rand
    assert 0.7 < ratio < 1.3, f"LWE std={std_lwe:.1f} vs random std={std_rand:.1f}"
    print(f"T6 PASS: LWE se parece a aleatorio (std ratio={ratio:.3f} ≈ 1)")

if __name__ == "__main__":
    print("=== Checkpoint M33: Criptografía Post-Cuántica ===\n")
    test_lwe_error_pequeno()
    test_mlkem_shared_secret()
    test_mldsa_verificacion()
    test_wots_keygen()
    test_overhead_pqc()
    test_lwe_parece_aleatorio()
    print("\nTodos los tests pasaron.")
```

---

**Próximo módulo:** [M34 — Implementación PQC con liboqs](./modulo-34-liboqs.md)
