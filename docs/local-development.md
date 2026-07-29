# Desenvolvimento Local

## Pré-requisitos

- Java 21 (JDK)
- Git

## 1. Build

```bash
# Build completo (sem testes, mais rápido)
./gradlew :sonar-application:build -x test
```

O zip da distribuição sai em:
```
sonar-application/build/distributions/sonarqube-26.8-SNAPSHOT.zip
```

## 2. Executar

### Com script automático (recomendado)

```bash
./scripts/start.sh -e community -l all
```

Faz tudo: verifica build, descompacta o zip e inicia o servidor com tail dos logs.

### Com debug remoto (JDWP porta 5005)

```bash
./debug-server.sh
```

Conecte sua IDE à porta `5005` para breakpoints.

### Manualmente

```bash
cd sonar-application/build/distributions/
unzip sonarqube-26.8-SNAPSHOT.zip -d .
cd sonarqube-26.8-SNAPSHOT/

# Iniciar
bin/linux-x86-64/sonar.sh start

# Parar
bin/linux-x86-64/sonar.sh stop
```

## 3. Parar

```bash
./stop.sh
```

## 4. Banco de dados

A Community Edition usa **H2 embutido** por padrão — não requer PostgreSQL.

Dados ficam em: `sonar-application/build/distributions/sonarqube-26.8-SNAPSHOT/data/`

Se precisar de PostgreSQL, configure `sonar.jdbc.url`, `sonar.jdbc.username` e `sonar.jdbc.password` no `conf/sonar.properties` da distribuição descompactada.

## 5. IDE (IntelliJ)

```bash
./gradlew ide
```

Depois abra `build.gradle` como projeto no IntelliJ.

## 6. Acesso

- URL: `http://localhost:9000`
- Login: `admin` / `admin`

## 7. Logs

```bash
# Todos os processos
./scripts/logs.sh

# Só Elasticsearch
tail -f sonar-application/build/distributions/sonarqube-*/logs/es.log

# Só Web Server
tail -f sonar-application/build/distributions/sonarqube-*/logs/web.log

# Só Compute Engine
tail -f sonar-application/build/distributions/sonarqube-*/logs/ce.log
```
