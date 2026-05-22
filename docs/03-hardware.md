# 🖥️ Configurações de Hardware

Este guia mostra como adaptar o `docker-compose.yml` para diferentes tipos de GPU e máquinas.

---

## Hardware de referência (usado neste projeto)

| Componente | Especificação |
|---|---|
| GPU | NVIDIA GeForce RTX 2060 12 GB |
| Arquitetura | Turing (Compute Capability 7.5) |
| VRAM | 12 GB GDDR6 |
| OS | Windows 11 + WSL2 |
| Backend vLLM | XFormers (FlashAttention-2 não suportado em CC < 8.0) |
| Engine vLLM | V0 (V1 requer CC ≥ 8.0) |

---

## Tabela de compatibilidade por GPU

| GPU | VRAM | CC | Modelo recomendado | `--max-model-len` | `--gpu-memory-utilization` |
|---|---|---|---|---|---|
| RTX 4090 | 24 GB | 8.9 | Qwen2.5-14B-AWQ | 8192 | 0.90 |
| RTX 3090 / 4080 | 24 GB | 8.6 | Qwen2.5-14B-AWQ | 8192 | 0.88 |
| RTX 3080 Ti / 4070 Ti | 12-16 GB | 8.6 | Qwen2.5-7B-AWQ | 6144 | 0.85 |
| RTX 3070 / 4060 Ti | 8 GB | 8.6 | Qwen2.5-3B-AWQ | 4096 | 0.85 |
| **RTX 2060 12GB** ← *este projeto* | **12 GB** | **7.5** | **Qwen2.5-3B-AWQ** | **3072** | **0.82** |
| RTX 2060 / 2070 | 6-8 GB | 7.5 | Qwen2.5-1.5B-AWQ | 2048 | 0.80 |
| GTX 1080 Ti | 11 GB | 6.1 | Qwen2.5-3B-AWQ | 2048 | 0.78 |
| Sem GPU | — | — | Ver seção CPU-only | — | — |

> **CC = Compute Capability.** Veja a CC da sua GPU em: https://developer.nvidia.com/cuda-gpus

---

## Configurações por tipo de GPU

### 🟢 GPUs Ampere/Ada (RTX 30xx / 40xx — CC 8.0+)

```yaml
command: >
  --model Qwen/Qwen2.5-7B-Instruct-AWQ
  --host 0.0.0.0
  --port 8000
  --served-model-name qwen
  --quantization awq
  --dtype bfloat16         # bfloat16 é preferível em Ampere+
  --gpu-memory-utilization 0.88
  --max-model-len 8192
  --max-num-seqs 32
  --max-num-batched-tokens 8192
  --enable-prefix-caching
  --tensor-parallel-size 1
  # SEM --enforce-eager (FlashAttention-2 funciona aqui)
```

Diferenças em relação ao RTX 2060:
- Remove `--enforce-eager` → habilita CUDA graphs e FlashAttention-2
- Usa `bfloat16` em vez de `float16` (mais estável numericamente)
- Aumenta `--max-model-len` para aproveitar mais contexto
- Pode usar modelo maior (7B, 14B)

---

### 🟡 GPUs Turing/Volta (RTX 20xx, GTX 16xx — CC 7.0-7.5)

> *Esta é a configuração deste projeto (RTX 2060 12GB)*

```yaml
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
  --port 8000
  --served-model-name qwen
  --quantization awq
  --dtype float16           # float16 (bfloat16 não suportado em Turing)
  --gpu-memory-utilization 0.82
  --max-model-len 3072      # Ajustar conforme VRAM
  --max-num-seqs 16
  --max-num-batched-tokens 3072
  --enable-prefix-caching
  --tensor-parallel-size 1
  --enforce-eager           # OBRIGATÓRIO para CC < 8.0
```

**Notas importantes para Turing:**
- `--enforce-eager` é **obrigatório** — vLLM V1 não suporta CC < 8.0
- Use `float16` (bfloat16 não tem suporte nativo em Turing)
- vLLM cai automaticamente para o engine V0 e backend XFormers
- Performance é boa, mas ~15-20% menor que em Ampere com FlashAttention-2

---

### 🔴 GPUs com pouca VRAM (6-8 GB)

```yaml
command: >
  --model Qwen/Qwen2.5-1.5B-Instruct-AWQ
  --host 0.0.0.0
  --port 8000
  --served-model-name qwen
  --quantization awq
  --dtype float16
  --gpu-memory-utilization 0.80
  --max-model-len 2048
  --max-num-seqs 8
  --max-num-batched-tokens 2048
  --enable-prefix-caching
  --tensor-parallel-size 1
  --enforce-eager
```

Modelos para 6-8 GB VRAM:
- `Qwen/Qwen2.5-1.5B-Instruct-AWQ` (~1 GB VRAM)
- `microsoft/phi-2` (~3.5 GB VRAM, sem token necessário)
- `TinyLlama/TinyLlama-1.1B-Chat-v1.0` (~1 GB VRAM)

---

### ⚪ Sem GPU (CPU-only)

> ⚠️ Muito lento para uso em produção. Apenas para testes.

Troque o serviço `vllm` por **Ollama** (melhor suporte CPU):

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama:/root/.ollama

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open-webui:/app/backend/data

volumes:
  ollama:
  open-webui:
```

Após subir:
```bash
# Baixar um modelo pelo Ollama
docker exec -it ollama ollama pull qwen2.5:3b
```

---

## Multi-GPU (2+ GPUs iguais)

Para usar 2 GPUs em tensor parallelism:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all        # Usa todas as GPUs
          capabilities: [gpu]

command: >
  --model Qwen/Qwen2.5-14B-Instruct-AWQ
  --tensor-parallel-size 2  # Divide o modelo entre 2 GPUs
  ...
```

> As GPUs precisam ser do mesmo modelo e ter NVLINK ou comunicação PCIe.

---

## Modelos alternativos recomendados

| Modelo | Parâmetros | VRAM (AWQ) | Token HF? | Qualidade |
|---|---|---|---|---|
| Qwen/Qwen2.5-3B-Instruct-AWQ | 3B | ~2 GB | Sim | ⭐⭐⭐ |
| Qwen/Qwen2.5-7B-Instruct-AWQ | 7B | ~5 GB | Sim | ⭐⭐⭐⭐ |
| Qwen/Qwen2.5-14B-Instruct-AWQ | 14B | ~10 GB | Sim | ⭐⭐⭐⭐⭐ |
| microsoft/phi-2 | 2.7B | ~3.5 GB | Não | ⭐⭐⭐ |
| mistralai/Mistral-7B-Instruct-v0.3 | 7B | ~5 GB | Sim | ⭐⭐⭐⭐ |
| google/gemma-2-2b-it | 2B | ~2.5 GB | Sim | ⭐⭐⭐ |

Para trocar o modelo, edite apenas o `.env`:
```env
VLLM_MODEL=Qwen/Qwen2.5-7B-Instruct-AWQ
SERVED_MODEL_NAME=qwen-7b
```

Depois reinicie:
```bash
docker compose restart vllm
```
