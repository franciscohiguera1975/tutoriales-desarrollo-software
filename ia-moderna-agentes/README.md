# Tutorial 3 — IA Moderna y Agentes

> Del Transformer al agente autónomo en producción.
> Cubre LLMs, RAG, agentes, MCP, sistemas multi-agente, LLMOps
> y tecnologías emergentes de IA.

---

## Setup del entorno

```bash
conda create -n ia-moderna python=3.11 -y
conda activate ia-moderna

# Deep Learning
pip install torch torchvision torchaudio
pip install transformers datasets tokenizers accelerate
pip install diffusers

# LLMs y fine-tuning
pip install peft trl bitsandbytes
pip install openai anthropic

# RAG y bases vectoriales
pip install langchain langchain-community langchain-openai
pip install llama-index llama-index-core
pip install chromadb pinecone-client weaviate-client
pip install sentence-transformers faiss-cpu

# Agentes
pip install crewai autogen-agentchat

# LLMOps y deployment
pip install fastapi uvicorn
pip install langsmith langfuse
pip install vllm  # requiere GPU

# Utilidades
pip install numpy pandas matplotlib jupyter ipykernel
python -m ipykernel install --user --name ia-moderna --display-name "IA Moderna"
```

---

## Módulos

### Parte 1 — Deep Learning Avanzado
| Módulo | Tema | Archivo |
|---|---|---|
| M01 | RNN, LSTM y GRU — Redes Recurrentes | [modulo-01-rnn-lstm-gru.md](./modulo-01-rnn-lstm-gru.md) |
| M02 | Mecanismo de Atención — de seq2seq a Transformer | [modulo-02-atencion-seq2seq.md](./modulo-02-atencion-seq2seq.md) |
| M03 | Arquitectura Transformer en Detalle | [modulo-03-transformer.md](./modulo-03-transformer.md) |
| M04 | Vision Transformers (ViT) y Modelos Multimodales | [modulo-04-vit-multimodal.md](./modulo-04-vit-multimodal.md) |
| M05 | Modelos de Difusión — Stable Diffusion, DALL-E | [modulo-05-diffusion.md](./modulo-05-diffusion.md) |

### Parte 2 — Large Language Models
| Módulo | Tema | Archivo |
|---|---|---|
| M06 | Arquitectura Interna de LLMs — GPT, BERT, LLaMA | [modulo-06-arquitectura-llm.md](./modulo-06-arquitectura-llm.md) |
| M07 | Pre-entrenamiento y Tokenización | [modulo-07-preentrenamiento.md](./modulo-07-preentrenamiento.md) |
| M08 | Prompt Engineering — Zero-shot, Few-shot, CoT, ToT | [modulo-08-prompt-engineering.md](./modulo-08-prompt-engineering.md) |
| M09 | Fine-tuning Completo — SFT y RLHF | [modulo-09-finetuning-sft-rlhf.md](./modulo-09-finetuning-sft-rlhf.md) |
| M10 | Fine-tuning Eficiente — LoRA, QLoRA, PEFT | [modulo-10-lora-qlora-peft.md](./modulo-10-lora-qlora-peft.md) |

### Parte 3 — RAG y Bases de Conocimiento
| Módulo | Tema | Archivo |
|---|---|---|
| M11 | Embeddings y Representaciones Semánticas | [modulo-11-embeddings.md](./modulo-11-embeddings.md) |
| M12 | Bases de Datos Vectoriales — Chroma, Pinecone, pgvector | [modulo-12-vectordb.md](./modulo-12-vectordb.md) |
| M13 | Pipeline RAG Básico | [modulo-13-rag-basico.md](./modulo-13-rag-basico.md) |
| M14 | RAG Avanzado — HyDE, RAG-Fusion, Contextual Retrieval | [modulo-14-rag-avanzado.md](./modulo-14-rag-avanzado.md) |
| M15 | Graph RAG — Conocimiento Estructurado | [modulo-15-graph-rag.md](./modulo-15-graph-rag.md) |

### Parte 4 — Agentes Inteligentes
| Módulo | Tema | Archivo |
|---|---|---|
| M16 | Fundamentos de Agentes — ReAct y Tool Use | [modulo-16-agentes-react.md](./modulo-16-agentes-react.md) |
| M17 | LangChain — Cadenas y Agentes | [modulo-17-langchain.md](./modulo-17-langchain.md) |
| M18 | LlamaIndex — Indexación y Consulta | [modulo-18-llamaindex.md](./modulo-18-llamaindex.md) |
| M19 | CrewAI — Equipos de Agentes | [modulo-19-crewai.md](./modulo-19-crewai.md) |
| M20 | AutoGen — Conversaciones Multi-Agente | [modulo-20-autogen.md](./modulo-20-autogen.md) |

