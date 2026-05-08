# 📄 Файл: `Kubernetes вопросы.md`

tags: [kubernetes, devops, interview, questions]
aliases: [k8s-questions, kube-qa]
created: 2026-05-07
---

# 📑 Kubernetes: Вопросы для собеседования и самопроверки

> [!INFO] Структура
> Вопросы разделены по уровням: 🟢 Junior → 🟡 Middle → 🔴 Senior.  
> Каждый вопрос содержит: краткий ответ, подробное объяснение, DevOps-контекст и связанные команды.

📋 [[#🗂️ Оглавление для навигации|Оглавление]] | [[#🧪 Чек-лист подготовки|Чек-лист]] | [[#🔗 Связь с другими файлами|Связи]]

---

## 🗂️ Оглавление для навигации

### 🟢 Junior (базовое понимание)
- [[#1. Что такое Kubernetes и зачем он нужен в DevOps?|1. Что такое Kubernetes]]
- [[#2. В чём разница между Pod, Deployment и Service?|2. Pod vs Deployment vs Service]]
- [[#3. Как проверить состояние подов и что означают статусы?|3. Статусы подов]]
- [[#4. Что такое Namespace и зачем он нужен?|4. Namespace]]
- [[#5. Как получить доступ к приложению внутри кластера?|5. Доступ к приложению]]
- [[#6. Что такое ConfigMap и Secret? В чём разница?|6. ConfigMap vs Secret]]
- [[#7. Как обновить приложение без простоя?|7. Обновление без простоя]]
- [[#8. Что такое livenessProbe и readinessProbe?|8. Probes]]
- [[#9. Как посмотреть логи приложения?|9. Логи приложения]]
- [[#10. Что такое kubectl и как он аутентифицируется?|10. kubectl и аутентификация]]

### 🟡 Middle (применение, нюансы)
- [[#11. ⭐ В чём разница между Deployment, StatefulSet и DaemonSet?|11. Контроллеры ⭐]]
- [[#12. Как работает Service и kube-proxy?|12. Service и kube-proxy]]
- [[#13. Что такое ResourceQuota и LimitRange?|13. Квоты и лимиты]]
- [[#14. Как настроить авто-масштабирование (HPA)?|14. HPA]]
- [[#15. Что такое NetworkPolicy и как она работает?|15. NetworkPolicy]]
- [[#16. Как откатить деплой, если что-то пошло не так?|16. Откат деплоя]]
- [[#17. Что такое initContainer и когда его использовать?|17. initContainer]]
- [[#18. Как работает readinessProbe с startupProbe?|18. startupProbe]]
- [[#19. Что такое Taints и Tolerations?|19. Taints/Tolerations]]
- [[#20. Как отладить проблему с сетью в кластере?|20. Отладка сети]]

### 🔴 Senior (архитектура, trade-offs, troubleshooting)
- [[#21. ⭐ Опиши архитектуру Kubernetes: control plane vs worker nodes|21. Архитектура ⭐]]
- [[#22. Как работает scheduler и как повлиять на размещение пода?|22. Scheduler]]
- [[#23. Что такое etcd и почему он критичен?|23. etcd]]
- [[#24. Как работает CNI и в чём разница между Calico, Cilium, Flannel?|24. CNI-плагины]]
- [[#25. ⭐ Как спроектировать стратегию обновлений для критичного приложения?|25. Стратегия обновлений ⭐]]
- [[#26. Как обеспечить безопасность кластера на нескольких уровнях?|26. Безопасность кластера]]
- [[#27. Как оптимизировать стоимость кластера в облаке?|27. Оптимизация затрат]]
- [[#28. Как отладить проблему с производительностью в кластере?|28. Отладка производительности]]
- [[#29. Как спроектировать мульти-кластерную архитектуру?|29. Мульти-кластер]]
- [[#30. ⭐ Как обеспечить отказоустойчивость на уровне приложения в K8s?|30. Отказоустойчивость ⭐]]

---

## 🟢 Junior (базовое понимание)

### 1. Что такое Kubernetes и зачем он нужен в DevOps?
**Кратко**: Оркестратор контейнеров для автоматизации деплоя, масштабирования и управления приложениями.

**Подробно**: K8s абстрагирует инфраструктуру: вы описываете желаемое состояние (манифесты), а контроллеры приводят кластер к этому состоянию. Это позволяет:
- Декларативно управлять приложениями (GitOps)
- Автоматически восстанавливать сбои (self-healing)
- Масштабировать под нагрузкой (HPA)
- Обновлять без простоя (rolling updates)

**DevOps-контекст**: K8s — платформа для CI/CD: пайплайн собирает образ → пушит в registry → обновляет манифест в Git → ArgoCD синхронизирует кластер.

**Команды**: `kubectl apply`, `kubectl rollout`, `kubectl get pods`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 2. В чём разница между Pod, Deployment и Service?
**Кратко**: 
- `Pod` — минимальная единица (один или несколько контейнеров)
- `Deployment` — управляет репликами подов, обновлениями, откатами
- `Service` — стабильный сетевой эндпоинт для доступа к подам

**Подробно**:
```
[Deployment] → управляет → [ReplicaSet] → создаёт → [Pod x N]
                                      ↓
                              [Service] → балансировка трафика
```

**DevOps-контекст**: В манифестах вы описываете `Deployment`, а не отдельные поды — это позволяет K8s автоматически восстанавливать нужное количество реплик.

**Команды**: `kubectl create deployment`, `kubectl expose`, `kubectl get all`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 3. Как проверить состояние подов и что означают статусы?
**Кратко**: `kubectl get pods`; статусы: `Running`, `Pending`, `CrashLoopBackOff`, `ImagePullBackOff`, `Evicted`.

**Подробно**:
| Статус | Причина | Решение |
|--------|---------|---------|
| `Pending` | Нет ресурсов, не назначен на ноду | `kubectl describe pod`, проверить `events` |
| `ImagePullBackOff` | Ошибка загрузки образа | Проверить имя, теги, `imagePullSecrets` |
| `CrashLoopBackOff` | Приложение падает при старте | `kubectl logs`, `kubectl describe` |
| `Evicted` | Нехватка ресурсов на ноде | Проверить `kubectl describe node`, настроить `requests/limits` |

**DevOps-контекст**: В пайплайнах используйте `kubectl wait --for=condition=ready pod -l app=myapp` для синхронизации этапов.

**Команды**: `kubectl get pods -w`, `kubectl describe pod <name>`, `kubectl logs <pod> --previous`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 4. Что такое Namespace и зачем он нужен?
**Кратко**: Логическое разделение ресурсов внутри одного кластера.

**Подробно**: 
- Изоляция: `dev`, `staging`, `prod` в одном кластере
- Квоты: `ResourceQuota` ограничивает CPU/memory на namespace
- Права доступа: RBAC можно назначать на уровне namespace
- Сетевые политики: `NetworkPolicy` применяются внутри namespace

**DevOps-контекст**: В CI/CD можно деплоить в `pr-<number>` namespace для каждого пул-реквеста, тестируя изолированно.

**Команды**: `kubectl create ns staging`, `kubectl config set-context --current --namespace=prod`, `kubectl get all -n <ns>`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 5. Как получить доступ к приложению внутри кластера?
**Кратко**: Через `Service` + `port-forward` для локальной отладки, или `Ingress` для внешнего доступа.

**Подробно**:
- `ClusterIP`: доступ только внутри кластера (по DNS: `svc.ns.svc.cluster.local`)
- `NodePort`: порт на каждой ноде (для тестов)
- `LoadBalancer`: внешний балансировщик (облако)
- `port-forward`: туннель для локальной отладки

**DevOps-контекст**: В пайплайнах для e2e-тестов используйте `port-forward` или создавайте временный `Ingress` с тестовым доменом.

**Команды**: `kubectl port-forward svc/myapp 8080:80`, `kubectl run curl --image=curlimages/curl -it --rm -- curl http://myapp`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 6. Что такое ConfigMap и Secret? В чём разница?
**Кратко**: Объекты для хранения конфигурации; `Secret` — для чувствительных данных (хранится в base64).

**Подробно**:
- `ConfigMap`: текстовые конфиги, переменные окружения, файлы
- `Secret`: base64-кодированные данные; **не шифрование по умолчанию**!
- Оба монтируются как volume или inject-ятся как env-переменные

**DevOps-контекст**: Для продакшена используйте `External Secrets Operator` или `sops` + `sealed-secrets`, чтобы не хранить секреты в открытом виде в Git.

**Команды**: `kubectl create configmap`, `kubectl create secret generic`, `kubectl get secret <name> -o yaml`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 7. Как обновить приложение без простоя?
**Кратко**: Через `rolling update` в `Deployment`.

**Подробно**: 
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Сколько новых подов создать сверх desired
    maxUnavailable: 0    # Сколько старых можно убить до создания новых
```
K8s постепенно заменяет старые поды на новые, проверяя `readinessProbe`.

**DevOps-контекст**: В пайплайне: `kubectl set image deployment/app app=new-image:v2` → `kubectl rollout status` → при ошибке `kubectl rollout undo`.

**Команды**: `kubectl rollout status`, `kubectl rollout undo`, `kubectl set image`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 8. Что такое livenessProbe и readinessProbe?
**Кратко**: 
- `livenessProbe`: "жив ли процесс?" → если нет, перезапустить под
- `readinessProbe`: "готов ли принимать трафик?" → если нет, исключить из балансировки

**Подробно**:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # Ждём старта приложения
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
```

**DevOps-контекст**: Без `readinessProbe` трафик может пойти на ещё не инициализированный под → ошибки 5xx. Без `livenessProbe` "зависший" процесс не перезапустится.

**Команды**: Проверка через `kubectl describe pod` → раздел `Conditions` и `Events`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 9. Как посмотреть логи приложения?
**Кратко**: `kubectl logs <pod> [-c <container>] [--previous] [-f]`.

**Подробно**:
- `-c`: если в поде несколько контейнеров
- `--previous`: логи предыдущего инстанса (после креша)
- `-f`: follow (как `tail -f`)
- Логи пишутся в stdout/stderr — собираются через `container runtime`

**DevOps-контекст**: Для централизованного логирования используйте `Fluentd`/`Filebeat` → `Elasticsearch`/`Loki`. В CI: `kubectl logs -l app=myapp --tail=100` для пост-деплой проверок.

**Команды**: `kubectl logs -l app=myapp --tail=50 -f`, `kubectl logs <pod> --previous`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 10. Что такое kubectl и как он аутентифицируется?
**Кратко**: CLI-клиент для управления кластером; аутентификация через `kubeconfig` (сертификаты, токены, OIDC).

**Подробно**:
- `~/.kube/config` содержит: кластеры, пользователей, контексты
- Аутентификация: client certificates, service account tokens, OIDC (Keycloak, Google)
- Авторизация: RBAC (`Role`, `RoleBinding`, `ClusterRole`)

**DevOps-контекст**: В CI/CD используйте `ServiceAccount` с минимальными правами + `kubeconfig` из секрета. Никогда не коммитьте `kubeconfig` с правами админа в репо.

**Команды**: `kubectl config view`, `kubectl auth can-i create pods`, `kubectl create serviceaccount`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🟡 Middle (применение, нюансы)

### 11. ⭐ В чём разница между Deployment, StatefulSet и DaemonSet?
**Кратко**: 
- `Deployment`: статеслес приложения, поды взаимозаменяемы
- `StatefulSet`: stateful приложения, стабильные имена/сети/хранилище
- `DaemonSet`: один под на каждую ноду (агенты, мониторинг)

**Подробно**:
| Контроллер | Уникальные имена | Порядок запуска | Хранилище | Пример |
|------------|-----------------|-----------------|-----------|---------|
| Deployment | Нет (рандом) | Параллельно | Общий/эфемерный | Web-приложение |
| StatefulSet | Да (app-0, app-1) | Последовательно | Уникальный PVC на под | БД, Kafka |
| DaemonSet | Нет (привязан к ноде) | На каждой ноде | HostPath/локальный | Prometheus node-exporter |

**DevOps-контекст**: Для баз данных используйте `StatefulSet` + `headless Service` + `StorageClass` с `Retain`. Для агентов — `DaemonSet` с `tolerations` для master-нод.

**Команды**: `kubectl create statefulset`, `kubectl rollout status daemonset`, `kubectl get pvc -l app=mydb`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 12. Как работает Service и kube-proxy?
**Кратко**: `Service` создаёт виртуальный IP (ClusterIP); `kube-proxy` на каждой ноде перенаправляет трафик на поды.

**Подробно**:
- `kube-proxy` режимы:
  - `iptables` (по умолчанию): правила NAT в ядре
  - `ipvs`: балансировщик на уровне ядра (лучше для 1000+ сервисов)
  - `userspace`: устаревший, через proxy-процесс
- DNS: `CoreDNS` резолвит `svc.ns.svc.cluster.local` → ClusterIP

**DevOps-контекст**: При отладке сетевых проблем: `iptables-save | grep <service-ip>`, `kubectl get endpoints <svc>` (показывает реальные поды).

**Команды**: `kubectl get endpoints`, `kubectl run test --rm -it --image=busybox -- nslookup my-svc`, `kubectl get pod -n kube-system -l k8s-app=kube-proxy`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 13. Что такое ResourceQuota и LimitRange?
**Кратко**: 
- `ResourceQuota`: лимиты на весь namespace (суммарно)
- `LimitRange`: дефолтные лимиты на под/контейнер

**Подробно**:
```yaml
# ResourceQuota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    pods: "20"

# LimitRange
spec:
  limits:
  - type: Container
    default:
      memory: 512Mi
      cpu: "500m"
    defaultRequest:
      memory: 256Mi
      cpu: "250m"
```

**DevOps-контекст**: В multi-tenant кластерах обязательно: предотвращает "шумного соседа". В CI: создавайте временный namespace с квотами для тестов.

**Команды**: `kubectl describe quota`, `kubectl describe limitrange`, `kubectl top pods --containers`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 14. Как настроить авто-масштабирование (HPA)?
**Кратко**: `HorizontalPodAutoscaler` меняет количество реплик на основе метрик (CPU, custom metrics).

**Подробно**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**DevOps-контекст**: Требует `metrics-server`. Для custom-метрик (очереди, RPS) — `Prometheus Adapter`. В пайплайнах: нагрузочное тестирование → проверка, что HPA реагирует.

**Команды**: `kubectl autoscale deployment app --cpu-percent=70 --min=2 --max=10`, `kubectl get hpa -w`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 15. Что такое NetworkPolicy и как она работает?
**Кратко**: Правила файрвола на уровне подов (L3/L4); требует CNI с поддержкой (Calico, Cilium, Weave).

**Подробно**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**DevOps-контекст**: Zero-trust security: по умолчанию всё запрещено, разрешаем только необходимый трафик. В GitOps: храните политики в Git вместе с манифестами приложений.

**Команды**: `kubectl get networkpolicy`, `kubectl describe networkpolicy <name>`, `kubectl run test --rm -it --image=curlimages/curl -- curl http://backend`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 16. Как откатить деплой, если что-то пошло не так?
**Кратко**: `kubectl rollout undo deployment/<name>`.

**Подробно**:
- `Deployment` хранит историю `ReplicaSet` (по умолчанию 10 ревизий)
- `rollout undo` возвращает к предыдущему `ReplicaSet`
- Можно откатиться к конкретной ревизии: `--to-revision=3`

**DevOps-контекст**: В пайплайне после деплоя: `kubectl rollout status` → если таймаут/ошибка → `rollout undo` + алерт. Для сложных сценариев: `Argo Rollouts` с canary/analysis.

**Команды**: `kubectl rollout history deployment/app`, `kubectl rollout undo deployment/app --to-revision=2`, `kubectl rollout pause/resume`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 17. Что такое initContainer и когда его использовать?
**Кратко**: Контейнер, который выполняется до основного; для подготовки окружения.

**Подробно**:
- Выполняется последовательно, каждый должен завершиться с кодом 0
- Имеет отдельный образ, но тот же volume/network namespace
- Примеры: миграции БД, загрузка конфигов, ожидание зависимостей

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
```

**DevOps-контекст**: В пайплайнах: initContainer с `curl` для health-check зависимостей перед стартом основного приложения.

**Команды**: `kubectl describe pod <name>` → раздел `Init Containers`, `kubectl logs <pod> -c <init-container>`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 18. Как работает readinessProbe с startupProbe?
**Кратко**: `startupProbe` — для медленных стартов; пока он не пройдёт, `liveness` не проверяется.

**Подробно**:
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30  # 30 * 10s = 5 минут на старт
  periodSeconds: 10
readinessProbe:
  # начинает проверяться только после успешного startupProbe
```

**DevOps-контекст**: Критично для "тяжёлых" приложений (Java, .NET), где старт занимает минуты. Без `startupProbe` `livenessProbe` может убить под до его готовности.

**Команды**: `kubectl describe pod` → раздел `Conditions: Ready`, `kubectl get pod -o jsonpath='{.status.containerStatuses[*].ready}'`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 19. Что такое Taints и Tolerations?
**Кратко**: Механизм "отталкивания" подов от нод (`taint`) и "разрешения" на запуск (`toleration`).

**Подробно**:
```bash
# Добавить taint на ноду
kubectl taint nodes node1 key=value:NoSchedule

# В манифесте пода
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

**DevOps-контекст**: Выделение специализированных нод: `gpu: "true"` для ML, `dedicated=monitoring` для агентов. В CI: запускать тяжёлые тесты только на нодах с `taint=ci:NoSchedule`.

**Команды**: `kubectl describe node <name>` → раздел `Taints`, `kubectl get pods -o wide`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 20. Как отладить проблему с сетью в кластере?
**Кратко**: Пошагово: `kubectl get endpoints` → `nslookup` → `tcpdump` → `iptables`.

**Подробно**:
1. `kubectl get endpoints <svc>` — есть ли поды в балансировке?
2. `kubectl run dns-test --rm -it --image=busybox -- nslookup <svc>` — резолвится ли имя?
3. `kubectl run net-test --rm -it --image=curlimages/curl -- curl -v http://<svc>` — есть ли коннект?
4. На ноде: `iptables-save | grep <service-ip>`, `tcpdump -i any port <port>`

**DevOps-контекст**: В пайплайнах добавляйте шаг с `kubectl run test-pod` для проверки сетевой связности после деплоя.

**Команды**: `kubectl exec <pod> -- nslookup kubernetes.default`, `kubectl debug node/<node> -it --image=nicolaka/netshoot`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🔴 Senior (архитектура, trade-offs, troubleshooting)

### 21. ⭐ Опиши архитектуру Kubernetes: control plane vs worker nodes
**Кратко**: Control plane управляет кластером, worker nodes запускают приложения.

**Подробно**:
```
Control Plane:
├─ kube-apiserver: единая точка входа (аутентификация, авторизация, валидация)
├─ etcd: распределённое key-value хранилище (состояние кластера)
├─ kube-scheduler: назначает поды на ноды (по ресурсам, affinity, taints)
├─ kube-controller-manager: контроллеры (ReplicaSet, Node, Endpoint, etc.)
├─ cloud-controller-manager (опционально): интеграция с облаком

Worker Node:
├─ kubelet: агент, общается с API, управляет контейнерами
├─ kube-proxy: сетевые правила для Service
├─ container runtime: containerd / CRI-O / Docker (устар.)
├─ CNI-плагин: сеть (Calico, Cilium, Flannel)
```

**DevOps-контекст**: Для HA: минимум 3 master-ноды с внешним etcd, load balancer перед apiserver. В managed-сервисах (EKS, GKE) control plane управляется провайдером.

**Команды**: `kubectl get componentstatuses` (устар.), `kubectl get --raw=/readyz?verbose`, `etcdctl endpoint status`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 22. Как работает scheduler и как повлиять на размещение пода?
**Кратко**: Scheduler фильтрует ноды по требованиям, затем ранжирует по скорингу.

**Подробно**:
1. **Filter**: отсеивает ноды, не удовлетворяющие:
   - `resources.requests`
   - `nodeSelector`, `affinity/anti-affinity`
   - `taints/tolerations`
   - `podAffinity` (размещение рядом/вдали от других подов)
2. **Score**: ранжирует подходящие ноды по:
   - балансировке ресурсов
   - `nodeAffinity.weight`
   - `topologySpreadConstraints`

**DevOps-контекст**: Для stateful-приложений: `podAntiAffinity` чтобы реплики не попали на одну ноду. Для GPU: `nodeSelector: gpu-type: a100`.

**Команды**: `kubectl describe pod <name>` → раздел `Events: Scheduled`, `kubectl get nodes --show-labels`, `kubectl debug node/<node>`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 23. Что такое etcd и почему он критичен?
**Кратко**: Распределённое key-value хранилище на основе Raft; хранит всё состояние кластера.

**Подробно**:
- Консистентность: линейная (strong consistency) через Raft
- Хранение: все объекты в `/registry/<group>/<resource>/<name>`
- Производительность: чувствителен к latency диска и сети
- Бэкап: `etcdctl snapshot save` — обязательно для DR

**DevOps-контекст**: Потеря etcd = потеря состояния кластера. Регулярные снапшоты + тестирование восстановления. Мониторинг: `etcd_server_has_leader`, `etcd_disk_wal_fsync_duration_seconds`.

**Команды**: `etcdctl snapshot save backup.db`, `etcdctl snapshot restore`, `kubectl get --raw=/metrics | grep etcd`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 24. Как работает CNI и в чём разница между Calico, Cilium, Flannel?
**Кратко**: CNI-плагин настраивает сеть для подов; выбор влияет на безопасность, производительность, observability.

**Подробно**:
| Плагин | Модель сети | NetworkPolicy | Производительность | Особенности |
|--------|-------------|---------------|-------------------|-------------|
| Flannel | Overlay (VXLAN) | Нет (только через iptables) | Средняя | Простой, для начала |
| Calico | BGP / IPIP | Да (L3/L4) | Высокая | Продвинутые политики, eBPF-режим |
| Cilium | eBPF | Да (L3/L4/L7) | Очень высокая | Observability, security, service mesh |

**DevOps-контекст**: Для zero-trust: Calico/Cilium с `default-deny` политиками. Для observability: Cilium + Hubble для трассировки трафика.

**Команды**: `kubectl get pods -n kube-system -l k8s-app=calico-node`, `calicoctl node status`, `cilium status --verbose`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 25. ⭐ Как спроектировать стратегию обновлений для критичного приложения?
**Кратко**: Rolling update + readinessProbe + canary/analysis + автоматический откат.

**Подробно**:
```yaml
# Deployment
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0  # Ноль простоев

# + Argo Rollouts для canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
```

**DevOps-контекст**: 
1. Сбор метрик до/после деплоя (латентность, ошибки)
2. Автоматический анализ: если ошибка > 1% → откат
3. Feature flags для постепенного включения функционала

**Команды**: `kubectl rollout pause/resume`, `argocd app sync`, `istioctl experimental canary`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 26. Как обеспечить безопасность кластера на нескольких уровнях?
**Кратко**: Defence in depth: сеть, RBAC, образы, runtime, аудит.

**Подробно**:
1. **Сеть**: `NetworkPolicy` (default-deny), `CNI` с eBPF, `service mesh` для mTLS
2. **Доступ**: `RBAC` с минимальными правами, `PodSecurity` (baseline/restricted), `OPA/Gatekeeper` для политик
3. **Образы**: сканирование на уязвимости (Trivy), подписанные образы (cosign), `imagePullSecrets`
4. **Runtime**: `gVisor`/`Kata` для изоляции, `Falco` для детекции аномалий
5. **Аудит**: `audit.log` apiserver, `Falco`, `kube-bench` для CIS-чек

**DevOps-контекст**: В пайплайне: `trivy image`, `checkov`, `kube-score` → блокировка деплоя при критичных уязвимостях.

**Команды**: `kubectl auth can-i --list`, `kube-bench`, `trivy k8s --report=summary cluster`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 27. Как оптимизировать стоимость кластера в облаке?
**Кратко**: Правильный выбор типов нод, автоскейлинг, спотовые инстансы, мониторинг утилизации.

**Подробно**:
- **Ноды**: 
  - `Cluster Autoscaler` + `Node Groups` с разными типами (on-demand + spot)
  - `Karpenter` для быстрого и гибкого скейлинга
- **Поды**:
  - `requests/limits` по фактическому потреблению (анализ через `metrics-server`/`Prometheus`)
  - `VPA` (Vertical Pod Autoscaler) для рекомендаций
- **Хранилище**: 
  - `StorageClass` с `volumeBindingMode: WaitForFirstConsumer`
  - Удаление неиспользуемых `PVC`/`PV`

**DevOps-контекст**: Интеграция с `kube-cost`/`OpenCost` для аллокации затрат по командам. В пайплайнах: проверка, что деплой не превысит бюджет.

**Команды**: `kubectl top nodes --containers`, `kubectl describe node | grep -A 10 Allocatable`, `goldilocks get vpa`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 28. Как отладить проблему с производительностью в кластере?
**Кратко**: Метрики → логирование → трассировка → профилирование.

**Подробно**:
1. **Уровень кластера**: `kubectl top nodes/pods`, `metrics-server`, `Prometheus` + `Grafana`
2. **Уровень ноды**: `node-exporter`, `dmesg`, `iotop`, `perf`
3. **Уровень приложения**: `pprof`, `Jaeger` для трассировки, `log level=debug`
4. **Уровень сети**: `cilium monitor`, `tcpdump`, `hubble observe`

**DevOps-контекст**: Настройте дашборды с ключевыми метриками: 
- `container_cpu_usage_seconds_total`
- `kubelet_pods_started_total`
- `apiserver_request_duration_seconds`

**Команды**: `kubectl debug node/<node> -it --image=nicolaka/netshoot`, `cilium hubble ui`, `kubectl top pods --containers`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 29. Как спроектировать мульти-кластерную архитектуру?
**Кратко**: Разделение по окружениям/регионам + единое управление через Git + federation.

**Подробно**:
- **Паттерны**:
  - `Environment per cluster`: dev/stage/prod в отдельных кластерах
  - `Region per cluster`: низкая задержка, отказоустойчивость
  - `Tenancy per cluster`: изоляция команд/клиентов
- **Управление**:
  - `GitOps`: ArgoCD/Flux с `ApplicationSet` для деплоя в несколько кластеров
  - `Cluster API`: декларативное создание кластеров
  - `Service Mesh`: Istio multi-cluster для единого трафика
- **Синхронизация**:
  - `Secrets`: External Secrets + единый Vault
  - `Config`: ConfigMap через `kustomize` overlays

**DevOps-контекст**: В пайплайне: деплой сначала в `dev` → тесты → `staging` → canary в `prod`. Мониторинг: единый `Thanos`/`Mimir` для агрегации метрик.

**Команды**: `kubectl config use-context prod-eu`, `argocd app list --project prod`, `clusterctl move`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 30. ⭐ Как обеспечить отказоустойчивость на уровне приложения в K8s?
**Кратко**: Комбинация: `PodDisruptionBudget` + `anti-affinity` + `multi-zone` + `circuit breaker`.

**Подробно**:
```yaml
# 1. Распределение по зонам
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [web]
        topologyKey: topology.kubernetes.io/zone

# 2. Защита от массового удаления
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web

# 3. Circuit breaker в приложении
# (реализуется в коде или через service mesh)
```

**DevOps-контекст**: Тестирование отказоустойчивости: 
- `Chaos Mesh` / `Litmus` для инъекции сбоев
- `kubectl drain` для имитации выхода ноды из строя
- Проверка, что `PDB` не даёт удалить больше подов, чем допустимо

**Команды**: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`, `kubectl create pdb`, `chaos-mesh inject network-loss`

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🧪 Чек-лист подготовки к собеседованию

- [ ] Могу объяснить разницу между `Pod`, `Deployment`, `Service` за 1 минуту
- [ ] Понимаю, как работает `kube-proxy` и DNS в кластере
- [ ] Могу отладить `CrashLoopBackOff` по шагам
- [ ] Знаю, как настроить `HPA` и какие метрики доступны
- [ ] Понимаю trade-offs между `Calico`/`Cilium`/`Flannel`
- [ ] Могу спроектировать безопасный деплой с `NetworkPolicy` + `RBAC`
- [ ] Понимаю, как работает `etcd` и как делать бэкап
- [ ] Могу объяснить, как выбрать стратегию обновлений для критичного сервиса
- [ ] Знаю, как оптимизировать стоимость кластера в облаке
- [ ] Понимаю принципы мульти-кластерной архитектуры

> [!TIP] Практика
> Лучшая подготовка — развернуть локальный кластер и пройти сценарии из [[Kubernetes практика]].  
> Затем ответить на вопросы из этого файла без подглядывания в документацию.
