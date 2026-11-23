# Validate ReleaseFlow 🛡️ (Guard + Rules) `check-rules.yml`

Este workflow é o **“porteiro”** do ReleaseFlow: ele **entende o evento**, **detecta o tipo de branch**, **decide se o ReleaseFlow deve rodar** e **valida as regras do modelo de branches** (feature/bugfix/hotfix/release).

Ele foi feito como **Reusable Workflow** (`on: workflow_call`) para ser chamado pelos callers do `shared-workflows` (ex.: `devflow`, `releaseflow`, etc.).

---

## O que ele entrega

1. **Contexto normalizado** (event/head/base/tipo)
2. **Decisão de execução do ReleaseFlow** (`should_run` + `reason`)
3. **Decisão de Sonar** (`should_run_sonar` + `reason_sonar`)
4. **Validação das regras** (`ok=true/false`) + relatório detalhado no **Summary**

> **Importante**
- `Guard Decision` (decisão) e `Validate Rules` (validação) são coisas diferentes:
  - **Guard** decide **se** o releaseflow deve rodar.
  - **Validate** decide **se** o evento/branch está dentro das regras (e pode falhar o job).

---

## Trigger (como ele é executado)

```yaml
on:
  workflow_call:
```

Ou seja, ele **não** roda sozinho: precisa ser chamado por outro workflow via `uses: org/repo/.github/workflows/check-rules.yml@ref`.

---

## Inputs (contrato do workflow_call)

| Input | Tipo | Default | Para que serve |
|------|------|---------|----------------|
| `runs_on` | string (JSON array) | `["self-hosted","dev"]` | Labels do runner. O job usa `runs-on: ${{ fromJson(inputs.runs_on) }}`. |
| `main_branch` | string | `main` | Nome da branch principal (ex.: `main`). |
| `release_prefix` | string | `release/` | Prefixo das branches de release. |
| `feature_prefix` | string | `feature/` | Prefixo das branches feature. |
| `bugfix_prefix` | string | `bugfix/` | Prefixo das branches bugfix. |
| `hotfix_prefix` | string | `hotfix/` | Prefixo das branches hotfix. |
| `trigger_event_name` | string | `""` | Evento **original** do caller (push/pull_request/create/workflow_dispatch). Ajuda muito em `workflow_call`. |
| `push_created` | boolean | `false` | Copia `github.event.created` do caller (distingue “branch recém criada” via UI). |
| `ignore_bot_prs` | boolean | `true` | Ignora PRs de bots (`[bot]`, `release-please*`). |
| `allow_workflow_dispatch` | boolean | `true` | Permite rodar releaseflow via `workflow_dispatch`. |
| `hotfix_tag_regex` | string | `^v?[0-9]+\.[0-9]+\.[0-9]+.*$` | Regex para tags “release/semver” (base de hotfix). |

---

## Outputs (contrato do workflow_call)

> Estes outputs são expostos para o caller.

| Output | Tipo | Significado |
|--------|------|-------------|
| `ok` | string `true/false` | Resultado da validação final das regras. Se `false`, o job falha no final. |
| `branch_type` | string | `feature` \| `bugfix` \| `hotfix` \| `release` \| `automation` \| `other` |
| `head_branch` | string | Branch de origem do evento (head). |
| `base_branch` | string | Branch base (em PR). Em push/create normalmente vazio. |
| `event_name` | string | Evento efetivo considerado (prioriza `trigger_event_name`). |
| `should_run` | string `true/false` | Decisão do Guard: **se o ReleaseFlow deve rodar**. |
| `reason` | string | Motivo/“reason-code” da decisão. |
| `is_release_create` | string `true/false` | `true` quando é `create` de branch `release/**`. |
| `is_release_push` | string `true/false` | `true` quando é `push` em `release/**`. |
| `is_pr` | string `true/false` | `true` em eventos de PR. |
| `is_release_pr_merged` | string `true/false` | `true` quando um PR de `bugfix/**` ou `hotfix/**` foi **mergeado em** `release/**`. |
| `should_run_sonar` | string `true/false` | Decisão do Guard: se deve rodar Sonar nesse contexto. |
| `reason_sonar` | string | Motivo/“reason-code” para Sonar. |

---

## Permissões e dependências

