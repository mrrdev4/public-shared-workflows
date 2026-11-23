# Release Flow Orchestrator ♻️ `releaseflow.yml`

Workflow reusable (`workflow_call`) que atua como **orquestrador** do ReleaseFlow: ele **detecta o contexto** (evento/branch) e executa os jobs necessários para:

- **Dev Flow (PR):** build + push de imagem para testes/preview
- **Promotion (TI/TRG):** resolver **digest** + deploy imutável em TI/TRG + abrir PR do Release-Please
- **Publish (PRD):** detectar merge do Release-Please, criar **tag/release**, retagar a imagem e deployar PRD + abrir PR de sync `release → main`

---

## Visão geral (mais legível)

### Fluxos principais
| Fluxo | Gatilho (evento/branch) | Flags do `ctx` | Jobs executados (ordem) | Artefato principal |
|---|---|---|---|---|
| **Dev Flow (PR)** | `pull_request` aberto para `feature/*`, `bugfix/*`, `hotfix/*` (base: `main`, `release/**` ou `feature/*`) | `is_dev_flow=true` | `build` | Imagem com tags de PR/SHA (mutável, para preview/testes) |
| **Promotion (TI/TRG)** | `create` de `release/**` **ou** `push` de criação de `release/**` **ou** merge de `bugfix/*`/`hotfix/*` em `release/**` | `should_run_release_promotion=true` e `should_run_release_please_pr=true` | `resolve_digest` → `deploy_ti` → `deploy_trg` → `release_please_pr` | Imagem **por digest** (`image@sha256:...`, imutável) |
| **Publish (PRD)** | `push` em `release/**` detectado como merge de PR `release-please--*` | `should_run_release_please_publish=true` | `resolve_digest` → `release_please_publish` → `retag_release_image` → `deploy_prd` → `sync_release_to_main` | Imagem **tag SemVer** (`image:vX.Y.Z`) + GitHub Release/Tag |


