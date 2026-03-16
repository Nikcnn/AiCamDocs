# Service Map

## Обзор

Система состоит из 8 сервисов. Nginx принимает все внешние запросы и маршрутизирует их по сервисам. Внутри сети сервисы общаются напрямую по именам Docker-контейнеров.

Главным backend-ядром системы является `java-backend`: он отвечает за основную бизнес-логику, авторизацию, пользователей, группы, расписание, пространства, бронирования и лицензии.

Python `backend` — специализированный web/AI слой. Он обслуживает AI Tutor, Telegram webhook и интеграцию с `ai-server`. Бизнес-логику не дублирует — делегирует в Java Core.

---

## Схема взаимодействия

```text
                          [ Клиенты ]
          ┌─────────────┬──────────────┬────────────┐
       Браузер       Браузер        Telegram    RTSP-камеры
      (студент)  (препод/админ)    Bot API          │
          └──────────┬──────────────┘               │
                     ▼                               ▼
             [ Nginx / Reverse Proxy ]         [ Qt/C++ Server ]
                     │                               │
        ┌────────────┼───────────────┐               │
        ▼            ▼               ▼               │
   frontend      admin-panel      backend            │
   (React)        (React)        (Python)            │
        │             │               │              │
        └──────┬───────┘         ▼    ▼              ▼
               ▼            ai-server  PSG ◄───────┘
         java-backend         (C++)
           (Core)               │
               │                ▼
               │          LLM Provider
               ▼
             PSG
```

---

## Сервисы

| Сервис | Технология | Порт | Протокол | Назначение |
|---|---|---|---|---|
| `nginx` | Nginx | 80 / 443 | HTTP / HTTPS | Единая точка входа, reverse proxy, SSL termination |
| `frontend` | React | — | статика | Кабинет студента и преподавателя |
| `admin-panel` | React | — | статика | Панель администрирования |
| `java-backend` | Java / Spring Boot | 8080 | HTTP REST | Главный Core backend, вся бизнес-логика |
| `backend` | Python / FastAPI | 8000 | HTTP REST | Web/AI слой, AI Tutor, Telegram webhook |
| `ai-server` | C++ | 50051 | gRPC | Оценка ответов и генерация вопросов (LLM) |
| `qt-server` | C++ / Qt | — | HTTP REST | Бронирование аудиторий, мониторинг камер, YOLO |
| `telegram-bot` | Python / aiogram | — | Telegram Bot API | UI для бронирований через Telegram |
| `mssql` | Microsoft SQL Server | 1433 | TCP | Единая база данных |

---

## Маршруты Nginx

```text
/                      →  frontend (статика)
/admin/*               →  admin-panel (статика)
/core/api/*            →  java-backend:8080
/api/v1/*              →  backend:8000
/telegram/webhook      →  backend:8000
```

SSL-терминация происходит на уровне Nginx. Все внутренние запросы между сервисами — по HTTP без шифрования (внутренняя Docker-сеть).

---

## Связи между сервисами

### frontend / admin-panel → java-backend
- Протокол: HTTP REST
- Запросы на пользователей, группы, расписание, предметы, пространства, бронирования, лицензии
- Авторизация: JWT в заголовке `Authorization: Bearer <token>`

### frontend → backend
- Протокол: HTTP REST
- Используется только для сценариев AI Tutor (диалоговое оценивание)

### backend → java-backend
- Протокол: HTTP REST (внутренняя сеть)
- Python запрашивает у Java данные пользователей, групп и предметов
- Java остается единственным источником core бизнес-данных

### backend → ai-server
- Протокол: gRPC
- Контракт зафиксирован в `ai-server/proto/tutor.proto`
- Методы: `EvaluateAnswer`, `GenerateQuestion`

### java-backend → qt-server
- Протокол: HTTP REST (замена старого бинарного TCP QDataStream)
- Java отправляет запросы на поиск и бронирование аудиторий
- Qt-сервер возвращает JSON с результатом

### telegram-bot → backend
- Telegram отправляет апдейты на `POST /telegram/webhook`
- Python обрабатывает webhook и при необходимости обращается в java-backend

### qt-server → PostgreSQL
- Qt-сервер напрямую пишет в PostgreSQL через QSqlDatabase с драйвером QPSQL
- Таблицы: `auditory_journal`, аудитории, камеры, бронирования

### java-backend → PostgreSQL
- JPA / Hibernate через JDBC драйвер org.postgresql:postgresql
- Таблицы: users, groups, schedule, spaces, bookings, licenses

### backend → PostgreSQL
- SQLAlchemy через asyncpg (async) или psycopg2 (sync)
- Таблицы: tutor_sessions, tutor_messages, tutor_results

### ai-server → LLM Provider
- HTTP запросы к OpenAI API (или локальной модели)
- Только ai-server знает об LLM — остальные сервисы с LLM не общаются напрямую

---

## Доступ к PostgreSQL

| Сервис | Доступ | Таблицы |
|---|---|---|
| `java-backend` | Чтение / запись | users, groups, group_members, subjects, schedule, spaces, bookings, licenses |
| `backend` | Чтение / запись | tutor_sessions, tutor_messages, tutor_results |
| `backend` | Только чтение | users, groups, subjects (через Java API или прямой SELECT) |
| `qt-server` | Чтение / запись | auditories, auditory_journal, camera_journal |

---

## gRPC-контракт (tutor.proto)

```protobuf
syntax = "proto3";

package tutor;

service TutorService {
  rpc EvaluateAnswer (EvaluateRequest) returns (EvaluateResponse);
  rpc GenerateQuestion (GenerateRequest) returns (GenerateResponse);
}

message EvaluateRequest {
  string question = 1;
  string answer   = 2;
  string subject  = 3;
  string topic    = 4;
}

message EvaluateResponse {
  int32  bloom_level  = 1;
  float  confidence   = 2;
  string reasoning    = 3;
}

message GenerateRequest {
  string strategy       = 1;
  string subject        = 2;
  string topic          = 3;
  string last_answer    = 4;
  string dialog_history = 5;
  int32  current_bloom  = 6;
}

message GenerateResponse {
  string question = 1;
}
```

---

