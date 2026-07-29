# SonarQube — Visão Geral do Projeto

> Build customizado: `26.8-SNAPSHOT` Community Edition a partir do source.

## 1. Estrutura de Diretórios

| Diretório | Propósito |
|---|---|
| `server/` | Núcleo do servidor: ~32 submódulos (Web, CE, DB, ES, Auth, etc.) |
| `sonar-application/` | Empacotamento da distribuição final (zip, shadowJar, ES embutido) |
| `sonar-core/` | Classes compartilhadas entre Web e CE (config, plugin, i18n, etc.) |
| `sonar-plugin-api-impl/` | Implementação da API de plugin (carregamento de classes) |
| `sonar-ws/` | DTOs para Web Services |
| `sonar-ws-generator/` | Gerador de código para clientes WS |
| `sonar-duplications/` | Detecção de duplicações (CPD-based) |
| `sonar-markdown/` | Motor de renderização Markdown |
| `sonar-sarif/` | Suporte ao formato SARIF |
| `sonar-shutdowner/` | Utilitário de parada do processo |
| `sonar-testing-harness/` | Infraestrutura de testes |
| `sonar-testing-ldap/` | Servidor LDAP embutido para testes |
| `plugins/` | Plugins de exemplo/teste (`sonar-xoo-plugin`, `sonar-education-plugin`) |
| `buildSrc/` | Scripts de build customizados do Gradle |
| `deploy/` | Configurações de deploy (Coolify) |
| `docs/` | Documentação |
| `scripts/` | Scripts auxiliares (start, stop, logs) |
| `private/` | (gitignored) Módulos enterprise fechados |

## 2. Módulos Gradle

### `server/` — 32 submódulos

**Processos:**
- `sonar-main` — Orquestrador de processos (`SchedulerImpl`)
- `sonar-process` — IPC entre processos (shared memory)
- `sonar-shutdowner` — Parada do processo

**Web Server:**
- `sonar-webserver` — Entry point, Tomcat embarcado
- `sonar-webserver-api` — API interna
- `sonar-webserver-auth` — Autenticação
- `sonar-webserver-common` — Componentes compartilhados
- `sonar-webserver-core` — Núcleo (platform, startup)
- `sonar-webserver-es` — Elasticsearch
- `sonar-webserver-webapi` — API REST (`/api/*`)
- `sonar-webserver-webapi-v2` — API REST v2 (`/api/v2/*`)
- `sonar-webserver-pushapi` — Server-Sent Events
- `sonar-webserver-ws` — Web Services
- `sonar-webserver-monitoring` — Métricas de monitoramento

**Compute Engine:**
- `sonar-ce` — Entry point e orquestrador
- `sonar-ce-common` — Classes comuns
- `sonar-ce-task` — Definição de tarefas
- `sonar-ce-task-projectanalysis` — Steps da análise de projeto

**Banco de Dados:**
- `sonar-db-dao` — Camada de acesso (MyBatis)
- `sonar-db-migration` — Migrações versionadas do schema
- `sonar-db-profiling` — Profiling de queries

**Autenticação:**
- `sonar-auth-common`, `sonar-auth-bitbucket`, `sonar-auth-github`, `sonar-auth-gitlab`, `sonar-auth-ldap`, `sonar-auth-saml`

**Outros:**
- `sonar-server-common`, `sonar-statemachine`, `sonar-telemetry`, `sonar-telemetry-core`, `sonar-alm-client`, `sonar-events`

### Top-level (13 módulos)

`sonar-application`, `sonar-core`, `sonar-sarif`, `sonar-duplications`, `sonar-markdown`, `sonar-plugin-api-impl`, `sonar-testing-harness`, `sonar-testing-ldap`, `sonar-ws`, `sonar-ws-generator`

## 3. Arquitetura de Processos

O SonarQube roda como **4 processos Java separados**, gerenciados pelo `App.java`:

```
App.java (processo pai)
 │
 ├── Processo ELASTICSEARCH
 │   └── Embedded ES 9.4.3 (porta 9001)
 │
 ├── Processo WEB_SERVER
 │   └── Tomcat embarcado (porta 9000)
 │       ├── /api/*      → WebServiceEngine (JAX-RS-like)
 │       ├── /api/v2/*   → Spring MVC
 │       └── /            → Frontend React
 │
 └── Processo COMPUTE_ENGINE
     └── Processamento assíncrono de tarefas (análises, sync, purge)
```

### Fluxo de inicialização

1. `SchedulerImpl.schedule()` cria `ManagedProcessHandler` para cada processo
2. **ES** inicia primeiro
3. **Web** só inicia quando ES está operacional (`isEsOperational()`)
4. **CE** só inicia quando Web E ES estão operacionais
5. **Parada**: ordem inversa — CE → Web → ES

### IPC (Comunicação entre Processos)

Feita via **memória compartilhada** (arquivos mapeados). Cada processo monitora comandos do pai (`askedForStop()`, `askedForHardStop()`) e reporta seu estado (`OPERATIONAL`, `STOPPED`).

### Cluster (opcional)

Em modo cluster, usa **Hazelcast** para eleição de líder Web e estado compartilhado.

## 4. PlatformImpl — Inicialização em Camadas

O Web Server usa um contêiner Spring hierárquico com 4 níveis + SafeMode:

| Nível | Propósito |
|---|---|
| **Level 1** | Infraestrutura básica: DB, ES, DAOs |
| **Level 2** | Plugins e migração de banco |
| **SafeMode** | Modo seguro (DB precisa de migração, WS limitados) |
| **Level 3** | Componentes dependentes de plugins |
| **Level 4** | Sistema completo: Quality Profiles, Rules, Issues, Measures, Gates, Permissions, Webhooks, Notifications, ALM, Telemetry |
| **Startup** | Tarefas de inicialização: registrar métricas, perfis, gates, plugins, indexar ES |

