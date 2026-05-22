# 📖 docker-compose.yml — Explicação Completa

Este documento explica **cada configuração** do `docker-compose.yml` em detalhes, pensado para quem está aprendendo Docker e quer entender o que cada linha faz.

---

## Estrutura geral

Um arquivo `docker-compose.yml` define **serviços** (containers), **volumes** (armazenamento) e **redes** (comunicação entre containers).

```
services:      ← Lista de containers
  vllm:        ← Serviço 1: servidor LLM
  searxng:     ← Serviço 2: busca na web
  open-webui:  ← Serviço 3: interface de chat

volumes:       ← Armazenamento persistente
```

---

## Serviço 1: `vllm`

### `image: vllm/vllm-openai:v0.9.2`

Define qual imagem Docker usar. Formato: `repositório/nome:versão`

- `vllm/vllm-openai` → imagem oficial do vLLM com servidor OpenAI-compatible
- `:v0.9.2` → versão específica (fixada para reprodutibilidade)

> **Por que não usar `latest`?** Versões mais novas podem quebrar a compatibilidade. Sempre fixe versões em produção.

### `container_name: vllm-qwen`

Nome do container no Docker. Sem isso, Docker gera um nome aleatório como `ai-stack-vllm-1`. Com nome fixo, você pode fazer `docker logs vllm-qwen` diretamente.

### `restart: unless-stopped`

Política de reinicialização automática:

| Valor | Comportamento |
|---|---|
| `no` | Nunca reinicia |
| `always` | Sempre reinicia (até se você parar manualmente) |
| `unless-stopped` | Reinicia se cair, mas NÃO se você parar com `docker stop` |
| `on-failure` | Só reinicia se sair com código de erro |

`unless-stopped` é o padrão para serviços que devem ficar sempre ativos.

### `deploy.resources.reservations.devices`

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

Reserva acesso à GPU NVIDIA para o container:
- `driver: nvidia` → usa o NVIDIA Container Toolkit
- `count: 1` → aloca 1 GPU
- `capabilities: [gpu]` → habilita CUDA dentro do container

> Sem isso, o container roda normalmente mas **sem acesso à GPU** — o modelo rodaria só na CPU (muito lento).

### `environment`

Variáveis de ambiente passadas para dentro do container:

```yaml
- NVIDIA_VISIBLE_DEVICES=all
```
Torna todas as GPUs visíveis. Se você tiver múltiplas GPUs e quiser limitar, use `0`, `1`, ou `0,1`.

```yaml
- NVIDIA_DRIVER_CAPABILITIES=compute,utility
```
Define quais capacidades do driver NVIDIA estão disponíveis:
- `compute` → execução de código CUDA
- `utility` → ferramentas como `nvidia-smi`

```yaml
- CUDA_VISIBLE_DEVICES=0
```
Restringe o código Python/CUDA a usar apenas a GPU 0 (primeira). Útil em máquinas multi-GPU para evitar uso acidental de outra GPU.

```yaml
- HF_TOKEN=${HF_TOKEN}
- HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}
```
Token de autenticação do HuggingFace. O `${HF_TOKEN}` lê do arquivo `.env`. São duas variáveis porque bibliotecas diferentes checam nomes diferentes — passamos as duas por segurança.

### `ports: - "8000:8000"`

Formato: `"PORTA_HOST:PORTA_CONTAINER"`

- Porta `8000` do host → porta `8000` do container
- Acesse a API em `http://localhost:8000`
- Exemplo com porta diferente: `"9000:8000"` → acesse em `localhost:9000`

### `ipc: host`

IPC = Inter-Process Communication (comunicação entre processos).

`host` compartilha o namespace IPC do host com o container. Isso é necessário para o PyTorch usar **memória compartilhada** entre workers, especialmente com multi-GPU ou modelos grandes. Sem isso, pode aparecer erros de `SIGKILL` por falta de shared memory.

### `volumes: - huggingface:/root/.cache/huggingface`

Monta o volume `huggingface` no caminho `/root/.cache/huggingface` dentro do container.

**Por que isso importa?** Sem volume, cada vez que o container reinicia, os modelos baixados do HuggingFace (~2-15 GB) seriam perdidos e re-baixados. Com o volume, ficam persistentes.

### `command`

```yaml
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
  --port 8000
  --served-model-name qwen
  --quantization awq
  --dtype float16
  --gpu-memory-utilization 0.82
  --max-model-len 3072
  --max-num-seqs 16
  --max-num-batched-tokens 3072
  --enable-prefix-caching
  --tensor-parallel-size 1
  --enforce-eager
```

