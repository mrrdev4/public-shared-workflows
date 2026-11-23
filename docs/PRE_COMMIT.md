# Pre Commit Hooks 🧰 `pre-commit.yml`

Reusable workflow para executar **pre-commit hooks** de forma padronizada, com suporte a repositórios Go e hooks diversos.

Ele faz:
- detecção automática se o repo é Go (go.mod/go.work)
- detecção se existe `.pre-commit-config.yaml`
- setup de Go (quando aplicável) e instalação de ferramentas Go
- setup de Node.js (quando existir `.pre-commit-config.yaml`)
- execução do `pre-commit/action@v3`

---

## Contrato do `workflow_call`

### Inputs
| Input | Tipo | Required | Default | Descrição |
|---|---|:---:|---|---|
| `go-version` | `string` | `no` | `1.24.5` | Go version to use |
| `node-version` | `string` | `no` | `22` | Node.js version to use |

### Secrets
_(sem secrets)_

### Outputs
_(sem outputs)_

---

## Passo a passo (o que cada step faz)

| # | Step | Condição (`if`) |
|---:|---|---|
| 1 | Checkout repository 📋 | `` |
| 2 | Pre-commit config | `` |
| 3 | Set up Go 🛠️ | `steps.detect.outputs.is_go == 'true'` |
| 4 | Setup Node.js environment 🟢 | `steps.detect.outputs.has_precommit == 'true'` |
| 5 | Install Go tools 🔧 | `steps.detect.outputs.is_go == 'true'` |
| 6 | Run pre-commit hooks 🏃 | `steps.detect.outputs.has_precommit == 'true'` |
| 7 | Skip info | `steps.detect.outputs.has_precommit != 'true'` |

### Detecção inicial
O workflow faz 2 detecções iniciais e expõe via `GITHUB_OUTPUT`:

- `is_go`
  - `true` se o repo contém **`go.mod`** ou **`go.work`** em qualquer pasta (`git ls-files | grep ...`)
  - `false` caso contrário

- `has_precommit`
  - `true` se existe **`.pre-commit-config.yaml`** na raiz
  - `false` caso contrário


### Setup e ferramentas Go
Quando `is_go=true`, ele instala ferramentas via `go install`:

- `github.com/lietu/go-pre-commit@latest` (helper de hooks Go)
- `github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.3.0`
- `github.com/kisielk/errcheck@latest`
- `mvdan.cc/gofumpt@latest`
- `honnef.co/go/tools/cmd/staticcheck@latest`

> Obs.: como várias estão em `@latest`, os resultados podem mudar com o tempo. Se quiser reprodutibilidade total, é melhor piná-las por versão.


---

## Matriz de comportamento (detecção × ações)

| `is_go` | `has_precommit` | Setup Go | Install Go tools | Setup Node | Roda pre-commit | Resultado |
|---:|---:|:---:|:---:|:---:|:---:|---|
| false | false | — | — | — | — | Apenas imprime “Pulando hooks” |
| true  | false | ✅ | ✅ | — | — | Instala ferramentas Go, mas não roda hooks |
| false | true  | — | — | ✅ | ✅ | Roda hooks sem setup Go |
| true  | true  | ✅ | ✅ | ✅ | ✅ | Setup completo + hooks |


---

## FAQ

### 1) “Por que ele não rodou nenhum hook?”
Se não existir `.pre-commit-config.yaml`, o workflow apenas imprime a mensagem de *skip*.

### 2) “Meu repo não é Go, mas tem `.pre-commit-config.yaml`”
Ele roda normalmente. O `setup-go` e instalação de ferramentas Go só acontecem quando `go.mod`/`go.work` existir.

### 3) “Uso hooks que dependem de Node/Python/Java…”
O workflow só garante ambiente **Node** (e opcionalmente Go).
Se seus hooks exigirem Python/Java/etc., você pode:
- instalar essas runtimes no `.pre-commit-config.yaml` via `language: python`/`node` etc. (o `pre-commit` baixa ambientes), ou
- estender o workflow para fazer `setup-python`, `setup-java`, etc. conforme necessidade.

### 4) “O workflow instalou ferramentas Go, mas não rodou hooks”
Isso acontece quando `is_go=true` e `has_precommit=false`.
Se você quer evitar esse custo, dá pra condicionar `Install Go tools` também em `has_precommit=true`.

### 5) “Quero fixar versões de ferramentas”
Troque os `@latest` por versões específicas (ex.: `@v0.6.0`) para estabilidade.


---

## Exemplo de uso (caller)

```yaml
jobs:
  pre_commit:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/pre-commit.yml@<ref>
    with:
      go-version: "1.24.5"
      node-version: "22"
```

> Dica: se você quiser que isso rode automaticamente em PRs, faça o trigger no **caller** (`on: pull_request`) e chame esse workflow via `uses:`.
