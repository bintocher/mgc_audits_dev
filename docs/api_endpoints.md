# MGC Audits - Спецификация API

Этот документ описывает полную структуру REST API для бэкенд-сервиса MGC Audits.

**Базовый URL:** `/api/v1`

---

## Принципы API

1.  **Именование:** Все пути в URL используют `snake_case`.
2.  **Методы HTTP:**
    *   `POST`: Создание нового ресурса.
    *   `PUT`: Полное или частичное обновление существующего ресурса.
    *   `GET`: Получение одного ресурса или списка ресурсов.
    *   `DELETE`: Мягкое удаление ресурса (установка флага `deleted_at`).
3.  **Фильтрация и пагинация:** Каждый `GET` эндпоинт, возвращающий список, поддерживает следующие query-параметры:
    *   **Фильтры:** Параметры для фильтрации по полям сущности (например, `status_id`, `is_active`).
    *   **Сортировка:** `sort_by` (имя поля) и `order` (`asc` или `desc`).
    *   **Пагинация:** `page` (номер страницы, по умолчанию 1) и `size` (количество элементов на странице, по умолчанию 20).
4.  **Аутентификация:** Все эндпоинты, кроме `/auth/login` и `/auth/register`, требуют `Bearer` токен в заголовке `Authorization`.

---

## 1. Аутентификация

### `POST /api/v1/auth/login`
Вход в систему.
- **Request Body:** `{ "email": "...", "password": "..." }`
- **Response:** `{ "access_token": "...", "refresh_token": "..." }`

### `POST /api/v1/auth/refresh`
Обновление access токена.
- **Request Body:** `{ "refresh_token": "..." }`
- **Response:** `{ "access_token": "...", "refresh_token": "..." }`

### `POST /api/v1/auth/logout`
Выход из системы (делает недействительным refresh токен).
- **Response:** `204 No Content`

### `POST /api/v1/auth/register`
Регистрация по токену-приглашению.
- **Query Params:** `token` (токен из email).
- **Request Body:** `{ "first_name_ru": "...", "last_name_ru": "...", "password": "..." }`
- **Response:** `201 Created`

### `POST /api/v1/auth/telegram/link`
Выдать одноразовую ссылку/токен для привязки Telegram к текущему пользователю.
- **Response:** `{ "link": "...", "expires_at": "..." }`

### `POST /api/v1/auth/telegram/unlink`
Отвязать Telegram от текущего пользователя.
- **Response:** `204 No Content`

---

## 2. Пользователи и Роли

### Ресурс: `/api/v1/users`

### `GET /api/v1/users`
Получить список пользователей.
- **Query Params:**
    - `role_id` (фильтр по ID роли)
    - `is_active` (`true`/`false`, фильтр по статусу)
    - `location_id` (фильтр по ID локации)
    - `sort_by` (`email`, `last_name_ru`, `created_at`, ...), `order` (`asc`/`desc`)
    - `page`, `size`
- **Response:** `[{"id": "...", "email": "...", ...}]`

### `POST /api/v1/users`
Создать нового пользователя.
- **Request Body:** `{ "email": "...", "password": "...", ... }`
- **Response:** `201 Created` со всеми данными пользователя.

### `GET /api/v1/users/me`
Получить информацию о текущем авторизованном пользователе.
- **Response:** `{ "id": "...", "email": "...", "telegram_linked": true|false, "telegram_chat_id": 123456789, ...}`

### `PATCH /api/v1/users/me`
Обновление своего профиля.
- **Request Body:** `{ "first_name_ru": "...", "last_name_en": "..." }`
- **Response:** `200 OK` с обновленными данными.

### `GET /api/v1/users/{user_id}`
Получить информацию о конкретном пользователе.
- **Response:** `{ "id": "...", "email": "...", ...}`

### `PUT /api/v1/users/{user_id}`
Обновить информацию о пользователе.
- **Request Body:** `{ "first_name_ru": "...", "is_auditor": true, ... }`
- **Response:** `200 OK` с обновленными данными пользователя.

### `DELETE /api/v1/users/{user_id}`
Деактивировать (мягкое удаление) пользователя.
- **Response:** `204 No Content`

### Ресурс: `/api/v1/roles`
- `GET /api/v1/roles`: Список ролей.
- `POST /api/v1/roles`: Создать роль.
- `GET /api/v1/roles/{id}`: Получить роль.
- `PUT /api/v1/roles/{id}`: Обновить роль.
- `DELETE /api/v1/roles/{id}`: Удалить роль.
- `GET /api/v1/roles/{id}/permissions`: Список прав для роли.
- `POST /api/v1/roles/{id}/permissions`: Добавить права для роли.

