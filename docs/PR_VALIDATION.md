# `pr-validation.yml` — Validate PR Structure 🔍

Workflow reutilizável para **validar PRs** garantindo consistência de:
- **Título** (Conventional Commits / semantic PR title)
- **Corpo** (template com seções obrigatórias)

> Objetivo: padronizar PRs para o ReleaseFlow, facilitar revisão, changelog e auditoria.

---

## Como chamar (reusable)

Esse workflow é do tipo `workflow_call` (reutilizável). Ele não define `on: pull_request` por conta própria — quem dispara é o **caller**.

---

## Jobs

| Job | Runs-on | O que valida |
|---|---|---|
| `LintPrTitle` | `['self-hosted','dev']` | Título da PR segue Conventional Commits (`type(scope)!: subject`) via action semantic-pull-request. |
| `ValidatePrBody` | `['self-hosted','dev']` | Corpo da PR contém seções obrigatórias (template) via github-script. |

---

## Passo a passo (o que cada step faz)

| # | Job/Step | Condição (`if`)                                                                                                                                                                                                                       |
|--:|---|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | Job `LintPrTitle` | ``                                                                                                                                                                                                                                    |
| 2 | Job `ValidatePrBody` | `${{ github.event_name == 'pull_request' && !( github.actor == 'dependabot[bot]' \|\| startsWith(github.head_ref, 'dependabot/') \|\|  startsWith(github.head_ref, 'release-please--') \|\|  startsWith(github.head_ref, 'release/') ) }}` |

---

## Regras de Título (Conventional Commits)

Validação é feita por `amannn/action-semantic-pull-request@v5` no job `LintPrTitle`.

### Tipos permitidos

- `feat`
- `feat!`
- `fix`
- `chore`
- `docs`
- `style`
- `refactor`
- `perf`
- `test`
- `ci`
- `build`
- `revert`

### Formato esperado

- `type: subject`
- `type(scope): subject`
- `type!: subject`
- `type(scope)!: subject`

> Observação: o `!` indica **breaking change**.

---

## Regras do Corpo da PR (template obrigatório)

Validação é feita com `actions/github-script@v7` no job `ValidatePrBody`.

### Seções exigidas (o corpo precisa conter todas)

- `## 🔍 Descrição resumida`
- `## 🎟️ Issue relacionada`
- `## ✅ Checklist de mudanças`
- `## 🧪 Cobertura de testes`
- `## 🩹 Plano de rollback`

Se faltar alguma seção, o job falha com mensagem listando o que está faltando.

---

## Matriz de cenários (resultado esperado)

| Cenário | Título válido? | Corpo com todas seções? | Resultado |
|---|:---:|:---:|---|
| PR ok | ✅ | ✅ | ✅ Passa |
| Título inválido | ❌ | ✅ | ❌ Falha no `LintPrTitle` |
| Corpo incompleto | ✅ | ❌ | ❌ Falha no `ValidatePrBody` |
| Ambos ruins | ❌ | ❌ | ❌ Falha (em pelo menos um job) |

---

## FAQ

### 1) “Minha PR está falhando por causa do corpo, mas eu tenho o conteúdo — por que?”
O script valida por **`includes()`** (match exato). Se você mudou o texto do heading, emojis ou espaços, ele considera como ausente.

✅ Solução:
- manter exatamente os headings do template (copiar e colar)
- ou alterar o workflow para tolerar variações (regex/normalização)

### 2) “Quero adicionar novas seções obrigatórias”
Edite o array `requiredSections` no `github-script`.

### 3) “Quero permitir mais tipos no Conventional Commit”
Adicione no input `types` da action `amannn/action-semantic-pull-request`.

---

## Exemplo de uso (caller)

### Caller simples (valida PRs para `main` e `release/**`)

```yaml
name: PR Validation ✅

on:
  pull_request:
    branches:
      - main
      - "release/**"
    types: [opened, reopened, synchronize, ready_for_review]

permissions:
  contents: read
  pull-requests: read

jobs:
  pr_validation:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/pr-validation.yml@<ref>
```