### Matriz (evento × tipo de branch → flags)
| Cenário | `event` | `head_kind` | `base_branch` | `is_dev_flow` | `should_run_release_promotion` | `should_run_release_please_publish` | Observação |
|---|---|---|---|:---:|:---:|:---:|---|
| PR feature → main (aberta) | `pull_request` | `feature` | `main` | ✅ | ❌ | ❌ | Roda `build`. |
| PR bugfix → release/** (aberta) | `pull_request` | `bugfix` | `release/**` | ✅ | ❌ | ❌ | Roda `build`. |
| Merge PR bugfix/hotfix → release/** | `pull_request` (closed+merged) | `bugfix`/`hotfix` | `release/**` | ❌ | ✅ | ❌ | Promoção TI/TRG + RP PR (não é dev flow). |
| Criação release/** via `create` | `create` | `release` | N/A | ❌ | ✅ | ❌ | Promoção TI/TRG + RP PR. |
| Criação release/** via `push` (created/before=0000) | `push` | `release` | N/A | ❌ | ✅ | ❌ | Fallback quando o caller não tem `on.create`. |
| Push em release/** vindo de merge do Release-Please | `push` | `release` | N/A | ❌ | ❌ | ✅ | Publish + retag + PRD + sync. |
| PR release/** → main (sync) | `pull_request` | `release` | `main` | ❌ | ❌ | ❌ | Não roda build (por design). |
| PR release-please--* → release/** | `pull_request` | `bot` | `release/**` | ❌ | ❌ | ❌ | Branch bot não entra em dev/promo. |


> Nota importante: o job **`ctx`** é o “cérebro” (define `is_dev_flow`, `should_run_release_promotion`, `should_run_release_please_publish`, etc.).
> Os jobs seguintes executam **somente** quando o `if` permite.

---

## Contrato do `workflow_call`

### Inputs

#### Execução e roteamento
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels do runner. O workflow usa `fromJson(inputs.runs_on)` nos jobs `uses:` (build/deploy), então precisa ser **JSON válido**. Ex.: `["self-hosted","dev"]` ou `["ubuntu-latest"]`. |
| `trigger_event_name` | `string` | `no` | `"" (vazio)` | Override opcional do nome do evento. Se vazio, o `ctx` usa `GITHUB_EVENT_NAME`. Normalmente o caller passa `${{ github.event_name }}` para consistência. |
| `main_branch` | `string` | `no` | `main` | Nome da branch principal usada como referência no `ctx` (default `main`). Impacta `is_dev_flow` e a criação do PR de sync (base). |
| `release_prefix` | `string` | `no` | `release/` | Prefixo da branch de release (default `release/`). O `ctx` usa este prefixo para classificar branches (`head_kind=release`) e decidir promoção/publish. |
#### Build (imagem)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `dockerfile` | `string` | `no` | `Dockerfile` | Caminho do Dockerfile para o build (encaminhado para `build-docker.yml`). N/A se `build` não rodar. |
| `context` | `string` | `no` | `.` | Contexto do build Docker (encaminhado para `build-docker.yml`). N/A se `build` não rodar. |
| `registry_repo` | `string` | `no` | `"" (vazio)` | Nome do repositório/pasta de destino no registry para a imagem. Se vazio, o job `build` usa o nome do repo (`github.event.repository.name`). Também é usado para montar o path no Nexus em `resolve_digest`/`retag`. |
| `nexus_hosted` | `string` | `no` | `"" (vazio)` | Host do Nexus/registry (pode conter path e/ou scheme). Ex.: `host/repo/path`, `https://host/repo`. Usado por `build`, `resolve_digest` e `retag`. Se vazio, `resolve_digest` e `retag` tentam `vars.NEXUS_HOSTED`. |
| `insecure_registry` | `boolean` | `no` | `true` | Se `true`, permite TLS inseguro no fluxo de registry (propaga para `build-docker` e controla `curl -k` em `resolve_digest` e `--tls-verify=false` no skopeo em `retag`). |
#### Deploy (Helm/OpenShift)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `enable_deploy_ti` | `string` | `no` | `true` | Flag **string** (`"true"\|"false"`). Se `"true"`, habilita `deploy_ti` quando houver promoção (`should_run_release_promotion=true`). Obs.: `deploy_trg` depende do resultado de TI. |
| `enable_deploy_trg` | `string` | `no` | `true` | Flag **string**. Se `"true"`, habilita `deploy_trg` quando houver promoção e `deploy_ti` tiver sucesso. Se TI for desabilitado, TRG será **skipped** por dependência. |
| `enable_deploy_prd` | `string` | `no` | `true` | Flag **string**. Se `"true"`, habilita `deploy_prd` no publish path. Se `false`, também impede `sync_release_to_main` (o `if` exige `deploy_prd == success`). |
| `namespace_ti` | `string` | `no` | `ti` | Namespace/Project OpenShift usado no deploy de TI (`deploy-helm-openshift.yml`). N/A se `deploy_ti` não rodar. |
| `namespace_trg` | `string` | `no` | `trg` | Namespace/Project OpenShift usado no deploy de TRG. N/A se `deploy_trg` não rodar. |
| `namespace_prd` | `string` | `no` | `prod` | Namespace/Project OpenShift usado no deploy de PRD. N/A se `deploy_prd` não rodar. |
| `chart` | `string` | `no` | `./chart` | Chart Helm a ser aplicado (caminho local ou `repo/chart`). Encaminhado para todos os deploys (TI/TRG/PRD). |
| `repo` | `string` | `no` | `"" (vazio)` | URL do Helm repository (opcional). Encaminhado para `deploy-helm-openshift.yml`. N/A quando chart é local. |
| `repo_alias` | `string` | `no` | `"" (vazio)` | Alias do Helm repo (`helm repo add <alias> <url>`). N/A quando `repo` é vazio. |
| `repo_username` | `string` | `no` | `"" (vazio)` | Usuário do Helm repo (só tem efeito se `repo` não for vazio e o secret `HELM_PASSWORD` estiver disponível no deploy workflow). N/A para repos públicos. |
| `chart_version` | `string` | `no` | `"" (vazio)` | Versão do chart (`--version`). Recomendado para charts remotos. N/A para chart local. |
| `values` | `string` | `no` | `"" (vazio)` | Values inline (YAML) repassados para o deploy. Observação: não há envsubst; evite segredos aqui. |
| `value_files` | `string` | `no` | `"" (vazio)` | Values files “base” (aplicados quando o env específico estiver vazio). Aceita JSON array / CSV / arquivo único (interpretado pelo workflow de deploy). |
| `value_files_ti` | `string` | `no` | `"" (vazio)` | Values files específicos de TI. Se vazio, cai para `value_files`. |
| `value_files_trg` | `string` | `no` | `"" (vazio)` | Values files específicos de TRG. Se vazio, cai para `value_files`. |
| `value_files_prd` | `string` | `no` | `"" (vazio)` | Values files específicos de PRD. Se vazio, cai para `value_files`. |
| `validate_image` | `boolean` | `no` | `true` | Se `true`, o deploy valida a existência do manifest da imagem antes do Helm (fail-fast). Encaminhado para o workflow de deploy. Pode ser desabilitado em registries que exigem token/OAuth. |
| `skip_tls_verify` | `boolean` | `no` | `true` | Se `true`, usa `insecure_skip_tls_verify` no login do OpenShift e `curl -k` na validação do manifest (quando aplicável). |
| `timeout_helm` | `string` | `no` | `10m` | Timeout do Helm (`--timeout`) repassado ao deploy. Default `10m`. |
| `atomic` | `boolean` | `no` | `true` | Se `true`, `helm install/upgrade` roda com `--atomic` (rollback automático). Encaminhado ao deploy. |
#### Release Please
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `release_please_config` | `string` | `no` | `release-please-config.json` | Caminho do arquivo `release-please-config.json` no repo. Usado nos jobs `release_please_pr` e `release_please_publish`. |
| `release_please_manifest` | `string` | `no` | `.release-please-manifest.json` | Caminho do arquivo `.release-please-manifest.json` no repo. Usado nos jobs `release_please_pr` e `release_please_publish`. |

### Secrets
> O `workflow_call` não declara secrets, mas o workflow **referencia** os secrets abaixo. Portanto o caller deve passar/`inherit` corretamente.

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `NEXUS_PASS` | `yes` | Senha/token do Nexus/registry (Basic Auth). Mesmo uso do `NEXUS_USER`. |
| `NEXUS_USER` | `yes` | Usuário do Nexus/registry (Basic Auth). Necessário para `build` (push), `resolve_digest` (GET manifests) e `retag_release_image` (skopeo copy). N/A se você desabilitar todos esses caminhos (raro). |
| `RELEASE_PLEASE_TOKEN` | `condicional` | Token (PAT) opcional. Se vazio, `release_please_*` usa `github.token`. Recomendado quando rulesets exigem permissões elevadas para criar PRs/releases/tags ou quando `GITHUB_TOKEN` é restrito. |

### Vars (GitHub Variables)
| Variable | Obrigatória | Descrição (técnica) |
|---|:---:|---|
| `NEXUS_HOSTED` | `condicional` | Fallback para `inputs.nexus_hosted` em `resolve_digest` e `retag_release_image`. Se você sempre passa `inputs.nexus_hosted`, esta var é **N/A**. |

### Outputs
| Output | Descrição (técnica) |
|---|---|
| `promo_ref` | Ref (branch) promovida (ex.: release/2026-...) (vem do `ctx`: branch `release/**` alvo da promoção; vazio quando não é promoção). |
| `image_by_digest` | Imagem imutável resolvida para promoção (registry/repo/app@sha256:...) (imutável; usado para TI/TRG). |
| `digest` | Digest sha256 resolvido para a imagem promovida |
| `used_tag` | Tag (pr-.../sha-...) usada para resolver o digest |
| `pr_number` | PR associada ao commit (best-effort) usada na resolução do digest |
| `release_please_pr_url` | URL da PR criada/atualizada pelo release-please |
| `release_please_pr_number` | Número da PR do release-please |
| `tag_name` | Tag SemVer criada no GitHub pelo release-please publish |
| `image_ref` | Imagem final retaggeada para a tag SemVer (registry/repo/app:vX.Y.Z) (tag SemVer; usado no PRD). |
| `sync_pr_url` | URL da PR de sync release -> main |
| `sync_pr_number` | Número da PR de sync release -> main |

---

## Jobs e condições (alto nível)

| Ordem | Job | Tipo | Needs | Condição (`if`) | O que faz (técnico) |
|---:|---|---|---|---|---|
| 1 | `ctx` | run | N/A | `N/A` | Calcula contexto (`event_name`, `head/base`, `head_kind`, flags de decisão) e outputs de roteamento usados pelos demais jobs. |
| 2 | `build` | reusable (`uses`) | ctx | `${{ needs.ctx.outputs.event_name == 'pull_request' && needs.ctx.outputs.pr_action != 'closed' && needs.ctx.outputs.is_dev_flow == 'true' && needs.ctx.result == 'success' }}` | Chama `build-docker.yml` para build & push da imagem (tags PR/SHA) no Nexus/registry. |
| 3 | `resolve_digest` | run | ctx | `${{ needs.ctx.outputs.should_run_release_promotion == 'true' || needs.ctx.outputs.should_run_release_please_publish == 'true' }}` | Resolve `docker-content-digest` no Nexus via Docker Registry HTTP API v2, testando tags candidatas (PR/SHA) até encontrar um digest. |
| 4 | `deploy_ti` | reusable (`uses`) | ctx, resolve_digest | `${{ inputs.enable_deploy_ti == 'true' && needs.ctx.outputs.should_run_release_promotion == 'true' && needs.resolve_digest.result == 'success' && needs.resolve_digest.outputs.image_by_digest != '' }}` | Chama `deploy-helm-openshift.yml` para deploy em **TI** usando `image_by_digest` (imutável). |
| 5 | `deploy_trg` | reusable (`uses`) | ctx, resolve_digest, deploy_ti | `${{ inputs.enable_deploy_trg == 'true' && needs.ctx.outputs.should_run_release_promotion == 'true' && needs.deploy_ti.result == 'success' }}` | Chama `deploy-helm-openshift.yml` para deploy em **TRG** usando `image_by_digest` (imutável). |
| 6 | `release_please_pr` | run | ctx, deploy_trg | `${{ needs.deploy_trg.result == 'success' && needs.ctx.outputs.should_run_release_please_pr == 'true' }}` | Executa `googleapis/release-please-action@v4` em modo PR (skip release) apontando para a `promo_ref` (release/**). |
| 7 | `release_please_publish` | run | ctx, resolve_digest | `${{ needs.ctx.outputs.should_run_release_please_publish == 'true' }}` | Executa `release-please-action@v4` em modo publish (skip PR) após detectar merge do release-please na release/**; produz `tag_name`. |
| 8 | `retag_release_image` | run | ctx, resolve_digest, release_please_publish | `${{ needs.ctx.outputs.should_run_release_please_publish == 'true' && needs.release_please_publish.outputs.releases_created == 'true' && needs.release_please_publish.outputs.tag_name != '' && needs.resolve_digest.outputs.digest != '' }}` | Retaga/copia a imagem por digest para a tag SemVer via `skopeo copy` (imutável -> `:vX.Y.Z`). |
| 9 | `deploy_prd` | reusable (`uses`) | ctx, retag_release_image | `${{ inputs.enable_deploy_prd == 'true' && needs.ctx.outputs.should_run_release_please_publish == 'true' && needs.retag_release_image.result == 'success' && needs.retag_release_image.outputs.image_ref != '' }}` | Chama `deploy-helm-openshift.yml` para deploy em **PRD** usando a tag SemVer (`image_ref`). |
| 10 | `sync_release_to_main` | run | ctx, release_please_publish, deploy_prd | `${{ needs.release_please_publish.result == 'success' && needs.deploy_prd.result == 'success' && needs.release_please_publish.outputs.releases_created == 'true' && needs.release_please_publish.outputs.tag_name != '' }}` | Abre (best-effort) PR `release/** -> main` com `actions/github-script@v7`. Se já existir PR aberta, reutiliza; se não houver diff (422), só avisa. |
| 11 | `summary` | run | ctx, build, resolve_digest, deploy_ti, deploy_trg, deploy_prd, release_please_pr, release_please_publish, retag_release_image, sync_release_to_main | `${{ always() }}` | Escreve `GITHUB_STEP_SUMMARY` consolidando flags, resultados e links para PRs/tags. |

---

## Passo a passo por job (com condições e steps)

### `ctx` — Determine Context 🔎

**Condição (`if`)**: `N/A` (sempre roda)

**O que este job faz (detalhado)**
O script do `ctx`:

1. Lê o evento (`inputs.trigger_event_name` ou `GITHUB_EVENT_NAME`) e extrai:
   - `PR_ACTION`, `PR_MERGED`, `HEAD_BRANCH`, `BASE_BRANCH`
2. Classifica a branch (`head_kind`):
   - `bot` se `release-please--*` ou `dependabot/*`
   - `feature|bugfix|hotfix|release|main|other`
3. Define **Dev Flow** (`is_dev_flow=true`) somente quando:
   - evento é `pull_request` (**não** `pull_request_target`)
   - PR não está `closed`
   - `head_kind ∈ {feature, bugfix, hotfix}`
   - base é `main` **ou** `release/**` **ou** `feature/*`
4. Define **Promotion** (`should_run_release_promotion=true`) quando:
   - `create` de `release/**`, ou
   - `push` de criação de `release/**` (`created=true` **ou** `before=0000...`), ou
   - merge (PR closed+merged) de `bugfix/*`/`hotfix/*` em `release/**` (**bots são ignorados**)
5. Detecta **Publish** (`should_run_release_please_publish=true`) quando:
   - evento é `push` em `release/**`, e
   - o commit (`github.sha`) está associado a uma PR cuja `head.ref` começa com `release-please--*`
     (consulta `GET /repos/{repo}/commits/{sha}/pulls` — best-effort)

**Outputs principais do `ctx`**
- `event_name`, `head_branch`, `base_branch`, `head_kind`
- `is_dev_flow`
- `should_run_release_promotion` + `promo_ref` (branch `release/**` alvo)
- `should_run_release_please_pr` (espelho de promotion)
- `should_run_release_please_publish`

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Determine Context 🔎 | `N/A` | Define outputs via `id: ctx`. Executa script bash (ver seção do job). |

---

### `build`

**Condição (`if`)**: `${{ needs.ctx.outputs.event_name == 'pull_request' && needs.ctx.outputs.pr_action != 'closed' && needs.ctx.outputs.is_dev_flow == 'true' && needs.ctx.result == 'success' }}`

**O que faz**
Chama o reusable workflow `build-docker.yml` para buildar e fazer push da imagem (tags de PR/SHA) no Nexus/registry.

**Inputs repassados (principais)**
- `dockerfile`, `context`, `registry_repo`, `nexus_hosted`, `insecure_registry`

**Secrets repassados**
- `NEXUS_USER`, `NEXUS_PASS`

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Call reusable workflow (`uses:`) | `N/A` | Encaminha inputs/secrets para `mrrdev4/shared-workflows/.github/workflows/build-docker.yml@main`. Steps internos estão documentados no workflow chamado. |

---

### `resolve_digest`

**Condição (`if`)**: `${{ needs.ctx.outputs.should_run_release_promotion == 'true' || needs.ctx.outputs.should_run_release_please_publish == 'true' }}`

**O que faz (detalhado)**
Resolve o digest imutável da imagem no Nexus, usando Docker Registry HTTP API v2:

- Normaliza `inputs.nexus_hosted` (ou `vars.NEXUS_HOSTED`) aceitando:
  - `host`, `host/path`, `https://host/path`
- Monta `IMAGE_URI=<host>/<repo_path>/<app>`
- Decide o commit de referência:
  - Promotion: `HEAD` do checkout
  - Publish: `github.event.before` (commit anterior ao push na release/**)
- Busca PR associada ao commit via `GET /commits/<sha>/pulls?per_page=1`
- Testa tags candidatas em ordem:
  1. `pr-<PR>-<sha7>-rc` (se houver PR)
  2. `sha-<fullsha>`
  3. `sha-<sha7>`
- Para cada tag, faz `GET https://<host>/v2/<img_path>/manifests/<tag>` e lê `docker-content-digest`.

**Outputs do job**
- `digest`, `used_tag`, `pr_number`, `image_by_digest`, etc.

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Checkout | `N/A` | Executa action `actions/checkout@v4`. |
| 2 | Resolve Registry Vars | `N/A` | Define outputs via `id: vars`. Executa script bash (ver seção do job). |
| 3 | Resolve Digest | `N/A` | Define outputs via `id: out`. Executa script bash (ver seção do job). |

---

### `deploy_ti`

**Condição (`if`)**: `${{ inputs.enable_deploy_ti == 'true' && needs.ctx.outputs.should_run_release_promotion == 'true' && needs.resolve_digest.result == 'success' && needs.resolve_digest.outputs.image_by_digest != '' }}`

**O que faz**
Deploy em TI usando **imagem por digest** (`needs.resolve_digest.outputs.image_by_digest`) via `deploy-helm-openshift.yml`.

**Atenção**
- O job usa `environment: Ti` (GitHub Environment).
  Portanto o repositório do caller precisa ter o Environment **Ti** com `OC_SERVER` e `OC_TOKEN`.
- `value_files` usados: `value_files_ti` (se definido) senão `value_files`.

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Call reusable workflow (`uses:`) | `N/A` | Encaminha inputs/secrets para `mrrdev4/shared-workflows/.github/workflows/deploy-helm-openshift.yml@main`. Steps internos estão documentados no workflow chamado. |

---

### `deploy_trg`

**Condição (`if`)**: `${{ inputs.enable_deploy_trg == 'true' && needs.ctx.outputs.should_run_release_promotion == 'true' && needs.deploy_ti.result == 'success' }}`

**O que faz**
Deploy em TRG usando **imagem por digest** via `deploy-helm-openshift.yml`.

**Atenção**
- Depende do sucesso de `deploy_ti`. Se TI estiver desabilitado (`enable_deploy_ti="false"`), TRG tende a ficar **skipped** por dependência.
- Environment usado: `Trg`.

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Call reusable workflow (`uses:`) | `N/A` | Encaminha inputs/secrets para `mrrdev4/shared-workflows/.github/workflows/deploy-helm-openshift.yml@main`. Steps internos estão documentados no workflow chamado. |

---

### `release_please_pr`

**Condição (`if`)**: `${{ needs.deploy_trg.result == 'success' && needs.ctx.outputs.should_run_release_please_pr == 'true' }}`

**O que faz**
Cria/atualiza PR do Release-Please na branch `promo_ref` (release/**), com `skip-github-release=true` (só PR).

- Usa `secrets.RELEASE_PLEASE_TOKEN` se definido, senão `github.token`
- Arquivos usados: `release_please_config` e `release_please_manifest`

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Pick Token | `N/A` | Define outputs via `id: tok`. Executa script bash (ver seção do job). |
| 2 | (sem nome) | `N/A` | Executa action `googleapis/release-please-action@v4`. Define outputs via `id: rp`. |
| 3 | Capture Release-Please PR 🔗 | `N/A` | Define outputs via `id: pr`. Executa script bash (ver seção do job). |

---

### `release_please_publish`

**Condição (`if`)**: `${{ needs.ctx.outputs.should_run_release_please_publish == 'true' }}`

**O que faz**
Roda Release-Please em modo publish (`skip-github-pull-request=true`), criando Tag/Release no GitHub quando aplicável.

- `target-branch`: `needs.ctx.outputs.head_branch` (a própria `release/**` do push)
- Outputs relevantes: `releases_created`, `tag_name`

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Pick Token | `N/A` | Define outputs via `id: tok`. Executa script bash (ver seção do job). |
| 2 | (sem nome) | `N/A` | Executa action `googleapis/release-please-action@v4`. Define outputs via `id: rp`. |

---

### `retag_release_image`

**Condição (`if`)**: `${{ needs.ctx.outputs.should_run_release_please_publish == 'true' && needs.release_please_publish.outputs.releases_created == 'true' && needs.release_please_publish.outputs.tag_name != '' && needs.resolve_digest.outputs.digest != '' }}`

**O que faz (detalhado)**
Retaga a imagem por digest para a tag SemVer via `skopeo copy`:

- `SRC=docker://<host>/<img_path>@<digest>`
- `DST=docker://<host>/<img_path>:<tag_name>`
- Usa `--tls-verify=false` quando `insecure_registry=true`.

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Install Skopeo 🧰 | `N/A` | Executa script bash (ver seção do job). |
| 2 | Retag by digest | `N/A` | Define outputs via `id: retag`. Executa script bash (ver seção do job). |

---

### `deploy_prd`

**Condição (`if`)**: `${{ inputs.enable_deploy_prd == 'true' && needs.ctx.outputs.should_run_release_please_publish == 'true' && needs.retag_release_image.result == 'success' && needs.retag_release_image.outputs.image_ref != '' }}`

**O que faz**
Deploy em PRD usando a **tag SemVer** (`needs.retag_release_image.outputs.image_ref`) via `deploy-helm-openshift.yml`.

**Atenção**
- Environment usado: `Prd`.
- Se `enable_deploy_prd="false"`, o job será skipado e o `sync_release_to_main` também não roda (o `if` exige PRD success).

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Call reusable workflow (`uses:`) | `N/A` | Encaminha inputs/secrets para `mrrdev4/shared-workflows/.github/workflows/deploy-helm-openshift.yml@main`. Steps internos estão documentados no workflow chamado. |

---

### `sync_release_to_main`

**Condição (`if`)**: `${{ needs.release_please_publish.result == 'success' && needs.deploy_prd.result == 'success' && needs.release_please_publish.outputs.releases_created == 'true' && needs.release_please_publish.outputs.tag_name != '' }}`

**O que faz**
Abre PR `release/** → main` (best-effort) via `actions/github-script@v7`:

- Se já existir PR aberta com mesmo `head/base`, reutiliza.
- Se não houver diff, o GitHub API normalmente retorna **422**; o workflow captura e só emite warning.

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Open PR Release To Main | `N/A` | Executa action `actions/github-script@v7`. Define outputs via `id: sync_pr`. |

---

### `summary`

**Condição (`if`)**: `${{ always() }}`

**O que faz**
Consolida em `GITHUB_STEP_SUMMARY`:

- flags do `ctx`
- status dos jobs (success/skipped/failure)
- links da Release-Please PR e da PR de sync (quando existirem)
- digest/tag usados

| # | Step | Condição (`if`) | Detalhes |
|---:|---|---|---|
| 1 | Final summary | `N/A` | Executa script bash (ver seção do job). |

---

## Coerência e validações (o que realmente é “fail-fast”)

- ✅ `resolve_digest` e `retag_release_image` exigem `inputs.nexus_hosted` **ou** `vars.NEXUS_HOSTED` (senão falham).
- ✅ `resolve_digest` falha se **nenhuma** tag candidata existir no Nexus (lista candidatas e endpoint base).
- ✅ `retag_release_image` falha se `skopeo copy` falhar (auth/TLS/tag).
- ✅ `sync_release_to_main` é best-effort (422 “sem diff” vira warning e outputs vazios).

---

## FAQ de cenários (técnico)

### Criei `release/**` pela UI e não rodou promoção
Se o caller não tiver `on.create`, a criação pode chegar como `push` com `created=true` ou `before=0000...`.
✅ Garanta no caller pelo menos `push` em `release/**` — e preferencialmente também `create` em `release/**`.

### `deploy_trg` ficou skipped mesmo com `enable_deploy_trg="true"`
O `deploy_trg` depende de `deploy_ti.result == success`.
Se TI estiver desabilitado ou falhar, TRG não roda (por design atual).

### Publish não disparou após merge do Release-Please
O `ctx` detecta publish consultando PRs associadas ao commit do push e verificando se a `head.ref` começa com `release-please--`.
Se o merge gerar commit sem associação à PR (ou a API não retornar a PR), pode ficar **false** (best-effort).

### `resolve_digest` não encontrou digest
Significa que nenhuma das tags testadas existe no Nexus:
- `pr-<num>-<sha7>-rc` (se PR encontrada)
- `sha-<fullsha>`
- `sha-<sha7>`

Verifique se o `build` realmente publicou no mesmo path (`nexus_hosted`/`registry_repo`) e se as credenciais têm permissão de leitura em `/v2/.../manifests/<tag>`.

---

## Exemplo de caller (app repo)

```yaml
name: ReleaseFlow ♻️

on:
  pull_request:
    branches: [main, "release/**"]
    types: [opened, reopened, synchronize, ready_for_review, closed]
  push:
    branches: ["release/**"]
  create:
    branches: ["release/**"]
  workflow_dispatch: {}

permissions:
  contents: write
  pull-requests: write
  issues: write

concurrency:
  group: releaseflow-${{ github.repository }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  releaseflow:
    uses: mrrdev4/shared-workflows/.github/workflows/releaseflow.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      trigger_event_name: ${{ github.event_name }}

      main_branch: "main"
      release_prefix: "release/"

      # Build / Nexus
      dockerfile: "Dockerfile"
      context: "."
      nexus_hosted: ${{ vars.NEXUS_HOSTED }}
      registry_repo: "releaseflow"
      insecure_registry: false

      # Deploy (note que enable_* são strings)
      enable_deploy_ti: "true"
      enable_deploy_trg: "true"
      enable_deploy_prd: "true"
      namespace_ti: "meu-ns-ti"
      namespace_trg: "meu-ns-trg"
      namespace_prd: "meu-ns-prd"

      chart: "charts/myapp"
      values: ""
      value_files_ti: "values-ti.yaml"
      value_files_trg: "values-trg.yaml"
      value_files_prd: "values-prd.yaml"

      validate_image: true
      skip_tls_verify: false
      timeout_helm: "10m"
      atomic: true

      # Release Please
      release_please_config: "release-please-config.json"
      release_please_manifest: ".release-please-manifest.json"
```
