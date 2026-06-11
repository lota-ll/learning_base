# API для DevOps Engineers

> **Чому це важливо:** Кожен інструмент DevOps — Kubernetes, GitHub Actions, AWS, Prometheus — має API. Розуміти як API влаштований, як його тестувати і як захищати — обов'язкова компетенція.
> **Поточний рівень:** Python/Flask знайомий, HTTP розумієш на рівні браузера
> **Ціль модуля:** проектувати, реалізовувати, тестувати і захищати REST API; впевнено взаємодіяти з будь-яким зовнішнім API
> **Час:** Теорія ~5 год · Практика ~12 год

---

## Зміст

1. [Що таке API і навіщо](#1-що-таке-api-і-навіщо)
2. [HTTP — фундамент](#2-http--фундамент)
3. [REST — архітектурний стиль](#3-rest--архітектурний-стиль)
4. [Автентифікація та авторизація](#4-автентифікація-та-авторизація)
5. [Форматі даних: JSON, XML, Protobuf](#5-формати-даних-json-xml-protobuf)
6. [Версіонування API](#6-версіонування-api)
7. [Rate Limiting та Throttling](#7-rate-limiting-та-throttling)
8. [OpenAPI / Swagger — документація](#8-openapi--swagger--документація)
9. [GraphQL — альтернатива REST](#9-graphql--альтернатива-rest)
10. [gRPC — high-performance RPC](#10-grpc--high-performance-rpc)
11. [Webhooks](#11-webhooks)
12. [API Gateway](#12-api-gateway)
13. [Практичні проекти](#13-практичні-проекти)
14. [Типові помилки](#типові-помилки)
15. [Результат модуля](#результат-модуля)
16. [Interview Prep](#interview-prep)

---

## 1. Що таке API і навіщо

**API** (Application Programming Interface) — контракт між двома програмами про те, як вони спілкуються. Це не технологія, а концепція.

**Аналогія:** Офіціант у ресторані — це API. Ти (клієнт) не йдеш на кухню сам. Ти говориш офіціанту що хочеш (запит), він іде на кухню (сервер), і повертається з їжею (відповідь). Тобі не потрібно знати як влаштована кухня.

```
Клієнт (browser, mobile app, CLI, інший сервіс)
    │
    │  HTTP Request: GET /api/users/42
    ▼
API (контракт: endpoints, формат, auth)
    │
    │  Внутрішня логіка
    ▼
Backend (база даних, бізнес-логіка, інші сервіси)
    │
    ▼
HTTP Response: 200 OK { "id": 42, "name": "Lota" }
```

**Типи API які зустрінеш у DevOps:**

| Тип | Приклад | Коли використовують |
|-----|---------|---------------------|
| REST | GitHub API, AWS API | Web services, мікросервіси |
| gRPC | Kubernetes внутрішня комунікація | High-performance, між сервісами |
| GraphQL | GitHub GraphQL API | Гнучкі запити, BFF pattern |
| WebSocket | Grafana live metrics | Real-time, bi-directional |
| Webhook | GitHub → CI/CD trigger | Event-driven, callbacks |

---

## 2. HTTP — фундамент

### 2.1 HTTP Methods (Verbs)

REST використовує HTTP методи для вираження **наміру** операції:

| Метод | Призначення | Idempotent? | Safe? |
|-------|------------|-------------|-------|
| `GET` | Читання ресурсу | ✅ | ✅ |
| `POST` | Створення ресурсу | ❌ | ❌ |
| `PUT` | Повна заміна ресурсу | ✅ | ❌ |
| `PATCH` | Часткове оновлення | ❌ | ❌ |
| `DELETE` | Видалення ресурсу | ✅ | ❌ |
| `HEAD` | Як GET, але без body | ✅ | ✅ |
| `OPTIONS` | Які методи підтримує endpoint | ✅ | ✅ |

**Idempotent** — повторний виклик дає той самий результат. `DELETE /users/42` двічі — другий раз 404, але стан системи той самий.

**Safe** — не змінює стан сервера. `GET` завжди safe.

### 2.2 HTTP Status Codes

```
1xx — Інформаційні (рідко бачиш)
2xx — Успіх
3xx — Редиректи
4xx — Помилка КЛІЄНТА (ти зробив щось не так)
5xx — Помилка СЕРВЕРА (сервер зламався)
```

| Код | Назва | Коли використовувати |
|-----|-------|---------------------|
| `200` | OK | GET, PUT, PATCH успішно |
| `201` | Created | POST — ресурс створено |
| `204` | No Content | DELETE або PATCH без тіла відповіді |
| `301` | Moved Permanently | Редирект (кешується браузером) |
| `302` | Found | Тимчасовий редирект |
| `304` | Not Modified | Кеш актуальний (ETag/If-None-Match) |
| `400` | Bad Request | Некоректні дані від клієнта |
| `401` | Unauthorized | Не автентифікований (немає токена) |
| `403` | Forbidden | Автентифікований, але немає прав |
| `404` | Not Found | Ресурс не існує |
| `405` | Method Not Allowed | Метод не підтримується для endpoint |
| `409` | Conflict | Конфлікт стану (дублікат email) |
| `422` | Unprocessable Entity | Синтаксис OK, але семантика невірна |
| `429` | Too Many Requests | Rate limit перевищено |
| `500` | Internal Server Error | Щось впало на сервері |
| `502` | Bad Gateway | Upstream сервер повернув помилку |
| `503` | Service Unavailable | Сервер перевантажений або на обслуговуванні |
| `504` | Gateway Timeout | Upstream не відповів вчасно |

### 2.3 HTTP Headers

```bash
# Типові request headers
curl -v https://api.github.com/users/octocat

# Що побачиш:
# > GET /users/octocat HTTP/2
# > Host: api.github.com
# > User-Agent: curl/7.81.0
# > Accept: */*

# < HTTP/2 200
# < content-type: application/json; charset=utf-8
# < x-ratelimit-limit: 60
# < x-ratelimit-remaining: 59
# < x-ratelimit-reset: 1705312800
```

**Важливі headers:**

| Header | Напрям | Призначення |
|--------|--------|------------|
| `Content-Type` | Request/Response | Формат тіла (`application/json`) |
| `Accept` | Request | Який формат хоче клієнт |
| `Authorization` | Request | Токен автентифікації |
| `X-Request-ID` | Request/Response | Унікальний ID для трейсингу |
| `Cache-Control` | Response | Як кешувати відповідь |
| `ETag` | Response | Ідентифікатор версії ресурсу |
| `Location` | Response | URL нового ресурсу після POST |
| `Retry-After` | Response | Через скільки секунд повторити (429/503) |
| `X-RateLimit-*` | Response | Стан rate limiter |

### 2.4 HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1:
  - Один запит за раз на з'єднання
  - Keep-Alive для повторного використання з'єднання
  - Plain text protocol
  - Проблема: head-of-line blocking

HTTP/2:
  - Мультиплексинг: кілька запитів одночасно по одному з'єднанню
  - Header compression (HPACK)
  - Server Push
  - Binary protocol (швидший парсинг)
  - Стандарт для сучасних API

HTTP/3 (QUIC):
  - Побудований на UDP замість TCP
  - Швидший handshake (0-RTT)
  - Краще для мобільних мереж (зміна IP не руйнує з'єднання)
  - Cloudflare, Google вже підтримують
```

```bash
# Перевір яку версію HTTP використовує сайт
curl -sI --http2 https://api.github.com | head -5
curl -sI --http3 https://cloudflare.com | head -5

# Nginx: увімкнути HTTP/2
# listen 443 ssl http2;
```

---

## 3. REST — архітектурний стиль

### 3.1 REST Constraints (6 принципів Fielding)

REST — це не стандарт, а архітектурний стиль, описаний Roy Fielding у 2000 році. API вважається REST якщо відповідає цим принципам:

**1. Client-Server** — клієнт і сервер розділені, розвиваються незалежно.

**2. Stateless** — сервер не зберігає стан між запитами. Кожен запит — self-contained. Стан — у клієнта або в токені.

```
❌ Stateful: POST /login → сесія на сервері → GET /profile (сервер знає хто ти)
✅ Stateless: GET /profile + Authorization: Bearer <token> (токен несе стан)
```

**3. Cacheable** — відповіді мають позначати чи можна їх кешувати.

**4. Uniform Interface** — єдиний інтерфейс для всіх ресурсів (і є причина чому REST елегантний).

**5. Layered System** — клієнт не знає чи спілкується з кінцевим сервером чи з proxy/load balancer.

**6. Code on Demand** (опціонально) — сервер може надсилати виконуваний код (JavaScript).

### 3.2 Ресурси та URL Design

**Аналогія:** URL — це адреса, не дія. `/users/42` — адреса конкретного юзера. Що з ним робити — вказує HTTP метод.

```
✅ Правильний REST URL дизайн:
GET    /users              → список всіх юзерів
POST   /users              → створити нового юзера
GET    /users/42           → отримати юзера #42
PUT    /users/42           → повністю замінити юзера #42
PATCH  /users/42           → частково оновити юзера #42
DELETE /users/42           → видалити юзера #42
GET    /users/42/orders    → замовлення юзера #42
POST   /users/42/orders    → створити замовлення для юзера #42

❌ Погані URL (дія в URL — це не REST):
POST /createUser
GET  /getUserById?id=42
POST /deleteUser/42
GET  /users/42/getOrders

✅ Правила:
- Іменники у множині: /users, /orders, /products
- Lowercase з дефісом: /user-profiles (не /userProfiles, не /user_profiles)
- Вкладеність тільки для прямих залежностей (не більше 2-х рівнів)
- Фільтрація/сортування через query params: GET /users?role=admin&sort=created_at
```

### 3.3 Рівні зрілості REST (Richardson Maturity Model)

```
Level 0: Single endpoint, all via POST
  POST /api  {"action": "getUser", "id": 42}  ← так не роби

Level 1: Resources (різні URL для різних ресурсів)
  GET /users/42
  GET /orders/99

Level 2: HTTP Verbs + Status Codes (більшість "REST" API)
  GET /users/42 → 200 OK
  POST /users → 201 Created
  DELETE /users/42 → 204 No Content

Level 3: Hypermedia (HATEOAS) — відповідь містить посилання на пов'язані дії
  GET /users/42 →
  {
    "id": 42,
    "name": "Lota",
    "_links": {
      "self": { "href": "/users/42" },
      "orders": { "href": "/users/42/orders" },
      "delete": { "href": "/users/42", "method": "DELETE" }
    }
  }
```

Більшість production API — Level 2. Level 3 рідкісний але правильний.

### 3.4 Request та Response Design

```json
// POST /api/v1/users
// Request body:
{
  "name": "Lota Devops",
  "email": "lota@example.com",
  "role": "engineer"
}

// Response 201 Created:
// Location: /api/v1/users/42
{
  "id": 42,
  "name": "Lota Devops",
  "email": "lota@example.com",
  "role": "engineer",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Стандарт формату помилок (RFC 9457 / Problem Details):**

```json
// Response 422 Unprocessable Entity:
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation Error",
  "status": 422,
  "detail": "Request body contains invalid fields",
  "instance": "/api/v1/users",
  "errors": [
    {
      "field": "email",
      "message": "Email already exists"
    },
    {
      "field": "name",
      "message": "Name must be between 2 and 100 characters"
    }
  ]
}
```

**Пагінація:**

```json
// GET /api/v1/users?page=2&per_page=20
{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total": 150,
    "total_pages": 8,
    "next": "/api/v1/users?page=3&per_page=20",
    "prev": "/api/v1/users?page=1&per_page=20"
  }
}

// Cursor-based (краще для великих наборів):
// GET /api/v1/events?cursor=eyJpZCI6MTAwfQ&limit=20
{
  "data": [...],
  "next_cursor": "eyJpZCI6MTIwfQ",
  "has_more": true
}
```

---

## 4. Автентифікація та авторизація

**Автентифікація (AuthN)** — хто ти? Перевірка ідентичності.
**Авторизація (AuthZ)** — що тобі можна? Перевірка прав.

```
Request → [AuthN: хто ти?] → [AuthZ: що тобі можна?] → Handler
              ↓ 401                   ↓ 403
```

### 4.1 API Keys

Найпростіший метод. Статичний секретний рядок.

```bash
# Варіанти передачі API key:
# 1. Header (рекомендовано)
curl -H "X-API-Key: sk-abc123" https://api.example.com/data

# 2. Authorization header
curl -H "Authorization: ApiKey sk-abc123" https://api.example.com/data

# 3. Query param (НЕ рекомендовано — логується в nginx!)
curl "https://api.example.com/data?api_key=sk-abc123"
```

**Коли використовують:** server-to-server інтеграція, прості сервіси без потреби в identity юзера.

**Мінуси:** Не мають терміну дії (потребують ротації), не несуть інформацію про юзера.

### 4.2 Bearer Token / JWT

**JWT** (JSON Web Token) — self-contained токен з підписом. Сервер не зберігає сесії.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.    ← Header (base64)
eyJzdWIiOiI0MiIsIm5hbWUiOiJMb3RhIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzA1MzEyODAwLCJleHAiOjE3MDUzOTkyMDB9.  ← Payload (base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature (HMAC-SHA256)
```

**Структура payload (claims):**

```json
{
  "sub": "42",          // subject (user ID)
  "name": "Lota",       // custom claim
  "role": "admin",      // custom claim
  "iat": 1705312800,    // issued at (unix timestamp)
  "exp": 1705399200,    // expiration (unix timestamp)
  "iss": "auth.example.com",  // issuer
  "aud": "api.example.com"    // audience
}
```

```bash
# Використання:
curl -H "Authorization: Bearer eyJhbGci..." https://api.example.com/users/me

# Декодувати JWT (без верифікації!) — для дебагу
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiJ9.xxx" | \
  cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# або використай: https://jwt.io
```

**Access Token + Refresh Token pattern:**

```
Access Token:  короткий TTL (15 хвилин), використовується для кожного запиту
Refresh Token: довший TTL (7 днів), зберігається безпечно, використовується тільки
               для отримання нового Access Token

POST /auth/refresh
Authorization: Bearer <refresh_token>
→ 200 OK { "access_token": "...", "expires_in": 900 }
```

### 4.3 OAuth 2.0

OAuth 2.0 — фреймворк делегованої авторизації. "Дозволи GitHub читати мій профіль" — це OAuth.

```
Flows (Grant Types):
┌─────────────────────────────────────────────────────────┐
│ Authorization Code + PKCE  ← Веб і мобільні додатки   │
│ Client Credentials         ← Server-to-server (M2M)   │
│ Device Code                ← CLI tools, Smart TVs      │
└─────────────────────────────────────────────────────────┘

Authorization Code Flow (найпоширеніший):

User → App: "Увійти через GitHub"
App → GitHub: GET /authorize?client_id=xxx&redirect_uri=xxx&scope=read:user
GitHub → User: "Дозволити App читати ваш профіль?"
User → GitHub: Підтверджує
GitHub → App: redirect_uri?code=AUTH_CODE
App → GitHub: POST /access_token (code + client_secret)
GitHub → App: { "access_token": "gho_...", "scope": "read:user" }
App → GitHub API: GET /user (Authorization: Bearer gho_...)
```

**Scopes** — що саме дозволяємо:

```bash
# GitHub scopes:
# read:user   — читати профіль
# repo        — доступ до приватних репо
# write:org   — управляти організацією
# delete_repo — видаляти репо (НЕБЕЗПЕЧНО!)

# Принцип мінімальних привілеїв — запитуй тільки ті scopes що реально потрібні!
```

### 4.4 OpenID Connect (OIDC)

OIDC = OAuth 2.0 + Identity. Додає ID Token (JWT) з інформацією про юзера.

```bash
# Ендпоінти OIDC (Discovery Document):
curl https://accounts.google.com/.well-known/openid-configuration | python3 -m json.tool

# Результат містить:
# authorization_endpoint, token_endpoint, userinfo_endpoint, jwks_uri

# GitHub OIDC (для GitHub Actions → AWS без секретів!):
# .github/workflows/deploy.yml
# permissions:
#   id-token: write
# - uses: aws-actions/configure-aws-credentials@v4
#   with:
#     role-to-assume: arn:aws:iam::123456789:role/github-actions
#     aws-region: us-east-1
# Це те що ти вже робив у Тижні 5!
```

### 4.5 mTLS (Mutual TLS)

Обидві сторони (клієнт і сервер) мають сертифікати і верифікують один одного.

```
Звичайний TLS: клієнт верифікує сервер (HTTPS)
mTLS: обидва верифікують один одного

Де використовують: service mesh (Istio, Linkerd), внутрішня мікросервісна комунікація
```

---

## 5. Формати даних: JSON, XML, Protobuf

### 5.1 JSON (JavaScript Object Notation)

Стандарт де-факто для REST API.

```json
{
  "id": 42,
  "name": "Lota",
  "active": true,
  "score": 98.5,
  "tags": ["devops", "python"],
  "address": {
    "city": "Kyiv",
    "country": "UA"
  },
  "metadata": null
}
```

```bash
# jq — обов'язковий інструмент для роботи з JSON в CLI
sudo apt install jq

# Основні операції
curl -s https://api.github.com/users/octocat | jq '.'           # pretty print
curl -s https://api.github.com/users/octocat | jq '.login'      # поле
curl -s https://api.github.com/users/octocat | jq '{name: .name, repos: .public_repos}'  # projection
curl -s 'https://api.github.com/repos/torvalds/linux/issues?per_page=5' | jq '.[].title'  # масив
curl -s 'https://api.github.com/repos/torvalds/linux/issues' | jq '[.[] | {title: .title, user: .user.login}]'
curl -s 'https://api.github.com/orgs/docker/repos' | jq 'sort_by(.stargazers_count) | reverse | .[0:3] | .[].full_name'

# Фільтрація
echo '[{"name":"nginx","active":true},{"name":"apache","active":false}]' | \
  jq '[.[] | select(.active == true)]'
```

### 5.2 XML

Старший формат, все ще в enterprise (SOAP, AWS деякі сервіси, конфіги).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<user id="42">
  <name>Lota</name>
  <active>true</active>
  <tags>
    <tag>devops</tag>
    <tag>python</tag>
  </tags>
</user>
```

```bash
# xmllint — для роботи з XML
sudo apt install libxml2-utils
curl -s https://some-api.com/data.xml | xmllint --format -
```

### 5.3 Protocol Buffers (Protobuf)

Бінарний формат від Google. Компактніший і швидший за JSON. Використовується з gRPC.

```protobuf
// user.proto — схема (треба скомпілювати)
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  bool active = 3;
  repeated string tags = 4;
}
```

```
JSON:  {"id":42,"name":"Lota","active":true} → ~35 bytes (text)
Proto: binary encoding                        → ~10 bytes (binary)

Результат: ~3x менше + набагато швидший серіалізація/десеріалізація
Ціна: не читається людиною, потрібна схема
```

---

## 6. Версіонування API

**Навіщо:** API мінятися — неминуче. Клієнти не оновлюються одночасно з сервером. Версіонування дозволяє робити breaking changes безпечно.

**Breaking change** — зміна що ламає існуючих клієнтів:
- Видалення поля з відповіді
- Зміна типу поля
- Зміна структури URL
- Нові обов'язкові поля в запиті

**Non-breaking change** — можна без нової версії:
- Додавання нових необов'язкових полів
- Нові endpoint-и
- Нові query params

### Стратегії версіонування:

```bash
# 1. URL Path (найпоширеніший, найзрозуміліший)
curl https://api.example.com/v1/users
curl https://api.example.com/v2/users

# 2. Query Parameter
curl "https://api.example.com/users?version=2"

# 3. Header (чистіший URL, але менш зрозумілий)
curl -H "API-Version: 2" https://api.example.com/users
curl -H "Accept: application/vnd.example.v2+json" https://api.example.com/users

# 4. Subdomain
curl https://v2.api.example.com/users
```

**Рекомендація:** URL Path (`/v1/`, `/v2/`) — для публічних API. Header — для внутрішніх.

**Стратегія підтримки версій:**

```
v1 → Deprecated (попереджуй в headers: Deprecation: true, Sunset: <date>)
v2 → Current
v3 → Beta/Preview

Мінімум: підтримуй N і N-1 версії одночасно.
Sunset header повідомляє клієнтів коли версія буде вимкнена.
```

---

## 7. Rate Limiting та Throttling

**Rate Limiting** — обмеження кількості запитів від клієнта за одиницю часу. Захищає від перевантаження, abuse і DDoS.

### Алгоритми:

```
Token Bucket (найпоширеніший):
  - Відро наповнюється токенами зі швидкістю R tokens/sec
  - Кожен запит споживає 1 токен
  - Якщо токени є — пропускаємо, якщо ні — 429
  - Дозволяє burst (швидкий пік), потім throttle

Leaky Bucket:
  - Запити виходять з постійною швидкістю (smooth output)
  - Зайві — в чергу або відкидаються
  - Добре для downstream захисту

Fixed Window:
  - N запитів за фіксоване вікно (наприклад, 1000/год)
  - Проблема: 2000 запитів в момент зміни вікна (edge case)

Sliding Window:
  - Рахує запити за поточне ковзне вікно
  - Рівномірніший, але складніший у реалізації
```

```bash
# Як бачити rate limit статус — заголовки відповіді:
curl -sI https://api.github.com/users/octocat | grep -i ratelimit
# x-ratelimit-limit: 60
# x-ratelimit-remaining: 59
# x-ratelimit-reset: 1705312800   ← unix timestamp коли скинеться

# Якщо 429:
curl -sI https://api.github.com/users/octocat | grep -i retry
# Retry-After: 3600   ← секунди до наступної спроби
```

```python
# Правильна обробка rate limit в клієнті:
import time
import requests

def api_request_with_retry(url, headers, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)

        if response.status_code == 429:
            retry_after = int(response.headers.get('Retry-After', 60))
            print(f"Rate limited. Waiting {retry_after}s...")
            time.sleep(retry_after)
            continue

        if response.status_code >= 500:
            wait = 2 ** attempt  # exponential backoff: 1, 2, 4 секунди
            print(f"Server error. Retrying in {wait}s...")
            time.sleep(wait)
            continue

        return response

    raise Exception("Max retries exceeded")
```

---

## 8. OpenAPI / Swagger — документація

**OpenAPI Specification (OAS)** — стандарт для опису REST API в YAML/JSON. Swagger — екосистема інструментів навколо OAS.

**Навіщо:** Документація що завжди актуальна, генерація клієнтів, автоматичне тестування, mock servers.

```yaml
# openapi.yaml — приклад специфікації
openapi: 3.1.0
info:
  title: DevOps Platform API
  version: 1.0.0
  description: API для управління deployment конфігураціями

servers:
  - url: https://api.devops-platform.com/v1
    description: Production
  - url: http://localhost:5000/api/v1
    description: Local development

paths:
  /deployments:
    get:
      summary: Список deployments
      tags: [Deployments]
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, running, success, failed]
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        '200':
          description: Список deployments
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/DeploymentList'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: Створити deployment
      tags: [Deployments]
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDeploymentRequest'
      responses:
        '201':
          description: Deployment створено
          headers:
            Location:
              schema:
                type: string
              description: URL нового deployment
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Deployment'
        '422':
          $ref: '#/components/responses/ValidationError'

  /deployments/{id}:
    get:
      summary: Отримати deployment за ID
      tags: [Deployments]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Deployment'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Deployment:
      type: object
      properties:
        id:
          type: integer
          example: 42
        service:
          type: string
          example: user-service
        version:
          type: string
          example: v1.2.3
        status:
          type: string
          enum: [pending, running, success, failed]
        created_at:
          type: string
          format: date-time
      required: [id, service, version, status, created_at]

    CreateDeploymentRequest:
      type: object
      properties:
        service:
          type: string
          minLength: 1
          maxLength: 100
        version:
          type: string
          pattern: '^v\d+\.\d+\.\d+$'
        environment:
          type: string
          enum: [dev, staging, production]
      required: [service, version, environment]

    DeploymentList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Deployment'
        pagination:
          $ref: '#/components/schemas/Pagination'

    Pagination:
      type: object
      properties:
        page: { type: integer }
        per_page: { type: integer }
        total: { type: integer }

  responses:
    Unauthorized:
      description: Токен відсутній або недійсний
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    NotFound:
      description: Ресурс не знайдено
    ValidationError:
      description: Помилка валідації
    Error:
      content:
        application/json:
          schema:
            type: object
            properties:
              title: { type: string }
              status: { type: integer }
              detail: { type: string }
```

```bash
# Swagger UI — інтерактивна документація (локально)
docker run -p 8080:8080 \
  -e SWAGGER_JSON=/api/openapi.yaml \
  -v $(pwd)/openapi.yaml:/api/openapi.yaml \
  swaggerapi/swagger-ui

# Відкрий http://localhost:8080

# Генерація клієнта з OpenAPI специфікації
docker run --rm -v $(pwd):/local openapitools/openapi-generator-cli generate \
  -i /local/openapi.yaml \
  -g python \
  -o /local/client-python

# Валідація специфікації
npm install -g @stoplight/spectral-cli
spectral lint openapi.yaml
```

---

## 9. GraphQL — альтернатива REST

**Аналогія:** REST — меню в ресторані (фіксовані страви). GraphQL — шведський стіл (береш тільки що хочеш).

**Проблема REST яку вирішує GraphQL:**
- **Over-fetching:** `GET /users/42` повертає 30 полів, а тобі треба тільки `name` і `email`
- **Under-fetching:** щоб показати профіль юзера з його замовленнями треба 3 окремих запити

```graphql
# Один GraphQL запит замість трьох REST:
query GetUserWithOrders {
  user(id: 42) {
    name
    email
    orders(last: 5) {
      id
      total
      status
      items {
        product { name }
        quantity
      }
    }
  }
}
```

### GraphQL операції:

```graphql
# Query — читання (як GET в REST)
query {
  users(role: "admin") {
    id
    name
  }
}

# Mutation — запис (як POST/PUT/DELETE в REST)
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    createdAt
  }
}

# Subscription — real-time (як WebSocket)
subscription OnDeploymentUpdate($id: ID!) {
  deploymentUpdated(id: $id) {
    status
    progress
    logs
  }
}
```

```bash
# Тестування GraphQL через curl
curl -X POST https://api.github.com/graphql \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ viewer { login name repositories(first: 3) { nodes { name } } } }"
  }'

# GraphQL Introspection — документація схеми
curl -X POST https://api.github.com/graphql \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"query": "{ __schema { queryType { name } } }"}'
```

**REST vs GraphQL — коли що вибирати:**

| Критерій | REST | GraphQL |
|---------|------|---------|
| Простота | ✅ Простіший для початку | Потребує схему |
| Гнучкість запитів | Фіксовані відповіді | ✅ Точно що треба |
| Кешування | ✅ HTTP кеш (GET) | Складніше (POST) |
| File uploads | ✅ Просто | Потребує multipart |
| Інструментарій | ✅ Зрілий | Розвивається |
| Використання | Публічні API, CRUD | BFF, мобільні клієнти |

---

## 10. gRPC — high-performance RPC

**gRPC** — Remote Procedure Call фреймворк від Google. Використовує Protobuf + HTTP/2.

**Аналогія:** Замість того щоб описувати запит словами (HTTP/JSON), ти дзвониш напряму у функцію на віддаленому сервері.

```protobuf
// deployment.proto
syntax = "proto3";

service DeploymentService {
  rpc GetDeployment(GetDeploymentRequest) returns (Deployment);
  rpc CreateDeployment(CreateDeploymentRequest) returns (Deployment);
  rpc WatchDeployment(GetDeploymentRequest) returns (stream DeploymentEvent);
  // stream — server-side streaming (real-time updates!)
}

message GetDeploymentRequest {
  int32 id = 1;
}

message Deployment {
  int32 id = 1;
  string service = 2;
  string version = 3;
  Status status = 4;

  enum Status {
    PENDING = 0;
    RUNNING = 1;
    SUCCESS = 2;
    FAILED = 3;
  }
}
```

**Типи комунікації в gRPC:**

```
Unary:          Client → Single Request → Server → Single Response
Server Stream:  Client → Single Request → Server → Stream of Responses
Client Stream:  Client → Stream of Requests → Server → Single Response
Bidirectional:  Client ⟷ Stream ⟷ Server
```

```bash
# grpcurl — curl для gRPC
sudo apt install grpcurl

# List services
grpcurl -plaintext localhost:50051 list

# Describe service
grpcurl -plaintext localhost:50051 describe DeploymentService

# Call method
grpcurl -plaintext -d '{"id": 42}' localhost:50051 DeploymentService/GetDeployment
```

**Де gRPC зустрінеш у DevOps:**
- Kubernetes internal API (etcd, kubelet, controller-manager спілкуються через gRPC)
- Istio service mesh
- Prometheus remote write (підтримує gRPC)
- Terraform Cloud API (частково)

---

## 11. Webhooks

**Webhook** — "зворотній API". Замість того щоб клієнт постійно питав "чи є нові події?" (polling), сервер сам надсилає HTTP запит на твій endpoint коли щось відбулося.

```
Polling (погано):
Client → GET /events (нічого нового) — кожні 5 секунд
Client → GET /events (нічого нового)
Client → GET /events (є подія!) ← можна пропустити або із затримкою

Webhook (добре):
GitHub → POST /webhooks/github {"event": "push", "repo": "myapp"} ← миттєво
```

**Реальні приклади:**
- GitHub `push` → GitHub Actions pipeline
- Stripe `payment.success` → твій backend
- PagerDuty alert → Slack повідомлення
- AWS SNS → Lambda функція

```python
# Приклад Flask webhook handler:
import hmac
import hashlib
from flask import Flask, request, abort

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

@app.route('/webhooks/github', methods=['POST'])
def github_webhook():
    # 1. Верифікація підпису (обов'язково!)
    signature = request.headers.get('X-Hub-Signature-256', '')
    expected = 'sha256=' + hmac.new(
        WEBHOOK_SECRET.encode(),
        request.data,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        abort(401, "Invalid signature")

    # 2. Обробка події
    event = request.headers.get('X-GitHub-Event')
    payload = request.json

    if event == 'push':
        branch = payload['ref'].split('/')[-1]
        commit = payload['head_commit']['id'][:7]
        print(f"Push to {branch}: {commit}")
        # Тригер deployment тощо

    elif event == 'pull_request':
        action = payload['action']  # opened, closed, merged
        pr_title = payload['pull_request']['title']
        print(f"PR {action}: {pr_title}")

    # 3. Завжди повертай 200 швидко!
    # Важку обробку — в background task (Celery, RQ)
    return {'status': 'ok'}, 200
```

**Важливо для webhooks:**
- Завжди верифікуй підпис (HMAC secret)
- Повертай `200 OK` швидко (< 5 секунд), важке в background
- Реалізуй idempotency (webhook може прийти двічі)
- Логуй всі вхідні webhook події

```bash
# Тестування webhooks локально — ngrok
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo apt-key add -
sudo apt install ngrok
ngrok http 5000    # дає публічний URL що тунелює до localhost:5000
# Використай URL в GitHub webhook settings
```

---

## 12. API Gateway

**API Gateway** — єдина точка входу для всіх клієнтів до мікросервісів. Не треба клієнту знати де кожен сервіс.

```
Клієнти                    API Gateway              Мікросервіси
                    ┌──────────────────────┐
Browser ───────────→│  • Auth (JWT verify) │──→ /users → User Service :3001
Mobile App ────────→│  • Rate Limiting     │──→ /orders → Order Service :3002
3rd party ─────────→│  • SSL Termination   │──→ /payments → Payment Service :3003
                    │  • Request routing   │──→ /products → Product Service :3004
                    │  • Logging           │
                    │  • Load Balancing    │
                    └──────────────────────┘
```

**Популярні API Gateway:**
- **AWS API Gateway** — serverless, інтеграція з Lambda
- **Kong** — open source, Nginx-based, дуже гнучкий
- **Nginx** — сам по собі виконує роль API Gateway
- **Traefik** — популярний для Kubernetes
- **AWS ALB** — простіший варіант для AWS мікросервісів

```nginx
# Nginx як API Gateway:
upstream user_service {
    server user-service:3001;
}

upstream order_service {
    server order-service:3002;
}

server {
    listen 80;

    # Auth middleware — перевірка JWT на рівні nginx
    # (через lua або окремий auth сервіс)

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;

    location /api/v1/users {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://user_service;
        proxy_set_header X-Request-ID $request_id;
    }

    location /api/v1/orders {
        limit_req zone=api burst=10 nodelay;
        proxy_pass http://order_service;
        proxy_set_header X-Request-ID $request_id;
    }
}
```

> 🏗️ **Capstone зв'язок:** Nginx у твоєму capstone проекті виконує роль API Gateway — SSL termination, rate limiting, routing до Flask app. Додамо цю конфігурацію до `./nginx/` директорії.

---

## 13. Практичні проекти

### Задача 1 (2 год): Дослідження реальних API через curl та jq

> 💡 **Навіщо:** перш ніж будувати власне API — навчись читати чуже. GitHub API — чудовий приклад well-designed REST API.

```bash
# Налаштуй GitHub token
export GITHUB_TOKEN="ghp_ваш_token"
export GH_API="https://api.github.com"

# 1. Переглянь свій профіль
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" $GH_API/user | jq '{
  login: .login,
  name: .name,
  public_repos: .public_repos,
  followers: .followers
}'

# 2. Переглянь свої репозиторії
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "$GH_API/user/repos?sort=updated&per_page=5" | \
  jq '.[] | {name: .name, stars: .stargazers_count, language: .language}'

# 3. Переглянь rate limit статус
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" $GH_API/rate_limit | \
  jq '.resources.core | {limit, remaining, reset: (.reset | todate)}'

# 4. Знайди всі відкриті issues у репозиторії
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "$GH_API/repos/ansible/ansible/issues?state=open&per_page=5" | \
  jq '.[] | {number: .number, title: .title, author: .user.login}'

# 5. Створи gist через API
curl -s -X POST -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  $GH_API/gists \
  -d '{
    "description": "API test gist",
    "public": false,
    "files": {
      "hello.py": {
        "content": "print(\"Hello from API!\")"
      }
    }
  }' | jq '{id: .id, url: .html_url}'

# 6. Подивись headers відповіді (rate limit, cache info)
curl -sI -H "Authorization: Bearer $GITHUB_TOKEN" $GH_API/user
```

✅ **Перевірка:** ти отримав JSON відповіді, сформував `jq` фільтри, побачив rate limit headers

---

### Задача 2 (3 год): REST API на Flask з правильними практиками

> 💡 **Навіщо:** цей Flask додаток — основа capstone проекту. Будуємо його правильно з самого початку.

```bash
# Структура проекту
mkdir -p flask-api/{app,tests}
cd flask-api

pip install flask flask-jwt-extended marshmallow flasgger --break-system-packages
```

```python
# app/__init__.py
from flask import Flask
from .config import Config

def create_app(config=None):
    app = Flask(__name__)
    app.config.from_object(config or Config)

    from .routes import api
    app.register_blueprint(api, url_prefix='/api/v1')

    return app
```

```python
# app/config.py
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-change-in-production')
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY', 'jwt-secret-change-me')
    DEBUG = os.environ.get('DEBUG', 'false').lower() == 'true'
```

```python
# app/routes.py
from flask import Blueprint, request, jsonify
from datetime import datetime, timezone

api = Blueprint('api', __name__)

# In-memory сховище (в production — база даних)
deployments = {}
_next_id = 1

def make_deployment(service, version, environment):
    global _next_id
    dep = {
        "id": _next_id,
        "service": service,
        "version": version,
        "environment": environment,
        "status": "pending",
        "created_at": datetime.now(timezone.utc).isoformat(),
        "updated_at": datetime.now(timezone.utc).isoformat(),
    }
    deployments[_next_id] = dep
    _next_id += 1
    return dep

# ── Health check ─────────────────────────────────────────────────────
@api.route('/health', methods=['GET'])
def health():
    """Health check endpoint для load balancers."""
    return jsonify({
        "status": "healthy",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "version": "1.0.0"
    }), 200

# ── Deployments CRUD ─────────────────────────────────────────────────
@api.route('/deployments', methods=['GET'])
def list_deployments():
    status_filter = request.args.get('status')
    env_filter = request.args.get('environment')
    page = int(request.args.get('page', 1))
    per_page = min(int(request.args.get('per_page', 20)), 100)  # max 100

    items = list(deployments.values())

    if status_filter:
        items = [d for d in items if d['status'] == status_filter]
    if env_filter:
        items = [d for d in items if d['environment'] == env_filter]

    total = len(items)
    start = (page - 1) * per_page
    paginated = items[start:start + per_page]

    return jsonify({
        "data": paginated,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": (total + per_page - 1) // per_page
        }
    }), 200

@api.route('/deployments', methods=['POST'])
def create_deployment():
    data = request.get_json(silent=True)
    if not data:
        return jsonify({
            "title": "Bad Request",
            "status": 400,
            "detail": "Request body must be valid JSON"
        }), 400

    errors = []
    if not data.get('service'):
        errors.append({"field": "service", "message": "Required field"})
    if not data.get('version'):
        errors.append({"field": "version", "message": "Required field"})
    if data.get('environment') not in ('dev', 'staging', 'production'):
        errors.append({"field": "environment", "message": "Must be dev, staging, or production"})

    if errors:
        return jsonify({
            "title": "Validation Error",
            "status": 422,
            "detail": "Request contains invalid fields",
            "errors": errors
        }), 422

    dep = make_deployment(data['service'], data['version'], data['environment'])

    response = jsonify(dep)
    response.status_code = 201
    response.headers['Location'] = f"/api/v1/deployments/{dep['id']}"
    return response

@api.route('/deployments/<int:dep_id>', methods=['GET'])
def get_deployment(dep_id):
    dep = deployments.get(dep_id)
    if not dep:
        return jsonify({
            "title": "Not Found",
            "status": 404,
            "detail": f"Deployment {dep_id} not found"
        }), 404
    return jsonify(dep), 200

@api.route('/deployments/<int:dep_id>', methods=['PATCH'])
def update_deployment_status(dep_id):
    dep = deployments.get(dep_id)
    if not dep:
        return jsonify({"title": "Not Found", "status": 404}), 404

    data = request.get_json(silent=True) or {}
    allowed_statuses = ('pending', 'running', 'success', 'failed')

    if 'status' in data:
        if data['status'] not in allowed_statuses:
            return jsonify({
                "title": "Validation Error",
                "status": 422,
                "errors": [{"field": "status", "message": f"Must be one of: {allowed_statuses}"}]
            }), 422
        dep['status'] = data['status']
        dep['updated_at'] = datetime.now(timezone.utc).isoformat()

    return jsonify(dep), 200

@api.route('/deployments/<int:dep_id>', methods=['DELETE'])
def delete_deployment(dep_id):
    if dep_id not in deployments:
        return jsonify({"title": "Not Found", "status": 404}), 404
    del deployments[dep_id]
    return '', 204

# ── Error handlers ─────────────────────────────────────────────────
@api.app_errorhandler(404)
def not_found(e):
    return jsonify({"title": "Not Found", "status": 404, "detail": str(e)}), 404

@api.app_errorhandler(405)
def method_not_allowed(e):
    return jsonify({"title": "Method Not Allowed", "status": 405}), 405

@api.app_errorhandler(500)
def internal_error(e):
    return jsonify({"title": "Internal Server Error", "status": 500}), 500
```

```python
# run.py
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

```bash
# Запуск
python3 run.py

# Тестування через curl
BASE="http://localhost:5000/api/v1"

# Health check
curl -s $BASE/health | jq .

# Створення deployment
curl -s -X POST $BASE/deployments \
  -H "Content-Type: application/json" \
  -d '{"service":"user-service","version":"v1.2.3","environment":"staging"}' | jq .

# Список deployments
curl -s $BASE/deployments | jq '.data | length'

# Фільтрація
curl -s "$BASE/deployments?status=pending&environment=staging" | jq .

# Оновлення статусу
curl -s -X PATCH $BASE/deployments/1 \
  -H "Content-Type: application/json" \
  -d '{"status":"running"}' | jq .status

# Отримати конкретний
curl -s $BASE/deployments/1 | jq .

# Видалення
curl -s -X DELETE $BASE/deployments/1 -w "%{http_code}"

# Перевірка помилок — неправильні дані
curl -s -X POST $BASE/deployments \
  -H "Content-Type: application/json" \
  -d '{"service":""}' | jq .

# Неіснуючий ресурс
curl -s $BASE/deployments/999 | jq .
```

✅ **Перевірка:** всі CRUD операції повертають правильні status codes; помилки мають структурований формат; пагінація працює

---

### Задача 3 (2 год): Тестування API

> 💡 **Навіщо:** DevOps engineer повинен вміти тестувати API — і руками і автоматично. Це стане частиною CI pipeline.

```python
# tests/test_api.py
import pytest
import json
from app import create_app

@pytest.fixture
def client():
    app = create_app()
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

class TestHealthEndpoint:
    def test_health_returns_200(self, client):
        response = client.get('/api/v1/health')
        assert response.status_code == 200

    def test_health_returns_json(self, client):
        response = client.get('/api/v1/health')
        data = response.get_json()
        assert data['status'] == 'healthy'
        assert 'timestamp' in data

class TestDeployments:
    def _create_deployment(self, client, **kwargs):
        payload = {
            "service": "test-service",
            "version": "v1.0.0",
            "environment": "dev",
            **kwargs
        }
        return client.post(
            '/api/v1/deployments',
            data=json.dumps(payload),
            content_type='application/json'
        )

    def test_create_deployment_returns_201(self, client):
        response = self._create_deployment(client)
        assert response.status_code == 201

    def test_create_deployment_returns_location_header(self, client):
        response = self._create_deployment(client)
        assert 'Location' in response.headers

    def test_create_deployment_validates_required_fields(self, client):
        response = client.post(
            '/api/v1/deployments',
            data=json.dumps({}),
            content_type='application/json'
        )
        assert response.status_code == 422
        data = response.get_json()
        assert 'errors' in data
        field_names = [e['field'] for e in data['errors']]
        assert 'service' in field_names
        assert 'version' in field_names

    def test_create_deployment_validates_environment(self, client):
        response = self._create_deployment(client, environment='invalid')
        assert response.status_code == 422

    def test_list_deployments(self, client):
        self._create_deployment(client)
        self._create_deployment(client, service='other-service')

        response = client.get('/api/v1/deployments')
        assert response.status_code == 200
        data = response.get_json()
        assert 'data' in data
        assert 'pagination' in data
        assert data['pagination']['total'] >= 2

    def test_filter_by_status(self, client):
        self._create_deployment(client)  # status: pending

        response = client.get('/api/v1/deployments?status=running')
        data = response.get_json()
        # Новий deployment має статус pending, не running
        for dep in data['data']:
            assert dep['status'] == 'running'

    def test_get_nonexistent_deployment(self, client):
        response = client.get('/api/v1/deployments/99999')
        assert response.status_code == 404

    def test_update_deployment_status(self, client):
        create_resp = self._create_deployment(client)
        dep_id = create_resp.get_json()['id']

        response = client.patch(
            f'/api/v1/deployments/{dep_id}',
            data=json.dumps({"status": "running"}),
            content_type='application/json'
        )
        assert response.status_code == 200
        assert response.get_json()['status'] == 'running'

    def test_delete_deployment(self, client):
        create_resp = self._create_deployment(client)
        dep_id = create_resp.get_json()['id']

        response = client.delete(f'/api/v1/deployments/{dep_id}')
        assert response.status_code == 204

        get_response = client.get(f'/api/v1/deployments/{dep_id}')
        assert get_response.status_code == 404

    def test_delete_nonexistent_returns_404(self, client):
        response = client.delete('/api/v1/deployments/99999')
        assert response.status_code == 404
```

```bash
# Запуск тестів
pip install pytest --break-system-packages
pytest tests/ -v

# З покриттям
pip install pytest-cov --break-system-packages
pytest tests/ -v --cov=app --cov-report=term-missing

# Запуск конкретного тесту
pytest tests/test_api.py::TestDeployments::test_create_deployment_returns_201 -v
```

✅ **Перевірка:** всі тести зелені; coverage > 80% для `app/routes.py`

---

### Задача 4 (2 год): Rate Limiting та автоматична документація

> 💡 **Навіщо:** production API завжди має rate limiting; документація — не опція, а вимога.

```bash
pip install flask-limiter flasgger --break-system-packages
```

```python
# app/__init__.py — оновлений
from flask import Flask
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from flasgger import Swagger

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["200 per hour", "50 per minute"]
)

def create_app(config=None):
    app = Flask(__name__)
    app.config.from_object(config or Config)

    # Swagger UI конфігурація
    app.config['SWAGGER'] = {
        'title': 'DevOps Platform API',
        'version': '1.0.0',
        'uiversion': 3
    }

    limiter.init_app(app)
    Swagger(app)

    from .routes import api
    app.register_blueprint(api, url_prefix='/api/v1')

    return app
```

```python
# Додай в routes.py — rate limiting і docstrings для Swagger:
from app import limiter

@api.route('/deployments', methods=['POST'])
@limiter.limit("10 per minute")  # додатковий ліміт для POST
def create_deployment():
    """
    Створити новий deployment
    ---
    tags:
      - Deployments
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required: [service, version, environment]
            properties:
              service:
                type: string
                example: user-service
              version:
                type: string
                example: v1.2.3
              environment:
                type: string
                enum: [dev, staging, production]
    responses:
      201:
        description: Deployment created
      422:
        description: Validation error
      429:
        description: Rate limit exceeded
    """
    # ... існуючий код
```

```bash
# Перевір rate limiting
for i in {1..12}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
    http://localhost:5000/api/v1/deployments \
    -H "Content-Type: application/json" \
    -d '{"service":"test","version":"v1","environment":"dev"}')
  echo "Request $i: $STATUS"
done
# Запит 11 або 12 має повернути 429

# Swagger UI
# Відкрий http://localhost:5000/apidocs
```

✅ **Перевірка:** після 10 POST запитів за хвилину отримуєш `429 Too Many Requests`; Swagger UI доступний на `/apidocs`

---

### Задача 5 (1.5 год): Взаємодія з зовнішніми API — Python клієнт

> 💡 **Навіщо:** DevOps часто пише скрипти що взаємодіють з API (Slack, PagerDuty, AWS, GitHub). Ось правильний патерн.

```python
#!/usr/bin/env python3
# scripts/github_api_client.py
import os
import time
import requests
from datetime import datetime

class GitHubClient:
    """Клієнт для GitHub API з автоматичним retry та rate limit handling."""

    BASE_URL = "https://api.github.com"

    def __init__(self, token: str):
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github.v3+json",
            "X-GitHub-Api-Version": "2022-11-28"
        })

    def _request(self, method: str, path: str, **kwargs) -> requests.Response:
        url = f"{self.BASE_URL}{path}"

        for attempt in range(3):
            response = self.session.request(method, url, **kwargs)

            if response.status_code == 429:
                reset_time = int(response.headers.get('X-RateLimit-Reset', 0))
                wait = max(reset_time - int(time.time()), 1)
                print(f"Rate limited. Waiting {wait}s...")
                time.sleep(wait)
                continue

            if response.status_code >= 500:
                wait = 2 ** attempt
                print(f"Server error {response.status_code}. Retry in {wait}s...")
                time.sleep(wait)
                continue

            response.raise_for_status()
            return response

        raise Exception(f"Failed after 3 attempts: {method} {path}")

    def get_repo_info(self, owner: str, repo: str) -> dict:
        return self._request('GET', f'/repos/{owner}/{repo}').json()

    def list_open_prs(self, owner: str, repo: str) -> list:
        return self._request('GET', f'/repos/{owner}/{repo}/pulls',
                            params={'state': 'open', 'per_page': 50}).json()

    def get_rate_limit(self) -> dict:
        data = self._request('GET', '/rate_limit').json()
        core = data['resources']['core']
        return {
            'limit': core['limit'],
            'remaining': core['remaining'],
            'resets_at': datetime.fromtimestamp(core['reset']).isoformat()
        }

    def create_issue(self, owner: str, repo: str, title: str, body: str) -> dict:
        return self._request('POST', f'/repos/{owner}/{repo}/issues',
                            json={'title': title, 'body': body}).json()


def main():
    token = os.environ.get('GITHUB_TOKEN')
    if not token:
        raise ValueError("GITHUB_TOKEN environment variable is required")

    client = GitHubClient(token)

    # Rate limit статус
    rate = client.get_rate_limit()
    print(f"Rate limit: {rate['remaining']}/{rate['limit']}, resets at {rate['resets_at']}")

    # Інформація про репозиторій
    repo = client.get_repo_info('docker', 'compose')
    print(f"\nRepository: {repo['full_name']}")
    print(f"Stars: {repo['stargazers_count']}")
    print(f"Open Issues: {repo['open_issues_count']}")
    print(f"Language: {repo['language']}")

    # Відкриті PR
    prs = client.list_open_prs('docker', 'compose')
    print(f"\nOpen PRs: {len(prs)}")
    for pr in prs[:3]:
        print(f"  #{pr['number']}: {pr['title']} (@{pr['user']['login']})")


if __name__ == '__main__':
    main()
```

```bash
export GITHUB_TOKEN="ghp_ваш_токен"
python3 scripts/github_api_client.py
```

✅ **Перевірка:** скрипт виводить інформацію про репозиторій і PR без помилок; rate limit інформація коректна

---

## Типові помилки

| Симптом | Причина | Як виправити |
|---------|---------|--------------|
| `401 Unauthorized` | Немає або неправильний токен | Перевір `Authorization` header; токен має `Bearer ` prefix |
| `403 Forbidden` | Токен є але немає прав | Перевір scopes/permissions токена; для GitHub — чи є доступ до ресурсу |
| `404` на правильному URL | Ресурс видалено або у версії API | Перевір версію в URL (`/v1/` vs `/v2/`); можливо ресурс не існує |
| `415 Unsupported Media Type` | Забув `Content-Type: application/json` | Додай header `-H "Content-Type: application/json"` |
| `422` замість `400` | Структура JSON правильна, але дані некоректні | Перевір enum значення, формати дат, обов'язкові поля |
| JWT `expired signature` | Токен прострочений | Refresh access token через refresh token endpoint |
| JWT `invalid signature` | Неправильний secret key | Переконайся що JWT_SECRET_KEY однаковий між сервісами |
| `429` в prod | Перевищений rate limit | Додай `Retry-After` logic; збільш інтервал між запитами; кешуй відповіді |
| Webhook не отримуємо | Неправильна URL або firewall | Перевір `ngrok` чи прийшов запит; перевір signature verification (не відкидає?) |
| `502 Bad Gateway` | Nginx не може достукатись до upstream | Flask/gunicorn не запущений; неправильний порт у `proxy_pass` |
| `CORS` помилка в браузері | Сервер не повертає `Access-Control-Allow-Origin` | Додай `flask-cors` або CORS headers у nginx |

---

## Результат модуля

Після завершення ти повинен мати:

- [ ] Впевнене використання `curl` і `jq` для роботи з будь-яким API
- [ ] Flask REST API з правильними status codes, пагінацією та error handling
- [ ] Автоматичні тести з coverage > 80%
- [ ] Rate limiting налаштований через flask-limiter
- [ ] Swagger UI документація на `/apidocs`
- [ ] Python API клієнт з retry та rate limit handling
- [ ] Розуміння різниці між REST, GraphQL, gRPC і Webhook

**GitHub deliverable:** репо `flask-api/` з:
```
flask-api/
├── app/
│   ├── __init__.py
│   ├── config.py
│   └── routes.py
├── tests/
│   └── test_api.py
├── scripts/
│   └── github_api_client.py
├── openapi.yaml
├── requirements.txt
└── README.md   ← з прикладами curl команд
```

> 🏗️ **Capstone зв'язок:** Flask API з цього модуля стане `app/` частиною capstone проекту. Nginx config (Тиждень 3) додасть до нього rate limiting та SSL. GitHub Actions (Тиждень 1, вже зроблено!) буде деплоїти його автоматично.

---

## Interview Prep

**Питання які тобі зададуть:**

| Питання | Де ти це робив | Ключові слова відповіді |
|---------|---------------|------------------------|
| Різниця між `401` і `403`? | Розділ 2.2 + Задача 2 | authN vs authZ, токен є vs прав немає |
| Що таке idempotency і які методи idempotent? | Розділ 2.1 | GET/PUT/DELETE idempotent, POST ні, повторний виклик = той самий результат |
| Чим JWT кращий за session-based auth? | Розділ 4.2 | stateless, horizontal scaling, self-contained, microservices |
| Як захистити API від abuse? | Розділи 7, 4 | rate limiting (token bucket), API keys, JWT expiry, IP filtering |
| REST vs GraphQL — коли що? | Розділи 3, 9 | over/under-fetching, caching, flexibility vs simplicity |
| Що таке Webhook і як верифікувати? | Розділ 11 + Задача 2 | event-driven, HMAC signature, `X-Hub-Signature-256` |
| Навіщо API versioning? | Розділ 6 | breaking changes, backward compatibility, sunset policy |
| Як ти тестуєш API? | Задача 3 | pytest, curl, status codes, error cases, boundary conditions |
| Що перевіряєш коли бачиш `502 Bad Gateway`? | Типові помилки | upstream сервіс запущений?, правильний порт?, логи nginx і додатку |
| Що таке OIDC і де використовують? | Розділ 4.4 | OAuth2 + identity, GitHub Actions → AWS без секретів, JWT ID token |

**Питання які задай ТИ:**
- "Які API стандарти і конвенції прийняті у вашій команді для нових сервісів?"
- "Як ви документуєте внутрішні API — OpenAPI специфікація чи щось інше?"
- "Як влаштований API Gateway у вашій інфраструктурі?"

**Ресурс:** [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) · [JWT.io](https://jwt.io) — декодування токенів · [OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0) — офіційна специфікація

---

*Версія модуля: 1.0 · Охоплює: HTTP, REST, Auth, Formats, Versioning, Rate Limiting, OpenAPI, GraphQL, gRPC, Webhooks, API Gateway*
