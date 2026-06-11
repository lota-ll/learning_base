# Linux для DevOps Engineers

> **Чому саме зараз:** Linux — операційна система 70%+ хмарних workloads. Без впевненого знання Linux неможливо ефективно працювати з Docker, Kubernetes, CI/CD або AWS.
> **Поточний рівень:** знайомство з командним рядком, базові команди
> **Ціль модуля:** впевнено орієнтуватися в Linux-середовищі, розуміти архітектуру системи, вирішувати реальні проблеми в production
> **Час:** Теорія ~8 год · Практика ~16 год

---

## Зміст

1. [Basics — Ядро та архітектура](#1-basics--ядро-та-архітектура)
2. [Storage — Файлові системи та диски](#2-storage--файлові-системи-та-диски)
3. [Management — Адміністрування системи](#3-management--адміністрування-системи)
4. [Logging & Troubleshooting](#4-logging--troubleshooting)
5. [Security — Безпека Linux](#5-security--безпека-linux)
6. [Networking — Мережа в Linux](#6-networking--мережа-в-linux)
7. [Практичні проекти](#7-практичні-проекти)
8. [Типові помилки](#типові-помилки)
9. [Результат модуля](#результат-модуля)
10. [Interview Prep](#interview-prep)

---

## Середовище для практики

Перш ніж читати далі — підніми середовище:

```bash
# Варіант 1 — Vagrant + VirtualBox (рекомендовано для локальної роботи)
mkdir linux-lab && cd linux-lab
vagrant init ubuntu/jammy64
vagrant up
vagrant ssh

# Варіант 2 — AWS EC2 (якщо маєш акаунт)
# t2.micro, Ubuntu 22.04, Security Group: SSH 22

# Варіант 3 — Docker (для швидкого старту)
docker run -it ubuntu:22.04 bash
```

**Дистрибутиви які варто знати:**
| Дистрибутив | Де зустрінеш | Пакетний менеджер |
|-------------|-------------|-------------------|
| Ubuntu / Debian | AWS, GCP, Docker images | `apt` |
| RHEL / CentOS / Rocky | Enterprise, Ansible | `yum` / `dnf` |
| Alpine | Docker images (мінімальний розмір) | `apk` |
| CoreOS / Flatcar | Kubernetes nodes | (immutable) |

---

## 1. Basics — Ядро та архітектура

### 1.1 Kernel — що це і навіщо

Ядро — посередник між hardware і програмами. Воно керує:
- **процесами** (хто і скільки CPU отримує)
- **пам'яттю** (віртуальна адресація, swap)
- **пристроями** (драйвери)
- **файловою системою** (VFS шар)

```
┌─────────────────────────────────────┐
│         User Space                  │
│  bash  nginx  python  systemd       │
├─────────────────────────────────────┤
│         System Call Interface       │  ← межа між user/kernel space
├─────────────────────────────────────┤
│         Linux Kernel                │
│  ┌──────────┐ ┌──────────────────┐  │
│  │ Process  │ │ Memory Management│  │
│  │ Scheduler│ │ (mmap, malloc)   │  │
│  └──────────┘ └──────────────────┘  │
│  ┌──────────┐ ┌──────────────────┐  │
│  │ VFS      │ │ Network Stack    │  │
│  │ (файли)  │ │ (TCP/IP)        │  │
│  └──────────┘ └──────────────────┘  │
├─────────────────────────────────────┤
│         Hardware                    │
│  CPU     RAM     Disk     NIC       │
└─────────────────────────────────────┘
```

```bash
# Переглянь версію ядра
uname -r

# Детальна інформація про систему
uname -a

# Параметри ядра (runtime)
sysctl -a | head -30
sysctl vm.swappiness          # скільки агресивно використовується swap
sysctl net.core.somaxconn     # max черга TCP connections — важливо для nginx/load balancers
```

### 1.2 Linux Booting Process

```
BIOS/UEFI → Bootloader (GRUB2) → Kernel → initramfs → systemd (PID 1) → Services
```

**Крок за кроком:**
1. **BIOS/UEFI** — ініціалізує hardware, шукає завантажувальний пристрій
2. **GRUB2** — завантажує kernel image (`vmlinuz`) і `initrd`
3. **Kernel** — розпаковує `initramfs`, монтує root filesystem
4. **systemd** (PID 1) — запускає всі сервіси за залежностями

```bash
# Переглянь boot log
journalctl -b                  # весь boot log поточного завантаження
journalctl -b -1               # попереднє завантаження
journalctl -b --priority=err   # тільки помилки при старті

# GRUB конфігурація
cat /etc/default/grub
# Після змін — обов'язково:
sudo update-grub               # на Ubuntu/Debian
sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # на RHEL/CentOS

# Час завантаження
systemd-analyze
systemd-analyze blame          # які сервіси найдовше стартують
systemd-analyze critical-chain # критичний шлях завантаження
```

### 1.3 User Space vs Kernel Space

**Аналогія:** Kernel Space — це кухня ресторану (тільки для персоналу), User Space — зал для відвідувачів. Програми з User Space "замовляють" через system calls (офіціант).

```bash
# Подивись на system calls будь-якого процесу
strace ls -la 2>&1 | head -30

# Скільки system calls робить nginx при старті
strace -c nginx -t 2>&1

# /proc — вікно в Kernel Space
ls /proc/
cat /proc/meminfo      # стан пам'яті
cat /proc/cpuinfo      # інформація про CPU
cat /proc/loadavg      # load average
cat /proc/1/status     # статус systemd (PID 1)
```

### 1.4 Run Levels та Systemd Targets

**Run levels** — спадщина SysV init. Systemd замінив їх на **targets**:

| Run Level | Systemd Target | Опис |
|-----------|---------------|------|
| 0 | `poweroff.target` | Вимкнення |
| 1 | `rescue.target` | Single user mode |
| 3 | `multi-user.target` | Без GUI (сервери!) |
| 5 | `graphical.target` | З GUI |
| 6 | `reboot.target` | Перезавантаження |

```bash
# Поточний target
systemctl get-default

# Змінити default target (наприклад, відключити GUI на сервері)
sudo systemctl set-default multi-user.target

# Перейти в режим без перезавантаження
sudo systemctl isolate multi-user.target

# Rescue mode (якщо щось зламалось)
sudo systemctl rescue
```

### 1.5 Linux Processes

**Процес** — запущена програма в пам'яті з власним PID, адресним простором і ресурсами.

```bash
# Перегляд процесів
ps aux                 # всі процеси
ps aux | grep nginx    # знайти конкретний
top                    # динамічний перегляд (q — вийти)
htop                   # зручніший варіант (sudo apt install htop)

# Дерево процесів — покаже parent-child відносини
pstree -p
pstree -p | grep nginx

# Детально про процес
ls /proc/$(pgrep nginx | head -1)/  # файлова структура процесу
cat /proc/$(pgrep nginx | head -1)/status

# Пам'ять процесу
cat /proc/$(pgrep nginx | head -1)/maps | head -20
```

### 1.6 Process States

```
        fork()
NEW ──────────→ READY ←──────────────────┐
                  │                       │
              scheduled               preempted
                  ↓                       │
               RUNNING ─────────────────→─┘
                  │
         ┌────────┴────────┐
     I/O wait          exit()
         ↓                 ↓
      SLEEPING          ZOMBIE
         │                 │
     I/O done         parent reads
         ↓             exit code
       READY              END
```

```bash
# Стани процесів у ps:
# R — Running/Runnable
# S — Sleeping (interruptible, чекає I/O)
# D — Uninterruptible Sleep (чекає I/O, не можна kill!)
# Z — Zombie
# T — Stopped (SIGSTOP)

ps aux | awk '{print $8}' | sort | uniq -c
# Якщо багато D — проблема з диском або NFS
```

### 1.7 Inter-Process Communication (IPC)

| Механізм | Коли використовується | Приклад |
|----------|----------------------|---------|
| Pipes `\|` | Передача stdout→stdin | `ps aux \| grep nginx` |
| Named Pipes (FIFO) | Між непов'язаними процесами | `mkfifo /tmp/mypipe` |
| Signals | Управління процесами | `kill -SIGTERM 1234` |
| Shared Memory | Швидка передача даних | `ipcs -m` |
| Sockets | Мережева/локальна комунікація | nginx ↔ php-fpm через Unix socket |
| Message Queues | Асинхронна комунікація | `ipcs -q` |

```bash
# Переглянь IPC ресурси системи
ipcs

# Named pipe приклад
mkfifo /tmp/testpipe
echo "hello from process 1" > /tmp/testpipe &   # writer блокується
cat /tmp/testpipe                                 # reader розблокує writer

# Unix sockets — nginx і php-fpm спілкуються через них
ls -la /var/run/php/
```

### 1.8 Linux Signals

```bash
# Найважливіші сигнали
kill -l   # список всіх сигналів

# SIGTERM (15) — "будь ласка, завершись" (graceful shutdown)
kill -SIGTERM $(pgrep nginx)
# або
kill -15 $(pgrep nginx)

# SIGKILL (9) — "негайно завершись" (не перехоплюється!)
kill -9 $(pgrep -f frozen_process)

# SIGHUP (1) — "перечитай конфігурацію" (reload без restart)
kill -HUP $(pgrep nginx)
# Або через systemd:
sudo systemctl reload nginx

# SIGSTOP/SIGCONT — пауза/продовження
kill -STOP $(pgrep my_process)
kill -CONT $(pgrep my_process)

# Надіслати сигнал всім процесам за назвою
pkill -SIGTERM nginx
killall nginx
```

**Правило:** Завжди спочатку `SIGTERM`, дай 30 секунд, потім `SIGKILL`. Це дозволяє graceful shutdown.

### 1.9 Zombie Processes

**Zombie** — процес завершився, але батько ще не "прочитав" його exit code через `wait()`. Займає тільки рядок у таблиці процесів, але не пам'ять.

```bash
# Знайти zombies
ps aux | grep 'Z'
ps aux | awk '$8 == "Z"'

# Скільки zombie процесів
ps aux | grep -c ' Z '

# Zombie не можна kill! Треба kill батька або чекати
# Знайти батька zombie процесу
pgrep -P $(ps aux | awk '$8=="Z" {print $2}' | head -1) 2>/dev/null
# або
cat /proc/ZOMBIE_PID/status | grep PPid
```

**У DevOps контексті:** Zombie процеси в контейнерах — типова проблема, якщо PID 1 не реалізує `wait()`. Рішення — `tini` або `dumb-init` як PID 1 у Docker.

### 1.10 Process Control Groups (cgroups)

cgroups — механізм обмеження і обліку ресурсів для груп процесів. **Основа Docker і Kubernetes resource limits!**

```bash
# Переглянь cgroups v2 (сучасний стандарт)
ls /sys/fs/cgroup/
cat /sys/fs/cgroup/memory.stat

# Як Docker використовує cgroups
# Запусти контейнер з лімітами
docker run -d --name test-cgroup --memory="128m" --cpus="0.5" nginx

# Знайди cgroup контейнера
cat /sys/fs/cgroup/memory/docker/$(docker inspect test-cgroup --format='{{.Id}}')/memory.limit_in_bytes

# Подивись ліміти systemd сервісу (теж через cgroups)
systemctl show nginx | grep -i memory
systemctl show nginx | grep -i cpu
```

```bash
# Встанови ліміти для сервісу через systemd
sudo systemctl edit nginx
# Додай:
# [Service]
# MemoryLimit=512M
# CPUQuota=50%
```

### 1.11 Linux Namespaces

Namespaces — ізоляція ресурсів. Разом з cgroups — **основа контейнеризації**.

| Namespace | Що ізолює | Docker використовує |
|-----------|-----------|-------------------|
| `pid` | PID-и процесів | ✅ (PID 1 в контейнері) |
| `net` | Мережевий стек | ✅ (власний IP, interfaces) |
| `mnt` | Точки монтування | ✅ (власна файлова система) |
| `uts` | Hostname | ✅ |
| `ipc` | IPC ресурси | ✅ |
| `user` | UID/GID mapping | ✅ (rootless containers) |

```bash
# Переглянь namespaces процесу
ls -la /proc/self/ns/
ls -la /proc/1/ns/

# Namespaces запущеного контейнера
docker run -d --name test nginx
CONT_PID=$(docker inspect test --format='{{.State.Pid}}')
ls -la /proc/$CONT_PID/ns/

# Увійти в namespace контейнера (як docker exec, але вручну)
sudo nsenter -t $CONT_PID -n ip addr  # мережевий namespace контейнера
sudo nsenter -t $CONT_PID -m ls /     # файловий namespace
```

### 1.12 Sockets & Ports

```bash
# Переглянь всі відкриті порти
ss -tlnp           # TCP, listening, numeric, processes (сучасна заміна netstat)
ss -ulnp           # UDP
netstat -tlnp      # старий варіант (може не бути встановлений)

# Що слухає на конкретному порту
ss -tlnp | grep :80
lsof -i :80        # детальніше — який процес

# Перевір чи доступний порт
nc -zv localhost 80
nc -zv google.com 443

# Unix sockets
ss -xlnp           # Unix domain sockets
ls /var/run/        # типове місце для Unix sockets
```

### 1.13 Protocols (SSH, SCP, FTP)

```bash
# SSH — основний інструмент DevOps
# Підключення
ssh user@hostname
ssh -i ~/.ssh/mykey.pem ubuntu@10.0.0.1
ssh -p 2222 user@hostname          # нестандартний порт
ssh -L 8080:localhost:80 user@host  # Local port forwarding (тунель!)
ssh -J bastion user@internal-server # Jump host (ProxyJump)

# SSH Config (~/.ssh/config) — щоб не вводити параметри щоразу
cat >> ~/.ssh/config << 'EOF'
Host myserver
  HostName 10.0.0.1
  User ubuntu
  IdentityFile ~/.ssh/mykey.pem
  Port 22

Host internal
  HostName 192.168.1.10
  User ec2-user
  ProxyJump myserver
EOF
ssh myserver  # тепер просто

# SCP — копіювання файлів через SSH
scp file.txt user@host:/remote/path/
scp -r ./directory user@host:/remote/
scp user@host:/remote/file.txt ./local/

# rsync — кращий варіант (тільки різницю)
rsync -avz --progress ./local/ user@host:/remote/
rsync -avz --delete ./local/ user@host:/remote/  # --delete — синхронізація

# sftp — інтерактивний
sftp user@host
```

### 1.14 Shell

```bash
# Типи shell
cat /etc/shells     # доступні shells
echo $SHELL         # поточний shell
echo $0             # назва shell в поточній сесії

# bash vs sh vs zsh
# sh — мінімальний POSIX shell (є скрізь, але обмежений)
# bash — розширений, стандарт для скриптів DevOps
# zsh — інтерактивний, для розробників

# .bashrc vs .bash_profile vs .profile
# .bash_profile — виконується при LOGIN shell (ssh, логін у системі)
# .bashrc — при INTERACTIVE non-login shell (відкрити термінал)
# .profile — POSIX, для всіх shell

# Корисне в ~/.bashrc
cat >> ~/.bashrc << 'EOF'
# Зберігати більше history
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTTIMEFORMAT="%F %T "   # час кожної команди

# Не зберігати дублікати
export HISTCONTROL=ignoredups:erasedups

# Aliases
alias ll='ls -alF'
alias la='ls -A'
alias k='kubectl'
alias tf='terraform'

# PS1 з git branch
parse_git_branch() {
  git branch 2>/dev/null | grep -E '^\*' | awk '{print " ("$2")"}'
}
export PS1="\u@\h:\w\$(parse_git_branch)\$ "
EOF

source ~/.bashrc
```

### 1.15 Linux Command Line — Основні команди

```bash
# ── Навігація та файли ──────────────────────────────────────────────
pwd                    # де я знаходжусь
ls -la                 # список файлів з деталями
ls -lah                # + читаємий розмір (human-readable)
cd /etc/nginx          # перейти в директорію
cd -                   # повернутись у попередню директорію
mkdir -p a/b/c         # створити вкладені директорії
cp -r src/ dst/        # копіювати рекурсивно
mv old.txt new.txt     # перейменувати або перемістити
rm -rf /tmp/old/       # видалити рекурсивно (ОБЕРЕЖНО!)
ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/  # symlink

# ── Пошук ───────────────────────────────────────────────────────────
find /etc -name "*.conf" -type f      # знайти файли
find /var/log -mtime -1               # змінені за останню добу
find / -size +100M 2>/dev/null        # файли > 100MB
find /tmp -user www-data              # файли конкретного юзера
locate nginx.conf                     # швидкий пошук (база даних)
which python3                         # де знаходиться команда
type ls                               # alias, builtin чи binary?

# ── Перегляд файлів ─────────────────────────────────────────────────
cat /etc/hosts                        # весь файл
less /var/log/nginx/error.log         # посторінково (q — вийти)
head -20 /var/log/syslog              # перші 20 рядків
tail -50 /var/log/syslog              # останні 50 рядків
tail -f /var/log/nginx/access.log     # live follow (Ctrl+C — вийти)
tail -f /var/log/nginx/access.log | grep --line-buffered "ERROR"

# ── Текстова обробка ────────────────────────────────────────────────
grep "ERROR" /var/log/nginx/error.log
grep -r "listen 80" /etc/nginx/       # рекурсивно в директорії
grep -v "DEBUG" /var/log/app.log      # виключити рядки
grep -E "5[0-9]{2}" access.log        # regex — знайти 5xx помилки
grep -c "ERROR" error.log             # порахувати рядки

# Обробка колонок
awk '{print $1, $7}' access.log       # 1-а і 7-а колонки (IP і URL)
awk -F: '{print $1}' /etc/passwd      # поле 1, роздільник ":"
cut -d: -f1 /etc/passwd               # те саме через cut

# Трансформація
sed 's/old/new/g' config.txt          # заміна в stdout
sed -i 's/old/new/g' config.txt       # заміна в файлі (in-place!)
sed -n '10,20p' file.txt              # вивести рядки 10-20

# Сортування і унікальність
sort file.txt
sort -rn numbers.txt                  # reverse, numeric
sort | uniq                           # видалити дублікати
sort | uniq -c | sort -rn             # порахувати + відсортувати (top-N pattern)

# Приклад: top 10 IP у nginx access.log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# ── Архіви ──────────────────────────────────────────────────────────
tar -czf backup.tar.gz /etc/nginx/    # створити gz архів
tar -xzf backup.tar.gz                # розпакувати
tar -czf - /etc/nginx/ | ssh user@host "cat > /backup/nginx.tar.gz"  # архів через SSH

# ── Редактори ───────────────────────────────────────────────────────
# nano — простий
nano /etc/hosts

# vim — потужний, присутній скрізь
# Режими: Normal (навігація), Insert (i — писати), Visual (v — виділити)
# :w — зберегти, :q — вийти, :wq — зберегти і вийти, :q! — вийти без збереження
# /pattern — пошук, n — наступне, N — попереднє
# dd — видалити рядок, yy — копіювати рядок, p — вставити
# gg — початок файлу, G — кінець, :42 — перейти на рядок 42
# :%s/old/new/g — заміна у всьому файлі
vim /etc/nginx/nginx.conf
```

---

## 2. Storage — Файлові системи та диски

### 2.1 File System Hierarchy (FHS)

```
/
├── bin/     → /usr/bin     # бінарні файли (ls, cp, bash)
├── sbin/    → /usr/sbin    # системні бінарні (fdisk, ip)
├── etc/                    # конфігурації (ТІЛЬКИ конфіги, не дані!)
├── var/                    # змінні дані (логи, кеш, бази даних)
│   ├── log/
│   ├── lib/
│   └── www/
├── home/                   # домашні директорії користувачів
├── root/                   # домашня директорія root
├── tmp/                    # тимчасові файли (очищаються при reboot)
├── srv/                    # дані сервісів (веб-контент)
├── opt/                    # стороннє ПЗ
├── proc/                   # віртуальна FS — інформація про процеси
├── sys/                    # віртуальна FS — інформація про hardware
├── dev/                    # пристрої (диски, tty, null, random)
│   ├── sda, sdb            # SCSI/SATA диски
│   ├── nvme0n1             # NVMe диски
│   ├── null                # /dev/null — "смітник"
│   └── random, urandom     # генератори випадкових чисел
├── mnt/                    # тимчасові точки монтування
├── media/                  # знімні носії
├── boot/                   # файли завантаження (kernel, grub)
├── lib/     → /usr/lib     # бібліотеки
└── usr/                    # user system resources
    ├── bin/
    ├── lib/
    └── share/
```

```bash
# Розмір директорій
du -sh /var/log/
du -sh /* 2>/dev/null | sort -rh | head -10  # топ директорій за розміром
df -h                                          # вільне місце на розділах
df -i                                          # inodes (може закінчитись окремо від місця!)
```

### 2.2 File Systems

| FS | Де | Особливості |
|----|-----|------------|
| **ext4** | Ubuntu, Debian (за замовчуванням) | Стабільний, journaling, до 1 EB |
| **XFS** | RHEL, CentOS (за замовчуванням) | Паралельний I/O, великі файли, не можна зменшити |
| **Btrfs** | openSUSE, можна на Ubuntu | Snapshots, compression, checksums, RAID |
| **tmpfs** | `/tmp`, `/run` | В RAM, швидкий, не зберігає при reboot |
| **overlayfs** | Docker layers | Union mount (read-only layers + writable layer) |

```bash
# Визначити тип FS
df -T
lsblk -f
findmnt

# Створити і відформатувати
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.xfs /dev/sdc1

# Параметри монтування у /etc/fstab
cat /etc/fstab
# UUID=xxx /data ext4 defaults,noatime 0 2
# noatime — не оновлювати access time (прискорює I/O)
# nofail — не падати при відсутності пристрою (важливо для AWS EBS!)

# Перевірка і ремонт FS (тільки на unmounted!)
sudo fsck -f /dev/sdb1
sudo e2fsck -f /dev/sdb1   # для ext4
```

### 2.3 inodes

**inode** — структура даних, що містить метадані файлу (права, власник, розмір, timestamps, покажчики на блоки). Не містить ім'я файлу!

```
Directory entry:
  "nginx.conf" → inode #1234

inode #1234:
  owner: root
  permissions: 644
  size: 2048
  ctime: 2024-01-15
  blocks: [block_42, block_43, ...]

Blocks:
  block_42: [actual file data...]
```

```bash
# Переглянь inode файлу
ls -i /etc/nginx/nginx.conf
stat /etc/nginx/nginx.conf

# Кількість вільних inodes (може закінчитись!)
df -i

# Symlink — окремий inode, що вказує на шлях
# Hardlink — другий запит у директорії на той самий inode
ls -li /usr/bin/python*   # hardlinks мають однаковий inode number

# Знайти файли без імені (orphan inodes — зазвичай помилка)
find / -inum INODE_NUMBER 2>/dev/null
```

### 2.4 Volumes та Disk Management

```bash
# Перелік дисків і розділів
lsblk                  # дерево блочних пристроїв
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
fdisk -l               # детальна інформація (потребує root)

# Partitioning
sudo fdisk /dev/sdb    # інтерактивний partitioner (MBR)
sudo parted /dev/sdb   # для GPT дисків
sudo gdisk /dev/sdb    # GPT partition editor

# Монтування
sudo mount /dev/sdb1 /mnt/data
sudo mount -t ext4 -o noatime /dev/sdb1 /mnt/data
sudo umount /mnt/data

# Постійне монтування через /etc/fstab
# Отримай UUID
blkid /dev/sdb1
# Додай в /etc/fstab:
echo "UUID=your-uuid /mnt/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount -a    # перевір що монтується без помилок
```

### 2.5 Logical Volume Management (LVM)

**Аналогія:** LVM — це як об'єднати кілька фізичних USB-флешок в один великий диск, який можна динамічно збільшувати.

```
Physical Volumes (PV) → Volume Group (VG) → Logical Volumes (LV) → File System
    /dev/sdb1                 data_vg              lv_app                ext4
    /dev/sdc1                                       lv_logs               xfs
```

```bash
# Налаштування LVM з нуля
sudo apt install lvm2

# 1. Ініціалізуй фізичні томи
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs    # перегляд PV
sudo pvdisplay /dev/sdb

# 2. Створи Volume Group
sudo vgcreate data_vg /dev/sdb /dev/sdc
sudo vgs    # перегляд VG
sudo vgdisplay data_vg

# 3. Створи Logical Volume
sudo lvcreate -L 50G -n lv_app data_vg
sudo lvcreate -l 100%FREE -n lv_logs data_vg   # все що залишилось
sudo lvs    # перегляд LV

# 4. Відформатуй і змонтуй
sudo mkfs.ext4 /dev/data_vg/lv_app
sudo mount /dev/data_vg/lv_app /var/app

# Розширення (онлайн, без зупинки!)
sudo lvextend -L +20G /dev/data_vg/lv_app
sudo resize2fs /dev/data_vg/lv_app   # для ext4
# sudo xfs_growfs /var/app           # для XFS
```

**Чому LVM важливо знати:** AWS EC2 за замовчуванням використовує LVM на деяких AMI. Kubernetes Persistent Volumes часто використовують LVM на bare metal.

### 2.6 NFS (Network File System)

```bash
# Server side — встановлення і конфігурація
sudo apt install nfs-kernel-server

# /etc/exports — що і кому відкриваємо
cat /etc/exports
# /data/shared 10.0.0.0/24(rw,sync,no_subtree_check)
# /data/readonly *(ro,sync)

sudo exportfs -a    # застосувати конфігурацію
sudo systemctl restart nfs-kernel-server
sudo showmount -e localhost    # перевір що доступно

# Client side
sudo apt install nfs-common
sudo mount -t nfs 10.0.0.5:/data/shared /mnt/nfs
# Або через /etc/fstab:
# 10.0.0.5:/data/shared /mnt/nfs nfs defaults,_netdev,nofail 0 0

# Перевірка
df -h | grep nfs
ls /mnt/nfs
```

### 2.7 Backup and Restore

```bash
# rsync — стандартний DevOps інструмент
# -a: archive mode (права, symlinks, timestamps)
# -v: verbose, -z: compress, --progress: прогрес, --delete: видалити зайве

rsync -avz /var/app/data/ /backup/app-data/
rsync -avz --delete /etc/ backup-server:/backups/etc/

# Incremental backup через rsync + hardlinks
rsync -avz --link-dest=/backup/daily.1 /var/app/ /backup/daily.0
# Зберігає тільки змінені файли, незмінені — hardlinks (дешево)

# tar з ротацією
DATE=$(date +%Y%m%d)
tar -czf /backup/nginx-$DATE.tar.gz /etc/nginx/
# Видалити старіші 7 днів
find /backup/ -name "nginx-*.tar.gz" -mtime +7 -delete

# dd — disk image (для повного відновлення)
sudo dd if=/dev/sda of=/backup/disk.img bs=4M status=progress
sudo dd if=/backup/disk.img of=/dev/sda bs=4M status=progress  # відновлення

# Перевірка цілісності бекапу
tar -tzf /backup/nginx-20240115.tar.gz | head   # перевір вміст без розпаковки
md5sum /backup/nginx-20240115.tar.gz > /backup/nginx-20240115.md5
md5sum -c /backup/nginx-20240115.md5             # верифікація
```

---

## 3. Management — Адміністрування системи

### 3.1 User & Group Management

```bash
# Користувачі
cat /etc/passwd           # username:x:uid:gid:comment:home:shell
cat /etc/shadow           # паролі (hashed), тільки root
cat /etc/group            # groups

# Створити користувача
sudo useradd -m -s /bin/bash -G sudo,docker deploy_user
# -m: створити home, -s: shell, -G: додаткові групи

sudo useradd -r -s /sbin/nologin nginx  # system user (без shell)
sudo userdel -r old_user                # видалити + home директорію
sudo usermod -aG docker $USER           # додати в групу (a = append!)

# Пароль
sudo passwd deploy_user
sudo chage -l deploy_user               # параметри пароля (expiry etc)

# Групи
sudo groupadd devops
sudo groupmod -n new_name old_name
sudo gpasswd -d user group              # видалити з групи

# Перевірити поточного користувача
id                   # uid, gid, groups
whoami
groups               # список груп
w                    # хто зараз залогінений
last                 # history логінів
lastlog              # останній логін кожного юзера

# sudo конфігурація
sudo visudo          # редагувати /etc/sudoers ЗАВЖДИ через visudo!
# Приклад: deploy_user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

### 3.2 SSH Management

```bash
# Генерація ключів
ssh-keygen -t ed25519 -C "deploy@company.com" -f ~/.ssh/deploy_key
# ed25519 — сучасний, безпечніший за RSA
ssh-keygen -t rsa -b 4096 -C "legacy@company.com"   # якщо потрібен RSA

# Копіювання публічного ключа
ssh-copy-id -i ~/.ssh/deploy_key.pub user@server
# або вручну
cat ~/.ssh/deploy_key.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# SSH Agent — щоб не вводити passphrase кожного разу
eval $(ssh-agent -s)
ssh-add ~/.ssh/deploy_key
ssh-add -l    # список завантажених ключів

# /etc/ssh/sshd_config — конфіг SSH сервера
sudo vim /etc/ssh/sshd_config
# Важливі параметри безпеки:
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
# Port 2222              # нестандартний порт (базова обфускація)
# AllowUsers deploy_user ubuntu
# MaxAuthTries 3

sudo systemctl reload sshd
# ПЕРЕВІР з нового terminal до перезавантаження!

# SSH tunneling
ssh -L 5432:localhost:5432 user@db-server  # local port forwarding
# Тепер localhost:5432 → db-server:5432 (доступ до БД через SSH)

ssh -R 8080:localhost:3000 user@public-server  # remote port forwarding
# public-server:8080 → твій localhost:3000
```

### 3.3 Package Management

```bash
# ── apt (Debian/Ubuntu) ─────────────────────────────────────────────
sudo apt update                    # оновити список пакетів
sudo apt upgrade                   # оновити встановлені пакети
sudo apt install nginx             # встановити
sudo apt install -y nginx          # без підтвердження (-y)
sudo apt remove nginx              # видалити (залишити конфіги)
sudo apt purge nginx               # видалити + конфіги
sudo apt autoremove                # видалити непотрібні залежності
sudo apt list --installed          # список встановлених
sudo apt show nginx                # інформація про пакет
sudo apt search "web server"       # пошук
dpkg -l | grep nginx               # перевірити встановлені (dpkg)
dpkg -L nginx                      # файли пакета

# Додати репозиторій
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update && sudo apt install jenkins

# ── yum/dnf (RHEL/CentOS/Rocky) ────────────────────────────────────
sudo dnf install nginx             # встановити
sudo dnf update                    # оновити всі пакети
sudo dnf remove nginx              # видалити
sudo dnf list installed            # список встановлених
sudo dnf search nginx
sudo dnf info nginx
sudo rpm -qa | grep nginx          # rpm query
sudo rpm -ql nginx                 # файли пакета
# Додати репо
sudo dnf install epel-release      # EPEL репозиторій
```

### 3.4 Systemd Management

```bash
# Основні команди
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx       # зупинити + запустити
sudo systemctl reload nginx        # перечитати конфіг (без downtime!)
sudo systemctl status nginx        # статус + останні логи
sudo systemctl enable nginx        # autostart при boot
sudo systemctl disable nginx       # відключити autostart
sudo systemctl is-active nginx     # active/inactive (для скриптів)
sudo systemctl is-enabled nginx    # enabled/disabled

# Переглянути всі сервіси
systemctl list-units --type=service
systemctl list-units --type=service --state=failed
systemctl list-units --state=running

# Юніт файли
cat /lib/systemd/system/nginx.service   # системний (не редагуй!)
sudo systemctl edit nginx               # override (зберігається в /etc/systemd/system/)
sudo systemctl cat nginx                # показати повну конфігурацію

# Створення власного сервісу
sudo cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Python App
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
Environment=PORT=5000
Environment=ENV=production

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now myapp

# Dependencies між сервісами
systemctl list-dependencies nginx
```

### 3.5 Cronjob

```bash
# Формат: мін год день_міс місяць день_тижня команда
# * — будь-яке, */5 — кожні 5 одиниць, 1-5 — діапазон, 1,3,5 — список

crontab -e     # редагувати cron поточного юзера
crontab -l     # переглянути
crontab -r     # видалити ВСІ cron jobs (ОБЕРЕЖНО!)
sudo crontab -u nginx -e   # cron конкретного юзера

# Приклади
# Кожні 5 хвилин
*/5 * * * * /opt/scripts/health_check.sh

# О 2:30 щовівторка
30 2 * * 2 /opt/scripts/backup.sh

# Перший день кожного місяця о 00:00
0 0 1 * * /opt/scripts/monthly_report.sh

# Щодня о 3:00
0 3 * * * /opt/scripts/cleanup.sh >> /var/log/cleanup.log 2>&1

# Системні cron директорії
ls /etc/cron.d/      # файли-crond конфіги
ls /etc/cron.daily/  # скрипти що виконуються щодня
ls /etc/cron.weekly/
ls /etc/cron.monthly/

# Перевірка виконання
grep CRON /var/log/syslog | tail -20
journalctl -u cron | tail -20

# Альтернатива для складних schedules — systemd timers
sudo systemctl list-timers
```

### 3.6 Shell Scripting

```bash
#!/bin/bash
# Завжди починай з shebang
set -e          # зупинитись при помилці
set -u          # помилка при невизначеній змінній
set -o pipefail # помилка в pipeline = помилка скрипту
set -x          # debug mode — виводить кожну команду (розкоментуй для дебагу)

# ── Змінні ─────────────────────────────────────────────────────────
NAME="world"
echo "Hello, $NAME!"
echo "Hello, ${NAME}!"  # краща практика — завжди {}

# Змінні середовища
export APP_ENV="production"   # доступна дочірнім процесам
readonly MAX_RETRIES=3        # константа

# ── Умови ──────────────────────────────────────────────────────────
if [ -f "/etc/nginx/nginx.conf" ]; then
  echo "Nginx config exists"
elif [ -d "/etc/apache2" ]; then
  echo "Apache found"
else
  echo "No web server config found"
fi

# Порівняння рядків
if [[ "$ENV" == "production" ]]; then ...
if [[ "$NAME" =~ ^[A-Z] ]]; then ...   # regex в [[]]

# Числа
if [ "$COUNT" -gt 5 ]; then ...     # -gt -lt -ge -le -eq -ne
if (( COUNT > 5 )); then ...         # arithmetic context

# Файли
[ -f file ]   # file exists and is regular file
[ -d dir ]    # directory exists
[ -r file ]   # readable
[ -w file ]   # writable
[ -x file ]   # executable
[ -s file ]   # file exists and not empty
[ -L file ]   # is symlink
[ ! -f file ] # not exists

# ── Цикли ──────────────────────────────────────────────────────────
for i in 1 2 3; do
  echo "Item $i"
done

for file in /etc/nginx/*.conf; do
  echo "Processing: $file"
done

for i in $(seq 1 10); do
  echo "Iteration $i"
done

while read line; do
  echo "Line: $line"
done < /etc/hosts

# ── Функції ────────────────────────────────────────────────────────
log() {
  local level=$1
  local message=$2
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a /var/log/myapp.log
}

check_service() {
  local service=$1
  if systemctl is-active --quiet "$service"; then
    log "INFO" "$service is running"
    return 0
  else
    log "ERROR" "$service is NOT running"
    return 1
  fi
}

check_service nginx || exit 1

# ── Аргументи ──────────────────────────────────────────────────────
echo "Script: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Number of args: $#"
echo "Last exit code: $?"

# ── Error handling ─────────────────────────────────────────────────
cleanup() {
  echo "Cleaning up..."
  rm -f /tmp/script.lock
}
trap cleanup EXIT    # виконати при виході (навіть при помилці)
trap 'echo "Error on line $LINENO"' ERR

# ── Реальний приклад — health check скрипт ─────────────────────────
#!/bin/bash
set -euo pipefail

SERVICES=("nginx" "postgresql" "redis")
FAILURES=0

for service in "${SERVICES[@]}"; do
  if ! systemctl is-active --quiet "$service"; then
    echo "CRITICAL: $service is down! Attempting restart..."
    systemctl restart "$service" && echo "OK: $service restarted" || {
      echo "FAILED: Cannot restart $service"
      FAILURES=$((FAILURES + 1))
    }
  else
    echo "OK: $service is running"
  fi
done

exit $FAILURES
```

### 3.7 Process Management

```bash
# Пріоритет процесу — nice value (-20 = highest, 19 = lowest)
nice -n 10 tar -czf backup.tar.gz /data/   # запустити з низьким пріоритетом
sudo nice -n -5 nginx                        # вищий пріоритет (потребує sudo)
sudo renice -5 $(pgrep nginx)               # змінити для запущеного процесу

# Background і foreground
long_running_command &     # запустити у фоні
jobs                       # список фонових завдань
fg %1                      # повернути завдання 1 на передній план
bg %1                      # продовжити у фоні
Ctrl+Z                     # призупинити поточний процес (SIGSTOP)

# nohup — продовжити після виходу з сесії
nohup python3 app.py > /var/log/app.log 2>&1 &

# screen / tmux — мультиплексори (кращий варіант ніж nohup)
tmux new -s mysession        # нова сесія
tmux attach -t mysession     # приєднатись
Ctrl+B, D                    # відключитись (сесія продовжується)
tmux ls                      # список сесій

# Моніторинг CPU/RAM процесу в часі
pidstat -u -p $(pgrep nginx) 1 10   # CPU кожну секунду, 10 разів
pidstat -r -p $(pgrep nginx) 1 10   # RAM

# Ліміти ресурсів
ulimit -n                    # max open files
ulimit -n 65535              # встановити ліміт (в сесії)
cat /proc/sys/fs/file-max    # системний ліміт

# Постійні ліміти
sudo vim /etc/security/limits.conf
# nginx soft nofile 65535
# nginx hard nofile 65535
```

---

## 4. Logging & Troubleshooting

### 4.1 Syslog

```bash
# /var/log — основна директорія логів
ls /var/log/
# syslog / messages  — загальний системний лог
# auth.log           — авторизація, sudo, SSH
# kern.log           — ядро
# nginx/             — nginx access/error logs
# apt/               — пакетний менеджер

# rsyslog — традиційний syslog демон
cat /etc/rsyslog.conf
cat /etc/rsyslog.d/*.conf

# Конфігурація rsyslog (facility.severity → destination)
# *.info /var/log/messages        — все INFO і вище → messages
# mail.none /var/log/messages     — але не mail
# *.emerg :omusrmsg:*             — EMERGENCY → всіх користувачів

# Тест syslog
logger "Test message from my script"
logger -p local0.warning "Warning: disk usage high"
tail -1 /var/log/syslog   # перевір що отримали

# Severity levels (від 0 до 7):
# 0 emerg, 1 alert, 2 crit, 3 err, 4 warning, 5 notice, 6 info, 7 debug
```

### 4.2 journalctl

```bash
# journald — loggging daemon systemd (зберігає в бінарному форматі)
journalctl                          # всі логи (q — вийти)
journalctl -n 50                    # останні 50 рядків
journalctl -f                       # live follow
journalctl -b                       # поточне завантаження
journalctl -b -1                    # попереднє завантаження

# Фільтрація за сервісом
journalctl -u nginx
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-15 10:00" --until "2024-01-15 11:00"

# Фільтрація за пріоритетом
journalctl -p err                   # error і вище
journalctl -p warning               # warning і вище

# JSON виведення (для парсингу)
journalctl -u nginx -o json | python3 -m json.tool | head -50
journalctl -u nginx -o json-pretty | jq '.MESSAGE' | head -10

# Розмір і очищення
journalctl --disk-usage
sudo journalctl --vacuum-size=500M  # залишити не більше 500MB
sudo journalctl --vacuum-time=7d    # залишити не старше 7 днів

# Постійне збереження (за замовчуванням — тільки в пам'яті)
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

### 4.3 Log Rotation

```bash
cat /etc/logrotate.conf
ls /etc/logrotate.d/       # конфіги per-service

# Приклад конфігурації /etc/logrotate.d/myapp
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
  daily               # ротація щодня
  missingok           # не помилятись якщо файл відсутній
  rotate 14           # зберігати 14 ротацій
  compress            # gzip старі логи
  delaycompress       # gzip не одразу (дати nginx закрити файл)
  notifempty          # не ротувати порожні файли
  create 0640 myapp myapp  # права нового файлу
  postrotate
    systemctl reload myapp 2>/dev/null || true
  endscript
}
EOF

# Тестовий запуск (без реальної ротації)
sudo logrotate -d /etc/logrotate.d/myapp

# Форсована ротація
sudo logrotate -f /etc/logrotate.d/myapp
```

### 4.4 CPU / Memory / Disk I/O Моніторинг

```bash
# ── CPU ────────────────────────────────────────────────────────────
top                         # динамічний (q — вийти)
top -b -n 1                 # один snapshot (для скриптів)
htop                        # кращий top (F1 — help)
mpstat 1 5                  # CPU по кожному ядру, 1 сек, 5 разів
sar -u 1 5                  # System Activity Reporter — CPU
uptime                      # load average (1m, 5m, 15m)

# Load Average інтерпретація:
# На 4-ядерному CPU: LA=4.0 — 100% зайнято, LA=8.0 — черга
# Правило: LA > кількість CPU = проблема

# ── Memory ─────────────────────────────────────────────────────────
free -h                     # RAM і swap
cat /proc/meminfo           # детально
vmstat 1 5                  # RAM, swap, I/O статистика

# OOM Killer — ядро вбиває процеси при нестачі пам'яті
dmesg | grep -i "oom"
journalctl -k | grep -i "oom killer"

# ── Disk I/O ────────────────────────────────────────────────────────
iostat -xz 1 5              # I/O статистика по дисках
iotop                       # хто займає диск (як top для I/O)
iotop -o                    # тільки активні процеси

# Ключові метрики iostat:
# %util — % часу диск зайнятий (>90% = перевантаження)
# await — середній час очікування (ms, < 10ms — добре)
# r/s, w/s — операції читання/запису в секунду

# ── Мережа ─────────────────────────────────────────────────────────
sar -n DEV 1 5              # трафік по інтерфейсах
nethogs                     # трафік по процесах
nload                       # live bandwidth
```

### 4.5 Команди для Troubleshooting

```bash
# ── "Що відбувається прямо зараз?" ─────────────────────────────────
w                           # хто залогінений, load
ps auxf                     # дерево процесів
ss -tlnp                    # відкриті порти
netstat -s | grep -i error  # мережеві помилки
dmesg | tail -20            # останні повідомлення ядра

# ── "Де втрачається місце?" ─────────────────────────────────────────
df -h                       # вільне місце
du -sh /* 2>/dev/null | sort -rh | head  # топ директорій
du -sh /var/log/* | sort -rh | head 10  # найбільші логи
ncdu /                      # interactive disk usage (sudo apt install ncdu)

# ── "Що не так з мережею?" ─────────────────────────────────────────
ping -c 4 8.8.8.8           # базова доступність
traceroute 8.8.8.8          # маршрут пакетів
mtr 8.8.8.8                 # traceroute + ping в реальному часі
dig google.com              # DNS lookup
nslookup google.com
curl -v https://api.example.com/health  # HTTP запит з деталями
curl -o /dev/null -s -w "%{http_code}\t%{time_total}\n" https://site.com

# ── "Чому процес зависає?" ─────────────────────────────────────────
strace -p PID               # що робить процес (system calls)
lsof -p PID                 # відкриті файли і сокети процесу
lsof -i :80                 # хто використовує порт 80
cat /proc/PID/wchan          # в якому kernel call зависнув

# ── "Що відбувалось вчора вночі?" ──────────────────────────────────
journalctl --since "2024-01-15 02:00" --until "2024-01-15 04:00" -p err
grep "Jan 15 02" /var/log/syslog | grep -i error
last -F | head -20           # login history з часом

# ── Швидкий performance огляд (1-2 хвилини) ────────────────────────
uptime                       # load average
dmesg | tail -5              # нові помилки ядра
journalctl -b -p err -n 20  # помилки сервісів
df -h                        # диск
free -h                      # пам'ять
ss -s                        # TCP статистика
ps auxf | head -20           # топ процеси
```

---

## 5. Security — Безпека Linux

### 5.1 File Permissions

```bash
# ls -la виводить:
# -rw-r--r-- 1 root root 1234 Jan 15 12:00 file.txt
# │└──────┘└┘ │    │    │    │
# │  perms   │ │    │    │    └── filename
# │          │ │    │    └─────── size
# │          │ │    └──────────── group
# │          │ └───────────────── owner
# │          └─────────────────── hard links count
# └────────────────────────────── type (- file, d dir, l symlink)

# Права: rwxrwxrwx = owner|group|others
# r=4, w=2, x=1

chmod 755 script.sh         # rwxr-xr-x
chmod 644 file.txt          # rw-r--r--
chmod 600 ~/.ssh/id_rsa     # rw------- (private key!)
chmod 700 ~/.ssh/           # rwx------ (ssh dir)
chmod +x script.sh          # додати execute всім
chmod go-w file.txt         # забрати write у group та others
chmod -R 755 /var/www/      # рекурсивно

# Власник
chown user file.txt
chown user:group file.txt
chown -R www-data:www-data /var/www/

# SUID, SGID, Sticky bit
chmod u+s /usr/bin/passwd   # SUID: виконується як власник файлу (4xxx)
chmod g+s /var/shared/      # SGID: нові файли наслідують group (2xxx)
chmod +t /tmp/              # Sticky: тільки власник може видалити свій файл (1xxx)

# Знайти SUID файли (потенційна дірка)
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# ACL — розширені права (більше ніж 3 ролі)
sudo apt install acl
setfacl -m u:deploy:rw /etc/nginx/nginx.conf    # додати права для deploy
setfacl -m g:devops:rx /var/log/nginx/          # права для групи
getfacl /etc/nginx/nginx.conf                    # переглянути ACL
```

### 5.2 Firewalls

```bash
# ── UFW (Ubuntu — простий фронтенд для iptables) ────────────────────
sudo ufw status
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow ssh             # або: allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL тільки з internal
sudo ufw deny 23/tcp

sudo ufw delete allow 80/tcp   # видалити правило
sudo ufw status numbered       # з номерами
sudo ufw delete 3              # видалити правило #3

sudo ufw reload

# ── firewalld (RHEL/CentOS) ─────────────────────────────────────────
sudo systemctl start firewalld
sudo firewall-cmd --state
sudo firewall-cmd --list-all

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --reload
```

### 5.3 SELinux

```bash
# SELinux — Mandatory Access Control (MAC) в RHEL/CentOS
# Кожен файл і процес має контекст (label), і правила дозволяють взаємодію

# Статус
getenforce                  # Enforcing / Permissive / Disabled
sestatus

# Режими
sudo setenforce 0           # Permissive (логує але не блокує) — ТИМЧАСОВО
sudo setenforce 1           # Enforcing

# Постійний режим
sudo vim /etc/selinux/config
# SELINUX=enforcing   або   SELINUX=permissive

# Контекст файлів
ls -Z /var/www/html/
ps -eZ | grep nginx

# Проблема: nginx не може читати /data/web/ → неправильний контекст
# Подивись аудит логи
sudo ausearch -m avc -ts recent
sudo sealert -a /var/log/audit/audit.log   # зручніше

# Виправлення — встановити правильний контекст
sudo semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
sudo restorecon -Rv /data/web/

# Дозволити httpd підключатись до мережі (наприклад, до backend)
sudo setsebool -P httpd_can_network_connect on
sudo getsebool -a | grep httpd
```

### 5.4 AppArmor

```bash
# AppArmor — MAC для Ubuntu (простіший за SELinux)
# Профілі: enforce (блокує) або complain (логує)

# Статус
sudo apparmor_status
sudo aa-status

# Профілі знаходяться в /etc/apparmor.d/
ls /etc/apparmor.d/
cat /etc/apparmor.d/usr.sbin.nginx  # профіль nginx

# Режими профілів
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx    # включити блокування
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx   # тільки логувати
sudo aa-disable /etc/apparmor.d/usr.sbin.nginx    # відключити

# Перезавантажити профіль після змін
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

# Логи порушень
sudo dmesg | grep apparmor
sudo journalctl | grep apparmor
```

### 5.5 seccomp

```bash
# seccomp — фільтрація system calls (ядро блокує небезпечні syscalls)
# Docker та Kubernetes використовують seccomp за замовчуванням

# Переглянь seccomp профіль в Docker
docker inspect nginx | jq '.[0].HostConfig.SecurityOpt'

# Дефолтний seccomp профіль Docker блокує ~44 syscalls
# Наприклад: clone, mount, reboot — небезпечні для контейнера

# Власний profil для Docker
docker run --security-opt seccomp=/path/to/seccomp.json nginx

# Для процесу — перевір чи є seccomp
cat /proc/$(pgrep nginx | head -1)/status | grep -i seccomp
# Seccomp: 2  — означає MODE_FILTER (активний)
```

### 5.6 auditd

```bash
# auditd — система аудиту для compliance (хто що робив)
sudo apt install auditd

# Перегляд правил
sudo auditctl -l

# Моніторинг файлу
sudo auditctl -w /etc/passwd -p rwa -k passwd_changes
# -w: watch, -p: permissions (r=read,w=write,a=attr,x=execute), -k: key

# Моніторинг команди sudo
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -k root_cmds

# Перегляд логів
sudo ausearch -k passwd_changes
sudo ausearch -k passwd_changes -ts today
sudo ausearch -ui 1000 -ts recent   # дії конкретного uid за останній час

# Звіти
sudo aureport --summary
sudo aureport --login --failed         # невдалі логіни
sudo aureport --user --success         # успішні дії юзерів

# Постійні правила
sudo vim /etc/audit/rules.d/audit.rules
sudo systemctl restart auditd
```

### 5.7 System Hardening

```bash
# 1. Оновлення пакетів — автоматичне
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# 2. Disable непотрібні сервіси
systemctl list-units --type=service --state=running
sudo systemctl disable --now cups      # print service
sudo systemctl disable --now avahi-daemon  # mDNS (якщо не потрібен)

# 3. Kernel параметри безпеки (/etc/sysctl.d/)
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# Відключити IP forwarding (якщо не router/gateway)
net.ipv4.ip_forward = 0

# SYN flood захист
net.ipv4.tcp_syncookies = 1

# Захист від IP spoofing
net.ipv4.conf.all.rp_filter = 1

# Відмовити у відповідях на ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Захист /proc від reading іншими юзерами
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2

# Randomize memory layout (ASLR)
kernel.randomize_va_space = 2
EOF
sudo sysctl --system

# 4. SSH hardening (в /etc/ssh/sshd_config)
cat > /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowAgentForwarding no
EOF
sudo systemctl reload sshd

# 5. Перевірити відкриті порти
ss -tlnp | grep -v "127.0.0.1\|::1"  # порти що слухають назовні

# 6. Lynis — security audit tool
sudo apt install lynis
sudo lynis audit system    # детальний звіт з рекомендаціями
```

---

## 6. Networking — Мережа в Linux

### 6.1 eBPF

**eBPF** (extended Berkeley Packet Filter) — технологія, що дозволяє запускати sandboxed програми прямо в ядрі Linux без зміни ядра або завантаження модулів.

**Де застосовується в DevOps:**
- **Cilium** (Kubernetes networking) — замінює iptables
- **Falco** — runtime security monitoring
- **Pixie** — observability без інструментації коду
- **bcc tools** — performance analysis

```bash
# Встановлення BCC tools
sudo apt install bpfcc-tools linux-headers-$(uname -r)

# Корисні eBPF інструменти
sudo execsnoop-bpfcc         # всі запущені команди в системі (live!)
sudo opensnoop-bpfcc         # всі відкриті файли
sudo tcpconnect-bpfcc        # всі TCP з'єднання
sudo biolatency-bpfcc        # disk I/O latency histogram
sudo funclatency-bpfcc vfs_read   # latency ядерної функції

# bpftrace — скриптова мова для eBPF
sudo apt install bpftrace
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write { printf("%s called write\n", comm); }'
```

### 6.2 IPtables

```bash
# iptables — низькорівневий firewall (ufw/firewalld — frontend до нього)
# Chains: INPUT → пакети до системи
#         OUTPUT → пакети від системи
#         FORWARD → пакети через систему (router)
# Таблиці: filter (default), nat, mangle, raw

# Перегляд правил
sudo iptables -L -v -n        # filter table
sudo iptables -L -v -n -t nat # nat table
sudo iptables-save            # всі правила (для backup)

# Базові правила
sudo iptables -P INPUT DROP                    # default policy — скинути
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # дозволити відповіді
sudo iptables -A INPUT -i lo -j ACCEPT         # loopback
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # SSH
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT   # HTTP
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT  # HTTPS

# NAT — Port Forwarding (80 → 8080)
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# Зберегти правила
sudo iptables-save > /etc/iptables/rules.v4
sudo apt install iptables-persistent  # автозавантаження

# nftables — сучасна заміна iptables (Ubuntu 20.04+)
sudo nft list ruleset
```

### 6.3 hosts file та DNS

```bash
# /etc/hosts — локальне DNS (пріоритет над DNS сервером)
cat /etc/hosts
# 127.0.0.1  localhost
# 10.0.0.5   db-server db-server.internal

# Додати запис
echo "10.0.0.5 db-server.internal" | sudo tee -a /etc/hosts

# /etc/resolv.conf — DNS конфігурація
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search mycompany.internal    # domain search list

# /etc/nsswitch.conf — порядок резолюції
grep hosts /etc/nsswitch.conf
# hosts: files dns  ← спочатку /etc/hosts, потім DNS

# DNS утиліти
dig google.com                  # повна DNS відповідь
dig google.com +short           # тільки IP
dig google.com MX               # mail records
dig @8.8.8.8 google.com         # використати конкретний DNS сервер
dig -x 8.8.8.8                  # reverse lookup
nslookup google.com
host google.com

# systemd-resolved (сучасні Ubuntu)
resolvectl status
resolvectl query google.com
```

### 6.4 Proxy Configurations

```bash
# HTTP/HTTPS proxy — системні змінні середовища
export http_proxy="http://proxy.company.com:3128"
export https_proxy="http://proxy.company.com:3128"
export no_proxy="localhost,127.0.0.1,.internal.company.com"

# Постійно (додай в /etc/environment)
cat >> /etc/environment << 'EOF'
http_proxy="http://proxy.company.com:3128"
https_proxy="http://proxy.company.com:3128"
no_proxy="localhost,127.0.0.1,.internal"
EOF

# apt через proxy
cat > /etc/apt/apt.conf.d/proxy.conf << 'EOF'
Acquire::http::Proxy "http://proxy:3128/";
Acquire::https::Proxy "http://proxy:3128/";
EOF

# systemd сервіс через proxy
sudo systemctl edit docker
# [Service]
# Environment="HTTP_PROXY=http://proxy:3128"
# Environment="HTTPS_PROXY=http://proxy:3128"
# Environment="NO_PROXY=localhost,127.0.0.1"

# Docker daemon через proxy
mkdir -p ~/.docker
cat > ~/.docker/config.json << 'EOF'
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy:3128",
      "httpsProxy": "http://proxy:3128",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
EOF

# Перевірити що proxy працює
curl -v --proxy http://proxy:3128 https://google.com
```

---

## 7. Практичні проекти

### Проект 1: Веб-сервер з нуля (2-3 год)

> 💡 **Навіщо:** розуміння того як Linux сервісна архітектура працює end-to-end

```bash
# 1. Встанови і налаштуй Nginx
sudo apt update && sudo apt install -y nginx

# 2. Створи Virtual Host для статичного сайту
sudo mkdir -p /var/www/mysite/html
sudo chown -R $USER:$USER /var/www/mysite/

cat > /var/www/mysite/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<body><h1>Hello from $(hostname)!</h1></body>
</html>
EOF

cat > /etc/nginx/sites-available/mysite << 'EOF'
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite/html;
    index index.html;

    access_log /var/log/nginx/mysite-access.log;
    error_log /var/log/nginx/mysite-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 3. Налаштуй /etc/hosts для тестування
echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts

# 4. Перевір
curl http://mysite.local
```

✅ **Перевірка:** `curl http://mysite.local` повертає HTML з hostname сервера

### Проект 2: Автоматичний backup скрипт (1-2 год)

> 💡 **Навіщо:** shell scripting + cron + логування — типова DevOps задача

```bash
cat > /opt/scripts/backup.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/var/backups/app"
SOURCE_DIR="/var/www/mysite"
LOG_FILE="/var/log/backup.log"
KEEP_DAYS=7

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

cleanup_old() {
  find "$BACKUP_DIR" -name "*.tar.gz" -mtime "+$KEEP_DAYS" -delete
  log "Cleaned up backups older than $KEEP_DAYS days"
}

mkdir -p "$BACKUP_DIR"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/mysite_$DATE.tar.gz"

log "Starting backup..."
tar -czf "$BACKUP_FILE" "$SOURCE_DIR"
log "Backup created: $BACKUP_FILE ($(du -sh "$BACKUP_FILE" | cut -f1))"

cleanup_old
log "Backup complete. Total backups: $(ls "$BACKUP_DIR"/*.tar.gz | wc -l)"
SCRIPT

chmod +x /opt/scripts/backup.sh
# Додай в cron — щодня о 2:00
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/scripts/backup.sh") | crontab -
```

✅ **Перевірка:** `bash /opt/scripts/backup.sh` виконується без помилок; перевір `/var/backups/app/` і `/var/log/backup.log`

### Проект 3: System Health Monitor (2-3 год)

> 💡 **Навіщо:** monitoring скрипт + systemd + alerts — основа observability

```bash
cat > /opt/scripts/health_check.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

# Пороги
CPU_THRESHOLD=80
MEMORY_THRESHOLD=85
DISK_THRESHOLD=90

ALERT_LOG="/var/log/health_alerts.log"
SERVICES=("nginx" "cron")

alert() {
  local msg="ALERT: $1"
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $msg" | tee -a "$ALERT_LOG"
}

ok() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] OK: $1"
}

# CPU check
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d'.' -f1)
if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ]; then
  alert "High CPU: ${CPU_USAGE}% (threshold: ${CPU_THRESHOLD}%)"
else
  ok "CPU usage: ${CPU_USAGE}%"
fi

# Memory check
MEM_USAGE=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
if [ "$MEM_USAGE" -gt "$MEMORY_THRESHOLD" ]; then
  alert "High Memory: ${MEM_USAGE}% (threshold: ${MEMORY_THRESHOLD}%)"
else
  ok "Memory usage: ${MEM_USAGE}%"
fi

# Disk check
while read -r usage mount; do
  usage_num=${usage%\%}
  if [ "$usage_num" -gt "$DISK_THRESHOLD" ]; then
    alert "High Disk on $mount: ${usage} (threshold: ${DISK_THRESHOLD}%)"
  else
    ok "Disk $mount: ${usage}"
  fi
done < <(df -h | awk 'NR>1 && $6!="tmpfs" && $1!="tmpfs" {print $5, $6}')

# Services check
for service in "${SERVICES[@]}"; do
  if systemctl is-active --quiet "$service"; then
    ok "Service $service is running"
  else
    alert "Service $service is DOWN! Attempting restart..."
    systemctl restart "$service" && ok "$service restarted" || alert "CRITICAL: Cannot restart $service"
  fi
done
SCRIPT

chmod +x /opt/scripts/health_check.sh

# Запускай кожні 5 хвилин через cron
(crontab -l 2>/dev/null; echo "*/5 * * * * /opt/scripts/health_check.sh >> /var/log/health_check.log 2>&1") | crontab -
```

✅ **Перевірка:** `bash /opt/scripts/health_check.sh` виводить статус; `crontab -l` показує завдання

### Проект 4: Troubleshooting Scenarios

Практикуй на [sadservers.com](https://sadservers.com) — реальні сценарії зломаних Linux серверів.

Рекомендовані сценарії:
1. **"Saint John"** — неправильні права файлів
2. **"Tokyo"** — проблема з процесом і портом
3. **"Brasilia"** — файлова система переповнена
4. **"Manhattan"** — база даних не стартує

---

## Типові помилки

| Симптом | Причина | Як виправити |
|---------|---------|--------------|
| `Permission denied` при запуску скрипта | Відсутній execute bit | `chmod +x script.sh` |
| `command not found` для встановленої програми | Не в PATH або не в тій сесії | `which program`, `source ~/.bashrc`, або повний шлях `/usr/local/bin/program` |
| SSH `Permission denied (publickey)` | Неправильні права на ключ або authorized_keys | `chmod 600 ~/.ssh/id_rsa`, `chmod 700 ~/.ssh/` |
| Диск 100% але `du` не показує великих файлів | Deleted but open files (процес тримає файл відкритим) | `lsof +L1` знайде такі файли; `systemctl restart nginx` закриє їх |
| `No space left on device` але `df` показує місце | Закінчились inodes | `df -i`, знайти директорію з мільйонами дрібних файлів `find / -xdev -printf '%h\n' \| sort \| uniq -c \| sort -rn \| head` |
| Процес у стані `D` (Uninterruptible Sleep) | Чекає I/O (часто NFS або disk issue) | Перевір disk I/O: `iostat -x 1`, NFS з'єднання: `nfsstat` |
| `sudo: command not found` для звичайного користувача | Юзер не в sudoers | `sudo visudo` або `sudo usermod -aG sudo username` |
| nginx не стартує — `Address already in use` | Щось займає порт 80 | `ss -tlnp \| grep :80`, `sudo fuser -k 80/tcp` |
| Cron job не виконується | Неправильний PATH або відсутні змінні середовища | Додай `PATH=/usr/local/bin:/usr/bin:/bin` на початку crontab, логуй вивід |
| `bash: fork: retry: Resource temporarily unavailable` | Закінчились PID-и або процеси | `ulimit -u`, `cat /proc/sys/kernel/pid_max`, знайди fork bomb |

---

## Результат модуля

Після завершення ти повинен мати:

- [ ] Розгорнуте Linux середовище (Vagrant або EC2)
- [ ] Встановлений і налаштований Nginx з Virtual Host
- [ ] Власний shell скрипт для backup з ротацією
- [ ] Health check скрипт запущений через cron
- [ ] Розуміння як читати `journalctl`, `ss`, `df -i`, `ps auxf`
- [ ] Мінімум 3 вирішені сценарії на sadservers.com
- [ ] Hardened SSH конфіг (no password auth, no root login)
- [ ] Розуміння різниці між process states, namespaces і cgroups

**GitHub deliverable:** репо `linux-lab/` з:
```
linux-lab/
├── scripts/
│   ├── backup.sh
│   └── health_check.sh
├── nginx/
│   └── mysite.conf
├── docs/
│   └── troubleshooting.md   ← задокументуй 3 проблеми що вирішив
└── README.md
```

> 🏗️ **Capstone зв'язок:** Всі ці скрипти і конфіги стануть основою для Ansible ролей у Тижні 4 — замість запускати їх вручну, Ansible розгорне їх автоматично на всіх серверах.

---

## Interview Prep

**Питання які тобі зададуть:**

| Питання | Де ти це робив | Ключові слова відповіді |
|---------|---------------|------------------------|
| Що таке cgroups і навіщо вони? | Розділ 1.10 + Docker | resource limits, CPU/RAM isolation, основа контейнерів |
| Різниця між SIGTERM і SIGKILL? | Розділ 1.8 | graceful shutdown, перехоплення, 15 vs 9 |
| Як знайти що займає місце на диску? | Розділ 4.4 + 2.1 | `df -h`, `du -sh`, `ncdu`, deleted open files `lsof +L1` |
| Що таке inode? Чому може закінчитись місце, а inodes нема? | Розділ 2.3 | metadata, ім'я файлу в directory entry, `df -i` |
| Як Linux namespaces пов'язані з Docker? | Розділ 1.11 | pid/net/mnt/uts ізоляція, `nsenter` |
| Як зробити graceful reload nginx без downtime? | Розділ 3.4 | `systemctl reload`, SIGHUP, zero-downtime |
| Що таке LVM і коли його використовувати? | Розділ 2.5 | PV/VG/LV, dynamic resize, production storage |
| Як дебажити процес що зависає? | Розділ 4.5 | `strace -p`, `lsof -p`, `/proc/PID/wchan`, process state D |
| Різниця між AppArmor і SELinux? | Розділ 5.3-5.4 | MAC, profiles vs labels, Ubuntu vs RHEL |

**Питання які задай ТИ:**
- "Які Linux дистрибутиви використовує ваш production і чому?"
- "Як ви моніторите системні ресурси — яким стеком?"
- "Як влаштована ваша система логування: централізована чи per-server?"

**Ресурс:** [Linux Journey](https://linuxjourney.com) — інтерактивне навчання · [The Linux Command Line (book)](https://linuxcommand.org/tlcl.php) — безкоштовно · [sadservers.com](https://sadservers.com) — практика troubleshooting

---

*Версія модуля: 1.0 · Охоплює: Kernel, Storage, Management, Logging, Security, Networking*
