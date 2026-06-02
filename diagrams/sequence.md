# UML-диаграммы поведения

## Диаграмма прецедентов (Use Case Diagram)

```mermaid
graph LR
    MOD((Модератор))
    ADMIN((Администратор))
    PLATFORM((Платформа\nсоцсети))
    RKN((РКН /\nРегулятор))
    ML((ML-система\nавтоматизация))

    UC1[Публикация комментария]
    UC2[Автоматическая классификация]
    UC3[Автоматическое скрытие]
    UC4[Просмотр очереди]
    UC5[Ручная проверка]
    UC6[Разметка ошибок ML]
    UC7[Просмотр аналитики]
    UC8[Выгрузка отчёта для РКН]
    UC9[Управление моделями]
    UC10[Мониторинг качества]

    PLATFORM --> UC1
    UC1 --> UC2
    ML --> UC2
    ML --> UC3
    UC3 -.->|include| UC2
    MOD --> UC4
    MOD --> UC5
    MOD --> UC6
    MOD --> UC7
    ADMIN --> UC7
    ADMIN --> UC8
    ADMIN --> UC9
    ADMIN --> UC10
    RKN --> UC8
```

---

## Диаграмма последовательности: Автоматическая модерация (Happy Path)

```mermaid
sequenceDiagram
    actor User as Пользователь
    participant Plat as Платформа
    participant GW as API Gateway
    participant Ingest as Ingest Service
    participant Kafka as Kafka
    participant FE as Feature Extractor
    participant Redis as Redis
    participant ML as ML Inference
    participant MAS as Moderation Action
    participant DB as PostgreSQL
    participant S3 as S3

    User->>Plat: Публикует комментарий
    Plat->>GW: POST /api/comments {text, user_id, post_id}
    GW->>GW: Auth check, rate-limit
    GW->>Ingest: Forward request
    Ingest->>Ingest: Validate + anonymize
    Ingest-->>Plat: 202 Accepted (comment_id)
    Ingest->>Kafka: publish(comments-raw, comment)
    Ingest->>S3: backup(raw_comment) [async]

    Note over Kafka,ML: Async processing

    Kafka->>FE: consume(comment)
    FE->>Redis: GET user_features:{user_id}
    Redis-->>FE: {spam_history, report_count, ...}
    FE->>FE: tokenize + extract features
    FE->>ML: classify(features)
    ML->>ML: model.inference(tokens)
    Note right of ML: latency ~80–300 ms

    alt score >= 0.85 (нарушение, высокая уверенность)
        ML->>MAS: route(result, action=AUTO_HIDE)
        MAS->>Kafka: publish(decisions-out, {comment_id, HIDE})
        MAS->>DB: INSERT moderation_decisions
        Kafka->>Plat: apply_decision(comment_id, HIDE)
        Plat-->>User: Уведомление об удалении
    else 0.40 <= score < 0.85 (неопределённость)
        ML->>MAS: route(result, action=QUEUE)
        MAS->>DB: INSERT human_review_queue
        Note right of DB: Модератор увидит в очереди
    else score < 0.40 (норма)
        Note right of ML: Комментарий публикуется без действий
    end

    ML->>DB: INSERT classification_results [async]
```

---

## Диаграмма последовательности: Ручная проверка + Feedback

```mermaid
sequenceDiagram
    actor Mod as Модератор
    participant UI as Review Queue UI
    participant QS as Queue Service
    participant DB as PostgreSQL
    participant Kafka as Kafka (feedback-in)
    participant Train as Training Pipeline

    Mod->>UI: Открывает очередь
    UI->>QS: GET /queue/items?status=pending
    QS->>DB: SELECT from human_review_queue\nWHERE status='pending'\nORDER BY score DESC
    DB-->>QS: [{comment, score, ml_label, ...}]
    QS-->>UI: items list

    loop Для каждого элемента
        Mod->>UI: Просматривает комментарий
        Mod->>UI: Принимает решение (OK / VIOLATION)
        UI->>QS: POST /queue/{id}/decision {label, reason}
        QS->>DB: UPDATE queue SET status='resolved'
        QS->>DB: INSERT moderation_decisions
        QS->>DB: INSERT feedback_labels

        alt Решение модератора ≠ решение ML
            QS->>Kafka: publish(feedback-in, {comment_id, true_label})
            Note right of Kafka: Помечается как ошибка ML
        end
    end

    Note over Kafka,Train: Еженедельный retrain

    Train->>DB: SELECT feedback_labels (last 7 days)
    Train->>Train: Aggregate training batch
    Train->>Train: Fine-tune model
    Train->>Train: Evaluate metrics
    alt metrics > previous model
        Train->>Train: Promote to production
        Note right of Train: Новая модель в production
    else metrics worse
        Train->>Train: Keep current model
        Train->>Mod: Alert: retrain degraded
    end
```

---

## Диаграмма активностей: Onboarding новой ML-модели

```mermaid
flowchart TD
    START([Начало: новая версия модели]) --> TRAIN[Обучение на обновлённых данных\nTraining Pipeline]
    TRAIN --> EVAL{Метрики выше\nпорога?}
    EVAL -- Нет --> TUNE[Тюнинг гиперпараметров\nAirflow DAG]
    TUNE --> TRAIN
    EVAL -- Да --> REGISTER[Регистрация в MLflow\nстатус: Staging]
    REGISTER --> SHADOW[Shadow Mode: 10% трафика\n48 часов]
    SHADOW --> SHADOW_OK{Online-метрики\nв норме?}
    SHADOW_OK -- Нет --> ROLLBACK[Оставить предыдущую\nмодель в production]
    ROLLBACK --> ALERT[Alert команде ML]
    SHADOW_OK -- Да --> CANARY[Canary: 20% трафика\n24 часа]
    CANARY --> CANARY_OK{Нет деградации\nSLA?}
    CANARY_OK -- Нет --> ROLLBACK
    CANARY_OK -- Да --> PROMOTE[Promote to Production\n100% трафика]
    PROMOTE --> MONITOR[Мониторинг\nDrift Detection активен]
    MONITOR --> END([Конец: модель в production])
    ALERT --> END2([Конец: возврат к старой модели])
```