### Job permissions
```yaml
permissions:
  contents: read
  pull-requests: read
```

### Ferramentas esperadas no runner
- `git`, `bash`, `curl`
- `python3` (usado para ler o JSON do evento em `GITHUB_EVENT_PATH`)

### Token GitHub
O workflow usa `github.token` como `GH_TOKEN` para chamadas à API:
- `GET /repos/{owner}/{repo}/commits/{sha}/pulls`
- `GET /repos/{owner}/{repo}/pulls/{number}`

---

## Estrutura do Job `validate`

O workflow tem um único job:

```yaml
jobs:
  validate:
```

Ele expõe outputs do job (internos) e é composto pelos steps abaixo.

---

## Step-by-step (o que cada step faz)

### 1) Checkout Full 📥
**Objetivo:** garantir histórico completo + tags para validações “best-effort”.

Principais pontos:
- `fetch-depth: 0`
- `fetch-tags: true`
- `ref:` é selecionado assim:
  - se `trigger_event_name` é PR → checkout do `github.event.pull_request.head.sha`
  - se `trigger_event_name` é `create` e `ref_type == branch` → `refs/heads/<ref>`
  - fallback → `github.sha`

Isso evita “checkout no ref errado” em `workflow_call`.

---

### 2) Fetch All Branches & Tags 📥
Executa:
```bash
git fetch --prune origin \
  "+refs/heads/*:refs/remotes/origin/*" \
  "+refs/tags/*:refs/tags/*"
```

**Objetivo:** garantir que existam localmente:
- `origin/main`
- `origin/release/**`
- tags (para hotfix)

---

### 3) Detect Context 📝 (`id: ctx`)
**Objetivo:** construir o contexto de forma robusta em reusable workflows.

Problema que ele resolve:
- em `workflow_call`, `github.ref_name`, `github.head_ref` e `github.base_ref` podem vir vazios ou incompletos.

Solução:
- lê o payload do evento em `GITHUB_EVENT_PATH` com `python3`.

Regras:
- PR (`pull_request` / `pull_request_target`):
  - `HEAD_BRANCH` = `pull_request.head.ref`
  - `BASE_BRANCH` = `pull_request.base.ref`
- create:
  - `HEAD_BRANCH` = `ref` (branch/tag)
- outros (push / workflow_dispatch / etc.):
  - tenta `ref` e remove `refs/heads/`
  - fallback: `github.ref_name`

**Detecção de `branch_type`:**
- `automation`: `release-please--*`, `release-please-*`, `dependabot/*`
- `feature`: prefixo `feature_prefix`
- `bugfix`: prefixo `bugfix_prefix`
- `hotfix`: prefixo `hotfix_prefix`
- `release`: prefixo `release_prefix`
- senão: `other`

Outputs do step:
- `event_name`, `head_branch`, `base_branch`, `branch_type`

---

### 4) Guard Decision 🛡️ (`id: guard`)
**Objetivo:** decidir **se o ReleaseFlow deve rodar** e **se Sonar deve rodar**.

> Esse step não “valida regras profundas”. Ele faz a triagem e decide.

Ele produz:
- `should_run`, `reason`
- `should_run_sonar`, `reason_sonar`
- flags: `is_pr`, `is_release_create`, `is_release_push`, `is_release_pr_merged`

#### 4.1) Automations
Se `branch_type=automation`:
- `should_run=false`
- `reason=automation`

#### 4.2) workflow_dispatch (manual)
Se `event=workflow_dispatch`:
- se `allow_workflow_dispatch=true` → `should_run=true`, `reason=manual-dispatch`
- senão → `should_run=false`, `reason=manual-dispatch-disabled`

#### 4.3) create de `release/**`
Se `event=create` e `ref_type=branch` e a branch começa com `release_prefix`:
- `is_release_create=true`
- verifica tags que apontam para `HEAD`:
  - se houver tags que casam com `hotfix_tag_regex`:
    - `should_run=false`
    - `reason=release-branch-created-from-tag:<tags> (skip)`
    - **Sonar habilitado** como baseline:
      `should_run_sonar=true`, `reason_sonar=release-baseline-created-from-tag:<tags>`
  - se houver tags, mas **não** casarem com a regex:
    - `should_run=false`
    - `reason=release-branch-created-from-nonrelease-tag:<tags> (skip)`
    - `should_run_sonar=false`
    - `reason_sonar=release-created-from-nonrelease-tag (no baseline)`
  - se **não** houver tags:
    - `should_run=true`
    - `reason=release-branch-created-from-branch`
    - `should_run_sonar=true`
    - `reason_sonar=release-baseline-on-create`

