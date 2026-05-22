# 🧪 Diário de Bordo — Setup da Stack Local de IA

> Relato real e cronológico de todos os erros encontrados e corrigidos ao montar esta stack do zero, com uma NVIDIA RTX 2060 12GB no Windows 11 com WSL2.

Este documento existe para que você **não precise passar pelos mesmos problemas**. Cada erro abaixo foi encontrado em produção, diagnosticado nos logs e corrigido.

---

## Ambiente

| Componente | Especificação |
|---|---|
| GPU | NVIDIA GeForce RTX 2060 12 GB |
| Arquitetura | Turing — Compute Capability **7.5** |
| OS | Windows 11 + WSL2 |
| Docker | Docker Desktop + NVIDIA Container Toolkit |
| vLLM | v0.9.2 |
| Modelo | Qwen/Qwen2.5-3B-Instruct-AWQ |

---

## Problema 1 — `api_server.py: error: unrecognized arguments: vllm serve`

### O que aconteceu

Ao subir o container pela primeira vez com o `docker-compose.yml`:

```yaml
command: >
  vllm serve Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
  ...
```

O container subia e imediatamente fechava. Nos logs:

```
api_server.py: error: unrecognized arguments: vllm serve Qwen/Qwen2.5-3B-Instruct-AWQ
```

### Diagnóstico

A imagem `vllm/vllm-openai` usa `api_server.py` como **entrypoint**. O campo `command:` no Docker Compose não substitui o entrypoint — ele **adiciona argumentos** a ele. Logo, `vllm serve` era passado como argumento para `api_server.py`, que não entende esse subcomando.

A confusão é natural: na CLI você roda `vllm serve modelo`, mas dentro do Docker o `vllm serve` já é o entrypoint implícito da imagem.

### Correção

```yaml
# ❌ Antes
command: >
  vllm serve Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0

# ✅ Depois — só os argumentos, sem o subcomando
command: >
  Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
```

---

## Problema 2 — `api_server.py: error: unrecognized arguments: Qwen/...`

### O que aconteceu

Após remover `vllm serve`, novo erro:

```
api_server.py: error: unrecognized arguments: Qwen/Qwen2.5-3B-Instruct-AWQ
```

### Diagnóstico

No **vLLM v0.9+**, o nome do modelo deixou de ser aceito como argumento posicional. Precisa ser passado explicitamente com a flag `--model`.

### Correção

```yaml
# ❌ Antes — argumento posicional (removido no v0.9)
command: >
  Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0

# ✅ Depois — flag explícita
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
```

---

## Problema 3 — `VLLM_USE_V1=1 is not supported with Compute Capability < 8.0`

### O que aconteceu

O container subia mas travava durante a inicialização do engine:

```
NotImplementedError: VLLM_USE_V1=1 is not supported with Compute Capability < 8.0.
```

### Diagnóstico

O `docker-compose.yml` original tinha `VLLM_USE_V1=1` no `environment`. O engine V1 do vLLM exige GPUs **Ampere (RTX 30xx) ou mais recentes**, com CC ≥ 8.0. A RTX 2060 tem CC 7.5 (Turing) e não é suportada.

### Correção

```yaml
# ❌ Remover — engine V1 não suporta CC 7.5
environment:
  - VLLM_USE_V1=1
```

Sem essa variável, o vLLM detecta automaticamente e cai para o engine V0, exibindo apenas um aviso (não fatal):

```
WARNING: Compute Capability < 8.0 is not supported by the V1 Engine. Falling back to V0.
```

---

## Problema 4 — Crash na segunda interação (prefix caching + Triton)

### O que aconteceu

Após subir com sucesso, o modelo respondia normalmente na **primeira mensagem**. Na **segunda**, o OpenWebUI travava e o container morria:

```
/usr/local/lib/python3.12/dist-packages/vllm/attention/ops/prefix_prefill.py:36:0:
error: Failures have been detected while processing an MLIR pass pipeline

Pipeline failed while executing [`ConvertTritonGPUToLLVM` on 'builtin.module' operation]:
compute-capability=75

INFO:     Shutting down
INFO:     Application shutdown complete.
INFO:     Finished server process [1]
```

### Diagnóstico

