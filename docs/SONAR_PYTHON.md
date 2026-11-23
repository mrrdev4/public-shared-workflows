# Sonar Scan para Python 🐍 `sonar-python.yml`

Reusable workflow (`workflow_call`) para rodar **SonarQube scan** em projetos **Python**, com suporte a:

- detecção automática de toolchain (`uv` / `poetry` / `pipenv` / `pip`)
- execução opcional de testes + coverage (preferência `pytest-cov` → `coverage.xml`)
- PR analysis (`sonar.pullrequest.*`) e branch analysis (`sonar.branch.name`)
- Quality Gate
- suporte a **certificado interno** (truststore Java + `NODE_EXTRA_CA_CERTS`)
- outputs para dashboard e status do Quality Gate

---

## Quando usar

- PRs → análise com `sonar.pullrequest.*`
- Branches (push) → análise com `sonar.branch.name`
- Projetos com coverage:
  - default `coverage.xml` (Cobertura XML)
  - controla obrigatoriedade com `coverage_required`

---

## Pré-requisitos do runner

Este workflow assume que o runner possui:

- `bash` (shell padrão)
- `python3` (para execução dos testes/ferramentas; a versão alvo é instalada pelo `setup-python`)
- `openssl` (somente quando `SONAR_ROOT_CERT` não for informado; fallback via `openssl s_client`)
- utilitários básicos: `awk`, `sed`, `grep`
- `keytool` (vem do JDK instalado pelo `actions/setup-java`)

---

## Contrato do `workflow_call`

### Inputs