Se `create` mas não é `release/**`:
- `should_run=false`, `reason=create-not-release-branch`

#### 4.4) push em `release/**`
Se `event=push` e `head_branch` é `release/**`:
- `is_release_push=true`
- por padrão: `should_run_sonar=false`, `reason_sonar=push-release`

Casos:
1) `push_created=true` (fallback para repos que não escutam `create`)
   - repete lógica de tags:
     - criada de tag → **skip**
     - criada de branch → `should_run=true`, `reason=release-branch-created-from-branch (push-created-fallback)`, e Sonar baseline ligado
2) `push_created=false`
   - permite **somente** se o push veio de merge do **release-please**:
     - chama API: `/commits/<sha>/pulls`
     - procura PR cujo `head.ref` começa com `release-please--` ou `release-please-`
     - se achou:
       - `should_run=true`
       - `reason=release-please-merge-push:<head>`
     - senão:
       - `should_run=false`
       - `reason=push-release-disabled`

#### 4.5) push fora de release/** (DevFlow)
Se `event=push` e **não** é `release/**`:
- `should_run=false`, `reason=push-not-release-branch`

Mas Sonar pode ser ligado:
- se `head_branch` é `feature/**` ou `bugfix/**` ou `hotfix/**`:
  - conta `commits` no payload do push
  - se `push_created=true` e `commits_len=0` → `should_run_sonar=false` (`push-created-no-commits (skip sonar)`)
  - caso contrário → `should_run_sonar=true` (`push-dev-branch`)
- senão: `should_run_sonar=false` (`push-not-dev-branch`)

#### 4.6) PR (`pull_request` / `pull_request_target`)
- `is_pr=true`
- se `ignore_bot_prs=true` e o ator/usuário do PR é bot:
  - `should_run=false`, `reason=bot-pr`
  - `should_run_sonar=false`, `reason_sonar=bot-pr`

Se não for bot:
- obtém `action`, `merged` e `pr_head`
- PR aberta (`action != closed`):
  - `should_run=true`, `reason=pr-open`
  - Sonar só se head for feature/bugfix/hotfix:
    - `should_run_sonar=true`, `reason_sonar=pr-open`
    - senão: `should_run_sonar=false`, `reason_sonar=pr-open-not-feature-bugfix-hotfix`
- PR fechada:
  - `should_run_sonar=false`, `reason_sonar=pr-closed`
  - promove apenas se:
    - `base_branch` é `release/**`
    - e `merged=true`
    - e `pr_head` é `bugfix/**` ou `hotfix/**`
  - então:
    - `is_release_pr_merged=true`
    - `should_run=true`
    - `reason=pr-merged-into-release:<pr_head>`
  - caso contrário:
    - `should_run=false`
    - `reason=pr-not-merged-into-release` (ou `pr-merged-into-release-but-not-bugfix-hotfix:<pr_head>`)

---

### 5) Validate Rules 🛡️ (`id: validate`)
**Objetivo:** validar o modelo de branches e escrever um relatório detalhado no Summary.

Características:
- usa `continue-on-error: true` (para sempre escrever relatório)
- no final, um step separado falha o job se `ok != true`

#### Como ele monta o relatório
Ele mantém:
- `ok="true|false"`
- `report` acumulado
- funções:
  - `add "texto"`: adiciona no report e imprime no log
  - `fail "texto"`: marca `ok=false` e adiciona `❌ ...`

Um `trap` no `EXIT` garante que outputs (`ok` e `report`) sejam sempre gravados.

#### Recalcular contexto (local do step)
Ele recalcula contexto usando env padrão do Actions:
- PR: `GITHUB_HEAD_REF`, `GITHUB_BASE_REF`
- push/create: `GITHUB_REF_NAME`

E redetecta `branch_type` pelos prefixos.

#### Helpers “best-effort”
- `compare_with <base_ref>`:
  - `merge-base`
  - ahead/behind (contagem de commits)
