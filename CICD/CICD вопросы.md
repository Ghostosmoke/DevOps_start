
# 🎯 CI/CD для DevOps: Вопросы с ответами для собеседований

**Тема подтверждена:** `CI/CD — архитектура, паттерны, безопасность, масштабирование`  
**Уровень:** подготовка к собеседованию в топ-компанию (Middle → Architect)  
**Фокус:** теория + контекст продакшена + реальные кейсы

---

## 📋 План раскрытия темы

```
1. Фундамент: что такое CI, CD, Continuous Delivery vs Deployment
2. Архитектура пайплайнов: этапы, артефакты, идемпотентность
3. Безопасность: секреты, OIDC, supply chain security
4. Стратегии деплоя: blue-green, canary, rolling, feature flags
5. Тестирование в пайплайне: quality gates, shift-left, flaky tests
6. Артефакты и версионирование: промоушен без пересборки
7. Оптимизация: кэширование, параллелизация, selective testing
8. GitOps: ArgoCD/Flux, drift detection, SSOT
9. Мульти-окружения: promotion, config management, secrets scoping
10. Observability: метрики пайплайнов, alerting, SLO для CI/CD
11. Инфраструктура как код: интеграция Terraform/Pulumi в CI/CD
12. Масштабирование: self-hosted runners, distributed builds, queues
13. Compliance и аудит: кто, что, когда деплоил
14. Аварийные сценарии: rollback, hotfix, emergency access
15. Будущее CI/CD: ephemeral environments, AI-assisted pipelines
```

---

# 1️⃣ БЛОК ТЕОРИИ + ПРАКТИКИ (вопросы с ответами)

---

## 🔹 Вопрос 1: Фундамент — что такое CI, CD и в чём разница между Delivery и Deployment?

**Тема/Контекст**: Основы CI/CD, терминология  
**Вопрос**: Объясни разницу между Continuous Integration, Continuous Delivery и Continuous Deployment. Когда что применять?  
**Краткий ответ**:  
- **CI** — автоматическая сборка и тестирование каждого коммита.  
- **CD (Delivery)** — готовность к деплою в прод вручную.  
- **CD (Deployment)** — автоматический деплой в прод после прохождения тестов.  

**Развёрнутый ответ**:  

```
┌─────────────────────────────────────────┐
│ Continuous Integration (CI)             │
│ • Каждый push → сборка + тесты          │
│ • Цель: быстро найти баги               │
│ • Инструменты: GitHub Actions, GitLab CI│
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Continuous Delivery (CD)                │
│ • Артефакт готов к деплою в любой момент│
│ • Деплой в прод — ручной триггер        │
│ • Требует: staging, approval gates      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Continuous Deployment (CD)              │
│ • После CI + тестов → авто-деплой в прод│
│ • Требует: мощные тесты, feature flags  │
│ • Риск: баг в проде без ручного контроля│
└─────────────────────────────────────────┘
```

**Практика — пример пайплайна с manual approval**:
```yaml
# GitHub Actions: delivery vs deployment
name: Build and Deploy
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm ci && npm test && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: app-build
          path: dist/

  deploy-staging:  # Continuous Delivery
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging  # авто-деплой на staging

  deploy-prod:  # Continuous Deployment (с ручным апрувом)
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production  # только после manual approval
```

**Best practices**:
- Начинай с CI → Delivery → Deployment, не прыгай сразу на авто-деплой
- Для Continuous Deployment обязательны: канареечные деплои, автоматические откаты, feature flags
- Всегда логируй: кто, когда и какой артефакт деплоил

**Типичные ошибки**:
- ❌ Называть любой пайплайн "CI/CD", если нет автоматического деплоя
- ❌ Деплоить в прод без staging и approval
- ❌ Не различать "готов к деплою" и "деплоится сам"

**Уровень сложности**: Junior / Middle  
**Теги**: CI/CD, GitHub Actions, Best Practices, Terminology

---

## 🔹 Вопрос 2: Идемпотентность пайплайнов — почему это критично?

