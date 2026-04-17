Подтверждаю тему: **Docker Practice (Hands-On Deep Dive)**. 
В этом гайде мы соберём production-ready стек на основе Nginx, от локальной разработки до CI/CD и production-деплоя. Фокус на реальной инженерии: безопасность, наблюдаемость, изоляция, отладка и автоматизация.

---

### 1️⃣ БЛОК ТЕОРИИ (Практические концепции Docker)

#### 🎯 Что значит "практика в Docker" для DevOps?
**Простыми словами:** Это не умение написать `FROM nginx`, а способность создать воспроизводимый, безопасный и наблюдаемый образ, который одинаково ведёт себя на ноутбуке разработчика, в CI-раннере и в production. Это знание того, как слои влияют на размер и кэш, как сеть резолвит имена, как ограничения cgroups влияют на latency, и как отлаживать контейнер без SSH.

**Архитектура рабочего процесса:**
```
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Build Context  │───▶│  Docker Engine   │───▶│  Container       │
│ (.dockerignore) │    │ (BuildKit)       │    │ (runc + cgroups) │
└─────────────────┘    └────────┬─────────┘    └────────┬─────────┘
                                │                       │
                     ┌──────────▼──────────┐   ┌────────▼─────────┐
                     │  Image Registry     │   │  Runtime State   │
                     │ (layers + metadata) │   │ (ns + fs + net)  │
                     └─────────────────────┘   └──────────────────┘
```
- **Слои (Overlay2):** Каждый `COPY`/`RUN` = read-only слой. При запуске накладывается тонкий `diff` слой. Копирование больших файлов на поздних этапах дублирует данные → образ распухает.
- **Изоляция:** `PID` (своя таблица процессов), `NET` (свой стек, veth-пары), `MNT` (своя точка монтирования), `USER` (маппинг UID). Без `USER` namespace контейнер видит реальный root хоста.
- **Сеть:** В user-defined bridge-сетях Docker поднимает встроенный DNS (`127.0.0.11`). Контейнеры резолвят друг друга по именам сервисов, а не IP.
- **Storage:** `volumes` (управляются Docker, переживают recreation), `bind mounts` (хост-путь, удобно для dev), `tmpfs` (RAM, очищается при стопе).

#### 🔹 Реальные кейсы и антипаттерны
| Ситуация | Что происходит | Практическое решение |
|----------|----------------|----------------------|
| **`latest` в production** | Непредсказуемые обновления, слома кэша, сложно откатить | Фиксируйте теги (`nginx:1.25-alpine`) или используйте digest (`nginx@sha256:...`) |
| **Контейнер "умирает" без логов** | Приложение крашится до инициализации логов, или OOM убивает процесс | `HEALTHCHECK`, `dmesg -T | grep -i oom`, `docker inspect --format='{{.State.OOMKilled}}'` |
| **Медленная сборка в CI** | Каждый билд качает зависимости заново | `--mount=type=cache,target=/var/cache/apt`, `BUILDKIT_INLINE_CACHE=1` |
| **`docker compose up` роняет сервисы** | Нет healthcheck, `depends_on` только ждёт `running` | `condition: service_healthy`, ретраи в приложении, `restart: on-failure` |

#### 🔗 Связь с экосистемой
- **CI/CD:** `docker buildx bake` параллелит этапы, `kaniko`/`buildah` собирают без демона, Trivy сканирует CVE.
- **Kubernetes:** `docker compose` → `kompose` или ручная миграция в `Deployments`/`Services`. K8s использует `containerd` напрямую.
- **Security:** `docker scout`, `seccomp` профили, `cap_drop: ALL`, `read_only: true`, `no-new-privileges`.

#### 🔑 Ключевые моменты
- Образы immutable. Меняете конфиг → новый образ или bind mount в dev.
- Сеть и DNS работают только в пользовательских bridge-сетях.
- `read_only: true` + `tmpfs`/`volumes` = production standard.
- Graceful shutdown требует обработки `SIGTERM` и настройки `TimeoutStopSec`.

#### ⚠️ Частые ошибки
- Игнорирование `.dockerignore` → утечка `.git`, `node_modules`, секретов.
- Запуск от `root` → повышенный blast radius при уязвимости.
- Отсутствие `HEALTHCHECK` → оркестратор не видит deadlocks.
- `depends_on` без `condition: service_healthy` → race conditions при старте.
- Монтирование `/var/run/docker.sock` в CI → полный root-доступ к хосту.

---

### 2️⃣ БЛОК ПРАКТИКИ (Nginx Site → Production Stack)

#### 📁 Структура проекта
```
docker-nginx-practice/
├── Dockerfile
├── docker-compose.yml
├── nginx.conf          # Кастомная конфигурация
├── html/
│   └── index.html      # Статический сайт
├── .dockerignore
└── scripts/
    └── healthcheck.sh  # Надёжный healthcheck
```

#### 🛠 Конфигурационные файлы
**`html/index.html`**
```html
<!DOCTYPE html>
<html lang="ru">
<head><meta charset="UTF-8"><title>DevOps Practice Site</title></head>
<body style="font-family:system-ui; padding:2rem;">
  <h1>✅ Nginx Container Working</h1>
  <p>Hostname: <span id="host">unknown</span></p>
  <p>Uptime: <span id="time">unknown</span></p>
  <script>
    document.getElementById('host').textContent = window.location.hostname;
    document.getElementById('time').textContent = new Date().toISOString();
  </script>
</body>
</html>
```

