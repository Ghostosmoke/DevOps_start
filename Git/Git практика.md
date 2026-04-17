# 🛠️ Git для DevOps: Полноценная Практика (Hands-On)

**Формат:** Реальные сценарии из продакшена → пошаговое выполнение → DevOps-контекст → задания для самостоятельной отработки.  
**Рекомендация:** Создай чистый тестовый репозиторий (`mkdir git-practice && cd git-practice && git init`) и проходи каждый сценарий в нём. Не используй рабочие проекты для экспериментов с `reset --hard`, `filter-repo` или `force push`.

---

## 📁 СЦЕНАРИЙ 1: Базовый рабочий цикл + PR/MR в команде

### 🎯 Цель
Отработать стандартный поток: ветка → коммиты → пуш → Pull Request → merge с сохранением истории.

### 📋 Пошаговое выполнение
```bash
# 1. Инициализация и подключение к remote
git init --initial-branch=main
git remote add origin git@github.com:yourname/practice.git

# 2. Создание фича-ветки
git switch -c feat/api-healthcheck

# 3. Работа с кодом (имитация)
mkdir -p src && echo '{"status":"ok"}' > src/health.json
git add src/health.json
git commit -m "feat: add healthcheck endpoint"

# 4. Проверка перед пушем
git status --short   # должно быть чисто
git log --oneline    # видим 1 коммит в новой ветке

# 5. Отправка на сервер и привязка
git push -u origin feat/api-healthcheck
```

### 🔍 DevOps-контекст
- `-u` (`--set-upstream`) создаёт связь `local:feat/... ↔ remote:feat/...`. После этого достаточно `git push`/`git pull`.
- В продакшене PR/MR защищён правилами: требуется 2 аппрувера, статус CI зелёный, шаблон описания изменений.
- Merge-стратегия: `squash merge` (один чистый коммит в `main`) или `rebase merge` (линейная история).

### ⚠️ Подводные камни
- `git push` без `-u` в первый раз выдаст ошибку или запушит не в ту ветку.
- Если забыл `.gitignore` и закоммитил `.env` → история уже содержит секрет. Исправляется в Сценарии 4.

### 🧪 Задание для отработки
1. Создай ветку `fix/log-format`, добавь файл, закоммить.
2. На GitHub/GitLab создай PR с шаблоном описания.
3. Включи `Squash and merge` и проследи, как изменится история в `main`.
4. Проверь через `git log --oneline --graph --all`, что ветка исчезла, а коммит попал в `main`.

---

## 📁 СЦЕНАРИЙ 2: Разрешение конфликтов и интерактивный rebase

### 🎯 Цель
Научиться решать конфликты без паники и чистить историю перед мержом.

### 📋 Пошаговое выполнение
```bash
# 1. Симуляция расхождения веток
git checkout main
echo "version: 1.0" > config.yaml && git add . && git commit -m "chore: bump version"

git checkout feat/api-healthcheck
echo "version: 1.1" > config.yaml && git add . && git commit -m "chore: update config"

# 2. Попытка слияния → конфликт
git merge main
# ❌ CONFLICT (content): Merge conflict in config.yaml

# 3. Разрешение конфликта
cat config.yaml
# <<<<<<< HEAD
# version: 1.1
# =======
# version: 1.0
# >>>>>>> main

# Правим вручную или выбираем одну из версий:
echo "version: 1.1 # merged from main" > config.yaml
git add config.yaml
git commit -m "fix: resolve config conflict"  # или git merge --continue

# 4. Интерактивный rebase (чистка истории)
git log --oneline -5   # допустим, видим 3 лишних коммита
git rebase -i HEAD~3
# Откроется редактор. Замени pick на:
# pick abc123 feat: add healthcheck
# squash def456 chore: fix typo
# drop 789ghi chore: wip debug
# Сохрани → откроется окно для редактирования сообщения нового коммита.
```