**Тема/Контекст**: Архитектура пайплайнов, надёжность  
**Вопрос**: Что значит "идемпотентный пайплайн" и как его реализовать на практике?  
**Краткий ответ**: Идемпотентный пайплайн даёт одинаковый результат при многократном запуске с теми же входными данными, что позволяет безопасно ретраить, откатываться и избегать дублирования артефактов.  

**Развёрнутый ответ**:  

**Проблема без идемпотентности**:
```
Запуск 1: сборка → артефакт v1.0 → деплой → ОК
Запуск 2 (ретрай): сборка → артефакт v1.0.1 (другой хеш!) → деплой → дубликат в реестре
```

**Решение — детерминированные артефакты**:
```bash
# Использовать хеш коммита как часть тега
docker build -t registry/app:${GITHUB_SHA} \
  --build-arg COMMIT=${GITHUB_SHA} \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) .

# Проверка: если артефакт уже существует — не пересобирать
if docker manifest inspect registry/app:${GITHUB_SHA} &>/dev/null; then
  echo "✅ Артефакт уже существует, пропускаем сборку"
  exit 0
fi
```

**Идемпотентность в Infrastructure as Code**:
```hcl
# Terraform: plan + apply с файлом плана
terraform plan -out=tfplan -input=false
terraform apply -auto-approve tfplan

# Повторный запуск с теми же исходниками не изменит инфраструктуру
```

**Best practices**:
- ✅ Тегировать артефакты по `git SHA` или `semver`, никогда не использовать `latest` в прод
- ✅ Хэшировать содержимое артефактов и проверять перед публикацией
- ✅ Использовать `--no-cache` только там, где это осознанно нужно
- ✅ Применять `terraform plan` + `apply` с файлом плана

**Trade-offs**:
| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Тег по SHA | Детерминировано, уникально | Не человекочитаемо |
| Тег по semver | Человекочитаемо | Требует строгого контроля версий |
| Hybrid: `v1.2.3-${SHA:0:7}` | Лучшее из двух миров | Чуть сложнее парсить |

**Уровень сложности**: Middle  
**Теги**: Docker, Terraform, Idempotency, Artifact Management, Best Practices

---

## 🔹 Вопрос 3: Безопасность — как управлять секретами в пайплайне?

**Тема/Контекст**: Security, secrets management  
**Вопрос**: Какие способы передачи секретов в CI/CD существуют и какой выбрать для продакшена?  
**Краткий ответ**: Секреты нужно хранить в специализированных хранилищах (GitHub Secrets, Vault, AWS Secrets Manager), передавать через environment variables с ограниченным scope, никогда не логировать и использовать OIDC для временных кредов.  

**Развёрнутый ответ**:  

**Сравнение подходов**:

| Метод | Пример | Безопасность | Сложность | Когда использовать |
|-------|--------|--------------|-----------|-------------------|
| **CI-секреты** | `{{ secrets.DB_PASS }}` | ⭐⭐⭐ | Низкая | Стартапы, небольшие проекты |
| **OIDC + облачные роли** | AWS IAM Role via GitHub OIDC | ⭐⭐⭐⭐⭐ | Средняя | Продакшен, мульти-облако |
| **HashiCorp Vault** | Динамические секреты с TTL | ⭐⭐⭐⭐⭐ | Высокая | Энтерпрайз, строгий комплаенс |
| **External Secrets Operator** | Синхронизация Vault → K8s Secrets | ⭐⭐⭐⭐ | Средняя | Kubernetes-native окружения |

**Практика — OIDC в GitHub Actions + AWS**:
```yaml
# .github/workflows/deploy.yml
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/ci-cd-role
    role-session-name: github-actions-deploy
    aws-region: eu-west-1
    # Никаких долгоживущих ключей!
```

**Практика — Vault динамические секреты**:
```bash
# Генерация временного токена для деплоя
VAULT_TOKEN=$(vault write -field=token auth/github/login role=ci-cd)
vault kv get -field=password secret/prod/db | docker login -u ci-bot --password-stdin

# Токен живёт 15 минут, потом автоматически инвалидируется
```

