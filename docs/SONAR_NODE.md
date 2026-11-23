# Sonar Scan para Node.js 🟢 `sonar-node.yml`

Reusable workflow (`workflow_call`) para executar **SonarQube scan** em projetos **Node.js/TypeScript**, com suporte a:

- detecção automática de package manager (**pnpm / yarn / npm**)
- build opcional (quando existir `scripts.build`)
- execução opcional de **tests + coverage** (LCOV)
- PR analysis (`sonar.pullrequest.*`) e branch analysis (`sonar.branch.name`)
- **Quality Gate** (polling)
- suporte a **certificado interno** (truststore Java + `NODE_EXTRA_CA_CERTS`)
- outputs para dashboard e status do Quality Gate

---

## Quando usar

- **PRs** → análise com parâmetros `sonar.pullrequest.*`
- **Branches** (push) → análise com `sonar.branch.name`
- Projetos com coverage (recomendado):
  - ter `scripts.test:coverage` **ou** `scripts.coverage` no `package.json`
  - gerar `coverage/lcov.info` (default)

---

## Pré-requisitos do runner

Este workflow assume que o runner possui:

- `bash`
- `python3` (tipicamente usado para parse/URL-encode do dashboard e/ou hostname; quando indisponível, ajuste o workflow)
- `openssl` **somente** quando `SONAR_ROOT_CERT` não for informado (fallback de captura do chain)
- utilitários básicos: `awk`, `sed`, `grep`, `cat`, `test`
- `keytool` (vem do JDK instalado pelo `actions/setup-java`)

---

## Contrato do `workflow_call`

### Inputs