O `>` é sintaxe YAML para juntar múltiplas linhas em uma só string. Esses são os argumentos passados diretamente para o `api_server.py` do vLLM:

| Argumento | Valor | Explicação |
|---|---|---|
| `--model` | `Qwen/Qwen2.5-3B-Instruct-AWQ` | ID do modelo no HuggingFace |
| `--host` | `0.0.0.0` | Escuta em todas as interfaces de rede do container |
| `--port` | `8000` | Porta onde a API responde |
| `--served-model-name` | `qwen` | Nome do modelo na API (usado pelo OpenWebUI) |
| `--quantization` | `awq` | Tipo de quantização (AWQ = Activation-aware Weight Quantization) |
| `--dtype` | `float16` | Precisão dos cálculos (float16 = metade da precisão, metade da VRAM) |
| `--gpu-memory-utilization` | `0.82` | Usa 82% da VRAM disponível (deixa 18% de margem) |
| `--max-model-len` | `3072` | Tamanho máximo do contexto em tokens |
| `--max-num-seqs` | `16` | Máximo de requisições processadas simultaneamente |
| `--max-num-batched-tokens` | `3072` | Total de tokens em batch por step |
| `--enable-prefix-caching` | — | Cacheia prefixos comuns (ex: system prompt) → mais rápido |
| `--tensor-parallel-size` | `1` | Número de GPUs para tensor parallelism (1 = sem paralelismo) |
| `--enforce-eager` | — | Desabilita CUDA graphs (necessário para GPUs Turing/Volta) |

> **Sobre `--enforce-eager`:** GPUs com Compute Capability < 8.0 (como RTX 2060) não suportam CUDA graphs do vLLM V1. Esse flag força o modo eager (sem graphs), compatível com todas as GPUs CUDA.

### `healthcheck`

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/v1/models"]
  interval: 30s
  timeout: 10s
  retries: 10
```

Define como o Docker verifica se o container está saudável:
- `test` → comando executado dentro do container. `-f` faz curl retornar erro se HTTP ≠ 200
- `interval: 30s` → verifica a cada 30 segundos
- `timeout: 10s` → se não responder em 10s, conta como falha
- `retries: 10` → após 10 falhas, marca como `unhealthy`

O `open-webui` só sobe após o vLLM estar `healthy` (via `depends_on`).

---

## Serviço 2: `searxng`

### `image: searxng/searxng:latest`

Imagem oficial do SearXNG. Usamos `latest` aqui pois o SearXNG é estável e tem breaking changes raros.

### `volumes: - ./searxng:/etc/searxng`

`./searxng` = pasta `searxng/` relativa ao `docker-compose.yml` no host.
`/etc/searxng` = onde o SearXNG lê suas configurações dentro do container.

Isso permite editar o `settings.yml` do SearXNG sem entrar no container.

---

## Serviço 3: `open-webui`

### `depends_on`

```yaml
depends_on:
  vllm:
    condition: service_healthy
```

O OpenWebUI só inicia **depois que o vLLM estiver healthy**. Sem isso, o OpenWebUI tentaria conectar na API do vLLM antes dela estar pronta, causando erros.

### `environment — conexão com vLLM`

```yaml
- OPENAI_API_BASE_URL=http://vllm:8000/v1
```
`vllm` aqui é o **nome do serviço** no Docker Compose. O Docker cria automaticamente uma rede interna onde os containers se comunicam pelo nome do serviço — não precisa de IP.

```yaml
- OPENAI_API_KEY=EMPTY
```
O vLLM não exige autenticação por padrão. Mas o OpenWebUI exige que esse campo seja preenchido com algum valor (qualquer valor funciona).

### `extra_hosts`

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Permite que o container acesse serviços rodando no **host** (sua máquina). Por exemplo, se você tiver uma API Python rodando em `localhost:8080`, dentro do container você acessa via `host.docker.internal:8080`.

---

## Volumes

```yaml
volumes:
  huggingface:   # Cache dos modelos (~2-15 GB)
  open-webui:    # Banco SQLite do OpenWebUI
```

Volumes Docker são armazenamentos persistentes gerenciados pelo Docker. Diferente de bind mounts (`./pasta:/caminho`), volumes ficam em `/var/lib/docker/volumes/` no host.

**Comandos úteis:**
```bash
# Listar volumes
docker volume ls

# Inspecionar onde está no host
docker volume inspect ai-stack_huggingface

# Apagar um volume (APAGA O MODELO BAIXADO!)
docker volume rm ai-stack_huggingface
```
