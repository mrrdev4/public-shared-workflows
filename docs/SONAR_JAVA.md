# Sonar Scan Java ☕ (Maven/Gradle Plugin) `sonar-java.yml`

Reusable workflow (`workflow_call`) para executar **análise SonarQube em projetos Java**, com suporte a:

- **Maven** via `sonar-maven-plugin`
- **Gradle/Gradle Wrapper** via task `sonarqube`
- Execução opcional de **tests + JaCoCo** (com autodetecção do XML)
- **PR analysis** (`sonar.pullrequest.*`) e **branch analysis** (`sonar.branch.name`)
- **Quality Gate** (polling)
- Suporte a **certificado interno** do Sonar:
  - importa CA no truststore Java (`/tmp/my_truststore`)
  - exporta CA via `NODE_EXTRA_CA_CERTS` (para actions Node, ex.: quality gate)

---

## Quando usar

- Repositórios Java com `pom.xml` ou Gradle (`gradlew` / `build.gradle*`) **na raiz do repo**
- Pipelines onde você quer:
  - rodar build/test (e opcionalmente coverage)
  - publicar análise no Sonar e bloquear via Quality Gate

> **Monorepo (Java em subdiretório):** este workflow **não faz `cd context`** para detectar tool/build/test/scan.
> Do jeito que está hoje, ele assume que `pom.xml` / `gradlew` / `build.gradle*` estão no root. Para subdiretórios, é necessário adaptar o workflow.


## Pré-requisitos do runner

Este workflow assume que o runner possui:

- `bash` (shell padrão)
- `python3` (usado para extrair hostname de `sonar_host_url` e para URL-encode do `projectKey`)
- `openssl` **somente** quando `SONAR_ROOT_CERT` não for informado (fallback `openssl s_client`)
- Utilitários básicos: `awk`, `sed`, `grep`, `chmod`
- `keytool` (vem do JDK instalado pelo `actions/setup-java`)


---

## Contrato do `workflow_call`

### Inputs

#### Execução e diretório
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `runs_on` | `string` | `no` | ["self-hosted","dev"] | Lista JSON de labels do runner. Usado em `runs-on: fromJson(inputs.runs_on)`. Precisa ser JSON válido (ex.: `["self-hosted","dev"]`, `["ubuntu-latest"]`). |
| `context` | `string` | `no` | . | Diretório que deve existir no workspace. **Atenção:** este workflow valida `context`, mas a detecção de tool/build/test/scan ocorre no **root do repo** (não faz `cd context`). Para Java em subdiretório (monorepo), este workflow precisa ser adaptado. |

#### Java / JDK
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `java-version` | `string` | `no` | 21 | Versão do Java usada no `actions/setup-java@v4` (JDK). Default `21`. Deve ser compatível com seu projeto e com o plugin do Sonar. |
| `java-distribution` | `string` | `no` | temurin | Distribuição do JDK para `setup-java` (default `temurin`). |

#### Maven (repositórios privados)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `server-id` | `string` | `no` | "" (vazio) | ID do `<server>` no `settings.xml`. Só é usado quando você quer autenticar em repositório Maven privado (junto com `maven-username` e secret `MAVEN_PASSWORD`). N/A quando não usa repo privado. |
| `maven-username` | `string` | `no` | "" (vazio) | Username para autenticação Maven (configura `server-username` no `setup-java`). N/A quando não usa repo privado. |

#### Build / Tests / Coverage
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `unit_tests_enabled` | `boolean` | `no` | true | Controla execução de testes e geração best-effort do JaCoCo. Se `false`: Maven faz `package` com `-DskipTests=true`; Gradle faz `build -x test`. Ainda assim o Sonar roda (se `sonar_enabled=true`). |
| `coverage_required` | `boolean` | `no` | false | Se `true` e `unit_tests_enabled=true`, falha se não encontrar um JaCoCo XML válido (arquivo existente e não vazio). Se `false`, gera warning e segue. |
| `maven_cmd` | `string` | `no` | mvn -B -ntp -DskipTests=false test | Comando Maven (sem sonar) usado quando `unit_tests_enabled=true`. O workflow também tenta `jacoco:report` (best-effort) após o comando. |
| `gradle_cmd` | `string` | `no` | test | Comando Gradle (sem sonar) usado quando `unit_tests_enabled=true` (ex.: `test`). Se existir a task `jacocoTestReport`, o workflow tenta rodá-la (best-effort). |