**Запрещено категорически**:
```yaml
# ❌ Никогда так не делайте:
- run: echo "${{ secrets.API_KEY }}"  # попадёт в логи
- run: export KEY=hardcoded123  # хардкод в репозитории
- run: curl -H "Authorization: $SECRET" https://api  # может утечь в access logs
```

**Best practices**:
- ✅ Включать audit-логирование доступа к секретам
- ✅ Использовать `masking` для чувствительных данных в логах (`::add-mask::` в GitHub Actions)
- ✅ Ротировать секреты автоматически (Vault, AWS Secrets Manager)
- ✅ Ограничивать scope секретов по окружениям и веткам

**Уровень сложности**: Senior  
**Теги**: Security, Secrets Management, OIDC, Vault, AWS, GitHub Actions

---

## 🔹 Вопрос 4: Стратегии деплоя — blue-green vs canary vs rolling

**Тема/Контекст**: Deployment strategies, zero-downtime  
**Вопрос**: В чём разница между blue-green, canary и rolling update? Когда что применять?  
**Краткий ответ**: Blue-green переключает трафик между двумя идентичными окружениями мгновенно, canary направляет часть трафика на новую версию постепенно, rolling обновляет поды по одному — выбор зависит от требований к откату, ресурсам и скорости валидации.  

**Развёрнутый ответ**:  

**Сравнительная таблица**:

| Стратегия | Время деплоя | Откат | Ресурсы | Сложность | Лучше всего для |
|-----------|--------------|-------|---------|-----------|-----------------|
| **Rolling Update** | Медленно | Быстро | 1.5x | Низкая | Стандартные K8s деплои |
| **Blue-Green** | Быстро | Мгновенно | 2x | Средняя | Критичные сервисы, стейтлесс |
| **Canary** | Очень медленно | Постепенно | 1.2-1.5x | Высокая | Высоконагруженные, риск-аверсивные |
| **Feature Flags** | Мгновенно | Мгновенно | 1x | Средняя | A/B тесты, постепенный релиз функционала |

**Blue-Green — схема и пример**:
```
[Prod v1] ← 100% трафика (blue)
[Prod v2] ← развёрнута, тестирование (green)

→ Переключение Ingress/LoadBalancer → [Prod v2] получает 100%
→ [Prod v1] остаётся "на откат" на 24-48 часов
```

```yaml
# Kubernetes: два deployment + service selector
apiVersion: v1
kind: Service
meta
  name: myapp-prod
spec:
  selector:
    app: myapp
    version: blue  # ← переключаем на 'green' для деплоя
  ports:
    - port: 80
```

**Canary — пример с Argo Rollouts**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
meta
  name: myapp
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10  # 10% трафика на новую версию
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: error-rate-check  # Prometheus-метрики
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

**Rolling Update — стандартный K8s**:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # +1 под сверх desired
      maxUnavailable: 0  # ноль даунтайма
```

**Best practices**:
- ✅ Всегда тестировать стратегию в staging перед prod
- ✅ Интегрировать с observability: автоматический откат при росте 5xx > 1% или latency > P95
- ✅ Документировать процедуру отката для каждой стратегии
- ✅ Использовать health checks и readiness probes

**Типичные ошибки**:
- ❌ Применять canary без метрик и автоматического отката
- ❌ Забывать про миграции БД при blue-green (нужна обратная совместимость)
- ❌ Не тестировать откат — "а вдруг не сработает?"

**Уровень сложности**: Senior / Architect  
**Теги**: Kubernetes, Argo Rollouts, Deployment Strategies, Observability, SRE

---

## 🔹 Вопрос 5: Тестирование в пайплайне — какие чеки обязательны?

**Тема/Контекст**: Quality gates, testing pyramid  
**Вопрос**: Какие типы тестов обязательно включать в CI и где их запускать?  
**Краткий ответ**: Минимум: линтеры → юнит-тесты → интеграционные → e2e на staging; критичные чеки блокируют мерж, тяжёлые тесты — асинхронно, flaky-тесты изолируются и мониторятся.  

**Развёрнутый ответ**:  

**Тестовая пирамида в CI/CD**:
```
        /\
       /E2E\      ← Запускать после мержа в main, на staging
      /------\
     /Integration\ ← Запускать на PR, с моками/тестовыми БД
    /--------------\
   /Unit + Lint + Security\ ← Запускать на каждый push, блокируют мерж
  --------------------------