**`nginx.conf`** (Production-ready override)
```nginx
user  nginx;
worker_processes  auto;
error_log  /var/log/nginx/error.log warn;
pid        /tmp/nginx.pid;

events { worker_connections 1024; use epoll; }
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    tcp_nopush    on;
    keepalive_timeout  65;
    gzip          on;
    gzip_types    text/plain application/json text/css application/javascript;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }

        location / {
            try_files $uri $uri/ =404;
            add_header X-Content-Type-Options nosniff;
            add_header X-Frame-Options DENY;
        }
    }
}
```

**`Dockerfile`**
```dockerfile
# Фиксируем версию. Не используйте latest!
FROM nginx:1.25-alpine

# Копируем только необходимое
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/
COPY scripts/healthcheck.sh /scripts/healthcheck.sh

# Права и безопасность
RUN chmod +x /scripts/healthcheck.sh && \
    chown -R nginx:nginx /usr/share/nginx/html && \
    # Очистка кэша Alpine в том же слое для минимизации размера
    rm -rf /var/cache/apk/* /tmp/* /var/tmp/*

# Healthcheck (альтернатива HEALTHCHECK в compose)
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/scripts/healthcheck.sh"]

EXPOSE 80
# Запуск в foreground (Docker требует этого для отслеживания PID 1)
CMD ["nginx", "-g", "daemon off;"]
```

**`scripts/healthcheck.sh`**
```bash
#!/bin/sh
set -e
# Проверяем ответ, а не только exit code curl
RESP=$(curl -sf -o /dev/null -w "%{http_code}" http://localhost/health)
[ "$RESP" = "200" ] || exit 1
```

**`.dockerignore`**
```
.git
.gitignore
*.md
docker-compose.yml
scripts/
# Секреты и кэши
.env
node_modules/
.DS_Store
```

**`docker-compose.yml`** (v2, Production-ready)
```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: practice-nginx
    ports:
      - "8080:80"
    # Read-only FS + временные директории в RAM
    read_only: true
    tmpfs:
      - /tmp
      - /var/cache/nginx
      - /var/log/nginx
    # Ресурсы (cgroups v2)
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 128M
        reservations:
          cpus: "0.1"
          memory: 64M
    # Security hardening
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Нужно только если порт < 1024, но оставим для наглядности
    security_opt:
      - no-new-privileges:true
    # Healthcheck дублируется из Dockerfile для совместимости
    healthcheck:
      test: ["CMD", "/scripts/healthcheck.sh"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    networks:
      - frontend

networks:
  frontend:
    driver: bridge
```

#### 🚀 Сценарии запуска и отладки

**1. Локальная разработка (горячая перезагрузка конфигов)**
```bash
# Заменяем volume в compose или запускаем вручную:
docker run --rm -d \
  -p 8080:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  --name nginx-dev \
  nginx:1.25-alpine

# Перезагрузка без рестарта контейнера:
docker exec nginx-dev nginx -s reload
curl -I http://localhost:8080
```

**2. Production запуск через Compose**
```bash
# Валидация без запуска
docker compose config --quiet
# Сборка и запуск в фоне
docker compose up -d --build
# Проверка состояния
docker compose ps
docker compose logs -f --tail=20 web
# Проверка healthcheck
docker inspect --format='{{.State.Health.Status}}' practice-nginx
```

**3. Отладка и диагностика**
```bash
# 1. Посмотреть, что видит процесс внутри
docker exec practice-nginx ps aux
docker exec practice-nginx cat /proc/mounts | grep tmpfs

# 2. Проверить сеть и DNS
docker exec practice-nginx apk add --no-cache curl  # если alpine
docker exec practice-nginx curl -v http://localhost/health

# 3. Почему контейнер рестартится?
docker inspect practice-nginx --format='{{.State.Restarting}} {{.State.Error}}'
docker compose logs --tail=50 web | grep -iE 'error|fatal|panic'

# 4. Лимиты cgroups (хост-уровень)
CGROUP_ID=$(docker inspect -f '{{.State.Pid}}' practice-nginx)
cat /sys/fs/cgroup/docker/${CGROUP_ID}/memory.max
cat /sys/fs/cgroup/docker/${CGROUP_ID}/cpu.max
```

#### 📝 Best Practices Flags & Env
| Флаг / Переменная | Назначение |
|-------------------|------------|
| `DOCKER_BUILDKIT=1` | Включает современный builder (кэш, параллелизм, секреты) |
| `--init` / `init: true` | Запускает `tini` как PID 1 → корректный forward сигналов + reap zombie |
| `--read-only` + `tmpfs` | Запрещает запись на корневую FS → защита от ransomware/заполнения диска |
| `--security-opt no-new-privileges:true` | Блокирует `setuid`/`setgid` внутри контейнера |
| `COMPOSE_PROJECT_NAME=prod` | Изолирует стэки при нескольких окружениях на одном хосте |

---

