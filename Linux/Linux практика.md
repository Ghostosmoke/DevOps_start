

---

### 1️⃣ БЛОК ТЕОРИИ (Практические концепции)

#### 🎯 Что значит "практика в Linux" для DevOps?
**Простыми словами:** Это не знание всех флагов `tar`, а умение за 5 минут найти причину падения сервиса, восстановить место на диске, безопасно обновить ядро и написать скрипт, который не упадёт в 3 ночи.

**Ключевые практические области:**
```
┌─────────────────────────────────────────────────────────┐
│              DevOps Linux Practice Stack                │
├─────────────────────────────────────────────────────────┤
│ 🔍 Диагностика: strace, lsof, ss, journalctl, perf      │
│ 💾 Диски: df, du, lsof +L1, xfs_repair, LVM, snapshots  │
│ 🧵 Процессы: ps, top, kill, renice, cgroups, systemd    │
│ 🌐 Сеть: ss, tcpdump, iptables/nft, sysctl, netns       │
│ 🔐 Безопасность: auditd, fail2ban, SELinux, capabilities│
│ ⚡ Производительность: vmstat, iostat, pidstat, bpftrace│
│ 🤖 Автоматизация: bash, systemd timers, cron, inotify   │
└─────────────────────────────────────────────────────────┘
```

#### 🔧 Принципы практической работы
1. **Non-destructive first:** Сначала читай (`cat`, `less`, `ss`), потом меняй. Всегда имей план отката.
2. **Idempotency:** Скрипты должны быть безопасны при повторном запуске (`mkdir -p`, `grep -q || echo ...`).
3. **Observability by default:** Логируй действия, используй `set -x`, `logger`, `systemd-cat`.
4. **Least privilege:** Не работай под root без необходимости. Используй `sudo -u`, `capabilities`, `namespaces`.
5. **Document as you go:** Команды, которые сработали — сохраняй в runbook.

#### 📦 Реальные кейсы и антипаттерны
| Ситуация | Практическое решение | Почему это работает |
|----------|---------------------|-------------------|
| **"Место есть, но писать нельзя"** | `df -i` → `find / -xdev -type f -mmin -60 | wc -l` → `lsof +L1` | Inodes или открытые удалённые файлы — частая причина в CI-кэшах и логах |
| **Сервис "висит", но не падает** | `strace -p <PID> -T -tt` → `gdb -p <PID> -ex "bt"` → `kill -SIGQUIT` | Показывает, на каком syscall процесс заблокирован |
| **Высокий load, но низкий CPU** | `vmstat 1`, `iostat -x 1`, `pidstat -d 1` → ищем I/O wait | Load average включает процессы в uninterruptible sleep (D-state) |
| **Контейнер не резолвит имена** | `nsenter -t <PID> -n cat /etc/resolv.conf` → `tcpdump -n port 53` | Проверяем network namespace и DNS-трафик изолированно |
| **После обновления перестал работать сервис** | `journalctl -u service --since "1 hour ago" -p err` → `rpm -V package` / `debsums` | Журналы + проверка целостности файлов после апдейта |

#### 🔗 Связь с инструментами
- **Ansible:** Использует `ssh` + Python, но для speed-up нужен `pipelining`, `fact caching`.
- **Terraform:** Локальные `provisioners` выполняют bash-скрипты — они должны быть идемпотентны.
- **K8s:** `kubectl debug` запускает ephemeral container с теми же namespaces — это `nsenter` на стероидах.
- **CI/CD:** Раннеры в Docker используют `--init`, `--cap-drop`, `--read-only` — практика Linux напрямую влияет на безопасность пайплайнов.

#### 🔑 Ключевые моменты
- `strace` — твой лучший друг при отладке. Учись читать вывод.
- `systemd` управляет не только сервисами, но и cgroups, журналами, таймерами, сокетами.
- Сеть в Linux — это не только `ip a`, но и `ss`, `nftables`, `sysctl`, `netns`, `tc`.
- Автоматизация без идемпотентности = технический долг.

#### ⚠️ Частые ошибки
- `rm -rf /tmp/*` без проверки — можно удалить `/tmp/ssh-XXX/agent.YYY`.
- `kill -9` без попытки graceful shutdown.
- Игнорирование `dmesg -T` при проблемах с железом/драйверами.
- Хардкод путей в скриптах (`/home/user` вместо `$HOME`).
- Отсутствие таймаутов в `curl`/`wget` → зависшие пайплайны.

---

### 2️⃣ БЛОК ПРАКТИКИ (Команды, скрипты, конфиги)

#### 🔍 Диагностика: "Что происходит прямо сейчас?"
```bash
# 🎯 Быстрый аудит системы (сохрани в ~/bin/audit.sh)
#!/usr/bin/env bash
set -euo pipefail
echo "=== LOAD & CPU ==="
uptime
ps aux --sort=-%cpu | head -10
echo -e "\n=== MEMORY ==="
free -h
echo -e "\n=== DISK ==="
df -h / /var /tmp 2>/dev/null || true
df -i / 2>/dev/null | grep -v Filesystem || true
echo -e "\n=== TOP NETWORK ==="
ss -tulpn | grep -E ':(80|443|8080|22)' || true
echo -e "\n=== RECENT ERRORS ==="
journalctl -p err --since "10 min ago" --no-pager | tail -20

# 🎯 Найти процесс, держащий удалённый файл (освобождает место)
lsof +L1 | awk '$7 ~ /deleted/ {print $1, $2, $4, $10}' | column -t

# 🎯 Отследить, почему процесс не пишет в лог (открыт ли файл?)
PID=$(pgrep -f myapp)
ls -l /proc/$PID/fd/ | grep log
strace -p $PID -e trace=open,write -s 100 -T 2>&1 | grep -E 'open|write'

# 🎯 Проверить, не упирается ли приложение в лимиты
cat /proc/$(pgrep myapp)/limits | grep -E 'open files|memory|cpu'
```

