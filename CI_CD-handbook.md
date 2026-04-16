# 🚀 CI/CD для DevOps — Повний довідник

> **Джерела:** DevOps 01 Course (Artem Hrechanychenko) — відео 17, 18, 19 + PDF-презентація  
> "Introduction to Continuous Integration: Application, Structure, Build Steps, and Artifacts"

---

## ⚠️ Нотатки актуальності (станом на квітень 2026)

Матеріали курсу записані приблизно в 2023 році. Нижче — все, що знайдено застарілим.

**1. GitHub Actions — версії actions застаріли в усіх прикладах презентаційних матеріалів  (ПМ)**

Це найкритичніша помилка. В презентації всі приклади використовують версії, які офіційно deprecated або навіть повністю вимкнені:

| Дія в ПМ | Проблема | Актуальна версія |
|---|---|---|
| `actions/checkout@v2` | Deprecated | `actions/checkout@v4` |
| `actions/setup-node@v1` | Deprecated | `actions/setup-node@v4` |
| `actions/upload-artifact@v2` | **Повністю вимкнено з 30 січня 2025!** | `actions/upload-artifact@v4` |
| `actions/download-artifact@v2` | **Повністю вимкнено з 30 січня 2025!** | `actions/download-artifact@v4` |

Якщо ваш workflow містить `upload-artifact@v2` або `v3`, він вже не працює — GitHub повністю вимкнув ці версії.

**2. Node.js `'14'` в прикладах → EOL з квітня 2023**

ПМ-приклади використовують `node-version: '14'`. Node 14 помер майже 3 роки тому. Актуальний LTS: Node 22 (підтримка до квітня 2027).

**3. `npm install` → краще `npm ci` в CI-середовищах**

В одному прикладі ПМ (слайд 13) використовується `npm install` замість `npm ci`. Це неправильно для CI — `npm install` може встановлювати різні версії пакетів при різних запусках. `npm ci` — детермінований, завжди встановлює точно те, що у `package-lock.json`.

**4. Travis CI — більше не актуальний**

ПМ згадує Travis CI як один із популярних CI-інструментів нарівні з Jenkins і GitHub Actions. У 2020 компанію продали (Idera), вона припинила безкоштовний доступ для open-source, і більшість проєктів мігрували на GitHub Actions. У 2026 Travis CI — маргінальний сервіс, не є рекомендованим вибором для нових проєктів.

**5. CircleCI — менш рекомендований після 2023**

Після security incident у 2023 (витік секретів через скомпрометовану систему) довіра до CircleCI суттєво знизилась. Не є першим вибором для нових проєктів у 2026.

**6. Node 20 у GitHub Actions → deprecated з червня 2026**

Actions на Node 20 (`actions/checkout@v4`, `actions/setup-python@v5`) будуть переведені на Node 24 з червня 2026. Рекомендується оновити до `v6` для setup-node або дочекатись оновлень від авторів actions.

**7. Terraform → зверни увагу на OpenTofu**

З 2023 Terraform перейшов на ліцензію BSL (не open-source). OpenTofu — open-source fork від Linux Foundation з ідентичним синтаксисом. Якщо у компанії є ліцензійні вимоги — використовуй OpenTofu.

---

## Зміст

