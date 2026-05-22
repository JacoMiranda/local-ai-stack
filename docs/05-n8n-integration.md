# 🤖 Integração com n8n (Workflows Automáticos)

O **n8n** é uma ferramenta de automação visual (estilo Zapier) muito poderosa que roda localmente na nossa stack. 

Como o **vLLM** expõe uma API 100% compatível com a da OpenAI, podemos usar os nós (nodes) padrão da OpenAI dentro do n8n para conversar com o modelo local sem pagar nada.

---

## 1. Acessando o n8n

1. Com a stack rodando, abra no navegador: **http://localhost:5678**
2. Na primeira vez, ele pedirá para você criar um usuário e senha locais.
3. Após criar, você cairá na tela inicial de Workflows.

---

## 2. Configurando a credencial do vLLM (como se fosse OpenAI)

Para que o n8n fale com o seu modelo, precisamos criar uma credencial do tipo "OpenAI API" apontando para o servidor local em vez da nuvem.

1. No menu esquerdo do n8n, clique em **Credentials**
2. Clique no botão superior direito **Add Credential**
3. Busque por **OpenAI** e selecione a opção **OpenAI API**
4. Configure assim:
   - **Name:** `vLLM Local API`
   - **API Key:** `EMPTY` *(o vLLM não valida a chave, mas o n8n exige que o campo não fique vazio)*
   - **Base URL:** `http://vllm:8000/v1`
5. Clique em **Save**

> 💡 **Por que `http://vllm:8000/v1`?**
> Como o n8n e o vLLM estão na mesma rede do Docker Compose, eles conseguem se encontrar pelo nome do serviço (`vllm`). Não use `localhost` aqui, pois `localhost` dentro do n8n apontaria para o próprio container do n8n.

---

## 3. Criando seu primeiro Workflow com IA

Agora vamos testar a geração de texto.

1. Volte ao menu principal e clique em **Workflows** → **Add Workflow**
2. Clique no botão de **+** no meio da tela para adicionar o primeiro nó
3. Busque pelo nó **Manual Trigger** (gatilho para rodar manualmente)
4. Adicione um novo nó, busque por **OpenAI** e selecione.
5. Configure o nó OpenAI:
   - **Resource:** `Chat`
   - **Operation:** `Generate a Text completion`
   - **Credential for OpenAI API:** Selecione a credencial que você criou (`vLLM Local API`)
   - **Model:** O n8n vai carregar a lista de modelos disponíveis direto do vLLM. Selecione `qwen` (ou o modelo que você configurou no `.env`).
   - **Messages:**
     - Clique em `Add Message`
     - Role: `User`
     - Content: `Responda em uma frase: O que é a Teoria da Relatividade?`

---

## 4. Testando o Workflow

1. Feche as configurações do nó
2. Clique no botão inferior **Test Workflow**
3. O n8n fará uma requisição à API do vLLM. Em poucos segundos, o nó da OpenAI ficará verde.
4. Clique no nó da OpenAI para ver o resultado — a resposta do modelo estará na aba **Output** (saída).

---

## 💡 Ideias de automações

Agora você pode usar o nó OpenAI do vLLM em qualquer fluxo do n8n:

- **Classificação de Emails:** Ler emails do Gmail/Outlook e usar a IA local para extrair intenção ou classificar se é urgente.
- **Resumo de Notícias:** Puxar feeds RSS, usar a IA para resumir o conteúdo e postar num canal do Telegram/Discord.
- **Tradução Automática:** Converter textos recebidos em planilhas ou bancos de dados sem enviar os dados para APIs de terceiros.
- **Agentes de Atendimento:** Ligar o nó de Webhook do WhatsApp/Telegram ao modelo LLM local para responder clientes automaticamente.
