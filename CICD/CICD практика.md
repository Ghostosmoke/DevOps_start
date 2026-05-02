Отлично! 🎯 Делаем **практический минимум по DevOps** в том же формате: оглавление с навигацией, уровни сложности, чёткая структура и акцент на **реальные задачи, команды и сценарии**.

---

# 📑 Оглавление: Практика DevOps

### 🟢 Junior (базовые навыки)

- [[#1. Как быстро проверить, почему под в K8s не запускается?|1. Debug pod startup]]
- [[#2. Как написать минимальный Dockerfile для Python/Node.js приложения?|2. Базовый Dockerfile]]
- [[#3. Как найти, какой процесс занимает место на диске или память?|3. Диагностика ресурсов]]
- [[#4. Как проверить доступность порта и сетевого соединения?|4. Сетевая отладка]]
- [[#5. Как посмотреть логи приложения в контейнере и отфильтровать ошибки?|5. Работа с логами]]
- [[#6. Как создать и применить манифест Kubernetes из шаблона?|6. K8s YAML basics]]
- [[#7. Как написать простой bash-скрипт для автоматизации рутины?|7. Bash-автоматизация]]
- [[#8. Как добавить переменные окружения в деплоймент и проверить их?|8. Env variables в K8s]]
- [[#9. Как проверить, что приложение отвечает на health check?|9. Health check curl]]
- [[#10. Как откатить деплой в Kubernetes, если что-то пошло не так?|10. Rollback deployment]]

### 🟡 Middle (применение в реальных сценариях)

- [[#11. ⭐ Как отладить CrashLoopBackOff в поде: пошаговый алгоритм?|11. Debug CrashLoop ⭐]]
- [[#12. Как написать Terraform-модуль с переменными и валидацией?|12. Terraform модули]]
- [[#13. Как настроить Prometheus-алерт на рост ошибок в приложении?|13. Prometheus alerting]]
- [[#14. Как реализовать rolling update с проверкой готовности?|14. Rolling update K8s]]
- [[#15. Как отладить NetworkPolicy: почему трафик блокируется?|15. Debug NetworkPolicy]]
- [[#16. Как параметризовать Helm-чарт для разных окружений?|16. Helm values management]]
- [[#17. Как собрать и отправить логи в Loki/ELK с парсингом?|17. Log aggregation]]
- [[#18. Как безопасно добавить секрет в пайплайн и приложение?|18. Secrets management]]
- [[#19. Как настроить multi-stage пайплайн с тестами и деплоем?|19. CI/CD pipeline stages]]
- [[#20. Как настроить Horizontal Pod Autoscaler по CPU и кастомным метрикам?|20. HPA setup]]

### 🔴 Senior (архитектура и сложные сценарии)

- [[#21. ⭐ Как спроектировать и протестировать disaster recovery процедуру?|21. DR runbook ⭐]]
- [[#22. Как настроить GitOps-синхронизацию с авто-исправлением дрейфа?|22. GitOps ArgoCD]]
- [[#23. Как инструментировать приложение для distributed tracing?|23. OpenTelemetry setup]]
- [[#24. Как написать ValidatingAdmissionWebhook для политик безопасности?|24. K8s webhooks]]
- [[#25. Как проанализировать и оптимизировать затраты на облачную инфраструктуру?|25. Cost optimization]]
- [[#26. Как внедрить подписанные артефакты и проверку целостности (Sigstore)?|26. Supply chain security]]
- [[#27. Как создать Operator для автоматического восстановления состояния?|27. Kubernetes Operator]]
- [[#28. Как безопасно провести chaos-эксперимент в staging/prod?|28. Chaos engineering]]
- [[#29. Как управлять конфигурацией для 10+ кластеров без дублирования?|29. Multi-cluster config]]
- [[#30. ⭐ Как провести постмортем инцидента и превратить его в улучшения?|30. Incident postmortem ⭐]]

---

# 🟢 Junior (базовые навыки)

### 1. Как быстро проверить, почему под в K8s не запускается?

1. **Кратко:** Использовать `kubectl describe`, `kubectl logs`, проверить события и статусы.
2. **Подробно:**
    ```bash
    # 1. Проверить статус пода
    kubectl get pod <pod-name> -o wide
    
    # 2. Посмотреть события (самое важное!)
    kubectl describe pod <pod-name> | grep -A10 Events
    
    # 3. Проверить логи контейнера
    kubectl logs <pod-name> [-c <container-name>]
    
    # 4. Если под в Pending — проверить ресурсы и ноды
    kubectl describe node <node-name> | grep -A5 Allocatable
    kubectl get events --field-selector involvedObject.name=<pod-name>
    ```
    **Типичные причины:** ImagePullBackOff (неверный образ/секрет), CrashLoopBackOff (ошибка в приложении), Pending (нехватка ресурсов), CreateContainerConfigError (проблема с ConfigMap/Secret).
3. **DevOps-контекст:** В пайплайне можно добавить автоматическую диагностику: если деплой фейлится — собрать `describe` и `logs` в артефакт для анализа.
4. **Команды/Инструменты:** `kubectl describe`, `kubectl logs -f --tail=100`, `k9s` для интерактивного просмотра, `stern` для логов нескольких подов.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 2. Как написать минимальный Dockerfile для Python/Node.js приложения?

1. **Кратко:** Использовать multi-stage сборку, небезопасный пользователь, кэширование зависимостей.
2. **Подробно:**
    ```dockerfile
    # Python пример (multi-stage)
    FROM python:3.11-slim as builder
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --user --no-cache-dir -r requirements.txt
    
    FROM python:3.11-slim
    WORKDIR /app
    COPY --from=builder /root/.local /root/.local
    COPY . .
    ENV PATH=/root/.local/bin:$PATH
    USER nobody  # не запускать от root!
    CMD ["python", "app.py"]
    ```
    ```dockerfile
    # Node.js пример
    FROM node:18-alpine
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci --only=production  # кэш зависимостей
    COPY . .
    USER node  # небезопасный пользователь
    EXPOSE 3000
    CMD ["node", "server.js"]
    ```
3. **DevOps-контекст:** Multi-stage уменьшает размер образа (нет компиляторов в runtime). `npm ci` и `--no-cache-dir` ускоряют сборку в CI. `USER nobody` снижает риски при уязвимостях.
4. **Команды/Инструменты:** `docker build -t app:$GIT_SHA --build-arg VERSION=$VERSION .`, `docker scan app:$GIT_SHA`, `dive app:$GIT_SHA` для анализа слоёв.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 3. Как найти, какой процесс занимает место на диске или память?

1. **Кратко:** Использовать `du`, `ncdu` для диска; `top`, `ps`, `smem` для памяти.
2. **Подробно:**
    ```bash
    # Диск: найти большие директории
    du -h /var/log | sort -rh | head -10
    ncdu /  # интерактивный просмотр (установить: apt install ncdu)
    
    # Память: процессы по потреблению
    ps aux --sort=-%mem | head -10
    smem -r -k  # реальное использование (учитывает shared memory)
    
    # CPU: нагрузка в реальном времени
    top -c  # или htop, аттач к процессу: strace -p <pid>
    
    # В контейнере: ограничения cgroups
    cat /sys/fs/cgroup/memory.max  # лимит памяти
    cat /sys/fs/cgroup/cpu.max     # лимит CPU
    ```
3. **DevOps-контекст:** В K8s эти данные доступны через `kubectl top pod/node`. Для алертинга: Prometheus node_exporter собирает эти метрики. Важно отличать RSS от VSZ, учитывать кэш страницы.
4. **Команды/Инструменты:** `kubectl top pod -l app=myapp`, `htop`, `iotop`, `pidstat -d 1`, `promql: container_memory_usage_bytes`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 4. Как проверить доступность порта и сетевого соединения?

1. **Кратко:** Использовать `curl`, `nc`, `telnet`, `ss` для проверки L4/L7.
2. **Подробно:**
    ```bash
    # Проверка TCP-порта (L4)
    nc -zv example.com 443
    timeout 2 bash -c '</dev/tcp/example.com/443' && echo "OK"
    
    # Проверка HTTP/HTTPS (L7)
    curl -I https://example.com/health  # только заголовки
    curl -f http://localhost:8080/ready || echo "Not ready"
    
    # DNS-разрешение
    dig +short example.com
    nslookup example.com
    
    # Кто слушает порт локально
    ss -tlnp | grep :8080
    lsof -i :8080
    ```
3. **DevOps-контекст:** В K8s для проверки связности между подами: `kubectl exec -it <pod> -- nc -zv <service> <port>`. Для readiness/liveness проб: использовать `httpGet` или `exec` с `curl`/`grpc_health_probe`.
4. **Команды/Инструменты:** `curl -v --connect-timeout 5`, `mtr example.com` (трассировка), `kubectl exec -it pod -- /bin/sh`, `grpc_health_probe -addr=:50051`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 5. Как посмотреть логи приложения в контейнере и отфильтровать ошибки?

1. **Кратко:** `kubectl logs` с флагами `--tail`, `--since`, `grep` для фильтрации.
2. **Подробно:**
    ```bash
    # Последние 100 строк и следить за новыми
    kubectl logs -f <pod> --tail=100
    
    # Логи за последние 10 минут
    kubectl logs <pod> --since=10m
    
    # Фильтрация по уровню (если приложение пишет JSON/текст)
    kubectl logs <pod> | grep -i "error\|exception\|fatal"
    
    # Если несколько контейнеров в поде
    kubectl logs <pod> -c <container-name>
    
    # Предыдущий экземпляр (после краша и рестарта)
    kubectl logs <pod> --previous
    ```
3. **DevOps-контекст:** В продакшене логи агрегируются в Loki/ELK. Но для быстрой отладки `kubectl logs` незаменим. Важно настраивать ротацию логов в приложении, чтобы не переполнять диск.
4. **Команды/Инструменты:** `stern <pod-label>` для нескольких подов, `kail` для потокового просмотра, `jq` для парсинга JSON-логов: `kubectl logs <pod> | jq 'select(.level=="error")'`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 6. Как создать и применить манифест Kubernetes из шаблона?

1. **Кратко:** Использовать `kubectl apply -f`, шаблонизацию через `envsubst` или Helm.
2. **Подробно:**
    ```yaml
    # deployment.yaml.template
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APP_NAME}
      namespace: ${NAMESPACE}
    spec:
      replicas: ${REPLICAS:-3}
      selector:
        matchLabels:
          app: ${APP_NAME}
      template:
        metadata:
          labels:
            app: ${APP_NAME}
        spec:
          containers:
          - name: app
            image: ${IMAGE}:${TAG}
            ports:
            - containerPort: ${PORT:-8080}
    ```
    ```bash
    # Применение с подстановкой переменных
    export APP_NAME=myapp NAMESPACE=prod REPLICAS=5 IMAGE=registry/app TAG=v1.2.3
    envsubst < deployment.yaml.template | kubectl apply -f -
    
    # Или через kustomize (без envsubst)
    kubectl apply -k overlays/prod
    ```
3. **DevOps-контекст:** В CI/CD шаблоны рендерятся автоматически. `envsubst` прост, но не валидирует схему. Для сложных случаев — Helm или Kustomize с `kubectl --dry-run=client -o yaml` для предпросмотра.
4. **Команды/Инструменты:** `kubectl apply --dry-run=client -o yaml`, `kubeval`, `kubeconform`, `kustomize build`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 7. Как написать простой bash-скрипт для автоматизации рутины?

1. **Кратко:** Использовать `set -euo pipefail`, функции, логирование, обработку ошибок.
2. **Подробно:**
    ```bash
    #!/usr/bin/env bash
    set -euo pipefail  # строгий режим
    
    # Логирование с временем
    log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"; }
    
    # Проверка зависимостей
    command -v kubectl >/dev/null || { log "kubectl not found"; exit 1; }
    
    # Функция с обработкой ошибок
    deploy_app() {
      local namespace="${1:-default}"
      log "Deploying to $namespace"
      
      if ! kubectl apply -f manifests/ -n "$namespace"; then
        log "ERROR: Deployment failed"
        return 1
      fi
      log "SUCCESS: Deployed"
    }
    
    # Main
    main() {
      deploy_app "${1:-default}"
    }
    main "$@"
    ```
3. **DevOps-контекст:** Такие скрипты используются в CI-джобах, хуках, cron-задачах. Важно: не хардкодить секреты, использовать переменные окружения, добавлять таймауты на сетевые вызовы.
4. **Команды/Инструменты:** `shellcheck script.sh` для линтинга, `bash -n script.sh` для синтаксической проверки, `trap 'log "Interrupted"; exit 130' INT TERM`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 8. Как добавить переменные окружения в деплоймент и проверить их?

1. **Кратко:** Через `env` или `envFrom` в манифесте, проверка через `kubectl exec`.
2. **Подробно:**
    ```yaml
    # deployment.yaml
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: DB_HOST
          value: "postgres.default.svc"
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: password
        envFrom:
        - configMapRef:
            name: app-common-config
    ```
    ```bash
    # Проверка в запущенном поде
    kubectl exec -it <pod> -- printenv | grep DB_
    kubectl exec -it <pod> -- env | grep LOG_LEVEL
    
    # Проверка без запуска: рендер манифеста
    kubectl apply --dry-run=client -o yaml -f deployment.yaml | grep -A5 env
    ```
3. **DevOps-контекст:** Секреты не должны попадать в логи! Использовать `echo "$SECRET" | grep .` с осторожностью. В CI можно валидировать, что все обязательные env заданы: `conftest test deployment.yaml`.
4. **Команды/Инструменты:** `kubectl create configmap`, `kubectl create secret generic`, `sops` для шифрования секретов в Git.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 9. Как проверить, что приложение отвечает на health check?

1. **Кратко:** Использовать `curl` с проверкой статус-кода и таймаутом.
2. **Подробно:**
    ```bash
    # Базовая проверка HTTP
    curl -f --connect-timeout 5 --max-time 10 http://localhost:8080/health
    
    # Проверка с ожиданием (retry-логика)
    for i in {1..30}; do
      if curl -sf http://localhost:8080/ready; then
        echo "App is ready"
        break
      fi
      echo "Waiting... ($i/30)"
      sleep 2
    done
    
    # Для gRPC-приложений
    grpc_health_probe -addr=:50051 -connect-timeout=5s
    
    # В пайплайне: выход с кодом ошибки
    if ! curl -f http://app/health; then
      echo "Health check failed"
      exit 1
    fi
    ```
3. **DevOps-контекст:** Такие проверки используются в readiness/liveness пробах в K8s, в CI после деплоя на staging, в скриптах авто-восстановления. Важно отличать "приложение запущено" (liveness) от "готов обрабатывать трафик" (readiness).
4. **Команды/Инструменты:** `curl -w '%{http_code}\n' -o /dev/null -s`, `wget --spider`, `kubectl wait --for=condition=ready pod -l app=myapp --timeout=60s`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 10. Как откатить деплой в Kubernetes, если что-то пошло не так?

1. **Кратко:** Использовать `kubectl rollout undo` или применить предыдущий манифест.
2. **Подробно:**
    ```bash
    # 1. Проверить историю ревизий
    kubectl rollout history deployment/myapp
    
    # 2. Откатиться на предыдущую версию
    kubectl rollout undo deployment/myapp
    
    # 3. Или на конкретную ревизию
    kubectl rollout undo deployment/myapp --to-revision=2
    
    # 4. Следить за статусом отката
    kubectl rollout status deployment/myapp --timeout=60s
    
    # Альтернатива: применить старый манифест
    git checkout v1.2.2 -- k8s/deployment.yaml
    kubectl apply -f k8s/deployment.yaml
    ```
3. **DevOps-контекст:** `rollout undo` меняет только Deployment, но не откатывает миграции БД! Поэтому в пайплайне: сначала откат кода, затем (при необходимости) даун-миграции. Всегда тестировать откат на staging.
4. **Команды/Инструменты:** `kubectl rollout pause/resume`, `argocd app rollback myapp`, `flux reconcile deployment myapp --force`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

# 🟡 Middle (применение в реальных сценариях)

### 11. ⭐ Как отладить CrashLoopBackOff в поде: пошаговый алгоритм?

1. **Кратко:** Проверить логи, события, конфиги, ресурсы; воспроизвести локально; использовать ephemeral containers.
2. **Подробно:**
    ```bash
    # Шаг 1: Быстрая диагностика
    kubectl describe pod <pod> | grep -A20 "Events:"
    kubectl logs <pod> --previous  # логи до краша
    
    # Шаг 2: Проверка конфигурации
    kubectl get configmap,secret -l app=myapp -o yaml
    kubectl exec -it <pod> -- sh -c 'env | grep -i error' 2>/dev/null || true
    
    # Шаг 3: Проверка ресурсов
    kubectl describe node <node> | grep -A10 "Allocated resources"
    
    # Шаг 4: Воспроизведение локально (если возможно)
    docker run --rm -it \
      -e DB_URL=$DB_URL \
      --entrypoint /bin/sh \
      myapp:latest -c "./app.py"
    
    # Шаг 5: Ephemeral container для отладки (K8s 1.20+)
    kubectl debug -it <pod> --image=busybox --target=<app-container>
    # Теперь можно исследовать файловую систему, процессы
    ```
    **Чек-лист причин:**
    - ❌ Ошибка в коде (паника, исключение) → смотреть `logs --previous`
    - ❌ Нехватка памяти (OOMKilled) → `describe pod | grep -i memory`
    - ❌ Неправильный конфиг/секрет → проверить ConfigMap/Secret
    - ❌ Проблемы с инициализацией (initContainer) → `kubectl logs <pod> -c <init-container>`
3. **DevOps-контекст:** В прод-пайплайне можно добавить автоматический сбор диагностической информации при краше: `kubectl describe + logs + events` в S3 для постмортема.
4. **Команды/Инструменты:** `stern <pod> --prefix`, `k9s` для интерактивной отладки, `promql: kube_pod_container_status_restarts_total`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 12. Как написать Terraform-модуль с переменными и валидацией?

1. **Кратко:** Использовать `variables.tf`, `outputs.tf`, `validation` блоки, примеры в `examples/`.
2. **Подробно:**
    ```hcl
    # modules/webapp/variables.tf
    variable "app_name" {
      type        = string
      description = "Application name"
      nullable    = false
    }
    
    variable "instance_type" {
      type        = string
      default     = "t3.micro"
      validation {
        condition     = can(regex("^t[23]\\.(micro|small|medium)$", var.instance_type))
        error_message = "Instance type must be t2/t3 micro/small/medium"
      }
    }
    
    variable "tags" {
      type        = map(string)
      default     = {}
      description = "Additional tags for resources"
    }
    
    # modules/webapp/main.tf
    resource "aws_instance" "app" {
      ami           = data.aws_ami.ubuntu.id
      instance_type = var.instance_type
      
      tags = merge(
        { Name = var.app_name, ManagedBy = "terraform" },
        var.tags
      )
    }
    
    # modules/webapp/outputs.tf
    output "instance_id" {
      value       = aws_instance.app.id
      description = "ID of the created instance"
    }
    
    # Пример использования
    # main.tf
    module "webapp" {
      source        = "./modules/webapp"
      app_name      = "myapp-prod"
      instance_type = "t3.small"
      tags = {
        Environment = "production"
        Team        = "backend"
      }
    }
    ```
3. **DevOps-контекст:** Модули позволяют переиспользовать инфраструктурный код. Важно: документировать переменные в `README.md`, добавлять `examples/` для тестирования, использовать `terraform validate` и `tflint` в CI.
4. **Команды/Инструменты:** `terraform init -upgrade`, `terraform plan -out=tfplan`, `tflint`, `checkov -d .` для security-сканирования.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 13. Как настроить Prometheus-алерт на рост ошибок в приложении?

1. **Кратко:** Написать PromQL-запрос, создать Alertmanager rule, настроить уведомления.
2. **Подробно:**
    ```yaml
    # alerts.yaml (для Prometheus или kube-prometheus-stack)
    groups:
    - name: application
      rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / 
          sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Высокий уровень ошибок ({{ $value | humanizePercentage }})"
          description: "Сервис {{ $labels.job }} возвращает 5xx более 5% запросов"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
    
      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 5m
        labels:
          severity: critical
    ```
    ```yaml
    # alertmanager config (упрощённо)
    route:
      group_by: [alertname, team]
      routes:
      - match: { severity: critical }
        receiver: pagerduty-critical
      - match: { severity: warning }
        receiver: slack-warnings
    
    receivers:
    - name: slack-warnings
      slack_configs:
      - channel: '#devops-alerts'
        send_resolved: true
    ```
3. **DevOps-контекст:** Алерты должны быть actionable: иметь runbook, понятное описание, правильные пороги. Избегайте "алерт-шторма": используйте `for` для устойчивости, группируйте по сервисам. Тестируйте алерты: `promtool test rules alerts.yaml`.
4. **Команды/Инструменты:** `promtool check rules alerts.yaml`, `amtool alert query`, Grafana Explore для отладки PromQL.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 14. Как реализовать rolling update с проверкой готовности?

1. **Кратко:** Настроить `strategy.rollingUpdate` в Deployment и readiness probe.
2. **Подробно:**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    spec:
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 0      # не допускать простоя
          maxSurge: 1            # создавать +1 под при обновлении
      minReadySeconds: 10        # ждать 10с после readiness перед следующим
      template:
        spec:
          containers:
          - name: app
            readinessProbe:
              httpGet:
                path: /ready
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 5
              failureThreshold: 3
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 15
              periodSeconds: 10
    ```
    ```bash
    # Запуск обновления и мониторинг
    kubectl set image deployment/myapp app=myapp:v2.0
    kubectl rollout status deployment/myapp --timeout=300s
    
    # Если что-то пошло не так — откат
    kubectl rollout undo deployment/myapp
    ```
3. **DevOps-контекст:** Rolling update без readiness probe может привести к тому, что трафик пойдёт на ещё не готовый под → ошибки. `maxUnavailable: 0` гарантирует zero-downtime, но требует достаточно ресурсов для `maxSurge`.
4. **Команды/Инструменты:** `kubectl rollout history`, `argocd app sync --strategy rolling`, `flux suspend deployment/myapp` для паузы.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 15. Как отладить NetworkPolicy: почему трафик блокируется?

1. **Кратко:** Проверить политику, метки, namespace; использовать `kubectl describe`, `tcpdump`, `netshoot`.
2. **Подробно:**
    ```bash
    # 1. Посмотреть применённые политики
    kubectl get networkpolicy -n <namespace>
    kubectl describe networkpolicy <name> -n <namespace>
    
    # 2. Проверить метки пода и селектор политики
    kubectl get pod <pod> --show-labels
    kubectl get networkpolicy <name> -o yaml | grep -A5 podSelector
    
    # 3. Протестировать соединение из другого пода
    kubectl run debug --rm -it --image=nicolaka/netshoot --restart=Never -n <namespace> -- sh
    # Внутри контейнера:
    nc -zv <target-service> <port>
    curl -v http://<target-service>:<port>
    
    # 4. Проверить, видит ли CNI политику (для Calico)
    kubectl exec -n kube-system -c calico-node <calico-pod> -- calicoctl get networkpolicy
    
    # 5. Снизить уровень блокировки для отладки (осторожно!)
    # Временно добавить allow-all политику
    ```
    **Типичные ошибки:**
    - ❌ Неправильный `namespaceSelector` или `podSelector`
    - ❌ Политика применяется не к тому namespace
    - ❌ CNI-плагин не поддерживает NetworkPolicy (нужен Calico, Cilium, etc.)
3. **DevOps-контекст:** В CI можно валидировать политики: `conftest test networkpolicy.yaml` с правилами "запретить ingress извне". Для отладки в staging: использовать `cilium monitor` или `calico-node` логи.
4. **Команды/Инструменты:** `cilium endpoint list`, `calicoctl get workloadendpoint`, `kubectl sniff <pod>` (плагин ksniff).

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 16. Как параметризовать Helm-чарт для разных окружений?

1. **Кратко:** Использовать `values.yaml` + `values-<env>.yaml`, `--set` для переопределения.
2. **Подробно:**
    ```
    charts/myapp/
    ├── Chart.yaml
    ├── values.yaml          # базовые значения
    ├── values-dev.yaml      # переопределения для dev
    ├── values-staging.yaml
    ├── values-prod.yaml
    └── templates/
        ├── deployment.yaml  # шаблоны с {{ .Values.* }}
        └── service.yaml
    ```
    ```yaml
    # values.yaml (базовые)
    replicaCount: 2
    image:
      repository: myapp
      tag: "latest"
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    
    # values-prod.yaml
    replicaCount: 5
    image:
      tag: "v1.2.3"  # фиксированная версия
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
    ingress:
      enabled: true
      host: app.example.com
    ```
    ```bash
    # Установка/обновление
    helm upgrade --install myapp ./charts/myapp \
      -f ./charts/myapp/values-prod.yaml \
      --set image.tag=v1.2.3 \
      --namespace production \
      --atomic  # авто-откат при ошибке
    
    # Предпросмотр рендеренного манифеста
    helm template myapp ./charts/myapp -f values-prod.yaml --debug
    ```
3. **DevOps-контекст:** В пайплайне: хранить `values-*.yaml` в одном репо с кодом, использовать `helm lint` и `helm template | kubeconform` для валидации. Для секретов: `helm-secrets` плагин с SOPS.
4. **Команды/Инструменты:** `helm lint ./charts/myapp`, `helm unittest` (плагин), `ct lint --charts ./charts` (chart-testing в CI).

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 17. Как собрать и отправить логи в Loki/ELK с парсингом?

1. **Кратко:** Использовать fluent-bit/Promtail для сбора, настроить парсинг и лейблы.
2. **Подробно:**
    ```yaml
    # promtail config для Loki (K8s)
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      pipeline_stages:
      - cri: {}  # парсинг формата контейнерного рантайма
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'  # для Java-стеков
          max_wait_time: 3s
      - regex:
          expression: 'level=(?P<level>\w+).*msg="(?P<msg>.*)"'
          labels:
            level:  # добавить level как лейбл для фильтрации
      - labels:
          app:  # взять из меток K8s
      - output:
          source_labels: [msg]  # финальное сообщение
    ```
    ```bash
    # Пример запроса в Loki (LogQL)
    # Ошибки за последний час по приложению
    {app="myapp", level="error"} | json | line_format "{{.msg}}"
    
    # Агрегация по статус-кодам
    sum by (status) (count_over_time({job="nginx"} | regexp "HTTP/(?P<status>\\d+)" [5m]))
    ```
3. **DevOps-контекст:** Правильные лейблы критичны для производительности: не лейблируйте высокоразмерные поля (user_id, request_id) — используйте парсинг внутри запроса. В CI: тестировать парсеры на примерах логов.
4. **Команды/Инструменты:** `logcli query '{app="myapp"}' --since=1h`, `grafana explore` для интерактивного поиска, `fluent-bit -c config.conf -e` для отладки парсеров.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 18. Как безопасно добавить секрет в пайплайн и приложение?

1. **Кратко:** Хранить в секрете (K8s Secret / Vault), не в коде; использовать injection в runtime.
2. **Подробно:**
    ```yaml
    # 1. Создать секрет в K8s (не коммитить plain-текст!)
    kubectl create secret generic app-secret \
      --from-literal=db-password='SuperSecret123' \
      --dry-run=client -o yaml | sops -e - > k8s/secret.enc.yaml
    # Теперь secret.enc.yaml можно коммитить
    ```
    ```yaml
    # 2. Использовать в Deployment (без хардкода!)
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-password
        # Или: mount как файл (безопаснее для некоторых приложений)
        volumeMounts:
        - name: secrets
          mountPath: /secrets
          readOnly: true
      volumes:
      - name: secrets
        secret:
          secretName: app-secret
    ```
    ```yaml
    # 3. В пайплайне (GitHub Actions пример)
    jobs:
      deploy:
        steps:
        - name: Deploy
          run: ./deploy.sh
          env:
            DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # из GitHub Secrets
    ```
    **Best practices:**
    🔐 Никогда не логировать секреты: `echo "Connecting with $DB_PASSWORD"` → утечка!
    🔐 Использовать external secrets (Vault, AWS Secrets Manager) для динамической ротации
    🔐 Ограничивать доступ к секретам через RBAC: `kubectl auth can-i get secret --as=dev-user`
3. **DevOps-контекст:** В продакшене: external-secrets operator синхронизирует секреты из Vault в K8s. В пайплайне: использовать OIDC для аутентификации в облаке вместо long-lived keys.
4. **Команды/Инструменты:** `sops -d secret.enc.yaml`, `vault kv get secret/app`, `external-secrets.io` CRD, `trufflehog` для сканирования репо на секреты.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 19. Как настроить multi-stage пайплайн с тестами и деплоем?

1. **Кратко:** Разделить на stages: lint → test → build → deploy; использовать условия и артефакты.
2. **Подробно:**
    ```yaml
    # .gitlab-ci.yml пример
    stages:
      - validate
      - test
      - build
      - deploy
    
    lint:
      stage: validate
      image: node:18-alpine
      script:
        - npm ci
        - npm run lint
        - npm run typecheck
      rules:
        - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    
    test:
      stage: test
      image: node:18-alpine
      script:
        - npm ci
        - npm run test:unit -- --coverage
        - npm run test:integration
      artifacts:
        reports:
          coverage_report:
            coverage_format: cobertura
            path: coverage/cobertura-coverage.xml
      rules:
        - if: $CI_COMMIT_BRANCH == "main"
    
    build-image:
      stage: build
      image: docker:24
      services: [docker:24-dind]
      script:
        - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      needs: [test]
      rules:
        - if: $CI_COMMIT_BRANCH == "main"
    
    deploy-staging:
      stage: deploy
      image: bitnami/kubectl:latest
      script:
        - kubectl set image deployment/myapp app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n staging
        - kubectl rollout status deployment/myapp -n staging --timeout=120s
      environment:
        name: staging
        url: https://staging.example.com
      needs: [build-image]
      rules:
        - if: $CI_COMMIT_BRANCH == "main"
    ```
3. **DevOps-контекст:** Использование `needs` ускоряет пайплайн (параллельное выполнение независимых джоб). `rules` вместо `only/except` даёт больше гибкости. Артефакты тестов позволяют строить отчёты в Merge Request.
4. **Команды/Инструменты:** `gitlab-ci-lint`, `act` для локального запуска GitHub Actions, `tekton` для сложных пайплайнов.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 20. Как настроить Horizontal Pod Autoscaler по CPU и кастомным метрикам?

1. **Кратко:** Использовать HPA с metrics-server для CPU и Prometheus Adapter для кастомных метрик.
2. **Подробно:**
    ```yaml
    # 1. Базовый HPA по CPU
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: myapp-hpa
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: myapp
      minReplicas: 2
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70  # целевая утилизация 70%
    
    # 2. + кастомная метрика (например, RPS на под)
    spec:
      metrics:
      - type: Pods
        pods:
          metric:
            name: http_requests_per_second
          target:
            type: AverageValue
            averageValue: 100  # 100 запросов/сек на под
    ```
    ```bash
    # Проверка работы
    kubectl get hpa myapp-hpa --watch
    kubectl top pod -l app=myapp  # требует metrics-server
    
    # Отладка кастомных метрик
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq
    ```
3. **DevOps-контекст:** HPA не заменяет Cluster Autoscaler: если не хватает нод, поды останутся в Pending. Важно настраивать `minReadySeconds` и `behavior` для плавного скейлинга: `behavior.scaleDown.stabilizationWindowSeconds: 300`.
4. **Команды/Инструменты:** `kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10`, `prometheus-adapter` config, `kubectl debug` для отладки метрик.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

# 🔴 Senior (архитектура и сложные сценарии)

### 21. ⭐ Как спроектировать и протестировать disaster recovery процедуру?

1. **Кратко:** Определить RTO/RPO, задокументировать runbook, автоматизировать восстановление, регулярно проводить учения.
2. **Подробно:**
    ```markdown
    # DR Runbook: пример структуры
    
    ## 1. Trigger conditions
    - Полная недоступность региона > 15 мин
    - Потеря данных в первичной БД
    
    ## 2. Roles & Responsibilities
    - Incident Commander: @oncall-lead
    - DB Specialist: @dba-team
    - Infra: @platform-team
    
    ## 3. Recovery Steps (automated where possible)
    ### 3.1. Failover DNS
    ```bash
    # Переключить Route53 на вторичный регион
    aws route53 change-resource-record-sets \
      --hosted-zone-id Z123456 \
      --change-batch file://failover.json
    ```
    
    ### 3.2. Promote read replica
    ```bash
    # AWS RDS пример
    aws rds promote-read-replica \
      --db-instance-identifier myapp-replica-eu
    
    # Дождаться доступности
    aws rds wait db-instance-available \
      --db-instance-identifier myapp-replica-eu
    ```
    
    ### 3.3. Update app config
    ```bash
    # Обновить ConfigMap с новым endpoint БД
    kubectl patch configmap app-config -n prod \
      --patch '{"data":{"DB_HOST":"myapp-replica-eu.xxx.rds.amazonaws.com"}}'
    
    # Перезапустить поды для применения
    kubectl rollout restart deployment/myapp -n prod
    ```
    
    ## 4. Validation
    - [ ] Health check: `curl https://app.example.com/health`
    - [ ] Smoke tests: запуск e2e-тестов на staging
    - [ ] Проверка репликации в обратную сторону
    
    ## 5. Rollback procedure
    # Если failover неудачен: шаги отката...
    
    ## 6. Post-incident
    - Обновить runbook по итогам учений
    - Запланировать устранение root cause
    ```
    ```bash
    # Тестирование DR (на staging!)
    # 1. Simulate region failure (отключить сеть)
    # 2. Запустить runbook в "dry-run" режиме
    # 3. Измерить время восстановления (RTO)
    # 4. Проверить потерю данных (RPO)
    ```
3. **DevOps-контекст:** DR-тесты должны быть регулярными (ежеквартально) и автоматизированными там, где возможно. Использовать chaos engineering для имитации сбоев. Документировать всё в runbook, который доступен офлайн.
4. **Команды/Инструменты:** `aws fis` (Fault Injection Simulator), `chaos-mesh`, `terraform state pull` для бэкапа стейта, `velero` для бэкапа K8s.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 22. Как настроить GitOps-синхронизацию с авто-исправлением дрейфа?

1. **Кратко:** Использовать ArgoCD/Flux с `syncPolicy.automated`, `selfHeal`, мониторингом дрейфа.
2. **Подробно:**
    ```yaml
    # ArgoCD Application manifest
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: myapp-prod
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://git.example.com/infra
        path: apps/myapp/overlays/prod
        targetRevision: HEAD
      destination:
        server: https://kubernetes.default.svc
        namespace: production
      syncPolicy:
        automated:
          prune: true      # удалять ресурсы, удалённые из Git
          selfHeal: true   # исправлять дрейф автоматически
          allowEmpty: false
        syncOptions:
        - CreateNamespace=true
        - PrunePropagationPolicy=foreground
        - PruneLast=true
      ignoreDifferences:
      - group: apps
        kind: Deployment
        jsonPointers:
        - /spec/replicas  # игнорировать ручное масштабирование
    ```
    ```bash
    # Мониторинг дрейфа
    argocd app list | grep OutOfSync
    argocd app diff myapp-prod  # показать различия
    
    # Ручная синхронизация (если auto отключён)
    argocd app sync myapp-prod --prune
    
    # Откат к предыдущей ревизии в Git
    argocd app rollback myapp-prod
    ```
3. **DevOps-контекст:** `selfHeal: true` удобен, но может маскировать ручные изменения в проде. Лучше: запретить `kubectl edit` в prod через admission webhook, а дрейф использовать как сигнал для аудита. В пайплайне: `argocd app wait myapp-prod --health` для ожидания успешного деплоя.
4. **Команды/Инструменты:** `argocd appset` для управления множеством приложений, `flux reconcile source git`, `kubectl argo rollouts` для canary-деплоя.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 23. Как инструментировать приложение для distributed tracing?

1. **Кратко:** Использовать OpenTelemetry SDK, настроить exporter, propagate context между сервисами.
2. **Подробно:**
    ```python
    # Python пример с OpenTelemetry
    from opentelemetry import trace
    from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
    from opentelemetry.sdk.trace import TracerProvider
    from opentelemetry.sdk.trace.export import BatchSpanProcessor
    
    # Инициализация
    provider = TracerProvider()
    processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
    provider.add_span_processor(processor)
    trace.set_tracer_provider(provider)
    
    # Использование в коде
    tracer = trace.get_tracer(__name__)
    
    def process_order(order_id):
        with tracer.start_as_current_span("process_order") as span:
            span.set_attribute("order.id", order_id)
            # ... бизнес-логика
            call_payment_service(order_id)  # context автоматически прокидывается
    
    # HTTP-клиент: context propagation
    from opentelemetry.propagate import inject
    import requests
    
    def call_payment_service(order_id):
        headers = {}
        inject(headers)  # добавляет traceparent, tracestate
        requests.post("https://payment/api/charge", json={"order": order_id}, headers=headers)
    ```
    ```yaml
    # otel-collector config (упрощённо)
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      batch:
      tail_sampling:  # sample только ошибки или медленные запросы
        policies:
          - name: errors
            type: status_code
            status_code:
              status_codes: [ERROR]
    exporters:
      jaeger:
        endpoint: jaeger:14250
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, tail_sampling]
          exporters: [jaeger]
    ```
3. **DevOps-контекст:** Sampling критичен для стоимости: не трассировать 100% запросов. Использовать `tail_sampling` для ошибок и медленных запросов. В K8s: sidecar-инжекция через annotation для прозрачного инструментирования.
4. **Команды/Инструменты:** `otel-cli exec -- curl http://app`, `jaeger-ui` для визуализации, `prometheus: otel_trace_span_duration_seconds` для метрик.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 24. Как написать ValidatingAdmissionWebhook для политик безопасности?

1. **Кратко:** Создать webhook-сервис, зарегистрировать в K8s, валидировать ресурсы при создании/обновлении.
2. **Подробно:**
    ```go
    // Пример вебхука на Go (упрощённо)
    func validatePod(pod *corev1.Pod) error {
        // Правило 1: запретить запуск от root
        if pod.Spec.SecurityContext == nil || 
           pod.Spec.SecurityContext.RunAsUser == nil || 
           *pod.Spec.SecurityContext.RunAsUser == 0 {
            return fmt.Errorf("pods must not run as root")
        }
        
        // Правило 2: требовать limits ресурсов
        for _, container := range pod.Spec.Containers {
            if container.Resources.Limits.Cpu() == nil {
                return fmt.Errorf("container %s must have CPU limit", container.Name)
            }
        }
        return nil
    }
    
    // HTTP handler для AdmissionReview
    func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        var review admissionv1.AdmissionReview
        // ... парсинг запроса
        err := validatePod(&pod)
        review.Response = &admissionv1.AdmissionResponse{
            Allowed: err == nil,
            Result:  &metav1.Status{Message: err.Error()},
        }
        // ... отправка ответа
    }
    ```
    ```yaml
    # Регистрация вебхука в K8s
    apiVersion: admissionregistration.k8s.io/v1
    kind: ValidatingWebhookConfiguration
    metadata:
      name: security-policy.example.com
    webhooks:
    - name: validate.security.example.com
      clientConfig:
        service:
          namespace: webhook-system
          name: security-webhook
          path: /validate
        caBundle: <base64-cert>
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
      failurePolicy: Fail  # отклонять запрос при недоступности вебхука
      sideEffects: None
      admissionReviewVersions: ["v1"]
    ```
3. **DevOps-контекст:** Вебхуки влияют на latency создания ресурсов — держать их быстрыми (<100ms). Использовать `failurePolicy: Ignore` в staging для отладки, `Fail` в prod. Мониторить метрики вебхука: `admission_webhook_request_total`, `admission_webhook_latency_seconds`.
4. **Команды/Инструменты:** `kubectl auth can-i create validatingwebhookconfiguration`, `cert-manager` для авто-ротации сертификатов, `conftest` для тестирования политик офлайн.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 25. Как проанализировать и оптимизировать затраты на облачную инфраструктуру?

1. **Кратко:** Использовать tagging, cost allocation reports, правайзинг, автоскейлинг, резервирование.
2. **Подробно:**
    ```bash
    # 1. Анализ текущих затрат (AWS пример)
    # Экспорт billing данных в Athena
    aws ce get-cost-and-usage \
      --time-period Start=2024-01-01,End=2024-01-31 \
      --granularity MONTHLY \
      --metrics BlendedCost \
      --group-by Type=DIMENSION,Key=SERVICE
    
    # 2. Поиск неэффективных ресурсов
    # EC2 с низкой утилизацией CPU
    aws cloudwatch get-metric-statistics \
      --namespace AWS/EC2 \
      --metric-name CPUUtilization \
      --dimensions Name=InstanceId,Value=i-123456 \
      --start-time 2024-01-01T00:00:00Z \
      --end-time 2024-01-31T23:59:59Z \
      --period 3600 \
      --statistics Average
    
    # 3. Рекомендации по резервированию
    aws compute-optimizer get-enrollment-status
    aws compute-optimizer get-lambda-recommendations
    ```
    ```yaml
    # Terraform: оптимизация через правайзинг и spot
    resource "aws_instance" "worker" {
      ami           = data.aws_ami.ubuntu.id
      instance_type = "c5.large"
      
      # Spot для fault-tolerant workloads
      purchase_option = "spot"
      spot_options {
        instance_interruption_behavior = "terminate"
      }
      
      # Auto-stop для dev-окружений
      tags = {
        AutoStopSchedule = "weekdays-8pm-to-8am"  # обрабатывается Lambda
      }
    }
    
    # K8s: Vertical Pod Autoscaler для правайзинга
    apiVersion: autoscaling.k8s.io/v1
    kind: VerticalPodAutoscaler
    metadata:
      name: myapp-vpa
    spec:
      targetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: myapp
      updatePolicy:
        updateMode: "Auto"  # или "Initial" для разового применения
    ```
3. **DevOps-контекст:** Cost optimization — непрерывный процесс. Внедрить "cost as a metric": дашборды с затратами по командам/сервисам, алерты на аномальный рост. Использовать FinOps-практики: регулярные ревью с разработчиками.
4. **Команды/Инструменты:** `aws costexplorer`, `gcloud billing reports`, `kubecost` для K8s, `infracost` для оценки стоимости до применения Terraform.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 26. Как внедрить подписанные артефакты и проверку целостности (Sigstore)?

1. **Кратко:** Использовать Cosign для подписи образов, Fulcio для сертификатов, Rekor для прозрачности.
2. **Подробно:**
    ```bash
    # 1. Подпись образа при сборке (в пайплайне)
    cosign sign --yes \
      -a "build-id=$GITHUB_RUN_ID" \
      -a "commit=$GITHUB_SHA" \
      registry.example.com/myapp:v1.2.3
    
    # 2. Генерация ключа (опционально, если не использовать keyless)
    cosign generate-key-pair
    # Файлы: cosign.key (приватный), cosign.pub (публичный)
    
    # 3. Верификация перед деплоем
    cosign verify --key cosign.pub \
      --check-claims \
      registry.example.com/myapp:v1.2.3
    
    # 4. Интеграция в Admission Controller (K8s)
    # policy-controller проверяет подпись при создании пода
    ```
    ```yaml
    # ClusterImagePolicy для policy-controller
    apiVersion: policy.sigstore.dev/v1beta1
    kind: ClusterImagePolicy
    metadata:
      name: signed-images-only
    spec:
      images:
      - glob: "registry.example.com/**"
      authorities:
      - key:
          data: |
            -----BEGIN PUBLIC KEY-----
            MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
            -----END PUBLIC KEY-----
    ```
    ```yaml
    # GitHub Actions пример
    - name: Sign image
      uses: sigstore/cosign-installer@v3
      - name: Sign
        run: |
          cosign sign --yes \
            -a "github-repository=$GITHUB_REPOSITORY" \
            -a "github-ref=$GITHUB_REF" \
            $IMAGE_DIGEST
      env:
        COSIGN_PASSWORD: ""  # для keyless
    ```
3. **DevOps-контекст:** Keyless signing (через OIDC) удобнее для пайплайнов, но требует доверия к фулси. Для критичных систем — использовать приватные ключи с ротацией. Интегрировать проверку в GitOps: ArgoCD не синхронизирует неподписанные образы.
4. **Команды/Инструменты:** `cosign tree` для просмотра связанных артефактов, `rekor-cli search`, `kyverno` для политик подписи.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 27. Как создать Operator для автоматического восстановления состояния?

1. **Кратко:** Использовать Kubebuilder/Operator SDK, реализовать reconcile loop, настроить watch на кастомный ресурс.
2. **Подробно:**
    ```go
    // Пример reconcile-логики (упрощённо)
    func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
        // 1. Загрузить кастомный ресурс
        var myapp myappsv1.MyApp
        if err := r.Get(ctx, req.NamespacedName, &myapp); err != nil {
            return ctrl.Result{}, client.IgnoreNotFound(err)
        }
        
        // 2. Проверить состояние зависимостей
        dbReady, err := r.checkDatabaseReady(ctx, myapp.Spec.DatabaseRef)
        if err != nil || !dbReady {
            // Не готово: перезапросить через 30с
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        
        // 3. Синхронизировать желаемое состояние
        if err := r.reconcileDeployment(ctx, &myapp); err != nil {
            return ctrl.Result{}, err
        }
        
        // 4. Обновить статус ресурса
        myapp.Status.Ready = true
        myapp.Status.LastReconciled = metav1.Now()
        return ctrl.Result{}, r.Status().Update(ctx, &myapp)
    }
    
    // Setup: зарегистрировать watch
    func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
        return ctrl.NewControllerManagedBy(mgr).
            For(&myappsv1.MyApp{}).
            Owns(&appsv1.Deployment{}).  // следить за созданными ресурсами
            Complete(r)
    }
    ```
    ```yaml
    # Custom Resource Definition (CRD)
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: myapps.example.com
    spec:
      group: example.com
      versions:
      - name: v1
        served: true
        storage: true
        schema:
          openAPIV3Schema:
            type: object
            properties:
              spec:
                type: object
                properties:
                  replicas: { type: integer, minimum: 1 }
                  databaseRef: { type: string }
              status:
                type: object
                properties:
                  ready: { type: boolean }
                  lastReconciled: { type: string, format: date-time }
        subresources:
          status: {}  # включить subresource для обновления статуса
      scope: Namespaced
      names:
        plural: myapps
        singular: myapp
        kind: MyApp
    ```
3. **DevOps-контекст:** Операторы — мощный инструмент, но добавляют сложность. Использовать для состояния, которое трудно выразить декларативно (БД, очереди). Тестировать reconcile-логику с envtest, мониторить метрики: `controller_runtime_reconcile_total`, `reconcile_errors`.
4. **Команды/Инструменты:** `kubebuilder init`, `operator-sdk generate k8s`, `kubectl apply -f config/crd/bases`, `prometheus: controller_runtime_reconcile_time_seconds`.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 28. Как безопасно провести chaos-эксперимент в staging/prod?

1. **Кратко:** Определить гипотезу, ограничить бласт-радиус, настроить мониторинг и "стоп-кран", документировать результаты.
2. **Подробно:**
    ```yaml
    # Chaos Mesh эксперимент (убийство подов)
    apiVersion: chaos-mesh.org/v1alpha1
    kind: PodChaos
    metadata:
      name: kill-random-pods
      namespace: chaos-testing
    spec:
      action: kill
      mode: random-max-percent
      value: "30"  # убить максимум 30% подов
      selector:
        namespaces: ["production"]
        labelSelectors:
          app: myapp
          canary: "false"  # не трогать canary-поды
      duration: "30s"
      scheduler:
        cron: "@every 24h"  # запускать раз в сутки
      # Ограничения безопасности
      gracefulPeriod: "10s"  # дать время на graceful shutdown
    ```
    ```bash
    # Pre-checks перед запуском
    # 1. Убедиться, что мониторинг работает
    curl -s https://prometheus.example.com/-/healthy
    
    # 2. Проверить, что есть capacity для failover
    kubectl top nodes | grep -v "80%"  # ни одна нода не должна быть >80%
    
    # 3. Уведомить команду (через Slack/чат)
    # 4. Запустить в "dry-run" режиме (если поддерживается)
    
    # Во время эксперимента: мониторинг в реальном времени
    # Grafana дашборд с: error rate, latency, pod restarts
    
    # Экстренная остановка
    kubectl delete -f kill-random-pods.yaml
    # Или через API Chaos Mesh
    curl -X DELETE https://chaos-mesh/api/experiments/pod/kill-random-pods
    ```
3. **DevOps-контекст:** Никогда не запускать chaos в prod без: 1) одобрения команды, 2) работающего мониторинга, 3) плана отката. Начинать с staging, потом canary, потом prod. Документировать каждый эксперимент: гипотеза, результат, действия по улучшению.
4. **Команды/Инструменты:** `chaos-mesh dashboard`, `litmuschaos`, `aws fis`, `prometheus: chaos_experiment_active` метрика для алертинга.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 29. Как управлять конфигурацией для 10+ кластеров без дублирования?

1. **Кратко:** Использовать иерархию репозиториев, Kustomize overlays, GitOps-приложения с наследованием.
2. **Подробно:**
    ```
    # Структура репозитория
    infra/
    ├── bases/
    │   ├── myapp/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── kustomization.yaml
    │   └── monitoring/
    ├── overlays/
    │   ├── clusters/
    │   │   ├── eu-west-1/
    │   │   │   ├── kustomization.yaml  # специфика региона
    │   │   │   ├── values.yaml         # overrides
    │   │   │   └── secrets.enc.yaml
    │   │   ├── us-east-1/
    │   │   └── ap-south-1/
    │   └── environments/
    │       ├── dev/
    │       ├── staging/
    │       └── prod/
    └── apps/
        └── app-of-apps.yaml  # ArgoCD ApplicationSet
    ```
    ```yaml
    # overlays/clusters/eu-west-1/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ../../bases/myapp
    namespace: myapp-prod
    
    # Регион-специфичные настройки
    patches:
    - patch: |-
        - op: replace
          path: /spec/template/spec/containers/0/env/0/value
          value: "db-eu.example.com"
      target:
        kind: Deployment
        name: myapp
    
    # Генерация ConfigMap из values
    configMapGenerator:
    - name: app-config
      behavior: merge
      envs:
      - values.yaml
    ```
    ```yaml
    # ArgoCD ApplicationSet для масштабирования
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: myapp-multi-cluster
    spec:
      generators:
      - matrix:
          generators:
          - clusters: {}  # все подключённые кластера
          - list:
              elements:
              - env: dev
                values: values-dev.yaml
              - env: prod
                values: values-prod.yaml
      template:
        metadata:
          name: 'myapp-{{name}}-{{env}}'
        spec:
          project: default
          source:
            repoURL: https://git.example.com/infra
            path: 'apps/myapp/overlays/clusters/{{name}}'
            targetRevision: HEAD
          destination:
            server: '{{server}}'
            namespace: 'myapp-{{env}}'
    ```
3. **DevOps-контекст:** Избегать копипасты: использовать `kustomize build`, `helm template`, `jsonnet` для генерации. Валидировать конфиги в CI: `kubeconform`, `conftest`. Для секретов: `sops` с разными ключами на окружение.
4. **Команды/Инструменты:** `kustomize build overlays/clusters/eu-west-1`, `argocd appset generate`, `sops -e -i values.yaml`, `flux diff` для предпросмотра изменений.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]

---

### 30. ⭐ Как провести постмортем инцидента и превратить его в улучшения?

1. **Кратко:** Следовать блеймлесс-подходу, документировать таймлайн, root cause, action items с владельцами и дедлайнами.
2. **Подробно:**
    ```markdown
    # Postmortem Template
    
    ## Incident Summary
    - **Дата/время**: 2024-03-15 14:23–15:47 UTC
    - **Impact**: 40% пользователей не могли оформить заказ, ~$120k потерянной выручки
    - **Severity**: SEV-1
    
    ## Timeline (UTC)
    | Время | Событие | Ответственный |
    |-------|---------|--------------|
    | 14:23 | Алерт: HighErrorRate в payment-service | @oncall |
    | 14:28 | Обнаружено: исчерпание connection pool в БД | @dba |
    | 14:35 | Временное решение: увеличение max_connections | @dba |
    | 14:42 | Сервис восстановлен, но ошибки продолжаются | @backend |
    | 15:10 | Root cause: утечка соединений в новом релизе v2.3.1 | @backend |
    | 15:30 | Rollback на v2.3.0, ошибки прекратились | @oncall |
    | 15:47 | Инцидент закрыт | @incident-commander |
    
    ## Root Cause Analysis (5 Whys)
    1. Почему вырос error rate? → Приложения не могли подключиться к БД
    2. Почему не было соединений? → Pool исчерпан, новые запросы отклонялись
    3. Почему pool исчерпан? → В v2.3.1 добавлен код без закрытия соединений
    4. Почему код попал в прод? → Отсутствие теста на утечку ресурсов в CI
    5. Почему нет такого теста? → Не было требования в Definition of Done
    
    ## Contributing Factors
    - Недостаточное покрытие интеграционными тестами
    - Отсутствие мониторинга метрики `db_connection_pool_usage`
    - Долгий time-to-rollback (15 мин вместо целевых 5)
    
    ## Action Items
    | Задача | Владелец | Приоритет | Дедлайн | Статус |
    |--------|----------|-----------|---------|--------|
    | Добавить тест на утечку соединений в CI | @backend-lead | P0 | 2024-03-22 | ✅ Done |
    | Настроить алерт на pool usage > 80% | @platform | P1 | 2024-03-29 | 🔄 In Progress |
    | Улучшить runbook rollback: автоматизировать | @devops | P1 | 2024-04-05 | ⏳ Todo |
    | Обновить DoD: требовать проверку ресурсов | @tech-lead | P2 | 2024-04-12 | ⏳ Todo |
    
    ## Lessons Learned
    - ✅ Сработало: мониторинг быстро обнаружил аномалию
    - ❌ Не сработало: отсутствие алерта на pool usage
    - 💡 Идея: добавить chaos-тест на утечку соединений в staging
    ```
    ```bash
    # Автоматизация сбора данных для постмортема
    # Скрипт, который можно запустить во время инцидента:
    ./collect-incident-data.sh > incident-2024-03-15.tar.gz
    # Включает: логи, метрики, конфигурации, timeline из ChatOps
    ```
3. **DevOps-контекст:** Постмортем — не поиск виноватых, а улучшение системы. Делать их обязательными для всех SEV-1/2 инцидентов. Публично делиться learnings внутри компании (blameless culture). Отслеживать выполнение action items: если не сделано — эскалировать.
4. **Команды/Инструменты:** `jira` для трекинга action items, `confluence` для документации, `pagerduty postmortem`, `firehydrant` для специализированных инструментов.

[[#📑 Оглавление: Практика DevOps|↑ К оглавлению]]
