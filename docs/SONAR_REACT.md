# Sonar Scan React ⚛️ `sonar-react.yml`

Reusable workflow (`workflow_call`) para rodar **SonarQube scan** em aplicações **React** (Node/JS/TS), com suporte a:

- detecção automática de package manager (pnpm/yarn/npm)
- build/test/coverage opcionais (LCOV)
- parâmetros de PR vs branch
- Quality Gate
- suporte a **certificado interno** (truststore Java + `NODE_EXTRA_CA_CERTS`)
- outputs para dashboard e status do Quality Gate

---

## Quando usar

- PRs → análise com `sonar.pullrequest.*`
- Branches (push) → análise com `sonar.branch.name`
- Projetos com coverage:
  - `coverage/lcov.info` (default recomendado)
  - controla obrigatoriedade com `coverage_required`

---

## Pré-requisitos do runner

Este workflow assume que o runner possui:

- `bash` (shell padrão)
- `node`/`corepack` (a versão alvo é instalada no workflow)
- `openssl` (somente quando `SONAR_ROOT_CERT` não for informado; fallback `openssl s_client`)
- utilitários básicos: `awk`, `sed`, `grep`, `cat`
- `keytool` (vem do JDK instalado no step `Setup Java`)

---

## Contrato do `workflow_call`

### Inputs

