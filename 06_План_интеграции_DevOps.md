# 🔄 ПЛАН ИНТЕГРАЦИИ И DEVOPS

## Проект: Облачная система предиктивной диагностики оборудования для малого и среднего бизнеса.

---

## 1. ОБЩАЯ ИНФОРМАЦИЯ

| Параметр | Значение |
|----------|----------|
| **Проект** | Облачная система предиктивной диагностики оборудования для малого и среднего бизнеса. |
| **Код** | PG-2026-1 |
| **Дата** | 26.09.2026 |
| **Версия** | 1.0 |
| **Автор** | Команда "Аналитики" |

---

## 2. АРХИТЕКТУРА СИСТЕМЫ

### 2.1. Логическая архитектура

┌─────────────────────────────────────────────────────────┐
│ IoT-ДАТЧИКИ (вибрация, темп., ток) │
└────────────────────────┬────────────────────────────────┘
│ MQTT/HTTP
┌────────────────────────▼────────────────────────────────┐
│ API Gateway (Nginx) │
└────────────────────────┬────────────────────────────────┘
│
┌────────────────┼────────────────┐
│ │ │
┌───────▼──────┐ ┌───────▼──────┐ ┌───────▼──────┐
│ Ingestion │ │ Analytics │ │ Alerting │
│ Service │ │ Service │ │ Service │
│ (Python) │ │ (Python) │ │ (Python) │
└───────┬──────┘ └───────┬──────┘ └───────┬──────┘
│ │ │
└────────────────┼────────────────┘
│
┌────────────────┼────────────────┐
│ │ │
┌───────▼──────┐ ┌───────▼──────┐ ┌───────▼──────┐
│ PostgreSQL │ │ Redis │ │          │
│ (Primary) │ │ (Cache) │ │ Bot API │
└──────────────┘ └──────────────┘ └──────────────┘

### 2.2. Физическая архитектура

| Сервер | Роль | ОС | CPU | RAM | Disk |
|--------|------|----|-----|-----|------|
| SRV-APP-01 | Приложения | Linux | 4 core | 8 GB | 100 GB |
| SRV-DB-01 | PostgreSQL | Linux | 8 core | 16 GB | 500 GB SSD |
| SRV-CACHE-01 | Redis | Linux | 2 core | 4 GB | 50 GB |

---

## 3. CI/CD ПРОЦЕСС

### 3.1. Обзор CI/CD

Code → Build → Test → Deploy → Monitor


### 3.2. Этапы CI/CD

| Этап | Описание | Инструменты | Длительность |
|------|----------|-------------|--------------|
| Code | Разработка, review | Git, GitHub | — |
| Build | Сборка Docker | Docker, pip | 5 мин |
| Test | Тестирование | Pytest | 10 мин |
| Deploy (Dev) | Развёртывание | Docker Compose | 3 мин |
| Deploy (Prod) | Развёртывание | Docker, Nginx | 5 мин |
| Monitor | Мониторинг | Prometheus, Grafana | Постоянно |

### 3.3. Git Flow

main (production)
│
├───develop
│ ├───feature/sensors
│ └───hotfix/bug
│
└───release/v1.0


---

## 4. ИНФРАСТРУКТУРА КАК КОД (IaC)

### 4.1. Инструменты

| Инструмент | Назначение | Версия |
|------------|------------|--------|
| Docker | Контейнеризация | 24+ |
| Docker Compose | Оркестрация | 2.20+ |
| Ansible | Конфигурация | 2.15+ |

### 4.2. Структура репозитория

infrastructure/
├── docker/
│ ├── Dockerfile.api
│ ├── Dockerfile.worker
│ └── docker-compose.yml
├── ansible/
│ ├── playbooks/
│ └── inventory/
└── monitoring/
├── prometheus.yml
└── grafana/


### 4.3. Пример docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/predictguard
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  worker:
    build: ./worker
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/predictguard
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=predictguard
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  frontend:
    build: ./frontend
    ports:
      - "3000:80"

volumes:
  postgres_data:

  5. АВТОМАТИЗАЦИЯ РАЗВЁРТЫВАНИЯ
5.1. Dockerfile для backend
dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
5.2. GitHub Actions
yaml
name: CI/CD

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t predictguard/api:${{ github.sha }} .
      - run: docker push predictguard/api:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - run: ssh deploy@server "cd /app && docker-compose pull && docker-compose up -d"
6. МОНИТОРИНГ И ЛОГИРОВАНИЕ
6.1. Архитектура мониторинга
text
┌─────────────────────────────────────┐
│         Grafana Dashboard           │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│      Prometheus Server              │
└──────────────┬──────────────────────┘
               │
   ┌───────────┼───────────┐
   │           │           │
┌──▼──┐   ┌───▼───┐   ┌───▼───┐
│ API │   │  DB   │   │ Redis │
└─────┘   └───────┘   └───────┘
6.2. Ключевые метрики
Категория	Метрика	Порог
CPU	Использование	> 70%
Memory	Использование	> 80%
API	Response Time	> 500 мс
API	Error Rate	> 1%
Бизнес	Уведомлений/час	—
Бизнес	Аномалий/день	—
7. БЕЗОПАСНОСТЬ
Область	Мера
Сеть	Firewall, TLS
Аутентификация	JWT, bcrypt
Данные	AES-256, бэкапы
Приложение	WAF, rate limiting
Сканирование	Trivy, Dependabot
8. ПЛАН ВНЕДРЕНИЯ
Этап	Длительность	Ответственный
Подготовка инфраструктуры	1 нед	DevOps
Настройка CI/CD	1 нед	DevOps
Dev-развёртывание	3 дня	DevOps
Stage-развёртывание	3 дня	DevOps
Prod-развёртывание	1 день	DevOps
Гиперкар	2 нед	Команда
Чек-лист
☑ Описана архитектура
☑ Определён CI/CD
☑ Настроена IaC
☑ Описан мониторинг
☑ Описана безопасность
☑ Разработан план внедрения
Версия: 1.0 | Дата: 26.09.2026 | Автор: Команда "Аналитики"