```

**Пример пайплайна с quality gates**:
```yaml
# GitLab CI: этапы и условия
stages:
  - lint
  - test
  - build
  - deploy-staging
  - e2e
  - deploy-prod

lint:
  stage: lint
  script:
    - eslint --max-warnings=0 src/
    - terraform validate
    - trivy config .  # проверка IaC на уязвимости
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

unit-tests:
  stage: test
  script:
    - pytest --cov=app --cov-fail-under=80 --cov-report=xml
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

integration-tests:
  stage: test
  services: [postgres:15, redis:7]
  script:
    - pytest tests/integration --db-url=$CI_DB_URL
  variables:
    CI_DB_URL: postgres://user:pass@postgres:5432/test

e2e-tests:
  stage: e2e
  environment: staging
  script:
    - newman run postman/collection.json --env-var base_url=$STAGING_URL
  rules:
    - if: $CI_COMMIT_BRANCH == "main"  # только после мержа
  allow_failure: true  # не блокировать деплой, но алертить
```

**Quality gates — что блокирует мерж**:
```yaml
# Пример conditions для merge request
merge_checks:
  - lint: pass
  - unit_tests: pass AND coverage >= 80%
  - security_scan: no critical vulnerabilities
  - build: success
  # e2e — не блокирует, но требует ручного подтверждения для prod
```

**Работа с flaky-тестами**:
```bash
# Автоматическое определение нестабильных тестов
pytest --flaky-report=flaky.json

# В пайплайне: перезапускать flaky-тесты до 2 раз, но логировать
if [ $TEST_STATUS == "flaky" ]; then
  retry_test.sh --max-attempts 2
  alert_slack "Flaky test detected: $TEST_NAME"
