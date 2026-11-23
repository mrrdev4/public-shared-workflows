# Build & Push ♻️ `build-docker.yml`

Reusable workflow para **buildar imagem OCI** com **Buildah** e fazer **push** para:
- **Nexus** (registry interno)
- **GCP Artifact Registry (GAR)** via **Workload Identity Federation (WIF)**

Ele resolve automaticamente:
- nome da app (`<repo name>`)
- paths de registry (host + prefix + repo + app)
- estratégia de tags (push vs PR, e branches feature/bugfix/hotfix)
- outputs com URIs e digests (incluindo `@sha256:...`)

---

## Contrato do `workflow_call`

### Inputs

#### Execução e build
| Input | Tipo | Required | Default | Descrição |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels para `runs-on` (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). Precisa ser JSON válido porque o workflow usa `fromJson(inputs.runs_on)`. |
| `dockerfile` | `string` | `no` | `Dockerfile` | Caminho do Dockerfile/Containerfile relativo ao repo (usado pelo `redhat-actions/buildah-build`). Ex.: `Dockerfile`, `build/Dockerfile`. |
| `context` | `string` | `no` | `.` | Diretório de contexto do build (enviado ao Buildah). Geralmente `.`; ajuste quando o Dockerfile está fora da raiz. |
| `build_args` | `string` | `no` | `` | Argumentos de build do Buildah (`--build-arg`). Formato: `KEY=VALUE`, um por linha. Use somente para *build-time*; não é indicado para secrets. |
#### Destinos e registry (Nexus)
| Input | Tipo | Required | Default | Descrição |
|---|---|:---:|---|---|
| `push_to_nexus` | `boolean` | `no` | `true` | Quando `true`, faz push para o Nexus e define o **Nexus como destino primário** (outputs `image_uri`, `digest`, `image_by_digest` viram do Nexus). Requer `nexus_hosted` (ou `vars.NEXUS_HOSTED`) e `secrets.NEXUS_USER/NEXUS_PASS`. |
| `registry_repo` | `string` | `no` | `${{ github.event.repository.name }}` | Segmento **(Nexus)** usado na composição do path final. Ordem: `prefix` (do `nexus_hosted` quando tem path) + `registry_repo` + `<app>`. Se vazio, tenta `vars.NEXUS_REPO`. Se **não houver prefix e nem repo**, e `push_to_nexus=true`, o workflow falha. |
| `nexus_hosted` | `string` | `no` | `` | **(Nexus)** Host do registry. Aceita: - só host: `registry.exemplo.com` - host + path: `registry.exemplo.com/docker-hosted/releaseflow` Também aceita com `http(s)://` (o scheme é removido). Se vazio, usa `vars.NEXUS_HOSTED`. |
| `insecure_registry` | `boolean` | `no` | `true` | **(Nexus)** Controla `tls-verify` no push para o Nexus. `true` (default) desliga verificação TLS (`tls-verify=false`). Use quando o Nexus tem CA interna/self-signed. `false` liga verificação TLS. |
#### Destinos e registry (GCP Artifact Registry)
| Input | Tipo | Required | Default | Descrição |
|---|---|:---:|---|---|
| `push_to_gcp` | `boolean` | `no` | `false` | Quando `true`, faz push para o GCP Artifact Registry (GAR) via WIF. Requer `gcp_project_id`, `gcp_registry_repo`, `gcp_workload_identity_provider`, `gcp_service_account` (ou as `vars.*`). Se `push_to_nexus=false`, o GAR vira o destino primário. |
| `gcp_registry_host` | `string` | `no` | `` | **(GAR)** Host do Artifact Registry (sem project/repo). Ex.: `southamerica-east1-docker.pkg.dev`. Se vazio, usa `vars.GCP_REGISTRY_HOST`; se ainda vazio, aplica fallback `southamerica-east1-docker.pkg.dev`. **N/A** quando `push_to_gcp=false`. |
| `gcp_project_id` | `string` | `no` | `` | **(GAR)** Project ID usado para montar o path `project/repo/app`. Pode vir de `vars.GCP_PROJECT_ID`. **Obrigatório** quando `push_to_gcp=true`; **N/A** quando `push_to_gcp=false`. |
| `gcp_registry_repo` | `string` | `no` | `` | **(GAR)** Nome do repositório do Artifact Registry. Pode vir de `vars.GCP_REGISTRY_REPO`. **Obrigatório** quando `push_to_gcp=true`; **N/A** quando `push_to_gcp=false`. |
| `gcp_workload_identity_provider` | `string` | `no` | `` | **(GAR/WIF)** Resource do Workload Identity Provider (`projects/.../providers/...`). Usado pelo `google-github-actions/auth@v2`. Pode vir de `vars.GCP_WIF_PROVIDER`. **Obrigatório** quando `push_to_gcp=true`; **N/A** quando `push_to_gcp=false`. |
| `gcp_service_account` | `string` | `no` | `` | **(GAR/WIF)** Service Account (email) a ser impersonada via WIF. Pode vir de `vars.GCP_SERVICE_ACCOUNT`. **Obrigatório** quando `push_to_gcp=true`; **N/A** quando `push_to_gcp=false`. |
#### Tags
| Input | Tipo | Required | Default | Descrição |
|---|---|:---:|---|---|
| `push_sha_tag` | `boolean` | `no` | `false` | Quando `true`, além da tag principal, também publica `sha-<GITHUB_SHA>` (tag imutável por commit). Útil para rastreabilidade e para rotinas que resolvem digest por SHA. |
---