### 🔍 DevOps-контекст
- Конфликты в `main` блокируют деплой. Автоматизированная проверка конфликтов в CI (`git merge --no-commit --no-ff main || exit 1`) экономит часы.
- `rebase -i` критичен для генерации changelog и семантического версионирования: один PR = один логический коммит.
- `rerere.enabled=true` запоминает, как ты разрешал конфликты, и применяет автоматически при повторных rebase/merge.

### ⚠️ Подводные камни
- `rebase` меняет хеши. Никогда не ребейзи ветки, которые уже используют коллеги.
- После `rebase` пуш потребует `--force-with-lease`.

### 🧪 Задание для отработки
1. Создай 4 коммита в ветке с опечатками в сообщениях.
2. Через `git rebase -i HEAD~4` объедини их в 1 коммит с корректным сообщением.
3. Сделай `git push --force-with-lease origin <branch>` и убедись, что история обновилась.

---

## 📁 СЦЕНАРИЙ 3: Оптимизация Git в CI/CD пайплайнах

### 🎯 Цель
Ускорить клонирование, сэкономить место на раннерах и запускать пайплайн только при нужных изменениях.

### 📋 Пошаговое выполнение
```yaml
# .gitlab-ci.yml (аналогично для GitHub Actions)
stages:
  - validate
  - deploy

# 1. Оптимизированный клон
variables:
  GIT_STRATEGY: clone
  GIT_DEPTH: 1                # только последний коммит
  GIT_FETCH_EXTRA_FLAGS: "--tags --prune"

# 2. Кэширование .git между запусками
cache:
  key: git-objects
  paths:
    - .git/objects/
  policy: pull-push

validate:
  stage: validate
  script:
    # 3. Быстрая проверка тегов для версионирования
    - LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
    - echo "Building from $CI_COMMIT_SHA based on $LATEST_TAG"
    - |
      if [ "$CI_PIPELINE_SOURCE" = "push" ] && echo "$CI_COMMIT_MESSAGE" | grep -q "skip-ci"; then
        echo "⏭️ CI skipped by commit message"
        exit 0
      fi
  rules:
    - changes:
        - infra/**/*
        - src/**/*
      when: always
    - when: never  # не запускать при изменениях в docs/, tests/ и т.д.
```

### 🔍 DevOps-контекст
- `GIT_DEPTH=1` сокращает время клонирования с ~30с до ~2с на больших репо.
- Кэширование `.git/objects/` позволяет последующим джобам делать `git fetch` вместо полного клонирования.
- `rules.changes` (path-based triggers) экономят compute-минуты: пайплайн запускается только при изменении релевантных файлов.
- В GitHub Actions аналог: `fetch-depth: 1`, `actions/cache@v4`, `paths: [...]`.

### ⚠️ Подводные камни
- `GIT_DEPTH=1` ломает `git describe`, `git log --since`, `git bisect`. Если нужны теги/история → `GIT_DEPTH=50` или `git fetch --unshallow`.
- Кэш `.git/` может разрастаться. Добавляй `git gc --auto` в конец пайплайна.

### 🧪 Задание для отработки
1. Настрой CI-файл с `GIT_DEPTH=1` и кэшем `.git/objects`.
2. Запусти пайплайн дважды: первый раз (полный клон + кэш), второй (из кэша + fetch).
3. Замерь разницу во времени выполнения `git clone`/`fetch`.

---

## 📁 СЦЕНАРИЙ 4: Утечка секрета и безопасная очистка истории

### 🎯 Цель
Научиться реагировать на инцидент утечки секрета и полностью удалять его из истории.

### 📋 Пошаговое выполнение
```bash
# 1. Симуляция утечки
echo "DB_PASSWORD=supersecret123" > .env
git add .env && git commit -m "chore: add config"
git push  # секрет ушёл в историю

# 2. Немедленная реакция (до очистки!)
# ✅ 1. Ротируй пароль в БД/Vault
# ✅ 2. Добавь в .gitignore и удали из индекса
echo ".env" >> .gitignore
git rm --cached .env
git commit -m "security: remove secret from tracking"

# 3. Полная очистка истории (фильтрация)
# Установи: pip install git-filter-repo  (или brew, apt)
git filter-repo --invert-paths --path .env --force

# 4. Очистка локальных ссылок и мусора
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 5. Отправка на сервер
git push --force-with-lease origin main
```

