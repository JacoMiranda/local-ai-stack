# 🛠️ Troubleshooting — Erros Comuns e Soluções

Este guia cobre os erros mais comuns encontrados ao rodar a stack, incluindo os que foram resolvidos durante o desenvolvimento deste projeto.

---

## Índice

1. [api_server.py: error: unrecognized arguments: vllm serve](#erro-1-api_serverpy-error-unrecognized-arguments-vllm-serve)
2. [api_server.py: error: unrecognized arguments: Qwen/...](#erro-2-api_serverpy-error-unrecognized-arguments-qwen)
3. [VLLM_USE_V1=1 is not supported with Compute Capability < 8.0](#erro-3-vllm_use_v11-is-not-supported-with-compute-capability--80)
4. [GPU não detectada no Docker](#erro-4-gpu-não-detectada-no-docker)
5. [Container fica em "health: starting" para sempre](#erro-5-container-fica-em-health-starting-para-sempre)
6. [Erro 403 ao baixar modelo do HuggingFace](#erro-6-erro-403-ao-baixar-modelo-do-huggingface)
7. [CUDA out of memory](#erro-7-cuda-out-of-memory)
8. [OpenWebUI não conecta no vLLM](#erro-8-openwebui-não-conecta-no-vllm)
9. [docker: Cannot connect to the Docker daemon](#erro-9-docker-cannot-connect-to-the-docker-daemon)
10. [WSL: pin_memory=False warning](#erro-10-wsl-pin_memoryfalse-warning)

---

## Erro 1: `api_server.py: error: unrecognized arguments: vllm serve`

**Mensagem completa:**
```
api_server.py: error: unrecognized arguments: vllm serve Qwen/Qwen2.5-3B-Instruct-AWQ
```

**Causa:**
A imagem `vllm/vllm-openai` usa `api_server.py` como entrypoint. Passar `vllm serve` como `command:` no docker-compose faz com que esses argumentos sejam passados diretamente para `api_server.py`, que não entende o subcomando `vllm serve`.

**Solução:**
No `docker-compose.yml`, o `command:` deve começar diretamente com `--model`:

```yaml
# ❌ ERRADO
command: >
  vllm serve Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0

# ✅ CORRETO
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
```

---

## Erro 2: `api_server.py: error: unrecognized arguments: Qwen/...`

**Mensagem completa:**
```
api_server.py: error: unrecognized arguments: Qwen/Qwen2.5-3B-Instruct-AWQ
```

**Causa:**
No vLLM v0.9+, o nome do modelo não pode ser passado como argumento posicional. Deve ser passado com a flag `--model`.

**Solução:**
```yaml
# ❌ ERRADO (argumento posicional)
command: >
  Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0

# ✅ CORRETO (flag explícita)
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
```

---

## Erro 3: `VLLM_USE_V1=1 is not supported with Compute Capability < 8.0`

**Mensagem completa:**
```
NotImplementedError: VLLM_USE_V1=1 is not supported with Compute Capability < 8.0.
```

**Causa:**
A variável de ambiente `VLLM_USE_V1=1` força o novo engine V1 do vLLM, que requer GPUs Ampere ou mais recentes (CC ≥ 8.0). GPUs Turing (RTX 20xx) têm CC 7.5 e não são suportadas pelo V1.

**Solução:**
Remova `VLLM_USE_V1=1` do `environment` no `docker-compose.yml`. O vLLM vai automaticamente usar o engine V0 e exibir apenas um warning (não um erro).

```yaml
# ❌ REMOVER estas linhas para GPUs CC < 8.0
environment:
  - VLLM_USE_V1=1
```

O vLLM cai automaticamente para V0 com este aviso (que é normal):
```
WARNING: Compute Capability < 8.0 is not supported by the V1 Engine. Falling back to V0.
```

---

## Erro 4: GPU não detectada no Docker

**Sintomas:**
- `docker run --gpus all nvidia/cuda:... nvidia-smi` falha
- Container sobe mas sem GPU

**Diagnóstico:**
```bash
# Verificar se NVIDIA Container Toolkit está instalado
nvidia-ctk --version

# Verificar se o Docker consegue ver a GPU
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

**Soluções:**

1. **NVIDIA Container Toolkit não instalado:**
   Siga o [guia de instalação](01-installation.md#4-instalar-nvidia-container-toolkit).

2. **Driver NVIDIA desatualizado:**
   Atualize para o driver mais recente em: https://www.nvidia.com/drivers

3. **No Windows/WSL2 — Docker Desktop sem GPU:**
   Ative nas configurações do Docker Desktop:
   - Settings → Resources → WSL Integration
   - Marque sua distro Ubuntu

4. **Reiniciar Docker após instalação do toolkit:**
   ```bash
   sudo systemctl restart docker
   # ou no Windows: reiniciar Docker Desktop
   ```

---

## Erro 5: Container fica em "health: starting" para sempre

**Diagnóstico:**
```bash
docker compose ps
# Mostra: vllm-qwen   Up X minutes (health: starting)

docker logs vllm-qwen --tail 20
```

**Causas possíveis:**

1. **Modelo ainda sendo baixado** — Normal na primeira vez (~2-5 min para modelos AWQ)
   - Aguarde e verifique os logs
   - Deve aparecer: `Loading weights took X.XX seconds`

2. **VRAM insuficiente** — Ver [Erro 7: CUDA out of memory](#erro-7-cuda-out-of-memory)

3. **Token HuggingFace inválido** — Ver [Erro 6](#erro-6-erro-403-ao-baixar-modelo-do-huggingface)

4. **Healthcheck muito restrito** — Aumente `retries` no compose:
   ```yaml
   healthcheck:
     retries: 20   # de 10 para 20
   ```

---

## Erro 6: Erro 403 ao baixar modelo do HuggingFace

**Nos logs:**
```
huggingface_hub.errors.RepositoryNotFoundError: 403 Forbidden
```
ou
```
OSError: We couldn't connect to 'https://huggingface.co'
```

**Causas:**

1. **Token não configurado ou inválido:**
   - Verifique o arquivo `.env`: `HF_TOKEN=hf_SeuToken`
   - Confirme que o token existe em: https://huggingface.co/settings/tokens

2. **Licença não aceita:**
   - Acesse a página do modelo no HuggingFace
   - Clique em "Agree and access repository"
   - Isso é obrigatório para modelos "gated" como Qwen

3. **Token sem permissão de leitura:**
   - No HuggingFace, crie um token com tipo **"Read"**

---

## Erro 7: CUDA out of memory

**Nos logs:**
```
torch.cuda.OutOfMemoryError: CUDA out of memory.
```

**Causas e soluções:**

1. **Modelo muito grande para a GPU:**
   Troque por um modelo menor em `.env`:
   ```env
   VLLM_MODEL=Qwen/Qwen2.5-1.5B-Instruct-AWQ
   ```

2. **`--gpu-memory-utilization` muito alto:**
   Reduza para liberar margem:
   ```yaml
   --gpu-memory-utilization 0.75  # de 0.82 para 0.75
   ```

3. **`--max-model-len` muito grande:**
   ```yaml
   --max-model-len 2048  # de 3072 para 2048
   ```

4. **Outro processo usando a GPU:**
   ```bash
   # Ver o que está usando a GPU
   nvidia-smi
   
   # Matar processo (substitua PID)
   sudo kill -9 PID
   ```

---

## Erro 8: OpenWebUI não conecta no vLLM

**Sintomas:**
- OpenWebUI abre mas não mostra modelos
- Erro "Connection refused" no OpenWebUI

**Diagnóstico:**
```bash
# Testar a API do vLLM diretamente
curl http://localhost:8000/v1/models

# Ver se o vLLM está healthy
docker compose ps
```

**Soluções:**

1. **vLLM ainda não está healthy:**
   Aguarde o modelo carregar (ver logs com `docker compose logs -f vllm`)

2. **OpenWebUI subiu antes do vLLM:**
   ```bash
   docker compose restart open-webui
   ```

3. **URL da API errada no OpenWebUI:**
   Verifique no docker-compose:
   ```yaml
   - OPENAI_API_BASE_URL=http://vllm:8000/v1  # "vllm" = nome do serviço
   ```
   Dentro do Docker, use o nome do serviço, não `localhost`.

---

## Erro 9: `docker: Cannot connect to the Docker daemon`

**Causa:** Docker Desktop não está rodando.

**Solução Windows:**
1. Abra o menu Iniciar
2. Procure "Docker Desktop"
3. Abra e aguarde o ícone 🐳 ficar verde na bandeja do sistema (~30-60 segundos)
4. Depois execute o compose novamente

---

## Erro 10: WSL: `pin_memory=False` warning

**Mensagem:**
```
WARNING: Using 'pin_memory=False' as WSL is detected. This may slow down the performance.
```

**Causa:** No WSL2, o Docker não pode usar "pinned memory" do CUDA (memória paginada bloqueada), o que é uma limitação da virtualização WSL.

**Impacto:** Leve redução de performance na transferência de dados CPU↔GPU.

**Solução:** Não há solução direta para WSL. Para performance máxima, use Linux nativo. Para uso no Windows, a diferença na prática é pequena e o aviso pode ser ignorado.

---

## Comandos de diagnóstico rápido

```bash
# Ver status de todos os containers
docker compose ps

# Ver logs de todos os serviços
docker compose logs --tail 50

# Ver uso de GPU
nvidia-smi

# Ver uso de memória dos containers
docker stats

# Reiniciar tudo do zero (sem apagar dados)
docker compose down && docker compose up -d

# Reiniciar e apagar TUDO (inclusive modelos baixados)
docker compose down -v && docker compose up -d
```
