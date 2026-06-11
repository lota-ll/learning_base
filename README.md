# 📚 DevOps Learning Base

Персональна база знань у сфері **DevOps**.  
Матеріали базуються на курсі DevOps 01 (Artem Hrechanychenko) та доповнені актуальними даними станом на **червень 2026**.

---

## 📂 Вміст

| Файл | Опис |
|------|------|
| [`linux-handbook.md`](./linux-handbook.md) | Повний довідник з Linux: файлова система, права доступу, bash-скриптинг, cron, SSH, systemd |
| [`virtualization-containerization-handbook.md`](./virtualization-containerization-handbook.md) | Віртуалізація та контейнеризація: гіпервізори, KVM, Docker, Dockerfile, Docker Compose, оркестрація |
| [`containers-handbook-part-2.md`](./containers-handbook-part-2.md) | Розширений Docker: multi-stage builds, безпека, CI/CD інтеграція, внутрішня архітектура (namespaces, cgroups, OverlayFS) |
| [`CI_CD-handbook.md`](./CI_CD-handbook.md) | CI/CD від А до Я: принципи CI, GitHub Actions, GitLab CI, Jenkins, артефакти, GitOps, ArgoCD |
| [`linux-for-devops.md`](./linux-for-devops.md) | Advansed матеріал з поєднанням 6 розділів теоріх та 3 практичних проєктів |
| [`api-fundamentals.md`](./api-fundamentals.md) | Production-ready модуль з архітектури, безпеки та управління API, який консолідує 12 теоретичних концептів (від REST і HTTP/3 до gRPC, OAuth2 та API Gateways) із 5 практичними Flask-імплементаціями, що включають OpenAPI-специфікацію, rate limiting та unit-тестування |

---

## ⚠️ Примітка щодо актуальності

Оригінальні матеріали курсу записані у **2023 році**. Кожен довідник містить розділ **«Нотатки актуальності»** з переліком застарілого та рекомендованими замінами (оновлені версії actions, EOL-runtime тощо).

---

## 🗺️ Теми, які покривають матеріали

- **Linux**: CLI, права, процеси, bash-скриптинг, текстові утиліти, логування
- **Мережа**: SSH, основи TCP/IP, load balancing
- **Virtualization**: Type 1/Type 2 гіпервізори, KVM, Vagrant, Terraform
- **Containers**: Docker, Dockerfile best practices, Docker Compose, registry
- **CI/CD**: GitHub Actions, GitLab CI, Jenkins, artifacts, IaC, GitOps
- **API**: REST, HTTP/2/3, gRPC, Webhooks, Auth (JWT/OAuth2/mTLS), OpenAPI/Swagger, Rate Limiting, API Gateways