### 🔍 DevOps-контекст
- `git rm` или `commit --amend` **не удаляют** данные из `.git/objects`. Они остаются доступны по хешу до `gc`.
- `filter-repo` (официальная замена устаревшего `filter-branch`) переписывает все объекты. После этого **все клоны станут несовместимыми** с новым remote.
- В продакшене: сначала ротируй секрет → потом чисти историю → уведомляй команду → проверяй аудит-логи.

### ⚠️ Подводные камни
- Если кто-то уже сделал `git pull` после утечки, у него останется копия секрета. Ротация обязательна.
- `filter-repo` требует чистого рабочего дерева. Сделай `git stash` или закоммить изменения перед запуском.

### 🧪 Задание для отработки
1. Добавь файл `secret.key`, закоммить, запушь.
2. Выполни очистку через `filter-repo`.
3. Проверь результат: `git log --all --full-history -- secret.key` (должно вернуть пусто).
4. Добавь `pre-commit` хук с `gitleaks` и убедись, что повторная попытка добавить секрет блокируется.

---

## 📁 СЦЕНАРИЙ 5: Отладка падений и восстановление состояния

### 🎯 Цель
Найти коммит, сломавший сборку, и восстановить "потерянные" изменения.

### 📋 Пошаговое выполнение
```bash
# 1. Автоматический поиск баг-коммита (bisect)
git bisect start
git bisect bad HEAD          # текущая версия сломана
git bisect good v1.2.0       # эта работала

# Автоматизация через скрипт:
git bisect run ./test-build.sh  # скрипт: exit 0 = ok, exit 1 = fail, exit 125 = skip
# Git сам переключает коммиты и найдёт виновника.

git bisect reset  # выйти из режима

# 2. Восстановление после случайного reset
git reset --hard HEAD~3  # ⚠️ имитация ошибки
git log --oneline        # последние 3 коммита "исчезли"

git reflog               # найдём хеш потерянного коммита: abc123 HEAD@{3}: commit: critical fix
git switch -c recovery abc123  # или git reset --hard abc123
```

### 🔍 DevOps-контекст
- `bisect run` незаменим в CI: можно скормить ему `docker build` или `pytest` и за 5-10 шагов найти регрессию.
- `reflog` — локальный журнал. Не пушится. Хранится 90 дней по умолчанию. В ephemeral CI-раннерах `reflog` пуст после каждого запуска.
- В GitOps `git revert` предпочтительнее `reset`: он создаёт новый коммит-отмену, сохраняет аудит и не ломает синхронизацию.

### ⚠️ Подводные камни
- `bisect` работает только если есть чёткий критерий "good/bad". Для флейковых тестов используй `exit 125` (skip).
- `reflog` не спасёт, если уже сделан `git gc --prune=now` или репо клонировано заново.

### 🧪 Задание для отработки
1. Создай 10 коммитов. В 7-м поломай скрипт сборки.
2. Запусти `git bisect run ./build-check.sh` и найди виновника.
3. Сделай `git reset --hard HEAD~2`, восстанови состояние через `reflog`.

---

## 📁 СЦЕНАРИЙ 6: GitOps workflow (Infra as Code + Sync)

### 🎯 Цель
Отработать паттерн, где Git — единственный источник истины для инфраструктуры.

### 📋 Пошаговое выполнение
```bash
# 1. Структура репозитория
mkdir -p infra/{prod,staging,modules} apps/web
touch infra/prod/k8s/deployment.yaml infra/modules/network/main.tf

# 2. Изменение инфраструктуры через PR
git switch -c infra/update-replicas
echo "  replicas: 3" >> infra/prod/k8s/deployment.yaml
git add . && git commit -m "infra: scale web to 3 replicas"
git push -u origin infra/update-replicas
# → Создаём PR → CI запускает `terraform plan` / `kubeval` → Review → Merge

# 3. Откат изменений (Drift / Rollback)
# Если деплой сломал кластер:
git revert -m 1 <merge-commit-sha>  # -m 1 = откатить merge относительно main
git push origin main                # ArgoCD/Flux автоматически применит откат

# 4. Проверка синхронизации
# ArgoCD: argocd app get web-prod
# Flux: flux get kustomizations web-prod
```

