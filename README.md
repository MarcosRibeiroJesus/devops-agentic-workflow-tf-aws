# DevOps com Agentes de IA · Claude Code, Terraform e AWS

Slide deck e material de apoio para a live sobre workflows agênticos de DevOps usando Claude Code, Terraform e AWS (S3 + CloudFront).

## Pré-requisitos

Para acompanhar e reproduzir a demonstração na sua própria máquina, instale e verifique estas ferramentas antes da live:

- **AWS CLI** (com credenciais configuradas, `aws configure`)
- **Terraform**
- **Docker** (necessário para rodar o MCP server oficial da HashiCorp)
- **uv / uvx** (usado para subir o MCP server da AWS)
- **Claude Code** (CLI instalada e autenticada)
- **Conta AWS** (com permissão para criar recursos S3 e CloudFront)
- **Conta no GitHub**
- **Um editor de código** (recomendado: VS Code)

Trazer essas ferramentas já instaladas garante que você consiga rodar a esteira completa junto com a live: `/scaffold-terraform` → `/tf-plan` → `/tf-apply` → `/deploy`.

---

## Instalação — Linux (e WSL no Windows)

As instruções abaixo valem tanto para uma distro Linux nativa quanto para o WSL (Windows Subsystem for Linux) rodando Ubuntu/Debian. Exemplos usam `apt`; adapte para `dnf`/`pacman` se sua distro for outra.

### 0. WSL (apenas se estiver no Windows)

Se ainda não tem o WSL instalado, abra o PowerShell como administrador e rode:

```powershell
wsl --install
```

Isso instala o WSL2 com Ubuntu como distro padrão. Reinicie a máquina quando solicitado, abra o Ubuntu pelo menu Iniciar e conclua a criação do usuário. Todos os passos a seguir devem ser executados **dentro do terminal WSL (Ubuntu)**, não no PowerShell/CMD.

> Instale o Docker Desktop for Windows com a integração WSL2 habilitada (ver seção Docker abaixo) em vez de instalar o Docker diretamente dentro do WSL — é o caminho recomendado pela própria Docker.

### 1. AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verifique:

```bash
aws --version
```

Configure suas credenciais:

```bash
aws configure
```

Informe `AWS Access Key ID`, `AWS Secret Access Key`, região padrão (ex.: `us-east-1`) e formato de saída (ex.: `json`).

### 2. Terraform

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

Verifique:

```bash
terraform -version
```

### 3. Docker

No Windows com WSL: instale o **Docker Desktop for Windows**, e nas configurações habilite *Settings → Resources → WSL Integration* para a sua distro (ex.: Ubuntu). O comando `docker` fica disponível dentro do terminal WSL automaticamente.

Em Linux nativo:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Após instalar, feche e reabra o terminal (ou faça logout/login) para o grupo `docker` valer. Verifique:

```bash
docker --version
docker run hello-world
```

### 4. uv / uvx

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Reabra o terminal ou rode `source $HOME/.local/bin/env` para carregar o `uv` no `PATH`. Verifique:

```bash
uv --version
uvx --version
```

### 5. Claude Code

Requer Node.js 18+. Se ainda não tem Node instalado:

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

Instale o Claude Code:

```bash
npm install -g @anthropic-ai/claude-code
```

Autentique:

```bash
claude
```

Na primeira execução, siga o fluxo de login exibido no terminal (via navegador). Verifique:

```bash
claude --version
```

### 6. Conta AWS

Crie uma conta em [aws.amazon.com](https://aws.amazon.com) caso não tenha. O usuário/role usado no `aws configure` precisa de permissão para criar e gerenciar recursos S3 e CloudFront (para a live, uma policy administrativa ou uma policy customizada cobrindo `s3:*` e `cloudfront:*` é suficiente).

### 7. Conta no GitHub

Crie uma conta em [github.com](https://github.com) caso não tenha — necessária para o fluxo de CI/CD com GitHub Actions e OIDC usado no projeto.

### 8. Editor de código

Recomendado: [Visual Studio Code](https://code.visualstudio.com/). No Windows, instale o VS Code no Windows (não dentro do WSL) e use a extensão **WSL** (`ms-vscode-remote.remote-wsl`) para abrir o projeto diretamente na distro Linux — assim o terminal integrado já roda no WSL.

---

## Checklist rápido de verificação

Rode tudo junto para conferir se está tudo pronto:

```bash
aws --version && \
terraform -version && \
docker --version && \
uv --version && uvx --version && \
claude --version && \
aws sts get-caller-identity
```

Se todos os comandos retornarem versão/identidade sem erro, seu ambiente está pronto para rodar `/scaffold-terraform`, `/tf-plan`, `/tf-apply` e `/deploy` durante a live.
