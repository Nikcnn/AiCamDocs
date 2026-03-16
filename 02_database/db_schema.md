# Database Schema

## Обзор

Единая PostgreSQL 16 база данных для всех сервисов. Каждый сервис имеет доступ только к своим таблицам — см. `service_map.md`. Миграции управляются через Alembic (Python backend).

---

## Таблицы

### users

Единая таблица для всех типов пользователей системы.

```sql
CREATE TABLE users (
    id            SERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(20)  NOT NULL CHECK (role IN ('admin', 'teacher', 'student')),
    is_active     BOOLEAN      NOT NULL DEFAULT false,
    first_login   BOOLEAN      NOT NULL DEFAULT true,
    created_at    TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at    TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
```

| Поле | Тип | Описание |
|---|---|---|
| `id` | SERIAL | Первичный ключ |
| `email` | VARCHAR | Уникальный, индексируется |
| `password_hash` | VARCHAR | bcrypt, не plain |
| `role` | VARCHAR | admin / teacher / student |
| `is_active` | BOOLEAN | Аккаунт активирован через email |
| `first_login` | BOOLEAN | Сигнал для принудительной смены пароля (admin) |

---

### groups

Учебные группы. Привязывают студентов к преподавателям.

```sql
CREATE TABLE groups (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    teacher_id INT          NOT NULL REFERENCES users(id),
    created_at TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE INDEX idx_groups_teacher_id ON groups(teacher_id);
```

---

### group_members

Связка многие-ко-многим между студентами и группами.

```sql
CREATE TABLE group_members (
    group_id   INT       NOT NULL REFERENCES groups(id),
    student_id INT       NOT NULL REFERENCES users(id),
    joined_at  TIMESTAMP NOT NULL DEFAULT now(),
    PRIMARY KEY (group_id, student_id)
);

CREATE INDEX idx_group_members_student_id ON group_members(student_id);
```

---

### subjects

Справочник предметов.

```sql
CREATE TABLE subjects (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL UNIQUE,
    created_by INT          NOT NULL REFERENCES users(id),
    created_at TIMESTAMP    NOT NULL DEFAULT now()
);
```

---

### tutor_sessions

Основная таблица сессий оценивания. Один ряд — одна диалоговая сессия между студентом и AI.

```sql
CREATE TABLE tutor_sessions (
    id              SERIAL PRIMARY KEY,
    student_id      INT          NOT NULL REFERENCES users(id),
    subject_id      INT          NOT NULL REFERENCES subjects(id),
    topic           TEXT         NOT NULL,
    group_id        INT          REFERENCES groups(id),
    status          VARCHAR(20)  NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'completed', 'abandoned')),
    messages_count  INT          NOT NULL DEFAULT 0,
    consecutive_low INT          NOT NULL DEFAULT 0,
    last_strategy   VARCHAR(20)  CHECK (last_strategy IN
                        ('socratic', 'feynman', 'debate', 'clarify', 'advance')),
    bloom_scores    JSONB        NOT NULL DEFAULT '[]',
    started_at      TIMESTAMP    NOT NULL DEFAULT now(),
    completed_at    TIMESTAMP
);

CREATE INDEX idx_tutor_sessions_student_id ON tutor_sessions(student_id);
CREATE INDEX idx_tutor_sessions_group_id   ON tutor_sessions(group_id);
CREATE INDEX idx_tutor_sessions_status     ON tutor_sessions(status);
CREATE INDEX idx_tutor_sessions_cooldown   ON tutor_sessions(student_id, started_at DESC);
```

| Поле | Описание |
|---|---|
| `bloom_scores` | JSONB-массив всех оценок по порядку, например `[1, 2, 3, 3, 4]` |
| `consecutive_low` | Счётчик подряд идущих ответов уровня 1, сбрасывается при уровне >= 2 |
| `last_strategy` | Стратегия последнего вопроса — чтобы не повторять одну и ту же дважды |

---

### tutor_messages

Все сообщения диалога. Каждый вопрос AI и каждый ответ студента — отдельный ряд.