### 🔍 DevOps-контекст
- **Pull-модель деплоя**: кластер сам тянет изменения из Git. Нет прямого write-доступа у людей/CI к API K8s/Terraform.
- **Drift detection**: оператор каждые N минут сравнивает желаемое (Git) и фактическое (кластер). При рассинхроне алерт или автокоррекция.
- `git revert` в main = мгновенный rollback без переписывания истории. Идеально для compliance-аудита.

### ⚠️ Подводные камни
- Хранение состояний Terraform в Git **запрещено** (секреты, блокировки). Используй remote backend (S3 + DynamoDB, GCS, Terraform Cloud).
- Слишком частые коммиты в `infra/` могут вызвать "sync storm" у GitOps-оператора. Группируй изменения или настраивай `sync-wave`.

### 🧪 Задание для отработки
1. Разверни локально `kind` + `ArgoCD` (или используй `minikube` + `flux`).
2. Подключи свой репо как Application.
3. Измени манифест в ветке → сделай PR → смержи → убедись, что ArgoCD/Flux синхронизировал кластер.
4. Сделай `git revert`, проверь автоматический откат.

---

## 🛠️ Полезные алиасы и автоматизация

Добавь в `~/.gitconfig` для ускорения рутины:
```ini
[alias]
  st = status --short --branch
  co = switch
  ci = commit
  br = branch -vv
  log = log --oneline --graph --decorate --all
  diff = diff --color-words
  undo = reset --soft HEAD~1
  cleanup = !git fetch --prune && git remote prune origin && git gc --auto
  lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
  wip = !git add -A && git commit -m "wip: $(date +'%Y-%m-%d %H:%M')"
  [core]
    editor = code --wait  # или vim, nano
    autocrlf = input
    fsmonitor = true
  [push]
    default = simple
    followTags = true
  [pull]
    rebase = true
  [rerere]
    enabled = true
```

---

## 🧪 Чек-лист самостоятельной практики

- [ ] Создал репо, настроил `.gitignore` и `user.signingkey`
- [ ] Прошёл полный цикл: branch → commit → PR → squash merge
- [ ] Разрешил 2+ конфликта вручную и через `mergetool`
- [ ] Сделал `rebase -i`, объединил 3 коммита в 1
- [ ] Настроил CI с `GIT_DEPTH=1`, кэшем и path-based triggers
- [ ] Симулировал утечку секрета, очистил через `filter-repo`, проверил аудит
- [ ] Нашёл баг-коммит через `bisect run`
- [ ] Восстановил "потерянный" коммит через `reflog`
- [ ] Развернул GitOps-синхронизацию (ArgoCD/Flux) + откатил через `git revert`
- [ ] Настроил `pre-commit` с `gitleaks` и `terraform validate`

---

## ⚠️ Топ-5 ошибок в продакшене и как их избежать

| Ошибка | Последствия | Решение |
|:---|:---|:---|
| `git push --force` в `main` | Потеря чужих коммитов, сломанные CI-пайплайны, даунтайм | Branch Protection + `--force-with-lease` + ревью перед force |
| Игнорирование `.gitignore` | Утечка секретов, раздутый репо, медленный клон | `.gitignore` в корне репо + pre-commit scan + CI check |
| Долгоживущие ветки (>2 нед) | Merge hell, конфликты, блокировка релизов | Trunk-Based + feature flags + daily merge в `main` |
| Хранение `tfstate`/бинарников в Git | Утечка state, блокировки, 10x замедление | Remote backend + S3/Artifactory/Docker Registry |
| Деплой напрямую из CI без GitOps | Drift, невозможность аудита, "снежинки" | Pull-модель: ArgoCD/Flux синхронизирует кластер из Git |

---