- `best_release_ancestor()`:
  - percorre `origin/release/**` e escolhe a mais “próxima” que seja ancestral do HEAD
- `nearest_tag_by_regex()`:
  - procura a tag (que casa com `hotfix_tag_regex`) mais próxima e ancestral do HEAD

---

## Regras por tipo de branch

### `automation`
- Não bloqueia (validações ignoradas):
  - `release-please--*`, `release-please-*`, `dependabot/*`

---

### `feature/**`
**Origem (best-effort)**
- exige `origin/main` ancestral do HEAD
- registra `ahead/behind/merge-base`
- tenta indicar se foi criada do tip (quando `merge-base == tip` e `behind=0`)

**PR base permitida**
- somente `main` **ou** outra `feature/**`
- caso contrário: ❌

---

### `bugfix/**`
**Origem (best-effort)**
- exige existir algum `origin/release/**` ancestral do HEAD (`best_release_ancestor`)
- se não encontra: ❌

**PR base permitida**
- somente `release/**`
- caso contrário: ❌

---

### `hotfix/**`
**Origem (best-effort)**
- exige existir uma TAG ancestral que case com `hotfix_tag_regex`
- se não encontra: ❌
- se distância `0`, indica “criada diretamente da tag”

**PR base permitida**
- somente `release/**`
- caso contrário: ❌

---

### `release/**`
**Origem (best-effort)**
- permitido:
  - criada a partir de `main`
  - **ou** criada diretamente de TAG semver (distância `0`)
- se não for criada de TAG semver, exige `origin/main` ancestral do HEAD

**PR base permitida**
- PR cujo head é `release/**` só pode ter base `main`

**Push em `release/**`**
- se `push_created=true`: permitido (push de criação)
- se `push_created=false`: precisa estar associado a PR (via API)
  - se não tiver PR associado: ❌ “push direto”
  - se tiver PR:
    - imprime `head/base` do PR
    - **exceção**: “Update branch” do PR de sync (`release/** -> main`)
      - quando `pr_head == HEAD_BRANCH` e `pr_base == main`
      - permitido e anotado como OK (mas **não** deve iniciar o flow — isso é do Guard)
    - fora essa exceção, exige:
      - base do PR seja `release/**`
      - head do PR seja `bugfix/**` ou `hotfix/**` ou `release-please*` ou `dependabot/*`

---

## Summary e falha final

### Summary 📌
O workflow escreve no Summary:
- `success_ok`
- `should_run`
- `should_run_sonar`
- `reason` / `reason_sonar`
- flags `is_release_*`

### Fail if invalid ❌
Se `steps.validate.outputs.ok != 'true'`:
```bash
exit 1
```

---

## Como chamar no caller (exemplo)

```yaml
jobs:
  guard:
    name: Guard 🛡️
    uses: mrrdev4/shared-workflows/.github/workflows/check-rules.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      trigger_event_name: ${{ github.event_name }}
      push_created: ${{ github.event.created }}
      main_branch: "main"
      release_prefix: "release/"
      feature_prefix: "feature/"
      bugfix_prefix: "bugfix/"
      hotfix_prefix: "hotfix/"
      ignore_bot_prs: true
      allow_workflow_dispatch: true
      hotfix_tag_regex: '^v?[0-9]+\.[0-9]+\.[0-9]+.*$'
```

E usar as decisões:
```yaml
  releaseflow:
    needs: [guard]
    if: ${{ needs.guard.outputs.should_run == 'true' }}
    uses: mrrdev4/shared-workflows/.github/workflows/releaseflow.yml@<ref>
    secrets: inherit
```

```yaml
  sonar:
    needs: [guard]
    if: ${{ needs.guard.outputs.should_run_sonar == 'true' }}
    uses: mrrdev4/shared-workflows/.github/workflows/sonar-<lang>.yml@<ref>
    secrets: inherit
```

---

## Reason-codes úteis (para debug)

