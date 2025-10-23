# MGC Audits - Проектный документ разработки

## Версия: 1.0
## Дата: 02.10.2025

---

## 1. Общее описание проекта

### 1.1. Название проекта
**MGC Audits** - Система управления аудитами качества и несоответствиями

### 1.2. Цель проекта
Создание веб-приложения для планирования и ведения аудитов качества, фиксации несоответствий (findings), управления их жизненным циклом, назначения ответственных, контроля статусов и сроков, согласования решений и формирования отчетности.

### 1.3. Замена существующей системы
Система заменяет текущую систему учета аудитов ISQ-AA и предоставляет более расширенный функционал.

### 1.4. Ключевые бизнес-требования
- Многопредприятийная архитектура с изоляцией данных
- Гибкая настраиваемая система ролей и прав доступа
- Визуальный конструктор workflow для статусов
- Интеграция с Active Directory, Yandex Mail, Telegram
- Мультиязычность (русский, английский)
- Мобильная готовность (планируется мобильное приложение)
- REST API для внешних интеграций

---

## 2. Технологический стек

### 2.1. Backend
- **Язык:** Python 3.13
- **Фреймворк:** FastAPI (latest)
- **ORM:** SQLAlchemy 2.0+ (async support)
- **База данных:** PostgreSQL 18
- **Кеш/Очереди:** Redis 6.4.0
- **Фоновые задачи:** Celery 5.5.3
- **Авторизация:** JWT (FastAPI dependencies + python-jose)
- **Валидация:** Pydantic 2.12.0+ (встроен в FastAPI)

### 2.2. Frontend
- **Язык:** TypeScript 5.9+
- **Фреймворк:** React 18.4+
- **UI библиотека:** Ant Design 5.19+
- **Управление состоянием:** Redux Toolkit 2.2+
- **Роутинг:** React Router v6.24+
- **HTTP клиент:** Axios 1.7+
- **Интернационализация:** react-i18next 14.1+

### 2.3. Инфраструктура
- **Контейнеризация:** Docker 24+, Docker Compose 2.20+
- **Web-сервер:** Nginx 1.25+
- **ASGI сервер:** Uvicorn
- **ОС сервера:** Ubuntu 22/24 LTS
- **Файловое хранилище:** Yandex Object Storage (S3-совместимое)

### 2.4. Интеграции
- **LDAP:** python-ldap (ручная реализация логики)
- **Email:** fastapi-mail (или аналогичная асинхронная библиотека)
- **Telegram:** python-telegram-bot

---

## 3. Архитектура системы

### 3.1. Общая архитектура

```
                          ┌─────────────────┐
                          │   Пользователь  │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │   Nginx (80)    │
                          │   Reverse Proxy │
                          └────┬──────┬─────┘
                               │      │
                ┌──────────────┘      └──────────────┐
                │                                     │
        ┌───────▼────────┐                  ┌────────▼────────┐
        │  React Frontend│                  │ FastAPI Backend │
        │   (port 3000)  │                  │   (port 8000)   │
        └────────────────┘                  └─────┬──────┬────┘
                                                  │      │
                                    ┌─────────────┘      └─────────────┐
                                    │                                   │
                          ┌─────────▼─────────┐            ┌───────────▼──────────┐
                          │  PostgreSQL 18    │            │    Redis 6.4         │
                          │  (port 5432)      │            │    (port 6379)       │
                          └───────────────────┘            └──────────┬───────────┘
                                                                       │
                                                          ┌────────────▼───────────┐
                                                          │  Celery Workers         │
                                                          │  + Celery Beat          │
                                                          └─────────────────────────┘
                                                                       │
                                            ┌──────────────────────────┼──────────────────────────┐
                                            │                          │                          │
                                    ┌───────▼──────┐        ┌──────────▼──────┐      ┌──────────▼─────────┐
                                    │  Email SMTP  │        │  Telegram API   │      │   LDAP/AD Server   │
                                    │  (Yandex)    │        │                 │      │                    │
                                    └──────────────┘        └─────────────────┘      └────────────────────┘
```

### 3.2. Модульная структура Backend приложения
Используется типичная для FastAPI структура, с разделением на `api`, `core`, `crud`, `models`, `schemas`.

```
backend/
├── app/
│   ├── api/          # Роутеры и эндпоинты (routers)
│   ├── core/         # Конфигурация, безопасность
│   ├── crud/         # Функции для работы с БД (Create, Read, Update, Delete)
│   ├── models/       # Модели SQLAlchemy
│   ├── schemas/      # Схемы Pydantic (для валидации API)
│   └── services/     # Бизнес-логика
├── celery_worker.py  # Конфигурация и запуск воркера Celery
└── main.py           # Точка входа в приложение FastAPI
```

### 3.3. Архитектурные принципы
- Разделение на модули с помощью FastAPI Routers
- RESTful API для всех операций
- Асинхронная обработка через Celery для ресурсоемких задач (уведомления, отчеты)
- Полное покрытие API для административных функций (пользовательская админка)
- Мягкое удаление (soft delete) для всех сущностей
- Audit trail для критичных операций
- Stateless JWT аутентификация для API
- Хранение пользовательских файлов в S3-совместимом хранилище (Yandex Object Storage)
- Шифрование критичных настроек в базе данных

---

## 4. Модель данных

**Примечание:** Все сущности, описанные ниже, будут реализованы с использованием ORM `SQLAlchemy 2.0+` с асинхронным драйвером базы данных (`asyncpg`).

### 4.1. Основные сущности

