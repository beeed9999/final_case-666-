# Модель данных — кейс 2 «Кредитный конвейер»

Сущности: **Клиент**, **Заявка**, **Решение**, **Кредитный продукт**. Дополнительно — **Внешний запрос** (БКИ/СМЭВ) и **Событие аудита**, они нужны для интеграции и ИБ.

## ER-диаграмма

```mermaid
erDiagram
    CLIENT ||--o{ APPLICATION : "подаёт"
    PRODUCT ||--o{ APPLICATION : "выбран в"
    APPLICATION ||--o| DECISION : "получает"
    APPLICATION ||--o{ EXTERNAL_REQUEST : "порождает"
    APPLICATION ||--o{ AUDIT_EVENT : "журналируется"
    DECISION ||--o{ AUDIT_EVENT : "журналируется"

    CLIENT {
        string client_id PK "cli_NNNN"
        string passport_hash "16 hex, SHA-256, оригинал не хранится"
    }
    PRODUCT {
        string product_id PK "prod_standard"
        string name
        int min_amount
        int max_amount
        int min_term
        int max_term
        float rate "0.18"
        int min_age
        int max_age
        int min_income
    }
    APPLICATION {
        string app_id PK "app_NNNN"
        string client_id FK
        string product_id FK
        int age "18-70"
        float income "больше 0"
        string employment "5 значений"
        string family "3 значения"
        float amount "10k-1M"
        int term "3-60 мес."
    }
    DECISION {
        string app_id PK, FK
        string decision "APPROVED / REFUSED / NEED_MORE_DATA / MANUAL_REVIEW"
        string reason "машинный код"
        int score "0-100"
        float dti "от 0"
        float amount_approved
        int term_approved
        float rate
    }
    EXTERNAL_REQUEST {
        string request_id PK "req_NNNN"
        string app_id FK
        string service "BKI / SMEV"
        string status "SUCCESS / TIMEOUT / ERROR"
        int latency_ms
        json result "bki_score, delinquency, smev_status"
    }
    AUDIT_EVENT {
        uuid event_id PK
        string app_id FK
        string action "application_created / decision_made / recalculated"
        string decision
        string reason
        int score
        string actor
        datetime at "UTC"
    }
```

## Описание сущностей и связей

|Сущность|Назначение|Ключ|Связи|
|-|-|-|-|
|Client|Заёмщик. Хранится только хэш паспорта|client_id|1 клиент — много заявок|
|Application|Кредитная заявка: данные анкеты и параметры запроса|app_id|N:1 к Client, N:1 к Product|
|Decision|Предварительное решение по заявке|app_id|1:1 с Application (после расчёта; до этого 425)|
|Product|Условия продукта: суммы, сроки, ставка, возраст, минимальный доход|product_id|1 продукт — много заявок|
|ExternalRequest|Обращение к БКИ/СМЭВ: статус, время, результат|request_id|N:1 к Application, обычно 2 на заявку|
|AuditEvent|Неизменяемая запись о действии (append-only)|event_id|N:1 к Application|

## Соответствие файлам кейса

* Поля Application, Decision, ExternalRequest — `05_api_schemas.yaml`.
* Поля `applications.csv` (14 колонок) — `data_dictionary_sdel.md`; в CSV client_id, passport_hash и результаты БКИ/СМЭВ лежат в одной строке с заявкой для простоты прототипа.
* Продукты — `products.xlsx`; в версии кейса работает один продукт `prod_standard`, остальные (premium, student, pension, express) заведены в справочник как задел.
* Аудит — `ИБ.md`, п. 3.

## Ограничения модели

* Оригиналы паспортов и номера карт не хранятся (ИБ.md, п. 4).
* `reason` — машинный код из каталога `08_reasons_sdel.md`; текст клиенту берётся из каталога, а не хранится в решении.
* Поля пола и национальности в модели отсутствуют намеренно (NFR-05).
