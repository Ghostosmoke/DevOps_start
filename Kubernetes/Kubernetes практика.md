# 📄 Файл: `Kubernetes практика.md`


tags: [kubernetes, devops, practice, hands-on]
aliases: [k8s-practice, kube-practice]
created: 2026-05-07
---

# 🛠️ Kubernetes для DevOps: Полноценная Практика (Hands-On)

> [!INFO] Формат
> Реальные сценарии из продакшена → пошаговое выполнение → DevOps-контекст → задания для самостоятельной отработки.
> 
> 💡 **Рекомендация**: Используй `minikube`, `kind` или `k3d` для локальной практики. Не экспериментируй с `kubectl delete --all` в продакшен-кластерах.

📋 [[#🧪 Чек-лист самостоятельной практики|Чек-лист]] | [[#⚠️ Топ-5 ошибок в продакшене|Ошибки]] | [[#🔗 Связь с другими файлами|Связи]]

---

## 📁 СЦЕНАРИЙ 1: Развёртывание первого приложения в Pod

### 🎯 Цель
Запустить простое веб-приложение в изолированном контейнере внутри кластера.

### 📋 Пошаговое выполнение

```bash
# 1. Проверка подключения к кластеру
kubectl cluster-info
kubectl get nodes

# 2. Создание манифеста Pod (pod-nginx.yaml)
cat > pod-nginx.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    app: web
    env: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.25-alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF

# 3. Применение конфигурации
kubectl apply -f pod-nginx.yaml

# 4. Проверка статуса
kubectl get pods -w                    # Watch за изменениями
kubectl describe pod nginx-demo       # Детали: события, условия
kubectl logs nginx-demo               # Логи контейнера

# 5. Доступ к приложению (локально)
kubectl port-forward pod/nginx-demo 8080:80
# Теперь: curl http://localhost:8080
```

### 🔍 DevOps-контекст
- `Pod` — минимальная единица развёртывания в K8s, может содержать несколько контейнеров с общим networking/storage.
- `resources.requests/limits` критичны для планирования: без `limits` под может потреблять 100% ноды.
- `port-forward` используется для отладки, но **не для продакшена** — там нужны `Service`/`Ingress`.

### ⚠️ Подводные камни
- `ImagePullBackOff` — проверь имя образа, доступ к registry, `imagePullSecrets`.
- `CrashLoopBackOff` — приложение падает: смотри `kubectl logs` и `kubectl describe`.
- Pod без `livenessProbe` может "зависнуть" и не перезапуститься автоматически.

### 🧪 Задание для отработки
1. Создай Pod с образом `busybox`, который выполняет `sleep 3600`.
2. Зайди внутрь: `kubectl exec -it <pod> -- sh`.
3. Создай файл внутри контейнера, выйди, проверь, что файл остался.
4. Удали Pod: `kubectl delete pod <name>`. Что произошло с файлом? Почему?

---

## 📁 СЦЕНАРИЙ 2: Деплой приложения через Deployment + Service

### 🎯 Цель
Отработать паттерн: масштабируемое приложение + сетевой доступ внутри кластера.

### 📋 Пошаговое выполнение

```bash
# 1. Манифест Deployment (deployment-app.yaml)
cat > deployment-app.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: ghcr.io/ghostosmoke/demo-app:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "postgres.default.svc.cluster.local"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
EOF

# 2. Манифест Service (service-app.yaml)
cat > service-app.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
EOF

# 3. Применение и проверка
kubectl apply -f deployment-app.yaml -f service-app.yaml
kubectl get deployments,replicasets,pods,services
kubectl rollout status deployment/web-app

# 4. Тестирование внутри кластера
kubectl run curl-test --image=curlimages/curl -it --rm -- \
  curl http://web-app-svc.default.svc.cluster.local
```

### 🔍 DevOps-контекст
- `Deployment` управляет `ReplicaSet`, который обеспечивает желаемое количество реплик.
- `readinessProbe` исключает под из балансировки, пока приложение не готово принимать трафик.
- `ClusterIP` Service создаёт виртуальный IP, доступный **только внутри кластера**.
- DNS-имя: `<service>.<namespace>.svc.cluster.local` — основа service discovery в K8s.

### ⚠️ Подводные камни
- Неправильный `selector` в Service → поды не попадут в балансировку (проверяй `kubectl get endpoints`).
- `initialDelaySeconds` слишком мал → приложение не успело стартовать → рестарты.
- Забыл `resources.limits` → QoS-класс `BestEffort` → под первым убьётся при нехватке памяти.

### 🧪 Задание для отработки
1. Измени `replicas: 5`, примени, проверь масштабирование.
2. Удали один под вручную: `kubectl delete pod <name>`. Что сделал Deployment?
3. Обнови образ на `v2.0`, запусти `kubectl rollout undo` — что произошло?
4. Поэкспериментируй с `strategy.rollingUpdate.maxSurge/maxUnavailable`.

---

## 📁 СЦЕНАРИЙ 3: Работа с ConfigMap и Secret

### 🎯 Цель
Научиться выносить конфигурацию и секреты из кода приложения.

### 📋 Пошаговое выполнение

```bash
# 1. Создание ConfigMap (из файла и литералов)
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# Или из файла:
cat > config.properties <<EOF
database.url=jdbc:postgresql://db:5432/app
cache.ttl=3600
EOF
kubectl create configmap app-config-file --from-file=config.properties

# 2. Создание Secret (base64-кодирование)
echo -n "supersecret" | base64
# Создаём через manifest (безопаснее --dry-run + kustomize/sops в продакшене)
cat > secret-db.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Kubernetes сам закодирует в base64
  username: admin
  password: supersecret
EOF
kubectl apply -f secret-db.yaml

# 3. Монтирование в Pod
cat > pod-with-config.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config-file
EOF
kubectl apply -f pod-with-config.yaml

# 4. Проверка внутри контейнера
kubectl exec -it app-with-config -- sh
$ echo $LOG_LEVEL
$ cat /etc/config/config.properties
```

### 🔍 DevOps-контекст
- `ConfigMap`/`Secret` позволяют менять конфигурацию без пересборки образа.
- `Secret` хранится в etcd в base64 — **не шифрование**! Для продакшена: `EncryptionConfiguration`, `External Secrets Operator`, `Vault`.
- Обновление `ConfigMap` не перезапускает поды автоматически — нужен `reloader` (stakater/Reloader) или ручное обновление.

### ⚠️ Подводные камни
- `stringData` в манифестах виден в Git — используй `sops`, `sealed-secrets` или `external-secrets`.
- Большие `ConfigMap` (>1MB) могут не смонтироваться — лимит etcd.
- Изменение `Secret` требует перезапуска пода, если не используется volume-mount с авто-обновлением.

### 🧪 Задание для отработки
1. Создай `ConfigMap` с JSON-конфигом, смонтируй как volume.
2. Обнови значение в `ConfigMap`, проверь, изменилось ли оно внутри пода.
3. Установи `stakater/reloader`, настрой авто-рестарт при изменении конфига.
4. Попробуй `kubectl create secret generic ... --from-env-file=.env` — в чём риск?

---

## 📁 СЦЕНАРИЙ 4: Хранение данных: PersistentVolume + PVC

### 🎯 Цель
Обеспечить сохранность данных при перезапуске/перемещении пода.

### 📋 Пошаговое выполнение

```bash
# 1. Для локального кластера: создаём StorageClass (если нет)
# Minikube: default storageclass уже есть (standard)
kubectl get storageclass

# 2. Манифест PersistentVolumeClaim (pvc.yaml)
cat > pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF
kubectl apply -f pvc.yaml

# 3. Pod с монтированием тома
cat > pod-pvc.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo \$(date) >> /data/log.txt; sleep 10; done"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: app-data-pvc
EOF
kubectl apply -f pod-pvc.yaml

# 4. Проверка
kubectl exec -it app-with-storage -- cat /data/log.txt
kubectl delete pod app-with-storage
# Создаём под заново с тем же PVC — данные сохранились!
```

### 🔍 DevOps-контекст
- `PVC` — запрос на хранилище; `PV` — реальный ресурс (статический или динамический через `StorageClass`).
- `accessModes`: `ReadWriteOnce` (одна нода), `ReadOnlyMany`, `ReadWriteMany` (требует NFS/ceph).
- В облаках: `StorageClass` с `provisioner: ebs.csi.aws.com` / `disk.csi.gke.io` создаёт диски автоматически.

### ⚠️ Подводные камни
- Удаление `PVC` по умолчанию удаляет и `PV` с данными (`reclaimPolicy: Delete`) — в продакшене используй `Retain`.
- `ReadWriteMany` не поддерживается многими облачными дисками — нужен отдельный solution (EFS, Filestore, NFS).
- Не все приложения поддерживают одновременный доступ к одному тому — проверяй документацию.

### 🧪 Задание для отработки
1. Создай `StatefulSet` с 3 репликами, каждая со своим `PVC`.
2. Удали один под, проверь, что данные на томе сохранились и под подключился к тому же тому.
3. Поэкспериментируй с `volumeClaimTemplates` в StatefulSet.
4. Настрой `StorageClass` с `allowVolumeExpansion: true`, увеличь размер `PVC` на лету.

---

## 📁 СЦЕНАРИЙ 5: Доступ извне: Ingress + TLS

### 🎯 Цель
Настроить внешний доступ к приложению с автоматическим выпуском HTTPS-сертификата.

### 📋 Пошаговое выполнение

```bash
# 1. Установка Ingress Controller (для kind/minikube)
# Minikube:
minikube addons enable ingress

# Kind + Metallb + ingress-nginx (упрощённо):
# https://kind.sigs.k8s.io/docs/user/ingress/

# 2. Манифест Ingress (ingress-app.yaml)
cat > ingress-app.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod  # если используется cert-manager
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

# 3. Установка cert-manager (для автоматических сертификатов)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=available --timeout=60s deployment/cert-manager -n cert-manager

# 4. ClusterIssuer для Let's Encrypt (требует email и настройку DNS-01 или HTTP-01)
# Пример HTTP-01 (требует публичный IP и DNS):
cat > cluster-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
kubectl apply -f cluster-issuer.yaml

# 5. Применение и проверка
kubectl apply -f ingress-app.yaml
kubectl get ingress,certificate
# После настройки DNS: проверить https://app.example.com
```

### 🔍 DevOps-контекст
- `Ingress` — L7-маршрутизатор, заменяет множество `LoadBalancer` Services.
- `cert-manager` автоматизирует выпуск и обновление сертификатов — критично для продакшена.
- `ingressClassName` заменяет устаревшую аннотацию `kubernetes.io/ingress.class`.

### ⚠️ Подводные камни
- `cert-manager` требует публичного доступа к `.well-known/acme-challenge` для HTTP-01.
- DNS-пропагация может занимать время — используй `dig` для проверки.
- Ingress без `tls` секции работает только по HTTP — не забывай про редирект (аннотация `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"`).

### 🧪 Задание для отработки
1. Настрой Ingress с двумя путями: `/api` → сервис API, `/` → фронтенд.
2. Добавь базовую аутентификацию через аннотации nginx-ingress (`auth-type: basic`).
3. Настрой rate-limiting через аннотации.
4. Протестируй обновление сертификата: удали `secretName`, дождись авто-выпуска.

---

## 📁 СЦЕНАРИЙ 6: Отладка и мониторинг в K8s

### 🎯 Цель
Научиться быстро диагностировать проблемы в работающем кластере.

### 📋 Пошаговое выполнение

```bash
# 1. Быстрый обзор состояния
kubectl get all -A --show-labels
kubectl top nodes
kubectl top pods -A

# 2. Детальная диагностика пода
kubectl describe pod <pod-name>          # События, условия, причины падений
kubectl logs <pod> -c <container>        # Логи конкретного контейнера
kubectl logs <pod> --previous            # Логи предыдущего инстанса (после рестарта)
kubectl exec -it <pod> -- sh             # Интерактивный доступ

# 3. Проверка сети и DNS
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default
kubectl run net-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -v http://<service>.<namespace>.svc.cluster.local

# 4. Отладка ресурсов
kubectl explain pod.spec.containers.resources    # Документация прямо в CLI
kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get --show-kind --ignore-not-found -n <namespace>

# 5. Временный доступ для отладки (ephemeral containers — alpha/beta)
kubectl debug -it <pod> --image=busybox --target=<container-name>
```

### 🔍 DevOps-контекст
- `kubectl describe` — первый инструмент при любом инциденте: события (`Events`) часто содержат корневую причину.
- `--previous` для логов спасает при `CrashLoopBackOff`.
- `ephemeral containers` позволяют подключиться к работающему поду без изменения его манифеста (требует `debug`-образа с нужными утилитами).

### ⚠️ Подводные камни
- Логи в stdout/stderr не сохраняются после удаления пода — для долгосрочного хранения нужен `Fluentd`/`Loki`.
- `kubectl exec` не работает, если контейнер упал или не имеет shell — используй `debug`-контейнеры или sidecar-логирование.
- `top` требует `metrics-server` — если не установлен, будет ошибка.

### 🧪 Задание для отработки
1. Намеренно создай ошибку в манифесте (неверный образ), примени, найди причину через `describe`.
2. Сымитируй утечку памяти: задай `limits.memory: 64Mi`, запусти приложение, которое потребляет больше — отследи OOMKilled.
3. Настрой `metrics-server`, построй простой график CPU в `kubectl top`.
4. Используй `kubectl debug` для подключения к поду без bash.

---

## 🛠️ Полезные алиасы и утилиты

```bash
# ~/.bashrc или ~/.zshrc
alias k=kubectl
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kga='kubectl get all'
alias kgn='kubectl get nodes -o wide'
alias kdp='kubectl describe pod'
alias klg='kubectl logs --tail=50 -f'
alias kx='kubectl exec -it'
alias kctx='kubectl config use-context'

# Плагины (kubectl plugins)
# Установка: https://krew.sigs.k8s.io/docs/user-guide/setup/install/
kubectl krew install ctx ns view-utilization popeye tree

# Примеры:
k ctx          # Переключение между кластерами
k ns production # Переключение namespace
k popeye     # Аудит конфигураций на лучшие практики
k tree deployment/my-app  # Визуализация связанных ресурсов
```

---

## 🧪 Чек-лист самостоятельной практики

- [ ] Развернул локальный кластер (`minikube`/`kind`/`k3d`)
- [ ] Запустил приложение в Pod, получил доступ через `port-forward`
- [ ] Создал Deployment с 3 репликами + Service, протестировал внутри кластера
- [ ] Вынес конфигурацию в ConfigMap, секрет — в Secret, смонтировал в под
- [ ] Настроил PersistentVolumeClaim, проверил сохранность данных при рестарте
- [ ] Развернул Ingress + cert-manager, получил HTTPS-доступ (локально или на тестовом домене)
- [ ] Отладил падающий под через `describe`, `logs --previous`, `debug`
- [ ] Установил `krew` + плагины, настроил алиасы для ускорения работы
- [ ] Написал простой `Helm` chart или `Kustomize` overlay для своего приложения
- [ ] Документировал свой стенд в README.md с инструкцией по развёртыванию

---

## ⚠️ Топ-5 ошибок в продакшене и как их избежать

| Ошибка | Последствия | Решение |
|--------|-------------|---------|
| Нет `resources.limits` | Под потребляет 100% ноды → eviction других подов | Всегда задавай `requests` и `limits`, используй `LimitRange` |
| `latest` тег в продакшене | Непредсказуемые обновления, сложно откатиться | Фиксируй образы по хешу: `nginx@sha256:abc...` |
| Один namespace для всего | Сложность управления квотами, RBAC, сетевыми политиками | Разделяй по окружениям: `prod`, `staging`, `dev` |
| Нет `livenessProbe` | "Зависшее" приложение не перезапускается автоматически | Добавляй `liveness` + `readiness` для всех критичных сервисов |
| Хранение state в ephemeral storage | Потеря данных при перезапуске пода | Используй `PVC` + `StorageClass`, тестируй восстановление |