#### 💾 Диски и файловая система: "Где место?"
```bash
# 🎯 Найти самые большие директории (без учёта смонтированных FS)
du -h / --max-depth=2 2>/dev/null | grep -vE 'proc|sys|dev|run' | sort -hr | head -15

# 🎯 Найти файлы >100MB, изменённые за последние 7 дней
find / -xdev -type f -size +100M -mtime -7 -exec ls -lh {} \; 2>/dev/null | sort -k5 -hr

# 🎯 Очистить старые ядра (Debian/Ubuntu)
sudo apt autoremove --purge -y && sudo update-grub

# 🎯 Очистить старые ядра (RHEL/CentOS)
sudo package-cleanup --oldkernels --count=2 -y  # требует yum-utils

# 🎯 Безопасная очистка логов (без удаления inode)
find /var/log -name "*.log" -size +100M -exec sh -c '> "{}"' \;

# 🎯 Создать snapshot LVM (перед апдейтом)
sudo lvcreate -L 10G -s -n root_snap /dev/vg0/root
# Откат:
sudo umount / && mount /dev/vg0/root_snap / && reboot
```

#### 🧵 Процессы и systemd: "Управление жизненным циклом"
```bash
# 🎯 Создать systemd service с healthcheck и лимитами
# /etc/systemd/system/myapp.service
[Unit]
Description=My Production Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appgroup
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

# 🔒 Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/myapp /var/lib/myapp
PrivateTmp=true
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
SystemCallFilter=@system-service

# 💾 Resource limits (cgroups v2)
MemoryMax=1G
CPUQuota=150%
TasksMax=100

# 📝 Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target

# Применить:
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp

# 🎯 Graceful reload без даунтайма
sudo systemctl reload myapp  # если приложение поддерживает SIGHUP
# Или:
sudo systemctl restart myapp  # systemd сам дождётся TimeoutStopSec

# 🎯 Отследить, почему сервис не стартует
systemctl start myapp
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
# Если зависает на старте:
strace -f -p $(systemctl show myapp -p MainPID --value) -e trace=network,file
```

#### 🌐 Сеть: "Почему не подключается?"
```bash
# 🎯 Проверить, слушает ли приложение порт
ss -tlnp | grep :8080
# Если не видно — проверить, в каком network namespace процесс
PID=$(pgrep myapp)
nsenter -t $PID -n ss -tlnp | grep :8080

# 🎯 Протестировать подключение из другого namespace (как в K8s)
nsenter -t $(pgrep nginx) -n curl -v http://localhost:8080

# 🎯 Отследить DNS-резолвинг
strace -e trace=network curl -v http://example.com 2>&1 | grep -E 'connect|getaddrinfo'

# 🎯 Настроить sysctl для highload (в /etc/sysctl.d/99-highload.conf)
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65000
vm.swappiness = 1
# Применить:
sudo sysctl -p /etc/sysctl.d/99-highload.conf

# 🎯 Временно открыть порт для отладки (не в production!)
sudo iptables -I INPUT -p tcp --dport 9999 -s 10.0.0.0/8 -j ACCEPT
# Проверить:
sudo iptables -L INPUT -n -v | grep 9999
# Удалить:
sudo iptables -D INPUT -p tcp --dport 9999 -s 10.0.0.0/8 -j ACCEPT
```

#### 🔐 Безопасность: "Защита по умолчанию"
```bash
# 🎯 Аудит прав доступа к чувствительным файлам
find /etc /opt /usr/local -type f \( -name "*.conf" -o -name "*.key" -o -name "*.pem" \) -exec ls -l {} \; 2>/dev/null | grep -v '^-rw-r--r--'

# 🎯 Проверить capabilities у бинарников
getcap -r / 2>/dev/null | grep -v 'Permission denied'

# 🎯 Включить auditd для мониторинга доступа к /etc/shadow
sudo auditctl -w /etc/shadow -p wa -k shadow_access
# Просмотр:
sudo ausearch -k shadow_access -i

# 🎯 Ограничить пользователя в chroot (упрощённо)
sudo useradd -m -s /bin/rbash restricted
sudo mkdir -p /home/restricted/bin
sudo ln -s /bin/ls /home/restricted/bin/
# В ~/.bash_profile restricted:
# export PATH=$HOME/bin
# export SHELL=/bin/rbash

# 🎯 Fail2ban для защиты SSH (установка + базовая настройка)
sudo apt install fail2ban -y
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 1h
findtime = 10m
sudo systemctl enable --now fail2ban
```

#### 🤖 Автоматизация: "Надёжные скрипты"
```bash
#!/usr/bin/env bash
# 🎯 Шаблон продакшен-скрипта
set -euo pipefail  # Ошибки прерывают выполнение
IFS=$'\n\t'        # Безопасный парсинг
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="/var/log/myscript.log"
LOCK_FILE="/var/run/myscript.lock"

# Логирование
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# Блокировка от параллельного запуска
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Already running"; exit 1; }

# Cleanup при выходе
cleanup() { log "Cleanup..."; rm -f "$LOCK_FILE"; }
trap cleanup EXIT INT TERM

# Проверка зависимостей
command -v curl >/dev/null || { log "curl not found"; exit 2; }

# Основная логика с таймаутами и ретраями
main() {
  log "Starting..."
  for i in {1..3}; do
    if curl -sf --max-time 10 http://api.example.com/health; then
      log "Healthcheck passed"
      break
    fi
    log "Retry $i failed, waiting..."
    sleep $((2 ** i))  # Exponential backoff
  done
}

main "$@"
```
---