O flag `--enable-prefix-caching` ativa um kernel Triton customizado (`prefix_prefill.py`) para reutilizar partes do KV cache (como o system prompt) entre requests. Esse kernel é compilado **na GPU via MLIR/Triton na primeira execução** — que acontece na segunda interação da conversa, quando há contexto anterior para cachear.

O compilador Triton falha ao gerar código LLVM para CC 7.5. O erro é silencioso até esse momento.

### Correção

```yaml
# ❌ Remover — kernel Triton incompatível com CC 7.5
--enable-prefix-caching
```

**Impacto:** Pequena perda de performance em casos com system prompts longos repetidos. Para conversas normais, impacto zero.

**Quando é seguro usar:** GPUs Ampere (RTX 30xx) ou mais recentes (CC ≥ 8.0).

---

## Problema 5 — Modelo só gerava ~10 linhas

### O que aconteceu

Ao pedir ao modelo para escrever uma carta longa (50 linhas), ele produzia apenas ~10 linhas e parava. Não havia erro — o modelo simplesmente considerava que tinha terminado.

### Diagnóstico

Nos logs de inicialização do vLLM havia este aviso (ignorado inicialmente):

```
WARNING: Default sampling parameters have been overridden by the model's
Hugging Face generation config recommended from the model creator.
If this is not intended, please relaunch vLLM instance with `--generation-config vllm`.

Using default chat sampling params from model:
{'repetition_penalty': 1.05, 'temperature': 0.7, 'top_k': 20, 'top_p': 0.8}
```

O `generation_config.json` do Qwen2.5 no HuggingFace define:

```json
{
  "max_new_tokens": 512
}
```

512 tokens ≈ 350 palavras ≈ ~10 linhas curtas. O vLLM estava obedecendo esse limite definido pelo autor do modelo, truncando respostas longas silenciosamente.

### Correção

```yaml
# ✅ Adicionar — ignora o generation_config.json do modelo
--generation-config vllm
```

Com isso, o limite de saída passa a ser o contexto total disponível (`--max-model-len` − tokens do prompt), permitindo respostas muito mais longas.

**Observação:** O warning de inicialização desaparece após essa correção, confirmando que o config do modelo foi ignorado com sucesso.

---

## Estado final — flags de compatibilidade para RTX 2060 (CC 7.5)

Após todos os ajustes, o `command:` do vLLM ficou:

```yaml
command: >
  --model Qwen/Qwen2.5-3B-Instruct-AWQ
  --host 0.0.0.0
  --port 8000
  --served-model-name qwen
  --quantization awq
  --dtype float16                    # bfloat16 não suportado em Turing
  --gpu-memory-utilization 0.82
  --max-model-len 3072
  --max-num-seqs 16
  --max-num-batched-tokens 3072
  --tensor-parallel-size 1
  --enforce-eager                    # obrigatório: desabilita CUDA graphs (CC < 8.0)
  --generation-config vllm           # ignora max_new_tokens=512 do modelo
```

E no `environment`, removidos:
```yaml
# ❌ Removido — V1 engine não suporta CC 7.5
# - VLLM_USE_V1=1

# ❌ Removido — kernel Triton crasha em CC 7.5
# --enable-prefix-caching
```

---

## Recursos de hardware confirmados em funcionamento

```
GPU: RTX 2060 12GB (Turing, CC 7.5)
Engine: vLLM V0 (fallback automático)
Attention backend: XFormers (FlashAttention-2 requer CC 8.0+)
Model weights: 1.95 GiB (AWQ quantizado)
KV Cache: 7.56 GiB disponíveis
CUDA blocks: 13.764
Concorrência máxima: 71x (a 3072 tokens/req)
Tempo de carga (cold start): ~4 minutos (download incluso)
Tempo de carga (warm start): ~2 segundos (modelo em cache no volume)
```

---

## Lições aprendidas

1. **Sempre pin a versão da imagem Docker** — `vllm/vllm-openai:v0.9.2`, não `latest`
2. **Verifique o Compute Capability da sua GPU** antes de copiar qualquer compose da internet
3. **Leia os warnings de inicialização** — o aviso do `generation_config` era a pista do problema 5
4. **Volumes Docker** são essenciais: sem eles, o modelo (~2GB) é re-baixado a cada restart
5. **O `command:` no Docker Compose não substitui o entrypoint** — ele adiciona argumentos
6. **Erros que aparecem só na segunda interação** são os mais difíceis — o prefix caching só ativa quando há contexto para reutilizar
