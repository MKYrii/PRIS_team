# Архитектура системы

## Общая архитектурная диаграмма

```mermaid
flowchart TB
    subgraph EXT["Внешние системы"]
        PLATFORM[Платформа соцсети\nComment API]
        USERS[Пользователи]
        RKN[РКН / Регулятор]
    end

    subgraph INGESTION["Слой приёма (Ingestion Layer)"]
        GW[API Gateway\nKong/Nginx]
        INGEST[Comment Ingest\nService\nFastAPI]
    end

    subgraph STREAMING["Потоковый слой (Streaming Layer)"]
        KAFKA_IN[Kafka\ncomments-raw]
        KAFKA_OUT[Kafka\ndecisions-out]
        KAFKA_FB[Kafka\nfeedback-in]
    end

    subgraph ML_LAYER["ML-слой (Inference Layer)"]
        REDIS[Redis\nFeature Cache]
        INFERENCE[ML Inference\nService × N pods\nTorchServe / Triton]
        FEATURIZER[Feature\nExtractor]
    end

    subgraph ACTION["Слой решений (Action Layer)"]
        MOD_SVC[Moderation\nAction Service\nGo]
        QUEUE_SVC[Human Review\nQueue Service]
    end

    subgraph STORAGE["Слой хранения (Distributed Storage)"]
        PG[(PostgreSQL\nOperational DB)]
        CH[(ClickHouse\nAnalytics DB)]
        S3[(S3 / MinIO\nCold Storage)]
        MLFLOW[(MLflow\nModel Registry)]
    end

    subgraph ML_OPS["ML-платформа (MLOps)"]
        TRAINER[Training\nPipeline\nAirflow DAG]
        MONITOR[Model Monitor\nDrift Detection]
    end

    subgraph OBS["Наблюдаемость (Observability)"]
        PROM[Prometheus\n+ Alertmanager]
        GRAF[Grafana\nDashboard]
        SUPERSET[Apache Superset\nAnalytics UI]
    end

    USERS --> PLATFORM
    PLATFORM --> GW
    GW --> INGEST
    INGEST --> KAFKA_IN
    INGEST --> S3

    KAFKA_IN --> FEATURIZER
    FEATURIZER --> REDIS
    FEATURIZER --> INFERENCE
    REDIS --> INFERENCE

    INFERENCE --> MOD_SVC
    INFERENCE --> QUEUE_SVC
    INFERENCE --> CH

    MOD_SVC --> KAFKA_OUT
    MOD_SVC --> PG
    KAFKA_OUT --> PLATFORM

    QUEUE_SVC --> PG
    QUEUE_SVC --> GRAF

    PG --> KAFKA_FB
    KAFKA_FB --> TRAINER

    S3 --> TRAINER
    TRAINER --> MLFLOW
    MLFLOW --> INFERENCE

    MONITOR --> INFERENCE
    MONITOR --> PROM

    PG --> SUPERSET
    CH --> SUPERSET
    SUPERSET --> RKN

    INFERENCE --> PROM
    MOD_SVC --> PROM
    INGEST --> PROM
    PROM --> GRAF

    style ML_LAYER fill:#ede9fe,stroke:#7c3aed,color:#000000,font-weight:bold
    style ML_OPS fill:#ede9fe,stroke:#7c3aed,color:#000000,font-weight:bold
    style STREAMING fill:#fff7ed,stroke:#f97316,color:#000000,font-weight:bold
    style OBS fill:#f0fdf4,stroke:#22c55e,color:#000000,font-weight:bold
```

---

## Потоки данных

### Realtime-путь (критический путь, ≤ 5 сек)

```
Комментарий опубликован
        │
        ▼
  API Gateway (auth, rate-limit)  ~5 мс
        │
        ▼
  Comment Ingest Service          ~10 мс
        │
        ├──→ S3 (raw backup, async)
        │
        ▼
  Kafka: comments-raw             ~5 мс
        │
        ▼
  ML Inference Service            ~80–300 мс
  (feature extraction + model)
        │
        ├── score ≥ 0.85 ──→ Moderation Action Service → скрытие → Kafka out
        ├── 0.40–0.85  ──→ Human Review Queue → модератор
        └── score < 0.40 ──→ комментарий опубликован
        │
        ├──→ ClickHouse (async, аналитика)
        └──→ Prometheus (метрики инференса)
```

### Аналитический путь (eventual consistency, секунды)

```
ClickHouse ←── инференс-события (батчи 1 сек)
      │
      └──→ Superset / Grafana: тренды, топ-посты, карта токсичности
```

### ML-путь (еженедельный retrain)

```
Feedback Labels (PostgreSQL)
  + Historical Data (S3)
        │
        ▼
  Airflow DAG: Training Pipeline
        │
        ├── Data validation (Great Expectations)
        ├── Fine-tuning (GPU node)
        ├── Evaluation (offline metrics)
        ├── A/B shadow test
        └── Promotion to production
              │
              ▼
        MLflow → Inference Service (rolling update)
```

---

## Обоснование распределённого хранения

| Вопрос | Ответ |
|---|---|
| Почему не одна БД? | PostgreSQL не справится с OLAP-запросами по 500K событий/день; ClickHouse не подходит для транзакционных операций очереди |
| Почему Kafka, а не HTTP? | При пиковой нагрузке 50 RPS и latency ML 300 мс нужен буфер; Kafka позволяет ML-воркерам обрабатывать в своём темпе |
| Почему S3 для сырых данных? | Стоимость хранения в 10–100× дешевле RDB; нужен полный аудит для РКН за 3+ года |
| Почему Redis? | User-level фичи нужны за < 1 мс до инференса; PostgreSQL не даст такой latency |
