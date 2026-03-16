# Infrastructure & Deploy

## Обзор

Система разворачивается через Docker Compose на одном сервере. Все сервисы живут в одной Docker-сети и общаются по именам контейнеров. Nginx принимает все внешние запросы и маршрутизирует их внутрь.

---

## Требования к серверу

### Минимальные (продакшн)

| Компонент | Минимум |
|---|---|
| CPU | 4 ядра |
| RAM | 16 GB |
| Диск | 100 GB SSD |
| ОС | Ubuntu 22.04 LTS |
| Docker | 24+ |
| Docker Compose | 2.20+ |

### Рекомендуемые

| Компонент | Рекомендация |
|---|---|
| CPU | 8 ядер |
| RAM | 32 GB |
| Диск | 200 GB SSD |
| GPU | Опционально — если ai-server работает с локальной моделью |

> RAM делится преимущественно между Java-backend (~4 GB), ai-server (~4 GB) и PostgreSQL (~2 GB).
---

## Окружения

| Окружение | Назначение | Конфиг |
|---|---|---|
| `local` | Разработка на ноутбуке | `docker-compose.yml` |
| `staging` | Тестирование перед релизом | `docker-compose.prod.yml` + отдельный `.env.staging` |
| `production` | Боевой сервер | `docker-compose.prod.yml` + `.env.prod` |

Staging — копия продакшн-конфига, но с тестовыми данными и отдельной БД без персданных.

---

## Docker Compose

### docker-compose.yml (локальная разработка)

```yaml
services:

  java-backend:
    build: ./java-backend
    env_file: .env
    ports:
      - "8080:8080"

  backend:
    build: ./backend
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - java-backend
      - ai-server
    volumes:
      - ./backend:/app     # hot reload при разработке

  ai-server:
    build: ./ai-server
    env_file: .env
    ports:
      - "50051:50051"

  qt-server:
    build: ./qt-server
    env_file: .env
    network_mode: host      # доступ к RTSP-камерам в локальной сети

  telegram-bot:
    build: ./telegram-bot
    env_file: .env
    depends_on:
      - backend

  frontend:
    build:
      context: ./frontend
      target: dev           # dev-сборка с hot reload
    ports:
      - "3000:3000"

  admin-panel:
    build:
      context: ./admin-panel
      target: dev
    ports:
      - "3001:3001"
```

### docker-compose.prod.yml (продакшн)

```yaml
services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl/certs:ro
      - frontend_dist:/usr/share/nginx/frontend:ro
      - admin_dist:/usr/share/nginx/admin:ro
    depends_on:
      - backend
      - java-backend
    restart: always

  java-backend:
    image: ${REGISTRY}/java-backend:${TAG}
    env_file: .env.prod
    restart: always

  backend:
    image: ${REGISTRY}/backend:${TAG}
    env_file: .env.prod
    restart: always
    depends_on:
      - java-backend

  ai-server:
    image: ${REGISTRY}/ai-server:${TAG}
    env_file: .env.prod
    restart: always
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]   # убрать если нет GPU

  qt-server:
    image: ${REGISTRY}/qt-server:${TAG}
    env_file: .env.prod
    network_mode: host
    restart: always

  telegram-bot:
    image: ${REGISTRY}/telegram-bot:${TAG}
    env_file: .env.prod
    restart: always

  frontend:
    image: ${REGISTRY}/frontend:${TAG}
    volumes:
      - frontend_dist:/dist
    command: cp -r /app/dist/. /dist/

  admin-panel:
    image: ${REGISTRY}/admin-panel:${TAG}
    volumes:
      - admin_dist:/dist
    command: cp -r /app/dist/. /dist/

volumes:
  frontend_dist:
  admin_dist:
```

---

## Nginx

### nginx.conf

```nginx
events {}

http {
    include       mime.types;
    default_type  application/octet-stream;

    upstream backend {
        server backend:8000;
    }

    upstream java_backend {
        server java-backend:8080;
    }

    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name your-domain.com;

        ssl_certificate     /etc/ssl/certs/fullchain.pem;
        ssl_certificate_key /etc/ssl/certs/privkey.pem;

        # Frontend (студент / преподаватель)
        location / {
            root  /usr/share/nginx/frontend;
            index index.html;
            try_files $uri $uri/ /index.html;
        }

        # Admin panel
        location /admin/ {
            root  /usr/share/nginx/admin;
            index index.html;
            try_files $uri $uri/ /index.html;
        }

        # Python API (AI Tutor, Telegram webhook)
        location /api/v1/ {
            proxy_pass         http://backend;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
        }

        # Telegram webhook
        location /telegram/webhook {
            proxy_pass         http://backend;
            proxy_set_header   Host $host;
        }

        # Java Backend (пользователи, расписание, группы, Core API)
        location /core/api/ {
            proxy_pass         http://java_backend;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
        }
    }
}
```