#### 4.1.1. Enterprise (Предприятие)
```python
id: UUID (PK)
name: VARCHAR(255)
code: VARCHAR(50) UNIQUE
description: TEXT
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.2. Division (Дивизион)
```python
id: UUID (PK)
enterprise_id: UUID (FK -> Enterprise)
name: VARCHAR(255)
short_name: VARCHAR(50)
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.3. Location (Локация)
```python
id: UUID (PK)
division_id: UUID (FK -> Division)
name: VARCHAR(255)
short_name: VARCHAR(50)
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.4. User (Пользователь)
```python
id: UUID (PK)
email: VARCHAR(255) UNIQUE
username: VARCHAR(150) UNIQUE
# ФИО на русском
first_name_ru: VARCHAR(150)
last_name_ru: VARCHAR(150)
patronymic_ru: VARCHAR(150) (nullable)
# ФИО на английском
first_name_en: VARCHAR(150)
last_name_en: VARCHAR(150)
location_id: UUID (FK -> Location, nullable) # Принадлежность к локации
is_ldap_user: BOOLEAN
ldap_dn: VARCHAR(255) (nullable)
language: VARCHAR(5) DEFAULT 'ru'
telegram_chat_id: BIGINT (nullable)
telegram_linked_at: TIMESTAMP (nullable)
is_auditor: BOOLEAN DEFAULT FALSE
is_expert: BOOLEAN DEFAULT FALSE
is_active: BOOLEAN
is_staff: BOOLEAN
is_superuser: BOOLEAN
last_login: TIMESTAMP
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.5. Role (Роль)
```python
id: UUID (PK)
name: VARCHAR(100)
code: VARCHAR(50) UNIQUE
description: TEXT
is_system: BOOLEAN  # системная роль, нельзя удалить
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.6. Permission (Право)
```python
id: UUID (PK)
role_id: UUID (FK -> Role)
enterprise_id: UUID (FK -> Enterprise, nullable)  # NULL = все предприятия
resource: VARCHAR(100)  # 'audit', 'finding', 'dictionary', etc.
action: VARCHAR(20)  # 'create', 'read', 'update', 'delete'
created_at: TIMESTAMP
```

#### 4.1.7. UserRole (Связь пользователь-роль)
```python
id: UUID (PK)
user_id: UUID (FK -> User)
role_id: UUID (FK -> Role)
# Ограничение области действия роли
enterprise_id: UUID (FK -> Enterprise, nullable)
division_id: UUID (FK -> Division, nullable)
location_id: UUID (FK -> Location, nullable)
created_at: TIMESTAMP
```

#### 4.1.8. DictionaryType (Тип справочника)
```python
id: UUID (PK)
name: VARCHAR(255)
code: VARCHAR(100) UNIQUE
description: TEXT
created_at: TIMESTAMP
updated_at: TIMESTAMP
```

#### 4.1.9. Dictionary (Элемент справочника)
```python
id: UUID (PK)
dictionary_type_id: UUID (FK -> DictionaryType) # Связь с типом справочника
name: VARCHAR(255)
code: VARCHAR(100)
description: TEXT
enterprise_id: UUID (FK -> Enterprise, nullable)  # NULL = общий справочник
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.10. StandardChapter (Раздел стандарта)
```python
id: UUID (PK)
standard_id: UUID (FK -> QualificationStandard)
name: VARCHAR(500)
code: VARCHAR(100) # Например, "9.2.2.1"
description: TEXT
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.11. Status (Статус)
```python
id: UUID (PK)
name: VARCHAR(100)
code: VARCHAR(50)
color: VARCHAR(7)  # HEX цвет #RRGGBB для статуса
entity_type: VARCHAR(50)  # 'audit', 'finding', 'audit_plan'
order: INTEGER
is_initial: BOOLEAN
is_final: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.12. StatusTransition (Переход статусов)
```python
id: UUID (PK)
from_status_id: UUID (FK -> Status)
to_status_id: UUID (FK -> Status)
required_roles: JSONB  # [role_id, role_id, ...]
required_fields: JSONB  # ['field_name', 'field_name', ...]
require_comment: BOOLEAN
notification_config: JSONB  # настройки уведомлений
color: VARCHAR(7) (nullable)  # HEX цвет переходов для визуализации (опционально)
created_at: TIMESTAMP
```

#### 4.1.13. AuditPlan (План аудитов)
```python
id: UUID (PK)
title: VARCHAR(500)
date_from: DATE
date_to: DATE
enterprise_id: UUID (FK -> Enterprise)
status: VARCHAR(20)  # 'draft', 'approved', 'in_progress', 'completed'
created_by_id: UUID (FK -> User)
approved_by_id: UUID (FK -> User, nullable)
approved_at: TIMESTAMP (nullable)
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.14. AuditPlanItem (Пункт плана аудитов)
```python
id: UUID (PK)
audit_plan_id: UUID (FK -> AuditPlan)
audit_type_id: UUID (FK -> AuditType)
process_id: UUID (FK -> Process, nullable)
product_id: UUID (FK -> Product, nullable)
project_id: UUID (FK -> Project, nullable)
norm_id: UUID (FK -> QualificationStandard, nullable)
planned_auditor_id: UUID (FK -> User, nullable)
planned_date_from: DATE
planned_date_to: DATE
priority: VARCHAR(20)  # 'high', 'medium', 'low'
notes: TEXT
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.15. AuditorQualification (Квалификация аудитора)
```python
id: UUID (PK)
user_id: UUID (FK -> User)
status_id: UUID (FK -> Status) # Статус жизненного цикла: draft, pending_verification, active, expired
certification_body: VARCHAR(255)  # орган сертификации
certificate_number: VARCHAR(100)
certificate_date: DATE
expiry_date: DATE
standards: ManyToMany -> QualificationStandard # Стандарты, по которым есть квалификация
additional_info: TEXT
is_active: BOOLEAN
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.16. Audit (Аудит)
```python
id: UUID (PK)
title: VARCHAR(500)
audit_number: VARCHAR(100) UNIQUE
subject: VARCHAR(500)
enterprise_id: UUID (FK -> Enterprise)
audit_type_id: UUID (FK -> AuditType)
process_id: UUID (FK -> Process, nullable)
product_id: UUID (FK -> Product, nullable)
project_id: UUID (FK -> Project, nullable)
norm_id: UUID (FK -> QualificationStandard, nullable)
status_id: UUID (FK -> Status)
auditor_id: UUID (FK -> User)
audit_plan_item_id: UUID (FK -> AuditPlanItem, nullable)  # связь с планом
audit_date_from: DATE
audit_date_to: DATE
year: INTEGER

