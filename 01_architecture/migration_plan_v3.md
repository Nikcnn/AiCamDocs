# План миграции проекта: Переход к целевой архитектуре

Java Spring Boot — главное backend-ядро системы. Python (DRF) — только web/AI слой для AI Tutor и Telegram webhook. Qt/C++ монолит остается, меняются только протоколы связи и база данных.

---

## 1. Архитектурные и инфраструктурные изменения

### 1.1. Переход на монорепозиторий и контейнеризацию
- Объединить все компоненты в единый репозиторий (`project-root`).
- Написать `Dockerfile` для каждого сервиса (`backend`, `java-backend`, `ai-server`, `cv-service`, `telegram-bot`, `frontend`, `admin-panel`).
- Создать `docker-compose.yml` (для локальной разработки) и `docker-compose.prod.yml` (для продакшена) в корне проекта.
- Все сервисы общаются внутри единой изолированной Docker-сети по именам контейнеров.

### 1.2. Внедрение Nginx (Reverse Proxy)
- Nginx выступает единой точкой входа для всех внешних запросов.
- Маршрутизация:
  - `/` → `frontend` (React)
  - `/admin/*` → `admin-panel` (React)
  - `/core/api/*` → `java-backend:8080` (Главный Core API)
  - `/api/v1/*` → `backend:8000` (Python Web / AI Tutor API)
  - `/telegram/webhook` → `backend:8000`
- SSL-терминация на уровне Nginx.

### 1.3. Переход на PostgreSQL
- Отказ от MSSQL во всех сервисах в пользу PostgreSQL 16.
- Миграции схемы БД управляются через **Flyway или Liquibase** в `java-backend`.
- Структура таблиц описана в `db_schema.md`.

---

## 2. Java Backend (Главный Core-сервис)

### 2.1. Изменение зоны ответственности
- **Удалить:**
  - Пакет `com.schedule.server.tcp` (`CppTcpClient`, `SubjectTcpSender`) — прямые TCP-сокеты с C++ сервером больше не нужны.
  - Логику Telegram-бота (`BotBridgeService`, `BotBridgeController`) — эти функции переходят в связку Python `backend` + `telegram-bot`.
- **Оставить и расширить:**
  - `ScheduleService`, `SubjectService` — полноценная работа с расписанием и предметами.
  - `ScheduleController` — загрузка Excel, управление аудиториями.
  - `GlobalExceptionHandler`, `RequestLoggingFilter` — без изменений.

### 2.2. Изменение работы с БД
- Заменить драйвер `com.microsoft.sqlserver.jdbc.SQLServerDriver` на `org.postgresql:postgresql`.
- Обновить JPA-сущности под новую схему (`users`, `groups`, `schedule`, `spaces`, `bookings`, `licenses`).
- `ScheduleService` при загрузке Excel опирается на уникальный составной индекс `(room, weekday, time_start)` для детектирования коллизий.
- Включить Flyway или Liquibase для управления миграциями.

### 2.3. Авторизация
- Реализовать генерацию и валидацию JWT-токенов в Java Core.
- Все эндпоинты защищены через `Authorization: Bearer <token>`.

---

## 3. Python Backend (Web / AI Tutor слой)

Python-сервис (FastAPI) — специализированный web/AI модуль, не core.

### 3.1. Зона ответственности
- **AI Tutor:** Управление сессиями оценивания (`tutor_sessions`, `tutor_messages`). Логика дерева решений (`StrategySelector`) и стратегии Блума (clarify, feynman, socratic, debate, advance). Связь с `ai-server` по gRPC.
- **Telegram Webhook:** Прием и обработка апдейтов от Telegram-бота.
- **Делегирование в Java:** Для получения данных о пользователях, группах, расписании и доступе Python делает HTTP REST запросы к `java-backend` внутри Docker-сети.

### 3.2. Telegram Bot (aiogram)
- Удалить `UserStorage` и `users.json` — бот становится stateless.
- Все проверки броней, кулдаунов и ролей делегируются в API (Java или Python).

---

## 4. C++ / Qt (Минимальные изменения)

Qt-монолит **не нужно разбивать**. Qt GUI, QML, `AlgorithmManager`, `AuditoryFinder`, `CameraChecker`, `CameraWorker`, `TemporaryAuditoryCleaner` — всё это остается.

### 4.1. Замена протокола связи с Java (единственное жесткое требование)
- **Удалить:** бинарный `QDataStream` протокол (`Java2CPacket`, `C2JavaPacket`) и прямые TCP-сокеты `QTcpServer`/`QTcpSocket` для связи с Java-клиентами.
- **Заменить на:** HTTP REST или gRPC. Qt-сервер должен отвечать на запросы от Java через стандартные протоколы вместо бинарных пакетов.

### 4.2. Переход с MSSQL на PostgreSQL
- В `DatabaseConnector` заменить ODBC-строку подключения к SQL Server на PostgreSQL.
- Адаптировать запросы в `AlgorithmRequests` под синтаксис PostgreSQL (при наличии различий).
- Пул подключений `cameraDatabase` (используемый в `CameraChecker`) тоже переключить на PostgreSQL.

### 4.3. `ai-server` (C++ / gRPC — новый сервис)
- Вынести логику взаимодействия с LLM (OpenAI или лоакльная модель) в отдельный gRPC-сервер на порту `50051`.
- Реализовать контракты из `tutor.proto` (методы `EvaluateAnswer`, `GenerateQuestion`).
- `temperature=0.1` для оценки ответов, `temperature=0.75` для генерации вопросов.
- Это единственный новый C++ сервис, который нужно создать с нуля.

### 4.4. ESP8266 (умные кабинеты)
- При желании сохранить интеграцию: заменить бинарный TCP-пакет (9 байт) на HTTP REST запрос к Python или Java backend.
- Логика `CabinetStorage` и веб-интерфейс ESP8266 остаются без изменений.

---

## 5. Новые клиентские интерфейсы (Frontend и Admin Panel)

Разработать два новых React-приложения:
- **Frontend (Web AI Tutor):**
  - Личный кабинет студента: запуск сессий AI Tutor (запросы идут в Python backend).
  - Кабинет преподавателя: просмотр карт понимания по уровням Блума (запросы идут в Java backend).
- **Admin Panel:**
  - Авторизация администраторов с принудительной сменой пароля (Java).
  - Управление лицензиями (AI Tutor, CV) (Java).
  - Управление белыми списками (whitelist) пользователей (Java).
  - Загрузка расписания через Excel (Java).