---

## 2.1. API-токены

### Ресурс: `/api/v1/api_tokens`
- `GET /api/v1/api_tokens`: Список токенов (фильтры: `user_id`, `is_active`, `issued_for`).
- `POST /api/v1/api_tokens`: Создать токен.
    - **Request Body:** `{ "name": "...", "permission_mode": "inherit|custom", "custom_permissions": { ... }, "user_id": "..." (optional), "issued_for": "user|system", "rate_limit_per_minute": 30, "rate_limit_per_hour": 500, "allowed_ips": ["1.2.3.4/32"], "expires_at": "...", "description": "..." }`
    - **Response:** `201 Created` c одноразовым полем `plain_token` и `token_prefix`.
- `GET /api/v1/api_tokens/{id}`: Получить токен (без секрета).
- `PUT /api/v1/api_tokens/{id}`: Обновить метаданные (без секрета).
- `POST /api/v1/api_tokens/{id}/rotate`: Ротация секрета; возвращает новый `plain_token` один раз.
- `DELETE /api/v1/api_tokens/{id}`: Отозвать токен.

Примечания:
- Токены могут наследовать права пользователя (`inherit`) или иметь собственные (`custom`).
- Токены могут быть выданы для внешних систем (`issued_for=system`) для внесения данных в бэкенд.

---

## 3. Иерархия предприятия

### Ресурс: `/api/v1/enterprises`
- `GET /api/v1/enterprises`: Список предприятий.
- `POST /api/v1/enterprises`: Создать предприятие.
- `GET /api/v1/enterprises/{id}`: Получить одно предприятие.
- `PUT /api/v1/enterprises/{id}`: Обновить предприятие.
- `DELETE /api/v1/enterprises/{id}`: Удалить предприятие.

### Ресурс: `/api/v1/divisions`
- `GET /api/v1/divisions`: Список дивизионов (фильтр `enterprise_id`).
- `POST /api/v1/divisions`: Создать дивизион.
- `GET /api/v1/divisions/{id}`: Получить один дивизион.
- `PUT /api/v1/divisions/{id}`: Обновить дивизион.
- `DELETE /api/v1/divisions/{id}`: Удалить дивизион.

### Ресурс: `/api/v1/locations`
- `GET /api/v1/locations`: Список локаций (фильтр `division_id`).
- `POST /api/v1/locations`: Создать локацию.
- `GET /api/v1/locations/{id}`: Получить одну локацию.
- `PUT /api/v1/locations/{id}`: Обновить локацию.
- `DELETE /api/v1/locations/{id}`: Удалить локацию.

---

## 4. Справочники

### Ресурс: `/api/v1/dictionary_types`
- `GET /api/v1/dictionary_types`: Список всех типов справочников.
- `POST /api/v1/dictionary_types`: Создать новый тип.
- `GET /api/v1/dictionary_types/{id}`: Получить один тип.
- `PUT /api/v1/dictionary_types/{id}`: Обновить тип.
- `DELETE /api/v1/dictionary_types/{id}`: Удалить тип.

### Ресурс: `/api/v1/dictionaries`
- `GET /api/v1/dictionaries`: Список всех элементов всех справочников.
    - **Query Params:** `dictionary_type_id`, `enterprise_id`, `is_active`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/dictionaries`: Создать новый элемент справочника.
    - **Request Body:** `{ "name": "...", "code": "...", "dictionary_type_id": "..." }`
- `GET /api/v1/dictionaries/{id}`: Получить один элемент.
- `PUT /api/v1/dictionaries/{id}`: Обновить элемент.
- `DELETE /api/v1/dictionaries/{id}`: Удалить элемент.

---

## 5. Планы аудитов

### Ресурс: `/api/v1/audit_plans`
- `GET /api/v1/audit_plans`: Список планов аудитов.
    - **Query Params:** `enterprise_id`, `status`, `date_from`, `date_to`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/audit_plans`: Создать план.
- `GET /api/v1/audit_plans/{id}`: Получить план.
- `PUT /api/v1/audit_plans/{id}`: Обновить план.
- `DELETE /api/v1/audit_plans/{id}`: Удалить план.