# Новые поля: результаты и ресурсы
audit_result: VARCHAR(20) (nullable)  # 'green', 'yellow', 'red', 'no_noncompliance'
estimated_hours: DECIMAL (nullable)  # плановые часы на аудит
actual_hours: DECIMAL (nullable)  # фактические часы (заполняется после завершения)

# Новые поля: информация о переносах
postponed_reason: TEXT (nullable)  # причина отложения аудита
rescheduled_date: DATE (nullable)  # новая дата при переносе
rescheduled_by_id: UUID (FK -> User, nullable)  # кто перенес
rescheduled_at: TIMESTAMP (nullable)  # когда был перенос

# Утверждение графика
approved_by_id: UUID (FK -> User, nullable)  # кто утвердил график
approved_date: DATE (nullable)  # дата утверждения

created_by_id: UUID (FK -> User)
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.17. AuditComponent (Компонент аудита)
Новая модель для привязки компонентов (деталей, узлов, систем) из внешних систем (SAP, ERP) к аудитам. Позволяет детализировать, какие именно компоненты проверяются в рамках конкретного аудита.

```python
id: UUID (PK)
audit_id: UUID (FK -> Audit)
component_type: VARCHAR(50)  # 'part', 'assembly', 'system', 'product', 'subsystem'
sap_id: VARCHAR(100) (nullable)  # ID из SAP/ERP системы
part_number: VARCHAR(100) (nullable)  # номер детали
component_name: VARCHAR(500)  # название компонента
description: TEXT (nullable)  # описание компонента
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.18. Finding (Несоответствие)
```python
id: UUID (PK)
finding_number: INTEGER AUTO_INCREMENT
audit_id: UUID (FK -> Audit)
enterprise_id: UUID (FK -> Enterprise)
title: TEXT
description: TEXT
process_id: UUID (FK -> Process)
status_id: UUID (FK -> Status)
finding_type: VARCHAR(20)  # 'CAR1', 'CAR2', 'OFI'
resolver_id: UUID (FK -> User)
approver_id: UUID (FK -> User, nullable)
deadline: DATE
closing_date: DATE (nullable)
created_by_id: UUID (FK -> User)

# 5xWhy анализ (обязательно минимум 3 поля для CAR1/CAR2)
why_1: TEXT (nullable)
why_2: TEXT (nullable)
why_3: TEXT (nullable)
why_4: TEXT (nullable)
why_5: TEXT (nullable)

# Действия
immediate_action: TEXT (nullable) # Корректирующие действия
root_cause: TEXT (nullable) # Коренная причина
long_term_action: TEXT (nullable) # Долгосрочные корректирующие действия
action_verification: TEXT (nullable) # Проверка корректирующих действий
preventive_measures: TEXT (nullable) # Предупреждающие действия

created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.19. FindingDelegation (Делегирование)
```python
id: UUID (PK)
finding_id: UUID (FK -> Finding)
from_user_id: UUID (FK -> User)
to_user_id: UUID (FK -> User)
reason: TEXT
delegated_at: TIMESTAMP
revoked_at: TIMESTAMP (nullable)
```

#### 4.1.20. FindingComment (Комментарий)
```python
id: UUID (PK)
finding_id: UUID (FK -> Finding)
author_id: UUID (FK -> User)
text: TEXT
created_at: TIMESTAMP
updated_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.21. Attachment (Вложение)
Модель для хранения метаданных о файлах, загруженных пользователями. Физически файлы хранятся в Yandex Object Storage.
Модель является полиморфной и может быть связана с любой другой сущностью (Аудит, Несоответствие, Пользователь для сертификатов и т.д.).
```python
id: UUID (PK)
# Полиморфная связь
object_id: UUID
content_type: VARCHAR(100) # Имя модели, к которой привязан файл

