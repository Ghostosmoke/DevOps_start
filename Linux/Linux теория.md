Подтверждаю тему: **Linux (DevOps Core)**. 
Linux — фундаментальная тема для DevOps. Она огромна, поэтому в этом гайде я сфокусируюсь на **ключевых аспектах, которые реально используются в высоконагруженных системах, CI/CD и контейнеризации**. Если потребуется углубление в подтемы (например, `Linux Networking`, `Storage & Filesystems`, `Kernel Tuning` или `eBPF`), дай знать — разобьём на отдельные части.

---

### 1️⃣ БЛОК ТЕОРИИ

#### 🌍 Суть темы: от простого к глубокому
**Простыми словами:** Linux — это операционная система, состоящая из ядра (управляет железом, памятью, процессами) и пользовательского окружения (утилиты, оболочки, демоны). Для DevOps Linux — это "рабочая лошадка": на нём живут контейнеры, CI-раннеры, мониторинг, балансировщики и сами приложения.

**Архитектура и принципы работы:**
```
┌─────────────────────────────────────────────────────────┐
│                   User Space (Ring 3)                   │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │  Shell   │  │  Docker  │  │  NGINX   │  │  Apps    │ │
│  │ (bash/zsh)│  │  /K8s    │  │  /Proxy  │  │  (Go/Py) │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       └─────────────┼─────────────┼─────────────┘       │
│                     ▼ (Syscalls: open, read, write...)  │
├─────────────────────────────────────────────────────────┤
│                   Kernel Space (Ring 0)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │  VFS /   │ │ Process  │ │ Memory   │ │ Network     │ │
│  │  FS      │ │ Scheduler│ │ Manager  │ │ Stack/Netfilter│
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬──────┘ │
│       └─────────────┼─────────────┼────────────┘        │
│                     ▼ (Hardware Abstraction Layer)      │
├─────────────────────────────────────────────────────────┤
│                    Hardware (CPU/RAM/Disk/NIC)          │
└─────────────────────────────────────────────────────────┘
```
- **Syscalls:** Единственный легальный способ User Space взаимодействует с ядром. `strace` трассирует именно их.
- **Namespaces:** Изолируют ресурсы (PID, NET, MNT, UTS, IPC, USER). Основа контейнеров.
- **cgroups (Control Groups):** Ограничивают и измеряют ресурсы (CPU, RAM, IO, Network). Без них контейнер мог бы съесть всю память хоста.
- **VFS (Virtual File System):** Абстракция поверх ext4/xfs/btrfs. Позволяет единым API работать с разными ФС, сетевыми шарами, `/proc`, `/sys`.

#### 🔹 Загрузка системы (Boot Flow)
`BIOS/UEFI → Bootloader (GRUB) → Kernel (initramfs) → systemd (PID 1) → Targets → User Services`
- `initramfs` — минимальная ФС в RAM, подгружает драйверы для монтирования корневого раздела.
- `systemd` — заменяет SysVinit. Управляет сервисами, зависимостями, журналированием, cgroups.

#### 📦 Реальные кейсы и антипаттерны
| Ситуация | Что происходит | Как избежать |
|----------|----------------|--------------|
| **OOM Killer убивает БД** | Процесс превысил лимит памяти, ядро выбирает жертву по `oom_score` | Настраивать `MemoryLimit=` в systemd или `--memory` в Docker. Использовать `oom_score_adj` для критичных сервисов. |
| **Удалённые файлы не освобождают место** | Дескриптор открыт процессом, inode не освобождён до закрытия | `lsof +L1` → перезапустить сервис или очистить файл `> /proc/PID/fd/N` |
| **`kill -9` вместо graceful shutdown** | Приложение не успевает закрыть соединения, сбросить кэш, сохранить состояние | Всегда сначала `SIGTERM` (`kill -15`), ждать, только потом `SIGKILL` (`kill -9`). Настроить `TimeoutStopSec=` в systemd. |
| **`chmod 777` для "быстрого фикса"** | Снимает все ограничения безопасности, открывает вектор атак | Использовать ACL (`setfacl`), группы, capabilities (`setcap`), запускать от непривилегированного пользователя. |