### Ресурс: `/api/v1/audit_plan_items`
- `GET /api/v1/audit_plan_items`: Список пунктов плана.
    - **Query Params:** `audit_plan_id`, `planned_auditor_id`, `planned_date_from`, `planned_date_to`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/audit_plan_items`: Создать пункт плана.
- `GET /api/v1/audit_plan_items/{id}`: Получить пункт плана.
- `PUT /api/v1/audit_plan_items/{id}`: Обновить пункт плана.
- `DELETE /api/v1/audit_plan_items/{id}`: Удалить пункт плана.

---

## 6. Квалификации аудиторов

### Ресурс: `/api/v1/auditor_qualifications`
- `GET /api/v1/auditor_qualifications`: Список квалификаций.
- `POST /api/v1/auditor_qualifications`: Создать квалификацию.
- `GET /api/v1/auditor_qualifications/{id}`: Получить квалификацию.
- `PUT /api/v1/auditor_qualifications/{id}`: Обновить квалификацию.
- `DELETE /api/v1/auditor_qualifications/{id}`: Удалить квалификацию.

---

## 7. Аудиты

### Ресурс: `/api/v1/audits`
- `GET /api/v1/audits`: Список аудитов.
    - **Query Params:** `enterprise_id`, `audit_type_id`, `status_id`, `auditor_id`, `year`, `audit_date_from`, `audit_date_to`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/audits`: Создать аудит.
- `GET /api/v1/audits/{id}`: Получить аудит.
- `PUT /api/v1/audits/{id}`: Обновить аудит.
- `DELETE /api/v1/audits/{id}`: Удалить аудит.

---

### Ресурс: `/api/v1/audits/calendar` (НОВОЕ)
Специализированные эндпоинты для работы с графиком аудитов в произвольные периоды.

- `GET /api/v1/audits/calendar/schedule`: Получить график аудитов за произвольный период.
    - **Query Params:**
        - `date_from` (обязательно): начальная дата периода (ISO 8601)
        - `date_to` (обязательно): конечная дата периода (ISO 8601)
        - `enterprise_id` (опционально): фильтр по предприятию
        - `process_id` (опционально): фильтр по процессу
        - `product_id` (опционально): фильтр по продукту
        - `auditor_id` (опционально): фильтр по аудитору
        - `status_id` (опционально): фильтр по статусу
    - **Response:** Список аудитов в периоде с их компонентами, часами, статусом, результатом
    - **Пример:** `GET /api/v1/audits/calendar/schedule?date_from=2025-01-01&date_to=2025-12-31&enterprise_id=xxx`

- `GET /api/v1/audits/calendar/by_component`: Получить график по компонентам (детали, узлы, системы).
    - **Query Params:**
        - `date_from` (обязательно)
        - `date_to` (обязательно)
        - `component_type` (опционально): 'part', 'assembly', 'system', 'product', 'subsystem'
        - `sap_id` (опционально): фильтр по SAP ID
        - `enterprise_id` (опционально)
    - **Response:** Группировка аудитов по компонентам с информацией о SAP ID, количестве часов и статусе

- `POST /api/v1/audits/{id}/reschedule`: Перенести (переплан) аудит на новую дату.
    - **Request Body:**
        ```json
        {
          "new_date_from": "2025-02-15",
          "new_date_to": "2025-02-16",
          "reason": "Недоступность аудитора"
        }
        ```
    - **Response:** `200 OK` с обновленной информацией об аудите
    - **Примечание:** Автоматически заполняет `rescheduled_date`, `postponed_reason`, `rescheduled_by_id`, `rescheduled_at`

- `GET /api/v1/audits/{id}/reschedule_history`: История всех переносов аудита.
    - **Response:** Список переносов с датами, причинами, и кем переносилось

---

### Ресурс: `/api/v1/audit_components` (НОВОЕ)
Управление компонентами (деталями, узлами, системами) в рамках аудитов.

