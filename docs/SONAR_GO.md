# Sonar Scan para Go 🐹 `sonar-go.yml`

Reusable workflow para rodar **SonarQube scan** em projetos **Golang**, com suporte a:

- setup automático de versão do Go (`go.work`/`go.mod`) com fallback
- build via comando configurável
- execução opcional de testes + coverage
- parâmetros de PR vs branch
- Quality Gate
- suporte a **certificado interno** (truststore Java + `NODE_EXTRA_CA_CERTS`)
- outputs para dashboard e status do Quality Gate

---

## Quando usar

- PRs → análise com parâmetros `sonar.pullrequest.*`
- Branches (push) → análise com `sonar.branch.name`
- Projetos com coverage:
  - default: `coverage.out` (Go coverprofile)
  - controla obrigatoriedade com `coverage_required`

---

## Contrato do `workflow_call`

### Inputs

#### Execução e diretório
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | ["self-hosted","dev"] | Lista JSON de labels do runner. Usado no `runs-on: fromJson(inputs.runs_on)`. Precisa ser JSON válido (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `context` | `string` | `no` | . | Diretório base do projeto para o scan (`projectBaseDir`) e para setup-go (auto-detect em `go.work`/`go.mod`). **Atenção:** `go_build_cmd` e `go_test_cmd` rodam no workspace root (não fazem `cd context`). Em monorepos inclua `cd "${{ inputs.context }}"` dentro dos comandos. |
| `sonar_enabled` | `boolean` | `no` | true | Liga/desliga o scan e Quality Gate. **Mesmo com `false`**, o workflow ainda roda checkout + setup-go + `go_build_cmd` (esses steps não têm `if sonar_enabled`). Os steps de Sonar (Java/cert/args/scan/qg/dashboard/summary) ficam N/A. |

#### Go (versão e comandos)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `go-version-file` | `string` | `no` | "" (vazio) | Arquivo para `actions/setup-go` (ex.: `go.mod`, `go.work`, `.tool-versions`). Quando vazio, o workflow tenta auto-detectar `go.work` e depois `go.mod` em `inputs.context`. |
| `go-version` | `string` | `no` | 1.25 | Versão do Go usada **somente** no fallback (quando `go-version-file==""` e não existe `go.work` nem `go.mod` no `context`). |
| `go_build_cmd` | `string` | `no` | go build -v ./... | Comando de build executado via `eval`. Falha o job se retornar exit code != 0. Em monorepo, inclua `cd` para o módulo correto. Para “pular” build, use `true`. |
| `go_test_cmd` | `string` | `no` | go test -v ./... -coverprofile=coverage.out -covermode=atomic | Comando de testes/coverage via `eval`. O step termina com `test -s <sonar_coverage>` (arquivo precisa existir e não estar vazio). N/A quando `unit_tests_enabled=false`. |

#### Testes e coverage
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `unit_tests_enabled` | `boolean` | `no` | true | Quando `true`, executa testes/coverage se `sonar_enabled=true`. Quando `false`, pula testes e o Sonar roda sem coverage. |
| `coverage_required` | `boolean` | `no` | false | Quando `true`, falha se `sonar_coverage` estiver ausente/vazio (somente se `unit_tests_enabled=true`). Quando `false`, emite warning e segue. |
| `sonar_coverage` | `string` | `no` | coverage.out | Caminho do coverage report (Go coverprofile). Usado na validação e em `sonar.go.coverage.reportPaths`. Se vazio, desabilita coverage. |

#### Sonar projeto
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_host_url` | `string` | `no` | ${{ vars.SONAR_HOST_URL }} | URL do SonarQube. Copiada para `env.SONAR_HOST_URL` e usada para extrair hostname na importação do certificado. |
| `sonar_project_key` | `string` | `no` | "" (vazio) | Chave do projeto no Sonar (`sonar.projectKey`). **Obrigatório** quando `sonar_enabled=true` (workflow falha se vazio). |
| `sonar_project_name` | `string` | `no` | "" (vazio) | Nome do projeto (`sonar.projectName`). Se vazio, usa o nome do repositório. |
| `sonar_project_version` | `string` | `no` | "" (vazio) | Versão (`sonar.projectVersion`). Se vazio: PR → `pr-<num>-<sha7>-rc`; branch → `<sha7>`. |

#### Escopo de análise
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_sources` | `string` | `no` | . | Sources (`sonar.sources`). Default `.` relativo ao `projectBaseDir=context`. |
| `sonar_tests` | `string` | `no` | . | Tests (`sonar.tests`). Se vazio, não injeta `sonar.tests` (N/A). |
| `sonar_test_inclusions` | `string` | `no` | **/*_test.go | Glob de testes (`sonar.test.inclusions`). Se vazio, não injeta (N/A). |
| `sonar_exclusions` | `string` | `no` | "" (vazio) | Exclusões (`sonar.exclusions`). Se vazio, não injeta (N/A). |

#### Quality Gate e extras
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_quality_gate_timeout_sec` | `number` | `no` | 300 | Timeout do Quality Gate em segundos (polling). Default 300. |
| `sonar_extra_args` | `string` | `no` | "" (vazio) | Args extras `-D...` anexados literalmente ao final do args. N/A quando vazio. |

---

### Secrets

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `SONAR_TOKEN` | `condicional` | Token de análise do SonarQube. Obrigatório quando `sonar_enabled=true` (falha no step `Build Sonar args` se vazio). N/A quando `sonar_enabled=false`. |
| `SONAR_ROOT_CERT` | `condicional` | CA raiz em PEM (certificado interno). Se não informado, o workflow faz fallback via `openssl s_client` e importa o chain no truststore. |

> Observação: `SONAR_TOKEN` só é exigido quando `sonar_enabled=true`.

---

### Outputs

| Output | Descrição (técnica) |
|---|---|
| `sonar_dashboard_url` | URL do dashboard do projeto no SonarQube: `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`. N/A quando `sonar_enabled=false`. |
| `quality_gate_status` | Status retornado pelo Quality Gate (ex.: `OK`, `ERROR`). N/A quando `sonar_enabled=false`. |

---

## Permissões, concorrência e defaults

### Permissions
```yaml
contents: read
pull-requests: read
```

### Concurrency
```yaml
group: go-sonar-${{ github.repository }}-${{ github.event.pull_request.number || github.ref_name
  }}
cancel-in-progress: true
```

### Defaults
```yaml
run:
  shell: bash
```

---

## Passo a passo (o que cada step faz)

| # | Step | Condição (`if`) | Detalhes (técnico) |
|---:|---|---|---|
| 1 | Checkout 📥 | `N/A` | Action: `actions/checkout@v4`. |
| 2 | Validate Context 📁 | `N/A` | Script bash. |
| 3 | Setup Java ⚒️ | `${{ inputs.sonar_enabled }}` | Action: `actions/setup-java@v5`. |
| 4 | Detect Go Version File ⚒️ | `${{ inputs.go-version-file == '' }}` | Outputs via `id: detect`. Script bash. |
| 5 | Set Up Go (Explicit File) ⚒️ | `${{ inputs.go-version-file != '' }}` | Action: `actions/setup-go@v5`. |
| 6 | Set Up Go (Auto) ⚒️ | `${{ inputs.go-version-file == '' && steps.detect.outputs.fallback != '1' }}` | Action: `actions/setup-go@v5`. |
| 7 | Set Up Go (Fallback) ⚒️ | `${{ inputs.go-version-file == '' && steps.detect.outputs.fallback == '1' }}` | Action: `actions/setup-go@v5`. |
| 8 | Go Build 🧱 | `N/A` | Script bash. |
| 9 | Tests & Coverage 🧪 | `${{ inputs.sonar_enabled && inputs.unit_tests_enabled }}` | Script bash. |
| 10 | Verify Coverage File ✅ | `${{ inputs.sonar_enabled && inputs.unit_tests_enabled && inputs.sonar_coverage != '' }}` | Outputs via `id: cov`. Script bash. |
| 11 | Trust SonarQube Certificate 🔐 | `${{ inputs.sonar_enabled }}` | Script bash. |
| 12 | Build Sonar args 🧩 | `${{ inputs.sonar_enabled }}` | Outputs via `id: sonarargs`. Script bash. |
| 13 | Sonar Scan 🔍 | `${{ inputs.sonar_enabled }}` | Action: `SonarSource/sonarqube-scan-action@v7.0.0`. |
| 14 | Quality Gate 🧱 | `${{ inputs.sonar_enabled }}` | Action: `SonarSource/sonarqube-quality-gate-action@v1.2.0`. Outputs via `id: quality-gate`. |
| 15 | Compute Dashboard URL 🔗 | `${{ inputs.sonar_enabled }}` | Outputs via `id: dashboard`. Script bash. |
| 16 | Summary 📄 | `${{ inputs.sonar_enabled }}` | Script bash. |

---

## Detalhamento técnico por etapa

### 1) Checkout
- `actions/checkout@v4` com `fetch-depth: 0`.

### 2) Validate Context
- **Como está no workflow hoje:** apenas `test -d .` (valida o workspace root).
- **Atenção:** não valida `inputs.context`. Se `context` não existir, o erro aparece depois (no detect, no scan ou no quality gate).

### 3) Setup Java (Temurin 17)
- Necessário para:
  - importar certificado no truststore (`keytool`)
  - executar scan com `SONAR_SCANNER_OPTS` apontando para `/tmp/my_truststore`

### 4–7) Setup do Go (auto / explícito / fallback)
Ordem de prioridade:

1) `go-version-file` (input explícito): `go-version-file: <context>/<go-version-file>`
2) Auto-detect em `<context>`:
   - `go.work` → usa `go.work`
   - `go.mod` → usa `go.mod`
3) Fallback: usa `go-version` (default do workflow)

### 8) Go Build
- Executa `eval "${ inputs.go_build_cmd }"`.
- **Fail-fast:** qualquer erro quebra o job (inclusive quando `sonar_enabled=false`).

> Monorepo: inclua `cd "${ inputs.context }"` no comando.

### 9–10) Tests & Coverage + Verify Coverage
- **Tests & Coverage** roda apenas quando `sonar_enabled=true` e `unit_tests_enabled=true`.
- O step termina com `test -s "${ inputs.sonar_coverage }"` (arquivo precisa existir e não estar vazio).
- **Verify Coverage File**:
  - `found=true/false`
  - se `coverage_required=true` e `found=false` → falha
  - se `coverage_required=false` → warning e segue

### 11) Trust SonarQube Certificate
Cria truststore em `/tmp/my_truststore`:

- Copia `cacerts` do Java
- Se `SONAR_ROOT_CERT` existir: importa como alias `sonarqube-ca`
- Caso contrário: fallback via `openssl s_client` e importa o chain com aliases `sonarqube-<host>-NN`

Também exporta `NODE_EXTRA_CA_CERTS=/tmp/sonar-ca.pem`.

### 12) Build Sonar args
- Falha cedo se:
  - `sonar_project_key` estiver vazio
  - `SONAR_TOKEN` estiver vazio (quando `sonar_enabled=true`)
- Calcula versão (se não informada):
  - PR: `pr-<num>-<sha7>-rc`
  - branch: `<sha7>`
- Injeta coverage **somente** quando encontrado (`steps.cov.outputs.found=true`).
- PR vs branch:
  - PR: `sonar.pullrequest.key/branch/base`
  - branch: `sonar.branch.name`
- Anexa `sonar_extra_args` ao final.

### 13–14) Scan + Quality Gate
- Scan: `SonarSource/sonarqube-scan-action@v7.0.0` com `projectBaseDir=context`.
- Quality Gate: `scanMetadataReportFile: <context>/.scannerwork/report-task.txt`.

### 15–16) Dashboard + Summary
- Dashboard: `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`.
- Summary escreve `GITHUB_STEP_SUMMARY` com key, dashboard e status.

---

## Matriz de comportamento (inputs/evento × resultado)

### A) `sonar_enabled`
| `sonar_enabled` | O que acontece |
|---:|---|
| `false` | Roda: checkout + setup-go + **Go Build**. Pula: Java/cert/args/scan/quality gate/dashboard/summary (N/A). |
| `true` | Roda fluxo completo (build + scan), com testes/coverage conforme flags. |

### B) `unit_tests_enabled` e coverage
| `unit_tests_enabled` | `coverage_required` | Comportamento |
|---:|---:|---|
| `false` | N/A | Não roda testes/coverage. Sonar roda sem `sonar.go.coverage.reportPaths`. |
| `true` | `false` | Tenta gerar coverage; se ausente/vazio, warning e scan segue sem coverage. |
| `true` | `true` | Se ausente/vazio → falha antes do scan. |

### C) PR vs Branch
| Evento | Parâmetros no Sonar | `sonar.projectVersion` (auto) |
|---|---|---|
| `pull_request` / `pull_request_target` | `sonar.pullrequest.key/branch/base` | `pr-<num>-<sha7>-rc` |
| `push` / outros | `sonar.branch.name` | `<sha7>` |

---

## Coerência (validações reais e gotchas)

- ✅ `sonar_project_key` e `SONAR_TOKEN` são validados no step **Build Sonar args** (somente com `sonar_enabled=true`).
- ⚠ `inputs.context` não é validado explicitamente; se estiver errado, falha em steps posteriores.
- ⚠ `go_build_cmd`/`go_test_cmd` não fazem `cd context`. Em monorepo, inclua `cd` nos comandos.
- ✅ O Quality Gate aponta `report-task.txt` dentro do `context`.

---

## FAQ (cenários comuns)

### 1) “Não tenho testes ainda, quero rodar Sonar mesmo assim”
Use `unit_tests_enabled: false`.

### 2) “Coverage não aparece no Sonar”
O workflow só injeta `sonar.go.coverage.reportPaths` quando o arquivo existe e não está vazio.

### 3) “Meu monorepo Go usa outro diretório”
Use `context` e ajuste comandos:

```yaml
go_build_cmd: 'cd "${ inputs.context }" && go build -v ./...'
go_test_cmd: 'cd "${ inputs.context }" && go test -v ./... -coverprofile=coverage.out -covermode=atomic'
sonar_coverage: 'coverage.out'
```

### 4) “Falha de SSL/certificado no SonarQube”
Configure `secrets.SONAR_ROOT_CERT` (PEM) quando possível.

### 5) “Quero passar parâmetros extras do Sonar”
Use `sonar_extra_args` com `-D...` extras.

---

## Exemplo de uso (caller)

```yaml
jobs:
  sonar:
    uses: mrrdev4/shared-workflows/.github/workflows/sonar-go.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      go-version: "1.25"
      unit_tests_enabled: true
      coverage_required: false
      sonar_project_key: "mrrdev4_meu-projeto-go"
      sonar_project_name: "meu-projeto-go"
      sonar_host_url: ${ vars.SONAR_HOST_URL }
      sonar_coverage: "coverage.out"
      sonar_quality_gate_timeout_sec: 300
      sonar_extra_args: ""
```

> Se você estiver chamando isso condicionado pelo Guard do ReleaseFlow, use `if: needs.guard.outputs.should_run_sonar == 'true'`.
