# CI/CD Infrastructure Passport

> **Версия:** 1.0.0 | **Обновлено:** 2025-12-21

## Краткое описание

Полная документация CI/CD инфраструктуры платформы Vondi: GitHub Actions workflows, self-hosted runner, pipelines для микросервисов, тестирование, деплой, Harbor registry.

**Основа:** Self-hosted runner на svetu.rs для избежания лимитов GitHub Actions.

---

## Оглавление

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Self-Hosted Runner](#self-hosted-runner)
3. [GitHub Actions Workflows](#github-actions-workflows)
4. [Pipelines по сервисам](#pipelines-по-сервисам)
5. [Тестирование](#тестирование)
6. [Docker Build & Push](#docker-build--push)
7. [Deployment Process](#deployment-process)
8. [Harbor Registry](#harbor-registry)
9. [Secrets Management](#secrets-management)
10. [Мониторинг CI/CD](#мониторинг-cicd)
11. [Troubleshooting](#troubleshooting)

---

## Обзор архитектуры

### CI/CD Flow

```
Developer → GitHub Push/PR
              ↓
         GitHub Actions
              ↓
    Self-Hosted Runner (svetu.rs)
              ↓
    ┌─────────┼─────────┐
    │         │         │
  Lint    Tests    Build
    │         │         │
    └─────────┼─────────┘
              ↓
       Harbor Registry
              ↓
     Production Deploy
```

### Основные компоненты

| Компонент | Назначение | Расположение |
|-----------|-----------|--------------|
| **GitHub Actions** | Оркестрация CI/CD | GitHub Cloud |
| **Self-Hosted Runner** | Выполнение jobs | svetu.rs:2293673 |
| **Harbor Registry** | Docker images | harbor.svetu.rs |
| **SSH Deploy** | Production deployment | vondi.rs / svetu.rs |

### Преимущества self-hosted

1. ✅ **Нет лимитов времени** - GitHub Actions имеет лимит 2000 минут/месяц
2. ✅ **Локальные ресурсы** - доступ к внутренним БД, Redis, OpenSearch
3. ✅ **Кэширование** - Go modules, yarn cache на диске runner
4. ✅ **Скорость** - прямой доступ к Harbor registry
5. ✅ **Приватные модули** - доступ к `github.com/vondi-global/*` через токен

---

## Self-Hosted Runner

### Установка и конфигурация

**Сервер:** svetu.rs (root@svetu.rs)
**Директория:** `/opt/github-runner`
**Сервис:** `actions.runner.vondi-global.svetu-runner.service`

### Характеристики runner

```bash
# Проверка статуса
ssh root@svetu.rs "systemctl status actions.runner.vondi-global.svetu-runner"

# Вывод:
# Active: active (running)
# Main PID: 2293673
# Memory: 337.8M (peak: 6.4G)
# CPU: 9h 2min 12s
```

**Labels:**
- `self-hosted` (read-only)
- `Linux` (read-only)
- `X64` (read-only)
- `svetu` (custom)

**Version:** 2.330.0
**Node Version:** node20

### Управление runner

```bash
# Проверить статус runner через GitHub API
gh api orgs/vondi-global/actions/runners --jq '.runners[] | {name, status, busy}'
# Output: {"name":"svetu-runner","status":"online","busy":false}

# Перезапустить runner
ssh root@svetu.rs "systemctl restart actions.runner.vondi-global.svetu-runner"

# Логи runner (последние 10 минут)
ssh root@svetu.rs "journalctl -u actions.runner.vondi-global.svetu-runner --since '10 min ago'"

# Следить за логами в реальном времени
ssh root@svetu.rs "journalctl -u actions.runner.vondi-global.svetu-runner -f"
```

### Автозапуск

Runner настроен как systemd service и запускается автоматически при загрузке сервера.

```bash
# Проверить автозапуск
ssh root@svetu.rs "systemctl is-enabled actions.runner.vondi-global.svetu-runner"
# Output: enabled
```

### Директории и кэш

```bash
# Рабочая директория runner
/opt/github-runner/

# Кэш Go modules (переиспользуется между jobs)
~/.cache/go-build/
~/go/pkg/mod/

# Кэш Yarn (переиспользуется между jobs)
~/.cache/yarn/
```

**Важно:** Runner НЕ очищает workspace между jobs - экономия времени на `go mod download`.

---

## GitHub Actions Workflows

### Структура workflows по проектам

| Проект | Workflows | Trigger |
|--------|-----------|---------|
| **vondi** (monolith) | backend-build.yml, frontend-build.yml, backend-deploy.yml, frontend-deploy.yml | workflow_dispatch |
| **auth** | ci.yml | push/PR to main |
| **listings** | orders-service-ci.yml | push/PR to main/develop |
| **delivery** | ci.yml | push/PR to main |
| **warehouse** | ci.yml | push/PR to main |
| **lib** | ci.yml | push/PR to main |

### Типы triggers

**1. Push to main/develop** - автоматический запуск CI
```yaml
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
```

**2. Workflow Dispatch** - ручной запуск (monolith)
```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: false
        default: 'latest'
```

### Common Actions используемые везде

| Action | Version | Назначение |
|--------|---------|------------|
| `actions/checkout@v4` | v4 | Клонирование репозитория |
| `actions/setup-go@v5` | v5 | Установка Go (1.23 / 1.25) |
| `actions/setup-node@v4` | v4 | Установка Node.js 20 |
| `actions/cache@v4` | v4 | Кэширование зависимостей |
| `docker/login-action@v3` | v3 | Login в Docker registry |
| `docker/setup-buildx-action@v3` | v3 | Setup Docker Buildx |
| `docker/build-push-action@v6` | v6 | Build и push Docker images |
| `golangci/golangci-lint-action@v7` | v7 | Lint Go кода (НЕ используется - manual install) |

**Note:** golangci-lint устанавливается вручную через curl для v2.7.2 (поддержка Go 1.25).

---

## Pipelines по сервисам

### Monolith (vondi)

**Workflows:**
- `backend-build.yml` - сборка backend Docker image
- `frontend-build.yml` - сборка frontend Docker image
- `backend-deploy.yml` - деплой backend (blue-green)
- `frontend-deploy.yml` - деплой frontend (blue-green)

**Trigger:** Ручной запуск (`workflow_dispatch`)

**Runner:** `[self-hosted, svetu]`

**Steps:**
1. Checkout code
2. Set timestamp (`YYYYMMDD_HHMMSS`)
3. Login to Harbor registry
4. Setup Docker Buildx
5. Build and push image with tags:
   - `{registry}/svetu/backend/api:{timestamp}`
   - `{registry}/svetu/backend/api:latest`

**Build Args:**
```yaml
build-args: |
  GIT_COMMIT=${{ github.sha }}
  BUILD_TIME=${{ env.TIMESTAMP }}
```

**Deploy Steps (backend-deploy.yml):**
1. Setup SSH key
2. Add host to known_hosts
3. Copy `blue_green_backend.sh` to `/tmp/`
4. Execute script on server
5. Deployment notification

### Auth Service

**Workflow:** `ci.yml`

**Trigger:** Push/PR to main

**Runner:** `self-hosted` (БЕЗ label `svetu` - работает на любом)

**Jobs:**

**1. CI Job**
- Checkout code
- **ВАЖНО:** Check no `replace` directives in go.mod
- Configure private repo access (GO_MODULES_TOKEN)
- Setup Go 1.25
- Setup Node.js 20
- Install frontend deps (`examples/tasktracker/frontend`)
- Check frontend formatting (`yarn format:check`)
- Lint frontend (`yarn lint`)
- Code quality checks (`make check`)
- Run integration tests (`make test-integration`)
- Build application (`make build`)
- Upload binary artifact (retention: 7 days)

**Environment:**
```yaml
env:
  GOPRIVATE: github.com/vondi-global/*
  TESTCONTAINERS_RYUK_DISABLED: true
```

**Note:** Auth service включает проверку frontend компонентов (TaskTracker example).

### Listings Microservice

**Workflow:** `orders-service-ci.yml`

**Trigger:** Push/PR to main/develop/feature/orders-*

**Runner:** `ubuntu-latest` (GitHub hosted!)

**Jobs:**

**1. Lint**
- Setup Go 1.23
- Configure git for private modules
- Download dependencies
- Run `golangci-lint` v2.7.2 (timeout: 5m)

**2. Unit Tests**
- Services: PostgreSQL 15, Redis 7
- Run short tests (`go test -v -race -short ./internal/... ./pkg/...`)

**3. Integration Tests (E2E)**
- Services: PostgreSQL 15, Redis 7, OpenSearch 2.11
- Run migrations
- Run E2E tests (`go test -v -timeout=10m -tags=integration`)
- Upload test results (retention: 7 days)

**4. Performance Tests**
- Only on push to main
- Run benchmarks (`go test -bench=. -benchmem -benchtime=10s`)
- Analyze results and add to GitHub summary
- Upload benchmark results (retention: 30 days)

**5. Build**
- Build service (`make build`)
- Upload binary (retention: 7 days)

**6. Docker Build**
- Only on push to main/develop
- Build and push to Docker Hub: `sveturs/listings-service`
- Tags: branch name, SHA, semver

**7. Deploy to Dev**
- Only on push to develop
- Environment: development (https://dev.vondi.rs)

**Environment Variables (Tests):**
```yaml
VONDILISTINGS_DB_HOST: localhost
VONDILISTINGS_DB_PORT: 5432
VONDILISTINGS_DB_USER: listings_user
VONDILISTINGS_DB_PASSWORD: listings_password
VONDILISTINGS_DB_NAME: listings_db_test
VONDILISTINGS_REDIS_HOST: localhost
VONDILISTINGS_REDIS_PORT: 6379
VONDILISTINGS_OPENSEARCH_HOST: localhost
VONDILISTINGS_OPENSEARCH_PORT: 9200
```

### Delivery Microservice

**Workflow:** `ci.yml`

**Trigger:** Push/PR to main

**Runner:** `self-hosted`

**Jobs:**

**1. CI Job**
- Checkout code
- **ВАЖНО:** Check no `replace` directives in go.mod
- Configure private repo access
- Setup Go 1.25
- Download dependencies
- Check proto files are up to date (`make proto`)
- Format proto files (`make proto-format`)
- Lint proto files (`make proto-lint`)
- Code quality checks (`make check`)
- Run all tests (`make test-all`)
- Build application (`make build`)
- Upload binary artifact (retention: 7 days)

**Environment:**
```yaml
env:
  GOPRIVATE: github.com/vondi-global/*
  TESTCONTAINERS_RYUK_DISABLED: true
```

**Note:** Delivery service проверяет protobuf файлы на актуальность генерации.

### Warehouse (WMS)

**Workflow:** `ci.yml`

**Trigger:** Push/PR to main

**Runner:** `self-hosted`

**Jobs:**

**1. Validate**
- Checkout code
- **ВАЖНО:** Check no `replace` directives in go.mod

**2. Build and Test**
- Services: PostgreSQL 16, Redis 7
- Setup private module access
- Setup Go 1.23
- Download dependencies (`go mod download`)
- Verify dependencies (`go mod verify`)
- Build (`make build`)
- Run migrations (все `*.up.sql` файлы)
- Run tests (`make test`)
- Run tests with race detector
- Upload coverage to Codecov

**3. Lint**
- Setup Go 1.23
- Download dependencies
- Install and run golangci-lint (manual install, timeout: 5m)

**Environment (Tests):**
```yaml
WMS_DB_HOST: localhost
WMS_DB_PORT: 5432
WMS_DB_USER: wms_user
WMS_DB_PASSWORD: wms_secure_pass_2025
WMS_DB_NAME: wms_db
WMS_DB_SSL_MODE: disable
WMS_REDIS_HOST: localhost
WMS_REDIS_PORT: 6379
```

**Note:** WMS запускает миграции через psql напрямую в CI.

### Shared Library (lib)

**Workflow:** `ci.yml`

**Trigger:** Push/PR to main

**Runner:** `self-hosted`

**Jobs:** Простой CI (проверка компиляции)

---

## Тестирование

### Уровни тестирования

| Уровень | Команда | Назначение | Сервисы |
|---------|---------|-----------|---------|
| **Unit Tests** | `go test -short ./...` | Быстрые изолированные тесты | - |
| **Integration Tests** | `go test -tags=integration ./...` | Тесты с БД/Redis | PostgreSQL, Redis |
| **E2E Tests** | `go test -timeout=10m -tags=integration ./test/integration/...` | End-to-end сценарии | PostgreSQL, Redis, OpenSearch |
| **Race Detection** | `go test -race ./...` | Поиск гонок данных | - |
| **Benchmarks** | `go test -bench=. -benchmem` | Производительность | PostgreSQL, Redis |

### Test Services в CI

**PostgreSQL:**
```yaml
services:
  postgres:
    image: postgres:15-alpine
    env:
      POSTGRES_DB: listings_db_test
      POSTGRES_USER: listings_user
      POSTGRES_PASSWORD: listings_password
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

**Redis:**
```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - 6379:6379
    options: >-
      --health-cmd "redis-cli ping"
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

**OpenSearch:**
```yaml
services:
  opensearch:
    image: opensearchproject/opensearch:2.11.0
    env:
      discovery.type: single-node
      OPENSEARCH_JAVA_OPTS: "-Xms512m -Xmx512m"
      DISABLE_SECURITY_PLUGIN: true
    ports:
      - 9200:9200
    options: >-
      --health-cmd "curl -f http://localhost:9200/_cluster/health || exit 1"
      --health-interval 30s
      --health-timeout 10s
      --health-retries 5
```

### Test Artifacts

**Upload Coverage:**
```yaml
- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage.out
    flags: unittests
    name: codecov-umbrella
```

**Upload Test Results:**
```yaml
- name: Upload E2E test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: e2e-test-results
    path: |
      test-results/
      coverage.txt
    retention-days: 7
```

**Upload Binaries:**
```yaml
- name: Upload binary artifact
  uses: actions/upload-artifact@v4
  with:
    name: auth-service
    path: bin/auth-service
    retention-days: 7
```

### Frontend Testing (Auth Service)

Auth service включает проверку frontend компонентов:

```yaml
- name: Install frontend dependencies
  working-directory: examples/tasktracker/frontend
  run: yarn install --frozen-lockfile

- name: Check frontend code formatting
  working-directory: examples/tasktracker/frontend
  run: yarn format:check

- name: Lint frontend code
  working-directory: examples/tasktracker/frontend
  run: yarn lint
```

---

## Docker Build & Push

### Build Strategy

**Monolith (vondi):**
- **Buildx:** Используется для multi-platform builds
- **Cache:** `type=gha` (GitHub Actions cache)
- **Context:** `./backend` или `./frontend/svetu`
- **Tags:** `{timestamp}` и `latest`

**Microservices (listings):**
- **Buildx:** Используется
- **Cache:** `type=gha,mode=max`
- **Context:** `.` (корень репозитория)
- **Tags:** branch name, SHA, semver

### Build Arguments

```yaml
build-args: |
  GIT_COMMIT=${{ github.sha }}
  BUILD_TIME=${{ env.TIMESTAMP }}
```

**Пример использования в Dockerfile:**
```dockerfile
ARG GIT_COMMIT
ARG BUILD_TIME
LABEL git.commit=$GIT_COMMIT
LABEL build.time=$BUILD_TIME
```

### Docker Registry Login

```yaml
- name: Login to Docker Registry
  uses: docker/login-action@v3
  with:
    registry: ${{ secrets.DOCKER_REGISTRY }}
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

**Credentials:**
- `DOCKER_REGISTRY`: `harbor.svetu.rs`
- `DOCKER_USERNAME`: `admin`
- `DOCKER_PASSWORD`: Harbor admin password

### Image Naming Convention

**Monolith:**
```
harbor.svetu.rs/svetu/backend/api:20251221_143052
harbor.svetu.rs/svetu/backend/api:latest
harbor.svetu.rs/svetu/frontend/app:20251221_143052
harbor.svetu.rs/svetu/frontend/app:latest
```

**Microservices:**
```
sveturs/listings-service:main
sveturs/listings-service:main-abc1234
sveturs/listings-service:1.2.3
```

### Build Performance

**Cache Strategies:**

1. **Go Modules Cache:**
```yaml
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

2. **Yarn Cache:**
```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'yarn'
    cache-dependency-path: examples/tasktracker/frontend/yarn.lock
```

3. **Docker Layer Cache:**
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

**Типичное время сборки:**
- Backend (без cache): ~3-5 минут
- Backend (с cache): ~1-2 минуты
- Frontend: ~5-7 минут

---

## Deployment Process

### Blue-Green Deployment (Monolith)

**Скрипты:**
- `scripts/blue_green_backend.sh`
- `scripts/blue_green_frontend.sh`

**Workflow:**
1. Copy deployment script to `/tmp/` на сервере
2. Make executable (`chmod +x`)
3. Execute script через SSH
4. Script выполняет:
   - Остановку старого контейнера (blue)
   - Запуск нового контейнера (green)
   - Health check нового контейнера
   - Переключение traffic на green
   - Удаление blue контейнера

**SSH Setup:**
```yaml
- name: Setup SSH key
  uses: webfactory/ssh-agent@v0.7.0
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

- name: Add host key to known hosts
  run: |
    mkdir -p ~/.ssh
    ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts
```

### Manual Deployment (Monolith)

```bash
# Через GitHub Actions UI
gh workflow run backend-deploy.yml --repo vondi-global/vondi

# Или через Make
cd /p/github.com/vondi-global/vondi
make deploy backend  # из Makefile вызывает scripts/blue_green_deploy_on_svetu.rs.sh
```

### Microservices Deployment

**Listings:**
```yaml
- name: Deploy to dev server
  run: |
    echo "Deployment to dev environment will be configured"
    # TODO: Add actual deployment steps
```

**Note:** Автоматический deploy только на develop branch в dev окружение.

### Deployment Secrets

| Secret | Назначение |
|--------|-----------|
| `SSH_PRIVATE_KEY` | SSH ключ для доступа к серверу |
| `SSH_USER` | SSH user (root) |
| `SERVER_HOST` | Hostname сервера (svetu.rs / vondi.rs) |
| `DOCKER_REGISTRY` | Harbor registry URL |
| `DOCKER_USERNAME` | Harbor username |
| `DOCKER_PASSWORD` | Harbor password |
| `GO_MODULES_TOKEN` | GitHub token для приватных модулей |

---

## Harbor Registry

**URL:** `harbor.svetu.rs`
**Web UI:** https://harbor.svetu.rs

### Namespaces

| Namespace | Назначение | Примеры образов |
|-----------|------------|-----------------|
| **svetu/backend/** | Backend services | api:latest, api:20251221_143052 |
| **svetu/frontend/** | Frontend apps | app:latest, app:20251221_143052 |
| **svetu/db/** | Databases | postgres:15 |
| **svetu/opensearch/** | Search | opensearch:2.11.0 |
| **svetu/mail/** | Mail services | server:latest, webui:latest |
| **svetu/tools/** | Utilities | certbot:latest, migrate:latest |
| **svetu/nginx/** | Web servers | nginx:latest |
| **svetu/minio/** | S3 storage | minio:RELEASE.2023-09-30, mc:latest |

### Authentication

```bash
# Login to Harbor
docker login harbor.svetu.rs
# Username: admin
# Password: Harbor12345

# Pull image
docker pull harbor.svetu.rs/svetu/backend/api:latest

# Push image
docker tag my-image:latest harbor.svetu.rs/svetu/backend/api:v1.0.0
docker push harbor.svetu.rs/svetu/backend/api:v1.0.0
```

### Image Retention Policy

**Production images:**
- `latest` tag - всегда сохраняется
- Timestamp tags - сохраняются 30 дней
- Branch tags - сохраняются 14 дней

**Test images:**
- Retention: 7 дней

### Harbor в CI/CD

GitHub Actions автоматически пушит образы в Harbor:

```yaml
- name: Build and push Backend Docker image
  uses: docker/build-push-action@v6
  with:
    context: ./backend
    push: true
    tags: |
      ${{ secrets.DOCKER_REGISTRY }}/svetu/backend/api:${{ env.TIMESTAMP }}
      ${{ secrets.DOCKER_REGISTRY }}/svetu/backend/api:latest
```

---

## Secrets Management

### GitHub Organization Secrets

**Доступ:** Settings → Secrets and variables → Actions

| Secret Name | Type | Используется в |
|-------------|------|----------------|
| `DOCKER_REGISTRY` | Registry URL | Все build workflows |
| `DOCKER_USERNAME` | Harbor user | Все build workflows |
| `DOCKER_PASSWORD` | Harbor password | Все build workflows |
| `SSH_PRIVATE_KEY` | SSH key | Deploy workflows |
| `SSH_USER` | SSH username | Deploy workflows |
| `SERVER_HOST` | Hostname | Deploy workflows |
| `GO_MODULES_TOKEN` | GitHub token | Все Go workflows |
| `CODECOV_TOKEN` | Codecov token | Warehouse CI |

### Environment-Specific Secrets

**Development Environment:**
- No additional secrets (uses organization secrets)

**Production Environment:**
- Separate SSH keys
- Separate database credentials

### Private Go Modules Access

```yaml
- name: Configure private repository access
  run: |
    git config --global url."https://x-access-token:${{ secrets.GO_MODULES_TOKEN }}@github.com/".insteadOf "https://github.com/"

- name: Download dependencies
  env:
    GOPRIVATE: github.com/vondi-global/*
  run: go mod download
```

**Token Permissions Required:**
- `repo` (all)
- `read:packages`

---

## Мониторинг CI/CD

### GitHub UI

**Просмотр workflows:**
```bash
# Список workflows
gh workflow list --repo vondi-global/vondi

# Запуски конкретного workflow
gh run list --workflow=ci.yml --repo vondi-global/auth

# Просмотр конкретного запуска
gh run view <RUN_ID> --repo vondi-global/auth

# Логи failed jobs
gh run view <RUN_ID> --log-failed
```

### Watch CI в реальном времени

**Для Pull Request:**
```bash
# Создать PR
gh pr create --repo vondi-global/vondi --base main --head feature/my-feature

# Следить за CI checks
gh pr checks <PR_NUMBER> --watch
```

**Для Push:**
```bash
# После push
git push vondi feature/my-feature

# Следить за CI
gh run watch --repo vondi-global/auth
```

### Self-Hosted Runner Metrics

**Проверка статуса:**
```bash
# Статус runner
gh api orgs/vondi-global/actions/runners --jq '.runners[] | {name, status, busy, labels}'

# Output:
# {
#   "name": "svetu-runner",
#   "status": "online",
#   "busy": false,
#   "labels": ["self-hosted", "Linux", "X64", "svetu"]
# }
```

**Логи runner:**
```bash
# Последние логи
ssh root@svetu.rs "journalctl -u actions.runner.vondi-global.svetu-runner --since '1 hour ago'"

# Живые логи
ssh root@svetu.rs "journalctl -u actions.runner.vondi-global.svetu-runner -f"
```

### CI Summary в GitHub

После выполнения jobs результаты добавляются в GitHub Step Summary:

```yaml
- name: Analyze benchmark results
  run: |
    echo "## Benchmark Results" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
    cat benchmark_results.txt >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
```

```yaml
- name: Generate summary
  run: |
    echo "## CI/CD Summary" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "✅ Lint: ${{ needs.lint.result }}" >> $GITHUB_STEP_SUMMARY
    echo "✅ Unit Tests: ${{ needs.unit-tests.result }}" >> $GITHUB_STEP_SUMMARY
```

---

## Troubleshooting

### Runner Offline

**Проблема:** Runner показывает status: offline в GitHub

**Решение:**
```bash
# 1. Проверить статус сервиса
ssh root@svetu.rs "systemctl status actions.runner.vondi-global.svetu-runner"

# 2. Перезапустить если inactive
ssh root@svetu.rs "systemctl restart actions.runner.vondi-global.svetu-runner"

# 3. Проверить логи
ssh root@svetu.rs "journalctl -u actions.runner.vondi-global.svetu-runner --since '30 min ago'"

# 4. Проверить connectivity к GitHub
ssh root@svetu.rs "curl -I https://github.com"
```

### Go Modules Private Repo 403

**Проблема:** `go mod download` fails with 403 Forbidden

**Причина:** GO_MODULES_TOKEN expired или нет permissions

**Решение:**
```yaml
# Проверить наличие токена
- name: Configure private repository access
  run: |
    git config --global url."https://x-access-token:${{ secrets.GO_MODULES_TOKEN }}@github.com/".insteadOf "https://github.com/"

# Проверить GOPRIVATE
- name: Download dependencies
  env:
    GOPRIVATE: github.com/vondi-global/*
  run: go mod download
```

**Создать новый токен:**
1. GitHub → Settings → Developer settings → Personal access tokens
2. Scopes: `repo`, `read:packages`
3. Обновить в GitHub Secrets

### Replace Directives Error

**Проблема:** CI fails с ошибкой "go.mod contains replace directives"

```
❌ ERROR: go.mod contains replace directives!
replace github.com/vondi-global/lib => ../lib
```

**Причина:** `replace` директивы закоммичены в go.mod (для local dev только!)

**Решение:**
```bash
# 1. Удалить replace директивы из go.mod
vim go.mod  # Удалить все строки "replace ..."

# 2. Проверить локально
grep "^replace" go.mod  # Должно быть пусто

# 3. Закоммитить
git add go.mod
git commit -m "fix: remove replace directives from go.mod"
git push
```

**Автоматическая проверка в CI:**
```yaml
- name: Check no replace directives in go.mod
  run: |
    if grep -q "^replace" go.mod; then
      echo "❌ ERROR: go.mod contains replace directives!"
      grep "^replace" go.mod
      exit 1
    fi
```

### Tests Timeout

**Проблема:** Tests exceed timeout

**Решение:**
```yaml
# Увеличить timeout для E2E tests
- name: Run E2E tests
  run: go test -v -timeout=20m -tags=integration ./test/integration/...
```

### Docker Build Out of Space

**Проблема:** Docker build fails "no space left on device"

**Решение на runner:**
```bash
ssh root@svetu.rs

# Очистить unused images
docker image prune -a -f

# Очистить build cache
docker builder prune -a -f

# Проверить место
df -h /var/lib/docker
```

### Cache Miss (Go Modules)

**Проблема:** Cache miss каждый раз, долгая сборка

**Причина:** go.sum изменился или cache key неправильный

**Решение:**
```yaml
# Правильный cache key
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

**Проверить cache hits:**
- GitHub UI → Actions → Workflow run → Cache hits

### Harbor Push Failed

**Проблема:** `docker push` fails с 401 Unauthorized

**Решение:**
```bash
# 1. Проверить credentials в GitHub Secrets
# Settings → Secrets → DOCKER_USERNAME, DOCKER_PASSWORD

# 2. Тестировать login локально
docker login harbor.svetu.rs -u admin -p Harbor12345

# 3. Проверить Harbor доступность
curl -I https://harbor.svetu.rs
```

### SSH Deploy Failed

**Проблема:** SSH connection refused во время deploy

**Решение:**
```bash
# 1. Проверить SSH_PRIVATE_KEY secret
# Должен быть private key БЕЗ passphrase

# 2. Проверить known_hosts
- name: Add host key to known hosts
  run: |
    mkdir -p ~/.ssh
    ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

# 3. Проверить SSH доступ
ssh ${{ secrets.SSH_USER }}@${{ secrets.SERVER_HOST }} "echo Connected"
```

### Proto Files Out of Date

**Проблема:** CI fails "Generated proto files are out of date"

**Причина:** protobuf сгенерированные файлы не закоммичены

**Решение:**
```bash
# 1. Сгенерировать proto files
make proto

# 2. Форматировать
go tool golang.org/x/tools/cmd/goimports -local github.com/vondi-global/delivery -w gen/
go tool mvdan.cc/gofumpt -w gen/

# 3. Проверить diff
git diff gen/

# 4. Закоммитить
git add gen/
git commit -m "chore: update generated proto files"
```

---

## Best Practices

### 1. Version Management

**ВСЕГДА обновлять версии перед PR:**

```bash
# Обновить internal/version/version.go
Version = "1.2.3"

# Обновить CHANGELOG.md
## [1.2.3] - 2025-12-21
### Added
- New feature X
```

### 2. Pre-Commit Checks

**Запустить локально перед push:**

```bash
# Backend
cd backend && make format && make lint && make test

# Frontend
cd frontend/svetu && yarn format && yarn lint && yarn build
```

### 3. Replace Directives

**НИКОГДА не коммитить replace директивы:**

```bash
# Проверить перед коммитом
grep "^replace" go.mod

# Должно быть пусто!
```

### 4. PR Management

**Мержить ТОЛЬКО после GREEN CI:**

```bash
# Проверить статус PR
gh pr checks <PR_NUMBER>

# ✅ Все checks GREEN - можно мержить
# ❌ Есть failures - исправить и push
```

### 5. Cache Strategy

**Go Modules:**
- Всегда кэшировать `~/.cache/go-build` и `~/go/pkg/mod`
- Key: `${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}`

**Yarn:**
- Использовать `cache: 'yarn'` в `setup-node@v4`

**Docker:**
- `cache-from: type=gha`
- `cache-to: type=gha,mode=max`

### 6. Secrets Rotation

**Регулярно обновлять:**
- GO_MODULES_TOKEN (каждые 90 дней)
- SSH_PRIVATE_KEY (каждые 6 месяцев)
- DOCKER_PASSWORD (каждые 6 месяцев)

### 7. Monitoring

**Проверять runner health каждую неделю:**

```bash
# Статус
gh api orgs/vondi-global/actions/runners

# Memory usage
ssh root@svetu.rs "systemctl status actions.runner.vondi-global.svetu-runner | grep Memory"
```

---

## Changelog

### 1.0.0 - 2025-12-21

**Added:**
- Полная документация CI/CD инфраструктуры
- Self-hosted runner на svetu.rs
- GitHub Actions workflows для всех сервисов
- Тестирование (unit, integration, E2E, benchmarks)
- Docker build & push в Harbor registry
- Blue-green deployment для monolith
- Secrets management
- Мониторинг и troubleshooting

**Microservices Covered:**
- Monolith (vondi) - backend/frontend build & deploy
- Auth Service - CI с frontend checks
- Listings Service - полный CI/CD pipeline
- Delivery Service - CI с proto validation
- Warehouse (WMS) - CI с миграциями
- Shared Library (lib) - базовый CI

**Features:**
- Self-hosted runner избегает GitHub Actions limits
- Приватные Go modules через GO_MODULES_TOKEN
- Harbor registry для Docker images
- Автоматические тесты с PostgreSQL, Redis, OpenSearch
- Blue-green deployment скрипты
- PR checks перед мержем

---

## См. также

- [SYSTEM_PASSPORT.md](../../SYSTEM_PASSPORT.md) - Общая архитектура
- [docker.md](./docker.md) - Docker контейнеры
- [network.md](./network.md) - Сетевая топология
- [../services/auth.md](../services/auth.md) - Auth микросервис
- [../services/listings.md](../services/listings.md) - Listings микросервис
- [../services/delivery.md](../services/delivery.md) - Delivery микросервис