#### 🔗 Связь с экосистемой
- **Docker/K8s:** Используют `namespaces` + `cgroups` + `overlayfs` из ядра Linux.
- **CI/CD:** Раннеры (GitLab, GitHub Actions) работают в изолированных Linux-контейнерах/виртуалках.
- **Monitoring:** `node_exporter` читает `/proc`, `/sys`, `sysctl`. `tcpdump`/`eBPF` перехватывают сетевые пакеты на уровне ядра.

#### 🔑 Ключевые моменты
- Ядро обрабатывает прерывания, память, процессы, устройства. User Space работает через syscalls.
- `systemd` управляет жизненным циклом сервисов и cgroups.
- Контейнеры ≠ виртуальные машины. Они используют то же ядро, но изолированы через namespaces/cgroups.
- Диагностика всегда идёт сверху вниз: `systemctl/journalctl` → `top/htop` → `strace/lsof` → `dmesg`.

#### ⚠️ Частые ошибки
- Игнорирование `df -i` (inodes заканчиваются раньше места).
- Запуск сервисов от `root` без необходимости.
- Слепое использование `kill -9`.
- Отсутствие лимитов в `limits.conf` или `systemd` → OOM, starvation.
- Неумение читать `strace`/`lsof` при зависаниях.

---

### 2️⃣ БЛОК ПРАКТИКИ

#### 🔧 Управление процессами и ресурсами
```bash
# Локально: найти процесс, использующий CPU
top -b -n 1 | head -20  # -b: batch mode, -n 1: 1 итерация

# Продакшен: безопасно завершить сервис с grace period
systemctl stop myapp.service  
# systemd пошлёт SIGTERM, подождёт TimeoutStopSec (по умолч. 90s), потом SIGKILL

# Продвинутый мониторинг: отследить syscalls процесса
strace -p <PID> -T -tt -e trace=open,read,write,connect 2>&1 | head -50
# -T: время выполнения, -tt: точное время, -e: фильтрация
```

#### 💾 Диски и файловая система
```bash
# Проверка места и inodes (частая причина "место есть, но писать нельзя")
df -h   # Human-readable, блоки
df -i   # Inode usage

# Найти файлы, удалённые, но всё ещё удерживаемые процессами
lsof +L1 | awk '{print $1, $2, $4, $10}' | sort -k2 -n

# Очистка без перезапуска (осторожно! только для логов/кэшей)
> /var/log/app.log  # Truncate без удаления inode
```

#### 🛡️ Systemd Service (Production Best Practices)
`/etc/systemd/system/myapp.service`
```ini
[Unit]
Description=My Highload Service
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
User=appuser
Group=appgroup
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s

# 🔒 Ограничения ресурсов (cgroups v2)
MemoryMax=2G
CPUQuota=200%
# 📝 Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```
Применение: `sudo systemctl daemon-reload && sudo systemctl enable --now myapp`

#### 🌐 Сеть и диагностика
```bash
# Современная замена netstat
ss -tulpn | grep :8080  # -t: TCP, -u: UDP, -l: listen, -p: process, -n: numeric

# Диагностика зависших соединений
ss -tnp state close-wait   # Приложение не закрыло socket
ss -tnp state time-wait    # Нормально при высокой нагрузке, регулируется sysctl

# Тонкая настройка ядра (без перезагрузки)
sudo sysctl -w net.core.somaxconn=4096
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sudo sysctl -p  # Применить из /etc/sysctl.conf
```

#### 📝 Best Practices для CI/CD и скриптов
```bash
#!/usr/bin/env bash
set -euo pipefail  # Ошибка прерывает скрипт, unset переменные → ошибка, pipefail → учитывает ошибки в пайпах
IFS=$'\n\t'        # Безопасный парсинг строк

# CI: неблокирующий curl с retry
for i in {1..5}; do
  curl -sf --max-time 5 http://localhost:8080/health && break
  echo "Retry $i..."
  sleep 2
done
```

---