#### SonarQube (projeto)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_enabled` | `boolean` | `no` | true | Liga/desliga etapas de Sonar (validação de config, truststore, scan, quality gate, dashboard e summary). Quando `false`, o workflow apenas build/test (Maven/Gradle) e detecção de coverage (se `unit_tests_enabled=true`). |
| `sonar_host_url` | `string` | `no` | ${{ vars.SONAR_HOST_URL }} | URL do SonarQube (ex.: `https://sonar.seu-dominio`). Também é usada para extrair o hostname na importação do certificado. |
| `sonar_project_key` | `string` | `no` | "" (vazio) | Chave do projeto no Sonar (`sonar.projectKey`). Obrigatória quando `sonar_enabled=true`. |
| `sonar_project_name` | `string` | `no` | "" (vazio) | Nome do projeto (`sonar.projectName`). Obrigatório quando `sonar_enabled=true` (não há fallback automático). |

#### SonarQube (escopo e exclusões)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_sources` | `string` | `no` | . | Diretórios de código fonte (`sonar.sources`). Default `.` (relativo ao root do repo). |
| `sonar_tests` | `string` | `no` | "" (vazio) | Diretórios de testes (`sonar.tests`). N/A quando vazio (o workflow não injeta a propriedade). |
| `sonar_test_inclusions` | `string` | `no` | "" (vazio) | Glob de testes (`sonar.test.inclusions`). N/A quando vazio. |
| `sonar_exclusions` | `string` | `no` | "" (vazio) | Exclusões (`sonar.exclusions`). N/A quando vazio. |

#### SonarQube (coverage / quality gate / extras)
| Input | Tipo | Required | Default | Descrição (técnica) |
|---|---|:---:|---|---|
| `sonar_coverage` | `string` | `no` | "" (vazio) | Caminho do JaCoCo XML. Se preenchido, o workflow valida esse arquivo; se vazio, faz autodetecção por paths comuns (Maven/Gradle). Caminho é relativo ao root do repo. |
| `sonar_quality_gate_timeout_sec` | `number` | `no` | 300 | Timeout do Quality Gate (polling) em segundos. Default `300`. |
| `sonar_extra_args` | `string` | `no` | "" (vazio) | Args extras `-D...` anexados ao final do comando do Sonar. **Parsing é por split simples em espaço**; evite aspas complexas. N/A quando vazio. |
| `sonar_maven_plugin_version` | `string` | `no` | 5.5.0.6356 | Versão do `sonar-maven-plugin` usada no goal `...:<ver>:sonar`. Mantida explícita para evitar warnings e garantir reprodutibilidade. |

---

### Secrets

| Secret | Required | Descrição (técnica) |
|---|:---:|---|
| `SONAR_TOKEN` | `yes` | Token de análise do SonarQube. **Obrigatório** quando `sonar_enabled=true` (usado no scan Maven/Gradle e no Quality Gate). |
| `SONAR_ROOT_CERT` | `condicional` | CA raiz (PEM) para SonarQube em ambientes com certificado interno. Se não informado, o workflow faz fallback com `openssl s_client` e importa o chain no truststore. |
| `MAVEN_PASSWORD` | `condicional` | Senha/token para o `server-id` Maven (repos privados). Só é usado quando `server-id` e `maven-username` estão preenchidos (o `setup-java` monta `settings.xml`). |

**Notas**
- `SONAR_TOKEN` é obrigatório quando `sonar_enabled=true`.
- `MAVEN_PASSWORD` só é relevante quando você usa `server-id` + `maven-username`.

---

### Outputs

| Output | Descrição (técnica) |
|---|---|
| `sonar_dashboard_url` | URL do dashboard no SonarQube: `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`. N/A quando `sonar_enabled=false`. |
| `quality_gate_status` | Status retornado pelo Quality Gate (`OK`, `ERROR`, etc.). N/A quando `sonar_enabled=false`. |

---

## Permissões, concorrência e defaults

### Permissions
```yaml
contents: read
pull-requests: read
```

