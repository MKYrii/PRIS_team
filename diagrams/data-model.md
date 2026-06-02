# Структура данных

## ER-диаграмма (реляционная часть — PostgreSQL)

```mermaid
erDiagram
    COMMENTS {
        uuid id PK
        uuid platform_post_id FK
        uuid platform_user_id FK
        text content_hash
        text content_anonymized
        timestamp created_at
        timestamp ingested_at
        varchar lang
    }

    CLASSIFICATION_RESULTS {
        uuid id PK
        uuid comment_id FK
        float score_violation
        float score_toxic
        float score_hate
        float score_spam
        varchar label
        varchar model_version
        int latency_ms
        timestamp classified_at
    }

    MODERATION_DECISIONS {
        uuid id PK
        uuid comment_id FK
        uuid classification_id FK
        varchar decision
        varchar decision_source
        uuid reviewer_id FK
        varchar reason_code
        timestamp decided_at
    }

    HUMAN_REVIEW_QUEUE {
        uuid id PK
        uuid comment_id FK
        uuid classification_id FK
        float score_violation
        varchar priority
        varchar status
        uuid assigned_to FK
        timestamp enqueued_at
        timestamp resolved_at
    }

    FEEDBACK_LABELS {
        uuid id PK
        uuid comment_id FK
        uuid reviewer_id FK
        varchar true_label
        varchar model_label
        boolean is_error
        text notes
        timestamp labeled_at
    }

    MODERATORS {
        uuid id PK
        varchar username
        varchar role
        boolean is_active
    }

    MODEL_VERSIONS {
        varchar version PK
        varchar model_name
        float precision_val
        float recall_val
        float f1_score
        varchar status
        timestamp deployed_at
    }

    COMMENTS ||--o{ CLASSIFICATION_RESULTS : "classified_by"
    COMMENTS ||--o{ MODERATION_DECISIONS : "decided_for"
    COMMENTS ||--o| HUMAN_REVIEW_QUEUE : "queued_as"
    COMMENTS ||--o{ FEEDBACK_LABELS : "labeled_as"
    CLASSIFICATION_RESULTS ||--o{ MODERATION_DECISIONS : "based_on"
    CLASSIFICATION_RESULTS ||--o{ HUMAN_REVIEW_QUEUE : "triggers"
    MODERATORS ||--o{ MODERATION_DECISIONS : "makes"
    MODERATORS ||--o{ HUMAN_REVIEW_QUEUE : "assigned_to"
    MODERATORS ||--o{ FEEDBACK_LABELS : "provides"
    MODEL_VERSIONS ||--o{ CLASSIFICATION_RESULTS : "produces"
```

---

## Распределённая структура хранилищ

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ДАННЫЕ СИСТЕМЫ SentinelAI                       │
│                                                                     │
│  ┌───────────────────┐    ┌───────────────────┐                     │
│  │   Apache Kafka    │    │    Redis Cache    │                     │
│  │  (Streaming)      │    │  (Hot Data)       │                     │
│  │                   │    │                   │                     │
│  │ • comments-raw    │    │ • user_features   │                     │
│  │ • decisions-out   │    │ • recent_scores   │                     │
│  │ • feedback-in     │    │ • model_warmup    │                     │
│  │                   │    │                   │                     │
│  │ Retention: 7 дней │    │ TTL: 1–24 ч       │                     │
│  └───────────────────┘    └───────────────────┘                     │
│                                                                     │
│  ┌───────────────────┐    ┌───────────────────┐                     │
│  │   PostgreSQL      │    │   ClickHouse      │                     │
│  │  (Operational)    │    │  (Analytics)      │                     │
│  │                   │    │                   │                     │
│  │ • comments        │    │ • events_hourly   │                     │
│  │ • decisions       │    │ • sentiment_daily │                     │
│  │ • review_queue    │    │ • model_metrics   │                     │
│  │ • feedback_labels │    │ • content_trends  │                     │
│  │ • moderators      │    │                   │                     │
│  │                   │    │ Partitioned by    │                     │
│  │ Retention: 90 дней│    │ date (monthly)    │                     │
│  └───────────────────┘    └───────────────────┘                     │
│                                                                     │
│  ┌───────────────────┐    ┌───────────────────┐                     │
│  │   S3 / MinIO      │    │   MLflow          │                     │
│  │  (Cold Storage)   │    │  (ML Registry)    │                     │
│  │                   │    │                   │                     │
│  │ • raw_comments/   │    │ • model artifacts │                     │
│  │ • audit_logs/     │    │ • experiments     │                     │
│  │ • training_data/  │    │ • metrics history │                     │
│  │ • model_backups/  │    │ • feature schemas │                     │
│  │                   │    │                   │                     │
│  │ Retention: 3+ года│    │ Retention: всегда │                     │
│  └───────────────────┘    └───────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Почему данные распределены?

| Хранилище | Причина выбора |
|---|---|
| **Kafka** | Декаплинг продюсеров и консьюмеров; буфер при пиках нагрузки; гарантия доставки exactly-once |
| **Redis** | Latency < 1 мс для фич пользователя в момент инференса; TTL-управление |
| **PostgreSQL** | ACID-транзакции для решений модераторов; JOIN-запросы для очереди |
| **ClickHouse** | Columnar-хранение; агрегации по 500K событий/день за секунды |
| **S3** | Дешёвое хранение сырых данных; неограниченный объём; основа для обучения |
| **MLflow** | Стандарт версионирования моделей; сравнение экспериментов; reproducibility |

### Политика хранения и конфиденциальность

- **Сырое содержимое комментариев** не хранится в PostgreSQL — только `content_hash` и анонимизированная версия.
- **Полный текст** хранится в S3 с шифрованием AES-256, доступ по RBAC.
- Данные старше 3 лет удаляются в соответствии с ФЗ-152 (автоматический TTL).
- `platform_user_id` — хешированный псевдоним, не позволяет идентифицировать пользователя без ключа платформы.
