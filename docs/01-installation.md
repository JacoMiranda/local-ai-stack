# 📦 Instalação Completa do Zero

Este guia cobre tudo que você precisa instalar para rodar a stack, do zero ao chat funcionando.

---

## Índice

1. [Pré-requisitos de hardware](#1-pré-requisitos-de-hardware)
2. [Windows: instalar WSL2](#2-windows-instalar-wsl2)
3. [Instalar Docker Desktop](#3-instalar-docker-desktop)
4. [Instalar NVIDIA Container Toolkit](#4-instalar-nvidia-container-toolkit)
5. [Criar conta no HuggingFace e obter token](#5-criar-conta-no-huggingface-e-obter-token)
6. [Aceitar licença do modelo](#6-aceitar-licença-do-modelo)
7. [Clonar e configurar o projeto](#7-clonar-e-configurar-o-projeto)
8. [Subir a stack](#8-subir-a-stack)
9. [Primeiro acesso ao OpenWebUI](#9-primeiro-acesso-ao-openwebui)

---

## 1. Pré-requisitos de hardware

Antes de começar, confirme que sua máquina atende aos requisitos mínimos:

| Componente | Mínimo | Recomendado |
|---|---|---|
| GPU NVIDIA | 6 GB VRAM | 8-12 GB VRAM |
| RAM | 8 GB | 16 GB+ |
| Armazenamento | 10 GB livres | 20 GB+ livres |
| OS | Windows 10/11, Linux | Windows 11 ou Ubuntu 22.04+ |

> ❌ **AMD e Intel Arc** não são suportados pelo vLLM (requer CUDA). Para uso sem GPU, veja [docs/03-hardware.md](03-hardware.md#sem-gpu-cpu-only).

---

## 2. Windows: instalar WSL2

> **Pule esta etapa se usar Linux nativo.**

O Docker Desktop no Windows usa WSL2 (Windows Subsystem for Linux) como backend. É necessário instalá-lo primeiro.

**Abra o PowerShell como Administrador e execute:**

```powershell
wsl --install
```

Isso instala WSL2 com Ubuntu automaticamente. **Reinicie o computador** quando solicitado.

Após reiniciar, o Ubuntu vai abrir e pedir para criar usuário/senha. Faça isso.

**Verificar se o WSL2 está funcionando:**
```powershell
wsl --status
# Deve mostrar: Default Version: 2
```

---

## 3. Instalar Docker Desktop

1. Acesse: https://www.docker.com/products/docker-desktop/
2. Baixe o instalador para **Windows (AMD64)**
3. Execute o instalador
4. Na tela de opções, **marque "Use WSL 2 instead of Hyper-V"**
5. Finalize e reinicie se pedido

**Verificar instalação:**
```powershell
docker --version
# Ex: Docker version 27.x.x

docker compose version
# Ex: Docker Compose version v2.x.x
```

> ⚠️ Se aparecer erro `"docker daemon not running"`, abra o Docker Desktop pelo menu iniciar e aguarde o ícone ficar verde na bandeja do sistema.

---

## 4. Instalar NVIDIA Container Toolkit

Este passo permite que containers Docker acessem a GPU NVIDIA.

### 4.1. No Windows (via WSL2)

Abra o terminal Ubuntu (WSL2) e execute:

```bash
# Adicionar repositório NVIDIA
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Instalar
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configurar Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 4.2. No Linux nativo (Ubuntu/Debian)

Mesmo processo acima. Após configurar, reinicie o Docker:
```bash
sudo systemctl restart docker
```

### 4.3. Verificar GPU no Docker

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Deve aparecer a tabela com sua GPU. Se aparecer erro, veja [docs/04-troubleshooting.md](04-troubleshooting.md).

---

## 5. Criar conta no HuggingFace e obter token

HuggingFace é onde os modelos são hospedados. Você precisa de uma conta gratuita e um token de acesso.

1. Acesse: https://huggingface.co/join
2. Crie sua conta (gratuito)
3. Acesse: https://huggingface.co/settings/tokens
4. Clique em **"New token"**
5. Dê um nome (ex: `local-ai-stack`)
6. Selecione tipo **"Read"**
7. Clique em **"Generate a token"**
8. **Copie e guarde o token** — ele começa com `hf_`

---

## 6. Aceitar licença do modelo

Alguns modelos no HuggingFace exigem aceite de licença antes de baixar.

Para o **Qwen2.5-3B-Instruct-AWQ**:

1. Acesse: https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-AWQ
2. Logue com sua conta
3. Se houver um banner de licença, clique em **"Agree and access repository"**

> Sem esse aceite, o download vai falhar com erro 403 mesmo com token válido.

---

## 7. Clonar e configurar o projeto

```bash
# Clonar o repositório
git clone https://github.com/SEU_USUARIO/local-ai-stack.git
cd local-ai-stack

# Criar arquivo .env a partir do exemplo
cp .env.example .env
```

Agora edite o `.env` com seu editor de texto:

```bash
# Linux/WSL
nano .env

# Windows (PowerShell)
notepad .env
```

Preencha o `HF_TOKEN` com o token que você copiou:

```env
HF_TOKEN=hf_SeuTokenRealAqui
VLLM_MODEL=Qwen/Qwen2.5-3B-Instruct-AWQ
SERVED_MODEL_NAME=qwen
```

Salve e feche o arquivo.

---

## 8. Subir a stack

```bash
docker compose up -d
```

O que acontece na **primeira execução**:
1. Docker baixa as imagens (~5-10 GB total)
2. vLLM baixa o modelo do HuggingFace (~2 GB para o 3B-AWQ)
3. vLLM carrega o modelo na GPU (~4 minutos no RTX 2060)
4. OpenWebUI sobe (aguarda vLLM ficar healthy)

**Acompanhe o progresso:**
```bash
# Ver tudo em tempo real
docker compose logs -f

# Ver só o vLLM
docker compose logs -f vllm
```

Quando você ver esta mensagem, está pronto:
```
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

**Verificar status:**
```bash
docker compose ps
# Todos devem mostrar "Up" e vllm deve mostrar "(healthy)"
```

---

## 9. Primeiro acesso ao OpenWebUI

1. Abra o navegador em: **http://localhost:3000**
2. Na primeira vez, clique em **"Sign up"** para criar uma conta de admin
3. Use qualquer email e senha (são salvos localmente, não vai para nenhum servidor)
4. O modelo `qwen` já deve estar disponível no seletor de modelos
5. Comece a conversar! 🎉

### Habilitar busca na web

1. No OpenWebUI, clique no ícone de configurações ⚙️
2. Vá em **"Web Search"**
3. Habilite a busca
4. Selecione **SearXNG** como engine
5. Agora você pode usar o ícone 🌐 nas conversas para buscar na web

---

## ✅ Checklist final

- [ ] WSL2 instalado (Windows)
- [ ] Docker Desktop rodando
- [ ] `docker run --gpus all nvidia/cuda:... nvidia-smi` funciona
- [ ] `.env` criado com token HuggingFace válido
- [ ] Licença do modelo aceita no HuggingFace
- [ ] `docker compose up -d` executado
- [ ] `docker compose ps` mostra todos os serviços como `Up`
- [ ] http://localhost:3000 abre o OpenWebUI