original_file_name: VARCHAR(255)
s3_bucket: VARCHAR(100)
s3_key: VARCHAR(500)  # путь к объекту в S3
file_size: BIGINT
mimetype: VARCHAR(100)
uploaded_by_id: UUID (FK -> User)
uploaded_at: TIMESTAMP
deleted_at: TIMESTAMP (nullable)
```

#### 4.1.22. ChangeHistory (История изменений)
```python
id: UUID (PK)
entity_type: VARCHAR(50)  # 'audit', 'finding', etc.
entity_id: UUID
user_id: UUID (FK -> User)
field_name: VARCHAR(100)
old_value: TEXT
new_value: TEXT
changed_at: TIMESTAMP
```

#### 4.1.23. Notification (Уведомление)
```python
id: UUID (PK)
user_id: UUID (FK -> User)
event_type: VARCHAR(100)  # 'finding_created', 'deadline_approaching', 'overdue', etc.
entity_type: VARCHAR(50)
entity_id: UUID
title: VARCHAR(500)
message: TEXT
is_read: BOOLEAN
sent_email: BOOLEAN
sent_telegram: BOOLEAN
email_sent_at: TIMESTAMP (nullable)
telegram_sent_at: TIMESTAMP (nullable)
email_error: TEXT (nullable)  # причина ошибки отправки email
telegram_error: TEXT (nullable)  # причина ошибки отправки telegram
retry_count: INTEGER DEFAULT 0  # количество попыток повторной отправки
last_retry_at: TIMESTAMP (nullable)
notification_config: JSONB  # настройки уведомлений (за 3 дня, в 1 день, каждый день при просрочке)
created_at: TIMESTAMP
```

#### 4.1.24. NotificationQueue (Очередь уведомлений)
```python
id: UUID (PK)
notification_id: UUID (FK -> Notification)
channel: VARCHAR(20)  # 'email', 'telegram'
status: VARCHAR(20)  # 'pending', 'processing', 'sent', 'failed'
priority: INTEGER DEFAULT 0  # пока не используется (FIFO)
scheduled_at: TIMESTAMP  # когда запланирована отправка
sent_at: TIMESTAMP (nullable)
error_message: TEXT (nullable)
retry_count: INTEGER DEFAULT 0
max_retries: INTEGER DEFAULT 3
created_at: TIMESTAMP
updated_at: TIMESTAMP
```

#### 4.1.25. SystemSetting (Настройки системы)
```python
id: UUID (PK)
key: VARCHAR(100) UNIQUE
value: TEXT  # может хранить зашифрованное значение
value_type: VARCHAR(20)  # 'string', 'integer', 'boolean', 'json'
description: TEXT
category: VARCHAR(50)  # 'general', 'notifications', 's3', 'smtp', 'ldap', etc.
is_public: BOOLEAN DEFAULT false  # доступна ли настройка в пользовательской админке
is_encrypted: BOOLEAN DEFAULT false # шифровать ли значение в БД
updated_at: TIMESTAMP
updated_by_id: UUID (FK -> User)
```

**Ключевые настройки для уведомлений:**
```python
# Email Rate Limiting
'email.rate_limit_per_minute': 30  # максимум email в минуту
'email.rate_limit_per_hour': 500   # максимум email в час
'email.batch_size': 50              # размер пачки для отправки

# Telegram Rate Limiting
'telegram.rate_limit_per_second': 30  # максимум сообщений в секунду
'telegram.rate_limit_per_minute': 20  # максимум на пользователя в минуту
'telegram.batch_size': 20             # размер пачки для отправки

# Retry механизм
'notifications.max_retries': 3                    # максимум попыток
'notifications.retry_delay_seconds': [60, 300, 900]  # задержки: 1мин, 5мин, 15мин
'notifications.cleanup_after_days': 90            # удалять старые уведомления через 90 дней

# Мониторинг
'notifications.queue_alert_threshold': 100  # алерт при превышении N задач в очереди
'notifications.lag_alert_minutes': 15       # алерт при задержке обработки > 15 минут
```

**Ключевые настройки для подключений:**
```python
# Yandex Object Storage
's3.endpoint_url': 'https://storage.yandexcloud.net'
's3.region_name': 'ru-central1'
's3.access_key_id': '...' (is_encrypted=True)
's3.secret_access_key': '...' (is_encrypted=True)
's3.bucket_name': 'mgc-audits'
's3.app_data_prefix': 'app_data/'
's3.backup_data_prefix': 'backup_data/'

# SMTP
'smtp.host': 'smtp.yandex.ru'
'smtp.port': 587
'smtp.user': '...'
'smtp.password': '...' (is_encrypted=True)
'smtp.use_tls': True

# LDAP
'ldap.server_uri': 'ldap://your.domain.com'
'ldap.bind_dn': '...'
'ldap.bind_password': '...' (is_encrypted=True)
'ldap.user_search_base': 'ou=users,dc=your,dc=domain,dc=com'
```

#### 4.1.26. APIToken (API токен)
```python
id: UUID (PK)
name: VARCHAR(255)
token_prefix: VARCHAR(16)
token_hash: VARCHAR(255)
user_id: UUID (FK -> User, nullable)  # если привязан к пользователю
custom_permissions: JSONB  # или кастомные права
rate_limit_per_minute: INTEGER
rate_limit_per_hour: INTEGER
is_active: BOOLEAN
expires_at: TIMESTAMP (nullable)
created_at: TIMESTAMP
last_used_at: TIMESTAMP (nullable)
issued_for: VARCHAR(10)  # 'user' | 'system'
permission_mode: VARCHAR(10)  # 'inherit' | 'custom'
allowed_ips: CIDR[] (nullable)
description: TEXT (nullable)
```

#### 4.1.27. RegistrationInvite (Приглашение на регистрацию)
```python
id: UUID (PK)
email: VARCHAR(255) UNIQUE
token: VARCHAR(255) UNIQUE
expires_at: TIMESTAMP
is_activated: BOOLEAN DEFAULT FALSE
created_by_id: UUID (FK -> User, nullable)
created_at: TIMESTAMP
```

#### 4.1.28. S3Storage (Подключение к S3)
```python
id: UUID (PK)
name: VARCHAR(100)
endpoint_url: VARCHAR(255)
region: VARCHAR(50)
access_key_id: VARCHAR(200) (encrypted)
secret_access_key: VARCHAR(500) (encrypted)
bucket_name: VARCHAR(100)
prefix: VARCHAR(200)
enterprise_id: UUID (FK -> Enterprise, nullable)
division_id: UUID (FK -> Division, nullable)
is_default: BOOLEAN DEFAULT FALSE
is_active: BOOLEAN DEFAULT TRUE
health_status: VARCHAR(20)  # 'ok', 'degraded', 'error'
last_checked_at: TIMESTAMP (nullable)
error_message: TEXT (nullable)
created_at: TIMESTAMP
updated_at: TIMESTAMP
```

#### 4.1.29. EmailAccount (SMTP аккаунт)
```python
id: UUID (PK)
name: VARCHAR(100)
from_name: VARCHAR(100)
from_email: VARCHAR(255)
smtp_host: VARCHAR(255)
smtp_port: INTEGER
smtp_user: VARCHAR(255)
smtp_password: VARCHAR(500) (encrypted)
use_tls: BOOLEAN
enterprise_id: UUID (FK -> Enterprise, nullable)
division_id: UUID (FK -> Division, nullable)
is_default: BOOLEAN DEFAULT FALSE
is_active: BOOLEAN DEFAULT TRUE
health_status: VARCHAR(20)
last_checked_at: TIMESTAMP (nullable)
error_message: TEXT (nullable)
created_at: TIMESTAMP
updated_at: TIMESTAMP
```

#### 4.1.30. LdapConnection (LDAP подключение)
```python
id: UUID (PK)
name: VARCHAR(100)
server_uri: VARCHAR(255)
bind_dn: VARCHAR(255)
bind_password: VARCHAR(500) (encrypted)
user_search_base: VARCHAR(255)
enterprise_id: UUID (FK -> Enterprise, nullable)
division_id: UUID (FK -> Division, nullable)
is_default: BOOLEAN DEFAULT FALSE
is_active: BOOLEAN DEFAULT TRUE
health_status: VARCHAR(20)
last_checked_at: TIMESTAMP (nullable)
error_message: TEXT (nullable)
created_at: TIMESTAMP
updated_at: TIMESTAMP
```

### 4.2. Индексы базы данных

```sql
-- Пользователи
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_telegram ON users(telegram_chat_id) WHERE telegram_chat_id IS NOT NULL;
CREATE INDEX idx_users_ldap_dn ON users(ldap_dn) WHERE ldap_dn IS NOT NULL;