## 5. Compute Engine

Processa **tarefas assíncronas** da fila `ce_queue`:

| Tipo de Tarefa | Descrição |
|---|---|
| `REPORT` | Análise de projeto (scanner → métricas, issues, duplicações, quality gate, indexação ES) |
| `ISSUE_SYNC` | Sincronização de issues |
| `AUDIT_PURGE` | Limpeza de auditoria |
| `HISTORY_PURGE` | Limpeza de histórico |
| `PROJECT_EXPORT` | Exportação de projetos |

O `CeProcessingSchedulerImpl` faz polling da fila, `CeWorkerImpl` executa via `CeTaskProcessorRepository`.

## 6. Elasticsearch

Usado como **índice de busca secundário** (dados mestres no BD relacional):

| Índice | Classe |
|---|---|
| `issues` | `IssueIndex` |
| `rules` | `RuleIndex` |
| `components` | `ComponentIndex` |
| `projectmeasures` | `ProjectMeasuresIndex` |
| `metadatas` | `MetadataIndex` |
| `views` | `ViewIndex` |
| `activerules` | `ActiveRuleIndexer` |
| `permissions` | `PermissionIndexer` |

`IndexerStartupTask` sincroniza dados do BD para ES no startup.

## 7. Banco de Dados

**Camada de acesso**: MyBatis via `DbClient` (fachada central para todos os DAOs).

**Suporte**: PostgreSQL (recomendado), H2, Oracle, MSSQL.

**Schema DDL**: `server/sonar-db-dao/src/schema/schema-sq.ddl`

**Migrações**: `server/sonar-db-migration/` — cada versão tem seu pacote (`version/v202605/`, `version/v202502/`, etc.). `AutoDbMigration` roda automaticamente no startup se necessário.

## 8. Sistema de Plugins

- **API**: dependência externa (`org.sonarsource.api.plugin:sonar-plugin-api`)
- **Implementação**: `sonar-plugin-api-impl`
- **ClassLoaders isolados** por plugin (`PluginClassLoader`)
- **Extensões** registradas no Spring container via `ServerExtensionInstaller`
- **19 plugins empacotados**: Java, C#, JavaScript/TypeScript, Python, PHP, Go, Ruby, Kotlin, Scala, Rust, Flex, HTML, XML, IaC, Text, Jacoco, Clean as You Code, VB.NET

## 9. Principais Serviços (Web API)

Organizados em `server/sonar-webserver-webapi`:

- **authentication** — Login, JWT, OAuth2, SSO, SAML
- **issues** — CRUD de issues, transições, comentários, tags
- **rules** — Regras de qualidade, repositórios
- **qualityprofile** — Perfis de qualidade, herança, cópia
- **qualitygate** — Quality gates, condições
- **measures** — Métricas de projeto
- **components** — Projetos, módulos, diretórios
- **projects** — CRUD de projetos
- **permissions** — Permissões de usuários e grupos
- **user** — CRUD de usuários
- **usertoken** — Tokens de usuário
- **notifications** — Notificações por email
- **webhook** — Webhooks de projeto
- **ce** — Monitoramento de tarefas do CE
- **duplication** — Dados de duplicações
- **source** — Código fonte (syntax highlighting)
- **badge** — Badges (quality gate status)
- **hotspot** — Security hotspots
- **newcodeperiod** — New Code Period
- **almintegration** — Integração ALM (GitHub, GitLab, Azure, Bitbucket)
- **telemetry** — Estatísticas anônimas
- **plugins** — Gestão de plugins
- **projectdump** — Exportação de projetos
- **health** — Health check do node
- **branch** — Branches e Pull Requests

## 10. Distribuição Final (Zip)

Gerada pelo `Zip` task em `sonar-application/build.gradle`:

```
sonarqube-{version}/
├── bin/              # Scripts de inicialização (linux, mac, windows)
├── conf/             # sonar.properties
├── lib/              # shadowJar + scanner engine + shutdowner
├── lib/extensions/   # 19 plugins de linguagem
├── lib/jdbc/         # JDBC drivers (postgresql, mssql, h2)
├── web/              # Frontend React
├── elasticsearch/    # ES 9.4.3 (módulos X-Pack removidos)
├── jres/             # JREs por plataforma
└── licenses/         # Relatório de licenças
```

O **shadowJar** combina `sonar-main`, `sonar-ce`, `sonar-webserver`, `sonar-core`, `sonar-plugin-api-impl`, `sonar-process` em um uber-jar com `Main-Class: org.sonar.application.App`.

## 11. Mudança Recente: SONAR-31026

Adiciona tabelas para **SCA TTR (Time To Resolution) History**:

### `SCA_ISSUE_DIMENSIONS`
Tabela de referência/dimensão com combinações únicas de severidade + status identificadas por hash.

### `SCA_TTR_HISTORY`
Histórico de tempo até resolução para issues SCA, agrupado por entidade (projeto/portfolio) e dimensão. Permite análise temporal via `RECORDED_AT_EPOCH`.

Migrações em `server/sonar-db-migration/.../version/v202605/`.

## 12. Configurações Relevantes

| Propriedade | Padrão | Descrição |
|---|---|---|
| `sonar.jdbc.url` | `jdbc:h2:tcp://...` | URL do banco |
| `sonar.web.port` | `9000` | Porta do Web Server |
| `sonar.search.port` | `9001` | Porta do ES |
| `sonar.es.bootstrap.checks.disable` | `false` (true se H2) | Desabilita bootstrap checks do ES |
| `sonar.path.data` | `data/` | Diretório de dados |
| `sonar.path.logs` | `logs/` | Diretório de logs |
