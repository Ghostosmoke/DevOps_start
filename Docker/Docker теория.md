Подтверждаю тему: **Docker & Docker Compose**. 
Это фундамент контейнеризации в DevOps. В этом гайде я разберу архитектуру, механизмы изоляции, storage/networking, современные best practices для CI/CD и production, а также подготовлю вас к собеседованиям уровня Middle/Senior. Если потребуется углубление в `Docker Security`, `BuildKit/Bake`, `Docker-in-Docker vs CI runners` или `Migration to Kubernetes` — дайте знать, сделаем отдельную часть.

---

### 1️⃣ БЛОК ТЕОРИИ

#### 🌍 Суть темы: от простого к глубокому
**Простыми словами:** Docker упаковывает приложение со всеми зависимостями (библиотеки, конфиги, рантайм) в стандартизированный образ. При запуске образ становится контейнером — изолированным процессом, который ведёт себя одинаково на ноутбуке, CI-сервере и продакшене. Docker Compose декларативно описывает, как несколько контейнеров работают вместе.

**Архитектура и принципы работы:**
```
┌─────────────────────────────────────────────────────────────┐
│                      Docker CLI / Compose                   │
│  (docker run, docker compose up, docker buildx)             │
└───────────────────────┬─────────────────────────────────────┘
                        ▼ (REST API / gRPC)
┌─────────────────────────────────────────────────────────────┐
│                    Docker Daemon (dockerd)                  │
│  • Управляет образами, контейнерами, сетями, томами         │
│  • Делегирует выполнение контейнеров в containerd           │
│  • Запускает BuildKit для сборки образов                    │
└───────────────────────┬─────────────────────────────────────┘
                        ▼ (containerd API)
┌─────────────────────────────────────────────────────────────┐
│                    containerd                               │
│  • OCI-совместимый рантайм. Управляет жизненным циклом      │
│    контейнеров, образами, снимками, метриками               │
└───────────────────────┬─────────────────────────────────────┘
                        ▼ (runc)
┌─────────────────────────────────────────────────────────────┐
│                    runc (low-level runtime)                 │
│  • Запускает процесс, настраивает namespaces, cgroups,      │
│    rootfs, монтирует тома, применяет seccomp/AppArmor       │
│  • Использует Linux-фичи: PID/NET/MNT/USER namespaces,      │
│    cgroups v2, overlay2, seccomp, capabilities              │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Linux Kernel (Host)                      │
└─────────────────────────────────────────────────────────────┘
```
- **Слои образа (Overlay2):** Каждый `RUN`/`COPY` в Dockerfile создаёт read-only слой. При запуске поверх накладывается тонкий read-write слой. Copy-on-Write (CoW) экономит место и ускоряет старт.
- **BuildKit:** Современный движок сборки. Поддерживает параллельные этапы, кэширование зависимостей (`RUN --mount=type=cache`), секретов, мультиархитектурные образы.
- **Docker Compose v2:** Переписан на Go, встроен в Docker CLI (`docker compose`). Поддерживает `profiles`, `include`, `secrets`, `configs`, `depends_on` с healthchecks.

#### 🌐 Сетевая модель
| Драйвер | Изоляция | DNS | Когда использовать |
|---------|----------|-----|-------------------|
| `bridge` (default) | ✅ Да | ✅ Встроенный (`127.0.0.11`) | Локальная разработка, изолированные сервисы |
| `host` | ❌ Нет (共享 хост) | ❌ Нет хостов | Высокопроизводительные сетевые демоны, метрики |
| `none` | ✅ Полная | ❌ Нет | Изолированные задачи, ручная настройка сети |
| `macvlan`/`ipvlan` | ✅ L2/L3 | ✅ Внешний DHCP | Интеграция с legacy-сетью, bare-metal замена ВМ |

- **Встроенный DNS (`127.0.0.11`):** Работает только в пользовательских bridge-сетях. Контейнеры резолвят друг друга по имени сервиса, автоматически балансируя нагрузку между репликами.

#### 📦 Реальные кейсы и антипаттерны
| Ситуация | Что происходит | Как избежать |
|----------|----------------|--------------|
| **Образ весит 2GB+** | Включены dev-зависимости, кэш пакетов, многослойные `RUN` | Multi-stage build, `--mount=type=cache`, очистка кэша в том же `RUN`, `alpine`/`distroless` base |
| **Контейнер не останавливается gracefully** | Приложение не обрабатывает `SIGTERM`, zombie-процессы | `STOPSIGNAL SIGTERM`, `init: true` (tini), корректная обработка сигналов в коде |
| **`docker-compose up` в production** | Compose не управляет оркестрацией, нет самовосстановления, нет rolling updates | Использовать Compose только для dev/stage. В prod: K8s, ECS, Nomad или systemd-юниты для одиночных сервисов |
| **Проброс Docker Socket в CI** (`-v /var/run/docker.sock:/var/run/docker.sock`) | Дает root-доступ к хосту. Уязвимость контейнер-эскейпа | Использовать `docker-outside-of-docker` с ограниченным пользователем, `kaniko`, `buildah`, или rootless Docker |

