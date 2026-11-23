# Shared Workflows | TETRIX Platform
### Repositório Central de Workflows Reutilizáveis

[![GitHub Actions](https://img.shields.io/badge/github--actions-enabled-blue?logo=github-actions)](https://github.com/actions)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-green?logo=pre-commit)](https://github.com/pre-commit/pre-commit)
[![AWS](https://img.shields.io/badge/aws-enabled-orange?logo=amazon)](https://aws.amazon.com/)
[![Google Cloud](https://img.shields.io/badge/google--cloud-enabled-blue?logo=google-cloud)](https://cloud.google.com/)
[![OCI](https://img.shields.io/badge/oci-enabled-red?logo=oracle)](https://www.oracle.com/cloud/)
[![Terraform](https://img.shields.io/badge/terraform-version_1.12.1-3776AB?logo=terraform)](https://www.terraform.io/)
![GitHub](https://img.shields.io/badge/github-enabled-blue?logo=github)
[![Cyberarck Comjur](https://img.shields.io/badge/cyberark--conjur-enabled-green?logo=cyberark)](https://docs.cyberark.com/conjur-cloud/latest/en/content/resources/_topnav/cc_home.htm)
![licence](https://img.shields.io/badge/license-MIT-blue.svg)

# Java Build Workflow ☕

Este workflow reutilizável automatiza o processo de build, empacotamento e publicação de imagens Docker para aplicações Java baseadas em Maven. Ele é compatível com runners self-hosted que utilizam Podman/Buildah como container runtime.

## Funcionalidades

- 📥 Checkout do repositório
- ⚙️ Configuração de ambiente com Java + Maven (com cache)
- ☕ Build da aplicação com `mvn clean install -DskipTests`
- 🐳 Build de imagem Docker com `buildah`
- 🔐 Login no container registry
- 🚀 Push da imagem para o registro com a versão do `pom.xml`
- 📊 Sumário com metadados da execução na aba "Summary" do GitHub Actions

## Entradas

| Nome            | Tipo    | Descrição                          | Padrão    |
|-----------------|---------|------------------------------------|-----------|
| `java-version`  | string  | Versão do JDK                      | `"21"`    |
| `distribution`  | string  | Distribuição do JDK                | `"temurin"` |
| `maven-version` | string  | Versão do Maven                    | `"3.9.9"` |

## Variáveis de ambiente esperadas

| Variável         | Descrição                                       |
|------------------|-------------------------------------------------|
| `REGISTRY`       | Container registry (ex: `gcr.io`)               |
| `REGISTRY_USER`  | Usuário para autenticação                       |
| `REGISTRY_PASS`  | Senha/token para autenticação                   |
| `PROXY_*`        | Variáveis opcionais para proxy Maven (via JSON) |

## Permissões exigidas

- `contents: read`
- `actions: read`
- `pull-requests: write` (para publicar o `GITHUB_STEP_SUMMARY`)

---

# Go Build Workflow 🦭

Este workflow reutilizável automatiza o processo de build, teste e publicação de imagens Docker para projetos em Go. Ele foi projetado para ambientes com runners self-hosted que utilizam o Podman + Buildah como runtime/container builder.

## Funcionalidades

- 🚀 Checkout do código-fonte
- 📦 Resolução de dependências Go
- ⚒️ Compilação do binário Go (`go build`)
- 🧪 Execução dos testes unitários (`go test`)
- 🐳 Build da imagem Docker usando `buildah`
- 🔐 Login no registro de container (por padrão, `gcr.io`)
- 🚀 Push da imagem com tag curta (SHA-7)
- 📊 Geração de sumário com metadados no GitHub Actions

## Entradas

| Nome         | Tipo   | Descrição                      | Default |
|--------------|--------|-------------------------------|---------|
| `go-version` | string | Versão do Go utilizada         | `1.24`  |

## Variáveis de ambiente esperadas

- `REGISTRY` — Endereço do registro de container (ex: `gcr.io`)
- `REGISTRY_USER` — Usuário para autenticação no registry
- `REGISTRY_PASS` — Senha/token para autenticação no registry

## Permissões exigidas

O workflow solicita as seguintes permissões:

- `contents: read`
- `actions: read`
- `pull-requests: write` (necessário para gerar sumário com `GITHUB_STEP_SUMMARY`)

---

# Node Build Workflow 🌀

Este workflow reutilizável realiza o build e a publicação de imagens Docker para projetos Node.js com base no conteúdo do `package.json`. Ele é otimizado para runners self-hosted utilizando Podman + Buildah.

## ⚙️ Funcionalidades

- 📥 Checkout do repositório
- 🔍 Leitura automática de `node-version`, `version` e `name` do `package.json`
- ⚒️ Setup de ambiente Node.js com cache de dependências
- 🧱 Execução dos comandos `npm ci` e `npm run build`
- 🔐 Login no container registry
- 🐳 Build da imagem Docker com `buildah`
- 🚀 Push da imagem com a tag baseada na versão do `package.json`
- 📊 Geração de sumário com metadados no GitHub Actions

## 📥 Entradas

| Nome           | Tipo    | Obrigatório | Descrição                             | Padrão |
|----------------|---------|-------------|----------------------------------------|--------|
| `node-version` | string  | Não         | Versão do Node.js                     | `22`   |

> ⚠️ Se a chave `engines.node` estiver definida no `package.json`, ela será priorizada.

## 🌍 Variáveis de ambiente esperadas

| Variável         | Descrição                                      |
|------------------|------------------------------------------------|
| `REGISTRY`       | Container registry (ex: `gcr.io`)              |
| `REGISTRY_USER`  | Usuário para autenticação                      |
| `REGISTRY_PASS`  | Senha/token de acesso                          |

## 🔐 Permissões exigidas

- `contents: read`
- `actions: read`
- `pull-requests: write` (necessário para PRs do Dependabot)

## 📦 Resultado

- Imagem publicada com tag `version` definida em `package.json`.
- Sumário com:
  - Link para o container
  - Comando `docker pull`
  - SHA da execução
  - Versão usada do Node.js

---

# Python Build Workflow 🐍

Este workflow reutilizável automatiza o build e o push de imagens Docker para projetos Python. Ele é indicado para repositórios com `Dockerfile` e `requirements.txt`, rodando em runners self-hosted com Podman + Buildah.

## ⚙️ Funcionalidades

- 📥 Checkout do repositório
- 📄 Coleta de metadados do projeto (nome e versão do Python)
- 🐍 Setup do Python com cache de dependências via pip
- ⌛ Instalação dos pacotes a partir do `requirements.txt`
- 🔐 Login no container registry
- 🐳 Build da imagem Docker usando Buildah
- 🚀 Push da imagem com tag baseada no SHA curto
- 📊 Geração de sumário detalhado no GitHub Actions

## 📥 Entradas

| Nome             | Tipo    | Obrigatório | Descrição                          | Valor padrão |
|------------------|---------|-------------|-------------------------------------|--------------|
| `python-version` | string  | Não         | Versão do Python                    | `3.x`        |

## 🌍 Variáveis de ambiente esperadas

| Variável         | Descrição                                      |
|------------------|------------------------------------------------|
| `REGISTRY`       | Endereço do container registry (ex: `gcr.io`)  |
| `REGISTRY_USER`  | Usuário para autenticação                      |
| `REGISTRY_PASS`  | Senha/token para autenticação                  |

## 🔐 Permissões exigidas

- `contents: read`
- `actions: read`
- `pull-requests: write` (para sumário no PR/execução)

---

# Terraform Workflow 🌎

Este workflow reutilizável automatiza a execução completa do ciclo Terraform com integração ao CyberArk Conjur e autenticação no Google Cloud. Ele foi projetado para ambientes com múltiplos workspaces (`dev`, `staging`, `prod`, `qa`), usando `terraform workspace` e segregação de segredos por ambiente.

## ⚙️ Funcionalidades

- 📥 Checkout do repositório
- ⚒️ Setup de Node.js e Terraform
- 🔐 Integração com CyberArk Conjur para extração segura de segredos
- ☁️ Autenticação no Google Cloud via Service Account
- 🧪 Execução dos comandos Terraform:
  - `fmt`
  - `init`
  - `validate`
  - `plan`
  - `apply` (apenas em branches específicas)
- 📊 Geração automática de sumário com `terraform plan` em PRs

## 📥 Entradas (Inputs)

| Nome              | Tipo    | Obrigatório | Descrição                                                  | Valor padrão |
|-------------------|---------|-------------|-------------------------------------------------------------|--------------|
| `terraform-version` | string  | Não         | Versão do Terraform a ser usada                             | `1.12.2`     |
| `node-version`      | string  | Não         | Versão do Node.js para rodar scripts (ex: conjur-cli)       | `20`         |
| `secrets-path`      | string  | Sim         | Caminho base dos segredos no Conjur                         | —            |
| `secret-names`      | string  | Sim         | Lista de nomes dos segredos a serem exportados (espaço ou quebra de linha) | — |

## 🔐 Permissões exigidas

- `contents: read`
- `actions: read`
- `pull-requests: write` (para publicar comentários em PR com o `terraform plan`)

## 🌍 Variáveis de ambiente esperadas (injetadas via `GITHUB_ENV`)

- `WORKSPACE`: calculado dinamicamente com base no nome da branch
- `CONJUR_URL`: URL do appliance do Conjur
- `CONJUR_HOST_ID`: ID do host autenticador
- `CONJUR_API_KEY`: chave de API (segredo)

## 📦 Resultado esperado

- Execução segura de `terraform plan` e `terraform apply`
- Extração automática de segredos do Conjur
- Comentário automático com resultado do `plan` no PR
- Aplicação (`apply`) automática somente em branches: `develop`, `main`, `staging`

## 🌐 Diretório esperado

O `working-directory` padrão é `terraform/gke`, ou seja, os arquivos `.tf` devem estar nesse caminho no repositório.

---

# Get Conjur Secrets 🔐

Este workflow reutilizável tem como objetivo recuperar segredos armazenados no CyberArk Conjur e exportá-los como variáveis de ambiente seguras para uso em outros jobs ou workflows.

## ⚙️ Funcionalidades

- 📥 Checkout do repositório
- 🌍 Identificação automática do ambiente (`develop`, `main`, `staging`, `qa`, etc.)
- 🔐 Login no Conjur com credenciais dinâmicas por ambiente
- 🔄 Exportação de segredos como variáveis mascaradas no ambiente e como `outputs`
- 🔁 Mecanismo de retry com até 10 tentativas em caso de falhas temporárias

## 📥 Entradas (Inputs)

| Nome            | Tipo    | Obrigatório | Descrição                                              |
|-----------------|---------|-------------|---------------------------------------------------------|
| `secrets-path`  | string  | Sim         | Caminho base dos segredos no Conjur (ex: `apps/myapp`) |
| `secret-names`  | string  | Sim         | Lista de nomes dos segredos (separados por espaço ou quebra de linha) |

## 🔐 Permissões esperadas (via GitHub Secrets)

Este workflow espera que as seguintes secrets estejam disponíveis e configuradas por ambiente:

| Branch       | Secrets esperados                        |
|--------------|-------------------------------------------|
| `develop`    | `CONJUR_USERNAME_DEV`, `CONJUR_API_KEY_DEV` |
| `main`       | `CONJUR_USERNAME_PROD`, `CONJUR_API_KEY_PROD` |
| `staging`    | `CONJUR_USERNAME_STAGING`, `CONJUR_API_KEY_STAGING` |
| `qa`         | `CONJUR_USERNAME_QA`, `CONJUR_API_KEY_QA` |
| Outro        | `CONJUR_USERNAME_LAB`, `CONJUR_API_KEY_DEV` |
| Todas        | `CONJUR_URL`                             |

## 🔐 Segurança

- Os valores dos segredos são mascarados usando `::add-mask::`
- Os nomes dos segredos são validados por regex (somente letras, números, hífen, e underline)

## 📦 Resultado

- Variáveis de ambiente disponíveis com os nomes dos segredos em caixa alta
- Os mesmos valores também são exportados como `outputs` e podem ser utilizados em jobs posteriores
