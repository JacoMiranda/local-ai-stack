## Erro 11: Crash na segunda interação (prefix caching + Triton CC 7.5)

**Sintoma:**
- Primeira mensagem no OpenWebUI funciona normalmente
- Segunda mensagem trava e dá erro — vLLM derruba o servidor

**Nos logs:**
```
prefix_prefill.py:36:0: error: Failures have been detected while processing an MLIR pass pipeline
Pipeline failed while executing [ConvertTritonGPUToLLVM ... compute-capability=75]
INFO: Shutting down
INFO: Application shutdown complete.
```

**Causa:**
`--enable-prefix-caching` usa um kernel Triton customizado (`prefix_prefill.py`) para reutilizar KV cache de prefixos em comum (ex: system prompt). Esse kernel é compilado via MLIR/Triton na primeira vez que é acionado — o que acontece **na segunda interação** da conversa. O compilador Triton falha ao gerar código LLVM para GPUs com Compute Capability < 8.0 (Turing/Volta).

**Solução:**
Remova `--enable-prefix-caching` do comando do vLLM:

```yaml
# ❌ REMOVER — causa crash em CC 7.5
--enable-prefix-caching
```

**Impacto de remover:**
Pequena perda de performance quando há system prompts longos repetidos entre requests. Para uso normal (conversas sequenciais), o impacto é zero.

**Quando é seguro usar:**
Apenas em GPUs Ampere (RTX 30xx) ou mais recentes (CC ≥ 8.0).
