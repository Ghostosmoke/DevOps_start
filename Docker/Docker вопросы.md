# 📑 Оглавление для навигации

### 🟢 Junior

- [[#1. Чем контейнер отличается от виртуальной машины на уровне архитектуры?|1. Контейнер vs ВМ ⭐]]
- [[#2. Что такое Docker-образ и как устроены его слои?|2. Образы и слои]]
- [[#3. В чём разница между `docker run`, `docker start` и `docker restart`?|3. Жизненный цикл контейнера]]
- [[#4. Зачем нужен `.dockerignore` и что происходит, если его нет?|4. .dockerignore ⭐]]
- [[#5. Как проверить, что контейнер запущен и слушает нужный порт?|5. Проверка контейнера]]
- [[#6. Что делает команда `docker compose up -d` и зачем флаг `-d`?|6. Compose up -d]]

### 🟡 Middle

- [[#7. ⭐ Как работает `depends_on` в Compose и почему его недостаточно для гарантии готовности БД?|7. depends_on и readiness ⭐]]
- [[#8. ⭐ Объясните разницу между `volumes`, `bind mounts` и `tmpfs`. Когда что использовать?|8. Хранение данных: volumes ⭐]]
- [[#9. Почему в Dockerfile лучше использовать многоэтапную сборку (multi-stage)?|9. Multi-stage сборка]]
- [[#10. Как Docker обеспечивает сетевую изоляцию между контейнерами и как работает встроенный DNS?|10. Сеть и DNS в Docker]]
- [[#11. Что такое `HEALTHCHECK` и как оркестраторы реагируют на `unhealthy` статус?|11. HEALTHCHECK]]
- [[#12. Как ограничить потребление CPU и RAM для контейнера и какие механизмы ядра за этим стоят?|12. Лимиты ресурсов]]

### 🔴 Senior

- [[#13. ⭐ Опишите полный путь запуска контейнера от `docker run` до создания процесса в ядре Linux. Какую роль играют `containerd` и `runc`?|13. Архитектура запуска ⭐]]
- [[#14. Как работает кэширование слоёв в BuildKit? Как намеренно сломать кэш для конкретного этапа и зачем это нужно?|14. Кэширование BuildKit]]
- [[#15. Контейнер потребляет 100% CPU, не отвечает на healthcheck, но процесс в `top` внутри хоста показывает `S` (sleep). Как диагностировать?|15. Отладка зависшего контейнера]]
- [[#16. ⭐ В чём риски монтирования `/var/run/docker.sock` в CI-пайплайн? Какие альтернативы существуют для сборки и push образов?|16. Docker socket security ⭐]]
- [[#17. Как безопасно обновить образ в production-стеке Compose без даунтайма и потери состояния? Какие ограничения у подхода?|17. Zero-downtime деплой]]
- [[#18. ⭐ Как работают `capabilities`, `seccomp` и `AppArmor` в Docker? Как скомбинировать их для принципа наименьших привилегий?|18. Безопасность: capabilities, seccomp ⭐]]

---

# 🟢 Junior (базовое понимание)

### 1. ⭐ Чем контейнер отличается от виртуальной машины на уровне архитектуры?

1. **Кратко:** ВМ эмулирует железо и запускает гостевую ОС с отдельным ядром; контейнер делит ядро хоста, изолируя процессы через namespaces и cgroups.
2. **Подробно:** Гипервизор (Type 1/2) создаёт виртуальное железо для ВМ → гостевая ОС → приложения. Docker использует ядро хоста: каждый контейнер — изолированный процесс в своих namespaces (PID, NET, MNT) с лимитами через cgroups. Нет эмуляции = минимальный оверхед.
3. **DevOps-контекст:** Контейнеры стартуют за секунды, занимают МБ памяти — идеально для CI/раннеров, автоскейлинга, микросервисов. ВМ нужны для разнородных ОС (Windows на Linux-хосте) или требований к изоляции ядра.
4. **Команды:** `uname -a` (одно ядро на хосте), `lsns -t pid`, `docker info`, `cat /sys/fs/cgroup/docker/<id>/memory.max`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 2. Что такое Docker-образ и как устроены его слои?

1. **Кратко:** Образ — read-only шаблон из слоёв (layers), наложенных через UnionFS (overlay2); при запуске добавляется read-write слой.
2. **Подробно:** Каждая инструкция `FROM`, `RUN`, `COPY` создаёт новый слой с хешем. Слои кэшируются и переиспользуются между образами. При запуске создаётся тонкий writable-слой поверх read-only, куда пишутся изменения. Удаление файла в слое = создание "whiteout"-маркера.
3. **DevOps-контекст:** Понимание слоёв критично для оптимизации образов: порядок инструкций влияет на кэш, `RUN apt-get update && apt-get install` в одной строке уменьшает количество слоёв. `dive` помогает анализировать "вес" каждого слоя.
4. **Команды:** `docker history <image>`, `dive <image>`, `docker image inspect --format='{{.RootFS.Layers}}'`, `docker build --progress=plain`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 3. В чём разница между `docker run`, `docker start` и `docker restart`?

1. **Кратко:** `run` создаёт и запускает новый контейнер; `start` запускает остановленный; `restart` перезапускает (останавливает + запускает).
2. **Подробно:** `docker run` = `create` + `start`, принимает параметры (`-p`, `-v`, `-e`). `start/restart` используют сохранённую конфигурацию из `config.v2.json`. `restart` по умолчанию ждёт 10 секунд перед stop (`--time`), отправляя `SIGTERM` → `SIGKILL`.
3. **DevOps-контекст:** В orchestration важно понимать lifecycle: `create → start → stop → rm`. Compose управляет этим автоматически. В CI часто используют `run --rm` для одноразовых задач, чтобы не засорять список контейнеров.
4. **Команды:** `docker ps -a`, `docker inspect <id> | jq '.[0].State'`, `docker run --rm -it alpine sh`, `docker restart --time=0 <id>`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 4. ⭐ Зачем нужен `.dockerignore` и что происходит, если его нет?

1. **Кратко:** `.dockerignore` исключает файлы из build-контекста; без него весь каталог (`.git`, `node_modules`, секреты) отправляется демону.
2. **Подробно:** При `docker build` клиент архивирует текущую директорию и отправляет демону. Без `.dockerignore` в архив попадают: `.git/` (сотни МБ), `node_modules/`, логи, `.env` с секретами. Эти файлы могут попасть в слои образа, даже если не копируются явно.
3. **DevOps-контекст:** Критично для безопасности и скорости: утечка секретов в registry, раздувание кэша в CI, медленная сборка из-за большого контекста. Всегда добавляйте `**/.git`, `**/node_modules`, `**/*.log`, `secrets/`, `**/.env`.
4. **Команды:** `cat .dockerignore`, `docker build --progress=plain .` (показывает размер контекста), `du -sh .git node_modules`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 5. Как проверить, что контейнер запущен и слушает нужный порт?

1. **Кратко:** `docker ps`, `docker port <container>`, `ss -tulpn | grep <port>` на хосте.
2. **Подробно:** `docker ps` показывает статус и маппинг портов (`0.0.0.0:8080->80/tcp`). `docker port` выводит точный хост:контейнер маппинг. На хосте `ss`/`netstat` покажет, что процесс (`docker-proxy` или `containerd-shim`) действительно слушает порт.
3. **DevOps-контекст:** В CI перед интеграционными тестами проверяют readiness через `curl`/`wget` или `nc -z`. В healthchecks используют `test: ["CMD", "curl", "-f", "http://localhost:8080/health"]`.
4. **Команды:** `docker inspect -f '{{range .NetworkSettings.Ports}}{{.HostPort}}{{end}}' <id>`, `docker exec <id> ss -tln`, `curl -s http://localhost:8080/health`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 6. Что делает команда `docker compose up -d` и зачем флаг `-d`?

1. **Кратко:** Запускает стек сервисов из `docker-compose.yml` в фоновом режиме (`detached`).
2. **Подробно:** Compose создаёт сеть, volumes, затем контейнеры в порядке зависимостей. Без `-d` процесс блокирует терминал, стримит логи всех сервисов и завершается при `Ctrl+C` (останавливая контейнеры). `-d` возвращает управление сразу, контейнеры работают независимо.
3. **DevOps-контекст:** В CI/CD и production всегда используется `-d` + мониторинг через `logs`/`healthcheck`. Для отладки удобно запускать без `-d`, чтобы видеть логи в реальном времени.
4. **Команды:** `docker compose up -d`, `docker compose ps`, `docker compose logs -f <service>`, `docker compose down -v`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

# 🟡 Middle (применение, нюансы)

### 7. ⭐ Как работает `depends_on` в Compose и почему его недостаточно для гарантии готовности БД?

1. **Кратко:** `depends_on` гарантирует только порядок запуска контейнеров, но не готовность сервиса внутри (БД может быть `running`, но ещё не принимать соединения).
2. **Подробно:** Compose запускает зависимости перед зависимым сервисом, но не ждёт, пока приложение внутри контейнера станет доступным. Решение: `condition: service_healthy` + `HEALTHCHECK` в зависимом сервисе, который проверяет реальную готовность (напр. `pg_isready` для PostgreSQL).
3. **DevOps-контекст:** Без этого интеграционные тесты падают флэково: приложение стартует раньше, чем БД готова принимать соединения. В production это ведёт к ошибкам при деплое и перезапусках. Всегда используйте healthchecks для stateful-сервисов.
4. **Команды:** 
```yaml
# docker-compose.yml
services:
  db:
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
  app:
    depends_on:
      db: { condition: service_healthy }
```

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 8. ⭐ Объясните разницу между `volumes`, `bind mounts` и `tmpfs`. Когда что использовать?

1. **Кратко:** 
    - `volumes` — управляются Docker, хранятся в `/var/lib/docker/volumes`, переживают пересоздание контейнеров.
    - `bind mounts` — привязывают хост-путь, удобны для dev.
    - `tmpfs` — в RAM, очищаются при остановке.
2. **Подробно:** 
    - **Volumes**: изолированы от хоста, поддерживают драйверы (local, nfs, cloud), бэкапы через `docker run --volumes-from`.
    - **Bind mounts**: прямой доступ к хост-файлам, проблемы с правами на macOS/Windows (виртуализация), риск изменения хоста из контейнера.
    - **Tmpfs**: только в Linux, данные в памяти, не пишутся на диск — для секретов, сессий, кэша.
3. **DevOps-контекст:** Неправильный выбор ведёт к потере данных (забыли volume для БД), проблемам с правами в CI, или утечке чувствительной информации на диск. В production для данных — только volumes с бэкапами.
4. **Команды:** `docker volume create db-data`, `-v ./config:/app/config:ro` (bind, read-only), `--tmpfs /run:mode=1777`, `docker run --mount type=volume,source=db-data,target=/var/lib/postgresql/data`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 9. Почему в Dockerfile лучше использовать многоэтапную сборку (multi-stage)?

1. **Кратко:** Multi-stage позволяет использовать тяжелый образ для сборки (компиляторы, SDK), а в финальный образ копировать только бинарники.
2. **Подробно:** Первый `FROM builder` компилирует приложение со всеми зависимостями. Второй `FROM alpine` (или distroless) копирует только артефакты: `COPY --from=builder /app/bin ./bin`. Слои первого этапа не попадают в финальный образ, уменьшая его размер на 90%+.
3. **DevOps-контекст:** Резко уменьшает attack surface (нет компиляторов, шеллов в prod), ускоряет pull/deploy, экономит место в registry. Критично для безопасности: меньше пакетов = меньше CVE.
4. **Команды:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o bin/app

FROM alpine:3.19
COPY --from=builder /app/bin/app /usr/local/bin/
CMD ["app"]
```

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 10. Как Docker обеспечивает сетевую изоляцию между контейнерами и как работает встроенный DNS?

1. **Кратко:** Каждый network создаёт отдельный network namespace; контейнеры получают veth-пары, подключенные к bridge. Встроенный DNS (`127.0.0.11`) резолвит имена сервисов.
2. **Подробно:** При создании сети Docker настраивает `docker0` или custom bridge, выделяет подсеть. Каждый контейнер получает veth-пару: один конец в сети контейнера, другой — в bridge. Встроенный DNS слушает `127.0.0.11`, резолвит имена контейнеров/сервисов в их IP, обновляется при рестартах.
3. **DevOps-контекст:** Позволяет сервисам общаться по именам без хардкода IP — критично для динамического масштабирования. В Compose сервисы автоматически добавляются в DNS сети. Для изоляции создают отдельные networks (`frontend`, `backend`).
4. **Команды:** `docker network create app-net`, `docker network inspect app-net`, `nsenter -t <PID> -n ip a`, `docker exec <id> getent hosts db`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 11. Что такое `HEALTHCHECK` и как оркестраторы реагируют на `unhealthy` статус?

1. **Кратко:** `HEALTHCHECK` периодически выполняет команду внутри контейнера; статусы: `starting`, `healthy`, `unhealthy`.
2. **Подробно:** Инструкция в Dockerfile или Compose задаёт тест, интервал, таймаут, ретраи. При `unhealthy` Docker не убивает контейнер автоматически, но помечает его в `docker ps`. Оркестраторы (K8s, Swarm, Compose с `restart: on-failure`) используют статус для принятия решений: не направлять трафик, перезапустить, алертить.
3. **DevOps-контекст:** Обнаруживает "тихие" сбои: deadlocks, зависшие соединения, failed migrations, которые не крашат процесс, но ломают функциональность. Критично для readiness в CI и production.
4. **Команды:** 
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```
`docker inspect --format='{{.State.Health.Status}}' <id>`, `docker ps --filter "health=unhealthy"`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 12. Как ограничить потребление CPU и RAM для контейнера и какие механизмы ядра за этим стоят?

1. **Кратко:** Через `--cpus`, `--memory` (CLI) или `deploy.resources` (Compose). Под капотом: cgroups v2 `cpu.max`, `memory.max`.
2. **Подробно:** 
    - **CPU**: `--cpus=1.5` = CFS quota 150000/100000 микросекунд. Ядро троттлит процесс при превышении.
    - **Memory**: `--memory=512m` = hard limit на RSS + cache. При превышении — OOM Killer внутри cgroup.
    - **Swap**: `--memory-swap` контролирует лимит памяти+swap.
3. **DevOps-контекст:** Предотвращает noisy neighbor, starvation, OOM-краши хоста. Обязательно в multi-tenant CI/CD и shared хостах. В K8s это `resources.limits`, в Compose — `deploy.resources.limits`.
4. **Команды:** `docker stats`, `cat /sys/fs/cgroup/docker/<id>/memory.max`, `docker run --cpus=2 --memory=1g myapp`, `systemd-run --scope -p MemoryMax=512M docker run ...`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

# 🔴 Senior (архитектура, trade-offs, troubleshooting)

### 13. ⭐ Опишите полный путь запуска контейнера от `docker run` до создания процесса в ядре Linux. Какую роль играют `containerd` и `runc`?

1. **Кратко:** `docker run` → REST API к `dockerd` → `containerd` (gRPC) → `containerd-shim` → `runc` → namespaces/cgroups/rootfs → `execve` → PID 1 контейнера.
2. **Подробно:** 
    - `dockerd` принимает запрос, валидирует, отправляет задачу в `containerd` через gRPC.
    - `containerd` создаёт task, запускает `containerd-shim` (отвечает за мониторинг процесса, не умирает при рестарте dockerd).
    - `shim` вызывает `runc` (OCI runtime), который: создаёт namespaces, настраивает cgroups, монтирует rootfs из слоёв (overlay2), применяет seccomp/AppArmor, выполняет `execve` с процессом.
    - Процесс становится PID 1 в новом PID namespace.
3. **DevOps-контекст:** Разделение на `dockerd`/`containerd`/`runc` стандартизировано OCI. K8s использует `containerd` напрямую (CRI). Понимание цепочки критично для отладки startup failures, seccomp deny, namespace conflicts. `ctr` — low-level CLI для containerd.
4. **Команды:** `ctr` (containerd CLI), `runc spec`, `docker info --format '{{.ServerVersion}}'`, `strace -p $(pgrep containerd-shim)`, `ls -l /proc/<PID>/ns/`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 14. Как работает кэширование слоёв в BuildKit? Как намеренно сломать кэш для конкретного этапа и зачем это нужно?

1. **Кратко:** BuildKit кэширует слои по хэшу инструкций и содержимого файлов; изменение `COPY` инвалидирует кэш этого и последующих этапов.
2. **Подробно:** 
    - Кэш строится на графе зависимостей: хэш инструкции + хэши входных файлов.
    - `--no-cache` отключает полностью. `ARG CACHEBUST=1` + `--build-arg CACHEBUST=$(date +%s)` ломает кэш намеренно.
    - `--mount=type=cache` кэширует директории (apt, pip, go) между сборками, не создавая новых слоёв.
3. **DevOps-контекст:** В CI нужно кэшировать зависимости для скорости, но сбрасывать кэш при изменении `go.mod`/`package.json`/`requirements.txt`. Иначе — stale deps или медленные билды. BuildKit + `--mount=type=cache` ускоряет сборку в 5-10 раз.
4. **Команды:**
```dockerfile
# Dockerfile
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app/bin/app

# Build
docker build --build-arg BUILD_DATE=$(date) --progress=plain .
```
`docker buildx build --cache-from=type=registry,ref=myapp:cache --cache-to=type=registry,ref=myapp:cache,mode=max`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 15. Контейнер потребляет 100% CPU, не отвечает на healthcheck, но процесс в `top` внутри хоста показывает `S` (sleep). Как диагностировать?

1. **Кратко:** `S` (sleep) + 100% CPU обычно означает busy-wait в user-space или бесконечный цикл с `nanosleep(0)`; ядро не видит I/O wait, но процесс не прогрессирует.
2. **Подробно:** 
    - `docker top <id>` → найти PID на хосте.
    - `strace -p <PID> -T -tt` покажет зацикленный syscall (напр. `futex` с таймаутом 0).
    - `gdb -p <PID> -ex "thread apply all bt"` даст стек вызовов всех потоков.
    - `perf record -g -p <PID>` покажет hot functions, `perf report` визуализирует.
3. **DevOps-контекст:** В production такие процессы блокируют воркеры, исчерпывают соединения, но не падают. Требуют дамп стека и graceful restart с сохранением логов. Автоматизируйте сбор дампов при `unhealthy` статусе.
4. **Команды:** `kill -SIGQUIT <PID>` (если приложение поддерживает), `gdb -batch -p <PID> -ex "bt full"`, `perf top -p <PID>`, `docker restart --time=30 <id>`, `docker logs --tail=1000 <id>`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 16. ⭐ В чём риски монтирования `/var/run/docker.sock` в CI-пайплайн? Какие альтернативы существуют для сборки и push образов?

1. **Кратко:** Монтирование сокета даёт полный доступ к Docker API; злоумышленник может запустить `--privileged` контейнер, получить root хоста, установить бэкдор.
2. **Подробно:** Docker socket — это UNIX socket с правами `root:docker`. Любой процесс внутри контейнера может: создать привилегированный контейнер, смонтировать `/` хоста, прочитать `/etc/shadow`, запустить майнер. Это полный компромисс хоста.
3. **DevOps-контекст:** В CI безопасность пайплайна = безопасность хоста. Нарушение компрометирует весь кластер/репозиторий. Альтернативы: `kaniko`/`buildah` (сборка без демона), `docker buildx` с удалённым билд-сервером, rootless Docker, `docker-socket-proxy` с ограниченным API.
4. **Команды:** 
```bash
# Опасно:
docker run -v /var/run/docker.sock:/var/run/docker.sock ...

# Безопасно (kaniko в K8s):
/kaniko/executor --context . --destination registry/app:tag

# Socket proxy с ограничениями:
docker run -v /var/run/docker.sock:/var/run/docker.sock \
  -e ALLOW_START=1 -e ALLOW_STOP=1 \
  tecnativa/docker-socket-proxy
```

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 17. Как безопасно обновить образ в production-стеке Compose без даунтайма и потери состояния? Какие ограничения у подхода?

1. **Кратко:** `docker compose up -d --force-recreate --remove-orphans` создаёт новые контейнеры, останавливает старые; даунтайм ~время рестарта сервиса.
2. **Подробно:** Compose по умолчанию останавливает старый контейнер перед запуском нового. Для минимизации простоя: 1) запустить новый с другим портом/именем, 2) дождаться healthcheck, 3) переключить reverse proxy (Traefik/Caddy), 4) остановить старый. Stateful сервисы (БД) требуют ручного дампа/миграции, откат сложен.
3. **DevOps-контекст:** Для zero-downtime на одном хосте используют reverse proxy + healthchecks + parallel запуск + graceful drain. В enterprise мигрируют на K8s (rolling updates) или Swarm (native rolling). Compose не подходит для сложных стратегий деплоя.
4. **Команды:**
```bash
# Базовый апдейт:
docker compose up -d --force-recreate app

# С параллельным запуском и proxy:
docker compose -f compose.prod.yml -p app-v2 up -d
# ждать healthcheck
# переключить proxy на app-v2
docker compose -p app-v1 down
```

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---

### 18. ⭐ Как работают `capabilities`, `seccomp` и `AppArmor` в Docker? Как скомбинировать их для принципа наименьших привилегий?

1. **Кратко:** 
    - `capabilities` делят root на гранулы (`CAP_NET_BIND_SERVICE`).
    - `seccomp` фильтрует syscalls (разрешает/блокирует).
    - `AppArmor`/`SELinux` ограничивают доступ к ФС/сети на уровне MAC.
2. **Подробно:** 
    - Docker по умолчанию дропает ~40 опасных capabilities и использует `seccomp` профиль, блокирующий `ptrace`, `mount`, `unshare` и др.
    - `cap_drop: ALL` убирает все capabilities, `cap_add` возвращает нужные.
    - `security_opt: ["seccomp=custom.json", "apparmor=docker-custom"]` применяет кастомные политики.
3. **DevOps-контекст:** Принцип least privilege: контейнер должен видеть только то, что ему нужно. Снижает blast radius при эксплуатации уязвимостей. В CI сканируйте образы на наличие `--privileged`, `cap_add=SYS_ADMIN`, `seccomp=unconfined`.
4. **Команды:**
```yaml
# docker-compose.yml
services:
  app:
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE, CHOWN]
    security_opt:
      - seccomp=./seccomp-profile.json
      - apparmor=docker-default
    read_only: true
    tmpfs:
      - /tmp:mode=1777
```
`docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE --security-opt seccomp=/profile.json myapp`, `grep Cap /proc/<PID>/status`.

[[#📑 Оглавление для навигации|↑ К оглавлению]]

---