### `reason` (ReleaseFlow)
- `automation`
- `manual-dispatch`
- `manual-dispatch-disabled`
- `release-branch-created-from-branch`
- `release-branch-created-from-branch (push-created-fallback)`
- `release-branch-created-from-tag:<tags> (skip)`
- `release-branch-created-from-nonrelease-tag:<tags> (skip)`
- `push-release-disabled`
- `release-please-merge-push:<head>`
- `push-not-release-branch`
- `create-not-release-branch`
- `pr-open`
- `bot-pr`
- `pr-merged-into-release:<head>`
- `pr-not-merged-into-release`
- `pr-merged-into-release-but-not-bugfix-hotfix:<head>`

### `reason_sonar`
- `release-baseline-on-create`
- `release-baseline-created-from-tag:<tags>`
- `release-created-from-nonrelease-tag (no baseline)`
- `release-baseline-on-push-created-fallback`
- `push-release`
- `push-dev-branch`
- `push-created-no-commits (skip sonar)`
- `push-not-dev-branch`
- `pr-open`
- `pr-open-not-feature-bugfix-hotfix`
- `pr-closed`
- `bot-pr`

---

## Observações de implementação

- **Dois níveis de robustez de contexto**
  - `ctx` lê `GITHUB_EVENT_PATH` (mais confiável em reusable)
  - `validate` usa env padrão do Actions (mais simples, mas suficiente para as regras)
- **Validação “best-effort”**
  - A origem da branch é inferida por ancestrais (`merge-base`) e pode variar se `main` avançou.
  - O objetivo é pegar erros de origem mais comuns sem depender 100% de metadados do Git.

---

## Matriz de decisão e validações (evento × tipo de branch)

A matriz abaixo resume, para cada combinação relevante de **evento** e **tipo de branch**, o que o `check-rules.yml` faz:

- **`should_run`** / **`reason`**: decisão do *Guard* (se o `releaseflow.yml` deve rodar).
- **`should_run_sonar`** / **`reason_sonar`**: decisão do Guard (se o caller deve rodar Sonar).
- **Validações aplicadas**: checagens executadas no step **Validate Rules** (podem marcar `ok=false` e falhar o job).

> Observação: o step **Validate Rules** sempre roda (para gerar relatório), mesmo quando `should_run=false`.

### A) Evento `create`

| Tipo (head/ref) | should_run | should_run_sonar | Validações aplicadas (Validate Rules) |
|---|---:|---:|---|
| `create` de `release/**` **a partir de branch** (sem tags em `HEAD`) | ✅ `true` (`release-branch-created-from-branch`) | ✅ `true` (`release-baseline-on-create`) | **release origem**: permite se `origin/main` é ancestral do HEAD *(ou tag semver dist=0)* |
| `create` de `release/**` **a partir de tag semver** (tag em `HEAD` casa `hotfix_tag_regex`) | ⛔ `false` (`release-branch-created-from-tag:... (skip)`) | ✅ `true` (`release-baseline-created-from-tag:...`) | **release origem**: **permitido** se tag semver é a origem (dist=0). *Obs: “não deve iniciar o flow” fica registrado no report.* |
| `create` de `release/**` **a partir de tag NÃO-semver** (tag em `HEAD` não casa regex) | ⛔ `false` (`release-branch-created-from-nonrelease-tag:... (skip)`) | ⛔ `false` (`release-created-from-nonrelease-tag (no baseline)`) | **release origem**: tende a **falhar** se `origin/main` não for ancestral (bloqueia criação “errada” por tag fora do padrão) |
| `create` de qualquer **outra** branch (feature/bugfix/hotfix/other) | ⛔ `false` (`create-not-release-branch`) | ⛔ `false` | Validações específicas podem **não** rodar (depende do runner/env), mas o fluxo não inicia por design |

### B) Evento `push`

