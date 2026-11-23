# Deploy Helm no OpenShift ⛵ `deploy-helm-openshift.yml`

Reusable workflow (`workflow_call`) para **instalar/atualizar/remover** releases Helm em clusters **OpenShift**.

Ele implementa:

- Resolução de `release_name` (DNS-1123, <= 63)
- Login no cluster via `oc-login` (Environment secrets)
- Criação opcional de namespace/projeto
- Deploy Helm com values (arquivos + inline)
- Injeção de imagem (`image.*`) e registry secret (`registrySecret.*`)
- Validação opcional de imagem (manifest check via Docker Registry HTTP API v2)

> Dica: para **chart remoto**, normalmente dá para usar `checkout=false`.

---

## Contrato do `workflow_call`

### Inputs

#### Runner e ambiente
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels do runner. O workflow usa `fromJson(inputs.runs_on)`; portanto **precisa ser JSON válido** (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `environment` | `string` | `no` | `dev` | Nome do **GitHub Environment** aplicado ao job (`environment:`). É daqui que vêm `OC_SERVER` e `OC_TOKEN` (Environment secrets). Valores típicos: `dev`, `ti`, `trg`, `prd`. |
| `timeout` | `number` | `no` | `30` | Timeout do **job** (em minutos) aplicado em `timeout-minutes`. **Não confundir** com `timeout_helm` (timeout do Helm). |
| `checkout` | `boolean` | `no` | `true` | Se `true`, executa `actions/checkout@v4`. Necessário para **chart local** e para `dependency_update=true`. **N/A** quando o chart é remoto e você não precisa de arquivos do repo. |
| `namespace` | `string` | `yes` | `N/A` | Namespace/Project OpenShift alvo. Usado em `oc project` e passado para Helm via `--namespace`. **Obrigatório** (o workflow falha cedo se vazio). |
| `create_namespace` | `boolean` | `no` | `false` | Se `true`, quando o namespace não existir (ou sem acesso), tenta criar via `oc new-project` (fallback `oc create namespace`). Se `false`, falha com mensagem clara. **N/A** se o namespace já existir e o token tiver acesso. |
| `release` | `string` | `no` | `"" (vazio)` | Nome do release (entrada “raw”). Se vazio, usa `GITHUB_REPOSITORY##*/`. Sempre passa por sanitização DNS-1123 (lowercase, apenas `[a-z0-9-]`, máx 63). |

#### Chart e Helm (source + binários)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `chart` | `string` | `yes` | `N/A` | Chart Helm a ser usado. Pode ser **caminho local** (`charts/app`) ou **repo/chart** (`customrepo/app`). É passado direto para o comando Helm. |
| `task` | `string` | `no` | `upgrade` | Ação Helm: `upgrade` (default), `install`, `uninstall\|remove\|delete`. `upgrade` executa `helm upgrade --install` (idempotente). |
| `repo` | `string` | `no` | `"" (vazio)` | URL do repositório Helm. Se não-vazio, roda `helm repo add` + `helm repo update`. **N/A** quando o chart é local ou quando o runner já tem o repo configurado. |
| `repo_alias` | `string` | `no` | `"" (vazio)` | Alias usado no `helm repo add`. Se vazio, o workflow usa `customrepo`. **N/A** quando `repo` está vazio. |
| `repo_username` | `string` | `no` | `"" (vazio)` | Usuário para autenticação no Helm repo. Só é usado quando `repo` está preenchido **e** `secrets.HELM_PASSWORD` existe. **N/A** para repos públicos. |
| `chart_version` | `string` | `no` | `"" (vazio)` | Se preenchido, adiciona `--version <chart_version>`. Útil principalmente para **charts remotos** (fixar versão). Para chart local normalmente é **N/A**. |
| `dependency_update` | `boolean` | `no` | `false` | Se `true`, roda `helm dependency build <chart>` **somente** se existir `<chart>/Chart.yaml` (chart local). Para chart remoto normalmente é **N/A**. |
| `helm_version` | `string` | `no` | `v3.18.4` | Versão do Helm instalada via `azure/setup-helm@v4`. |
| `helm_cmd` | `string` | `no` | `helm` | Nome do binário Helm a executar (default `helm`). O workflow exporta como `HELM_BIN` e usa em todos os comandos (`helm_clean`). |
| `extra_helm_args` | `string` | `no` | `"" (vazio)` | Args adicionais do Helm. Aceita **JSON array** (recomendado) ou string. String é split simples (pode quebrar quotes complexas). **N/A** se vazio. |

#### Values
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `value_files` | `string` | `no` | `"" (vazio)` | Arquivos adicionais de values. Aceita 3 formatos: **JSON array**, **CSV** ou **arquivo único**. Ex.: `['values.yaml','values-dev.yaml']` (JSON), `values.yaml,values-dev.yaml` (CSV) ou `values-dev.yaml`. Gera `--values` para cada item. **N/A** se vazio. |
| `values` | `string` | `no` | `"" (vazio)` | Valores inline (YAML) adicionados via `--values /tmp/inline-values.yaml`. **Não faz envsubst**. Evite segredos aqui (podem aparecer em logs dependendo do chart/Helm). |

#### Flags do Helm
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `wait` | `boolean` | `no` | `true` | Se `true`, adiciona `--wait` (aguarda recursos ficarem prontos). |
| `atomic` | `boolean` | `no` | `true` | Se `true`, adiciona `--atomic` em `install/upgrade` (rollback automático em falha). **N/A** para `uninstall`. |
| `dry_run` | `boolean` | `no` | `false` | Se `true`, adiciona `--dry-run` (simula). Útil para validar templates/values. Não cria recursos. |
| `debug` | `boolean` | `no` | `false` | Se `true`, adiciona `--debug` no Helm (logs mais verbosos). |
| `timeout_helm` | `string` | `no` | `10m` | Timeout do Helm (`--timeout`). Se for apenas número (ex.: `10`), o workflow converte para `10m`. Ex.: `10m`, `1h`. **Não confundir** com `timeout` do job. |

#### Imagem (injeção no chart)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `image_ref` | `string` | `no` | `"" (vazio)` | Imagem completa (ex.: `registry/ns/app@sha256:...` ou `registry/ns/app:tag`). Tem **prioridade máxima**. Se definida, o workflow não tenta reconstruir via repository/tag/digest. |
| `image_repository` | `string` | `no` | `"" (vazio)` | Repositório da imagem (ex.: `registry/ns/app`). Usado para montar `image_ref` com `image_tag` (:`tag`) ou `image_digest` (`@digest`). **N/A** se você passa `image_ref`. |
| `image_tag` | `string` | `no` | `"" (vazio)` | Tag da imagem (ex.: `v1.2.3`, `pr-123-abc1234-rc`). Só usada quando `image_repository` está setado e `image_digest` não está setado. **N/A** quando usa digest. |
| `image_digest` | `string` | `no` | `"" (vazio)` | Digest sha256 da imagem (ex.: `sha256:...`). Quando setado com `image_repository`, gera `image_ref=repo@digest` (imutável). **N/A** quando usa tag. |
| `validate_image` | `boolean` | `no` | `true` | Se `true`, valida a existência do manifest antes do Helm (fail-fast). Faz `curl https://<host>/v2/<repo>/manifests/<tag\|digest>` com Accept de OCI/Docker v2. Requer `image_ref` resolvido; pode falhar em registries com OAuth/token — nesse caso, desative. |

#### Registry Secret / TLS
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `create_registry_secret` | `boolean` | `no` | `false` | Se `true`, injeta values `registrySecret.*` (e opcionalmente `imagePullSecrets[0].name`). **Requer** `registry_server` resolvido e `secrets.NEXUS_USER/NEXUS_PASS` (o workflow falha se faltarem). |
| `registry_secret_server` | `string` | `no` | `"" (vazio)` | Host do registry para o secret/validação. Pode vir com `https://` e/ou path; o workflow extrai apenas o host. Prioridade: `registry_secret_server` > `vars.NEXUS_HOSTED` > host de `image_ref`. **N/A** quando não houver secret/validação e não houver imagem. |
| `enforce_registry_secret_name` | `boolean` | `no` | `true` | Se `true`, garante um nome não-vazio para o secret (usa input ou default `secret-<app>-registry`, sanitizado). Se `false`, deixa em branco para o chart decidir. Importante quando `inject_image_pull_secret=true`. |
| `registry_secret_name` | `string` | `no` | `"" (vazio)` | Nome explícito do Secret. Só é respeitado quando `enforce_registry_secret_name=true`. Se vazio e enforce=true, o workflow gera `secret-<app>-registry`. |
| `inject_image_pull_secret` | `boolean` | `no` | `true` | Se `true` e existir `registry_secret_name`, injeta `--set-string imagePullSecrets[0].name=<secret>`. **N/A** quando `create_registry_secret=false`. |
| `skip_tls_verify` | `boolean` | `no` | `true` | Se `true`, usa `insecure_skip_tls_verify` no `oc-login` e `curl -k` na validação do manifest. Recomendado para clusters/registries com CA interna. |

---

### Secrets
| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `NEXUS_USER` | `condicional` | Credenciais do registry (Basic Auth). Obrigatório quando `create_registry_secret=true`. Pode ser necessário quando `validate_image=true` e o registry é privado. N/A para registry público. |
| `NEXUS_PASS` | `condicional` | Credenciais do registry (Basic Auth). Obrigatório quando `create_registry_secret=true`. Pode ser necessário quando `validate_image=true` e o registry é privado. N/A para registry público. |
| `HELM_PASSWORD` | `condicional` | Senha/token do Helm repository. Usado apenas se `repo` estiver preenchido e `repo_username` estiver preenchido. N/A para repos públicos. |

### Secrets obrigatórios (Environment secrets)
Este workflow **exige** no Environment selecionado (`inputs.environment`):

- `OC_SERVER` — URL da API do OpenShift (**obrigatório**)
- `OC_TOKEN` — token de acesso (**obrigatório**)

> O step **Guard 🛡️** falha cedo se `OC_SERVER` estiver vazio.

### Vars (GitHub Variables)
- `vars.NEXUS_HOSTED` — fallback para resolver host do registry quando `registry_secret_server` não foi informado. (N/A se você sempre passar `registry_secret_server`.)

---

### Outputs
| Output | Descrição (técnica) |
|---|---|
| `release_name` | Nome final do release após sanitização DNS-1123. Se `inputs.release` for vazio, usa o nome do repositório. Sempre <= 63 chars. |
| `app_name` | Alias do app (igual a `release_name`). Usado no summary e para defaults (ex.: `secret-<app>-registry`). |
| `namespace` | Namespace/Project efetivamente usado (igual ao input `namespace`, após validação). |
| `image_ref` | Imagem final resolvida. Pode ser vazio se nenhum input de imagem foi fornecido. Prioridade: `image_ref` > `image_repository+image_digest` > `image_repository+image_tag`. |
| `image_tag` | Tag reportada (vem de `inputs.image_tag`). Se você fornecer `image_ref` diretamente, o workflow **não parseia** a tag, então este output pode ficar vazio. |
| `registry_secret_name` | Nome final do secret **somente** quando `enforce_registry_secret_name=true`; caso contrário fica vazio (chart decide). |
| `registry_server` | Host do registry resolvido (sem scheme/path). Prioridade: `registry_secret_server` > `vars.NEXUS_HOSTED` > host de `image_ref`. Pode ficar vazio se nada foi informado. |

---

## Defaults / permissions / concurrency

### Defaults
```yaml
run:
  shell: bash
```

### Permissions
```yaml
contents: read
```

### Concurrency
N/A (não define `concurrency:`; controle no caller se necessário).

---

## Passo a passo (o que cada step faz)

| # | Step | Condição (`if`) |
|---:|---|---|
| 1 | Checkout 📥 | `${{ inputs.checkout }}` |
| 2 | Resolve Context 🧠 | `N/A` |
| 3 | Guard 🛡️ | `N/A` |
| 4 | Login no OpenShift ☁️ | `N/A` |
| 5 | Ensure Namespace/Project 📂 | `N/A` |
| 6 | Setup Helm ⛵ | `N/A` |
| 7 | Configurar Helm Repo 📦 | `${{ inputs.repo != '' }}` |
| 8 | Helm Dependency Build 📦 | `${{ inputs.dependency_update }}` |
| 9 | Helm Deploy 🚀 | `N/A` |
| 10 | Summary 📜 | `${{ success() && steps.deploy.conclusion == 'success' }}` |

---

## Coerência (validações que o workflow realmente faz)

- ✅ Falha cedo se `namespace` vier vazio (no step **Resolve Context**).
- ✅ Falha cedo se `secrets.OC_SERVER` vier vazio (no step **Guard**).
- ✅ Se o namespace não existir (ou sem acesso):
  - `create_namespace=false` → **falha**
  - `create_namespace=true` → tenta `oc new-project` (fallback `oc create namespace`) e depois `oc project`.
- ✅ `validate_image=true`:
  - exige imagem resolvida (senão falha)
  - valida manifest via `/v2/<repo>/manifests/<tag|digest>`
  - usa `-k` quando `skip_tls_verify=true`
  - usa Basic Auth quando `NEXUS_USER/NEXUS_PASS` estiverem presentes
- ✅ `create_registry_secret=true`:
  - exige `registry_server` resolvido
  - exige `NEXUS_USER` e `NEXUS_PASS` (senão falha)
  - injeta `registrySecret.*` e (opcional) `imagePullSecrets[0].name` quando habilitado.
- ✅ `task` aceita somente `install`, `upgrade`, `uninstall|remove|delete` (caso contrário falha).

---

## Matriz de cenários (inputs → comportamento)

### 1) Fonte do chart × steps necessários

| Cenário | `chart` | `repo` | `checkout` | `helm repo add` | `dependency_update` | Observações |
|---|---|---|---|---|---|---|
| Chart local (no repo) | `charts/myapp` | `""` | `true` | N/A | `true/false` | Requer checkout. `dependency_update` só faz sentido aqui. |
| Chart remoto (repo Helm) | `customrepo/mychart` | `https://...` | `false` (ou `true`) | `true` | `false` (recomendado) | Use `chart_version` para fixar versão. |
| Chart remoto sem `repo add` | `stable/mychart` | `""` | `false` (ou `true`) | N/A | N/A | Só funciona se o repo já existir no runner. |

### 2) `task` × comando Helm

| `task` | Comando executado | Uso típico |
|---|---|---|
| `upgrade` (default) | `helm upgrade <release> <chart> --install` | Idempotente (recomendado). |
| `install` | `helm install <release> <chart>` | Primeira instalação. |
| `uninstall` / `remove` / `delete` | `helm uninstall <release>` | Remover release. |

### 3) Registry secret × requisitos

| `create_registry_secret` | Precisa `NEXUS_USER/PASS` | Precisa `registry_server` | Injeta `imagePullSecrets[0].name` |
|---:|:---:|:---:|:---:|
| `false` | N/A | N/A | N/A |
| `true` | ✅ | ✅ (auto ou input) | ✅ quando `inject_image_pull_secret=true` **e** `enforce_registry_secret_name=true` |

### 4) Validação da imagem (manifest check)

| `validate_image` | Exige imagem resolvida | Pode exigir auth | Observação |
|---:|:---:|:---:|---|
| `false` | N/A | N/A | Não valida manifest. |
| `true` | ✅ | depende | Pode ser necessário desabilitar em registries com token/OAuth. |

---

## FAQ de cenários (técnico)

### “Deploy OK”, mas pod ficou `ImagePullBackOff`
Causas típicas:
- imagem/tag não existe (ou digest incorreto)
- registry privado sem secret/`imagePullSecrets`
- chart ignora `image.*` (usa chaves diferentes)

Ações:
- mantenha `validate_image=true` quando o registry suportar Docker Registry API v2
- use `create_registry_secret=true` + `inject_image_pull_secret=true`
- valide no chart quais chaves de imagem ele consome.

### “create_registry_secret=true” e nada aconteceu
O workflow **só injeta values**. Se o chart não tiver templates para `registrySecret.*` / `imagePullSecrets`, nada será criado.
➡️ Ajuste o chart para suportar essas chaves.

### Falha no login do OpenShift
Confirme `OC_SERVER` e `OC_TOKEN` no **Environment** que você passou em `inputs.environment`. Em clusters com CA interna, mantenha `skip_tls_verify=true`.

---

## Exemplos de caller

### A) Chart local + upgrade

```yaml
jobs:
  deploy:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/deploy-helm-openshift.yml@<ref>
    secrets: inherit
    with:
      environment: dev
      namespace: my-namespace
      chart: charts/myapp
      value_files: '["values.yaml","values-dev.yaml"]'
      task: upgrade
      image_ref: "registry.local/ns/app:sha-abcdef1"
```

### B) Chart remoto (repo Helm) + versão fixa

```yaml
jobs:
  deploy:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/deploy-helm-openshift.yml@<ref>
    secrets: inherit
    with:
      environment: ti
      namespace: my-namespace
      repo: "https://nexus.example.local/repository/helm-hosted"
      repo_alias: "customrepo"
      chart: "customrepo/sonarqube"
      chart_version: "2026.1.0"
      task: upgrade
```

### C) Criar/injetar registry secret (Nexus)

```yaml
jobs:
  deploy:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/deploy-helm-openshift.yml@<ref>
    secrets: inherit
    with:
      environment: trg
      namespace: my-namespace
      chart: charts/myapp
      task: upgrade

      image_repository: "hosted-docker-registry.../docker-hosted/releaseflow/myapp"
      image_tag: "sha-abcdef1"

      create_registry_secret: true
      inject_image_pull_secret: true
      enforce_registry_secret_name: true
```