-- Иерархия
CREATE INDEX idx_divisions_enterprise ON divisions(enterprise_id);
CREATE INDEX idx_locations_division ON locations(division_id);

-- Планы аудитов
CREATE INDEX idx_audit_plans_enterprise ON audit_plans(enterprise_id);
CREATE INDEX idx_audit_plans_dates ON audit_plans(date_from, date_to);
CREATE INDEX idx_audit_plans_status ON audit_plans(status);

-- Пункты планов аудитов
CREATE INDEX idx_audit_plan_items_plan ON audit_plan_items(audit_plan_id);
CREATE INDEX idx_audit_plan_items_auditor ON audit_plan_items(planned_auditor_id);
CREATE INDEX idx_audit_plan_items_dates ON audit_plan_items(planned_date_from, planned_date_to);

-- Квалификация аудиторов
CREATE INDEX idx_auditor_qualifications_user ON auditor_qualifications(user_id);
CREATE INDEX idx_auditor_qualifications_active ON auditor_qualifications(user_id, is_active) WHERE is_active = true;
CREATE INDEX idx_auditor_qualifications_expiry ON auditor_qualifications(expiry_date);

-- Аудиты
CREATE INDEX idx_audits_plan_item ON audits(audit_plan_item_id);
CREATE INDEX idx_audits_status ON audits(status_id);
CREATE INDEX idx_audits_auditor ON audits(auditor_id);
CREATE INDEX idx_audits_dates ON audits(audit_date_from, audit_date_to);
CREATE INDEX idx_audits_year ON audits(year);
CREATE INDEX idx_audits_rescheduled ON audits(rescheduled_date) WHERE rescheduled_date IS NOT NULL;

-- Компоненты аудитов
CREATE INDEX idx_audit_components_audit ON audit_components(audit_id);
CREATE INDEX idx_audit_components_sap ON audit_components(sap_id) WHERE sap_id IS NOT NULL;
CREATE INDEX idx_audit_components_type ON audit_components(component_type);

-- Findings
CREATE INDEX idx_findings_audit ON findings(audit_id);
CREATE INDEX idx_findings_enterprise ON findings(enterprise_id);
CREATE INDEX idx_findings_status ON findings(status_id);
CREATE INDEX idx_findings_resolver ON findings(resolver_id);
CREATE INDEX idx_findings_approver ON findings(approver_id);
CREATE INDEX idx_findings_deadline ON findings(deadline);
CREATE INDEX idx_findings_process ON findings(process_id);

-- Attachments
CREATE INDEX idx_attachments_entity ON attachments(content_type, object_id);

-- История изменений
CREATE INDEX idx_change_history_entity ON change_history(entity_type, entity_id);
CREATE INDEX idx_change_history_user ON change_history(user_id);
CREATE INDEX idx_change_history_date ON change_history(changed_at);

-- Уведомления
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = false;
CREATE INDEX idx_notifications_created ON notifications(created_at);
CREATE INDEX idx_notifications_retry ON notifications(retry_count, last_retry_at) WHERE retry_count > 0;

-- Очередь уведомлений
CREATE INDEX idx_notification_queue_status ON notification_queue(status);
CREATE INDEX idx_notification_queue_channel ON notification_queue(channel, status);
CREATE INDEX idx_notification_queue_scheduled ON notification_queue(scheduled_at, status) WHERE status = 'pending';
CREATE INDEX idx_notification_queue_notification ON notification_queue(notification_id);
CREATE INDEX idx_notification_queue_created ON notification_queue(created_at);

-- JSONB
CREATE INDEX idx_status_transitions_required_roles ON status_transitions USING gin(required_roles);
CREATE INDEX idx_status_transitions_required_fields ON status_transitions USING gin(required_fields);
```

### 4.3. Партиционирование (PostgreSQL 18)

**Стратегия партиционирования для оптимизации запросов графика аудитов и масштабируемости.**

Партиционирование необходимо для таблиц с высоким объемом записей и частыми запросами за временные периоды:

#### 4.3.1. Таблица `audits` - RANGE по audit_date_from
**Обоснование:** Основная таблица для графика аудитов. Часто выполняются запросы "Аудиты за период [date_from, date_to]". Партиционирование по дате аудита ускоряет эти запросы на 10-50x.

```sql
CREATE TABLE audits (
    id UUID,
    title VARCHAR(500),
    audit_date_from DATE NOT NULL,
    -- ... остальные поля ...
    PARTITION BY RANGE (DATE_TRUNC('month', audit_date_from::timestamp))
);

