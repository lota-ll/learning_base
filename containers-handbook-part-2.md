# 🐳 Контейнери для DevOps — Частина 2: Advanced Practices & Deep Dive

> **Джерела:** DevOps 01 Course (Artem Hrechanychenko) — відео 17 та 18 плейлиста.  
> Відео 17: практичне докеризування + Dockerfile + Security + CI/CD.  
> Відео 18: внутрішня архітектура контейнерів, ядро Linux, Docker Engine, Networking, Storage, Monitoring.

---

## ⚠️ Нотатки актуальності (станом на квітень 2026)

Матеріали курсу записані приблизно в 2023 році. Нижче перелічено, що з того часу застаріло та що рекомендовано натомість. Відповідні місця в тексті позначені міткою `[ОНОВЛЕНО]`.

**1. Python 3.8 → EOL з жовтня 2024**
Python 3.8 офіційно завершив підтримку 7 жовтня 2024. Більше немає жодних security-патчів. У всіх нових проєктах використовуй `python:3.12-slim` або `python:3.13-slim`.

**2. Node.js 14 та 16 → давно EOL**
Node 14 завершив підтримку у квітні 2023, Node 16 — у вересні 2023. Вони мають відомі незакриті CVE. Актуальні LTS-версії: Node 22 (підтримка до квітня 2027) та Node 24 (до квітня 2028). Для нових проєктів рекомендовано `node:22-slim`.

**3. Alpine для Python — не рекомендується**
Курс подає `alpine` як однозначно "найкращий" варіант через розмір. Це поширена помилка. Alpine використовує бібліотеку `musl libc` замість стандартної `glibc`. Для Python-проєктів це означає: pip не може встановити pre-built wheels з PyPI — він компілює пакети з вихідного коду, що робить білд у 50 разів повільнішим. Також є задокументовані проблеми з DNS в Kubernetes та крашами при певних конфігураціях потоків. Для Python рекомендовано: `python:3.12-slim` (Debian slim, glibc).

**4. `version: '3'` в Docker Compose — застаріло**
Поле `version` в `docker-compose.yml` є офіційно застарілим починаючи з Docker Compose V2 (v2.25.0+, 2024). Docker Compose V2 автоматично використовує актуальну схему — вказувати версію більше не потрібно, і це генерує warning. Просто видали рядок `version:` з файлів.

**5. `docker-compose` → `docker compose`**
Docker Compose V1 (Python, команда з дефісом `docker-compose`) офіційно deprecated. Актуальний стандарт — Docker Compose V2 (Go, команда з пробілом `docker compose`). У всіх прикладах нижче використовується V2-синтаксис.

---

## Зміст