### Secrets
| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `NEXUS_USER` | `condicional` | Usuário do Nexus. **Obrigatório em runtime** quando `push_to_nexus=true` (o script valida e falha se vazio). **N/A** quando `push_to_nexus=false`. |
| `NEXUS_PASS` | `condicional` | Senha/token do Nexus. **Obrigatório em runtime** quando `push_to_nexus=true`. **N/A** quando `push_to_nexus=false`. |

> Nota: apesar de `workflow_call.secrets` marcar como `required: false`, o step **Resolve Registries** valida e exige `NEXUS_USER/NEXUS_PASS` quando `push_to_nexus=true`.

---

### Outputs
| Output | Descrição (técnica) |
|---|---|
| `tag` | Tag **principal** calculada. Regras: - PR em `feature\|bugfix\|hotfix/*` → `pr-<num>-<sha7>-rc` - PR em outras branches → `pr-<num>-<dd-mm-yyyy>-<sha7>` - Push em `feature\|bugfix\|hotfix/*` → `<sha7>` - Push em outras branches → `<dd-mm-yyyy>-<sha7>-<branch-normalizada>` |
| `tag_sha` | Sempre calculada como `sha-<fullsha>`. **Só é publicada** (push) quando `push_sha_tag=true`. Caso contrário, fica como referência, mas não existe no registry. |
| `image_uri` | URI do **destino primário** (sem tag), no formato `<host>/<path>`. Primário = Nexus quando `push_to_nexus=true`, senão GAR. Combine com `:${tag}` para referenciar por tag. |
| `image_path` | Path do destino primário **sem host** (ex.: `docker-hosted/releaseflow/app`). Útil para steps que exigem `registry` + `image` separados. |
| `digest` | Digest do destino primário (header/manifest digest retornado pelo push). Primário = Nexus quando `push_to_nexus=true`, senão GAR. |
| `image_by_digest` | Referência imutável do destino primário: `<image_uri>@<digest>`. Vazio apenas se o push falhar e não houver digest. |
| `image_uri_nexus` | URI completa no Nexus `<host>/<path>`. **N/A** quando `push_to_nexus=false` (ou quando `vars/inputs` de Nexus não foram definidos). |
| `image_uri_gcp` | URI completa no GAR `<host>/<project>/<repo>/<app>`. **N/A** quando `push_to_gcp=false`. |
| `digest_nexus` | Digest retornado pelo push no Nexus. **N/A** quando `push_to_nexus=false`. |
| `digest_gcp` | Digest retornado pelo push no GAR. **N/A** quando `push_to_gcp=false`. |
| `image_by_digest_nexus` | Nexus por digest: `<image_uri_nexus>@<digest_nexus>`. **N/A** quando `push_to_nexus=false`. |
| `image_by_digest_gcp` | GAR por digest: `<image_uri_gcp>@<digest_gcp>`. **N/A** quando `push_to_gcp=false`. |

---

## Coerência (validações que o workflow realmente faz)

- ✅ Exige ao menos um destino: se `push_to_nexus=false` **e** `push_to_gcp=false` → **falha** no step **Resolve Registries**.
- ✅ Quando `push_to_nexus=true`:
  - exige `nexus_hosted` (ou `vars.NEXUS_HOSTED`)
  - exige `secrets.NEXUS_USER` e `secrets.NEXUS_PASS`
  - exige que exista **prefix/path** (via `nexus_hosted` com path) **ou** `registry_repo` (via input/`vars.NEXUS_REPO`), senão falha.
- ✅ Quando `push_to_gcp=true`:
  - exige `gcp_project_id`, `gcp_registry_repo`, `gcp_workload_identity_provider`, `gcp_service_account` (input ou `vars.*`)
  - autentica por WIF (`google-github-actions/auth@v2`) e usa `oauth2accesstoken` no push.
- ✅ Define destino **primário**:
  - Nexus quando `push_to_nexus=true`
  - senão GAR
  Isso impacta diretamente `image_uri`, `digest` e `image_by_digest`.

---


## Defaults / permissions / concurrency

### Defaults
```yaml
run:
  shell: bash
```

### Permissions
N/A (não define `permissions:`; herda o padrão do repositório/caller).

### Concurrency
N/A (não define `concurrency:`; se precisar, controle no workflow caller).

---

## Passo a passo (o que cada step faz)