```sql
CREATE TABLE tutor_messages (
    id               SERIAL PRIMARY KEY,
    session_id       INT         NOT NULL REFERENCES tutor_sessions(id),
    role             VARCHAR(10) NOT NULL CHECK (role IN ('ai', 'student')),
    content          TEXT        NOT NULL,
    strategy         VARCHAR(20) CHECK (strategy IN
                         ('socratic', 'feynman', 'debate', 'clarify', 'advance')),
    bloom_level      SMALLINT    CHECK (bloom_level BETWEEN 1 AND 6),
    bloom_confidence FLOAT       CHECK (bloom_confidence BETWEEN 0.0 AND 1.0),
    created_at       TIMESTAMP   NOT NULL DEFAULT now()
);

CREATE INDEX idx_tutor_messages_session_id ON tutor_messages(session_id);
CREATE INDEX idx_tutor_messages_last_ai    ON tutor_messages(session_id, role, created_at DESC);
```

| Поле | Заполняется | Описание |
|---|---|---|
| `strategy` | Только для `role = 'ai'` | Какую стратегию применил AI |
| `bloom_level` | Только для `role = 'student'` | Оценка ответа по Блуму |
| `bloom_confidence` | Только для `role = 'student'` | Уверенность модели при оценке |

---

### tutor_results

Итоговый результат завершённой сессии. Пишется один раз при переходе в `completed`.

```sql
CREATE TABLE tutor_results (
    id                 SERIAL PRIMARY KEY,
    session_id         INT         NOT NULL UNIQUE REFERENCES tutor_sessions(id),
    student_id         INT         NOT NULL REFERENCES users(id),
    group_id           INT         REFERENCES groups(id),
    subject_id         INT         NOT NULL REFERENCES subjects(id),
    topic              TEXT        NOT NULL,
    final_bloom_level  SMALLINT    NOT NULL CHECK (final_bloom_level BETWEEN 1 AND 6),
    bloom_label        VARCHAR(50) NOT NULL,
    messages_count     INT         NOT NULL,
    completion_reason  TEXT        NOT NULL,
    strategies_used    JSONB       NOT NULL DEFAULT '[]',
    completed_at       TIMESTAMP   NOT NULL DEFAULT now()
);

CREATE INDEX idx_tutor_results_student_id      ON tutor_results(student_id);
CREATE INDEX idx_tutor_results_group_id        ON tutor_results(group_id);
CREATE INDEX idx_tutor_results_subject_id      ON tutor_results(subject_id);
CREATE INDEX idx_tutor_results_bloom_group     ON tutor_results(group_id, subject_id);
```

---

### licenses

Таблица лицензий продуктов.

```sql
CREATE TABLE licenses (
    id           SERIAL PRIMARY KEY,
    product      VARCHAR(20)  NOT NULL CHECK (product IN ('ai_tutor', 'cv')),
    license_key  VARCHAR(255) NOT NULL UNIQUE,
    hardware_id  VARCHAR(255) NOT NULL,
    is_active    BOOLEAN      NOT NULL DEFAULT false,
    activated_at TIMESTAMP,
    expires_at   TIMESTAMP
);
```

---

### schedule

Расписание занятий, загружается через Excel.

```sql
CREATE TABLE schedule (
    id          SERIAL PRIMARY KEY,
    group_id    INT         NOT NULL REFERENCES groups(id),
    subject_id  INT         NOT NULL REFERENCES subjects(id),
    room        VARCHAR(100) NOT NULL,
    weekday     VARCHAR(3)  NOT NULL CHECK (weekday IN
                    ('mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun')),
    time_start  TIME        NOT NULL,
    time_end    TIME        NOT NULL,
    uploaded_at TIMESTAMP   NOT NULL DEFAULT now(),
    uploaded_by INT         NOT NULL REFERENCES users(id),
    UNIQUE (room, weekday, time_start)
);

CREATE INDEX idx_schedule_group_id   ON schedule(group_id);
CREATE INDEX idx_schedule_subject_id ON schedule(subject_id);
```

Уникальный составной индекс `(room, weekday, time_start)` — детектирует коллизии при загрузке расписания.

---

### spaces

Справочник пространств — аудитории и переговорки.