**Частина A — Continuous Integration**
1. [Огляд DevOps та CI в Agile](#1-огляд-devops-та-ci-в-agile)
2. [Принципи Continuous Integration](#2-принципи-continuous-integration)
3. [Переваги CI та 12-кроковий цикл](#3-переваги-ci-та-12-кроковий-цикл)
4. [CI-інструменти — огляд та вибір у 2026](#4-ci-інструменти--огляд-та-вибір-у-2026)
5. [Git Webhooks — як запускається пайплайн](#5-git-webhooks--як-запускається-пайплайн)
6. [Структура застосунку та CI](#6-структура-застосунку-та-ci)
7. [Кроки CI-процесу та Build Systems](#7-кроки-ci-процесу-та-build-systems)
8. [Артефакти в CI](#8-артефакти-в-ci)
9. [CI Internals: Server, Runners, взаємодія](#9-ci-internals-server-runners-взаємодія)
10. [Анатомія пайплайну: GitHub Actions та GitLab CI](#10-анатомія-пайплайну-github-actions-та-gitlab-ci)
11. [Best Practices та виклики в CI](#11-best-practices-та-виклики-в-ci)

**Частина B — Continuous Delivery / Deployment**
12. [CI vs CD: ключова різниця](#12-ci-vs-cd-ключова-різниця)
13. [Continuous Delivery vs Continuous Deployment](#13-continuous-delivery-vs-continuous-deployment)
14. [Структура CD-пайплайну та середовища](#14-структура-cd-пайплайну-та-середовища)
15. [Піраміда тестування](#15-піраміда-тестування)
16. [Infrastructure as Code (IaC)](#16-infrastructure-as-code-iac)
17. [GitOps та ArgoCD](#17-gitops-та-argocd)

**Частина C — Інфраструктура застосунків**
18. [Бази даних: SQL та NoSQL](#18-бази-даних-sql-та-nosql)
19. [Реплікація та Шардинг](#19-реплікація-та-шардинг)
20. [Основи мереж та Load Balancing](#20-основи-мереж-та-load-balancing)
21. [Патерни архітектур](#21-патерни-архітектур)
22. [Швидка шпаргалка](#22-швидка-шпаргалка)

---

# Частина A — Continuous Integration

## 1. Огляд DevOps та CI в Agile

### Що таке DevOps

**DevOps** — набір практик, що об'єднує software development (Dev) та IT operations (Ops). Мета: скоротити цикл розробки та забезпечити безперервну доставку з високою якістю.

```
Dev ←──────────────────────────────────────────── Ops
     Plan → Code → Build → Test → Release → Deploy → Monitor
     ◄────────────── Швидкий зворотний зв'язок ──────────────►
```

**Ключові активності DevOps-інженера:**

| Активність | Що включає | Інструменти |
|---|---|---|
| **Code** | Розробка, code review, VCS, merging | Git, GitHub, GitLab |
| **Build** | CI-інструменти, build status | Jenkins, GitHub Actions |
| **Test** | Автоматичне тестування | pytest, Jest, JUnit |
| **Package** | Artifact repository, pre-deployment staging | Nexus, Artifactory, ECR |
| **Release** | Change management, release automation | Jira, release pipelines |
| **Configure** | IaC, конфігурація інфраструктури | Terraform, Ansible |
| **Monitor** | APM, end-user experience | Prometheus, Grafana, Datadog |

### CI та Agile

CI — природне доповнення до Agile-розробки:

```
Agile Development:     Code → Build → Integrate
Continuous Integration:                         → Test → Release
Continuous Delivery:                                          → Deploy (вручну на prod)
Continuous Deployment:                                        → Deploy (автоматично)
DevOps:              ◄──────────── весь цикл + Monitor ─────────────────────────────►
```

CI підтримує Agile-цінності: **ітеративна розробка**, **часті коміти**, **швидкий зворотний зв'язок**.

---

## 2. Принципи Continuous Integration

CI — практика, де розробники **кілька разів на день** інтегрують код у спільний репозиторій, і кожен коміт запускає автоматичні перевірки.

### 4 ключові принципи (за Мартіном Фаулером)

**1. Один спільний репозиторій (Single Source Repository)**
Весь код та ресурси, необхідні для збірки — в одному місці. Немає "паралельних" кодових баз.

**2. Автоматизований процес збірки (Automate the Build)**
Будь-хто може отримати свіжий, готовий до запуску артефакт в будь-який момент, просто запустивши одну команду або push в репозиторій.

**3. Збірка самотестується (Make Builds Self-Testing)**
Після кожної збірки — автоматичні тести. Якщо тести не пройшли, збірка "зламана" і команда отримує повідомлення.

**4. Всі комітять у mainline щодня (Commit to Mainline Daily)**
Часті маленькі коміти запобігають "merge hell" — ситуації, коли гілки настільки розійшлись, що їх злиття займає дні.

```
Developer → Writes code → git commit → git push
                                            ↓
                                   [CI Pipeline triggered]
                                            ↓
                               Lint → Test → Build → Artifact
                                            ↓
                          ✅ Everything OK → Maintainer merges PR
                          ❌ Problem detected → Developer fixes → repeats
```

---

## 3. Переваги CI та 12-кроковий цикл

### Переваги

**Fewer Merge Conflicts** — регулярна інтеграція запобігає масштабним конфліктам злиття.

**Faster Error Detection** — баги виявляються і виправляються одразу після коміту, поки контекст свіжий.

**Reduced Debugging Time** — менші зміни = легше знайти причину проблеми.

**More Time for Features** — автоматика бере на себе рутинні перевірки → розробники фокусуються на функціональності.

### 12-кроковий CI-цикл

Повний цикл від коду до звіту, який описано в презентації:

```
 1. Source code         → Розробник пише код
 2. Version Control     → git commit + git push
 3. Source code build   → Компіляція / packaging
 4. Static code analysis→ Lint, SonarQube (без запуску коду)
 5. Unit tests          → Ізольовані тести окремих функцій
 6. Code coverage       → Відсоток коду покритий тестами
 7. Built artifact      → Зібраний артефакт (jar, binary, Docker image)
 8. Test "fixtures"     → Підготовка тестових даних і середовища
 9. Provision to test   → Розгортання на тестовому оточенні
10. Functional tests    → Перевірка поведінки застосунку
11. Publish reports     → Збереження та публікація звітів
12. Development team    → Сповіщення про результати
```

---

## 4. CI-інструменти — огляд та вибір у 2026

### Повна екосистема DevOps-інструментів

CI — лише одна з частин. Повна екосистема DevOps виглядає так:

| Категорія | Інструменти |
|---|---|
| **SCM/VCS** | Git, GitHub, GitLab, Bitbucket |
| **CI/Build** | Jenkins, GitHub Actions, GitLab CI, Bamboo |
| **Test** | JUnit, pytest, Selenium, JMeter, Cucumber |
| **Artifact Management** | Nexus, JFrog Artifactory, Docker Registry |
| **Deploy** | Ansible, SSH, Kubernetes, ArgoCD |
| **Config Management** | Ansible, Chef, Puppet, Terraform |
| **Orchestration** | Kubernetes, Docker Swarm |
| **Monitoring/Logging** | Prometheus, Grafana, ELK Stack, Splunk |

### Порівняння основних CI-інструментів (2026)

| Інструмент | Модель | Частка ринку | Найкращий для |
|---|---|---|---|
| **GitHub Actions** | SaaS / Self-hosted | 68% GitHub-проєктів | Open-source, стартапи, GitHub-проєкти |
| **GitLab CI** | SaaS / Self-hosted | +34% YoY enterprise | All-in-one, DevSecOps, regulated industries |
| **Jenkins** | Self-hosted | 80% Fortune 500 (legacy) | Enterprise, складні пайплайни, on-prem |
| **Azure DevOps** | SaaS / Self-hosted | Microsoft ecosystem | .NET, Azure-інфраструктура |
| **AWS CodeBuild** | SaaS | AWS ecosystem | AWS-нативна інфраструктура |
| ~~**Travis CI**~~ | ~~SaaS~~ | ~~Занепадає~~ | ⚠️ **Не рекомендується** для нових проєктів |
| ~~**CircleCI**~~ | ~~SaaS~~ | ~~Менший~~ | ⚠️ Security incident 2023, обережно |

> **Travis CI у 2026:** У 2020 компанію продали Idera, скасували безкоштовний доступ для open-source. Більшість проєктів мігрували на GitHub Actions. Travis CI — більше не рекомендований вибір для нових проєктів, хоча і продовжує існувати як платний сервіс.

---

## 5. Git Webhooks — як запускається пайплайн

### Що таке Webhook?

**Webhook** — HTTP POST-запит, який Git-платформа (GitHub, GitLab) надсилає на вказану URL при певній події (push, PR).

```
Developer → git push → GitHub Repository
                              ↓
                    HTTP POST to webhook URL
                    (JSON payload з інформацією про commit)
                              ↓
                       CI Server (Jenkins / etc.)
                              ↓
               "Є нові зміни! Запускаю pipeline..."
                              ↓
                    Fetch latest code → Run pipeline
```

### Схема: GitHub → Jenkins

```
Local PC → git push → GitHub → webhook (HTTP POST) → Jenkins → build .jar → deploy
```

### Приклад: Webhook у GitHub Actions

У GitHub Actions налаштовувати webhook вручну не потрібно. Подія `on: [push]` — це вбудований trigger, який працює через внутрішній webhook-механізм GitHub.

```yaml
# [ВИПРАВЛЕНО] Актуальні версії actions (v2 вже не працює!)
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4        # ← НЕ @v2, як в PDF!
      - name: Run tests
        run: echo "Running tests..."
```

### Приклад: Webhook у Jenkins

```groovy
// Jenkinsfile
pipeline {
    agent any
    triggers {
        githubPush()    // Слухати GitHub webhook
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

---

## 6. Структура застосунку та CI

Структура застосунку визначає, як організувати CI-пайплайн.

### Типова тришарова архітектура

```
Client (Browser) → HTML/CSS/JS    → latency (швидкодія критична)
                         ↓
                   Server (Backend) → Business Logic → High Availability
                         ↓
                   Database        → Data Storage → Scalability
```

### Як структура впливає на CI

```
Frontend  → npm run build / vite build  → static files
Backend   → python/go/java build        → binary / Docker image
Database  → schema migrations           → flyway / liquibase

Відповідно — ОКРЕМІ пайплайни або окремі jobs:
```

```yaml
# Мульти-job workflow: Frontend та Backend паралельно
# [ВИПРАВЛЕНО] Актуальні версії замість @v2 з PDF
name: CI
on: [push]

jobs:
  test-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4       # ← @v4, не @v2 як в PDF
      - name: Run backend tests
        run: pytest tests/

  test-frontend:
    runs-on: ubuntu-latest             # або windows-latest якщо потрібно
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4    # ← @v4, не @v1 як в PDF
        with:
          node-version: '22'           # ← '22', не '14' як в PDF!
          cache: 'npm'
      - run: npm ci                    # ← npm ci, не npm install!
      - name: Run frontend tests
        run: npm test
```

---

## 7. Кроки CI-процесу та Build Systems

### Кроки CI в правильному порядку (Fail Fast!)

```
1. Checkout code      → git clone / fetch
2. Lint               → перевірка стандартів коду (~1 хв, найдешевше — першим!)
3. Unit Tests         → швидкі тести без залежностей (~5 хв)
4. Build              → компіляція / docker build (~10 хв)
5. Integration Tests  → потребують залежностей (~20 хв)
6. Publish Artifact   → push до Registry
```

### npm ci vs npm install — чому важливо

Це одна з найпоширеніших помилок у CI-пайплайнах (в тому числі в PDF).

| | `npm install` | `npm ci` |
|---|---|---|
| Використовує `package.json` | Так | Ні |
| Використовує `package-lock.json` | Так (може оновлювати) | Так (суворо) |
| Оновлює lockfile | Так | **Ніколи** |
| Видаляє `node_modules` перед встановленням | Ні | Так (завжди чисто) |
| Гарантує точні версії | **Ні** | **Так** |
| Швидкість | Повільніше | У 2 рази швидше |
| Призначення | Локальна розробка | **CI/CD та production** |

```bash
# ❌ Неправильно для CI (з PDF слайду 13)
- run: npm install

# ✅ Правильно для CI
- run: npm ci
```

> `npm ci` без `package-lock.json` завершується з помилкою. Це feature, не bug — `package-lock.json` обов'язково має бути в репозиторії.

### Build Systems для різних мов

```
Мова           Build System       Команда в CI
──────────────────────────────────────────────────
Java           Maven              mvn clean package
               Gradle             ./gradlew build
JavaScript     npm                npm ci && npm run build
               Yarn               yarn install --frozen-lockfile && yarn build
Python         pip + setuptools   pip install -r requirements.txt && python setup.py
               Poetry             poetry install && poetry build
Go             go build           go build ./...
Rust           cargo              cargo build --release
Docker         Docker Engine      docker build -t myapp .
```

### Приклад: Node.js CI (повністю виправлений відносно PDF)

```yaml
# [ВИПРАВЛЕНО] Слайд 22 з PDF — оновлені версії та npm ci
name: CI
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # ← @v4 (не @v2)
      - uses: actions/setup-node@v4        # ← @v4 (не @v1)
        with:
          node-version: '22'               # ← '22' (не '14')
          cache: 'npm'                     # Автоматичне кешування npm
      - run: npm ci                        # ← npm ci (не npm install)
      - run: npm run build --if-present
```

---

## 8. Артефакти в CI

### Що таке артефакт?

**Артефакт** — файли, що виробляються кроком у CI/CD пайплайні. Можуть включати скомпільований код, логи або звіти. Артефакти можна зберігати та передавати між jobs в одному пайплайні або використовувати в майбутніх запусках.

| Тип проєкту | Артефакт |
|---|---|
| Web-додаток | Docker image, static files |
| Java | `.jar` / `.war` |
| Go / C++ | Скомпільований бінарник |
| Android | `.apk` / `.aab` |
| iOS | `.ipa` |
| Embedded | Firmware (прошивка) |

### Ключова властивість: "Build Once, Deploy Many"

```
CI → [Build] → Artifact → Dev env
                       ↓ (same artifact)
                     Staging env
                       ↓ (same artifact)
                     Production env
```

Один артефакт проходить через всі середовища без перезбірки — гарантія, що в prod піде саме те, що тестувалось.

### Приклад: Artifacts у GitHub Actions

```yaml
# [ВИПРАВЛЕНО] upload-artifact@v2 вимкнено з 30 січня 2025!
name: CI
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm run build --if-present

      - uses: actions/upload-artifact@v4  # ← @v4! (не @v2 як в PDF)
        with:
          name: dist-artifact
          path: dist/
          retention-days: 7              # Автовидалення через 7 днів

  deploy:
    needs: build                         # Залежить від job build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4  # ← @v4! (не @v2 як в PDF)
        with:
          name: dist-artifact
          path: dist/

      - name: Deploy
        run: ./deploy.sh
```

### Сховища артефактів

| Сховище | Тип | Use case |
|---|---|---|
| **GitHub Container Registry (ghcr.io)** | SaaS | Docker images у GitHub |
| **GitLab Container Registry** | SaaS/Self-hosted | Docker images у GitLab |
| **AWS ECR** | SaaS | Docker images в AWS |
| **JFrog Artifactory** | Self-hosted/SaaS | Universal: Docker, Maven, npm |
| **Sonatype Nexus** | Self-hosted | Maven, npm, Docker (enterprise) |
| **AWS S3** | SaaS | Generic files, static assets |

---

## 9. CI Internals: Server, Runners, взаємодія

### Ключові компоненти CI-системи

```
Source Code Repository    → Git (GitHub, GitLab, Bitbucket)
Build System              → npm, Maven, Gradle, pip, make
Pipeline Definition       → .github/workflows/*.yml, .gitlab-ci.yml, Jenkinsfile
Automation Server         → Jenkins, GitHub Actions platform, GitLab Runner
Runners / Agents          → Виконують джоби
```

### CI Server-Runner Interaction

```
Developer commits → Git Repository → Webhook HTTP POST → CI Server
                                                               ↓
                                                    "Є нова джоба!"
                                                               ↓
                                               Знайти вільний Runner
                                               (враховуючи runner tags)
                                                               ↓
                                                         Runner:
                                                   - Клонує репозиторій
                                                   - Виконує кроки job
                                                   - Повертає результат
                                                               ↓
                                               CI Server агрегує результати
                                                               ↓
                                      Сповіщення (Slack/Email) + Dashboard update
```

### Runners — статичний vs динамічний флот

**Статичний флот (Static Fleet):**
```
GitLab Server
      ↓ pipeline instance
Agent #1 (Runner + Docker) → CI jobs: container#1, container#2, container#3
Agent #2 (Runner + Docker) → CI jobs: container#1, container#2, container#3

Переваги: Немає часу "підняття", зручно для GPU workloads
Недоліки: Платиш постійно, накопичується сміття (старі Docker images, кеші)
          Потрібне регулярне очищення
```

**Динамічний флот (Dynamic Fleet):**
```
GitLab Server
      ↓ pipeline instance
Runner → Platform API (K8s, EC2) → Spawn ephemeral containers
                                   CI job: container#1 ... #N
                                   (після завершення — видаляються)

Переваги: Платиш тільки за час виконання, завжди чисте середовище
Недоліки: Час на "підняття" (~30 сек для K8s pod, ~1-3 хв для EC2)
          Потрібна розумна конфігурація кешування залежностей
```

### Runner Tags

Runners можна тегувати для спрямування конкретних джоб на конкретні runner-и:

```yaml
# GitLab CI — вибір runner за тегом
test-frontend:
  tags:
    - frontend    # Буде виконано ЛИШЕ на runners з тегом 'frontend'
  script:
    - npm ci
    - npm test

test-backend:
  tags:
    - backend     # Буде виконано ЛИШЕ на runners з тегом 'backend'
  script:
    - pytest tests/

gpu-training:
  tags:
    - gpu         # Виконати на runner-і з GPU (для ML)
  script:
    - python train.py
```

---

## 10. Анатомія пайплайну: GitHub Actions та GitLab CI

### GitHub Actions — повний сучасний приклад

```yaml
# .github/workflows/ci.yml
# [ВСІ ВЕРСІЇ АКТУАЛЬНІ на квітень 2026]

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'       # Щопонеділка о 6:00 UTC
  workflow_dispatch:           # Ручний запуск

jobs:

  # ─── Job 1: Lint ──────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install flake8 black mypy
      - run: flake8 . --max-line-length=88
      - run: black --check .

  # ─── Job 2: Unit Tests ────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    needs: lint                # Починається ПІСЛЯ lint
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      - run: pip install -r requirements.txt pytest pytest-cov
      - run: pytest --cov=src --cov-report=xml

      - uses: actions/upload-artifact@v4      # ← @v4!
        if: always()                          # Зберігати навіть якщо тести впали
        with:
          name: coverage-${{ matrix.python-version }}
          path: coverage.xml
          retention-days: 7

  # ─── Job 3: Build Docker image ────────────────────────────────
  build:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}  # Секрет — не в коді!

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}
          cache-from: type=registry,ref=myorg/myapp:buildcache
          cache-to: type=registry,ref=myorg/myapp:buildcache,mode=max

  # ─── Job 4: Deploy to staging ─────────────────────────────────
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Kubernetes
        run: kubectl set image deployment/myapp myapp=myorg/myapp:${{ github.sha }}
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_STAGING }}
```

### GitLab CI — повний приклад

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - test
  - build
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install flake8 black
    - flake8 . && black --check .
  only:
    - merge_requests
    - main

test:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt pytest
    - pytest --junit-xml=report.xml
  artifacts:
    reports:
      junit: report.xml          # GitLab показує результати в UI
    expire_in: 1 week
    when: always                 # Зберігати навіть при failure

build:
  stage: build
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - main

deploy-staging:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy-production:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n prod
  environment:
    name: production
  when: manual                   # Потребує ручного підтвердження (Delivery mode)
  only:
    - main
```

---

## 11. Best Practices та виклики в CI

### 8 DevOps CI/CD Best Practices (з презентації)

**1. Commit Early and Commit Often**
Часті маленькі коміти → менше конфліктів, простіше дебагувати.

**2. Keep the Builds Green**
Зламаний build — пріоритет №1 для виправлення. "Broken build is a team emergency."

**3. Build Only Once**
Один артефакт — через всі середовища. Не перезбирати для кожного env.

**4. Streamline Tests**
Паралелізація тестів, видалення дублікатів, пріоритизація швидких тестів.

**5. CI/CD Pipeline is the Only Way**
Жодних ручних деплойментів. Всі зміни — тільки через пайплайн.

**6. Monitoring and Measuring Your Pipeline**
Відстежуй час виконання кожного кроку. Якщо build grows > 15 хвилин — оптимізуй.

**7. Make it a Team Effort**
DevOps-інженер + розробники разом визначають що, як і коли тестувати.

**8. Clean Your Environment**
Ephemeral runners або очищення перед кожним запуском. Ніяких "залишків" від попередніх збірок.

### Fail Fast — принцип пріоритизації

```yaml
# Правильний порядок: спочатку найшвидші та найдешевші перевірки
jobs:
  lint:           # ~1 хв  — дешево, першим!
  unit-test:      # ~5 хв  — швидко, після lint
    needs: lint
  build:          # ~10 хв — тільки якщо тести пройшли
    needs: unit-test
  integration:    # ~20 хв — найдорожче, останнім
    needs: build
```

> Якщо unit-тести не пройшли — немає сенсу запускати Docker build. Пайплайн зупиняється одразу.

### Швидкість пайплайну

**Ціль: < 15 хвилин.** Довший пайплайн блокує розробників.

```yaml
# Паралелізація тестів через matrix
test:
  strategy:
    matrix:
      shard: [1, 2, 3, 4]
  steps:
    - run: pytest tests/ --shard=${{ matrix.shard }}/4

# Кешування залежностей
- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'npm'          # Автоматично кешує node_modules
```

### Секрети в пайплайнах

```yaml
# ❌ НІКОЛИ в коді
env:
  DB_PASSWORD: "my_secret_password"

# ✅ Через Secrets
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  # Зберігається в: Settings → Secrets and variables → Actions
  # Автоматично маскується в логах (не відображається)
```

**Системи управління секретами:**
- **GitHub Secrets / GitLab Variables** — вбудовані, прості, маскуються в логах
- **HashiCorp Vault** — enterprise, динамічні секрети (видаються на обмежений час), аудит
- **AWS Secrets Manager** — для AWS-інфраструктури
- **OIDC (OpenID Connect)** — сучасний підхід: замість довгоживучих токенів, CI-платформа отримує короткоживучий токен від хмарного провайдера

```yaml
# OIDC: GitHub Actions → AWS без зберігання AWS credentials
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: eu-west-1
    # Немає AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY!
    # GitHub отримує тимчасовий token через OIDC
```

### Виклики в CI

```
1. Підтримка comprehensive test suite
   → Тести застарівають, потребують постійної підтримки

2. Швидкість збірки
   → З часом builds ростуть. Потрібне кешування та паралелізація

3. Заохочення розробників до частих комітів
   → Культурне питання — пайплайн має бути швидким і не заважати роботі

4. Версіонування actions
   → actions застарівають (як v2 з PDF). Рекомендовано Dependabot для автооновлень
```

### Автоматичне оновлення actions (Dependabot)

```yaml
# .github/dependabot.yml
# Автоматично відкриває PR при появі нових версій actions
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"    # Перевірка щотижня
```

---

# Частина B — Continuous Delivery / Deployment

## 12. CI vs CD: ключова різниця

```
CI (Continuous Integration)
└── ПЕРЕВІРКА коду: lint → unit tests → build → artifact
    Питання: "Чи готовий код до релізу?"

CD (Continuous Delivery / Deployment)
└── ДОСТАВКА артефакту: інфраструктура → розгортання → production
    Питання: "Як доставити готовий artifact на сервери?"
```

```
Code → Lint → Test → Build → [Artifact] → Deploy Dev → Deploy Staging → Deploy Prod
◄──────── CI ──────────────►◄────────────────── CD ──────────────────────────────────►
```

---

## 13. Continuous Delivery vs Continuous Deployment

| | Continuous Delivery | Continuous Deployment |
|---|---|---|
| Автоматизація | До підготовки релізу | Повністю, до Production |
| Production деплой | **Ручний** (кнопка) | **Автоматичний** |
| Підходить для | Банки, regulated industries | Netflix, Amazon, стартапи |
| Передумова | Якісний тест-suite | Дуже якісний тест-suite |

```
Continuous Delivery:
CI → Artifact → Dev → Staging → [👤 Manual Approve] → Prod

Continuous Deployment:
CI → Artifact → Dev → Staging → E2E Tests → Prod (автоматично!)
```

---

## 14. Структура CD-пайплайну та середовища

### Середовища

```
Dev          → Staging           → Production
──────────────────────────────────────────────────────
Автодеплой    Автодеплой           Delivery = ручно
при merge     після Dev            Deployment = авто
              Тут E2E тести
```

### Стратегії деплойменту

**Rolling Update:**
```
v1 v1 v1 v1 → v2 v1 v1 v1 → v2 v2 v1 v1 → v2 v2 v2 v2
```

**Blue-Green Deployment:**
```
Blue (Prod: v1) ← трафік
Green (Idle: v2) ← deploy + test
→ переключити:
Blue (Idle: v1)  ← rollback якщо потрібно
Green (Prod: v2) ← трафік
```

**Canary Deployment:**
```
v1: 100% → v2: 5% → v2: 20% → v2: 100%
(поступово, з моніторингом метрик)
```

---

## 15. Піраміда тестування

```
        ▲
        │       /─────────────\
        │      /  E2E / UAT    \         Повільні, дорогі
        │     /─────────────────────\
        │    /  Integration Tests    \   Середня швидкість
        │   /─────────────────────────────\
        │  /       Unit Tests              \ Швидкі, дешеві ← першими в CI!
        ▼/─────────────────────────────────\
```

| Рівень | Швидкість | Де в пайплайні | Що перевіряє |
|---|---|---|---|
| **Unit Tests** | Секунди | CI (Job 1) | Окремі функції, ізольовано |
| **Integration Tests** | Хвилини | CI (Job 2) | Взаємодія між модулями |
| **Smoke Tests** | Хвилини | Після деплою на Staging | Базова працездатність |
| **Performance Tests** | Хвилини-години | Staging | Навантаження |
| **E2E / UAT** | Десятки хвилин | Staging перед Prod | Симуляція реального юзера |

---

## 16. Infrastructure as Code (IaC)

Замість ручних кліків у хмарній консолі — уся інфраструктура описується у текстових файлах і зберігається в Git.

```
Без IaC:   AWS Console → клікати → ручна конфігурація
З IaC:     terraform apply (Git commit → pipeline → infrastructure)
```

### Переваги IaC

```
Version Control  → Інфраструктура в Git: history, diff, rollback
Code Review      → PR для змін в інфраструктурі, як для коду
Reproducibility  → Однакове середовище у Dev, Staging, Prod
CI/CD Integration→ Infrastructure пайплайни (plan → apply)
```

### Terraform

```hcl
provider "aws" { region = "eu-west-1" }

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = "web-server", Environment = "production" }
}
```

```bash
terraform init     # Ініціалізувати провайдерів
terraform plan     # Dry run: що зміниться
terraform apply    # Застосувати
terraform destroy  # Знищити
```

> **Нотатка 2026:** Після зміни ліцензії Terraform на BSL у 2023, з'явився **OpenTofu** — open-source fork від Linux Foundation. Синтаксис ідентичний. Вибір залежить від ліцензійних вимог.

### IaC у пайплайні

```yaml
jobs:
  terraform:
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -out=tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      - if: github.ref == 'refs/heads/main'
        run: terraform apply tfplan
```

---

## 17. GitOps та ArgoCD

**GitOps** — підхід, де Git — єдине джерело правди про стан інфраструктури.

```
Traditional CD ("push"):     GitOps ("pull"):
CI → docker build            CI → docker build
   → kubectl apply (push)       → git commit (оновлює маніфест)
                             ArgoCD бачить зміну → sync кластер
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/my-app-config.git
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Видаляти ресурси, яких немає в Git
      selfHeal: true    # Відновлювати ручні зміни у кластері
```

---

# Частина C — Інфраструктура застосунків

## 18. Бази даних: SQL та NoSQL

### SQL (Реляційні)

```
Коли: структуровані дані, ACID-транзакції, незмінна схема
Приклади: PostgreSQL, MySQL, MariaDB
Cloud PaaS: AWS RDS, Google Cloud SQL, Azure Database
```

### NoSQL

```
Коли: неструктуровані дані, гнучка схема, горизонтальне масштабування
Types:
  Document:    MongoDB (JSON)
  Key-Value:   Redis (кеш, сесії)
  Wide Column: Cassandra, DynamoDB (Big Data)
  Time-Series: InfluxDB, TimescaleDB (IoT, метрики)
```

```yaml
# Docker Compose: PostgreSQL
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${{ secrets.DB_PASSWORD }}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      retries: 5
```

---

## 19. Реплікація та Шардинг

### Реплікація

```
Write →  Primary (Main) ← читання і запис
              ↓ Replication
         Read Replica 1 ← тільки читання
         Read Replica 2 ← тільки читання

Якщо Primary падає → автоматичний failover (Multi-AZ в AWS RDS)
```

### Шардинг

```
Таблиця users (1 млрд записів)
→ Shard 1: ID 1-250M        → Server 1
→ Shard 2: ID 250M-500M     → Server 2
→ Shard 3: ID 500M-750M     → Server 3
→ Shard 4: ID 750M-1B       → Server 4
      ↑
Proxy/LB маршрутизує запит до потрібного шарда
```

---

## 20. Основи мереж та Load Balancing

### Ключові рівні для DevOps

| Рівень | Протоколи | DevOps використання |
|---|---|---|
| **L7 Application** | HTTP, HTTPS, DNS | Nginx, ALB, API Gateway |
| **L4 Transport** | TCP, UDP + Порти | NLB, Security Groups |
| **L3 Network** | IP | VPC, Subnet, Routing |

### Load Balancing

```
Проблема: 10 000 req/s → один сервер → crash

Рішення: Load Balancer розподіляє між кількома серверами

Client → Load Balancer → Server 1 (33%)
                      → Server 2 (33%)
                      → Server 3 (33%)
```

**L4 (TCP)** — швидкий, без аналізу вмісту запиту (AWS NLB)  
**L7 (HTTP)** — розумний: `/api/*` → backend, `/static/*` → CDN (AWS ALB, Nginx)

**Алгоритми балансування:**
```
Round Robin      → 1→2→3→1→2→3 (рівноцінні сервери)
Least Connections→ до сервера з мінімальним навантаженням
IP Hash          → hash(client_ip) → один сервер (sticky sessions)
Weighted         → Server A 70%, Server B 30% (різна потужність)
```

---

## 21. Патерни архітектур

### Microservices

```
Моноліт → десятки дрібних незалежних сервісів
Кожен: окремий repo, Dockerfile, pipeline, deployment

DevOps задачі:
 - Окремий CI/CD для кожного сервісу
 - Service mesh (Istio, Linkerd)
 - Centralized logging (ELK Stack)
 - Distributed tracing (Jaeger, Zipkin)
```

### Event-Driven Architecture (Kafka/RabbitMQ)

```
User Action → Event → Message Queue → Service A (async)
                                    → Service B (async)

Приклад Uber: замов таксі → Event → Driver matching + Billing (паралельно)
```

### Serverless

```
AWS Lambda: код запускається лише при trigger, оплата за мілісекунди
S3 upload → Lambda (resize photo) → S3 (зберегти)
```

### IoT

```
IoT Device → MQTT Broker → Pipeline → Time-Series DB (InfluxDB)
Порада лектора: для IoT використовуй Time-Series DB (InfluxDB, TimescaleDB)
замість звичайної реляційної. При міграції між хмарами — перевір механізми
кешування MQTT повідомлень (пристрої можуть працювати без стабільного інтернету).
```

---

## 22. Швидка шпаргалка

### GitHub Actions — АКТУАЛЬНІ версії (квітень 2026)

| Action | Актуальна версія | Увага |
|---|---|---|
| `actions/checkout` | `@v4` | v6 вже доступний |
| `actions/setup-node` | `@v4` | v6 вже доступний |
| `actions/setup-python` | `@v5` | Node 20 deprecation coming |
| `actions/upload-artifact` | `@v4` | **v2 і v3 вимкнені з 30.01.2025!** |
| `actions/download-artifact` | `@v4` | **v2 і v3 вимкнені з 30.01.2025!** |
| `actions/cache` | `@v4` | v5 доступний |
| `docker/build-push-action` | `@v5` | |

### npm ci vs npm install

```
npm install  → локальна розробка, може змінювати версії
npm ci       → CI/CD та production, завжди точні версії з lockfile
```

### CI-інструменти — вибір у 2026

| Завдання | Рекомендований інструмент |
|---|---|
| GitHub-проєкти, стартапи | GitHub Actions |
| All-in-one DevOps платформа | GitLab CI |
| Enterprise, legacy, on-prem | Jenkins |
| CD для Kubernetes (GitOps) | ArgoCD або Flux |
| IaC | Terraform або OpenTofu |
| Config Management | Ansible |
| Secrets Management | HashiCorp Vault або AWS Secrets Manager |
| Автооновлення залежностей | Dependabot або Renovate |

### Піраміда тестів у пайплайні

```
1. Lint              ~1 хв   ← першим!
2. Unit Tests        ~5 хв
3. Build             ~10 хв  ← після тестів
4. Integration       ~20 хв
5. E2E / UAT         ~30 хв  ← на staging
```

### 12-кроковий CI-цикл (з презентації)

```
1. Source code     9. Provision to test
2. VCS             10. Functional tests
3. Build           11. Publish reports
4. Static analysis 12. Development team
5. Unit tests
6. Code coverage
7. Built artifact
8. Test fixtures
```

### Checklist сучасного CI/CD

- [ ] GitHub Actions версії `@v4` або вище (не v2/v3!)
- [ ] `npm ci` замість `npm install`
- [ ] Node.js 22+ у workflows (не 14/16/18)
- [ ] Python 3.12+ у workflows (не 3.8!)
- [ ] Fail Fast: lint → unit → build → integration
- [ ] Пайплайн < 15 хвилин
- [ ] Кешування залежностей
- [ ] Секрети в Secrets Manager (не в коді)
- [ ] Semantic versioning для Docker images
- [ ] Lifecycle policy для очищення старих артефактів
- [ ] Dependabot для автооновлення actions
- [ ] OIDC замість довгоживучих cloud credentials
- [ ] Manual approval перед Production (Continuous Delivery)

---

*Довідник базується на матеріалах курсу DevOps 01 (Artem Hrechanychenko), відео 17, 18, 19 плейлиста та PDF-презентації "Introduction to Continuous Integration". Актуалізовано станом на квітень 2026.*