#### Execução e diretório
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels do runner. Usado em `runs-on: fromJson(inputs.runs_on)`. Precisa ser JSON válido (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `context` | `string` | `no` | `.` | Diretório do projeto Node (onde está o `package.json`). Os steps de install/build/test normalmente operam nesse diretório. Em monorepo, aponte para o subdiretório do app. |
| `sonar_enabled` | `boolean` | `no` | `true` | Liga/desliga Sonar (certificado, args, scan, quality gate, dashboard e summary). **Importante:** os steps de Node (setup/instalação) e o build (se existir script) podem rodar mesmo com `false` (o `if` do build depende do `scripts.build`). |

#### Node.js
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `node-version` | `string` | `no` | `20` | Versão do Node usada no `actions/setup-node`. Ex.: `20`, `22`. |

#### Sonar (projeto)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_host_url` | `string` | `no` | `${{ vars.SONAR_HOST_URL }}` | URL base do SonarQube. Usada em `SONAR_HOST_URL` e para compor `sonar_dashboard_url`. Também é usada para extrair o hostname na importação do certificado. |
| `sonar_project_key` | `string` | `no` | `""` | **Obrigatório na prática** quando `sonar_enabled=true`. O workflow falha no step de build dos args se estiver vazio. |
| `sonar_project_name` | `string` | `no` | `""` | Nome do projeto (`sonar.projectName`). Se vazio, normalmente usa fallback com o nome do repo (quando implementado no script de args). |
| `sonar_project_version` | `string` | `no` | `""` | Se vazio, calcula automaticamente: **PR** → `pr-<num>-<sha7>-rc`; **branch** → `<sha7>`. |

#### Escopo de análise
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_sources` | `string` | `no` | `.` | Diretórios de source (`sonar.sources`). Normalmente relativo ao `context`. |
| `sonar_tests` | `string` | `no` | `.` | Diretórios de teste (`sonar.tests`). Se você não quer declarar explicitamente, deixe `N/A` (vazio) e não injete a propriedade. |
| `sonar_test_inclusions` | `string` | `no` | `**/*.test.js,**/*.test.jsx,**/*.test.ts,**/*.test.tsx,**/*.spec.js,**/*.spec.jsx,**/*.spec.ts,**/*.spec.tsx` | Glob de testes (`sonar.test.inclusions`). N/A quando vazio. |
| `sonar_exclusions` | `string` | `no` | `""` | Exclusões (`sonar.exclusions`). N/A quando vazio. |

#### Testes e coverage
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `unit_tests_enabled` | `boolean` | `no` | `true` | Se `false`, não executa testes/coverage e o Sonar roda sem `sonar.javascript.lcov.reportPaths`. |
| `coverage_required` | `boolean` | `no` | `false` | Se `true` e `unit_tests_enabled=true`, falha quando o arquivo de coverage (`sonar_coverage`) não existir ou estiver vazio. |
| `sonar_coverage` | `string` | `no` | `coverage/lcov.info` | Caminho do LCOV. O workflow valida existência/tamanho e só injeta `sonar.javascript.lcov.reportPaths` quando o arquivo existe e está não vazio. Use `""` para desabilitar coverage. |

#### Quality Gate e extras
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_quality_gate_timeout_sec` | `number` | `no` | `300` | Timeout (segundos) do polling do quality gate (`sonarqube-quality-gate-action`). |
| `sonar_extra_args` | `string` | `no` | `""` | Args extras `-D...` anexados literalmente ao final do args. N/A quando vazio. Evite aspas complexas (depende do parsing do script). |

---

### Secrets

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `SONAR_TOKEN` | `condicional` | Token do SonarQube. **Obrigatório na prática** quando `sonar_enabled=true`. |
| `SONAR_ROOT_CERT` | `condicional` | CA raiz em PEM (certificado interno). Se não informado, o workflow faz fallback via `openssl s_client` para capturar o chain e importar no truststore. |

---

### Outputs

| Output | Descrição (técnica) |
|---|---|
| `sonar_dashboard_url` | URL do dashboard no SonarQube: `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`. N/A quando `sonar_enabled=false`. |
| `quality_gate_status` | Status retornado pelo quality gate (ex.: `OK`, `ERROR`). N/A quando `sonar_enabled=false`. |

---

## Permissões, concorrência e defaults

### Permissions (do workflow)
```yaml
contents: read
pull-requests: read
```

### Concurrency (do workflow)
```yaml
group: node-sonar-${{ github.repository }}-${{ github.event.pull_request.number || github.ref_name }}
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
| 1 | Checkout 📥 | `N/A` | Checkout do repo (`fetch-depth: 0`). |
| 2 | Validate Context 📁 | `N/A` | Valida que `inputs.context` existe antes de rodar install/build/test. |
| 3 | Setup Java ⚒️ | `${{ inputs.sonar_enabled }}` | Instala JDK (para `keytool`) e prepara o ambiente para truststore do Sonar. |
| 4 | Detect Package Manager 🔎 | `N/A` | Detecta pnpm/yarn/npm por lockfiles e define comandos de install/run. |
| 5 | Set Up Node.js 🟢 | `N/A` | Configura Node (`actions/setup-node`) com a versão informada. |
| 6 | Enable Corepack 📦 | `N/A` | Habilita Corepack (necessário para pnpm/yarn modernos). N/A para npm, mas não atrapalha. |
| 7 | Install Dependencies 📦 | `N/A` | Instala dependências (`pnpm install` / `yarn install` / `npm ci|install`) no `context`. |
| 8 | Detect Scripts 🔎 | `N/A` | Lê `package.json` e define flags: `has_build`, `has_test`, `has_test_coverage` e `coverage_script`. |
| 9 | Node Build 🧱 | `${{ steps.scripts.outputs.has_build == 'true' }}` | Executa `run build` pelo package manager detectado. |
| 10 | Tests & Coverage 🧪 | `${{ inputs.sonar_enabled && inputs.unit_tests_enabled && steps.scripts.outputs.has_test == 'true' }}` | Executa testes e tenta gerar coverage (prioriza `test:coverage`/`coverage`). |
| 11 | Verify Coverage File ✅ | `${{ inputs.sonar_enabled && inputs.unit_tests_enabled && inputs.sonar_coverage != '' }}` | Valida se o arquivo `sonar_coverage` existe/não está vazio; falha se `coverage_required=true`. |
| 12 | Trust SonarQube Certificate 🔐 | `${{ inputs.sonar_enabled }}` | Cria `/tmp/my_truststore`, importa CA e exporta `NODE_EXTRA_CA_CERTS` e `SONAR_SCANNER_OPTS`. |
| 13 | Build Sonar args 🧩 | `${{ inputs.sonar_enabled }}` | Monta args `-D...` (PR vs branch) e injeta LCOV quando disponível. |
| 14 | Sonar Scan 🔍 | `${{ inputs.sonar_enabled }}` | Roda `SonarSource/sonarqube-scan-action` com `projectBaseDir=context`. |
| 15 | Quality Gate 🧱 | `${{ inputs.sonar_enabled }}` | Faz polling do QG via `sonarqube-quality-gate-action` apontando para `<context>/.scannerwork/report-task.txt`. |
| 16 | Compute Dashboard URL 🔗 | `${{ inputs.sonar_enabled }}` | Calcula e exporta URL do dashboard (`dashboard?id=<projectKey>`). |
| 17 | Summary 📄 | `${{ inputs.sonar_enabled }}` | Publica resumo no `GITHUB_STEP_SUMMARY` com dashboard e status. |

---

## Detalhes das lógicas principais

### Detect Package Manager
Heurística por lockfile:

- **pnpm**: `pnpm-lock.yaml`
- **yarn**: `yarn.lock`
- **npm**: `package-lock.json` ou `npm-shrinkwrap.json`
- fallback: npm

**Comandos típicos**
- pnpm: `pnpm install --frozen-lockfile` / `pnpm run <script>`
- yarn: `yarn install --frozen-lockfile` / `yarn <script>`
- npm: `npm ci` (se lockfile) senão `npm install` / `npm run <script>`

### Detect Scripts
Lê `package.json` e detecta:

- `has_build`: existe `scripts.build`
- `has_test`: existe `scripts.test`
- `has_test_coverage`: existe `scripts.test:coverage` **ou** `scripts.coverage`
  - define `coverage_script` como `test:coverage` (prioridade) ou `coverage`

### Tests & Coverage
Ordem de execução (quando roda):
1. Se existir `scripts.test:coverage` ou `scripts.coverage`, roda esse script (recomendado).
2. Fallback: roda `test -- --coverage` (compatível com Jest/Vitest em vários setups).

> **Importante:** o step só roda se existir `scripts.test`. Se seu projeto usa outro nome (ex.: `test:ci`), adicione `scripts.test` ou adapte o workflow.

### Coverage (LCOV)
- O workflow só injeta `sonar.javascript.lcov.reportPaths` quando:
  - `unit_tests_enabled=true`
  - o arquivo `sonar_coverage` existe e está não vazio

Se não existir:
- `coverage_required=false` → warning (N/A) e segue sem coverage
- `coverage_required=true` → falha antes do scan

### Trust SonarQube Certificate
Cria truststore em `/tmp/my_truststore`:

- copia o `cacerts` do JDK
- importa CA via `SONAR_ROOT_CERT` (alias `sonarqube-ca`) **ou** fallback `openssl s_client` (importa o chain)
- exporta:
  - `NODE_EXTRA_CA_CERTS=/tmp/sonar-ca.pem`
  - `SONAR_SCANNER_OPTS` com `-Djavax.net.ssl.trustStore=/tmp/my_truststore ...`

### Build Sonar args
Monta propriedades `-D...`, tipicamente incluindo:

- obrigatórias: `sonar.projectKey`, `sonar.projectName`, `sonar.sources`
- PR vs branch:
  - PR: `sonar.pullrequest.key/branch/base`
  - branch: `sonar.branch.name`
- coverage (se existir): `sonar.javascript.lcov.reportPaths=<sonar_coverage>`
- extras: `sonar_extra_args`

---

## Matriz de comportamento (inputs/evento × resultado)

### A) `sonar_enabled`
| `sonar_enabled` | O que acontece |
|---:|---|
| `false` | Pula: certificado, args, scan, quality gate, dashboard e summary. O setup Node/install e o build (se existir) podem rodar (N/A). |
| `true` | Executa fluxo completo de análise. |

### B) `unit_tests_enabled` e coverage
| `unit_tests_enabled` | `coverage_required` | Comportamento |
|---:|---:|---|
| `false` | N/A | Não roda testes/coverage. Sonar roda sem `sonar.javascript.lcov.reportPaths`. |
| `true` | `false` | Tenta gerar coverage; se não existir, warning e segue sem coverage. |
| `true` | `true` | Se o arquivo não existir/vazio → falha antes do scan. |

### C) PR vs Branch
| Evento | Parâmetros no Sonar | `sonar.projectVersion` (auto) |
|---|---|---|
| `pull_request` / `pull_request_target` | `sonar.pullrequest.key/branch/base` | `pr-<num>-<sha7>-rc` |
| `push` / outros | `sonar.branch.name` | `<sha7>` |

---

## Coerência e gotchas (técnico)

- ✅ `sonar_project_key` é **obrigatório** quando `sonar_enabled=true` (falha no step de args).
- ✅ `SONAR_TOKEN` é **obrigatório** quando `sonar_enabled=true` (scan/quality gate).
- ⚠ Sem `scripts.test`, o step de testes não roda (mesmo com `unit_tests_enabled=true`).
- ⚠ O coverage só entra no Sonar se o arquivo existir e estiver não vazio.
- ✅ Para ambientes com SSL interno, prefira `SONAR_ROOT_CERT` (evita dependência do `openssl s_client`).

---

## FAQ (cenários comuns)

### 1) “Não tenho testes ainda, quero rodar Sonar mesmo assim”
Use `unit_tests_enabled: false`.

### 2) “Coverage não aparece no Sonar”
Garanta que:
- o job gerou `coverage/lcov.info` (ou ajuste `sonar_coverage`)
- o arquivo existe e não está vazio (senão o workflow não injeta `sonar.javascript.lcov.reportPaths`)

### 3) “O fallback `test -- --coverage` não funciona”
Crie `scripts.test:coverage` (recomendado) ou `scripts.coverage` no `package.json`.

### 4) “Falha de SSL/certificado no SonarQube”
Configure `secrets.SONAR_ROOT_CERT` (PEM) quando possível.

### 5) “Quero passar parâmetros extras do Sonar”
Use `sonar_extra_args` com `-D...` adicionais.

---

## Exemplo de uso (caller)

```yaml
jobs:
  sonar:
    uses: mrrdev4/shared-workflows/.github/workflows/sonar-node.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      node-version: "22"

      sonar_enabled: true
      sonar_project_key: "mrrdev4_meu-projeto-node"
      sonar_project_name: "meu-projeto-node"
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}

      unit_tests_enabled: true
      coverage_required: false
      sonar_coverage: "coverage/lcov.info"

      sonar_quality_gate_timeout_sec: 300
      sonar_extra_args: ""
```

> Se você estiver chamando isso condicionado pelo Guard do ReleaseFlow, use `if: needs.guard.outputs.should_run_sonar == 'true'`.