---

## CI/CD

Три стадии на каждый push в `main`. Инструмент — GitHub Actions или GitLab CI.

### Стадии

```
push → main
   │
   ├── 1. TEST
   │       java-backend: mvn test
   │       backend:      pytest tests/
   │       telegram-bot: pytest tests/
   │
   ├── 2. BUILD
   │       docker build каждого сервиса
   │       docker push → registry (Docker Hub / self-hosted)
   │       тег образа = git commit SHA
   │
   └── 3. DEPLOY
           SSH на сервер
           export TAG=<commit-sha>
           docker-compose -f docker-compose.prod.yml pull
           docker-compose -f docker-compose.prod.yml up -d
           docker image prune -f   (чистим старые образы)
```

### .github/workflows/deploy.yml (скелет)

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Test java-backend
        run: |
          cd java-backend
          mvn test
      - name: Test backend
        run: |
          cd backend
          pip install -r requirements.txt
          pytest tests/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push images
        run: |
          docker login -u ${{ secrets.REGISTRY_USER }} \
                       -p ${{ secrets.REGISTRY_PASS }}
          TAG=${{ github.sha }}
          docker build -t ${{ secrets.REGISTRY }}/java-backend:$TAG ./java-backend
          docker push ${{ secrets.REGISTRY }}/java-backend:$TAG
          docker build -t ${{ secrets.REGISTRY }}/backend:$TAG ./backend
          docker push ${{ secrets.REGISTRY }}/backend:$TAG
          # повторить для каждого сервиса

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/project
            export TAG=${{ github.sha }}
            export REGISTRY=${{ secrets.REGISTRY }}
            docker-compose -f docker-compose.prod.yml pull
            docker-compose -f docker-compose.prod.yml up -d
            docker image prune -f
```

---
## Бэкапы PostgreSQL

```bash
# /opt/scripts/backup.sh

#!/bin/bash
DATE=$(date +%Y-%m-%d)
BACKUP_DIR=/opt/backups

mkdir -p $BACKUP_DIR

# Дамп БД
docker exec postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB \
  | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Удалить бэкапы старше 7 дней
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

# Загрузить в S3 (опционально)
# aws s3 cp $BACKUP_DIR/db_$DATE.sql.gz s3://your-bucket/backups/
```

```bash
# Добавить в cron (каждую ночь в 03:00)
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### Восстановление из бэкапа

```bash
gunzip -c /opt/backups/db_2026-01-01.sql.gz \
  | docker exec -i postgres psql -U $POSTGRES_USER $POSTGRES_DB
```


---

## Мониторинг

Минимальный стек — без лишней сложности на старте.

### Healthcheck endpoint

Каждый сервис отдаёт `GET /health` со статусом `200 OK`.

Пример для Java: `GET /core/api/health`
Пример для Python: `GET /api/v1/health`

### Uptime Kuma

Self-hosted мониторинг. Проверяет healthcheck каждые 30 секунд. При падении шлёт алерт в Telegram.

```bash
# Установка
docker run -d \
  --name uptime-kuma \
  -p 3002:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

Настроить мониторы на:
- `https://your-domain.com/core/api/health` — java-backend
- `https://your-domain.com/api/v1/health` — backend
- `http://ai-server:50051` — ai-server (TCP probe)
- `https://your-domain.com` — nginx (общая доступность)

### Логи

Все сервисы пишут в stdout — Docker собирает автоматически.

```bash
# Логи конкретного сервиса
docker logs java-backend --tail 100 -f

# Логи всех сервисов
docker-compose logs --tail 50 -f
```

Если нужно больше — добавить Grafana + Loki позже как отдельный сервис. На старте это не первый приоритет.

---

## Первый деплой — чеклист

```
[ ] Сервер с Ubuntu 22.04, Docker 24+, Docker Compose 2.20+
[ ] Склонировать репозиторий в /opt/project
[ ] Скопировать .env.example → .env.prod, заполнить все переменные
[ ] Настроить SSL-сертификат (Let's Encrypt или корпоративный)
[ ] Положить сертификаты в ./ssl/
[ ] Запустить: docker-compose -f docker-compose.prod.yml up -d
[ ] Убедиться, что java-backend успешно запустился и накатил миграции (Flyway/Liquibase)
[ ] Создать admin-пользователя вручную или через SQL-скрипт инициализации
[ ] Настроить Telegram webhook: POST https://api.telegram.org/bot<TOKEN>/setWebhook
[ ] Поднять Uptime Kuma, настроить мониторы и алерты
[ ] Настроить cron для бэкапов
[ ] Проверить все healthcheck endpoints
```
