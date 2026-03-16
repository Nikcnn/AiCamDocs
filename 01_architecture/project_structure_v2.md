# Project Structure

## Обзор

Проект организован как **polyrepo** — каждый сервис в отдельном репозитории. Вся инфраструктура, документация и контракты между сервисами хранятся в отдельном репозитории `infra`.

---

## Репозитории

| Репозиторий | Команда | Технология |
|---|---|---|
| `org/infra` | DevOps | Docker Compose, Nginx, docs |
| `org/java-backend` | Java-команда | Java / Spring Boot |
| `org/backend` | Python-команда | Python / FastAPI |
| `org/ai-server` | C++-команда | C++ / gRPC |
| `org/qt-server` | C++-команда | C++ / Qt |
| `org/telegram-bot` | Python-команда | Python / aiogram |
| `org/frontend` | React-команда | React |
| `org/admin-panel` | React-команда | React |

---

## org/infra — структура

Единая точка сборки и документации. DevOps-команда владеет этим репо.

```
infra/
│
├── docs/                               ← вся техническая документация
│   ├── 01_architecture/
│   │   ├── project_structure.md        ← этот файл
│   │   └── service_map.md
│   ├── 02_database/
│   │   ├── db_schema.md
│   │   └── erd_schema.xml
│   ├── 03_api/
│   │   └── api_spec.yaml
│   ├── 04_ai_logic/
│   │   ├── ai_evaluation_logic.md
│   │   └── ai_tutor_logic.py
│   ├── 05_infrastructure/
│   │   └── infrastructure.md
│   └── 06_flows/
│       ├── auth/
│       ├── tutor/
│       ├── bookings/
│       └── admin/
│
├── nginx/
│   └── nginx.conf
│
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .gitignore
└── README.md
```

---

## org/java-backend — структура

Java Spring Boot — главный Core-сервис.

```
java-backend/
├── src/
│   └── main/
│       ├── java/
│       │   ├── controllers/
│       │   ├── services/
│       │   ├── repositories/
│       │   ├── entities/
│       │   ├── security/
│       │   └── config/
│       └── resources/
│           └── db/migration/           ← Flyway/Liquibase migrations
├── Dockerfile
└── pom.xml
```

---

## org/backend — структура

Python FastAPI — только web/AI слой.

```
backend/
├── main.py
├── config.py
├── database.py
├── dependencies.py
├── models/
│   ├── session.py
│   ├── message.py
│   └── result.py
├── routers/
│   ├── tutor.py
│   ├── telegram.py
│   └── health.py
├── tutor/
│   ├── models.py
│   ├── prompts.py
│   ├── evaluator.py
│   ├── strategy_selector.py
│   ├── question_generator.py
│   └── session_manager.py
├── tests/
│   ├── test_tutor.py
│   └── test_telegram.py
├── Dockerfile
└── requirements.txt
```

---

## org/ai-server — структура

C++ gRPC сервер — взаимодействие с LLM.

```
ai-server/
├── src/
│   ├── main.cpp
│   ├── evaluator.cpp
│   └── generator.cpp
├── proto/
│   └── tutor.proto                     ← gRPC контракт (единый источник истины)
├── Dockerfile
└── CMakeLists.txt
```

---

## org/qt-server — структура

C++ / Qt монолит — камеры, YOLO, бронирование аудиторий.

```
qt-server/
├── src/
│   ├── main.cpp
│   ├── AlgorithmManager/
│   ├── AuditoryFinder/
│   ├── CameraWorker/
│   ├── CameraChecker/
│   ├── DatabaseConnector/
│   └── HttpServer/                     ← REST API (замена бинарного TCP)
├── Dockerfile
└── CMakeLists.txt
```

---

## org/telegram-bot — структура

```
telegram-bot/
├── main.py
├── handlers/
│   ├── booking.py
│   └── cooldown.py
├── keyboards.py
├── Dockerfile
└── requirements.txt
```

---

## org/frontend и org/admin-panel — структура

```
frontend/          admin-panel/
├── src/           ├── src/
│   ├── pages/     │   ├── pages/
│   ├── components/│   ├── components/
│   └── api/       │   └── api/
├── Dockerfile     ├── Dockerfile
└── package.json   └── package.json
```

---

## Зоны ответственности

| Сервис | Язык | Назначение |
|---|---|---|
| `java-backend` | Java / Spring Boot | Пользователи, группы, расписание, авторизация, лицензии |
| `backend` | Python / FastAPI | AI Tutor, Telegram webhook |
| `ai-server` | C++ / gRPC | Инференс LLM, оценка ответов по Блуму, генерация вопросов |
| `qt-server` | C++ / Qt | RTSP-камеры, YOLO, бронирование аудиторий |
| `telegram-bot` | Python / aiogram | Бронирование через Telegram |
| `frontend` | React | Кабинет студента и преподавателя |
| `admin-panel` | React | Управление пользователями, лицензиями |

---

## Контракты между репозиториями

Контракты хранятся в `infra/docs` и являются единственным источником истины. Менять можно только через PR в `infra`.

| Контракт | Файл | Затрагивает |
|---|---|---|
| REST API Java Core | `docs/03_api/api_spec.yaml` | java-backend, frontend, admin-panel, backend |
| gRPC контракт LLM | `ai-server/proto/tutor.proto` | ai-server, backend |
| Схема БД | `docs/02_database/db_schema.md` | java-backend, backend, qt-server |
| ENV-переменные | `infra/.env.example` | все сервисы |

---

## Порядок разработки

1. **Схема БД** — зафиксировать в `db_schema.md`, написать миграции в `java-backend`
2. **java-backend** — основная бизнес-логика, авторизация, Core API
3. **ai-server** — параллельно с backend, контракт зафиксирован в `tutor.proto`
4. **backend (Python)** — AI Tutor, подключается после java-backend и ai-server
5. **qt-server** — независим, пишет в БД напрямую
6. **telegram-bot** — после стабилизации backend
7. **frontend / admin-panel** — когда API стабилен

---

## Переменные окружения

Файл `infra/.env.example` коммитится как шаблон. Каждая команда копирует нужные переменные в свой `.env`. Сам `.env` — всегда в `.gitignore`.

```env
# Java Backend
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/db
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=
JWT_SECRET=

# Python Backend
DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/db
JAVA_BACKEND_URL=http://java-backend:8080
AI_SERVER_HOST=ai-server
AI_SERVER_PORT=50051

# AI Server
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o-mini
GRPC_PORT=50051

# Telegram
TELEGRAM_BOT_TOKEN=
TELEGRAM_WEBHOOK_URL=
BACKEND_URL=http://backend:8000
```

---

## Правила для команды

- Каждый репозиторий — независимый CI/CD пайплайн
- Ветки: `main` — продакшн, `dev` — разработка, `feature/название` — фичи
- Перед мержем в `dev` — запустить тесты своего сервиса
- Изменения в контрактах (API, proto, схема БД) — только через PR в `infra`, с уведомлением всех команд
- Миграции БД — только через Flyway/Liquibase в `java-backend`, руками в БД не лазить
- Промпты не хранить в БД — только в `backend/tutor/prompts.py`