#### 🔗 Связь с экосистемой
- **CI/CD:** GitHub Actions / GitLab CI используют кэширование слоёв, `docker buildx bake`, параллельную сборку.
- **Kubernetes:** K8s использует `containerd` напрямую, без `dockerd`. Docker Compose → `kind-compose` или `kompose` для миграции.
- **Security:** Trivy/Grype сканируют образы, `docker scout` анализирует CVE, `seccomp`/`AppArmor` профили ограничивают syscalls.
- **Monitoring:** `cadvisor` + `node_exporter` собирают метрики cgroups/overlay2. Логирование через `json-file`, `syslog`, `fluentd` драйверы.

#### 🔑 Ключевые моменты
- Контейнеры ≠ ВМ. Они делят ядро хоста, изоляция через namespaces + cgroups.
- Образы immutable и layers-based. Кэширование слоёв ускоряет CI.
- Compose — инструмент декларативной оркестровки на одном хосте. Не заменяет K8s в production.
- Security-by-default: не root, read-only rootfs, drop capabilities, `no-new-privileges`, healthchecks.

#### ⚠️ Частые ошибки
- Запуск от `root` без `USER` директивы.
- Игнорирование `.dockerignore` → огромный контекст сборки, утечка секретов.
- `depends_on` без `condition: service_healthy` → сервис стартует до готовности БД.
- Отсутствие `HEALTHCHECK` → оркестратор не видит deadlocks.
- Использование `latest` тегов → недетерминированные сборки, слома кэша.

---

### 2️⃣ БЛОК ПРАКТИКИ

#### 🔧 Dockerfile (Production Best Practices)
```dockerfile
# 1. Этап сборки зависимостей и кода
FROM golang:1.22-alpine AS builder
WORKDIR /app
# Кэшируем go модули отдельно от кода
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .
# Собираем статический бинарник, отключаем CGO для портабельности
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server

# 2. Минимальный финальный образ
FROM alpine:3.19
# Создаём непривилегированного пользователя
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
# Копируем только бинарник и конфиги
COPY --from=builder --chown=appuser:appgroup /app/server ./
COPY config.yml ./config.yml
# Переключаемся на непривилегированного пользователя
USER appuser
# Healthcheck для оркестрации
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1
# Запуск (обрабатывает сигналы корректно)
ENTRYPOINT ["/app/server"]
```

#### 🐙 docker-compose.yml (v2 Syntax)
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      # Включаем BuildKit кэш для зависимостей
      args:
        BUILDKIT_INLINE_CACHE: 1
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      # Секреты монтируются как файлы, не передаются через env
    secrets:
      - db_password
    volumes:
      - app_logs:/var/log/app
      - /etc/localtime:/etc/localtime:ro  # Синхронизация времени
    # Ограничения ресурсов (cgroups)
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    # Read-only rootfs + временная ФС для логов/кэшей
    read_only: true
    tmpfs:
      - /tmp
      - /var/cache
    # Автоматический рестарт только при сбоях
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    # Сеть для DNS-резолвинга
    networks:
      - backend
    # Убираем лишние capabilities, оставляем только необходимые
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

secrets:
  db_password:
    file: ./secrets/db_pass.txt

volumes:
  app_logs:
  pg_data:

networks:
  backend:
    driver: bridge
    # Включаем встроенный DNS
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

#### 🚀 Сценарии запуска
```bash
# 📍 Локально: быстрый старт с пересборкой и очисткой
docker compose up --build -V  # -V: удаляет анонимные тома при recreate

# 🛠 CI: кэширование слоёв, параллельная сборка, экспорт артефакта
docker buildx create --use --name ci-builder
docker buildx build --cache-from=type=local,src=/tmp/.buildx-cache \
                    --cache-to=type=local,dest=/tmp/.buildx-cache,mode=max \
                    --push -t registry.example.com/app:${CI_COMMIT_SHA} .

# 🌍 Production: проверка конфига, запуск в фоне, просмотр логов
docker compose config --quiet  # Валидация без запуска
docker compose up -d
docker compose logs -f --tail=50 app  # Живые логи с контекстом
docker compose top app                # Процессы внутри контейнера
docker compose ps --format json       # Для парсинга в скриптах
```

#### 📝 Best Practices Flags & Env
| Флаг / Переменная | Назначение |
|-------------------|------------|
| `--init` / `init: true` | Запускает `tini` как PID 1 для корректного forward сигналов и reap zombie |
| `--security-opt no-new-privileges:true` | Запрещает `setuid`/`setgid` внутри контейнера |
| `DOCKER_BUILDKIT=1` | Включает современный builder (по умолч. в Docker 23+) |
| `COMPOSE_PROJECT_NAME=myenv` | Изолирует стэки при нескольких окружениях на одном хосте |
| `--pull always` / `image: tag@sha256:...` | Гарантирует детерминированную версию базового образа |

---