#### Execução e diretório
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels do runner. Usado em `runs-on: fromJson(inputs.runs_on)`. Precisa ser JSON válido (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `context` | `string` | `no` | `.` | Diretório base do app React/Node (onde está `package.json`). Usado para detectar package manager/scripts e também como `projectBaseDir` do Sonar scan. Deve existir no workspace. |
| `sonar_enabled` | `boolean` | `no` | `True` | Liga/desliga as etapas de Sonar (setup Java, truststore, args, scan, quality gate, dashboard e summary). **Atenção:** o step `Build` não depende desta flag (só de `has_build`). |

#### Node.js
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `node-version` | `string` | `no` | `20` | Versão do Node.js instalada no runner (ex.: `20`, `22`). |

#### Testes e coverage
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `unit_tests_enabled` | `boolean` | `no` | `True` | Se `false`, não roda testes/coverage e o Sonar roda sem `sonar.javascript.lcov.reportPaths`. Se `true`, roda testes/coverage apenas quando `sonar_enabled=true` e existir script `test`. |
| `coverage_required` | `boolean` | `no` | `False` | Se `true` e `unit_tests_enabled=true`, falha se o arquivo `sonar_coverage` não existir ou estiver vazio. Se `false`, apenas warning e segue sem coverage. |
| `sonar_coverage` | `string` | `no` | `coverage/lcov.info` | Caminho relativo ao `context` do arquivo LCOV (ex.: `coverage/lcov.info`). Usado para validação e para `sonar.javascript.lcov.reportPaths` quando encontrado. |

#### Sonar projeto
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_host_url` | `string` | `no` | `${{ vars.SONAR_HOST_URL }}` | URL do SonarQube (ex.: `https://sonar.seudominio`). Usado no scan e também para extrair o hostname na importação do certificado. |
| `sonar_project_key` | `string` | `no` | `` | Chave do projeto no Sonar (`sonar.projectKey`). **Obrigatória na prática** quando `sonar_enabled=true`. |
| `sonar_project_name` | `string` | `no` | `` | Nome do projeto (`sonar.projectName`). Se vazio, o workflow usa fallback para o nome do repositório. |
| `sonar_project_version` | `string` | `no` | `` | Versão (`sonar.projectVersion`). Se vazio: PR → `pr-<num>-<sha7>-rc`; branch → `<sha7>`. |

#### Escopo de análise
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_sources` | `string` | `no` | `.` | Diretórios de código para análise (`sonar.sources`). Default `.` (relativo ao `projectBaseDir=context`). |
| `sonar_tests` | `string` | `no` | `.` | Diretórios de testes (`sonar.tests`). Default `.`. N/A quando você não quer setar explicitamente (deixe vazio). |
| `sonar_test_inclusions` | `string` | `no` | `**/*.test.js,**/*.test.jsx,**/*.test.ts,**/*.test.tsx,**/*.spec.js,**/*.spec.jsx,**/*.spec.ts,**/*.spec.tsx` | Glob de testes (`sonar.test.inclusions`). N/A quando vazio. |
| `sonar_exclusions` | `string` | `no` | `**/node_modules/**,**/dist/**,**/build/**,**/coverage/**` | Exclusões (`sonar.exclusions`). N/A quando vazio. |

#### Quality Gate e extras
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_quality_gate_timeout_sec` | `number` | `no` | `300` | Timeout do Quality Gate em segundos (polling). Default `300`. |
| `sonar_extra_args` | `string` | `no` | `` | Args extras `-D...` anexados literalmente ao final do args do scanner (ex.: `-Dsonar.javascript.node.maxspace=4096`). N/A quando vazio. |

---

### Secrets

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `SONAR_TOKEN` | `condicional` | Token de análise do SonarQube. **Obrigatório na prática** quando `sonar_enabled=true`. |
| `SONAR_ROOT_CERT` | `condicional` | CA raiz (PEM) do SonarQube. Se ausente, o workflow faz fallback via `openssl s_client` para capturar o chain e importar no truststore. |

---

### Outputs

| Output | Descrição (técnica) |
|---|---|
| `sonar_dashboard_url` | URL do dashboard do projeto no SonarQube: `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`. N/A quando `sonar_enabled=false`. |
| `quality_gate_status` | Status do Quality Gate (ex.: `OK`, `ERROR`). N/A quando `sonar_enabled=false`. |

---

## Permissões, concorrência e defaults

### Permissions
```yaml
contents: read
pull-requests: read
```

### Concurrency
```yaml
group: react-sonar-${{ github.repository }}-${{ github.event.pull_request.number || github.ref_name }}
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
| 1 | Checkout 📋 | `N/A` | Faz checkout do repositório (idealmente `fetch-depth: 0`) para suportar parâmetros de PR/branch e cálculo de version/tag. |
| 2 | Validate Context 📁 | `N/A` | Valida que `inputs.context` existe antes de instalar deps/rodar build/test/scan. |
| 3 | Detect Package Manager 🔎 | `N/A` | Detecta automaticamente `pnpm`/`yarn`/`npm` via lockfiles e define prefixo de execução (`pnpm run`, `yarn`, `npm run`). |
| 4 | Set Up Node.js 🛠️ | `N/A` | Instala Node na versão `node-version` e habilita cache compatível com o package manager detectado (quando configurado no workflow). |
| 5 | Enable Corepack 📦 | `N/A` | Executa `corepack enable` para suportar pnpm/yarn gerenciados pelo Node. |
| 6 | Install dependencies 📦 | `N/A` | Instala dependências com base no manager detectado (ex.: `pnpm install --frozen-lockfile`, `yarn install --immutable`, `npm ci`). |
| 7 | Detect Scripts (build/test/coverage) 🔎 | `N/A` | Lê `package.json` e detecta `scripts.build`, `scripts.test` e scripts de coverage (`test:coverage`/`coverage`). |
| 8 | Build 🧱 | ``${{ steps.scripts.outputs.has_build == 'true' }}`` | Executa build somente se existir `scripts.build`. **Não depende de `sonar_enabled`**. |
| 9 | Test With Coverage 🧪 | ``${{ inputs.sonar_enabled && inputs.unit_tests_enabled && steps.scripts.outputs.has_test == 'true' }}`` | Roda testes e tenta gerar coverage (preferência: script `test:coverage`/`coverage`, fallback: `test -- --coverage`). |
| 10 | Verify Coverage File ✅ | ``${{ inputs.sonar_enabled && inputs.unit_tests_enabled && inputs.sonar_coverage != '' }}`` | Verifica se o arquivo LCOV existe e não está vazio. Se `coverage_required=true` e não encontrar, falha antes do scan. |
| 11 | Setup Java ⚒️ | ``${{ inputs.sonar_enabled }}`` | Instala JDK para criar truststore e executar o scanner com `SONAR_SCANNER_OPTS`. |
| 12 | Trust SonarQube Certificate 🔐 | ``${{ inputs.sonar_enabled }}`` | Cria truststore `/tmp/my_truststore` e exporta `/tmp/sonar-ca.pem` (define `NODE_EXTRA_CA_CERTS`). |
| 13 | Build Sonar args 🧩 | ``${{ inputs.sonar_enabled }}`` | Monta `-D...` do scanner: projectKey/name/version, PR vs branch, sources/tests/exclusions e coverage (quando encontrado). |
| 14 | Sonar Scan 🔍 | ``${{ inputs.sonar_enabled }}`` | Executa o scan com `projectBaseDir=context` e args montados no step anterior. |
| 15 | Quality Gate 🧱 | ``${{ inputs.sonar_enabled }}`` | Executa o Quality Gate lendo `<context>/.scannerwork/report-task.txt` com timeout `sonar_quality_gate_timeout_sec`. |
| 16 | Compute Dashboard URL 🔗 | ``${{ inputs.sonar_enabled }}`` | Monta URL do dashboard e salva em output `sonar_dashboard_url`. |
| 17 | Summary 📄 | ``${{ inputs.sonar_enabled }}`` | Escreve resumo no `GITHUB_STEP_SUMMARY` (key, dashboard e status do quality gate). |

---

## Detalhamento técnico (comportamento)

### Detect Package Manager
Ordem de detecção (best-effort):

- **pnpm** quando existe `pnpm-lock.yaml`
- **yarn** quando existe `yarn.lock`
- **npm** quando existe `package-lock.json` / `npm-shrinkwrap.json`
- fallback → assume npm

Isso define o prefixo de execução (ex.: `pnpm run`, `yarn`, `npm run`) e o comando de instalação.

### Detect Scripts (build/test/coverage)
Lê `package.json` e detecta:

- `has_build`: existe `scripts.build`
- `has_test`: existe `scripts.test`
- `has_test_coverage`: existe `scripts.test:coverage` ou `scripts.coverage`
  - se existir, define `coverage_script` como `test:coverage` (prioridade) ou `coverage`

### Tests & Coverage
O step **Test With Coverage** roda quando:

- `sonar_enabled=true`
- `unit_tests_enabled=true`
- e existe `scripts.test`

Estratégia:
1) se existir `scripts.test:coverage` ou `scripts.coverage`, roda esse script
2) fallback: roda `test -- --coverage` (padrão Jest/Vitest)

### Verify Coverage File
- considera `sonar_coverage` (default: `coverage/lcov.info`)
- se o arquivo existir e estiver não vazio:
  - `found=true` e `path=<arquivo>`
- senão:
  - warning
  - se `coverage_required=true`, falha

### Trust SonarQube Certificate
Cria truststore `/tmp/my_truststore` a partir do `cacerts` do Java e importa CA do Sonar:

- se `secrets.SONAR_ROOT_CERT` existir (PEM) → importa como alias `sonarqube-ca`
- senão → faz `openssl s_client` no host do Sonar e importa todo o chain

Exporta também:
- `NODE_EXTRA_CA_CERTS=/tmp/sonar-ca.pem`
- o scanner usa `SONAR_SCANNER_OPTS` apontando para o truststore

### Build Sonar args
Se `sonar_project_version` estiver vazio:
- PR: `pr-<PR_NUMBER>-<SHA7>-rc`
- branch/push: `<SHA7>`

Monta os `-D...`:

- sempre:
  - `sonar.sourceEncoding=UTF-8`
  - `sonar.projectVersion=<version>`
  - `sonar.projectKey=<sonar_project_key>` (**obrigatório na prática**)
  - `sonar.projectName=<sonar_project_name>` (fallback repo name)
  - `sonar.sources=<sonar_sources>`
  - `sonar.verbose=true`
- opcionais: `sonar.tests`, `sonar.test.inclusions`, `sonar.exclusions`
- coverage:
  - se `unit_tests_enabled=true` e coverage encontrado:
    - `sonar.javascript.lcov.reportPaths=<path>`
- PR vs branch:
  - PR: `sonar.pullrequest.key/branch/base`
  - branch: `sonar.branch.name=<ref_name>`
- `sonar_extra_args` é anexado literal no final.

---

## Matriz de comportamento (inputs/evento × resultado)

### A) `sonar_enabled`
| `sonar_enabled` | O que acontece |
|---:|---|
| `false` | Pula: Setup Java, certificado, args, scan, quality gate, dashboard e summary. **Build ainda pode rodar** se existir `scripts.build`. |
| `true` | Executa fluxo completo de análise (com testes/coverage conforme flags). |

### B) `unit_tests_enabled` e coverage
| `unit_tests_enabled` | `coverage_required` | Comportamento |
|---:|---:|---|
| `false` | qualquer | Não roda testes/coverage. Sonar roda sem `sonar.javascript.lcov.reportPaths`. |
| `true` | `false` | Tenta gerar coverage; se ausente/vazio, warning e scan segue sem coverage. |
| `true` | `true` | Se ausente/vazio → falha antes do scan. |

### C) PR vs Branch
| Evento | Parâmetros no Sonar | `sonar.projectVersion` (auto) |
|---|---|---|
| `pull_request` / `pull_request_target` | `sonar.pullrequest.key/branch/base` | `pr-<num>-<sha7>-rc` |
| `push` / outros | `sonar.branch.name` | `<sha7>` |

---

## Coerência e gotchas (técnico)

- ✅ Coverage só é injetado no Sonar quando o arquivo existe e não está vazio.
- ⚠ Se `sonar_enabled=false`, o workflow ainda pode fazer **build** (se existir `scripts.build`). Se você quer “pular tudo”, use `if` no **caller**.
- ⚠ Projetos sem `scripts.test` não rodam testes mesmo com `unit_tests_enabled=true` (porque o workflow detecta e só executa se existir).
- ⚠ Se seu padrão de arquivo LCOV é diferente, ajuste `sonar_coverage`.

---

## FAQ (cenários comuns)

### 1) “Não tenho testes ainda, quero rodar Sonar mesmo assim”
✅ Use `unit_tests_enabled: false`. O Sonar roda normalmente, só sem coverage.

### 2) “Coverage não aparece no Sonar”
O workflow só injeta `sonar.javascript.lcov.reportPaths` quando `sonar_coverage` existe e não está vazio.

### 3) “Meu projeto usa pnpm/yarn”
Não precisa mudar nada: o workflow detecta automaticamente pelo lockfile.

### 4) “Falha de SSL/certificado no SonarQube”
✅ Configure `secrets.SONAR_ROOT_CERT` (PEM) quando possível.

### 5) “Quero passar parâmetros extras”
✅ Use `sonar_extra_args` para anexar `-D...` extras.

---

## Exemplo de uso (caller)

```yaml
jobs:
  sonar:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/sonar-react.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      node-version: "22"

      sonar_enabled: true
      unit_tests_enabled: true
      coverage_required: false
      sonar_coverage: "coverage/lcov.info"

      sonar_project_key: "mrrdev4_meu-react"
      sonar_project_name: "meu-react"
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}

      sonar_sources: "."
      sonar_tests: "."
      sonar_test_inclusions: "**/*.{{spec,test}}.{{js,jsx,ts,tsx}}"
      sonar_exclusions: "**/node_modules/**,**/dist/**,**/build/**,**/coverage/**"

      sonar_quality_gate_timeout_sec: 300
      sonar_extra_args: ""
```

> Se você estiver chamando isso condicionado pelo Guard do ReleaseFlow, use `if: needs.guard.outputs.should_run_sonar == 'true'`.

---

## Checklist rápido

- [ ] `package.json` presente em `context`
- [ ] `sonar_project_key` definido (quando `sonar_enabled=true`)
- [ ] `SONAR_TOKEN` disponível (quando `sonar_enabled=true`)
- [ ] Se quiser coverage: script de coverage gera `coverage/lcov.info` (ou ajuste `sonar_coverage`)
- [ ] Se exigir coverage: `coverage_required=true`
