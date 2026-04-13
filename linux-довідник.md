# 🐧 Linux — Повний довідник

> Довідник охоплює такі ключові теми: архітектуру файлової системи, керування користувачами, права доступу, автоматизацію, bash-скриптинг, планування завдань та логування.

---

## Зміст

1. [Архітектура файлової системи](#1-архітектура-файлової-системи)
2. [Керування пристроями](#2-керування-пристроями)
3. [Навігація та базові команди CLI](#3-навігація-та-базові-команди-cli)
4. [Керування користувачами](#4-керування-користувачами)
5. [Права доступу до файлів](#5-права-доступу-до-файлів)
6. [Маніпуляції з вмістом файлів](#6-маніпуляції-з-вмістом-файлів)
7. [Керування процесами](#7-керування-процесами)
8. [Пакетні менеджери](#8-пакетні-менеджери)
9. [Bash-скриптинг](#9-bash-скриптинг)
10. [Текстові утиліти: grep, sed, awk](#10-текстові-утиліти-grep-sed-awk)
11. [Pipes та перенаправлення](#11-pipes-та-перенаправлення)
12. [Планування завдань: Cron](#12-планування-завдань-cron)
13. [Логування та ротація логів](#13-логування-та-ротація-логів)
14. [Мережеві команди](#14-мережеві-команди)
15. [SSH та віддалений доступ](#15-ssh-та-віддалений-доступ)
16. [Troubleshooting: підхід та інструменти](#16-troubleshooting-підхід-та-інструменти)

---

## 1. Архітектура файлової системи

Linux використовує єдину ієрархічну файлову систему, яка починається з кореневого каталогу `/`. Все — диски, пристрої, мережеві ресурси — монтується в цьому дереві.

### Ключові директорії

| Директорія | Призначення |
|---|---|
| `/` | Корінь файлової системи |
| `/bin`, `/usr/bin` | Бінарні файли та програми для всіх користувачів |
| `/sbin`, `/usr/sbin` | Системні утиліти, що вимагають прав root |
| `/etc` | Конфігураційні файли системи та сервісів (nginx, ssh, cron тощо) |
| `/home` | Домашні директорії користувачів (`/home/username`) |
| `/root` | Домашня директорія суперкористувача root |
| `/tmp` | Тимчасові файли (очищаються при перезавантаженні) |
| `/var` | Змінні дані: логи, кеш, черги друку (`/var/log`, `/var/cache`) |
| `/var/log` | Системні та аплікаційні логи |
| `/proc` | Псевдо-ФС: інформація про процеси та ядро в реальному часі |
| `/sys` | Псевдо-ФС: інтерфейс до драйверів та пристроїв (sysfs) |
| `/dev` | Файли пристроїв (диски, термінали тощо) |
| `/mnt`, `/media` | Точки монтування зовнішніх дисків і накопичувачів |
| `/opt` | Опціональне стороннє ПЗ |
| `/lib`, `/usr/lib` | Спільні бібліотеки (.so файли) |
| `/boot` | Файли завантажувача (kernel, initrd, grub) |

### Монтування (Mount)

Монтування — підключення файлової системи (диска, USB, NFS) до певної точки дерева каталогів.

```bash
# Переглянути всі змонтовані ФС
mount
df -h

# Змонтувати диск вручну
mount /dev/sdb1 /mnt/data

# Розмонтувати
umount /mnt/data
```

**Автоматичне монтування при завантаженні — `/etc/fstab`:**

```
# <пристрій>         <точка монтув.>  <тип ФС>  <опції>         <dump> <pass>
UUID=1234-ABCD       /mnt/data        ext4      defaults,noatime  0      2
192.168.1.10:/share  /mnt/nfs         nfs       defaults          0      0
```

### Іноди (Inodes)

Inode — структура метаданих, що зберігає інформацію про кожен файл:

- Розмір файлу
- Права доступу (permission bits)
- Власник (UID/GID)
- Часові мітки: створення, зміни, доступу
- Вказівники на фізичні блоки даних на диску

```bash
# Переглянути inode файлу
ls -i filename
stat filename

# Перевірити використання інодів (важливо! можна вичерпати inode при повному диску)
df -i
```

> ⚠️ **Важливо:** Можна вичерпати ліміт інодів навіть якщо на диску є вільне місце. Це заблокує створення нових файлів. Типова причина — велика кількість дрібних файлів (логи, кеш, сесії).

---

## 2. Керування пристроями

У Linux **будь-який пристрій — це файл** у директорії `/dev`.

### Типи файлів пристроїв

| Позначення | Тип | Приклад |
|---|---|---|
| `b` | Block device (диски) | `/dev/sda`, `/dev/nvme0n1` |
| `c` | Character device (символьні) | `/dev/tty`, `/dev/null`, `/dev/random` |

### Корисні команди

```bash
# Список блочних пристроїв
lsblk

# Детальна інформація про диск
fdisk -l /dev/sda

# Перегляд підключених USB/PCI пристроїв
lsusb
lspci

# Моніторинг подій підключення пристроїв (udev)
udevadm monitor

# Перегляд повідомлень ядра про пристрої
dmesg | grep -i usb
dmesg | tail -50
```

### udev — динамічне виявлення пристроїв

`udev` — менеджер пристроїв ядра Linux. Автоматично виявляє та обробляє події підключення/відключення пристроїв. Правила зберігаються у `/etc/udev/rules.d/`.

---

## 3. Навігація та базові команди CLI

### Навігація

```bash
pwd                   # Поточна директорія (Print Working Directory)
ls                    # Список файлів
ls -la                # Детальний список з прихованими файлами
ls -lh                # Розміри у зручному форматі (KB, MB)
ls -lt                # Сортування за датою зміни
cd /path/to/dir       # Перехід до директорії
cd ~                  # Перехід у домашню директорію
cd ..                 # На рівень вище
cd -                  # Повернення в попередню директорію
```

### Робота з файлами та директоріями

```bash
# Створення
mkdir mydir                    # Створити директорію
mkdir -p /path/to/nested/dir   # Створити з усіма батьківськими директоріями
touch file.txt                 # Створити порожній файл або оновити дату

# Копіювання
cp file.txt /tmp/              # Копіювати файл
cp -r /src/dir /dst/dir        # Копіювати директорію рекурсивно
cp -p file.txt /tmp/           # Зберегти метадані (права, дату)

# Переміщення та перейменування
mv file.txt /tmp/              # Перемістити
mv old_name.txt new_name.txt   # Перейменувати

# Видалення
rm file.txt                    # Видалити файл
rm -rf /path/to/dir            # Видалити директорію рекурсивно (обережно!)
rmdir emptydir                 # Видалити лише порожню директорію

# Пошук файлів
find / -name "*.log" 2>/dev/null          # Пошук по імені
find /var/log -type f -mtime -7           # Файли змінені за останні 7 днів
find . -size +100M                        # Файли більше 100 МБ
locate filename                           # Швидкий пошук через індекс (updatedb)
which python3                             # Де знаходиться виконуваний файл
```

### Перегляд файлів

```bash
cat file.txt              # Вивести весь вміст
less file.txt             # Посторінковий перегляд (q — вихід, /pattern — пошук)
head -n 20 file.txt       # Перші 20 рядків
tail -n 50 file.txt       # Останні 50 рядків
tail -f /var/log/app.log  # Стежити за файлом у реальному часі (live monitoring)
wc -l file.txt            # Кількість рядків у файлі
```

### Дискова статистика

```bash
df -h                     # Використання дисків (human-readable)
du -sh /var/log           # Розмір директорії
du -sh * | sort -h        # Розміри всіх підкаталогів з сортуванням
```

---

## 4. Керування користувачами

Linux — багатокористувацька система. Кожен процес виконується від імені певного користувача.

### Ключові файли

| Файл | Вміст |
|---|---|
| `/etc/passwd` | Список користувачів: `username:x:UID:GID:comment:home:shell` |
| `/etc/shadow` | Хеші паролів (доступно лише root) |
| `/etc/group` | Список груп та їх учасники |
| `/etc/sudoers` | Налаштування привілеїв sudo |

```bash
# Переглянути запис користувача
getent passwd username
cat /etc/passwd | grep username
id username               # UID, GID та групи користувача
whoami                    # Поточний користувач
who                       # Хто зараз увійшов у систему
w                         # Детальна інформація про активних користувачів
last                      # Історія входів
```

### Керування користувачами

```bash
# Створення
useradd -m -s /bin/bash -G sudo,docker newuser
# -m = створити домашню директорію
# -s = shell
# -G = додаткові групи

useradd -r -s /usr/sbin/nologin serviceuser  # Системний користувач (для сервісів)

# Встановлення пароля
passwd username

# Зміна налаштувань
usermod -aG docker username   # Додати до групи (без -a замінить всі групи)
usermod -s /bin/zsh username  # Змінити shell
usermod -L username           # Заблокувати обліковий запис
usermod -U username           # Розблокувати

# Видалення
userdel username              # Видалити без домашньої директорії
userdel -r username           # Видалити разом з домашньою директорією

# Групи
groupadd devops
groupdel devops
groups username               # Групи користувача
```

### sudo та принцип найменших привілеїв

```bash
sudo command              # Виконати команду як root
sudo -u otheruser command # Виконати як інший користувач
sudo -i                   # Інтерактивний shell від root
su - username             # Переключитись на іншого користувача

# Редагування /etc/sudoers (завжди через visudo!)
visudo
```

**Приклади записів у `/etc/sudoers`:**

```
# Повний доступ без пароля
username ALL=(ALL) NOPASSWD: ALL

# Дозволити лише конкретні команди
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, /usr/bin/docker

# Група devops може використовувати sudo
%devops ALL=(ALL) ALL
```

> 🔐 **Принцип найменших привілеїв (Least Privilege):** надавайте користувачам лише ті права, які їм дійсно потрібні. Ніколи не запускайте сервіси від імені root без крайньої потреби.

---

## 5. Права доступу до файлів

### Структура прав

```
-rwxr-xr--  1  alice  devops  4096  Jan 10 12:00  script.sh
│└──┬───┘       │      │
│   │           │      └── Група (Group)
│   │           └───────── Власник (Owner)
│   └───────────────────── Права: owner | group | others
└───────────────────────── Тип: - файл, d директорія, l посилання
```

| Символ | Число | Значення для файлу | Значення для директорії |
|---|---|---|---|
| `r` | 4 | Читання | Перегляд вмісту (ls) |
| `w` | 2 | Запис | Створення/видалення файлів |
| `x` | 1 | Виконання | Вхід у директорію (cd) |
| `-` | 0 | Відсутнє право | Відсутнє право |

### chmod — зміна прав

```bash
# Числовий (вісімковий) формат
chmod 755 script.sh     # rwxr-xr-x (стандарт для скриптів і директорій)
chmod 644 config.txt    # rw-r--r-- (стандарт для файлів)
chmod 600 secret.key    # rw------- (приватні ключі, паролі)
chmod 777 file          # ⚠️ Небезпечно! Всі мають всі права

# Символьний формат
chmod +x script.sh      # Додати право виконання для всіх
chmod u+x script.sh     # Тільки для власника
chmod g-w file.txt      # Забрати право запису у групи
chmod o= file.txt       # Очистити всі права для "інших"
chmod a+r file.txt      # Всім (all) додати читання

# Рекурсивно
chmod -R 755 /var/www/html
```

### Типові числові значення прав

| Код | Права | Використання |
|---|---|---|
| `700` | `rwx------` | Директорія лише для власника |
| `755` | `rwxr-xr-x` | Публічні скрипти, веб-директорії |
| `644` | `rw-r--r--` | Звичайні файли, конфіги |
| `600` | `rw-------` | Приватні ключі SSH, паролі |
| `400` | `r--------` | Тільки для читання власником |

### chown та chgrp — зміна власника

```bash
chown alice file.txt             # Змінити власника
chown alice:devops file.txt      # Змінити власника і групу
chown -R www-data:www-data /var/www/  # Рекурсивно

chgrp devops /srv/project        # Змінити тільки групу
```

### Спеціальні біти прав

```bash
# SUID (Set User ID) — файл виконується з правами власника
chmod u+s /usr/bin/passwd     # 4755
# SGID (Set Group ID) — нові файли успадковують групу директорії  
chmod g+s /srv/shared/         # 2755
# Sticky bit — видалення файлу лише самим власником (для /tmp)
chmod +t /tmp                  # 1777
```

---

## 6. Маніпуляції з вмістом файлів

### Перегляд та редагування

```bash
cat file.txt                  # Вивести вміст
cat -n file.txt               # З нумерацією рядків
tac file.txt                  # Вивести у зворотному порядку

# Редактори
nano file.txt                 # Простий редактор (Ctrl+O зберегти, Ctrl+X вийти)
vim file.txt                  # Потужний редактор (i — вставка, :wq — зберегти і вийти)

# Порівняння файлів
diff file1.txt file2.txt
diff -u file1.txt file2.txt   # Unified формат (зручніший)
```

### Створення та запис

```bash
# Створення та перезапис
echo "Hello World" > file.txt        # Перезаписати (або створити)
echo "New line" >> file.txt          # Додати рядок в кінець (append)

# Heredoc — багаторядковий запис
cat > config.txt << EOF
server_name = myapp
port = 8080
debug = false
EOF

# Перенаправлення виводу команди у файл
date > /tmp/now.txt
ls -la >> /tmp/listing.txt
```

### Пошук у файлах

```bash
grep "error" /var/log/app.log           # Знайти рядки з "error"
grep -i "error" file.txt                # Без урахування регістру
grep -r "TODO" /src/                    # Рекурсивний пошук по директорії
grep -n "pattern" file.txt              # З номерами рядків
grep -v "DEBUG" app.log                 # Виключити рядки з "DEBUG"
grep -c "ERROR" app.log                 # Підрахувати кількість збігів
grep -E "error|warning|critical" *.log  # Розширені регулярні вирази
grep -A 3 -B 3 "ERROR" app.log          # 3 рядки до і після знайденого
```

### Архівування

```bash
# Створення tar-архіву
tar -czvf backup.tar.gz /var/www/html/
# -c create, -z gzip, -v verbose, -f file

# Розпакування
tar -xzvf backup.tar.gz
tar -xzvf backup.tar.gz -C /tmp/       # Розпакувати в конкретну директорію

# Переглянути вміст архіву
tar -tzvf backup.tar.gz

# zip/unzip
zip -r archive.zip /path/to/dir
unzip archive.zip -d /target/dir
```

---

## 7. Керування процесами

Кожна запущена програма — це **процес** з унікальним PID (Process ID).

### Перегляд процесів

```bash
ps aux                  # Всі процеси (a=all, u=user format, x=без термінала)
ps aux | grep nginx     # Знайти конкретний процес
pstree                  # Дерево процесів
pgrep nginx             # Отримати PID за ім'ям
pidof nginx             # Аналог pgrep
```

### Інтерактивний моніторинг

```bash
top                     # Базовий диспетчер задач
htop                    # Покращений (потрібно встановити: apt install htop)
# У htop: F5 = дерево, F9 = kill, F6 = сортування
```

### Сигнали та завершення процесів

```bash
kill PID                # Надіслати SIGTERM (graceful shutdown)
kill -9 PID             # SIGKILL — примусове завершення
kill -15 PID            # SIGTERM — м'яке завершення (за замовчуванням)
kill -HUP PID           # SIGHUP — перечитати конфігурацію (reload)
killall nginx           # Завершити всі процеси за ім'ям
pkill -u username       # Завершити всі процеси користувача

# Перевірка
kill -0 PID             # Перевірити чи існує процес (без надсилання сигналу)
```

### Фонові процеси та jobs

```bash
command &               # Запустити у фоні
jobs                    # Список фонових задач
fg %1                   # Повернути задачу №1 на передній план
bg %1                   # Відправити призупинену задачу у фон
Ctrl+Z                  # Призупинити поточний процес
Ctrl+C                  # Завершити поточний процес (SIGINT)

# nohup — запуск, що не зупиниться при закритті сесії
nohup ./long_script.sh > output.log 2>&1 &

# screen / tmux — термінальні мультиплексори
screen -S mysession     # Нова сесія
screen -r mysession     # Підключитись до існуючої
```

### Пріоритети процесів

```bash
nice -n 10 command      # Запустити з низьким пріоритетом (-20 до 19)
renice -n 5 -p PID      # Змінити пріоритет запущеного процесу
```

### Системні сервіси (systemd)

```bash
systemctl status nginx          # Статус сервісу
systemctl start nginx           # Запустити
systemctl stop nginx            # Зупинити
systemctl restart nginx         # Перезапустити
systemctl reload nginx          # Перезавантажити конфіг без зупинки
systemctl enable nginx          # Автозапуск при старті системи
systemctl disable nginx         # Вимкнути автозапуск
systemctl list-units --type=service  # Всі сервіси
journalctl -u nginx             # Логи конкретного сервісу
journalctl -u nginx -f          # Логи в реальному часі
journalctl --since "1 hour ago" # Логи за останню годину
```

---

## 8. Пакетні менеджери

### Debian / Ubuntu — APT

```bash
apt update                        # Оновити список пакетів із репозиторіїв
apt upgrade                       # Оновити всі встановлені пакети
apt install nginx                 # Встановити пакет
apt install -y nginx              # Без підтвердження (для скриптів)
apt remove nginx                  # Видалити пакет (зберегти конфіги)
apt purge nginx                   # Видалити пакет і конфіги
apt autoremove                    # Видалити непотрібні залежності
apt search nginx                  # Пошук пакету
apt show nginx                    # Інформація про пакет
dpkg -l | grep nginx              # Перевірити чи встановлено
dpkg -L nginx                     # Файли, встановлені пакетом
```

### RHEL / CentOS / Fedora — YUM / DNF

```bash
dnf update                        # Оновити всі пакети
dnf install nginx                 # Встановити
dnf remove nginx                  # Видалити
dnf search nginx                  # Пошук
dnf info nginx                    # Інформація
dnf list installed                # Список встановлених
rpm -qa | grep nginx              # Пошук через rpm
rpm -ql nginx                     # Файли пакету
```

### Управління репозиторіями

```bash
# Debian/Ubuntu
add-apt-repository ppa:ondrej/php   # Додати PPA
cat /etc/apt/sources.list           # Список репозиторіїв

# RHEL/CentOS
dnf config-manager --add-repo URL
ls /etc/yum.repos.d/               # Конфіги репозиторіїв
```

> 🏢 **В корпоративному середовищі:** для безпеки часто розгортають власні ізольовані репозиторії (Nexus, Artifactory, APT Mirror) замість публічних. Це дозволяє контролювати версії та перевіряти пакети на вразливості.

---

## 9. Bash-скриптинг

### Структура скрипту

```bash
#!/bin/bash
# Shebang — вказує інтерпретатор
# set -e: зупинити скрипт при будь-якій помилці
# set -u: помилка при використанні невизначеної змінної
# set -o pipefail: помилка якщо будь-яка команда в pipe впала
set -euo pipefail

echo "Script started"
```

### Змінні

```bash
NAME="Alice"               # Оголошення (без пробілів навколо =)
echo $NAME                 # Використання
echo "${NAME}"             # Безпечний варіант (рекомендується)
echo "Hello, ${NAME}!"

RESULT=$(date)             # Зберегти вивід команди
COUNT=$((5 + 3))           # Арифметика

# Змінні за замовчуванням
DB_PORT=${DB_PORT:-5432}   # Використати 5432 якщо DB_PORT не задано

# Спеціальні змінні
echo $0   # Ім'я скрипту
echo $1   # Перший аргумент
echo $@   # Всі аргументи
echo $#   # Кількість аргументів
echo $?   # Код виходу попередньої команди (0 = успіх)
echo $$   # PID поточного процесу
```

### Умовні конструкції

```bash
# if/elif/else
if [[ $1 == "start" ]]; then
    echo "Starting..."
elif [[ $1 == "stop" ]]; then
    echo "Stopping..."
else
    echo "Unknown command: $1"
fi

# Перевірки файлів
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "Config exists"
fi

if [[ -d /var/log/myapp ]]; then
    echo "Directory exists"
fi

# Перевірки рядків
if [[ -z "$VAR" ]]; then echo "Empty"; fi   # Порожній рядок
if [[ -n "$VAR" ]]; then echo "Not empty"; fi

# Числові порівняння
if [[ $NUM -gt 10 ]]; then echo "Greater"; fi
# -eq, -ne, -lt, -le, -gt, -ge
```

### Цикли

```bash
# for — перебір елементів
for server in web1 web2 web3; do
    echo "Deploying to $server..."
    ssh $server "systemctl restart myapp"
done

# for — числовий
for i in {1..10}; do
    echo "Step $i"
done

# for — по файлах
for log in /var/log/*.log; do
    echo "Processing: $log"
done

# while — умовний
COUNT=0
while [[ $COUNT -lt 5 ]]; do
    echo "Count: $COUNT"
    ((COUNT++))
done

# while — читання файлу рядок за рядком
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt
```

### Функції

```bash
# Оголошення функції
log_message() {
    local level=$1        # local = локальна змінна
    local message=$2
    local timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$timestamp] [$level] $message"
}

# Виклик
log_message "INFO" "Script started"
log_message "ERROR" "Connection failed"

# Функція з поверненням значення
get_cpu_usage() {
    local usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
    echo $usage  # "повернення" через echo
}
CPU=$(get_cpu_usage)
echo "CPU usage: $CPU%"
```

### Обробка помилок та виходи

```bash
# Перевірка коду виходу
if ! command -v docker &>/dev/null; then
    echo "ERROR: Docker not found"
    exit 1
fi

# Trap — виконати код при виході або помилці
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/lock_file
}
trap cleanup EXIT          # При будь-якому виході
trap cleanup ERR           # При помилці
trap cleanup INT TERM      # При Ctrl+C або kill

# Перевірка чи запущено з root
if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root"
    exit 1
fi
```

### Практичний приклад: скрипт розгортання

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
DEPLOY_DIR="/opt/$APP_NAME"
BACKUP_DIR="/opt/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

log() { echo "[$(date '+%H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; exit 1; }

# Перевірки
[[ $# -ne 1 ]] && error "Usage: $0 <version>"
VERSION=$1

log "Deploying $APP_NAME version $VERSION"

# Бекап поточної версії
if [[ -d "$DEPLOY_DIR" ]]; then
    log "Backing up current version..."
    tar -czf "$BACKUP_DIR/${APP_NAME}_${TIMESTAMP}.tar.gz" "$DEPLOY_DIR"
fi

# Розгортання
log "Starting deployment..."
systemctl stop "$APP_NAME" || true
rsync -av "/releases/$VERSION/" "$DEPLOY_DIR/"
systemctl start "$APP_NAME"

log "Deployment of $APP_NAME v$VERSION completed successfully"
```

---

## 10. Текстові утиліти: grep, sed, awk

### grep — пошук за шаблоном

```bash
grep "pattern" file.txt
grep -E "^ERROR|^WARN" app.log    # Розширені regex
grep -P "\d{1,3}\.\d{1,3}" file  # Perl-сумісні regex

# Корисні прапори
-i    # Ігнорувати регістр
-r    # Рекурсивно по директорії
-l    # Показати лише імена файлів
-n    # Показати номери рядків
-v    # Інвертований пошук (рядки БЕЗ шаблону)
-c    # Підрахувати збіги
-A N  # N рядків після збігу
-B N  # N рядків до збігу
-C N  # N рядків до і після

# Приклади
grep -rn "TODO" /src/ --include="*.py"
grep "ERROR" /var/log/nginx/error.log | tail -20
```

### sed — потоковий редактор

```bash
# Заміна тексту (s/знайти/замінити/прапори)
sed 's/foo/bar/' file.txt          # Перше входження у кожному рядку
sed 's/foo/bar/g' file.txt         # Всі входження (global)
sed 's/foo/bar/gi' file.txt        # Без урахування регістру
sed -i 's/foo/bar/g' file.txt      # Заміна безпосередньо у файлі
sed -i.bak 's/foo/bar/g' file.txt  # Заміна з бекапом (.bak)

# Видалення рядків
sed '/^#/d' config.txt              # Видалити рядки-коментарі
sed '/^$/d' file.txt                # Видалити порожні рядки
sed '5d' file.txt                   # Видалити 5-й рядок

# Виведення конкретних рядків
sed -n '10,20p' file.txt            # Рядки 10-20
sed -n '/START/,/END/p' file.txt    # Між маркерами

# Практичний приклад: заміна IP в конфігу
sed -i "s/listen 80/listen ${PORT}/g" /etc/nginx/nginx.conf
```

### awk — обробка стовпців та табличних даних

```bash
# Базовий синтаксис
awk '{print $1}' file.txt          # Перший стовпець (роздільник — пробіл)
awk -F: '{print $1}' /etc/passwd   # Перший стовпець, роздільник ":"
awk '{print $1, $NF}' file.txt     # Перший та останній стовпці

# Умови
awk '$3 > 100 {print $1, $3}' data.txt   # Рядки де 3-й стовпець > 100
awk '/ERROR/ {print $0}' app.log          # Рядки, що містять ERROR

# Вбудовані змінні
# NR — номер рядка, NF — кількість стовпців, FS — роздільник, $0 — весь рядок

# Підрахунок
awk '{sum += $3} END {print "Total:", sum}' data.txt

# Практичний приклад: виведення IP з nginx access.log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

---

## 11. Pipes та перенаправлення

### Стандартні потоки

| Потік | Номер | Опис |
|---|---|---|
| `stdin` | 0 | Стандартний ввід (клавіатура за замовчуванням) |
| `stdout` | 1 | Стандартний вивід (екран за замовчуванням) |
| `stderr` | 2 | Вивід помилок |

### Перенаправлення

```bash
command > file.txt       # Перенаправити stdout у файл (перезапис)
command >> file.txt      # Додати stdout в кінець файлу
command 2> errors.log    # Перенаправити stderr
command 2>> errors.log   # Додати stderr
command > out.log 2>&1   # stdout і stderr в один файл
command &> out.log       # Скорочений варіант попереднього
command 2>/dev/null      # Приховати помилки (/dev/null — "чорна діра")
command > /dev/null 2>&1 # Приховати весь вивід

# Ввід з файлу
command < input.txt
```

### Конвеєри (Pipes)

```bash
# | передає stdout однієї команди як stdin для наступної
cat /var/log/nginx/access.log | grep "404" | wc -l
ps aux | grep -v grep | grep nginx | awk '{print $2}'

# tee — одночасно вивести і записати у файл
command | tee output.log          # Перезаписати
command | tee -a output.log       # Додати

# xargs — конвертувати stdin в аргументи
find . -name "*.tmp" | xargs rm
find . -name "*.log" | xargs -I {} gzip {}
echo "nginx apache2" | xargs -n 1 systemctl restart
```

---

## 12. Планування завдань: Cron

### Синтаксис crontab

```
* * * * *  command
│ │ │ │ │
│ │ │ │ └── День тижня (0-7, де 0 і 7 = неділя)
│ │ │ └──── Місяць (1-12)
│ │ └────── День місяця (1-31)
│ └──────── Година (0-23)
└────────── Хвилина (0-59)
```

### Спеціальні синтаксиси

| Запис | Значення |
|---|---|
| `*` | Будь-яке значення |
| `*/15` | Кожні 15 одиниць |
| `1,15,30` | О 1-й, 15-й і 30-й хвилині |
| `1-5` | З 1 по 5 |
| `@reboot` | При завантаженні системи |
| `@daily` | Раз на день (00:00) |
| `@weekly` | Раз на тиждень |
| `@hourly` | Кожну годину |

### Управління crontab

```bash
crontab -e              # Редагувати crontab поточного користувача
crontab -l              # Переглянути задачі
crontab -r              # Видалити всі задачі (обережно!)
crontab -u username -e  # Редагувати crontab іншого користувача (від root)

# Системні cron-директорії
/etc/cron.d/       # Системні задачі у конфіг-файлах
/etc/cron.hourly/  # Скрипти що виконуються щогодини
/etc/cron.daily/   # Щоденно
/etc/cron.weekly/  # Щотижня
```

### Приклади cron-записів

```bash
# Кожні 15 хвилин
*/15 * * * * /opt/scripts/health-check.sh

# Щодня о 3:00 ночі
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# По понеділках о 9:00
0 9 * * 1 /opt/scripts/weekly-report.sh

# Перший день кожного місяця
0 0 1 * * /opt/scripts/monthly-cleanup.sh

# Кожні 5 хвилин з 8:00 до 18:00 у робочі дні
*/5 8-18 * * 1-5 /opt/scripts/monitor.sh

# При перезавантаженні
@reboot /opt/scripts/init.sh > /var/log/init.log 2>&1
```

> ⚠️ **Критично важливо в cron:** завжди вказуйте **абсолютні шляхи** до скриптів та команд (`/usr/bin/python3`, не `python3`). Cron має обмежений `PATH`. Завжди перенаправляйте вивід у лог-файл.

---

## 13. Логування та ротація логів

### Розташування логів

| Файл / директорія | Вміст |
|---|---|
| `/var/log/syslog` | Загальний системний журнал (Debian) |
| `/var/log/messages` | Загальний журнал (RHEL) |
| `/var/log/auth.log` | Автентифікація, sudo, SSH (Debian) |
| `/var/log/secure` | Автентифікація (RHEL) |
| `/var/log/kern.log` | Повідомлення ядра |
| `/var/log/nginx/` | Логи Nginx (access.log, error.log) |
| `/var/log/mysql/` | Логи MySQL |
| `/var/log/journal/` | Журнал systemd (journald) |

### Перегляд логів

```bash
tail -f /var/log/syslog                    # Моніторинг у реальному часі
tail -n 100 /var/log/auth.log              # Останні 100 рядків
grep "Failed password" /var/log/auth.log   # Невдалі спроби входу
journalctl -u nginx --since "2024-01-10"   # Логи сервісу з певної дати
journalctl -p err                          # Тільки помилки
```

### Рівні логування

```
CRITICAL  — критична помилка, система не може продовжити роботу
ERROR     — помилка, операція не виконана
WARNING   — попередження, щось пішло не так але система продовжує роботу
INFO      — інформаційне повідомлення про нормальну роботу
DEBUG     — детальна інформація для відлагодження (вимикати на Production)
```

### logrotate — ротація логів

Конфігурація в `/etc/logrotate.conf` та `/etc/logrotate.d/`:

```
/var/log/myapp/*.log {
    daily              # Ротація щодня
    rotate 14          # Зберігати 14 файлів
    compress           # Стискати gzip
    delaycompress      # Стискати лише другий та старіші файли
    missingok          # Не помилятись якщо файл відсутній
    notifempty         # Не ротувати порожній файл
    create 0640 www-data adm   # Створити новий файл з правами
    postrotate
        systemctl reload nginx   # Дія після ротації
    endscript
}
```

```bash
# Перевірити конфігурацію logrotate
logrotate -d /etc/logrotate.d/myapp    # Debug режим (без реальної ротації)
logrotate -f /etc/logrotate.d/myapp    # Примусова ротація
```

### Логування у bash-скриптах

```bash
LOG_FILE="/var/log/myapp/deploy.log"
LOG_LEVEL="INFO"

log() {
    local level=$1
    shift
    echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') [$level] $*" | tee -a "$LOG_FILE"
}

log INFO  "Deployment started"
log ERROR "Connection to DB failed"
log DEBUG "Variable value: $VAR"
```

> 🕐 **Важливо:** Використовуйте **UTC** для всіх часових міток у логах. Це спрощує кореляцію подій між різними серверами та часовими поясами.

---

## 14. Мережеві команди

### Перевірка мережі

```bash
ip addr show           # IP-адреси інтерфейсів (замінює ifconfig)
ip route show          # Таблиця маршрутизації
ip link show           # Стан мережевих інтерфейсів

ping -c 4 google.com   # Перевірка доступності хоста
traceroute google.com  # Шлях пакету до хоста
mtr google.com         # Поєднання ping і traceroute

# DNS
nslookup google.com
dig google.com
dig @8.8.8.8 google.com A   # Запит до конкретного DNS-сервера
cat /etc/resolv.conf         # Налаштування DNS
```

### Порти та з'єднання

```bash
ss -tulpn                       # Відкриті порти і процеси що їх слухають
ss -tulpn | grep :80            # Конкретний порт
netstat -tulpn                  # Аналог ss (застарілий, але ще поширений)
lsof -i :8080                   # Який процес використовує порт 8080

# Перевірка доступності порту
nc -zv hostname 80              # TCP
nc -zvw 3 hostname 80           # З таймаутом 3 сек
curl -I https://example.com     # HTTP-запит
```

### Завантаження файлів

```bash
curl -O https://example.com/file.tar.gz             # Завантажити файл
curl -L -o output.tar.gz https://example.com/file   # З перенаправленнями
curl -u user:pass https://api.example.com/data      # З автентифікацією
curl -H "Authorization: Bearer TOKEN" https://api.example.com

wget https://example.com/file.tar.gz                # Альтернатива curl
wget -q --show-progress https://example.com/file    # Тихий режим з прогресом
```

### Firewall

```bash
# UFW (Uncomplicated Firewall) — Debian/Ubuntu
ufw status
ufw allow 80/tcp
ufw allow 22/tcp
ufw deny 3306
ufw enable

# firewalld — RHEL/CentOS
firewall-cmd --state
firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

---

## 15. SSH та віддалений доступ

### Базові підключення

```bash
ssh user@hostname                        # Підключення
ssh -p 2222 user@hostname               # Нестандартний порт
ssh -i ~/.ssh/mykey.pem user@hostname   # З конкретним ключем
ssh -L 8080:localhost:80 user@server    # Тунель (Local Port Forwarding)
ssh -R 9090:localhost:3000 user@server  # Зворотній тунель
```

### Управління ключами SSH

```bash
# Генерація ключової пари
ssh-keygen -t ed25519 -C "your@email.com"    # Сучасний алгоритм
ssh-keygen -t rsa -b 4096 -C "your@email.com"

# Копіювання публічного ключа на сервер
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/mykey.pub user@hostname

# Файли
~/.ssh/id_ed25519        # Приватний ключ (права: 600)
~/.ssh/id_ed25519.pub    # Публічний ключ
~/.ssh/authorized_keys   # Дозволені ключі на сервері (права: 600)
~/.ssh/known_hosts       # Відомі хости
```

### SSH Config (`~/.ssh/config`)

```
Host myserver
    HostName 192.168.1.100
    User admin
    Port 2222
    IdentityFile ~/.ssh/mykey.pem
    ServerAliveInterval 60

Host prod-*
    User ubuntu
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking yes
```

```bash
# Тепер можна підключитись просто:
ssh myserver
ssh prod-web1
```

### Передача файлів

```bash
# SCP (Secure Copy)
scp file.txt user@host:/remote/path/
scp -r /local/dir/ user@host:/remote/
scp user@host:/remote/file.txt /local/

# rsync — ефективна синхронізація
rsync -avz /local/dir/ user@host:/remote/dir/
rsync -avz --delete /src/ user@host:/dst/    # Дзеркальна синхронізація
rsync -avz --progress /src/ /dst/            # З прогресом
```

---

## 16. Troubleshooting: підхід та інструменти

### Методологія

1. **Визначити симптом** — що саме не працює? Яка помилка?
2. **Зрозуміти архітектуру** — де в ланцюжку може бути проблема?
3. **Перевірити логи** — `/var/log/`, `journalctl`
4. **Звузити проблему** — ізолювати компоненти
5. **Перевірити базові речі** — диск, пам'ять, мережа, права доступу
6. **Зафіксувати рішення** — задокументувати причину і виправлення

### Ресурси системи

```bash
# CPU та пам'ять
free -h                         # Використання пам'яті
vmstat 1 5                      # Статистика за 5 секунд
sar -u 1 5                      # CPU з утилітою sar

# Диск
df -h                           # Місце на дисках
du -sh /var/log/ | sort -h      # Великі директорії
iostat -xz 1                    # I/O статистика

# Мережа
iftop                           # Мережевий трафік в реальному часі
nethogs                         # Трафік по процесах
```

### Корисні команди для дебагу

```bash
# Перевірка прав та власника
ls -la /path/to/file
namei -l /path/to/file         # Права на кожному рівні шляху

# Перевірка конфігурацій
nginx -t                        # Перевірити конфіг Nginx
apache2ctl configtest           # Apache

# Відлагодження мережі
tcpdump -i eth0 port 80         # Захоплення пакетів
tcpdump -i any host 10.0.0.1   # Трафік до/від хоста

# Strace — системні виклики процесу
strace -p PID                   # Приєднатись до процесу
strace -e open,read command     # Відстежити конкретні виклики

# lsof — відкриті файли та з'єднання
lsof -p PID                     # Файли відкриті процесом
lsof /var/log/app.log           # Хто використовує файл
```

### Найпоширеніші проблеми

| Проблема | Що перевірити |
|---|---|
| Сервіс не стартує | `journalctl -u service -n 50` |
| Немає місця на диску | `df -h`, `du -sh /* 2>/dev/null \| sort -h` |
| Вичерпано inode | `df -i` |
| Не вдається підключитись | `ss -tulpn`, firewall, SELinux |
| Повільна система | `top`, `iostat`, `free -h` |
| Помилка "Permission denied" | `ls -la`, `namei -l`, `id` |
| OOM (Out of Memory) | `dmesg \| grep -i oom` |

---

## Шпаргалки та швидкі рецепти

### 10 команд які рятують найчастіше

```bash
# 1. Знайти що займає місце
du -sh /* 2>/dev/null | sort -h | tail -20

# 2. Топ-10 IP за кількістю запитів у nginx
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 3. Знайти всі файли більше 100МБ
find / -type f -size +100M 2>/dev/null | xargs ls -lh

# 4. Хто заходив в систему сьогодні
last | grep "$(date +%b\ %e)"

# 5. Всі запущені веб-сервіси
ss -tulpn | grep -E ':(80|443|8080|8443)'

# 6. Кількість з'єднань по статусах TCP
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# 7. Знайти процес на порту
lsof -ti :8080 | xargs kill -9

# 8. Поточні помилки в системних логах
journalctl -p err --since "1 hour ago" --no-pager

# 9. Список всіх крон-задач у системі
for user in $(cut -f1 -d: /etc/passwd); do crontab -u $user -l 2>/dev/null; done

# 10. Перевірити SSL-сертифікат
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>/dev/null | \
    openssl x509 -noout -dates
```

---

*Довідник базується на матеріалах відеокурсу DevOps Linux та доповнений практичними рецептами для повсякденної роботи.*