#### Execução e diretório
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | `["self-hosted","dev"]` | Lista JSON de labels do runner. Usado em `runs-on: fromJson(inputs.runs_on)`. Precisa ser JSON válido (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `context` | `string` | `no` | `.` | Diretório base do projeto (onde estão `pyproject.toml`/`requirements.txt` etc.). É usado como `projectBaseDir` no scan e também como diretório para detecção de tool (`uv`/`poetry`/`pipenv`/`pip`). |
| `sonar_enabled` | `boolean` | `no` | `True` | Liga/desliga as etapas de Sonar (setup Java, truststore, build de args, scan, quality gate, dashboard e summary). **Atenção:** pelo desenho atual, a instalação de deps e os testes/coverage rodam mesmo com `sonar_enabled=false` (porque esses steps não têm `if` de Sonar). |

#### Python
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `python-version` | `string` | `no` | `3.12` | Versão do Python instalada pelo `actions/setup-python` (ex.: `3.12`). Vale tanto para instalação de deps quanto para execução de testes. |

#### Testes e coverage
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `unit_tests_enabled` | `boolean` | `no` | `True` | Se `true`, roda testes e tenta gerar coverage (`sonar_coverage`). Se `false`, pula a execução de testes/coverage; o scan (quando habilitado) roda sem `sonar.python.coverage.reportPaths`. |
| `coverage_required` | `boolean` | `no` | `False` | Se `true` e `unit_tests_enabled=true`, falha se o arquivo `sonar_coverage` não existir ou estiver vazio. Se `false`, emite warning e segue sem coverage. |
| `sonar_coverage` | `string` | `no` | `coverage.xml` | Caminho (relativo ao `context`) do arquivo de coverage em XML (Cobertura), normalmente `coverage.xml`. Usado na verificação e injetado em `sonar.python.coverage.reportPaths` quando encontrado. |

#### Sonar projeto
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_host_url` | `string` | `no` | `${{ vars.SONAR_HOST_URL }}` | URL do SonarQube. Também é usada para extrair o hostname na importação do certificado (fallback via `openssl s_client`). |
| `sonar_project_key` | `string` | `no` | `` | Chave do projeto no Sonar (`sonar.projectKey`). **Obrigatório na prática** quando `sonar_enabled=true` (o workflow falha cedo se vazio). |
| `sonar_project_name` | `string` | `no` | `` | Nome do projeto (`sonar.projectName`). Se vazio, o workflow usa fallback para o nome do repositório (conforme descrito na doc original). |
| `sonar_project_version` | `string` | `no` | `` | Versão (`sonar.projectVersion`). Se vazio, calcula automaticamente: PR → `pr-<num>-<sha7>-rc`; branch → `<sha7>`. |

#### Escopo de análise
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_sources` | `string` | `no` | `.` | Diretórios de código a analisar (`sonar.sources`). Default `.` (relativo ao `projectBaseDir=context`). |
| `sonar_tests` | `string` | `no` | `.` | Diretórios de testes (`sonar.tests`). Default `.`. N/A quando você não quer setar explicitamente (deixe vazio). |
| `sonar_test_inclusions` | `string` | `no` | `` | Glob de testes (`sonar.test.inclusions`). N/A quando vazio. |
| `sonar_exclusions` | `string` | `no` | `` | Exclusões (`sonar.exclusions`). N/A quando vazio. |

#### Quality Gate e extras
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_quality_gate_timeout_sec` | `number` | `no` | `300` | Timeout do Quality Gate em segundos (polling). Default `300`. |
| `sonar_extra_args` | `string` | `no` | `` | Args extras `-D...` anexados literalmente ao final do args do scanner (ex.: `-Dsonar.coverage.exclusions=**/tests/**`). N/A quando vazio. |

---

### Secrets

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `SONAR_TOKEN` | `condicional` | Token de análise do SonarQube. **Obrigatório na prática** quando `sonar_enabled=true` (usado no scan e no quality gate). |
| `SONAR_ROOT_CERT` | `condicional` | CA raiz em PEM para ambientes com certificado interno. Se ausente, o workflow faz fallback via `openssl s_client` para capturar o chain e importar no truststore. |

> Observação: `SONAR_TOKEN` é obrigatório **na prática** quando `sonar_enabled=true`.

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
group: python-sonar-${{ github.repository }}-${{ github.event.pull_request.number || github.ref_name }}
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
| 1 | Checkout 📥 | `N/A` | Faz checkout com histórico completo (normalmente `fetch-depth: 0`) para suportar contexto de PR/branch. |
| 2 | Validate Context 📁 | `N/A` | Valida que o diretório `context` existe antes de instalar deps/rodar testes/scan (se o diretório estiver errado, falha cedo). |
| 3 | Detect Python Tool 🔎 | `N/A` | Detecta automaticamente o gerenciador: `uv` → `poetry` → `pipenv` → fallback `pip`. Define `steps.py.outputs.tool`. |
| 4 | Set Up Python 🐍 | `N/A` | Instala a versão do Python (`python-version`) e habilita cache (quando configurado). |
| 5 | Install Deps (Uv) ⚡ | ``steps.py.outputs.tool == 'uv'`` | Instala `uv` e resolve dependências (sincroniza ambiente). |
| 6 | Sync Env (Uv) ⚡ | ``steps.py.outputs.tool == 'uv'`` | Executa `uv sync` para instalar deps (com base no lockfile). |
| 7 | Install Deps (Poetry) 🎭 | ``steps.py.outputs.tool == 'poetry'`` | Instala Poetry e executa `poetry install` (geralmente com `virtualenvs.create false`). |
| 8 | Install Deps (Pipenv) 📦 | ``steps.py.outputs.tool == 'pipenv'`` | Instala Pipenv e executa `pipenv install --dev --deploy`. |
| 9 | Install Deps (Pip) 📦 | ``steps.py.outputs.tool == 'pip'`` | Instala deps via `requirements.txt` (se existir) ou `pip install .` (pyproject). Se nenhum existir, segue com warning. |
| 10 | Tests & Coverage 🧪 | ``${{ inputs.unit_tests_enabled }}`` | Roda testes e tenta gerar `sonar_coverage`. Preferência: `pytest --cov ... --cov-report=xml`. Fallback: `coverage run -m pytest` / `coverage xml`. |
| 11 | Verify Coverage File ✅ | `N/A` | Verifica se `sonar_coverage` existe e não está vazio. Se `coverage_required=true` e não encontrar, falha antes do scan. |
| 12 | Setup Java ⚒️ | ``${{ inputs.sonar_enabled }}`` | Instala JDK para criar truststore e suportar scanner com `SONAR_SCANNER_OPTS`. |
| 13 | Trust SonarQube Certificate 🔐 | ``${{ inputs.sonar_enabled }}`` | Cria truststore em `/tmp/my_truststore` e exporta `/tmp/sonar-ca.pem` (define `NODE_EXTRA_CA_CERTS`). |
| 14 | Build Sonar args 🧩 | ``${{ inputs.sonar_enabled }}`` | Monta os `-D...` do scanner, incluindo PR vs branch e coverage (quando encontrado). |
| 15 | Sonar Scan 🔍 | ``${{ inputs.sonar_enabled }}`` | Executa `SonarSource/sonarqube-scan-action` com `projectBaseDir=context` e `args` montados no step anterior. |
| 16 | Quality Gate 🧱 | ``${{ inputs.sonar_enabled }}`` | Executa `SonarSource/sonarqube-quality-gate-action` lendo `<context>/.scannerwork/report-task.txt`. |
| 17 | Compute Dashboard URL 🔗 | ``${{ inputs.sonar_enabled }}`` | Monta URL do dashboard e salva em output `sonar_dashboard_url`. |
| 18 | Summary 📄 | ``${{ inputs.sonar_enabled }}`` | Escreve resumo no `GITHUB_STEP_SUMMARY` (key, dashboard e status do quality gate). |

---

## Detalhamento técnico (comportamento do workflow)

### 1) Detect Python Tool 🔎 (auto)
Ordem de detecção descrita no workflow (best-effort):

1. **uv**
   - se existir `uv.lock` **ou**
   - se `pyproject.toml` contiver `[tool.uv]`
2. **poetry**
   - se existir `poetry.lock` **ou**
   - se `pyproject.toml` contiver `[tool.poetry]`
3. **pipenv**
   - se existir `Pipfile`
4. fallback: **pip**

Isso define `steps.py.outputs.tool` e comanda os steps de instalação.

### 2) Install / Sync dependencies
Dependendo do tool detectado:

- **uv**
  - instala `uv` (via action) e executa `uv sync`
  - prefixo de execução vira `uv run`
- **poetry**
  - instala Poetry via `pip`
  - executa `poetry install`
  - prefixo de execução vira `poetry run`
- **pipenv**
  - instala Pipenv via `pip`
  - executa `pipenv install --dev --deploy`
  - prefixo de execução vira `pipenv run`
- **pip**
  - se existir `requirements.txt` → `pip install -r requirements.txt`
  - senão, se existir `pyproject.toml` → `pip install .`
  - senão → warning e segue

### 3) Tests & Coverage 🧪
Executa apenas quando `unit_tests_enabled=true`.

Estratégia (best-effort):
1) Preferência **pytest + pytest-cov**
   `pytest -q --cov=. --cov-report=xml:<sonar_coverage>`

2) Se falhar por `--cov` “unrecognized arguments”:
   - fallback com **coverage.py**:
     - `coverage run -m pytest -q`
     - `coverage xml -o <sonar_coverage>`

3) Se **pytest não existir**:
   - tenta `unittest` + coverage best-effort
   - sempre tenta pelo menos `python -m unittest discover -v`

### 4) Verify Coverage File ✅
- Se `unit_tests_enabled=false` → considera `found=false` (N/A) e não bloqueia.
- Se `sonar_coverage` existir e não estiver vazio → `found=true`.
- Caso contrário:
  - warning
  - se `coverage_required=true` → falha antes do scan.

### 5) Trust SonarQube Certificate 🔐
Cria truststore em `/tmp/my_truststore`:

- copia `cacerts` do Java
- se `SONAR_ROOT_CERT` existir → importa como alias `sonarqube-ca`
- senão → fallback `openssl s_client` para coletar o chain e importar

Também grava `/tmp/sonar-ca.pem` e exporta:
- `NODE_EXTRA_CA_CERTS=/tmp/sonar-ca.pem`
- `SONAR_SCANNER_OPTS` apontando para o truststore

### 6) Build Sonar args 🧩
Monta os `-D...` do scanner:

- sempre:
  - `sonar.projectKey` (**obrigatório na prática**)
  - `sonar.projectName` (fallback para repo quando vazio)
  - `sonar.projectVersion` (auto quando vazio)
  - `sonar.sources`, `sonar.sourceEncoding=UTF-8`
- opcionais:
  - `sonar.tests`, `sonar.test.inclusions`, `sonar.exclusions`
- coverage (quando `unit_tests_enabled=true` e coverage encontrado):
  - `sonar.python.coverage.reportPaths=<path>`
- PR vs branch:
  - PR: `sonar.pullrequest.key/branch/base`
  - branch: `sonar.branch.name=<ref_name>`
- `sonar_extra_args` é anexado literalmente no final.

### 7) Sonar Scan + Quality Gate
- Scan: `SonarSource/sonarqube-scan-action` com `projectBaseDir=context`.
- Quality Gate: `SonarSource/sonarqube-quality-gate-action`, lendo `<context>/.scannerwork/report-task.txt`.

---

## Matriz de comportamento (inputs/evento × resultado)

### A) `sonar_enabled`
| `sonar_enabled` | O que acontece |
|---:|---|
| `false` | Pula: setup Java, truststore, args, scan, quality gate, dashboard e summary. **Ainda pode rodar instalação de deps e testes/coverage**, dependendo de `unit_tests_enabled`. |
| `true` | Executa fluxo completo de análise (com testes/coverage conforme flags). |

### B) `unit_tests_enabled` e coverage
| `unit_tests_enabled` | `coverage_required` | Comportamento |
|---:|---:|---|
| `false` | N/A | Não roda testes/coverage. Scan roda sem `sonar.python.coverage.reportPaths`. |
| `true` | `false` | Tenta gerar coverage; se ausente/vazio, warning e scan segue sem coverage. |
| `true` | `true` | Se ausente/vazio → falha antes do scan. |

### C) PR vs Branch
| Evento | Parâmetros no Sonar | `sonar.projectVersion` (auto) |
|---|---|---|
| `pull_request` / `pull_request_target` | `sonar.pullrequest.key/branch/base` | `pr-<num>-<sha7>-rc` |
| `push` / outros | `sonar.branch.name` | `<sha7>` |

---

## Coerência e gotchas (técnico)

- ✅ Coverage só é injetado no Sonar quando o arquivo existir e não estiver vazio.
- ⚠ Se `sonar_enabled=false`, o job ainda pode instalar deps e rodar testes (dependendo de `unit_tests_enabled`). Se você quer “pular tudo”, use `if` no **caller**.
- ⚠ `sonar_extra_args` é concatenado no final; evite strings com aspas/escapes muito complexos.

---

## FAQ (cenários comuns)

### 1) “Uso Poetry/Uv/Pipenv — preciso mudar algo?”
Não. O workflow detecta automaticamente via `uv.lock`, `poetry.lock`, `Pipfile` ou fallback `pip`.

### 2) “Não tenho testes ainda, quero rodar Sonar mesmo assim”
Use `unit_tests_enabled: false`.

### 3) “Coverage não aparece no Sonar”
O workflow só injeta `sonar.python.coverage.reportPaths` se `sonar_coverage` existir e não estiver vazio.

### 4) “Falha de SSL/certificado no SonarQube”
Configure `secrets.SONAR_ROOT_CERT` (PEM) quando possível.

### 5) “Quero parâmetros extras”
Use `sonar_extra_args` com `-D...` extras.

---

## Exemplo de uso (caller)

```yaml
jobs:
  sonar:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/sonar-python.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      python-version: "3.12"

      unit_tests_enabled: true
      coverage_required: false
      sonar_coverage: "coverage.xml"

      sonar_enabled: true
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}
      sonar_project_key: "mrrdev4_meu-projeto-python"
      sonar_project_name: "meu-projeto-python"

      sonar_quality_gate_timeout_sec: 300
      sonar_extra_args: ""
```

> Se você estiver chamando isso condicionado pelo Guard do ReleaseFlow, use `if: needs.guard.outputs.should_run_sonar == 'true'`.