### Parte 5 — MCP y Skills
| Módulo | Tema | Archivo |
|---|---|---|
| M21 | Model Context Protocol — Arquitectura y Protocolo | [modulo-21-mcp-arquitectura.md](./modulo-21-mcp-arquitectura.md) |
| M22 | Crear Servidores MCP Personalizados | [modulo-22-mcp-servidores.md](./modulo-22-mcp-servidores.md) |
| M23 | Skills y Herramientas en Claude Code | [modulo-23-skills-claude.md](./modulo-23-skills-claude.md) |
| M24 | Integración MCP con Bases de Datos y APIs Externas | [modulo-24-mcp-integraciones.md](./modulo-24-mcp-integraciones.md) |

### Parte 6 — Sistemas Multi-Agente
| Módulo | Tema | Archivo |
|---|---|---|
| M25 | Orquestación de Agentes | [modulo-25-orquestacion.md](./modulo-25-orquestacion.md) |
| M26 | Memoria a Largo Plazo en Agentes | [modulo-26-memoria-agentes.md](./modulo-26-memoria-agentes.md) |
| M27 | Agentes con Acceso a Código — Code Interpreter | [modulo-27-code-interpreter.md](./modulo-27-code-interpreter.md) |
| M28 | Sistemas Autónomos y AI Safety | [modulo-28-autonomos-safety.md](./modulo-28-autonomos-safety.md) |

### Parte 7 — LLMOps y Producción
| Módulo | Tema | Archivo |
|---|---|---|
| M29 | Evaluación de LLMs — Benchmarks, RAGAS, LLM-as-Judge | [modulo-29-evaluacion-llm.md](./modulo-29-evaluacion-llm.md) |
| M30 | Observabilidad — LangSmith, Langfuse, Trazas | [modulo-30-observabilidad.md](./modulo-30-observabilidad.md) |
| M31 | Deployment — FastAPI, vLLM, llama.cpp, Ollama | [modulo-31-deployment.md](./modulo-31-deployment.md) |
| M32 | Optimización — Quantización, Pruning, Destilación | [modulo-32-optimizacion.md](./modulo-32-optimizacion.md) |

### Parte 8 — Tecnologías Emergentes
| Módulo | Tema | Archivo |
|---|---|---|
| M33 | Modelos de Razonamiento — o1, DeepSeek-R1, Gemini | [modulo-33-razonamiento.md](./modulo-33-razonamiento.md) |
| M34 | AI Agents en Producción — Casos Reales | [modulo-34-agentes-produccion.md](./modulo-34-agentes-produccion.md) |
| M35 | Convergencia IA + Computación Cuántica | [modulo-35-quantum-ia.md](./modulo-35-quantum-ia.md) |
| M36 | Roadmap y Tendencias 2025-2030 | [modulo-36-roadmap.md](./modulo-36-roadmap.md) |

---

## Checklist de progreso

- [x] M01 — RNN, LSTM, GRU
- [x] M02 — Atención y seq2seq
- [x] M03 — Transformer
- [x] M04 — ViT y Multimodal
- [x] M05 — Diffusion Models
- [x] M06 — Arquitectura LLMs
- [x] M07 — Pre-entrenamiento
- [x] M08 — Prompt Engineering
- [x] M09 — Fine-tuning SFT/RLHF
- [x] M10 — LoRA/QLoRA/PEFT
- [x] M11 — Embeddings
- [x] M12 — Vector Databases
- [x] M13 — RAG Básico
- [x] M14 — RAG Avanzado
- [x] M15 — Graph RAG
- [x] M16 — Agentes ReAct
- [x] M17 — LangChain
- [x] M18 — LlamaIndex
- [x] M19 — CrewAI
- [x] M20 — AutoGen
- [x] M21 — MCP Arquitectura
- [x] M22 — MCP Servidores
- [x] M23 — Skills Claude Code
- [x] M24 — MCP Integraciones
- [x] M25 — Orquestación
- [x] M26 — Memoria Agentes
- [x] M27 — Code Interpreter
- [x] M28 — Safety
- [x] M29 — Evaluación LLMs
- [x] M30 — Observabilidad
- [x] M31 — Deployment
- [x] M32 — Optimización
- [x] M33 — Razonamiento
- [x] M34 — Agentes Producción
- [x] M35 — Quantum + IA
- [x] M36 — Roadmap