| # | Step | Condição (`if`) |
|---:|---|---|
| 1 | Checkout 📥 | `N/A` |
| 2 | Resolve Registries 📌 | `N/A` |
| 3 | Compute Tags 🏷️ | `N/A` |
| 4 | Build Image 🛠️ | `N/A` |
| 5 | Retag For Other Registries 🔖 | `${{ inputs.push_to_nexus && inputs.push_to_gcp }}` |
| 6 | Push to Nexus ♻️ | `${{ inputs.push_to_nexus }}` |
| 7 | Auth to Google Cloud | `${{ inputs.push_to_gcp }}` |
| 8 | Push to GCP Artifact Registry ♻️ | `${{ inputs.push_to_gcp }}` |
| 9 | Set Outputs 📝 | `N/A` |
| 10 | Summary 📜 | `${{ always() }}` |

---

## Matriz de comportamento

### A) Destinos (Nexus × GAR)
| `push_to_nexus` | `push_to_gcp` | Build (image base) | Push | Step extra |
|---:|---:|---|---|---|
| ✅ | ❌ | Nexus (primário) | Só Nexus | — |
| ❌ | ✅ | GAR (primário) | Só GAR | — |
| ✅ | ✅ | Nexus (primário) | Nexus + GAR | `Retag For Other Registries` para gerar tags no segundo destino |
| ❌ | ❌ | — | — | ❌ workflow falha (precisa ao menos um destino) |


### B) Tags extras
| `push_sha_tag` | O que muda |
|---:|---|
| `false` | Publica somente `tag` |
| `true` | Publica `tag` **e** `tag_sha` (`sha-<fullsha>`) |

---

## FAQ (cenários comuns)

### 1) “Por que o workflow falhou logo no começo?”
O step **Resolve Registries** falha se `push_to_nexus=false` e `push_to_gcp=false`.

### 2) “Meu Nexus é `host/docker-hosted/releaseflow` — como preencher?”
Você tem 2 opções equivalentes:
✅ Solução:

- `nexus_hosted: "host/docker-hosted/releaseflow"` e `registry_repo: ""`
- `nexus_hosted: "host/docker-hosted"` e `registry_repo: "releaseflow"`

> O workflow monta `image_path_nexus` como:
> - `prefix/repo/app`, ou `prefix/app`, ou `repo/app`.
> Se não houver nem prefix nem repo e você estiver com `push_to_nexus=true`, ele falha.

### 3) “Como sai a tag em PR?”
- PR de `feature/**|bugfix/**|hotfix/**` → `pr-<num>-<sha7>-rc`
- PR de outras branches → `pr-<num>-<data>-<sha7>`

### 4) “Quero também uma tag `sha-<fullsha>`”
Ative `push_sha_tag: true`. Isso adiciona uma segunda tag no push.

### 5) “Preciso de docker login no GAR?”
Não. O workflow usa **Workload Identity Federation** via `google-github-actions/auth@v2` e empurra com `push-to-registry` usando `oauth2accesstoken`.

### 6) “Falha TLS no Nexus (cert interno)”
✅ Solução:
- Use `insecure_registry: true` para desligar `tls-verify` no push para Nexus.

---

## Exemplos de uso (caller)

### A) Push somente para Nexus

```yaml
jobs:
  build:
    uses: mrrdev4/shared-workflows/.github/workflows/build-docker.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      dockerfile: "Dockerfile"
      context: "."
      build_args: ""
      push_to_nexus: true
      push_to_gcp: false
      # Opção 1: host + path completo
      nexus_hosted: "hosted-docker-registry-.../docker-hosted/releaseflow"
      registry_repo: ""
      insecure_registry: false
      push_sha_tag: true
```

### B) Push somente para GCP Artifact Registry (WIF)

```yaml
jobs:
  build:
    uses: mrrdev4/shared-workflows/.github/workflows/build-docker.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      dockerfile: "Dockerfile"
      context: "."
      push_to_nexus: false
      push_to_gcp: true

      gcp_registry_host: "southamerica-east1-docker.pkg.dev"
      gcp_project_id: "meu-projeto"
      gcp_registry_repo: "meu-repo"
      gcp_workload_identity_provider: "projects/....../providers/...."
      gcp_service_account: "gha-wif@meu-projeto.iam.gserviceaccount.com"
```

### C) Push para Nexus **e** GAR

```yaml
jobs:
  build:
    uses: mrrdev4/shared-workflows/.github/workflows/build-docker.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      dockerfile: "Dockerfile"
      context: "."
      push_to_nexus: true
      push_to_gcp: true

      nexus_hosted: "hosted-docker-registry-.../docker-hosted/releaseflow"
      registry_repo: ""

      gcp_project_id: "meu-projeto"
      gcp_registry_repo: "meu-repo"
      gcp_workload_identity_provider: "projects/....../providers/...."
      gcp_service_account: "gha-wif@meu-projeto.iam.gserviceaccount.com"
```

> Dica: se o build for condicionado pelo ReleaseFlow Guard, use `if: needs.guard.outputs.should_run == 'true'`.
