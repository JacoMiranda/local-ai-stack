# 🤖 Local AI Stack

> Stack completa de IA local com inferência via GPU, interface de chat e busca na web — tudo self-hosted, sem nuvem, sem custo por token.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![vLLM](https://img.shields.io/badge/vLLM-v0.9.2-orange)
![Open WebUI](https://img.shields.io/badge/Open_WebUI-latest-blue)
![NVIDIA](https://img.shields.io/badge/NVIDIA-GPU-76B900?logo=nvidia)

## 🧩 O que é esse projeto?

Esse repositório contém uma stack Docker pronta para rodar um **assistente de IA local** com:

| Componente | Função | Porta |
|---|---|---|
| **vLLM** | Servidor de inferência LLM com GPU | `8000` |
| **Open WebUI** | Interface de chat (estilo ChatGPT) | `3000` |
| **SearXNG** | Busca na web privada (RAG) | `8080` |

### Por que usar isso?

- 🔒 **Privacidade total** — seus dados nunca saem da sua máquina
- 💸 **Sem custo por token** — sem API da OpenAI/Anthropic
- ⚡ **Resposta rápida** — inferência local com sua GPU
- 🌐 **Busca na web** — o modelo pode pesquisar na internet via SearXNG
- 🔁 **Reproduzível** — sobe com um comando

---

## 🖥️ Ambiente utilizado (hardware de referência)

> Esta configuração foi desenvolvida e testada neste hardware:

| Componente | Especificação |
|---|---|
| **GPU** | NVIDIA GeForce RTX 2060 (12 GB VRAM) |
| **Arquitetura GPU** | Turing (Compute Capability 7.5) |
| **CPU** | (compatível com qualquer x86-64) |
| **RAM** | (mínimo recomendado: 16 GB) |
| **OS** | Windows 11 com WSL2 |
| **Docker** | Docker Desktop + NVIDIA Container Toolkit |
| **CUDA** | 12.x |
| **vLLM** | v0.9.2 |
| **Modelo** | Qwen/Qwen2.5-3B-Instruct-AWQ |

> ⚠️ O RTX 2060 tem Compute Capability 7.5 (Turing). O vLLM usa automaticamente o **backend XFormers** (em vez de FlashAttention-2, que requer CC 8.0+). A performance é boa, mas ligeiramente menor que em GPUs Ampere/Ada.

---

## ⚡ Quick Start (início rápido)

```bash
# 1. Clone o repositório
git clone https://github.com/SEU_USUARIO/local-ai-stack.git
cd local-ai-stack

# 2. Configure as variáveis de ambiente
cp .env.example .env
# Edite o .env e preencha seu HF_TOKEN

# 3. Suba a stack
docker compose up -d

# 4. Acesse o chat
# http://localhost:3000
```

> O modelo (~2GB) é baixado automaticamente do HuggingFace na primeira vez. Aguarde 3-5 minutos.

---

## 📋 Pré-requisitos

Veja o guia completo em **[docs/01-installation.md](docs/01-installation.md)**.

Resumo:
- [ ] Docker Desktop instalado
- [ ] NVIDIA Container Toolkit instalado
- [ ] Conta no HuggingFace (gratuita) + token de acesso
- [ ] Aceitar a licença do modelo no HuggingFace

---

## 🔧 Configuração do modelo (`.env`)

Edite o arquivo `.env` para trocar o modelo conforme sua GPU:

```env
HF_TOKEN=hf_SeuTokenAqui
VLLM_MODEL=Qwen/Qwen2.5-3B-Instruct-AWQ
SERVED_MODEL_NAME=qwen
```

Veja alternativas por GPU em **[docs/03-hardware.md](docs/03-hardware.md)**.

---

## 📚 Documentação

| Arquivo | Conteúdo |
|---|---|
| [DEVLOG.md](DEVLOG.md) | 📓 Diário de bordo — todos os erros e correções do setup real |
| [docs/01-installation.md](docs/01-installation.md) | Instalação completa do zero |
| [docs/02-docker-compose-explained.md](docs/02-docker-compose-explained.md) | Explicação de cada linha do docker-compose |
| [docs/03-hardware.md](docs/03-hardware.md) | Configurações por tipo de GPU |
| [docs/04-troubleshooting.md](docs/04-troubleshooting.md) | Erros comuns e soluções |

---

## 🌐 URLs dos serviços

Após subir a stack:

| Serviço | URL | Descrição |
|---|---|---|
| Open WebUI | http://localhost:3000 | Interface de chat |
| vLLM API | http://localhost:8000/v1 | API OpenAI-compatible |
| vLLM Docs | http://localhost:8000/docs | Swagger UI da API |
| SearXNG | http://localhost:8080 | Motor de busca |

---

## 🛑 Comandos úteis

```bash
# Subir tudo
docker compose up -d

# Ver logs em tempo real
docker compose logs -f

# Ver logs só do vLLM
docker compose logs -f vllm

# Parar tudo (sem apagar dados)
docker compose down

# Parar e apagar todos os dados (CUIDADO!)
docker compose down -v

# Ver status dos containers
docker compose ps

# Reiniciar só o vLLM
docker compose restart vllm
```

---

## 📊 Consumo de recursos (RTX 2060 12GB)

Com o modelo **Qwen2.5-3B-Instruct-AWQ**:

| Recurso | Uso |
|---|---|
| VRAM (model weights) | ~1.95 GB |
| VRAM (KV Cache) | ~7.56 GB |
| VRAM total utilizada | ~9.8 GB (de 12 GB) |
| Max concorrência | 71 requests simultâneos |
| Context window | 3.072 tokens |

---

## 📄 Licença

MIT — use, modifique e distribua livremente.