```sql
CREATE TABLE spaces (
    id        SERIAL PRIMARY KEY,
    name      VARCHAR(255) NOT NULL,
    capacity  INT          NOT NULL,
    camera_id VARCHAR(100)
);
```

---

### bookings

Бронирования пространств через Telegram.

```sql
CREATE TABLE bookings (
    id                   SERIAL PRIMARY KEY,
    telegram_id          VARCHAR(100) NOT NULL,
    space_id             INT          NOT NULL REFERENCES spaces(id),
    date                 DATE         NOT NULL,
    time_start           TIME         NOT NULL,
    time_end             TIME         NOT NULL,
    status               VARCHAR(20)  NOT NULL DEFAULT 'active'
                             CHECK (status IN ('active', 'cancelled')),
    cancelled_at         TIMESTAMP,
    cooldown_expires_at  TIMESTAMP,
    created_at           TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE INDEX idx_bookings_telegram_id ON bookings(telegram_id);
CREATE INDEX idx_bookings_space_date  ON bookings(space_id, date, time_start);
```

`cooldown_expires_at` — проставляется при отмене бронирования (`cancelled_at + 30 минут`). Пока не истёк — новое бронирование запрещено.

---

### cv_zones

Зоны внутри пространства, отслеживаемые CV-модулем.

```sql
CREATE TABLE cv_zones (
    id                SERIAL PRIMARY KEY,
    space_id          INT          NOT NULL REFERENCES spaces(id),
    zone_name         VARCHAR(100) NOT NULL,
    current_occupancy INT          NOT NULL DEFAULT 0,
    confidence        FLOAT        NOT NULL DEFAULT 0.0,
    updated_at        TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE INDEX idx_cv_zones_space_id ON cv_zones(space_id);
```

---

### cv_occupancy_logs

Лог занятости для heatmap и аналитики.

```sql
CREATE TABLE cv_occupancy_logs (
    id             SERIAL    PRIMARY KEY,
    space_id       INT       NOT NULL REFERENCES spaces(id),
    occupancy_rate FLOAT     NOT NULL CHECK (occupancy_rate BETWEEN 0.0 AND 1.0),
    recorded_at    TIMESTAMP NOT NULL DEFAULT now()
);

CREATE INDEX idx_cv_occupancy_logs_space_time ON cv_occupancy_logs(space_id, recorded_at);
```

---

## Ключевые запросы

### Проверка кулдауна студента (15 минут)

```sql
SELECT started_at
FROM tutor_sessions
WHERE student_id = :student_id
ORDER BY started_at DESC
LIMIT 1;
```

Покрывается индексом `idx_tutor_sessions_cooldown`.

---

### GroupBloomMap — карта уровней группы

```sql
SELECT
    u.id,
    u.email,
    tr.subject_id,
    tr.topic,
    tr.final_bloom_level,
    tr.bloom_label,
    tr.completed_at
FROM tutor_results tr
JOIN users u ON u.id = tr.student_id
WHERE tr.group_id = :group_id
  AND (:subject_id IS NULL OR tr.subject_id = :subject_id)
ORDER BY tr.completed_at DESC;
```

---

### Получение последнего вопроса AI в сессии

```sql
SELECT content, strategy
FROM tutor_messages
WHERE session_id = :session_id
  AND role = 'ai'
ORDER BY created_at DESC
LIMIT 1;
```

Покрывается индексом `idx_tutor_messages_last_ai`.

---

### Проверка занятости пространства

```sql
SELECT id FROM bookings
WHERE space_id  = :space_id
  AND date      = :date
  AND status    = 'active'
  AND time_start < :time_end
  AND time_end  > :time_start;
```

---

## Что НЕ хранится в БД

| Что | Где вместо этого |
|---|---|
| Промпты для LLM | `backend/tutor/prompts.py` |
| Полная история LLM-запросов | Только итоговые оценки и тексты в `tutor_messages` |
| JWT-токены | Stateless. Исключение — blacklist при logout (опциональная таблица `token_blacklist` с `jti` и `expires_at`) |
| Конфиги сервисов | `.env` файлы |