### Concurrency
```yaml
group: java-sonar-${{ github.repository }}-${{ github.event.pull_request.number ||
  github.ref_name }}
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
| 3 | Validate Sonar Config 🧾 | `${{ inputs.sonar_enabled }}` | Script bash. |
| 4 | Detect Build Tool 🔎 | `N/A` | Outputs via `id: bt`. Script bash. |
| 5 | Decide Maven Credentials ✅ | `N/A` | Outputs via `id: mvn_creds`. Script bash. |
| 6 | Set Up Java ☕ (com settings.xml) | `${{ steps.mvn_creds.outputs.use == 'true' }}` | Action: `actions/setup-java@v4`. |
| 7 | Set Up Java ☕ | `${{ steps.mvn_creds.outputs.use != 'true' }}` | Action: `actions/setup-java@v4`. |
| 8 | Resolve Maven Bin 🧰 | `${{ steps.bt.outputs.tool == 'maven' }}` | Outputs via `id: mvn`. Script bash. |
| 9 | Install Maven 🗃️ | `${{ steps.bt.outputs.tool == 'maven' && steps.mvn.outputs.need_setup == 'true' }}` | Action: `stCarolas/setup-maven@v5`. |
| 10 | Resolve Gradle Bin 🧰 | `${{ steps.bt.outputs.tool == 'gradle' || steps.bt.outputs.tool == 'gradle-wrapper' }}` | Outputs via `id: grd`. Script bash. |
| 11 | Install Gradle ⚡ | `${{ steps.bt.outputs.tool == 'gradle' && steps.grd.outputs.need_setup == 'true' }}` | Action: `gradle/actions/setup-gradle@v5`. |
| 12 | Trust SonarQube Certificate 🔐 | `${{ inputs.sonar_enabled }}` | Script bash. |
| 13 | Compute Project Version/Tag 🏷️ | `N/A` | Outputs via `id: tag`. Script bash. |
| 14 | Build/Test Maven 🧪 | `${{ steps.bt.outputs.tool == 'maven' }}` | Script bash. |
| 15 | Build/Test Gradle 🧪 | `${{ steps.bt.outputs.tool == 'gradle' || steps.bt.outputs.tool == 'gradle-wrapper' }}` | Script bash. |
| 16 | Detect Coverage Path 🔎 | `${{ inputs.unit_tests_enabled }}` | Outputs via `id: cov`. Script bash. |
| 17 | SonarQube Scan (Maven Plugin) 🔍 | `${{ inputs.sonar_enabled && steps.bt.outputs.tool == 'maven' }}` | Script bash. |
| 18 | SonarQube Scan (Gradle Task) 🔍 | `${{ inputs.sonar_enabled && (steps.bt.outputs.tool == 'gradle' || steps.bt.outputs.tool == 'gradle-wrapper') }}` | Script bash. |
| 19 | Locate report-task.txt 📄 | `${{ inputs.sonar_enabled }}` | Outputs via `id: report`. Script bash. |
| 20 | Quality Gate 🧱 | `${{ inputs.sonar_enabled }}` | Action: `SonarSource/sonarqube-quality-gate-action@v1.2.0`. Outputs via `id: quality_gate`. |
| 21 | Compute Dashboard URL 🔗 | `${{ inputs.sonar_enabled }}` | Outputs via `id: dashboard`. Script bash. |
| 22 | Summary 📄 | `${{ inputs.sonar_enabled }}` | Script bash. |

---

## Detalhamento técnico por etapa

### 1) Checkout 📥
- `actions/checkout@v4` com `fetch-depth: 0` (histórico completo ajuda PR/branch e versionamento do scan).

### 2) Validate Context 📁
- Verifica se o diretório `inputs.context` existe: `test -d "<context>"`.
- **Observação:** esta etapa não muda o diretório de execução; ela só valida a existência.

### 3) Validate Sonar Config 🧾 (quando `sonar_enabled=true`)
Falha cedo se algum estiver vazio:
- `sonar_host_url`
- `sonar_project_key`
- `sonar_project_name`

### 4) Detect Build Tool 🔎
Detecta **no root do repo** (não usa `context`):
- Maven → existe `pom.xml`
- Gradle Wrapper → existe `gradlew`
- Gradle (sem wrapper) → existe `build.gradle*`
- Se não encontrar nenhum → falha com erro.

### 5) Decide Maven Credentials ✅
Decide se deve escrever credenciais Maven no `settings.xml`:
- `use=true` apenas quando:
  - `inputs.server-id` **e**
  - `inputs.maven-username` **e**
  - `secrets.MAVEN_PASSWORD`
  estão preenchidos.

### 6–7) Set Up Java ☕ (+ cache Maven/Gradle)
- `actions/setup-java@v4` com:
  - `distribution` / `java-version`
  - cache automático: Maven quando tool = Maven, senão Gradle
- Quando `use=true`, também configura `server-id/server-username/server-password`, gerando `settings.xml` para repositórios privados.

### 8–11) Resolve/Install Maven/Gradle bins 🧰
**Maven**
- Prioridade: `./mvnw` → `mvn` no PATH → instala Maven e usa `mvn`.

**Gradle**
- Prioridade: `./gradlew` → `gradle` no PATH → instala Gradle e usa `gradle`.

### 12) Trust SonarQube Certificate 🔐 (quando `sonar_enabled=true`)
Cria truststore em `/tmp/my_truststore`:
- copia `"$JAVA_HOME/lib/security/cacerts"`
- se `SONAR_ROOT_CERT` existir → importa como alias `sonarqube-ca`
- senão → fallback `openssl s_client`, extrai chain e importa múltiplos certificados com aliases `sonarqube-<host>-NN`

Exporta:
- `NODE_EXTRA_CA_CERTS=/tmp/sonar-ca.pem`
- `JAVA_TOOL_OPTIONS`, `MAVEN_OPTS`, `GRADLE_OPTS` com `-Djavax.net.ssl.trustStore=...`

### 13) Compute Project Version/Tag 🏷️
Define `sonar.projectVersion`:
- PR (`pull_request`) → `pr-<PR_NUM>-<SHA7>-rc`
- Branch → `<SHA7>`

### 14–15) Build/Test (Maven/Gradle) 🧪
**Maven**
- Se `unit_tests_enabled=true`: roda `inputs.maven_cmd` (default `mvn ... test`)
- Se `unit_tests_enabled=false`: força `mvn ... -DskipTests=true package`
- Se testes habilitados: tenta `jacoco:report || true` (best-effort)

**Gradle**
- Se `unit_tests_enabled=true`: roda `gradle_cmd` (default `test`)
  - se existir task `jacocoTestReport`, tenta rodar `jacocoTestReport || true`
- Se `unit_tests_enabled=false`: roda `build -x test`

### 16) Detect Coverage Path 🔎 (quando `unit_tests_enabled=true`)
Procura JaCoCo XML:
- se `inputs.sonar_coverage` estiver preenchido, valida esse arquivo
- senão, tenta candidatos comuns:
  - `target/site/jacoco/jacoco.xml`
  - `target/site/jacoco-it/jacoco.xml`
  - `build/reports/jacoco/test/jacocoTestReport.xml`
  - `build/reports/jacoco/test/jacoco.xml`
  - `build/reports/jacoco/jacoco.xml`

Se não encontrar:
- emite warning
- se `coverage_required=true` → falha

### 17–18) Sonar Scan 🔍
**Maven Plugin**
- Executa:
  - `org.sonarsource.scanner.maven:sonar-maven-plugin:<ver>:sonar`
- Injeta `-D...`:
  - host/token
  - projectKey/name/version
  - sources
  - opcionais: tests/test_inclusions/exclusions
  - coverage (quando `unit_tests_enabled=true` e coverage encontrado): `sonar.coverage.jacoco.xmlReportPaths=...`
  - PR vs branch:
    - PR: `sonar.pullrequest.*`
    - branch: `sonar.branch.name=<ref_name>`
  - extras: `sonar_extra_args` (split simples por espaço)

**Gradle Task**
- Verifica se existe task `sonarqube`:
  - se não existir, falha pedindo plugin `org.sonarqube`
- Executa `gradle sonarqube` com os mesmos `-D...`

### 19) Locate report-task.txt 📄 (quando `sonar_enabled=true`)
Procura metadata do scan em:
- `.scannerwork/report-task.txt`
- `target/sonar/report-task.txt`
- `build/sonar/report-task.txt`

Se não encontrar → falha (quality gate não consegue rodar).

### 20) Quality Gate 🧱 (quando `sonar_enabled=true`)
- `SonarSource/sonarqube-quality-gate-action@v1.2.0`
- `pollingTimeoutSec`: `sonar_quality_gate_timeout_sec`
- `scanMetadataReportFile`: `${ inputs.context }/${ steps.report.outputs.path }`

> **Atenção:** o `steps.report.outputs.path` é encontrado relativo ao root do repo, mas o action recebe prefixo de `context`.
> Para `context="."` isso é OK. Para subdiretório, pode ficar inconsistente.

### 21) Compute Dashboard URL 🔗
Monta:
- `<sonar_host_url>/dashboard?id=<urlencoded(projectKey)>`

### 22) Summary 📄
Escreve em `GITHUB_STEP_SUMMARY`:
- project key
- version/tag
- dashboard URL
- quality gate status

---

## Matriz de comportamento (evento × tool × flags)

### 1) PR vs Branch
| Evento | `sonar.projectVersion` | Modo | Propriedades |
|---|---|---|---|
| `pull_request` | `pr-<num>-<sha7>-rc` | PR analysis | `sonar.pullrequest.key/branch/base` |
| outros (`push`, `workflow_dispatch`) | `<sha7>` | Branch analysis | `sonar.branch.name=<ref_name>` |

### 2) Maven vs Gradle
| Tool detectado | Como roda o scan | Requisito |
|---|---|---|
| `maven` | `sonar-maven-plugin:<ver>:sonar` | `pom.xml` no root |
| `gradle-wrapper` / `gradle` | `gradle sonarqube` | task `sonarqube` (plugin `org.sonarqube`) |

### 3) Tests/Coverage
| `unit_tests_enabled` | `coverage_required` | Resultado |
|---:|---:|---|
| `false` | N/A | Compila sem testes; scan roda sem `sonar.coverage.jacoco.xmlReportPaths` |
| `true` | `false` | Tenta achar JaCoCo; se não achar, warning e segue |
| `true` | `true` | Se não achar JaCoCo, falha antes do scan |

---

## Coerência (validações reais + gotchas)

- ✅ `sonar_*` obrigatórios são validados quando `sonar_enabled=true`.
- ✅ Repositório Maven privado só é configurado se **todos** (`server-id`, `maven-username`, `MAVEN_PASSWORD`) estiverem presentes.
- ⚠ Detecção de tool/build/test/scan ocorre no **root do repo** (não usa `context`).
- ⚠ `sonar_extra_args` faz split simples por espaço; evite argumentos com aspas/escapes complexos.

---

## FAQ (cenários comuns)

### 1) `mvn: command not found`
O workflow tenta `./mvnw`; se não existir, tenta `mvn` no PATH; se não tiver, instala Maven.

### 2) Gradle sem task `sonarqube`
O workflow falha de propósito.
✅ Solução: aplicar o plugin `org.sonarqube` no `build.gradle(.kts)`.

### 3) JaCoCo XML não encontrado
- Se `coverage_required=false`: warning e segue.
- Se `coverage_required=true`: falha.

✅ Solução: habilitar geração XML do JaCoCo e/ou setar `sonar_coverage` com o caminho exato.

### 4) Erros SSL/PKIX
✅ Use `SONAR_ROOT_CERT` (PEM) sempre que possível para evitar dependência do `openssl`.

---

## Exemplo de uso (caller)

### Maven (padrão)

```yaml
jobs:
  sonar:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/sonar-java.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      java-version: "21"
      java-distribution: "temurin"

      unit_tests_enabled: true
      coverage_required: false

      sonar_enabled: true
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}
      sonar_project_key: "mrrdev4_meu-projeto-java"
      sonar_project_name: "meu-projeto-java"

      sonar_quality_gate_timeout_sec: 300
      sonar_extra_args: ""