- `GET /api/v1/audit_components`: Список компонентов.
    - **Query Params:** `audit_id`, `component_type`, `sap_id`, `part_number`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/audit_components`: Создать компонент для аудита.
    - **Request Body:**
        ```json
        {
          "audit_id": "uuid",
          "component_type": "part",
          "sap_id": "123456",
          "part_number": "ABC-123",
          "component_name": "Передний бампер",
          "description": "Деталь для проверки..."
        }
        ```
- `GET /api/v1/audit_components/{id}`: Получить компонент.
- `PUT /api/v1/audit_components/{id}`: Обновить компонент.
- `DELETE /api/v1/audit_components/{id}`: Удалить компонент.
- `GET /api/v1/audits/{audit_id}/components`: Список компонентов для конкретного аудита.

---

## 8. Несоответствия (Findings)

### Ресурс: `/api/v1/findings`
- `GET /api/v1/findings`: Список несоответствий.
    - **Query Params:** `audit_id`, `enterprise_id`, `status_id`, `resolver_id`, `approver_id`, `deadline_from`, `deadline_to`, `sort_by`, `order`, `page`, `size`.
- `POST /api/v1/findings`: Создать несоответствие.
- `GET /api/v1/findings/{id}`: Получить несоответствие.
- `PUT /api/v1/findings/{id}`: Обновить несоответствие.
- `DELETE /api/v1/findings/{id}`: Удалить несоответствие.

### Ресурс: `/api/v1/finding_comments`
- `GET /api/v1/finding_comments`: Комментарии к несоответствию (фильтр `finding_id`).
- `POST /api/v1/finding_comments`: Добавить комментарий.
- `PUT /api/v1/finding_comments/{id}`: Редактировать комментарий.
- `DELETE /api/v1/finding_comments/{id}`: Удалить комментарий.

### Ресурс: `/api/v1/finding_delegations`
- `GET /api/v1/finding_delegations`: История делегирований (фильтр `finding_id`).
- `POST /api/v1/findings/{id}/delegate`: Делегировать ответственность.
- `POST /api/v1/finding_delegations/{id}/revoke`: Отозвать делегирование.

---

## 9. Вложения (Attachments)

### Ресурс: `/api/v1/attachments`
Полиморфный ресурс для работы с файлами. Вложения могут быть привязаны к любой сущности в системе (аудит, несоответствие, пользователь и т.д.) через query-параметры `object_id` и `content_type`.

- `GET /api/v1/attachments`: Список вложений (фильтры `object_id`, `content_type`).
- `POST /api/v1/attachments`: Загрузить вложение.
- `GET /api/v1/attachments/{id}`: Получить метаданные вложения.
- `DELETE /api/v1/attachments/{id}`: Удалить вложение.

---

## 10. Workflow

### Ресурс: `/api/v1/workflow/statuses`
- `GET /api/v1/workflow/statuses`: Список всех статусов.
- `POST /api/v1/workflow/statuses`: Создать новый статус.
- `PUT /api/v1/workflow/statuses/{id}`: Обновить статус.
- `DELETE /api/v1/workflow/statuses/{id}`: Удалить статус.

### Ресурс: `/api/v1/workflow/transitions`
- `GET /api/v1/workflow/transitions`: Список всех переходов между статусами.
- `POST /api/v1/workflow/transitions`: Создать новый переход.
- `PUT /api/v1/workflow/transitions/{id}`: Обновить переход.
- `DELETE /api/v1/workflow/transitions/{id}`: Удалить переход.

---

## 11. Отчеты

### Ресурс: `/api/v1/reports`
- `GET /api/v1/reports/findings`: Отчет по несоответствиям (с фильтрами).
- `GET /api/v1/reports/by_processes`: Отчет по процессам.
- `GET /api/v1/reports/by_solvers`: Отчет по исполнителям.
- `GET /api/v1/reports/.../export`: Экспорт любого отчета в Excel (добавляется `/export` к URL).

---

## 12. Дашборд

### Ресурс: `/api/v1/dashboard`
- `GET /api/v1/dashboard/stats`: Общая статистика для дашборда.
- `GET /api/v1/dashboard/my_tasks`: Задачи текущего пользователя.

---

## 13. Уведомления

### Ресурс: `/api/v1/notifications`
- `GET /api/v1/notifications`: Список уведомлений для текущего пользователя.
- `POST /api/v1/notifications/read_all`: Отметить все уведомления как прочитанные.
- `POST /api/v1/notifications/{id}/read`: Отметить конкретное уведомление как прочитанное.

### Статистика уведомлений
- `GET /api/v1/notifications/stats?days=7`: Статистика за последние N дней (до 7), агрегаты по каналам (`email`, `telegram`) и кодам событий.

### Администрирование уведомлений
- `GET /api/v1/admin/notifications/queue_stats`: Статистика очередей (pending, processing, failed), лаг, размер, скорость.
- `GET /api/v1/admin/notifications/failed`: Список неотправленных уведомлений с причинами.
- `POST /api/v1/admin/notifications/{id}/retry`: Повторная отправка конкретного уведомления.
- `POST /api/v1/admin/notifications/retry_all`: Повторная отправка всех неудачных.
- `GET /api/v1/admin/system/health`: Базовое состояние бэкенда (БД, Redis, Celery воркеры, очереди).

### Каталог событий и правила
- `GET /api/v1/admin/notification_events`: Список доступных событий.
- `POST /api/v1/admin/notification_events`: Добавить пользовательское событие.
- `PUT /api/v1/admin/notification_events/{event_code}`: Обновить метаданные события.
- `DELETE /api/v1/admin/notification_events/{event_code}`: Удалить пользовательское событие.

- `GET /api/v1/admin/notification_rules`: Список правил (фильтры: `event_code`, `entity_type`, `scope`).
- `POST /api/v1/admin/notification_rules`: Создать правило доставки.
- `GET /api/v1/admin/notification_rules/{id}`: Получить правило.
- `PUT /api/v1/admin/notification_rules/{id}`: Обновить правило.
- `DELETE /api/v1/admin/notification_rules/{id}`: Удалить правило.

### Шаблоны уведомлений
- `GET /api/v1/admin/notification_templates?event_code=&channel=`: Получить шаблоны.
- `PUT /api/v1/admin/notification_templates/{event_code}/{channel}`: Сохранить шаблон (мультиязычность).
- `POST /api/v1/admin/notifications/test_send`: Тестовая отправка по правилу/событию.

### Подписки пользователей
- `GET /api/v1/subscriptions`: Мои подписки.
- `POST /api/v1/subscriptions`: Подписаться на объект `{entity_type, entity_id, event_code?}`.
- `DELETE /api/v1/subscriptions/{id}`: Отписаться.

### Еженедельный дайджест
- `GET /api/v1/notifications/digest/preview`: Превью дайджеста.

---

## 14. Настройки

### Ресурс: `/api/v1/settings`
- `GET /api/v1/settings`: Получить список всех публичных настроек системы.
- `GET /api/v1/settings/{key}`: Получить конкретную настройку по ключу.
- `PUT /api/v1/settings/{key}`: Обновить настройку.

Примечание: Часть настроек интеграций (S3/SMTP/LDAP) управляется отдельными ресурсами `integrations/*` ниже.

---

## 15. Интеграции (подключения)

### Ресурс: `/api/v1/integrations/s3_storages`
- `GET /api/v1/integrations/s3_storages`: Список S3-подключений (фильтры: `enterprise_id`, `division_id`, `is_active`).
- `POST /api/v1/integrations/s3_storages`: Создать S3-подключение.
- `GET /api/v1/integrations/s3_storages/{id}`: Получить S3-подключение.
- `PUT /api/v1/integrations/s3_storages/{id}`: Обновить S3-подключение.
- `DELETE /api/v1/integrations/s3_storages/{id}`: Удалить S3-подключение.
- `POST /api/v1/integrations/s3_storages/{id}/test`: Проверить подключение.

### Ресурс: `/api/v1/integrations/email_accounts`
- `GET /api/v1/integrations/email_accounts`: Список email-аккаунтов (фильтры: `enterprise_id`, `division_id`, `is_active`).
- `POST /api/v1/integrations/email_accounts`: Создать email-аккаунт.
- `GET /api/v1/integrations/email_accounts/{id}`: Получить email-аккаунт.
- `PUT /api/v1/integrations/email_accounts/{id}`: Обновить email-аккаунт.
- `DELETE /api/v1/integrations/email_accounts/{id}`: Удалить email-аккаунт.
- `POST /api/v1/integrations/email_accounts/{id}/test`: Проверить подключение SMTP.

### Ресурс: `/api/v1/integrations/ldap_connections`
- `GET /api/v1/integrations/ldap_connections`: Список LDAP-подключений (фильтры: `enterprise_id`, `division_id`, `is_active`).
- `POST /api/v1/integrations/ldap_connections`: Создать LDAP-подключение.
- `GET /api/v1/integrations/ldap_connections/{id}`: Получить LDAP-подключение.
- `PUT /api/v1/integrations/ldap_connections/{id}`: Обновить LDAP-подключение.
- `DELETE /api/v1/integrations/ldap_connections/{id}`: Удалить LDAP-подключение.
- `POST /api/v1/integrations/ldap_connections/{id}/test`: Проверить подключение LDAP.

Примечание: для маршрутизации можно указывать `enterprise_id`/`division_id` (nullable). Подключение с `is_default=true` используется по умолчанию.

---

## 16. История изменений

### Ресурс: `/api/v1/change_history`
- `GET /api/v1/change_history`: Список изменений (фильтры: `entity_type`, `entity_id`, `user_id`, диапазон дат).

Также доступны укороченные представления:
- `GET /api/v1/audits/{id}/history`
- `GET /api/v1/findings/{id}/history`