-- Партиции на 3 месяца вперед и 12 месяцев назад
CREATE TABLE audits_2024_q4 PARTITION OF audits
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

CREATE TABLE audits_2025_q1 PARTITION OF audits
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

CREATE TABLE audits_2025_q2 PARTITION OF audits
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

-- Индексы для быстрых запросов внутри партиций
CREATE INDEX ON audits_2024_q4(audit_date_from, audit_date_to, enterprise_id);
CREATE INDEX ON audits_2024_q4(status_id, auditor_id);
CREATE INDEX ON audits_2025_q1(audit_date_from, audit_date_to, enterprise_id);
CREATE INDEX ON audits_2025_q1(status_id, auditor_id);
CREATE INDEX ON audits_2025_q2(audit_date_from, audit_date_to, enterprise_id);
CREATE INDEX ON audits_2025_q2(status_id, auditor_id);
```

#### 4.3.2. Таблица `audit_components` - RANGE по created_at
**Обоснование:** Компоненты аудитов могут быть в большом количестве. Запросы часто фильтруют по дате создания и связанному аудиту. Партиционирование ускоряет выборку по компонентам конкретного периода.

```sql
CREATE TABLE audit_components (
    id UUID,
    audit_id UUID,
    component_type VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    PARTITION BY RANGE (DATE_TRUNC('month', created_at))
);

CREATE TABLE audit_components_2024_q4 PARTITION OF audit_components
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

CREATE TABLE audit_components_2025_q1 PARTITION OF audit_components
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Индексы для быстрого поиска в пределах партиции
CREATE INDEX ON audit_components_2024_q4(audit_id, component_type);
CREATE INDEX ON audit_components_2024_q4(sap_id) WHERE sap_id IS NOT NULL;
CREATE INDEX ON audit_components_2025_q1(audit_id, component_type);
CREATE INDEX ON audit_components_2025_q1(sap_id) WHERE sap_id IS NOT NULL;
```

#### 4.3.3. Таблица `findings` - RANGE по created_at
**Обоснование:** Несоответствия накапливаются с течением времени. Запросы часто ищут findings за определенный период. Партиционирование исключает ненужные таблицы из полнотекстовых сканов.

```sql
CREATE TABLE findings (
    id UUID,
    audit_id UUID,
    created_at TIMESTAMP NOT NULL,
    -- ... остальные поля ...
    PARTITION BY RANGE (DATE_TRUNC('month', created_at))
);

CREATE TABLE findings_2024_q4 PARTITION OF findings
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

CREATE TABLE findings_2025_q1 PARTITION OF findings
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Индексы для быстрого поиска в пределах партиции
CREATE INDEX ON findings_2024_q4(audit_id, status_id);
CREATE INDEX ON findings_2024_q4(resolver_id, deadline);
CREATE INDEX ON findings_2025_q1(audit_id, status_id);
CREATE INDEX ON findings_2025_q1(resolver_id, deadline);
```

#### 4.3.4. Таблица `notifications` - RANGE по created_at
**Обоснование:** Уведомления - самая быстрорастущая таблица (может быть миллионы записей). Партиционирование критично для:
- Статистики уведомлений (≤7 дней)
- Очистки старых записей (удаление по партициям)
- Мониторинга очередей

```sql
CREATE TABLE notifications (
    id UUID,
    created_at TIMESTAMP NOT NULL,
    -- ... остальные поля ...
    PARTITION BY RANGE (DATE_TRUNC('day', created_at))
);

-- Ежемесячные партиции
CREATE TABLE notifications_2024_10 PARTITION OF notifications
    FOR VALUES FROM ('2024-10-01') TO ('2024-11-01');

CREATE TABLE notifications_2024_11 PARTITION OF notifications
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');

CREATE TABLE notifications_2024_12 PARTITION OF notifications
    FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

CREATE TABLE notifications_2025_01 PARTITION OF notifications
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Индексы для быстрого поиска в пределах партиции
CREATE INDEX ON notifications_2024_10(user_id, is_read, created_at DESC);
CREATE INDEX ON notifications_2024_10(event_type, created_at);
CREATE INDEX ON notifications_2024_11(user_id, is_read, created_at DESC);
CREATE INDEX ON notifications_2024_11(event_type, created_at);
-- ... аналогично для других партиций
```

#### 4.3.5. Таблица `notification_queue` - RANGE по created_at
**Обоснование:** Очередь уведомлений быстро растет. Партиционирование позволяет:
- Быстро удалять обработанные уведомления старше N дней
- Ускорить поиск по статусам в текущем периоде
- Мониторить лаг обработки за периоды

```sql
CREATE TABLE notification_queue (
    id UUID,
    created_at TIMESTAMP NOT NULL,
    status VARCHAR(20),
    -- ... остальные поля ...
    PARTITION BY RANGE (DATE_TRUNC('week', created_at))
);

CREATE TABLE notification_queue_2024_w40 PARTITION OF notification_queue
    FOR VALUES FROM ('2024-09-30') TO ('2024-10-07');

CREATE TABLE notification_queue_2024_w41 PARTITION OF notification_queue
    FOR VALUES FROM ('2024-10-07') TO ('2024-10-14');

