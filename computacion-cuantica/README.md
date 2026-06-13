# Tutorial 2 — Computación Cuántica y Quantum ML

> Desde qubits y puertas cuánticas hasta Quantum Machine Learning
> y criptografía post-cuántica. Triple stack: Qiskit + PennyLane + Cirq.

---

## Setup del entorno

```bash
conda create -n quantum python=3.11 -y
conda activate quantum

# Frameworks cuánticos
pip install qiskit qiskit-aer qiskit-ibm-runtime
pip install pennylane pennylane-qiskit
pip install cirq cirq-google

# ML y visualización
pip install torch numpy matplotlib jupyter ipykernel
pip install qiskit-machine-learning

# Criptografía post-cuántica (Módulos 32-34)
pip install liboqs-python

python -m ipykernel install --user --name quantum --display-name "Quantum"
```

### Cuenta IBM Quantum (gratuita)
1. Registrarse en https://quantum.ibm.com
2. Copiar el API token desde el dashboard
3. Configurar en Python:
```python
from qiskit_ibm_runtime import QiskitRuntimeService
QiskitRuntimeService.save_account(channel="ibm_quantum", token="TU_TOKEN")
```

---

## Módulos

### Parte 1 — Fundamentos
| Módulo | Tema | Archivo |
|---|---|---|
| M01 | Mecánica Cuántica para Programadores | [modulo-01-mecanica-cuantica.md](./modulo-01-mecanica-cuantica.md) |
| M02 | Qubits, Superposición y Entrelazamiento | [modulo-02-qubits.md](./modulo-02-qubits.md) |
| M03 | Puertas Cuánticas y Lectura de Circuitos | [modulo-03-puertas-circuitos.md](./modulo-03-puertas-circuitos.md) |

### Parte 2 — Frameworks
| Módulo | Tema | Archivo |
|---|---|---|
| M04 | Setup Triple Stack — Qiskit, PennyLane, Cirq | [modulo-04-setup-triple-stack.md](./modulo-04-setup-triple-stack.md) |
| M05 | Qiskit en Profundidad — IBM Quantum | [modulo-05-qiskit.md](./modulo-05-qiskit.md) |
| M06 | PennyLane — Diferenciación Automática Cuántica | [modulo-06-pennylane.md](./modulo-06-pennylane.md) |
| M07 | Cirq — Google Quantum AI | [modulo-07-cirq.md](./modulo-07-cirq.md) |

### Parte 3 — Algoritmos Cuánticos
| Módulo | Tema | Archivo |
|---|---|---|
| M08 | Deutsch-Jozsa y Bernstein-Vazirani | [modulo-08-deutsch-bernstein.md](./modulo-08-deutsch-bernstein.md) |
| M09 | Algoritmo de Grover — Búsqueda Cuadrática | [modulo-09-grover.md](./modulo-09-grover.md) |
| M10 | Transformada de Fourier Cuántica (QFT) | [modulo-10-qft.md](./modulo-10-qft.md) |
| M11 | Algoritmo de Shor — Factorización | [modulo-11-shor.md](./modulo-11-shor.md) |
| M12 | Estimación de Fase Cuántica (QPE) | [modulo-12-qpe.md](./modulo-12-qpe.md) |

### Parte 4 — Hardware y Ruido
| Módulo | Tema | Archivo |
|---|---|---|
| M13 | Tipos de Hardware Cuántico | [modulo-13-hardware.md](./modulo-13-hardware.md) |
| M14 | Ruido, Decoherencia y Error Mitigation | [modulo-14-ruido-error.md](./modulo-14-ruido-error.md) |
| M15 | Corrección de Errores Cuánticos (QEC) | [modulo-15-qec.md](./modulo-15-qec.md) |
| M16 | Ejecutar en Hardware Real — IBM y Google | [modulo-16-hardware-real.md](./modulo-16-hardware-real.md) |

