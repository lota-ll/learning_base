# 🖥️ Virtualization & Containerization для DevOps — Повний довідник

> Довідник охоплює ключові теми: типи та архітектуру віртуалізації, гіпервізори, IaC для VM, контейнеризацію, Docker, Dockerfile, реєстри образів та оркестрацію.

---

## Зміст

1. [Вступ до віртуалізації](#1-вступ-до-віртуалізації)
2. [Типи віртуалізації](#2-типи-віртуалізації)
3. [Гіпервізори](#3-гіпервізори)
4. [KVM та внутрішня архітектура](#4-kvm-та-внутрішня-архітектура)
5. [Use Cases віртуалізації](#5-use-cases-віртуалізації)
6. [Переваги та недоліки віртуалізації](#6-переваги-та-недоліки-віртуалізації)
7. [IaC та автоматизація VM](#7-iac-та-автоматизація-vm)
8. [Моніторинг та безпека VM](#8-моніторинг-та-безпека-vm)
9. [Хмара та віртуалізація](#9-хмара-та-віртуалізація)
10. [Вступ до контейнеризації](#10-вступ-до-контейнеризації)
11. [Архітектура контейнерів](#11-архітектура-контейнерів)
12. [Docker: основні команди](#12-docker-основні-команди)
13. [Dockerfile: написання та best practices](#13-dockerfile-написання-та-best-practices)
14. [Реєстри образів](#14-реєстри-образів)
15. [Docker Compose](#15-docker-compose)
16. [Оркестрація контейнерів](#16-оркестрація-контейнерів)
17. [Контейнери vs Віртуальні машини](#17-контейнери-vs-віртуальні-машини)
18. [Контейнери в SDLC та CI/CD](#18-контейнери-в-sdlc-та-cicd)
19. [Безпека контейнерів](#19-безпека-контейнерів)
20. [Шпаргалка: швидкі команди](#20-шпаргалка-швидкі-команди)

---

## 1. Вступ до віртуалізації

**Віртуалізація** — це технологія створення віртуальних (а не фізичних) версій комп'ютерних ресурсів: апаратних платформ, сховищ даних та мережевих ресурсів.

### Чому це важливо для DevOps?

Віртуалізація — це фундамент сучасної IT-інфраструктури. Вона дозволяє:
- Запускати кілька ізольованих середовищ на одному фізичному сервері
- Швидко розгортати та відтворювати конфігурації
- Ефективніше використовувати апаратні ресурси
- Забезпечити disaster recovery та масштабування

### Ключові поняття

| Термін | Визначення |
|---|---|
| **Host OS** | Операційна система фізичного сервера |
| **Guest OS** | ОС, що працює всередині VM |
| **Hypervisor** | ПЗ, що створює VM та управляє ними |
| **VM (Virtual Machine)** | Ізольоване програмне середовище, що емулює фізичний комп'ютер |
| **Snapshot** | Миттєвий знімок стану VM для резервування |
| **Template/Image** | Шаблон для швидкого розгортання нових VM |

---

## 2. Типи віртуалізації

### Серверна (Server Virtualization)
Найпоширеніший тип. Один фізичний сервер ділиться на кілька ізольованих VM. Кожна VM має свою Guest OS, CPU, RAM, диск.

**Інструменти:** VMware vSphere, KVM, Hyper-V, Xen

### Мережева (Network Virtualization)
Відокремлює мережеві функції від апаратного забезпечення. Дозволяє створювати програмно-керовані мережі.

**Технології:** SDN (Software-Defined Networking), NFV (Network Function Virtualization), Open vSwitch

**Переваги:**
- Динамічна маршрутизація залежно від потреб застосунку
- Зниження споживання енергії (з 40A до 20A у прикладі з презентації)
- Централізоване управління

### Сховища даних (Storage Virtualization)
Об'єднання сховищ з кількох фізичних пристроїв у єдиний пул. Elastic storage — автоматично розширюється під потреби.

**Технології:** SAN (Storage Area Network), NAS, vSAN

### Десктопна (Desktop Virtualization / VDI)
Централізоване управління робочими столами в дата-центрах. Користувачі підключаються до VM з будь-якого пристрою.

**Переваги:** Безпека, єдине середовище, зручність для віддаленої роботи

### Застосункова (Application Virtualization)
Ізолює програми від ОС та інших застосунків. Дозволяє запускати кілька версій одного ПЗ одночасно.

**Приклади:** Citrix, Microsoft App-V

---

## 3. Гіпервізори

Гіпервізор (VMM — Virtual Machine Monitor) — програмний рівень між апаратним забезпеченням та гостьовими ОС.

### Type 1 — Bare Metal (на голому залізі)

Працює **безпосередньо на апаратному забезпеченні** без Host OS. Максимальна продуктивність і безпека.

```
┌─────────────┬─────────────┬─────────────┐
│   Guest OS  │   Guest OS  │   Guest OS  │
│    (VM 1)   │    (VM 2)   │    (VM 3)   │
├─────────────┴─────────────┴─────────────┤
│              HYPERVISOR (Type 1)         │
├─────────────────────────────────────────┤
│           HOST HARDWARE                  │
└─────────────────────────────────────────┘
```

**Приклади:** VMware vSphere/ESXi, Microsoft Hyper-V, Oracle VM Server, Xen, KVM

**Використання:** Дата-центри, enterprise середовища, хмарні провайдери

### Type 2 — Hosted (поверх ОС)

Запускається **як додаток** поверх стандартної ОС. Трохи нижча продуктивність через додатковий рівень.

```
┌─────────────┬─────────────┐
│   Guest OS  │   Guest OS  │
│    (VM 1)   │    (VM 2)   │
├─────────────┴─────────────┤
│     HYPERVISOR (Type 2)   │
├───────────────────────────┤
│        HOST OS            │
├───────────────────────────┤
│      HOST HARDWARE        │
└───────────────────────────┘
```

**Приклади:** VMware Workstation, Oracle VirtualBox, Parallels Desktop

**Використання:** Розробка, тестування, навчання, desktop virtualization

### Порівняння

| Критерій | Type 1 | Type 2 |
|---|---|---|
| Продуктивність | Висока | Середня |
| Складність налаштування | Висока | Низька |
| Ізоляція | Повна | Часткова |
| Використання | Production | Dev/Test |
| Приклади | ESXi, Hyper-V | VirtualBox, VMware WS |

---

## 4. KVM та внутрішня архітектура

### KVM (Kernel-based Virtual Machine)

KVM — повнофункціональне рішення для віртуалізації, вбудоване в ядро Linux. Підтримує архітектури AMD64 (AMD-V) та Intel 64 (Intel VT).

**Стек технологій KVM:**

```
┌────────────────────────────────────────────┐
│              Guest OS (додатки)             │
├────────────────────────────────────────────┤
│         QEMU (емуляція пристроїв)           │
├────────────────────────────────────────────┤
│       KVM Kernel Module (/dev/kvm)          │
├────────────────────────────────────────────┤
│    Hardware-Assisted Virtualization         │
│         Intel-VT       AMD-V               │
├────────────────────────────────────────────┤
│              HARDWARE                       │
└────────────────────────────────────────────┘
```

**Компоненти:**
- **QEMU** — емулятор, що обробляє пристрої (диски, мережа, відео)
- **KVM kernel module** — забезпечує апаратну акселерацію
- **libvirt** — API-бібліотека та daemon для управління VM
- **virsh** — CLI для управління через libvirt
- **virt-manager** — GUI для управління VM

### Управління VM через libvirt/virsh

```bash
# Перевірити чи підтримується апаратна віртуалізація
egrep -c '(vmx|svm)' /proc/cpuinfo   # > 0 означає підтримку

# Встановлення KVM та інструментів (Ubuntu)
apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virt-manager

# Перевірка статусу служби
systemctl status libvirtd

# Основні команди virsh
virsh list                    # Запущені VM
virsh list --all              # Всі VM (включно зупинені)
virsh start myvm              # Запустити VM
virsh shutdown myvm           # М'яке вимкнення
virsh destroy myvm            # Примусове вимкнення
virsh suspend myvm            # Призупинити VM
virsh resume myvm             # Відновити VM
virsh snapshot-create-as myvm snap1  # Зробити snapshot
virsh snapshot-revert myvm snap1     # Відновити зі snapshot
virsh dumpxml myvm            # Вивести XML-конфігурацію VM
virsh dominfo myvm            # Інформація про VM
virsh vcpuinfo myvm           # Інформація про vCPU

# Створення VM через virt-install
virt-install \
  --name myvm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/myvm.qcow2,size=20 \
  --os-variant ubuntu22.04 \
  --network bridge=virbr0 \
  --graphics none \
  --cdrom /path/to/ubuntu.iso \
  --console pty,target_type=serial
```

### Networking в KVM

```bash
# Переглянути мережі
virsh net-list --all

# Режими мережевих адаптерів
# NAT (за замовч.) — VM має доступ в інтернет, але недоступна ззовні
# Bridge — VM у тій самій мережі, що і host
# Isolated — VM ізольовані, без доступу назовні

# Bridged networking (для production)
ip link add name br0 type bridge
ip link set eth0 master br0
ip link set br0 up
```

---

## 5. Use Cases віртуалізації

### Development Environments

VM дозволяють відтворити production-середовище без обмежень апаратного забезпечення. Кожен розробник має однакове, ізольоване оточення.

**Інструменти:** VirtualBox, KVM, VMware Workstation

```
Host Machine
├── VM: Ubuntu 22.04 (Production-like)
│   ├── Python 3.9
│   ├── PostgreSQL 14
│   └── Nginx 1.22
├── VM: Windows Server (Legacy apps)
└── VM: CentOS 7 (Testing)
```

### CI/CD Pipelines

VM забезпечують стандартизоване середовище виконання для кожної збірки.

**Варіанти:**
- Self-hosted runners на VM (GitHub Actions, GitLab Runner)
- Динамічний пул runners на EC2+ASG або Google Compute Engine
- Кожен pipeline запускається в чистій VM — гарантована відтворюваність

### Тестування в паралельних середовищах

```
┌─────────────────────────────────────────────┐
│              Parallel Testing               │
├───────────────┬──────────────┬──────────────┤
│  Test Machine │ Test Machine │ Test Machine │
│      A        │      B       │      C       │
├───────────────┼──────────────┼──────────────┤
│  Ubuntu 20    │  Ubuntu 22   │  CentOS 7    │
│  Python 3.8   │  Python 3.10 │  Python 3.7  │
└───────────────┴──────────────┴──────────────┘
```

### Disaster Recovery

```bash
# Disk backup — резервна копія диска VM
dd if=/dev/sda | gzip > /backup/vm-disk-$(date +%Y%m%d).img.gz

# VM Snapshot (libvirt)
virsh snapshot-create-as prod-db \
  "before-migration-$(date +%Y%m%d)" \
  "Snapshot before DB migration"

# Відновлення
virsh snapshot-revert prod-db "before-migration-20240115"
```

### Load Balancing та Scalability

- **Scale Up** — збільшення ресурсів однієї VM (CPU, RAM)
- **Scale Out** — додавання нових VM до пулу за допомогою load balancer

```
Internet
    │
    ▼
Load Balancer (Nginx/HAProxy)
    │
    ├──► VM 1 (Web Server)
    ├──► VM 2 (Web Server)
    └──► VM 3 (Web Server)
```

### Server Virtualization та Мікросервіси

| Підхід | Монолітний | Мікросервісний |
|---|---|---|
| Деплоймент | Весь застосунок | Кожен сервіс окремо |
| Масштабування | Клонування всього | Масштабування окремих сервісів |
| Ізоляція | Спільна VM | Окремі VM/контейнери |
| Відмовостійкість | Одна точка відмови | Ізольовані збої |

---

## 6. Переваги та недоліки віртуалізації

### Переваги ✅

| Перевага | Опис |
|---|---|
| **Економія коштів** | Менше фізичного обладнання, менше енергоспоживання |
| **Гнучкість** | Кілька ОС та застосунків на одному сервері |
| **Масштабованість** | Легко додавати/видаляти ресурси |
| **Agility** | Швидке розгортання нових середовищ |
| **Disaster Recovery** | Snapshots, резервні копії, швидке відновлення |
| **Централізація** | Спрощення управління інфраструктурою |

### Недоліки ⚠️

| Недолік | Опис |
|---|---|
| **Складність** | Потребує знань для налаштування і управління |
| **Overhead** | Гіпервізор споживає частину ресурсів (CPU/RAM) |
| **Безпека** | VM escape attacks, inter-VM attacks, вразливості гіпервізора |
| **Ліцензування** | Складна модель ліцензій у VM-середовищах |
| **Resource contention** | VM можуть конкурувати за ресурси |

---

## 7. IaC та автоматизація VM

### Vagrant — локальна автоматизація

Vagrant дозволяє описати VM у коді (Vagrantfile) та відтворити її одною командою.

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  
  # Мережа
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  
  # Ресурси
  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 2
    v.name = "dev-server"
  end
  
  # Спільні директорії
  config.vm.synced_folder "./app", "/var/www/app"
  
  # Provisioning скриптом
  config.vm.provision "shell", path: "provision/setup.sh"
  
  # Provisioning Ansible
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```

```bash
vagrant up           # Створити та запустити VM
vagrant ssh          # Підключитись до VM
vagrant halt         # Зупинити VM
vagrant destroy      # Видалити VM
vagrant snapshot save mysnap  # Зробити snapshot
vagrant status       # Статус VM
```

### Terraform — IaC для хмари та on-premise

```hcl
# Terraform + libvirt (on-premise KVM)
terraform {
  required_providers {
    libvirt = {
      source = "dmacvicar/libvirt"
    }
  }
}

provider "libvirt" {
  uri = "qemu:///system"
}

# Базовий образ
resource "libvirt_volume" "ubuntu_base" {
  name   = "ubuntu-base"
  source = "https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
  format = "qcow2"
}

# Диск VM
resource "libvirt_volume" "vm_disk" {
  name           = "webserver-disk"
  base_volume_id = libvirt_volume.ubuntu_base.id
  size           = 21474836480  # 20GB
}

# Cloud-init конфіг
resource "libvirt_cloudinit_disk" "cloud_init" {
  name      = "cloud-init.iso"
  user_data = <<-EOF
    #cloud-config
    hostname: webserver
    users:
      - name: ubuntu
        ssh_authorized_keys:
          - ${file("~/.ssh/id_rsa.pub")}
        sudo: ALL=(ALL) NOPASSWD:ALL
  EOF
}

# VM
resource "libvirt_domain" "webserver" {
  name   = "webserver"
  memory = "2048"
  vcpu   = 2

  cloudinit = libvirt_cloudinit_disk.cloud_init.id

  disk {
    volume_id = libvirt_volume.vm_disk.id
  }

  network_interface {
    network_name   = "default"
    wait_for_lease = true
  }
}
```

```bash
terraform init      # Ініціалізація провайдерів
terraform plan      # Перегляд змін
terraform apply     # Застосування
terraform destroy   # Видалення ресурсів
```

### Terraform для AWS EC2

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "WebServer"
    Environment = "Production"
  }

  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = "my-key-pair"

  user_data = <<-EOF
    #!/bin/bash
    apt update -y
    apt install -y nginx
    systemctl start nginx
  EOF
}
```

### Ansible — конфігурація VM

```yaml
# playbook.yml
---
- name: Configure web server
  hosts: webservers
  become: true
  
  vars:
    app_port: 8080
    
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes
        
    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-enabled/app.conf
      notify: Reload nginx
      
    - name: Ensure nginx is running
      systemd:
        name: nginx
        state: started
        enabled: yes
        
  handlers:
    - name: Reload nginx
      systemd:
        name: nginx
        state: reloaded
```

---

## 8. Моніторинг та безпека VM

### Ключові метрики для моніторингу VM

**Продуктивність:**
- CPU Utilization (%) — завантаженість процесора
- Memory Utilization (%) — використання RAM
- Disk Read/Write IOPS — операції вводу/виводу
- Disk Latency (ms) — затримка дискових операцій
- Network Throughput (Mbps) — пропускна здатність мережі
- CPU Ready Time — час очікування CPU (ознака contention)

**Надійність:**
- Uptime — час безперервної роботи
- Error Rate — кількість системних помилок
- Failed Login Attempts — невдалі спроби авторизації

**Ефективність:**
- I/O Wait — CPU чекає на диск (ознака проблем зі сховищем)
- Balloon Memory (VMware) — тиск на пам'ять

### Інструменти моніторингу

```
┌─────────────────────────────────────────────────────┐
│                  Monitoring Stack                    │
├──────────────┬──────────────┬───────────────────────┤
│  Prometheus  │   Grafana    │    Alertmanager        │
│  (збір метрик)│ (візуалізація)│  (сповіщення)        │
├──────────────┴──────────────┴───────────────────────┤
│         node_exporter  libvirt_exporter              │
│              (агенти на VM)                          │
└─────────────────────────────────────────────────────┘
```

**Інші інструменти:** Zabbix, Nagios, Datadog, Netdata

```bash
# Встановлення node_exporter для моніторингу VM
wget https://github.com/prometheus/node_exporter/releases/latest/...
./node_exporter --web.listen-address=":9100"

# Базова перевірка стану VM вручну
top                          # CPU та пам'ять
iostat -xz 1                 # Диск I/O
vmstat 1 5                   # Статистика системи
sar -u 1 5                   # CPU за часом
free -h                      # Пам'ять
df -h                        # Дисковий простір
```

### Загрози безпеки в VM-середовищах

**VM Escape Attack** — атакуючий запускає код у VM, що дозволяє вийти за межі ізоляції та взаємодіяти безпосередньо з гіпервізором.

**Inter-VM Attack** — атака з однієї VM на іншу через спільну пам'ять, мережу або інші ресурси без компрометації гіпервізора.

**Hypervisor Vulnerability** — вразливості у самому гіпервізорі можуть скомпрометувати всі VM на хості.

### Best Practices безпеки

```bash
# 1. Регулярне оновлення гіпервізора
apt update && apt upgrade -y qemu-kvm libvirt-daemon

# 2. Ізоляція мереж VM (VLAN)
virsh net-define isolated-network.xml

# 3. Обмеження ресурсів (щоб одна VM не "з'їла" всі ресурси)
virsh schedinfo myvm --set vcpu_quota=50000  # 50% CPU

# 4. Аудит доступу
auditd  # моніторинг системних викликів

# 5. Шифрування дисків VM
# LUKS для шифрування disk images

# 6. Відключення непотрібних пристроїв у VM
# Видалити USB, serial ports з XML конфігурації

# 7. IDS/IPS
# Впровадити Snort або Suricata для виявлення атак
```

---

## 9. Хмара та віртуалізація

### On-Premise vs Cloud

| Критерій | On-Premise | Cloud |
|---|---|---|
| Вартість заліза | Висока | Низька (pay-as-you-go) |
| Обслуговування | Власна команда | Провайдер |
| Масштабування | Обмежене | Необмежене (Auto Scaling) |
| Безпека | Повний контроль | Спільна відповідальність |
| DR | Складніше | Простіше (багато регіонів) |

### AWS EC2 — хмарна VM

```bash
# AWS CLI — управління EC2
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --key-name my-keypair \
  --security-group-ids sg-12345678 \
  --count 1

# Auto Scaling Group для автоматичного масштабування
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateName=my-template \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3

# Зупинити/запустити EC2
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

### Гібридне середовище

**Hybrid Cloud** поєднує on-premise інфраструктуру з хмарою:
- On-premise: основне навантаження, чутливі дані
- Cloud: burst capacity, DR, dev/test середовища

```
On-Premise DC ─── AWS Direct Connect ─── AWS Cloud
  (VMware vSphere)                        (EC2, RDS, S3)
       │                                        │
  Sensitive Data                         Burst Workloads
  Core Services                          DR Failover
```

### Serverless Computing

Serverless — крайній ступінь абстракції від інфраструктури. Розробник пише функції, провайдер управляє VM.

**AWS Lambda:**
```python
import json

def handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Hello from Lambda!'})
    }
```

**Порівняння підходів:**

```
Traditional → VM → Containers → Serverless
[Full control] ←────────────→ [Abstraction]
[High overhead] ←───────────→ [Low overhead]
```

---

## 10. Вступ до контейнеризації

### Проблема яку вирішують контейнери

Класична проблема DevOps: **"It works on my machine"** — різні середовища (dev, staging, production) мають різні версії залежностей, ОС, конфігурації.

**"Matrix from Hell"** — традиційна проблема сумісності:
```
         MySQL   Node.js   Angular   React
          │        │          │        │
          ▼        ▼          ▼        ▼
       Dependencies  &  Libraries
              ↓
         Operating System
              ↓
           Hardware
```

Кожна залежність потребує своїх версій бібліотек — виникають конфлікти.

### Рішення: контейнери

**Контейнер** — легкий, автономний, виконуваний пакет ПЗ, що включає застосунок з усіма його залежностями: код, runtime, системні бібліотеки, конфігурацію.

```
┌─────────────────────────────────────────────┐
│  MySQL Container  │  Node Container  │ React │
│  [App + Libs + Dep│ [App + Libs + Dep│       │
├───────────────────┴──────────────────┴───────┤
│              Container Engine (Docker)        │
├──────────────────────────────────────────────┤
│                 Operating System              │
├──────────────────────────────────────────────┤
│                   Hardware                    │
└──────────────────────────────────────────────┘
```

### Переваги контейнерів

| Проблема | Традиційний підхід | Контейнери |
|---|---|---|
| Встановлення | Час + багато команд | Один `docker run` |
| Залежності | "Dependency Hell" | Ізольовані в образі |
| Пакування | Складно | Простий Dockerfile |
| Dev→Stage→Prod | Дублювання зусиль | Один образ |
| Ізоляція | Неможлива | Вбудована |
| Масштабування | Складно | Легко |

---

## 11. Архітектура контейнерів

### Образ (Image) vs Контейнер (Container)

```
Dockerfile ──(build)──► Docker Image ──(run)──► Container
(Рецепт)               (Шаблон/Blueprint)      (Запущений екземпляр)
```

- **Image** — незмінний (immutable) шаблон з усіма залежностями. Зберігається в реєстрі.
- **Container** — запущений екземпляр образу. Є тонкий read-write шар зверху.

### Шарова архітектура образів

```
┌─────────────────────────────────┐ ◄── Container Layer (R/W)
│  Layer 4: CMD java -jar app.jar │
├─────────────────────────────────┤
│  Layer 3: COPY app.jar /app/    │
├─────────────────────────────────┤  Image Layers (Read-Only)
│  Layer 2: RUN apt install openjdk│
├─────────────────────────────────┤
│  Layer 1: FROM ubuntu:22.04     │
└─────────────────────────────────┘
```

**Переваги шарів:**
- Повторне використання спільних шарів між образами
- Швидке завантаження (тільки нові шари)
- Ефективне зберігання

### Архітектура Docker в Linux

```
Docker Client → REST API → Docker Engine (dockerd)
                               │
                          libcontainerd
                               │
                     containerd + runc
                               │
              ┌────────────────┼────────────────┐
         Namespaces      Control Groups    Layer FS
         (pid, net,      (cgroups: CPU,   (OverlayFS,
          ipc, mnt, uts)  Memory, I/O)    AUFS, btrfs)
```

**Ключові механізми Linux:**
- **Namespaces** — ізоляція процесів, мережі, файлової системи
- **cgroups** — обмеження ресурсів (CPU, RAM, I/O)
- **OverlayFS** — шарова файлова система

---

## 12. Docker: основні команди

### Встановлення Docker

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh

# Додати користувача в групу docker (без sudo)
usermod -aG docker $USER
newgrp docker

# Перевірити
docker --version
docker info
```

### Управління образами

```bash
# Завантажити образ з Docker Hub
docker pull nginx:1.25
docker pull ubuntu:22.04
docker pull python:3.11-slim

# Переглянути локальні образи
docker images
docker image ls

# Видалити образ
docker rmi nginx:1.25
docker image prune          # Видалити всі dangling образи
docker image prune -a       # Видалити всі невикористані образи

# Переглянути шари образу
docker history nginx:1.25

# Зберегти образ у файл / завантажити з файлу
docker save myapp:1.0 | gzip > myapp.tar.gz
docker load < myapp.tar.gz

# Теги
docker tag myapp:latest myapp:1.0.0
docker tag myapp:1.0.0 registry.example.com/myapp:1.0.0
```

### Запуск контейнерів

```bash
# Базовий запуск
docker run nginx

# Інтерактивний режим
docker run -it ubuntu:22.04 bash

# Фоновий режим (detached)
docker run -d nginx

# З налаштуваннями
docker run -d \
  --name myapp \
  -p 8080:80 \                      # порт хоста:порт контейнера
  -v /host/path:/container/path \   # volume mount
  -e DATABASE_URL=postgres://... \  # змінна середовища
  --memory="512m" \                 # обмеження пам'яті
  --cpus="0.5" \                    # обмеження CPU
  --restart unless-stopped \        # политика перезапуску
  --network mynetwork \             # підключити до мережі
  nginx:1.25

# Тимчасовий контейнер (видаляється після завершення)
docker run --rm -it python:3.11 python3

# Виконати команду в запущеному контейнері
docker exec -it myapp bash
docker exec myapp ls /var/www

# Переглянути логи
docker logs myapp
docker logs -f myapp              # follow (live)
docker logs --tail 100 myapp      # останні 100 рядків
docker logs --since 1h myapp      # за останню годину
```

### Управління контейнерами

```bash
# Список контейнерів
docker ps                         # тільки запущені
docker ps -a                      # всі
docker ps -q                      # тільки ID

# Зупинити / запустити / перезапустити
docker stop myapp
docker start myapp
docker restart myapp
docker kill myapp                 # примусово (SIGKILL)

# Видалити контейнер
docker rm myapp
docker rm -f myapp               # примусово запущений
docker container prune           # видалити всі зупинені

# Інформація про контейнер
docker inspect myapp             # детальна JSON-інформація
docker stats                     # ресурси всіх контейнерів в реальному часі
docker top myapp                 # процеси всередині контейнера

# Копіювання файлів
docker cp myapp:/var/log/app.log ./
docker cp ./config.yaml myapp:/etc/app/

# Зупинити всі контейнери
docker stop $(docker ps -q)

# Видалити всі зупинені контейнери, невикористані образи, мережі
docker system prune -a
```

### Мережі Docker

```bash
# Переглянути мережі
docker network ls

# Типи мереж
# bridge  — за замовч., контейнери спілкуються через bridge
# host    — контейнер використовує мережу хоста напряму
# none    — без мережевого доступу
# overlay — для Docker Swarm (multi-host)

# Створити мережу
docker network create mynetwork
docker network create --driver bridge --subnet 172.20.0.0/16 mynetwork

# Підключити контейнер до мережі
docker network connect mynetwork myapp

# Контейнери в одній мережі можуть звертатись один до одного за ім'ям
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet \
  -e DB_HOST=db myapp  # 'db' — ім'я контейнера, доступне як hostname
```

### Volumes — постійне зберігання даних

```bash
# Named volumes (рекомендовано для даних)
docker volume create mydata
docker run -d -v mydata:/var/lib/postgresql/data postgres

# Bind mounts (монтування директорії хоста)
docker run -d -v /host/path:/container/path nginx

# tmpfs (в пам'яті, тимчасово)
docker run --tmpfs /tmp myapp

# Управління volumes
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune           # видалити невикористані
```

---

## 13. Dockerfile: написання та best practices

### Синтаксис інструкцій Dockerfile

| Інструкція | Призначення |
|---|---|
| `FROM` | Базовий образ |
| `WORKDIR` | Робоча директорія всередині контейнера |
| `COPY` | Копіювання файлів з хоста в образ |
| `ADD` | Як COPY, але підтримує URL та tar-архіви |
| `RUN` | Виконання команди під час збірки |
| `ENV` | Встановлення змінних середовища |
| `ARG` | Аргументи збірки (лише під час build) |
| `EXPOSE` | Документування порту (не відкриває!) |
| `CMD` | Команда за замовчуванням при запуску |
| `ENTRYPOINT` | Незмінна команда запуску |
| `VOLUME` | Оголошення точки монтування |
| `USER` | Від якого користувача запускати процес |
| `LABEL` | Метадані образу |
| `HEALTHCHECK` | Перевірка здоров'я контейнера |

### Приклад: Python/Django застосунок

```dockerfile
# --- Базовий образ ---
FROM python:3.11-slim AS base

# Метадані
LABEL maintainer="devops@company.com"
LABEL version="1.0"

# Змінні середовища
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_HOME=/app

# Робоча директорія
WORKDIR $APP_HOME

# --- Встановлення залежностей ---
FROM base AS dependencies

# Системні пакети (окремим шаром для кешування)
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Python залежності (копіюємо тільки requirements для кешування)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# --- Фінальний образ ---
FROM dependencies AS final

# Код застосунку
COPY . .

# Не root користувач (безпека!)
RUN useradd -m appuser && chown -R appuser:appuser $APP_HOME
USER appuser

# Документація порту
EXPOSE 8001

# Перевірка здоров'я
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8001/health/ || exit 1

# Точка входу
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver", "0.0.0.0:8001"]
```

### Multi-stage build — зменшення розміру образу

```dockerfile
# STAGE 1: Збірка (builder)
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# STAGE 2: Фінальний образ (тільки production artifacts)
FROM nginx:alpine AS final
# Копіюємо тільки зібраний артефакт — node_modules не включаються!
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Результат: образ у 10-100x менший, ніж з node_modules
```

### .dockerignore

```dockerignore
# .dockerignore — виключити з контексту збірки
node_modules/
.git/
.env
*.log
dist/
__pycache__/
*.pyc
.pytest_cache/
Dockerfile*
docker-compose*
README.md
```

### Best Practices Dockerfile

```dockerfile
# ✅ ПРАВИЛЬНО: Групувати RUN команди та очищати кеш
RUN apt-get update && apt-get install -y \
    nginx \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ❌ НЕПРАВИЛЬНО: Окремі RUN — більше шарів, більший образ
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y curl

# ✅ Копіювати залежності окремо від коду (кешування шарів)
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .                    # Зміна коду не скидає кеш pip

# ✅ Використовувати мінімальні базові образи
FROM python:3.11-slim       # slim — без зайвих пакетів
FROM python:3.11-alpine     # ще менший, але можуть бути проблеми з native libs

# ✅ Ніколи не запускати від root
USER appuser

# ✅ Фіксувати версії базових образів
FROM ubuntu:22.04           # конкретна версія, не 'latest'

# ✅ Використовувати HEALTHCHECK
HEALTHCHECK --interval=30s CMD wget -q -O- http://localhost/health || exit 1

# ✅ Secrets НЕ в Dockerfile — передавати через ENV або secrets manager
# ❌ НІКОЛИ:
ENV DB_PASSWORD=mysecret   # буде видно в docker history!
```

### Збірка образу

```bash
# Базова збірка
docker build -t myapp:1.0 .

# З вказаним Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# З build arguments
docker build --build-arg APP_VERSION=1.2.3 -t myapp:1.2.3 .

# З platform (для ARM/AMD64)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 .

# Без кешу
docker build --no-cache -t myapp:1.0 .

# Переглянути розмір шарів
docker history myapp:1.0
```

---

## 14. Реєстри образів

### Типи реєстрів

| Реєстр | Тип | Опис |
|---|---|---|
| Docker Hub | Публічний | Найбільший публічний реєстр |
| GitHub Container Registry (GHCR) | Публічний/Приватний | Інтеграція з GitHub |
| Amazon ECR | Приватний (AWS) | Native інтеграція з AWS |
| Google Artifact Registry | Приватний (GCP) | Native інтеграція з GCP |
| Azure Container Registry | Приватний (Azure) | Native інтеграція з Azure |
| Harbor | Self-hosted | Open-source enterprise реєстр |
| Quay.io | Публічний/Приватний | Red Hat реєстр |

### Робота з реєстрами

```bash
# Авторизація
docker login                              # Docker Hub
docker login registry.example.com        # Приватний реєстр
docker login ghcr.io                      # GitHub Container Registry
aws ecr get-login-password --region us-east-1 | docker login --username AWS \
    --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Push образу
docker tag myapp:1.0 username/myapp:1.0
docker push username/myapp:1.0

# Семантичне версіонування тегів
docker tag myapp:latest myapp:2.1.3
docker tag myapp:latest myapp:2.1
docker tag myapp:latest myapp:2

# Pull образу
docker pull registry/image:tag
docker pull username/myapp:1.0

# Видалити тег з реєстру
docker rmi username/myapp:old-tag
```

### Версіонування та rollback

```
Registry Repository: myapp
├── latest     → вказує на 2.1.3
├── 2          → вказує на 2.1.3
├── 2.1        → вказує на 2.1.3
├── 2.1.3      → конкретний образ (immutable)
├── 2.0.0      → попередня версія
└── 1.5.0      → стара версія

# Rollback — просто запустити попередній тег
docker run myapp:2.0.0
```

---

## 15. Docker Compose

Docker Compose дозволяє описати та запустити **multi-container** застосунки одним файлом.

### docker-compose.yml

```yaml
version: '3.9'

services:
  # Веб-застосунок
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:latest
    container_name: myapp-web
    ports:
      - "8080:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379/0
      - DEBUG=false
    volumes:
      - ./static:/app/static
      - media_data:/app/media
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped
    networks:
      - backend
      - frontend

  # База даних
  db:
    image: postgres:15
    container_name: myapp-db
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

  # Redis кеш
  cache:
    image: redis:7-alpine
    container_name: myapp-redis
    volumes:
      - redis_data:/data
    restart: unless-stopped
    networks:
      - backend

  # Nginx reverse proxy
  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./static:/var/www/static:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - web
    restart: unless-stopped
    networks:
      - frontend

volumes:
  postgres_data:
  redis_data:
  media_data:

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

### Команди Docker Compose

```bash
docker compose up -d          # Запустити всі сервіси у фоні
docker compose down           # Зупинити та видалити контейнери
docker compose down -v        # Також видалити volumes
docker compose ps             # Статус сервісів
docker compose logs -f web    # Логи конкретного сервісу
docker compose build          # Перезбірка образів
docker compose pull           # Оновити образи
docker compose restart web    # Перезапустити сервіс
docker compose exec web bash  # Shell в контейнер
docker compose scale web=3    # Масштабувати сервіс

# Production override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 16. Оркестрація контейнерів

Оркестрація — автоматизація розгортання, масштабування та управління контейнерами у production.

### Порівняння оркестраторів

| Оркестратор | Опис | Коли використовувати |
|---|---|---|
| **Kubernetes (K8s)** | Найпотужніший, де facto standard | Production, складні системи |
| **Docker Swarm** | Простий, вбудований у Docker | Малі проекти, простота |
| **Nomad (HashiCorp)** | Гнучкий, підтримує не лише контейнери | Різнорідні workloads |
| **AWS ECS/EKS** | Managed Kubernetes/Docker у AWS | Cloud-native на AWS |
| **Google GKE** | Managed Kubernetes у GCP | Cloud-native на GCP |

### Що автоматизує оркестратор

- Scheduling — на якому вузлі запустити контейнер
- Scaling — автоматичне масштабування за навантаженням
- Self-healing — перезапуск впалих контейнерів
- Load balancing — розподіл трафіку
- Rolling updates — оновлення без downtime
- Resource management — CPU, пам'ять
- Service discovery — контейнери знаходять один одного
- Configuration & Secrets management

### Kubernetes: основні концепції

```yaml
# Pod — мінімальна одиниця в K8s
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 8000

---
# Deployment — управляє ReplicaSet та rolling updates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"

---
# Service — мережевий доступ до Pods
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

```bash
# kubectl — CLI для K8s
kubectl get pods                      # Список подів
kubectl get deployments               # Список деплойментів
kubectl describe pod myapp-pod        # Деталі поду
kubectl logs myapp-pod                # Логи
kubectl exec -it myapp-pod -- bash    # Shell в под
kubectl apply -f deployment.yaml      # Застосувати конфіг
kubectl delete -f deployment.yaml     # Видалити ресурси
kubectl scale deployment myapp --replicas=5  # Масштабувати
kubectl rollout status deployment/myapp      # Статус rollout
kubectl rollout undo deployment/myapp        # Rollback
```

---

## 17. Контейнери vs Віртуальні машини

### Архітектурне порівняння

```
Virtual Machines              Containers
─────────────────             ──────────────────
┌──────┬──────┐               ┌───┬───┬───┬───┐
│ App1 │ App2 │               │A1 │A2 │A3 │A4 │
│Libs  │Libs  │               │Lib│Lib│Lib│Lib│
│Guest │Guest │               └───┴───┴───┴───┘
│ OS   │  OS  │               ┌────────────────┐
├──────┴──────┤               │  Docker Engine │
│  Hypervisor │               ├────────────────┤
├─────────────┤               │   Host OS      │
│   Host OS   │               ├────────────────┤
├─────────────┤               │   Hardware     │
│  Hardware   │               └────────────────┘
└─────────────┘
```

### Детальне порівняння

| Критерій | Віртуальні машини | Контейнери |
|---|---|---|
| **Ізоляція** | Повна (окрема ОС) | Часткова (shared kernel) |
| **Розмір** | Гігабайти (OS + App) | Мегабайти (тільки App) |
| **Час запуску** | Хвилини | Секунди |
| **Overhead** | Значний (Guest OS) | Мінімальний |
| **Портабельність** | Обмежена | Висока |
| **Безпека** | Сильніша ізоляція | Менша ізоляція |
| **Масштабування** | Повільніше | Швидке |
| **Управління** | Складніше | Простіше |
| **Стан** | Stateful (збережений стан) | Stateless (за замовч.) |
| **ОС всередині** | Повна Guest OS | Немає (shared kernel) |

### Коли що використовувати

**Використовуй VM, якщо:**
- Потрібна повна ізоляція (compliance, multi-tenant)
- Legacy застосунки, що вимагають специфічну ОС
- Різні ОС на одному хості (Windows + Linux)
- Максимальна безпека

**Використовуй контейнери, якщо:**
- Мікросервісна архітектура
- Швидкі deployments і масштабування
- CI/CD pipeline середовища
- Хмарні native застосунки

**Hybrid підхід (найкраще):**
```
Bare Metal / Cloud VM
    └── Docker Engine
         ├── Container: Web App
         ├── Container: API
         ├── Container: Worker
         └── Container: Redis
```

---

## 18. Контейнери в SDLC та CI/CD

### Традиційний SDLC vs Container-centric

**Традиційний:**
- Розробник пише код локально → інші налаштування на staging → prod знову інший
- Повільний feedback loop, часті "у мене працює"

**Container-centric:**
- Розробник розробляє всередині контейнера → той самий контейнер на staging та prod
- Відтворювані збірки, швидкий feedback

### CI/CD Pipeline з контейнерами

```
┌──────┐   push   ┌──────────┐  build  ┌───────────┐
│ Code │ ────────►│  CI/CD   │────────►│  Docker   │
│ Git  │          │ Pipeline │         │   Image   │
└──────┘          └──────────┘         └─────┬─────┘
                       │                     │ push
                  Run Tests            ┌─────▼─────┐
                       │               │  Registry  │
                  ✅ Pass              └─────┬─────┘
                                            │ pull
                                      ┌─────▼─────┐  ┌────────────┐
                                      │  Deploy   │  │  Deploy    │
                                      │  Staging  │  │ Production │
                                      └───────────┘  └────────────┘
```

### GitHub Actions приклад

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          ssh deploy@server "docker pull ${{ env.IMAGE_NAME }}:latest && \
            docker compose up -d --no-deps --build web"
```

### GitLab CI/CD приклад

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE

test:
  stage: test
  image: $IMAGE
  script:
    - python -m pytest tests/

deploy-staging:
  stage: deploy
  environment: staging
  script:
    - docker pull $IMAGE
    - docker tag $IMAGE myapp:staging
    - docker compose -f docker-compose.staging.yml up -d
  only:
    - develop

deploy-prod:
  stage: deploy
  environment: production
  script:
    - docker pull $IMAGE
    - docker tag $IMAGE myapp:latest
    - docker compose up -d
  only:
    - main
  when: manual  # Ручне підтвердження для prod
```

---

## 19. Безпека контейнерів

### Основні загрози

- **Image vulnerabilities** — вразливості у базових образах або залежностях
- **Container escape** — вихід з ізоляції контейнера
- **Privileged containers** — контейнери з повними правами root хоста
- **Secrets exposure** — паролі/ключі у змінних середовища або образах
- **Unrestricted network** — відсутність network policies

### Best Practices безпеки

```dockerfile
# 1. Мінімальні базові образи (менша поверхня атаки)
FROM alpine:3.18
FROM gcr.io/distroless/python3  # distroless — без shell, менше можливостей

# 2. Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 3. Read-only filesystem
# docker run --read-only myapp

# 4. Не зберігати secrets в образі
# ❌ Ніколи:
ENV DB_PASSWORD=mysecret
# ✅ Передавати через Docker secrets або env файли:
# docker run --secret db_password myapp
```

```bash
# 5. Сканування образів на вразливості
docker scout cves myapp:1.0          # Docker Scout
trivy image myapp:1.0                # Trivy (Aqua Security)
snyk container test myapp:1.0        # Snyk

# 6. Обмеження ресурсів (DoS prevention)
docker run \
  --memory="512m" \
  --cpus="1.0" \
  --pids-limit=100 \
  --security-opt no-new-privileges \
  myapp

# 7. Заборона privileged mode (якщо не потрібно)
# docker run --privileged  ← ніколи у production!

# 8. Read-only контейнер
docker run --read-only \
  --tmpfs /tmp \
  myapp

# 9. Network policies — обмежити зв'язок між контейнерами
# Через Docker networks або Kubernetes NetworkPolicy

# 10. Runtime security
# Falco — моніторинг системних викликів у runtime
# AppArmor / Seccomp profiles
docker run --security-opt seccomp=/path/to/seccomp.json myapp
```

### Секрети в контейнерах

```bash
# Docker Secrets (тільки Swarm)
echo "my_password" | docker secret create db_pass -
docker service create \
  --name myapp \
  --secret db_pass \
  myapp:1.0
# Всередині контейнера: /run/secrets/db_pass

# Environment файл (не коміть в git!)
docker run --env-file .env.prod myapp

# Kubernetes Secrets
kubectl create secret generic db-secret \
  --from-literal=password=mypassword
# Або через Vault, AWS Secrets Manager, etc.
```

---

## 20. Шпаргалка: швидкі команди

### Docker Quick Reference

```bash
# ═══ ОБРАЗИ ═══
docker pull image:tag                  # Завантажити
docker build -t name:tag .             # Збудувати
docker images                          # Список
docker rmi image:tag                   # Видалити
docker push registry/image:tag         # Відправити в реєстр
docker history image:tag               # Шари образу
docker save image | gzip > file.tar.gz # Зберегти у файл

# ═══ КОНТЕЙНЕРИ ═══
docker run -d -p 80:80 --name myapp nginx   # Запустити
docker ps                              # Запущені
docker ps -a                           # Всі
docker logs -f myapp                   # Логи live
docker exec -it myapp bash             # Shell
docker stop myapp                      # Зупинити
docker rm myapp                        # Видалити
docker inspect myapp                   # JSON деталі
docker stats                           # Ресурси live
docker cp myapp:/file ./               # Копіювати файл

# ═══ МЕРЕЖІ ═══
docker network ls                      # Список
docker network create mynet            # Створити
docker network connect mynet myapp     # Підключити

# ═══ VOLUMES ═══
docker volume ls                       # Список
docker volume create myvol             # Створити
docker volume inspect myvol            # Деталі
docker volume prune                    # Очистити невикористані

# ═══ DOCKER COMPOSE ═══
docker compose up -d                   # Запустити
docker compose down                    # Зупинити
docker compose ps                      # Статус
docker compose logs -f service         # Логи
docker compose exec service bash       # Shell
docker compose build                   # Зібрати

# ═══ СИСТЕМА ═══
docker system df                       # Використання диска
docker system prune -a                 # Очистити все
```

### Virt/KVM Quick Reference

```bash
# ═══ VM MANAGEMENT ═══
virsh list --all                       # Всі VM
virsh start vmname                     # Запустити VM
virsh shutdown vmname                  # Зупинити (soft)
virsh destroy vmname                   # Зупинити (hard)
virsh suspend vmname                   # Призупинити
virsh resume vmname                    # Відновити
virsh dominfo vmname                   # Інформація про VM
virsh dumpxml vmname                   # XML конфіг

# ═══ SNAPSHOTS ═══
virsh snapshot-create-as vm snap1      # Створити snapshot
virsh snapshot-list vm                 # Список snapshots
virsh snapshot-revert vm snap1         # Відновити
virsh snapshot-delete vm snap1         # Видалити

# ═══ STORAGE ═══
virsh vol-list default                 # Volumes в пулі
qemu-img info disk.qcow2              # Інфо про диск
qemu-img resize disk.qcow2 +10G       # Розширити диск

# ═══ NETWORK ═══
virsh net-list --all                   # Мережі
virsh net-start default                # Запустити мережу
virsh net-edit default                 # Редагувати мережу
```

### Терміни та абревіатури

| Термін | Розшифровка |
|---|---|
| VM | Virtual Machine |
| KVM | Kernel-based Virtual Machine |
| QEMU | Quick Emulator |
| VDI | Virtual Desktop Infrastructure |
| SDN | Software-Defined Networking |
| NFV | Network Function Virtualization |
| IaC | Infrastructure as Code |
| CI/CD | Continuous Integration / Continuous Delivery |
| SDLC | Software Development Life Cycle |
| ECR | Elastic Container Registry (AWS) |
| GKE | Google Kubernetes Engine |
| EKS | Elastic Kubernetes Service (AWS) |
| OCI | Open Container Initiative |

---

*Довідник базується на матеріалах курсу DevOps 01 (Artem Hrechanychenko) та доповнений практичними прикладами.*