```

### Maven com repositório privado (settings.xml)

```yaml
jobs:
  sonar:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/sonar-java.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      server-id: "nexus"
      maven-username: "ci-user"

      sonar_enabled: true
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}
      sonar_project_key: "mrrdev4_meu-projeto-java"
      sonar_project_name: "meu-projeto-java"
```

### Gradle

```yaml
jobs:
  sonar:
    uses: mrrdev4/publlic-shared-workflows/.github/workflows/sonar-java.yml@<ref>
    secrets: inherit
    with:
      runs_on: '["self-hosted","dev"]'
      context: "."
      java-version: "21"
      gradle_cmd: "test"

      sonar_enabled: true
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}
      sonar_project_key: "mrrdev4_meu-projeto-gradle"
      sonar_project_name: "meu-projeto-gradle"
```

---

## Checklist rápido

- [ ] `sonar_project_key` e `sonar_project_name` definidos
- [ ] `sonar_host_url` (ou `vars.SONAR_HOST_URL`) definido
- [ ] `SONAR_TOKEN` disponível via secrets
- [ ] Maven: `pom.xml` no root (ou adapte o workflow)
- [ ] Gradle: plugin `org.sonarqube` e task `sonarqube` disponível
- [ ] Se exigir coverage: `coverage_required=true` e JaCoCo XML gerado