### Parte 5 — Computación Variacional
| Módulo | Tema | Archivo |
|---|---|---|
| M17 | Circuitos Variacionales Parametrizados (VQC) | [modulo-17-vqc.md](./modulo-17-vqc.md) |
| M18 | VQE — Variational Quantum Eigensolver | [modulo-18-vqe.md](./modulo-18-vqe.md) |
| M19 | QAOA — Quantum Approximate Optimization | [modulo-19-qaoa.md](./modulo-19-qaoa.md) |
| M20 | Gradientes Cuánticos — Parameter Shift Rule | [modulo-20-gradientes.md](./modulo-20-gradientes.md) |

### Parte 6 — Quantum Machine Learning
| Módulo | Tema | Archivo |
|---|---|---|
| M21 | Codificación de Datos Clásicos en Qubits | [modulo-21-encoding.md](./modulo-21-encoding.md) |
| M22 | Quantum SVM y Kernels Cuánticos | [modulo-22-qsvm.md](./modulo-22-qsvm.md) |
| M23 | Quantum Neural Networks (QNN) | [modulo-23-qnn.md](./modulo-23-qnn.md) |
| M24 | Modelos Híbridos Clásico-Cuántico con PyTorch | [modulo-24-hibrido-pytorch.md](./modulo-24-hibrido-pytorch.md) |
| M25 | Quantum Clustering y Quantum PCA | [modulo-25-qclustering-qpca.md](./modulo-25-qclustering-qpca.md) |
| M26 | QGAN — Quantum Generative Adversarial Networks | [modulo-26-qgan.md](./modulo-26-qgan.md) |
| M27 | QCNN — Quantum Convolutional Neural Networks | [modulo-27-qcnn.md](./modulo-27-qcnn.md) |
| M28 | Quantum Reinforcement Learning | [modulo-28-qrl.md](./modulo-28-qrl.md) |

### Parte 7 — Aplicaciones
| Módulo | Tema | Archivo |
|---|---|---|
| M29 | Química Cuántica y Farmacología | [modulo-29-quimica.md](./modulo-29-quimica.md) |
| M30 | Optimización Cuántica — Finanzas y Logística | [modulo-30-optimizacion.md](./modulo-30-optimizacion.md) |
| M31 | Quantum NLP | [modulo-31-qnlp.md](./modulo-31-qnlp.md) |

### Parte 8 — Criptografía
| Módulo | Tema | Archivo |
|---|---|---|
| M32 | Criptografía Cuántica — QKD y Protocolo BB84 | [modulo-32-qkd.md](./modulo-32-qkd.md) |
| M33 | Criptografía Post-Cuántica — Estándares NIST 2024 | [modulo-33-post-cuantica.md](./modulo-33-post-cuantica.md) |
| M34 | Implementación PQC con liboqs | [modulo-34-liboqs.md](./modulo-34-liboqs.md) |

---

## Checklist de progreso

- [x] M01 — Mecánica Cuántica *(migrado desde tutorial-quantum)*
- [x] M02 — Qubits y Entrelazamiento
- [x] M03 — Puertas y Circuitos
- [x] M04 — Setup Triple Stack
- [x] M05 — Qiskit
- [x] M06 — PennyLane
- [x] M07 — Cirq
- [x] M08 — Deutsch-Bernstein
- [x] M09 — Grover
- [x] M10 — QFT
- [x] M11 — Shor
- [x] M12 — QPE
- [x] M13 — Hardware
- [x] M14 — Ruido y Error Mitigation
- [x] M15 — QEC
- [x] M16 — Hardware Real
- [x] M17 — VQC
- [x] M18 — VQE
- [x] M19 — QAOA
- [x] M20 — Gradientes Cuánticos
- [x] M21 — Encoding
- [x] M22 — QSVM
- [x] M23 — QNN
- [x] M24 — Híbrido PyTorch
- [x] M25 — QClustering + QPCA
- [x] M26 — QGAN
- [x] M27 — QCNN
- [x] M28 — QRL
- [x] M29 — Química Cuántica
- [x] M30 — Optimización
- [x] M31 — QNLP
- [x] M32 — QKD
- [x] M33 — Post-Cuántica
- [x] M34 — liboqs