-- Индексы для быстрого поиска
CREATE INDEX ON notification_queue_2024_w40(status, scheduled_at);
CREATE INDEX ON notification_queue_2024_w40(channel, status);
CREATE INDEX ON notification_queue_2024_w41(status, scheduled_at);
CREATE INDEX ON notification_queue_2024_w41(channel, status);
```

#### 4.3.6. Таблица `change_history` - RANGE по changed_at
**Обоснование:** История изменений растет быстро. Партиционирование позволяет:
- Быстро удалять старые записи (по партициям)
- Ускорить поиск истории за период
- Архивировать старые данные отдельно

```sql
CREATE TABLE change_history (
    id UUID,
    changed_at TIMESTAMP NOT NULL,
    -- ... остальные поля ...
    PARTITION BY RANGE (DATE_TRUNC('month', changed_at))
);

CREATE TABLE change_history_2024_q4 PARTITION OF change_history
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

CREATE TABLE change_history_2025_q1 PARTITION OF change_history
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Индексы для быстрого поиска
CREATE INDEX ON change_history_2024_q4(entity_type, entity_id, changed_at DESC);
CREATE INDEX ON change_history_2024_q4(user_id, changed_at);
CREATE INDEX ON change_history_2025_q1(entity_type, entity_id, changed_at DESC);
CREATE INDEX ON change_history_2025_q1(user_id, changed_at);
```

#### 4.3.7. Управление партициями

**Автоматическое создание новых партиций:**
- Ежемесячно для `audits`, `findings`, `change_history` (в начале месяца)
- Еженедельно для `notification_queue` (в начале недели)
- Ежедневно для `notifications` (в начале дня)

**Это можно настроить через:**
- Cron-задачу на сервере (Celery Beat)
- pg_partman расширение PostgreSQL
- Вручную через миграции

**Удаление старых партиций:**
- Уведомления: старше 7 дней
- История: старше 90 дней
- Findings/Audits/Components: архивировать, не удалять


---

## 5. Безопасность

### 5.1. Шифрование настроек

Критически важные настройки, хранящиеся в таблице `SystemSetting` (например, пароли, API-ключи), подлежат шифрованию в базе данных.
- Для шифрования используется симметричный алгоритм AES.
- Ключ для шифрования (`SETTINGS_ENCRYPTION_KEY`) должен быть сгенерирован один раз и храниться в переменной окружения в `.env` файле на сервере.
- Поле `value` у записей с флагом `is_encrypted=True` автоматически шифруется перед сохранением в БД и расшифровывается при чтении.
- В административной панели должна быть возможность ввести новое значение, которое будет зашифровано на сервере, но не должна отображаться текущее расшифрованное значение.

### 5.2. Механизм единой сессии
### 5.3. Безопасность API-токенов

- Хранение только `token_hash`; «сырой» токен показывается один раз при создании/ротации.
- `permission_mode`: `inherit` — наследование прав привязанного пользователя; `custom` — права описываются в `custom_permissions`.
- Ограничения: `allowed_ips` (белый список IP/CIDR), `rate_limit_per_minute/hour` на уровне токена.
- `issued_for='system'` позволяет выдавать токены для внешних интеграций.
- Все действия по токенам фиксируются в `ChangeHistory` с `entity_type='api_token'`.
При каждом успешном логине будет генерироваться новый идентификатор сессии (session_id), который будет сохраняться для пользователя (например, в Redis или в самой модели User). Этот же `session_id` будет закодирован в JWT токен.
При каждом запросе к защищенному эндпоинту, система будет сравнивать `session_id` из токена с тем, что хранится на сервере для данного пользователя. Если они не совпадают, это означает, что пользователь вошел в систему с другого устройства, и текущая сессия становится недействительной (возвращается ошибка 401).

---

## 6. Ключевая бизнес-логика

### 6.1. Регистрация пользователей по приглашениям
1.  **Создание приглашения**: Администратор (системный или уровня предприятия/дивизиона) вводит один или несколько email адресов в специальном интерфейсе.
2.  **Генерация токена**: Для каждого email система создает запись в таблице `RegistrationInvite` с уникальным, криптографически стойким токеном и ограниченным сроком действия (например, 72 часа).
3.  **Отправка письма**: Система отправляет на указанный email письмо со ссылкой для регистрации, содержащей сгенерированный токен.
4.  **Регистрация пользователя**: Пользователь переходит по ссылке. Если токен валиден и не истек, ему открывается форма для ввода:
    - ФИО на русском (Фамилия, Имя, Отчество)
    - ФИО на английском (Фамилия, Имя)
    - Пароль + подтверждение пароля
5.  **Активация**: После успешного заполнения формы создается новая запись `User`, а запись `RegistrationInvite` помечается как `is_activated=True`.

### 6.2. Управление квалификациями аудиторов
Процесс позволяет пользователям самостоятельно управлять своими данными с последующей верификацией.
1.  **Создание (Аудитор)**: Пользователь сам заполняет все поля своей квалификации (орган сертификации, номер, даты, стандарты) и прикрепляет подтверждающие файлы. Статус новой записи — `draft` (Черновик).
2.  **Отправка на проверку (Аудитор)**: Когда все данные внесены, пользователь отправляет квалификацию на проверку. Статус меняется на `pending_verification` (На проверке).
3.  **Верификация (Администратор)**: Администратор дивизиона получает уведомление. Он проверяет предоставленные данные и документы. Он может **утвердить** (статус меняется на `active` (Действующая)) или **отклонить** (статус возвращается в `draft` с обязательным комментарием о причине).
4.  **Истечение срока**: Фоновая задача (Celery Beat) ежедневно проверяет квалификации. Если `expiry_date` прошла, статус автоматически меняется на `expired` (Истек срок).

### 6.3. Проверка квалификации при назначении на аудит
При попытке назначить пользователя в качестве аудитора (`planned_auditor_id` в плане или `auditor_id` в самом аудите), система выполняет проверку:
- Определяются стандарты (`norm_id`), указанные в аудите или пункте плана.
- Система проверяет, есть ли у кандидата в аудиторы активная (`is_active=True`) и не просроченная (`expiry_date`) запись в `AuditorQualification`, которая покрывает требуемые стандарты.
- Если совпадения не найдены, API вернет ошибку, и назначение будет невозможно.

### 6.4. Валидация полей несоответствия (Finding)
Обязательность полей в сущности `Finding` зависит от ее типа (`finding_type`). Валидация происходит на уровне API при сохранении.
-   **CAR1**: Обязательны для заполнения все 5 полей решения (`immediate_action`, `root_cause`, `long_term_action`, `action_verification`, `preventive_measures`) И как минимум 3 из 5 полей `why_*`.
-   **CAR2**: Обязательно только поле `immediate_action` И как минимум 3 из 5 полей `why_*`.
-   **OFI**: На данный момент от заказчика не получены уточнения по этому типу несоответствия

### 6.8. Привязка Telegram пользователем
- Пользователь инициирует привязку через endpoint, получает одноразовую ссылку/токен.
- После подтверждения в боте сохраняются `telegram_chat_id` и `telegram_linked_at`.
- Отвязка очищает `telegram_chat_id` и `telegram_linked_at`.

### 6.9. Семантика уведомлений и статистика
- Уведомление — событие, требующее доставки по каналам (Email/Telegram).
- Попытки и статусы доставки ведутся в `NotificationQueue`.
- Предоставляется статистика за ≤7 дней по каналам и типам событий.

### 6.5. Экспорт данных аудита
Для любого аудита должна быть доступна функция экспорта.
-   Процесс инициируется пользователем через интерфейс и выполняется асинхронно с помощью Celery.
-   Задача собирает всю информацию, связанную с аудитом:
    -   Основные данные самого аудита.
    -   Все связанные несоответствия (`Findings`).
    -   Все вложения (`Attachments`), прикрепленные как к самому аудиту, так и ко всем его несоответствиям.
    -   Вся история изменений (`ChangeHistory`) по аудиту и его несоответствиям.
-   Результат упаковывается в единый ZIP-архив (содержащий вложения и отдельный файл `history.json`) и становится доступен для скачивания.

### 6.6. Дашборд "Мои задачи"
После входа в систему пользователь видит персонализированный дашборд, который агрегирует информацию о его текущих задачах:
-   Список несоответствий (`Findings`), где он указан как ответственный (`resolver_id`), и которые находятся в активном статусе.
-   Список предстоящих аудитов, где он назначен аудитором (`auditor_id`).
-   Возможно, список аудитов, где он является ответственным за проверку (`approver_id`).

### 6.7. Делегирование ответственности
В системе реализован механизм гибкой смены ответственного.
-   **Текущий исполнитель**: Поля `auditor_id` (в аудите) и `resolver_id` (в несоответствии) всегда хранят ID **текущего** ответственного сотрудника.
-   **Процесс делегирования**: Когда один сотрудник передает задачу другому, система просто обновляет значение в поле `resolver_id` / `auditor_id`.
-   **История**: Каждая такая передача фиксируется в `FindingDelegation` или аналогичной таблице для аудитов. Это позволяет отследить всю цепочку ответственных.
-   **"Главный" ответственный**: Понятие "главного" ответственного (инициатора) реализуется на уровне логики. Например, при эскалации или отправке уведомлений о просрочке, система анализирует историю делегирования, чтобы уведомить не только текущего исполнителя, но и того, кто изначально передал ему задачу.

### 6.10. Работа с произвольными периодами в аудитах
- Пользователи могут выбирать произвольные даты для аудитов (например, "с 15.03.2024 по 20.03.2024").
- Система автоматически рассчитывает `audit_date_from` и `audit_date_to` на основе выбранного периода.
- При экспорте данных аудита, если период не указан, экспортируются все данные за текущий год.
- При создании нового аудита, если период не указан, система предлагает текущий год.
- При редактировании аудита, если период не указан, система сохраняет текущий период.

### 6.11. График аудитов (Audit Calendar)
**Система поддерживает гибкую визуализацию графика аудитов с произвольными периодами без привязки к календарному году, месяцу или неделе.**

#### Функциональность:
1. **Просмотр графика за произвольный период**: Пользователь выбирает `date_from` и `date_to`. API возвращает все аудиты в этом диапазоне.
   - Параметры: `date_from`, `date_to`, фильтры по процессам, продуктам, аудиторам, статусам.
   - Результат: Список аудитов с их компонентами, часами, статусом, результатом.

2. **Просмотр графика по компонентам**: Группировка аудитов по компонентам (детали, узлы, системы).
   - Позволяет видеть, какие компоненты проверяются и когда.
   - Фильтры: `component_type`, `sap_id`, `part_number`.
   - Полезно для отслеживания полноты аудита по всем компонентам.

3. **Переносы аудитов**: При необходимости перенести аудит:
   - Заполняются поля `rescheduled_date`, `postponed_reason`, `rescheduled_by_id`, `rescheduled_at`.
   - История переносов сохраняется для audit trail.

4. **Статусы и результаты**: Каждый аудит имеет:
   - `status_id`: текущий статус (запланирован, проводится, отложен, завершен и т.д.)
   - `audit_result`: результат (green, yellow, red, no_noncompliance)
   - Цветовое кодирование для быстрой визуализации на фронте.

5. **Компоненты в аудите**: Модель `AuditComponent` позволяет отслеживать:
   - Какие детали/узлы/системы проверяются.
   - SAP ID для интеграции с внешними системами.
   - Количество часов на каждый компонент.

---

## 7. Безопасность