| Tipo (head) | should_run | should_run_sonar | Validações aplicadas (Validate Rules) |
|---|---:|---:|---|
| `push` em `release/**` com `push_created=true` (fallback da criação) **sem tags** | ✅ `true` (`release-branch-created-from-branch (push-created-fallback)`) | ✅ `true` (`release-baseline-on-push-created-fallback`) | **release origem**: mesma regra do `create` (main ancestral ou tag semver dist=0) |
| `push` em `release/**` com `push_created=true` **com tag semver em HEAD** | ⛔ `false` (`release-branch-created-from-tag:... (skip)`) | ⛔ `false` *(Guard não liga Sonar nesse fallback quando é tag)* | **release origem**: permitido se tag semver dist=0 (mas o flow não inicia) |
| `push` em `release/**` com `push_created=false` vindo de merge do **release-please** | ✅ `true` (`release-please-merge-push:<rp_head>`) | ⛔ `false` (`push-release`) | **release push**: exige PR associado ao commit; permite `pr_head=release-please*` com `pr_base=release/**` |
| `push` em `release/**` com `push_created=false` **sem** evidência de release-please | ⛔ `false` (`push-release-disabled`) | ⛔ `false` (`push-release`) | **release push**: normalmente **falha** se for push direto (sem PR associado) ou PR não permitido |
| `push` em `feature/**` / `bugfix/**` / `hotfix/**` com `push_created=true` e `commits=[]` (criação via UI) | ⛔ `false` (`push-not-release-branch`) | ⛔ `false` (`push-created-no-commits (skip sonar)`) | **origem best-effort**: valida se feature veio de `main`, bugfix veio de `release/**`, hotfix veio de tag semver |
| `push` em `feature/**` / `bugfix/**` / `hotfix/**` com commits normais | ⛔ `false` (`push-not-release-branch`) | ✅ `true` (`push-dev-branch`) | **origem best-effort** + relatório |
| `push` em qualquer outra branch | ⛔ `false` (`push-not-release-branch`) | ⛔ `false` (`push-not-dev-branch`) | Sem regras específicas |

### C) Evento `pull_request` (PR aberta / em atualização)

| Tipo (PR head) | should_run | should_run_sonar | Validações aplicadas (Validate Rules) |
|---|---:|---:|---|
| PR aberta de `feature/**` | ✅ `true` (`pr-open`) | ✅ `true` (`pr-open`) | **base permitida**: `main` ou `feature/**` |
| PR aberta de `bugfix/**` | ✅ `true` (`pr-open`) | ✅ `true` (`pr-open`) | **base permitida**: `release/**` |
| PR aberta de `hotfix/**` | ✅ `true` (`pr-open`) | ✅ `true` (`pr-open`) | **base permitida**: `release/**` |
| PR aberta de `release/**` | ✅ `true` (`pr-open`) | ⛔ `false` (`pr-open-not-feature-bugfix-hotfix`) | **base permitida**: `main` |
| PR aberta de `other` | ✅ `true` (`pr-open`) | ⛔ `false` (`pr-open-not-feature-bugfix-hotfix`) | Sem regras específicas |
| PR de bot (`[bot]` / `release-please*`) com `ignore_bot_prs=true` | ⛔ `false` (`bot-pr`) | ⛔ `false` (`bot-pr`) | Validações ignoradas pelo Guard; o job ainda reporta contexto |

### D) Evento `pull_request` (PR fechada)

| Cenário | should_run | should_run_sonar | Validações aplicadas (Validate Rules) |
|---|---:|---:|---|
| PR **mergeada** `bugfix/**` → `release/**` | ✅ `true` (`pr-merged-into-release:<head>`) | ⛔ `false` (`pr-closed`) | Regras de base `release/**` (ok) + report |
| PR **mergeada** `hotfix/**` → `release/**` | ✅ `true` (`pr-merged-into-release:<head>`) | ⛔ `false` (`pr-closed`) | Regras de base `release/**` (ok) + report |
| PR mergeada “outro head” → `release/**` | ⛔ `false` (`pr-merged-into-release-but-not-bugfix-hotfix:<head>`) | ⛔ `false` (`pr-closed`) | Pode falhar se base/head não permitido |
| PR fechada sem merge (qualquer) | ⛔ `false` (`pr-not-merged-into-release`) | ⛔ `false` (`pr-closed`) | Report apenas (sem promoção) |

### E) Evento `workflow_dispatch` (manual)

| Onde foi disparado | should_run | should_run_sonar | Validações aplicadas (Validate Rules) |
|---|---:|---:|---|
| Manual com `allow_workflow_dispatch=true` | ✅ `true` (`manual-dispatch`) | ⛔ `false` (`not-applicable`) | Aplica regras do tipo de branch atual (feature/bugfix/hotfix/release) se o contexto permitir |
| Manual com `allow_workflow_dispatch=false` | ⛔ `false` (`manual-dispatch-disabled`) | ⛔ `false` | Idem (validações ainda reportam) |