**Частина A — Практичний Docker (відео 17)**
1. [Докеризація Flask-додатку](#1-докеризація-flask-додатку)
2. [Flask + Redis + Docker Compose](#2-flask--redis--docker-compose)
3. [Мікросервіси: Flask + Redis + Nginx](#3-мікросервіси-flask--redis--nginx)
4. [Просунуті техніки Dockerfile](#4-просунуті-техніки-dockerfile)
5. [Multi-stage Builds](#5-multi-stage-builds)
6. [Оптимізація розміру образів](#6-оптимізація-розміру-образів)
7. [Безпека контейнерів](#7-безпека-контейнерів)
8. [CI/CD з Docker](#8-cicd-з-docker)

**Частина B — Внутрішня архітектура (відео 18)**
9. [Еволюція ізоляції та ядро Linux](#9-еволюція-ізоляції-та-ядро-linux)
10. [Namespaces — ізоляція видимості](#10-namespaces--ізоляція-видимості)
11. [Cgroups — обмеження ресурсів](#11-cgroups--обмеження-ресурсів)
12. [UnionFS / OverlayFS — шарувата файлова система](#12-unionfs--overlayfs--шарувата-файлова-система)
13. [Ручне створення контейнера без Docker](#13-ручне-створення-контейнера-без-docker)
14. [Архітектура Docker Engine](#14-архітектура-docker-engine)
15. [OCI — стандарти та альтернативні рантайми](#15-oci--стандарти-та-альтернативні-рантайми)
16. [Docker Networking в деталях](#16-docker-networking-в-деталях)
17. [Storage: Volumes, Bind Mounts, tmpfs](#17-storage-volumes-bind-mounts-tmpfs)
18. [Моніторинг контейнерів](#18-моніторинг-контейнерів)
19. [Дебагінг контейнерів](#19-дебагінг-контейнерів)
20. [Швидка шпаргалка](#20-швидка-шпаргалка)

---

# Частина A — Практичний Docker

## 1. Докеризація Flask-додатку

### Архітектура запиту

```
Web Browser → Nginx (Web Server) → Gunicorn (WSGI) → Flask (Python App)
```

### Мінімальний Flask-додаток

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, DevOps01!'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```

### Dockerfile для Flask `[ОНОВЛЕНО]`

```dockerfile
# [ОНОВЛЕНО] python:3.8-slim → python:3.12-slim
# Python 3.8 — EOL з жовтня 2024, більше немає security-патчів.
# python:3.12-slim базується на Debian Trixie/Bookworm (glibc) — правильний вибір.
# НЕ використовуй python:3.12-alpine для Python — musl libc викликає 50x повільніші
# білди через компіляцію пакетів з вихідного коду замість pre-built wheels.
FROM python:3.12-slim

WORKDIR /app

# СПОЧАТКУ requirements.txt — оптимізація кешу шарів
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

# ПОТІМ код (змінюється частіше)
COPY . /app

EXPOSE 50800

CMD ["python3", "./simple-flask-application.py"]
```

### Збірка та запуск

```bash
docker build -t simple-flask-app .
# Маппінг портів: <Host_PORT>:<Container_PORT>
docker run -p 5000:50800 simple-flask-app
```

---

## 2. Flask + Redis + Docker Compose

### Що таке Redis?

Redis — in-memory data structure store:
- **Кешування** — швидкий доступ до даних
- **Session storage** — зберігання сесій
- **Лічильники / черги** — атомарні операції

```
User → App → [Cache Hit?] → Redis (мікросекунди)
                ↓ Miss
              → Database (мілісекунди) → пишемо в Redis
```

### Flask з Redis

```python
from flask import Flask
import redis

app = Flask(__name__)
# 'redis' — ім'я сервісу в Docker Compose, Docker DNS резолвить автоматично
cache = redis.Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = cache.incr('hits')
    return f'Hello Docker! This page has been viewed {count} times.\n'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=50800)
```

> **Service Discovery:** Контейнери в Docker Compose звертаються один до одного по **імені сервісу**. Docker DNS автоматично резолвить `redis` в IP-адресу відповідного контейнера.

### Docker Compose `[ОНОВЛЕНО]`

```yaml
# [ОНОВЛЕНО] Поле 'version:' видалено — воно застаріле в Docker Compose V2 (v2.25+).
# Docker Compose V2 автоматично використовує актуальну схему.
# Команда: 'docker compose' (з пробілом, V2), а не 'docker-compose' (V1, deprecated).

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
    depends_on:
      - redis
    networks:
      - webnet

  redis:
    image: "redis:alpine"   # Redis:alpine — нормально, musl-проблема стосується Python
    networks:
      - webnet

networks:
  webnet:
```

```bash
# [ОНОВЛЕНО] Docker Compose V2 — команда з пробілом
docker compose up -d           # Запустити у фоні
docker compose down            # Зупинити та видалити
docker compose logs -f web     # Логи сервісу
docker compose ps              # Стан сервісів
```

---

## 3. Мікросервіси: Flask + Redis + Nginx

### Monolith vs Microservices

| | Моноліт | Мікросервіси |
|---|---|---|
| Розгортання | Єдиний артефакт | Незалежні сервіси |
| Масштабування | Вся система | Окремі компоненти |
| Відмовостійкість | Падає все | Падає лише один сервіс |
| Складність | Проста розробка | Складна оркестрація |

### Nginx як Frontend Proxy

```
Client → Nginx:80 → Flask:5000 → Redis:6379
```

Nginx у мікросервісах:
- **Reverse Proxy** — маршрутизація запитів
- **Load Balancing** — розподіл навантаження
- **SSL Termination** — HTTPS на рівні проксі
- **Security** — приховує внутрішню структуру

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://web:5000/api/data;  # 'web' — ім'я сервісу
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Docker Compose для трирівневої архітектури `[ОНОВЛЕНО]`

```yaml
# [ОНОВЛЕНО] Без поля 'version:' — Docker Compose V2 стандарт
services:
  web:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - redis
    networks:
      - webnet

  redis:
    image: "redis:alpine"
    networks:
      - webnet

  nginx:
    image: "nginx:alpine"
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - web
    networks:
      - webnet

networks:
  webnet:
```

---

## 4. Просунуті техніки Dockerfile

### ARG vs ENV

| | ARG | ENV |
|---|---|---|
| Коли доступний | Тільки під час **build** | Build **і** runtime |
| Де видно | Лише в Dockerfile | В контейнері як змінна середовища |
| Переоснащення | `--build-arg` при `docker build` | `-e` при `docker run` |
| Безпека | Краще для build-параметрів | Видно через `docker inspect` |

```dockerfile
ARG DEFAULT_APP_PORT=3000
RUN echo "Port: $DEFAULT_APP_PORT"

ENV APP_PORT=3000
```

```bash
docker build --build-arg DEFAULT_APP_PORT=8080 -t my-image .
docker run -e APP_PORT=8080 my-image
docker run --env-file .env my-image
```

### ENTRYPOINT vs CMD

**Ключовий принцип:**
- `ENTRYPOINT` — незмінна команда-виконавець
- `CMD` — параметри за замовчуванням (можна перевизначити при `docker run`)

> **Важливо:** Завжди використовуй **exec-форму** `["executable", "param"]` замість shell-форми `executable param`. Причина: shell-форма запускає процес через `/bin/sh -c`, і PID 1 стає `sh`, а не твій застосунок. Це означає, що SIGTERM при `docker stop` не буде доставлено твоєму процесу — контейнер буде вбито примусово через таймаут.

```dockerfile
FROM python:3.12-slim
ENTRYPOINT ["python3"]            # Завжди python3
CMD ["-m", "http.server", "8080"] # Дефолтні параметри
```

```bash
docker run my-image                          # python3 -m http.server 8080
docker run my-image -c "print('Hello')"      # python3 -c "print('Hello')"
docker run --entrypoint /bin/bash my-image   # Повністю перевизначає ENTRYPOINT
```

| Комбінація | Результат |
|---|---|
| Нема нічого | Помилка |
| Тільки `CMD ["app", "arg"]` | `app arg` |
| Тільки `ENTRYPOINT ["ep"]` | `ep` |
| `ENTRYPOINT ["ep"]` + `CMD ["arg"]` | `ep arg` |
| `ENTRYPOINT ["ep", "p1"]` + `CMD ["p1", "p2"]` | `ep p1 p1 p2` |

---

## 5. Multi-stage Builds

### Навіщо?

Один Dockerfile — кілька `FROM`. Великий образ для збірки → мінімальний для production.

```
[Build Stage]         [Production Stage]
golang:alpine    →    scratch / alpine
компілятор            тільки бінарник
залежності збірки     без build-tools
~800 MB               ~10 MB
```

### Приклад: Go-додаток

```dockerfile
# Stage 1: Збірка
FROM golang:alpine AS builder

RUN apk update && apk add --no-cache git
WORKDIR $GOPATH/src/myapp/
COPY . .
RUN go get -d -v
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /go/bin/hello .

# Stage 2: Мінімальний production-образ
FROM scratch

COPY --from=builder /go/bin/hello /go/bin/hello
ENTRYPOINT ["/go/bin/hello"]
```

### Приклад: Node.js `[ОНОВЛЕНО]`

```dockerfile
# [ОНОВЛЕНО] node:16 → node:22-slim
# Node 14 — EOL квітень 2023, Node 16 — EOL вересень 2023.
# Node 22 — активний LTS до квітня 2027.
# node:22-slim (Debian slim, glibc) — правильний вибір для production.

FROM node:22-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-slim AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

```bash
docker build -t my-app .                  # Повна збірка
docker build --target builder -t debug .  # Зупинитись на конкретному stage
```

### FROM scratch — абсолютний мінімум

`FROM scratch` — порожній образ без ОС. Підходить для статично скомпільованих бінарників (Go, Rust). Забезпечує мінімальну поверхню атаки.

---

## 6. Оптимізація розміру образів

### Стратегія 1: Правильний базовий образ `[ОНОВЛЕНО]`

```
# Python (станом на 2026):
python:3.12          → ~1 GB    (повний Debian)
python:3.12-slim     → ~130 MB  (Debian slim, glibc) ← РЕКОМЕНДОВАНО
python:3.12-alpine   → ~52 MB   (Alpine, musl libc)  ← НЕ рекомендовано для Python

# Node.js (станом на 2026):
node:22              → ~1.1 GB  (повний Debian)
node:22-slim         → ~200 MB  (Debian slim, glibc) ← РЕКОМЕНДОВАНО
node:22-alpine       → ~135 MB  (Alpine, musl libc)  ← прийнятно, але є нюанси

# Go / Rust:
scratch              → 0 MB     ← ідеально для статичних бінарників
```

> **Чому НЕ alpine для Python?**  
> Alpine використовує `musl libc` замість `glibc`. Колеса (wheels) у PyPI зібрані під `glibc`. На Alpine `pip install` змушений компілювати пакети з вихідного коду — це може бути у 50 разів повільніше. Крім того, є задокументовані проблеми з DNS в Kubernetes та крашами при multithreading. Для Node.js ситуація менш критична, але сумісність теж не гарантована.

### Стратегія 2: Кешування шарів

Docker кешує кожен шар. Якщо шар змінився — всі наступні перебудовуються.

**Правило:** Рідко змінювані інструкції — вгорі. Часто змінювані — внизу.

```dockerfile
# Погано: зміна будь-якого файлу → pip install заново
COPY . /app
RUN pip install -r requirements.txt

# Добре: requirements.txt змінюється рідко → pip install кешується
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . /app
```

```
Dockerfile A:   A → B → C
Dockerfile B:   A → B_CHANGED → C

A: cache HIT
B: cache MISS (змінився)
C: cache MISS (попередній шар не з кешу — весь ланцюг перебудовується)
```

### Стратегія 3: Об'єднання RUN-команд

```dockerfile
# Погано: 3 окремих шари
RUN apt-get update
RUN apt-get install -y curl wget
RUN rm -rf /var/lib/apt/lists/*

# Добре: 1 шар, кеш видаляється одразу
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### Стратегія 4: .dockerignore

```
# Node.js
**/node_modules/
**/dist
.git
npm-debug.log
.env
.aws

# Python
__pycache__
*.pyc
*.pyo
.Python
env/
pip-log.txt
.coverage
.pytest_cache
.mypy_cache
.git
```

---

## 7. Безпека контейнерів

### Non-root user `[ОНОВЛЕНО]`

```dockerfile
# [ОНОВЛЕНО] node:14 → node:22-slim
FROM node:22-slim

RUN useradd -m appuser
USER appuser

WORKDIR /home/appuser/app
COPY --chown=appuser:appuser . .
CMD ["node", "server.js"]
```

```bash
docker exec <id> whoami    # Перевірити поточного користувача
```

### Обмеження ресурсів

```bash
docker run \
  --memory=400M \
  --memory-swap=500M \
  --cpus=2 \
  my-image
```

### Scanning образів

```bash
docker scout cves my-image    # Docker Scout (вбудований у Docker Desktop)
trivy image my-image           # Trivy (популярний open-source, рекомендовано)
snyk container test my-image   # Snyk
```

### Privileged Containers

```bash
# НЕ використовувати без гострої потреби
docker run --privileged my-image
```

`--privileged` дає контейнеру майже повний доступ до ядра хоста.

**Альтернатива — точкові capabilities:**

```bash
docker run \
  --cap-drop=ALL \
  --cap-add=NET_ADMIN \
  my-image
```

| Capability | Що дозволяє |
|---|---|
| `NET_ADMIN` | Мережеве адміністрування |
| `SYS_PTRACE` | Дебагінг процесів |
| `CHOWN` | Зміна власника файлів |
| `SETUID/SETGID` | Зміна UID/GID процесу |

**Legit use cases для privileged:**
- Пряма взаємодія з hardware-пристроями
- Маніпуляції з параметрами ядра (system-level testing)
- Docker-in-Docker (запуск Docker daemon в контейнері для CI)

---

## 8. CI/CD з Docker

### Базовий CI flow

```bash
# [ОНОВЛЕНО] docker-compose → docker compose (V2)
docker build -t my-app:latest .        # 1. Збірка
docker run my-app:latest npm test      # 2. Тести
docker push my-app:${CI_COMMIT_SHA}    # 3. Push до Registry
```

### Стратегії тегування

```
4.7.6
│ │ └── Patch (виправлення багів)
│ └──── Minor (нові функції, без зламу API)
└────── Major (зміни, що ламають API)
```

> **`latest` — погана практика для production.** Не несе інформації про версію, неможливо відстежити що задеплоєно.

```bash
docker build -t myapp:1.2.3-prod .
docker build -t myapp:1.2.3-$(git rev-parse --short HEAD) .
docker build -t myapp:${VERSION}-${CI_PIPELINE_ID}-$(date +%Y%m%d) .
```

### GitHub Actions

```yaml
name: Docker CI/CD
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: myrepo/myapp:${{ github.sha }}
          cache-from: type=registry,ref=myrepo/myapp:buildcache
          cache-to: type=registry,ref=myrepo/myapp:buildcache,mode=max
```

---

# Частина B — Внутрішня архітектура

## 9. Еволюція ізоляції та ядро Linux

### Передісторія

| Технологія | Рік | Що робить |
|---|---|---|
| `chroot` | 1979 | Змінює кореневий каталог процесу |
| FreeBSD Jails | 2000 | Ізоляція ФС та мережі |
| LXC (Linux Containers) | 2008 | Використовує namespaces + cgroups |
| Docker | 2013 | Зручний інтерфейс над LXC → власний libcontainer |

### Ключовий інсайт

> **Контейнер — це не віртуальна машина.** Це звичайний Linux-процес, якому ядро показує обмежений "вид" на систему за допомогою namespaces та обмежує ресурси через cgroups.

```
VM:        Hardware → Hypervisor → Guest OS → App
Container: Hardware → Host OS Kernel → [namespace + cgroup] → App
```

Контейнер **ділить ядро** з хостом. Тому запускається за мілісекунди, а не хвилини.

---

## 10. Namespaces — ізоляція видимості

Namespace — механізм ядра Linux, що дає процесу ізольований "вид" на певний ресурс системи.

### PID Namespace

```
Host OS:                   Container:
PID 1   → systemd          PID 1  → my-app   (насправді PID 847 на хості)
PID 2   → kthreadd         PID 2  → worker
PID 847 → my-app
```

Процес у контейнері вважає себе `PID 1`. Це критично: `PID 1` відповідає за обробку сигналів і reaping zombie-процесів. Саме тому важливо використовувати exec-форму в ENTRYPOINT.

### Network Namespace

Кожен контейнер отримує власний ізольований мережевий стек: власні інтерфейси, таблицю маршрутизації, правила `iptables` та `localhost`.

```bash
ip netns list                     # Переглянути мережеві namespaces на хості
ip netns exec <ns> ip addr        # Інтерфейси конкретного namespace
```

### IPC Namespace

Ізолює засоби міжпроцесної комунікації: shared memory, semaphores, message queues. Процеси у різних контейнерах не можуть "бачити" IPC-ресурси одне одного.

### User Namespace

Дозволяє процесу мати `UID 0` (root) **всередині контейнера**, але відображатись як непривілейований користувач на рівні хоста.

```
Всередині контейнера: uid=0(root)
На хост-машині:       uid=1000(appuser)
```

Це — основа **rootless containers** (Podman, Docker rootless mode).

### Зведена таблиця namespaces

| Namespace | Що ізолює |
|---|---|
| PID | Дерево процесів |
| Network | Мережевий стек (інтерфейси, routing, iptables) |
| IPC | Shared memory, semaphores, message queues |
| User | Маппінг UID/GID (root → unprivileged) |
| Mount | Дерево монтувань файлової системи |
| UTS | Hostname та domain name |
| Cgroup | Видимість cgroup-ієрархії |

---

## 11. Cgroups — обмеження ресурсів

**cgroups (Control Groups)** — механізм ядра для обмеження, обліку та ізоляції використання фізичних ресурсів групами процесів.

### Принцип роботи

Реалізовані як **псевдофайлова система** у `/sys/fs/cgroup/`. Ліміти задаються записом значень у файли:

```bash
# Docker робить це автоматично при --memory=200M
echo "209715200" > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
```

### Що обмежують cgroups

| Ресурс | Підсистема |
|---|---|
| CPU (час) | `cpu`, `cpuacct` |
| CPU (cores) | `cpuset` |
| Пам'ять | `memory` |
| Диск (I/O) | `blkio` |
| Мережа | `net_cls`, `net_prio` |

```bash
docker run \
  --memory=400M \
  --memory-swap=500M \
  --cpus=1.5 \
  --cpu-shares=512 \
  my-image

# Перевірити OOM Kill
docker inspect --format='{{.State.OOMKilled}}' <id>
```

### cgroups v1 vs v2

| | cgroups v1 | cgroups v2 |
|---|---|---|
| Ієрархія | Кілька незалежних | Єдина уніфікована |
| Де зараз | Старіші дистрибутиви | Ubuntu 21.10+, RHEL 9+, Fedora 31+ |

---

## 12. UnionFS / OverlayFS — шарувата файлова система

### Концепція

Образ Docker складається з **незмінних (read-only) шарів**. При запуску контейнера додається **read-write шар** зверху. Всі шари об'єднуються через **OverlayFS**.

```
┌─────────────────────────────┐
│  Container Layer (R/W)      │  ← зміни під час роботи контейнера
├─────────────────────────────┤
│  Image Layer 3 (R/O)        │  ← COPY . /app
├─────────────────────────────┤
│  Image Layer 2 (R/O)        │  ← RUN pip install
├─────────────────────────────┤
│  Image Layer 1 (R/O)        │  ← FROM python:3.12-slim
└─────────────────────────────┘
```

### Copy-on-Write (CoW)

Якщо контейнер модифікує файл з read-only шару:
1. Файл **копіюється** у R/W шар контейнера
2. Зміни вносяться у копію
3. Оригінал залишається незмінним

Це дозволяє сотням контейнерів ділити один базовий образ **без дублювання даних**.

### OverlayFS — технічні деталі

```
lowerdir  = шари образу (R/O)
upperdir  = R/W шар контейнера
workdir   = технічна директорія OverlayFS
merged    = результат (те, що бачить процес)
```

```bash
docker history my-image         # Переглянути шари образу
docker diff <container_id>      # Зміни у ФС (A=Added, C=Changed, D=Deleted)
```

---

## 13. Ручне створення контейнера без Docker

Демонстрація того, що Docker — це зручна обгортка над нативними механізмами Linux.

```bash
# 1. Завантажити базову файлову систему
mkdir /tmp/mycontainer
# debootstrap stable /tmp/mycontainer http://deb.debian.org/debian/

# 2. Налаштувати cgroup
mkdir -p /sys/fs/cgroup/memory/mycontainer
echo "104857600" > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
echo "$$" > /sys/fs/cgroup/memory/mycontainer/cgroup.procs

# 3. Створити Network namespace
ip netns add mycontainer-ns
ip link add veth0 type veth peer name veth1
ip link set veth1 netns mycontainer-ns
ip netns exec mycontainer-ns ip addr add 10.0.0.2/24 dev veth1
ip netns exec mycontainer-ns ip link set veth1 up

# 4. Запустити ізольований процес
unshare --pid --net --mount --uts --ipc --fork \
  chroot /tmp/mycontainer /bin/sh
```

> **Висновок:** Docker — це зручний API поверх `unshare`, `chroot`, `cgroups` та `OverlayFS`. Розуміння нижнього рівня допомагає ефективно діагностувати проблеми.

---

## 14. Архітектура Docker Engine

Docker — не монолітна програма, а **стек компонентів**:

```
┌─────────────────────────────────────┐
│          Docker CLI                 │  docker build / run / push
└─────────────────┬───────────────────┘
                  │ REST API (unix socket / TCP)
┌─────────────────▼───────────────────┐
│       Docker Daemon (dockerd)       │  REST API server, image management
└─────────────────┬───────────────────┘
                  │ gRPC
┌─────────────────▼───────────────────┐
│           containerd                │  Lifecycle: pull, create, start, stop
└──────────┬──────────────┬───────────┘
           │              │
┌──────────▼──────┐ ┌─────▼───────────┐
│containerd-shim  │ │containerd-shim  │  Зберігає контейнер живим якщо
│  (container 1)  │ │  (container 2)  │  containerd перезапускається
└──────┬──────────┘ └────┬────────────┘
       │                 │
┌──────▼─────────────────▼────────────┐
│               runc                  │  Низькорівневий OCI runtime
│  namespaces + cgroups + OverlayFS   │  (взаємодія з ядром Linux)
└─────────────────────────────────────┘
```

**Docker CLI** — клієнт, надсилає команди через REST API до dockerd.

**dockerd** — REST API server. Управляє образами, мережами, volumes. Спілкується з containerd через gRPC.

**containerd** — високорівневий runtime (CNCF проект). Завантаження образів, lifecycle контейнерів, snapshots.

**containerd-shim** — прошарок між containerd та runc. Дозволяє контейнерам продовжувати працювати якщо containerd перезапускається.

**runc** — низькорівневий OCI runtime. Налаштовує namespaces, cgroups, монтує OverlayFS, запускає процес.

---

## 15. OCI — стандарти та альтернативні рантайми

### Open Container Initiative (OCI)

OCI визначає відкриті стандарти:
- **OCI Image Spec** — формат образу (шари, маніфест, конфігурація)
- **OCI Runtime Spec** — як запускати контейнер
- **OCI Distribution Spec** — протокол Registry (push/pull)

**Мета:** Будь-який OCI-сумісний runtime може запустити будь-який OCI-образ.

### Альтернативні рантайми

| Runtime | Мова | Особливість | Use case |
|---|---|---|---|
| **runc** | Go | Референсна реалізація OCI | Стандартний (Docker, containerd) |
| **crun** | C | Швидший і легший за runc | Podman, оптимізація startup |
| **Kata Containers** | Go/Rust | Легковагові VM замість namespace-ізоляції | Підвищена безпека (multi-tenant) |
| **gVisor (runsc)** | Go | Syscall interception (user-space kernel) | Google Cloud Run |
| **Podman** | Go | Daemonless, rootless | RHEL/Fedora, без root-демона |

```bash
docker run --runtime=crun my-image
docker run --runtime=kata-runtime my-image
```

---

## 16. Docker Networking в деталях

### Мережеві режими

#### Bridge (дефолтний)

```
Container1 ──┐
             ├── docker0 (virtual bridge) ── iptables NAT ── Internet
Container2 ──┘
```

Контейнери отримують IP з підмережі `172.17.0.0/16`. Вихід в інтернет через NAT хоста.

```bash
# Default bridge — БЕЗ DNS між контейнерами
docker run --network bridge my-image

# User-defined bridge — З DNS резолюцією по іменах!
docker network create my-net
docker run --network my-net --name web my-image
docker run --network my-net --name db postgres
# 'web' може звертатись до 'db' по імені
```

> **User-defined bridge vs default bridge:** У user-defined bridge Docker автоматично надає DNS-резолюцію по іменах. У default bridge цього немає — тільки по IP.

#### Host

Контейнер використовує мережевий стек хоста напряму (без NAT, максимальна продуктивність, ризик конфлікту портів).

```bash
docker run --network host my-image
```

#### None

Повна мережева ізоляція — лише `localhost`.

```bash
docker run --network none my-image
```

#### Overlay / VXLAN

Для контейнерів на різних фізичних хостах (Docker Swarm, Kubernetes).

```
Host A                    Host B
Container1 ──┐            ┌── Container3
             ├── VXLAN ───┤
Container2 ──┘            └── Container4
```

### Як працює Port Mapping (під капотом)

Docker автоматично додає правило `iptables`:

```bash
# Docker додає щось подібне до:
iptables -t nat -A DOCKER -p tcp --dport 5000 \
  -j DNAT --to-destination 172.17.0.2:50800
```

```bash
docker network ls
docker network inspect my-net
docker network connect my-net <id>
docker exec web nslookup redis
```

---

## 17. Storage: Volumes, Bind Mounts, tmpfs

Файлова система контейнера **ephemeral** — після видалення всі зміни втрачаються.

### Порівняння

| | Volumes | Bind Mounts | tmpfs |
|---|---|---|---|
| Де | `/var/lib/docker/volumes/` | Будь-де на хості | RAM |
| Керується Docker | Так | Ні | Так |
| Use case | Бази даних | Конфіги, dev hot-reload | Кеш, секрети |
| Продуктивність | Висока | Висока | Максимальна |

### Volumes (рекомендований підхід для даних)

```bash
docker volume create my-data
docker run -v my-data:/var/lib/postgresql/data postgres

docker volume ls
docker volume inspect my-data
docker volume prune
```

```yaml
services:
  db:
    image: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

### Bind Mounts (конфіги та розробка)

```bash
# Read-only конфіг Nginx
docker run -v ./nginx.conf:/etc/nginx/nginx.conf:ro nginx

# Hot-reload для розробки
docker run -v $(pwd)/src:/app/src my-dev-image
```

### tmpfs (в пам'яті)

```bash
docker run --tmpfs /tmp my-image
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=100m my-image
```

Use cases: Redis (максимальна I/O), секрети без запису на диск, тимчасовий кеш.

```yaml
services:
  redis:
    image: redis:alpine
    tmpfs:
      - /data:size=500m
```

---

## 18. Моніторинг контейнерів

### Рівні моніторингу

```
┌──────────────────────────────────────────┐
│  Бізнес-метрики (всередині app)          │  HTTP /metrics, custom metrics
├──────────────────────────────────────────┤
│  Container Runtime метрики               │  containerd / Docker daemon
├──────────────────────────────────────────┤
│  Host-рівень                             │  CPU, RAM, Disk, Network хоста
└──────────────────────────────────────────┘
```

> **Важливо:** Архітектуру моніторингу закладають при написанні коду. Додаток повинен самостійно експозити метрики (наприклад, через `/metrics` у форматі Prometheus).

### Вбудовані інструменти Docker

```bash
docker stats
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
docker stats --no-stream
```

### cAdvisor (Google)

Автоматично зчитує метрики з cgroups та OverlayFS, експозує у форматі Prometheus.

```bash
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:latest
# Web UI: http://localhost:8080
# Prometheus: http://localhost:8080/metrics
```

### Prometheus + Grafana + cAdvisor

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=secret
    depends_on:
      - prometheus

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
  - job_name: 'my-app'
    static_configs:
      - targets: ['web:5000']
```

### Метрики у Flask

```python
from prometheus_client import Counter, Histogram, generate_latest
from flask import Flask, Response

app = Flask(__name__)
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency')

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')

@app.route('/')
def index():
    REQUEST_COUNT.labels(method='GET', endpoint='/').inc()
    return 'Hello!'
```

---

## 19. Дебагінг контейнерів

### Основні команди

```bash
# Логи
docker logs -f <id>
docker logs --tail=100 -t <id>
docker logs --since=1h <id>

# Shell в контейнер
docker exec -it <id> /bin/bash
docker exec -it <id> /bin/sh       # Alpine
docker exec <id> env | grep DB
docker exec <id> ps aux

# Стан та метадані
docker inspect <id>
docker inspect --format='{{.State.Status}}' <id>
docker inspect --format='{{.NetworkSettings.IPAddress}}' <id>
docker stats --no-stream
docker diff <id>                    # Зміни у ФС (A/C/D)
```

### Практичний кейс: знайти версію в production

**Питання з Q&A лекції:** Як знайти точну версію застосунку в контейнері, що вже давно крутиться у prod, якщо початковий тег невідомий?

**Підхід лектора:** Не обмежуйся рівнем Docker-абстракції. Зайди всередину і запитай версію у самого бінарника.

```bash
# 1. Запитати версію у застосунку напряму
docker exec <id> ./app-name --version
docker exec <id> node --version
docker exec <id> python --version

# 2. Перевірити змінні середовища
docker exec <id> env | grep -i version
docker inspect --format='{{.Config.Env}}' <id>

# 3. Перевірити labels образу
docker inspect --format='{{.Config.Labels}}' <id>

# 4. SHA digest образу
docker inspect --format='{{.Image}}' <id>

# 5. Дослідити шари
docker history <image_id>

# 6. Зберегти знімок для аналізу
docker commit <id> debug-snapshot:latest
docker export <id> > container-fs.tar
tar tf container-fs.tar | grep -i version
```

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1
```

```yaml
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
```

### Типові проблеми та вирішення

| Симптом | Причина | Дія |
|---|---|---|
| Container exits immediately | CMD завершується / помилка | `docker logs <id>` → запусти з `-it /bin/sh` |
| Port already in use | Порт зайнятий | `lsof -i :PORT` або `ss -tlpn` |
| Не з'єднуються контейнери | Різні мережі | `docker network inspect` → спільна мережа |
| Permission denied | Wrong user | `docker exec <id> whoami` → виправ `USER` |
| OOMKilled | Перевищено RAM | `docker inspect | grep OOMKilled` → збільш `--memory` |
| Slow build | Кеш не використовується | Переставь `COPY requirements.txt` вище |
| Image too large | Зайві файли / базовий образ | Multi-stage, slim замість alpine для Python |

---

## 20. Швидка шпаргалка

### Dockerfile — повний набір інструкцій

| Інструкція | Призначення |
|---|---|
| `FROM` | Базовий образ |
| `WORKDIR` | Встановити робочу директорію |
| `COPY` | Скопіювати файли |
| `ADD` | Копіювати + розпаковувати архіви |
| `RUN` | Виконати команду при збірці |
| `CMD` | Параметри за замовчуванням |
| `ENTRYPOINT` | Незмінна точка входу |
| `EXPOSE` | Документувати порт |
| `ENV` | Змінна середовища (build + runtime) |
| `ARG` | Змінна лише для build |
| `VOLUME` | Оголосити mount point |
| `USER` | Змінити поточного користувача |
| `LABEL` | Метадані образу |
| `HEALTHCHECK` | Перевірка стану |

### Актуальні версії базових образів (квітень 2026)

| Runtime | Рекомендований образ | НЕ використовувати |
|---|---|---|
| Python | `python:3.12-slim` або `python:3.13-slim` | `python:3.8-*` (EOL), `python:*-alpine` |
| Node.js | `node:22-slim` або `node:24-slim` | `node:14`, `node:16`, `node:18` (всі EOL) |
| Go | `golang:1.22-alpine` → `scratch` | — |
| Nginx | `nginx:alpine` | — |
| Redis | `redis:alpine` | — |

### Docker Networking

| Режим | Ізоляція | Use case |
|---|---|---|
| `bridge` (user-defined) | Так (NAT) + DNS | Рекомендований для більшості сервісів |
| `host` | Ні | High-performance, мінімальна latency |
| `none` | Повна | Batch jobs без мережі |
| `overlay` | Між хостами | Multi-host (Swarm, K8s) |

### Storage

| Тип | Де | Lifetime | Use case |
|---|---|---|---|
| Volume | `/var/lib/docker/volumes/` | Незалежно від контейнера | Бази даних |
| Bind Mount | Будь-де на хості | Незалежно від контейнера | Конфіги, dev |
| tmpfs | RAM | Тільки поки контейнер живий | Кеш, секрети |

### Ядро Linux — компоненти контейнерів

| Компонент | Що робить |
|---|---|
| PID Namespace | Ізолює дерево процесів |
| Network Namespace | Ізолює мережевий стек |
| IPC Namespace | Ізолює міжпроцесну комунікацію |
| User Namespace | root → unprivileged на хості |
| Mount Namespace | Ізолює дерево монтувань |
| UTS Namespace | Ізолює hostname |
| cgroups | Обмежує CPU, RAM, Disk I/O |
| OverlayFS | Шарувата файлова система образів |

### Docker Engine — стек

```
Docker CLI → dockerd → containerd → containerd-shim → runc → Kernel
```

### Моніторинг стек

```
App (/metrics) → cAdvisor → Prometheus → Grafana
docker stats (швидкий погляд)
```

### Checklist безпеки

- [ ] Офіційний базовий образ з конкретним тегом (не `latest`)
- [ ] Non-root user (`USER appuser`)
- [ ] Ліміти ресурсів (`--memory`, `--cpus`)
- [ ] Scanning образів (Trivy / Docker Scout)
- [ ] Лише необхідні порти відкриті
- [ ] Секрети не в Dockerfile — ENV або Docker Secrets
- [ ] Read-only ФС де можливо (`--read-only`)
- [ ] `--privileged` не використовується без потреби
- [ ] `.dockerignore` налаштований
- [ ] Multi-stage build для production

### Checklist оновлення з курсу до 2026

- [ ] `python:3.8-*` → `python:3.12-slim` або `python:3.13-slim`
- [ ] `node:14`, `node:16` → `node:22-slim`
- [ ] `python:*-alpine` → `python:3.12-slim` (musl libc проблеми)
- [ ] `version: '3'` у docker-compose.yml → видалити повністю
- [ ] `docker-compose` → `docker compose` (V2, без дефісу)

---

*Довідник базується на матеріалах курсу DevOps 01 (Artem Hrechanychenko), відео 17 та 18 плейлиста PLZgiAwkeHf_aLE1oGrcJ31mjnNMtnjrGn. Актуалізовано станом на квітень 2026.*
