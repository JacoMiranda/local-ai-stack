# 🤖 Integração com n8n (Workflows Automáticos)

O **n8n** é uma ferramenta de automação visual (estilo Zapier) muito poderosa que roda localmente na nossa stack. 

Como o **vLLM** expõe uma API 100% compatível com a da OpenAI, podemos usar os nós (nodes) padrão da OpenAI dentro do n8n para conversar com o modelo local sem pagar nada.

---

## 1. Acessando o n8n

1. Com a stack rodando, abra no navegador: **http://localhost:5678**
2. Na primeira vez, ele pedirá para você criar um usuário e senha locais.
3. Após criar, você cairá na tela inicial de Workflows.

---

## 2. Criando seu primeiro Workflow com IA (via HTTP Request)

Como o n8n atualiza frequentemente seus nós nativos da OpenAI (muitas vezes tentando acessar rotas da API de Assistants como `/v1/responses`), a forma **mais segura e garantida** de usar o vLLM local é através do nó universal **HTTP Request**.

1. Vá em **Workflows** → **Add Workflow**
2. Adicione um gatilho como o **Manual Trigger** (para rodar manualmente).
3. Adicione um novo nó e busque por **HTTP Request**.
4. Configure o nó com os padrões da API OpenAI:
   - **Method:** `POST`
   - **URL:** `http://vllm:8000/v1/chat/completions`
   - **Authentication:** `None` *(não precisamos de token internamente)*
   - **Send Body:** `ON`
   - **Body Content Type:** `JSON`

5. Na caixa de **JSON / Body Parameters**, cole a estrutura de conversa:
```json
{
  "model": "qwen",
  "messages": [
    {
      "role": "user",
      "content": "Responda em uma frase: O que é a Teoria da Relatividade?"
    }
  ]
}
```

> 💡 **Nota sobre a URL e Modelo:**
> Usamos `http://vllm:8000/v1` porque ambos estão na mesma rede do Docker Compose (nome do serviço "vllm"). O `model` no JSON deve bater com o nome carregado no vLLM (padrão do projeto é `qwen`).

---

## 3. Testando e Lendo a Resposta

1. Clique no botão inferior **Test step**.
2. O n8n fará uma requisição direta ao seu modelo na GPU.
3. No painel de **Output** (Saída), você receberá um JSON completo. A resposta do LLM fica sempre no caminho:
   `body.choices[0].message.content`
4. Você pode usar um nó **Set** (ou Edit Fields) logo depois para extrair apenas essa variável e passá-la limpa para o próximo passo do seu fluxo!

---

## 💡 Ideias de automações

Agora você pode usar o nó OpenAI do vLLM em qualquer fluxo do n8n:

- **Classificação de Emails:** Ler emails do Gmail/Outlook e usar a IA local para extrair intenção ou classificar se é urgente.
- **Resumo de Notícias:** Puxar feeds RSS, usar a IA para resumir o conteúdo e postar num canal do Telegram/Discord.
- **Tradução Automática:** Converter textos recebidos em planilhas ou bancos de dados sem enviar os dados para APIs de terceiros.
- **Agentes de Atendimento:** Ligar o nó de Webhook do WhatsApp/Telegram ao modelo LLM local para responder clientes automaticamente.