fi
```

**Best practices**:
- ✅ Запускать быстрые чеки (линт, юнит) на каждый PR — быстрый feedback
- ✅ Тяжёлые тесты (e2e, performance) — асинхронно, после мержа
- ✅ Изолировать flaky-тесты и мониторить их стабильность
- ✅ Использовать `--cov-fail-under` для контроля покрытия

**Trade-offs**:
| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Все тесты на каждый PR | Максимальная уверенность | Долгий feedback, дорого |
| Selective testing | Быстро, экономит ресурсы | Сложнее настроить, риск пропустить баг |
| Параллелизация тестов | Ускорение в 3-5 раз | Требует инфраструктуры, сложнее дебажить |

**Уровень сложности**: Middle  
**Теги**: Testing, GitLab CI, Quality Gates, pytest, Coverage, Shift-Left

---

## 🔹 Вопрос 6: Артефакты — как продвигать между окружениями без пересборки?

**Тема/Контекст**: Artifact management, promotion pipeline  
**Вопрос**: Как организовать промоушен артефактов между окружениями, чтобы гарантировать, что в прод попадает тот же бинарник, что тестировался на staging?  
**Краткий ответ**: Собирать артефакт один раз с уникальным неизменяемым тегом (git SHA / semver), хранить в реестре с метаданными, продвигать ссылку на тот же артефакт через окружения, не пересобирая.  

**Развёрнутый ответ**:  

**Правильный поток (Immutable Artifacts)**:
```
1. CI: build → docker build -t registry/app:${GIT_SHA}
2. Push в registry + генерация SBOM + подпись (cosign)
3. CD staging: deploy ${GIT_SHA}
4. После тестов: CD prod → deploy тот же ${GIT_SHA}
5. Audit log: кто, когда и какой артефакт продвинул
```

**Пример с GitHub Actions + multi-env**:
```yaml
# build.yml
- name: Build and push
  run: |
    docker build -t registry/app:${{ github.sha }} .
    docker push registry/app:${{ github.sha }}
    # Дополнительно тегировать semver для релизов
    if [[ $GITHUB_REF =~ ^refs/tags/v ]]; then
      docker tag registry/app:${{ github.sha }} registry/app:${GITHUB_REF#refs/tags/}
      docker push registry/app:${GITHUB_REF#refs/tags/}
    fi
  # Сохранить хеш для следующих стадий
  - name: Set output
    run: echo "image_tag=${{ github.sha }}" >> $GITHUB_OUTPUT

# deploy-prod.yml
- name: Deploy
  run: |
    # Используем тот же SHA, который прошёл staging
    kubectl set image deployment/app app=registry/app:${{ needs.build.outputs.image_tag }}
    # Проверка: артефакт подписан?
    cosign verify registry/app:${{ needs.build.outputs.image_tag }} --key cosign.pub
```

**Supply Chain Security — обязательные чеки**:
```bash
# Генерация SBOM (Software Bill of Materials)
syft registry/app:${GIT_SHA} -o spdx-json > sbom.json

# Подпись артефакта
cosign sign --key $COSIGN_KEY registry/app:${GIT_SHA}

# Верификация перед деплоем
cosign verify registry/app:${GIT_SHA} --key cosign.pub || exit 1
```

**Best practices**:
- ✅ Иммутабельные теги: один хеш → один артефакт
- ✅ Подписывать артефакты (cosign, notary) и генерировать SBOM (syft)
- ✅ Хранить метаданные: кто собрал, когда, какие тесты прошли
- ✅ Использовать промоушен-пайплайны, а не ручные `docker pull`

**Типичные ошибки**:
- ❌ Пересобирать артефакт для каждого окружения → риск расхождений
- ❌ Использовать `latest` тег → невозможно отследить, что деплоилось
- ❌ Не подписывать артефакты → риск supply chain атак

**Уровень сложности**: Middle / Senior  
**Теги**: Docker, Artifact Management, GitOps, Promotion, Supply Chain Security, Cosign

---

## 🔹 Вопрос 7: Оптимизация — как ускорить медленный пайплайн?

**Тема/Контекст**: Performance, caching, parallelization  
**Вопрос**: Пайплайн выполняется 40 минут. Как ускорить без потери качества проверок?  
**Краткий ответ**: Параллелизация независимых этапов, кэширование зависимостей и Docker-слоёв, selective testing (запускать только тесты затронутых модулей), использование self-hosted runners с предустановленными инструментами.  

**Развёрнутый ответ**:  

**Диагностика — где тратится время**:
```bash
# GitHub Actions: включить step timing
- name: Run tests
  run: npm test
  # В логах увидишь: [Step duration: 12m 34s]

# GitLab CI: использовать reports
test:
  script:
    - time pytest --durations=10
  artifacts:
    reports:
      junit: pytest-report.xml
```

**Конкретные приёмы ускорения**:

1. **Кэш зависимостей**:
```yaml
# GitHub Actions
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      ~/.cache/pip
      node_modules
    key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json', '**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-deps-
```

2. **Docker layer caching**:
```dockerfile
# Multi-stage build с оптимизацией слоёв
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci  # кэшируется, если package-lock не менялся

FROM deps AS builder
COPY . .
RUN npm run build  # этот слой пересобирается при изменении кода

FROM alpine:3.19
COPY --from=builder /app/dist /app
CMD ["/app"]
```

3. **Параллелизация тестов**:
```yaml
# GitLab CI: parallel matrix
test:
  parallel:
    matrix:
      - TEST_GROUP: [unit, integration, e2e]
      - NODE_VERSION: [18, 20]
  script:
    - run-tests.sh $TEST_GROUP --node $NODE_VERSION
```

4. **Selective testing** (запускать только тесты затронутых файлов):
```bash
# Пример с pytest + git diff
CHANGED_DIRS=$(git diff --name-only origin/main | grep '\.py$' | xargs -I{} dirname {} | sort -u)
pytest $(echo $CHANGED_DIRS | sed 's|/|.|g')/tests/ || echo "No tests for changed files"
```

5. **Self-hosted runners с предустановленными образами**:
- Docker-in-Docker с кэшем образов
- Pre-warmed Terraform provider cache
- Pre-installed language runtimes

**Метрики для мониторинга**:
```yaml
# Целевые значения для здорового пайплайна
metrics:
  mean_time_to_feedback: "< 10 min"  # для PR
  cache_hit_rate: "> 80%"
  flaky_test_rate: "< 1%"
  pipeline_success_rate: "> 95%"
```

**Trade-offs**:
| Оптимизация | Экономия времени | Риски |
|-------------|------------------|-------|
| Кэш зависимостей | 30-50% | Устаревший кэш → баги |
| Параллелизация | 2-5x | Сложнее дебажить, больше ресурсов |
| Selective testing | 20-70% | Риск пропустить баг в затронутом коде |
| Shallow clone | 10-30% | Не хватает истории для changelog |

**Уровень сложности**: Senior  
**Теги**: Performance, Caching, Docker, GitHub Actions, GitLab CI, Optimization

---

## 🔹 Вопрос 8: GitOps — в чём преимущество перед классическим push-деплоем?

**Тема/Контекст**: GitOps, ArgoCD, Flux  
**Вопрос**: Почему компания переходит на GitOps? Какие проблемы это решает и какие новые сложности добавляет?  
**Краткий ответ**: GitOps использует Git как единственный источник истины, а агент в кластере (ArgoCD/Flux) сам тянет изменения, обеспечивая аудит, откат и автоматическое восстановление при дрейфе, но требует зрелости процессов и разделения репозиториев.  

**Развёрнутый ответ**:  

**Архитектура GitOps vs Push**:
```
Push-подход (классический CI/CD):
[CI] --(kubectl apply)--> [Kubernetes]
• CI имеет креды на кластер
• Нет автоматического восстановления при ручных изменениях

GitOps-подход:
[Dev] --> PR в gitops-repo --> merge
                          ↓
[ArgoCD в кластере] видит изменение --> sync --> Kubernetes
• Кластер не принимает внешние push-запросы
• Любое расхождение с Git автоматически исправляется (self-heal)
```

**Преимущества GitOps**:
- ✅ **Аудит**: вся история изменений в Git (кто, что, когда)
- ✅ **Откат**: `git revert` → авто-синхронизация, без сложных скриптов
- ✅ **Drift detection**: если кто-то изменил кластер вручную, ArgoCD покажет расхождение и может авто-исправить
- ✅ **Безопасность**: нет долгоживущих кредов в CI, кластер не принимает внешние push-запросы
- ✅ **Консистентность**: одно и то же описание для dev/staging/prod (через overlays)

**Пример `Application` для ArgoCD**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
meta
  name: myapp-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/gitops-repo
    path: apps/myapp/overlays/prod
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true      # удалять ресурсы, которых нет в Git
      selfHeal: true   # авто-исправление дрейфа
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  healthChecks:
    - group: apps
      kind: Deployment
      check: true
```

**Сложности и trade-offs**:
| Проблема | Решение |
|----------|---------|
| Разделение app-repo и gitops-repo | Использовать `kustomize build` или `helm template` в CI для генерации манифестов |
| Управление секретами | External Secrets Operator + Vault, или sealed-secrets |
| Онбординг новых сервисов | Шаблоны приложений, CLI-утилиты (`argocd app create`) |
| Не подходит для serverless | Использовать гибридный подход: GitOps для K8s, CI/CD для Lambda/Cloud Run |

**Best practices**:
- ✅ Использовать Kustomize/Helm overlays для разных окружений
- ✅ Включать `prune: true` и `selfHeal: true` только после тестов в staging
- ✅ Интегрировать с notification-системами (Slack, PagerDuty) при рассинхронизации
- ✅ Требовать PR + review для изменений в gitops-repo, даже для аварийных фиксов

**Уровень сложности**: Architect  
**Теги**: GitOps, ArgoCD, Kubernetes, Flux, IaC, Security