---

## FAQ de cenários (casos reais do dia a dia)

### 1) “Criei a branch via UI e o Sonar não rodou”
Quando a branch é criada pelo GitHub UI, muitas vezes o evento chega como:
- `push_created=true`
- `commits=[]`

Nesse caso o Guard define:
- `should_run_sonar=false`
- `reason_sonar=push-created-no-commits (skip sonar)`

✅ **Como resolver:** faça **um commit** (qualquer alteração) na branch — no próximo push o Sonar passa a rodar (`push-dev-branch`).

---

### 2) “Criei uma `release/**` e não caiu no `create`, caiu no `push`”
Alguns repos/callers não escutam `create`, então a primeira atualização da branch chega como `push` com:
- `push_created=true`

O Guard trata isso como **fallback**:
- `reason=release-branch-created-from-branch (push-created-fallback)`
- e liga Sonar baseline (`release-baseline-on-push-created-fallback`).

---

### 3) “Criei `release/**` a partir de tag `vX.Y.Z` e o flow não iniciou”
Isso é esperado para cenários de hotfix:
- `should_run=false`
- `reason=release-branch-created-from-tag:... (skip)`

Mas ele **pode ligar Sonar baseline** para gerar uma base de análise:
- `should_run_sonar=true`
- `reason_sonar=release-baseline-created-from-tag:...`

---

### 4) “Criei `release/**` a partir de uma tag que não é semver e falhou”
Se a tag **não** casa `hotfix_tag_regex`, o Guard marca como `skip`, mas o **Validate Rules** tende a falhar:
- porque `release/**` só é aceita vindo de `main` **ou** de tag semver (distância 0)

✅ **Como resolver:** use tag semver (ex.: `v1.2.3`) ou crie a `release/**` a partir da `main`.

---

### 5) “Por que push em `release/**` é bloqueado?”
Para evitar “drift” e garantir rastreabilidade, **push direto em release** é considerado inválido.

O Validate Rules exige:
- push em release associado a **PR**
- e PR deve ser:
  - `bugfix/**` → `release/**`
  - `hotfix/**` → `release/**`
  - ou automations (`release-please*`, `dependabot/*`)

Se não houver PR associado ao commit, ele falha com:
- “Push direto em release/** não permitido…”

✅ **Como resolver:** sempre alterar `release/**` via **PR**.

---

### 6) “Cliquei `Update branch` no PR de sync (release → main) e gerou push na release/**”
Isso gera um **merge commit** em `release/**` associado a um PR cuja base é `main`.
Sem exceção, o Validate Rules falharia — por isso existe uma **exceção explícita**:

Se:
- `pr_head == HEAD_BRANCH` (a própria release)
- e `pr_base == main`

Então:
- ✅ permitido
- e o report marca: “Update branch do PR de sync (release → main) — permitido (não inicia o flow).”

---

### 7) “O release-please mergeou algo na release/** e o Sonar não rodou”
Em `push` em `release/**`, o Guard define:
- `should_run_sonar=false`
- `reason_sonar=push-release`

Isso é proposital: o publish/tag do release-please não deve rodar análises pesadas nesse evento.

✅ **Como rodar Sonar na release:** ele é ligado no baseline (`create`/`push_created`) e em PRs abertas de feature/bugfix/hotfix.

---

### 8) “Bugfix criada da main está falhando no validate”
Bugfix deve ter origem (best-effort) em alguma `origin/release/**`. Se não encontrar uma release ancestral, ele falha:

- “bugfix/** deve ser criada a partir de `release/**`…”

✅ **Como resolver:** crie bugfix a partir da **release** correta.

---

### 9) “Hotfix criada de release (sem tag) está falhando no validate”
Hotfix deve ter origem em tag semver (regex `hotfix_tag_regex`). Se não achar tag compatível no histórico, falha:

- “hotfix/** deve ter origem em TAG…”

✅ **Como resolver:** crie hotfix a partir de uma **tag semver** (ex.: `v1.2.3`), não da branch.

---

### 10) “Meu PR feature → release/** falha”
Feature PR só pode ter base:
- `main` **ou** `feature/**`

✅ **Como resolver:** feature deve ir para `main`. Para ajustar release use `bugfix/**` ou `hotfix/**`.
