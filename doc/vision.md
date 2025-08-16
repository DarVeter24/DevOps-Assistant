# Техническое видение проекта Smart DevSecOps/MLOps Assistant

## Содержание

1. Технологии
2. Принцип разработки  
3. Архитектура проекта
4. Структура проекта
5. Модель данных
6. Работа с LLM
7. Мониторинг LLM
8. Сценарии работы
9. Деплой
10. Подход к конфигурированию
11. Подход к логгированию
12. CI/CD pipeline RAG/CAG

---

## Описание
Техническое видение для разработки Smart DevSecOps/MLOps Assistant - разговорного помощника для автоматизации DevOps задач с использованием технологий дообучения LLM модели через RAG + CAG.

**Текущий статус:** Основная MLOps инфраструктура (K8s, Kafka, Airflow, MLFlow, KServe, Spark) уже развернута на baremetal.

---

## 1. Технологии

### ML/AI компоненты
- **LLM**: GPT-OSS-20b/Gemma 3 270M for chat (локальный деплой)
- **Vector Search**: OpenSearch 2.x с Vector Search + Neural Search
- **Embeddings**: sentence-transformers

### Приложение
- **Backend**: FastAPI
- **Frontend**: Open WebUI (готовый чат-интерфейс)
- **Automation**: N8N для workflow'ов

### Devops Инфраструктура
- **Orchestration**: Kubernetes + ArgoCD (GitOps)
- **Monitoring**: Prometheus + Grafana + Alertmanager
- **Logging**: OpenSearch Stack (OpenSearch + Logstash + OpenSearch Dashboards)
- **Storage**: PostgreSQL + MinIO + Nexus + Redis
- **Security**: HashiCorp Vault + KSOPS

### MLops платформа
- **Streaming**: Apache Kafka
- **Processing**: Apache Spark
- **Orchestration**: Apache Airflow
- **Tracking**: MLFlow
- **Serving**: KServe

### Интеграция с MLOps стеком
- Kafka, Airflow, MLFlow, Spark, KServe

## 2. Принцип разработки

### Архитектурные принципы
- **Модульность** - разделение на RAG и CAG пайплайны
- **Микросервисная архитектура** - компоненты развертываются отдельно
- **Event-driven** - использование Kafka для стриминга событий

### Качество кода
- **Контейнеризация** - все компоненты в Docker/Kubernetes
- **GitOps** - управление через ArgoCD
- **Мониторинг** - полное покрытие метриками и логами

## 3. Архитектура проекта

### Архитектурные принципы
1. **Компонентный подход** - четкое разделение ответственности между модулями
2. **Функциональное программирование** - использование функций вместо классов  
3. **Простая схема взаимодействия**: User → Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b)

### 🏗️ Архитектурная концепция

**Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b/Gemma 3 270M)**

```
┌─────────────────────────────────────────────────────────┐
│                 Open WebUI                              │
│              Диалоговый интерфейс                      │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                FastAPI Backend                         │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐ │
│ │  RAG Pipeline   │ │  CAG Pipeline   │ │ Action      │ │
│ │ • Vector Search │ │ • Runbook Cache │ │ Agent       │ │
│ │ • Real-time Logs│ │ • Markdown Docs │ │ • kubectl   │ │
│ │ • Metrics       │ │ • Procedures    │ │ • Prometheus│ │
│ └─────────────────┘ └─────────────────┘ └─────────────┘ │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │          Адаптер OpenAI ↔ KServe                    │ │
│ │   • Преобразование запросов/ответов                 │ │
│ │   • Стриминг токенов                                │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                    KServe                              │
│          Kubernetes Model Serving                      │
│    • Масштабирование модели                            │
│    • Health checks                                     │
│    • Роутинг запросов                                  │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              LLM model (GPT-OSS-20b/Gemma 3 270M)      │
│              Языковая модель                           │
│    • Генерация ответов                                 │
│    • Обработка промптов                                │
└─────────────────────────────────────────────────────────┘
```

### Поток данных

**User → Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b/Gemma 3 270M)**

**User**  
Пишешь/говоришь запрос.

**→ Open WebUI**  
Удобный чат-интерфейс: принимает ввод, шлёт его в формате OpenAI API на твой бэкенд и показывает ответ (при желании — со звуком).

**→ FastAPI (оркестратор)**

- **RAG/CAG:** подмешивает факты из векторки/логов/метрик (RAG) и быстрые знания из кэша/рунбуков (CAG).
- **Агент:** решает, нужны ли действия сверх текстового ответа; выбирает и вызывает разрешённые инструменты (kubectl, Prometheus, и т.п.); проверяет параметры и права (confirm/dry-run/RBAC), при необходимости запрашивает подтверждение; исполняет действие, логирует его и подкладывает результат в контекст ответа.
- **Адаптер OpenAI↔KServe:** переводит запрос/ответ между форматом OpenAI-чата и протоколом KServe (и, при необходимости, организует стриминг токенов).

**→ KServe**  
Kubernetes-сервис инференса: принимает запрос в своём REST/gRPC формате, маршрутизирует его к контейнеру с моделью, масштабирует и следит за здоровьем.

**→ LLM model (GPT-OSS-20b/Gemma 3 270M)**  
Сама LLM-модель: генерирует ответ по полученному промпту и возвращает его назад по цепочке  

**LLM model (GPT-OSS-20b/Gemma 3 270M) → KServe → FastAPI (agent, адаптер OpenAI KServe, RAG/CAG Pipelines) → Open WebUI → User**

## 4. Структура проекта

### Файловая структура проекта (Runtime)
```
/
├── main.py                   # Запуск FastAPI backend
├── config.py                 # Конфигурация (OpenSearch, Redis, KServe)
├── api/                      # FastAPI endpoints для Open WebUI
│   ├── __init__.py
│   ├── chat.py               # /v1/chat/completions endpoint
│   └── middleware.py         # CORS, логирование, аутентификация
├── core/                     # Основная логика
│   ├── __init__.py
│   └── chat_handler.py       # Оркестрация запросов (CAG → RAG → LLM)
├── rag/                      # RAG компоненты (runtime)
│   ├── __init__.py
│   └── search.py             # Поиск в OpenSearch (BM25/векторный)
├── cag/                      # CAG компоненты (runtime)
│   ├── __init__.py
│   └── cache.py              # Работа с Redis-кэшем
├── llm/                      # LLM интеграция
│   ├── __init__.py
│   ├── kserve_client.py      # Клиент для KServe/vLLM backend
│   └── openai_adapter.py     # Адаптер OpenAI → backend (и обратно)
├── schemas/                  # Pydantic модели запросов/ответов
│   └── chat_models.py        # Модели для /v1/chat/completions
├── data/                     # Работа с исходными данными
│   ├── raw/                  # Исходники (md/txt/pdf→txt)
│   └── loader.py             # Скрипт загрузки/индексации в OpenSearch
├── dags/                     # DAGи Airflow (V1+)
│   ├── index_docs.py         # ETL: extract → transform → embed → load → invalidate_cache
│   ├── rag_pipeline.py       # Периодическое обновление векторной БД
│   ├── cag_pipeline.py       # Обслуживание/прогрев/инвалидация кэша
│   └── data_processors/           # Шаги, переиспользуемые в DAGах
│       ├── embedding_generator.py # Подсчёт эмбеддингов (batch/mini-batch)
│       ├── opensearch_indexer.py  # Upsert/создание индексов, маппинги, алиасы
│       └── redis_updater.py       # Массовое обновление/инвалидация ключей Redis
├── Dockerfile                # Контейнер для деплоя
└── .env                      # Переменные окружения (в проде — через Secrets)
```

## 5. Модель данных

### Источники данных
- **Динамические**: Prometheus метрики, OpenSearch логи, Kubernetes API
- **Статические**: Runbooks, документация, troubleshooting guides
- **ML метаданные**: MLFlow эксперименты, модели, артефакты

### RAG Pipeline структуры
- **Vector Search** - поиск по метрикам и логам
- **Real-time Logs** - обработка логов в реальном времени
- **Metrics** - анализ метрик Prometheus

### CAG Pipeline структуры  
- **Runbook Cache** - кэш процедур и инструкций
- **Markdown Docs** - документация в формате Markdown
- **Procedures** - пошаговые процедуры

## 6. Работа с LLM

### Подход к взаимодействию с языковой моделью

1. **Локальный деплой**: LLM model (GPT-OSS-20b/Gemma 3 270M) через KServe
2. **RAG обогащение**: подмешивание контекста из метрик/логов
3. **CAG кэширование**: быстрый доступ к статическим знаниям
4. **Action Agent**: выполнение действий через инструменты

### Интеграция с инфраструктурой
- **KServe** - сервинг модели в Kubernetes
- **OpenAI API совместимость** - через адаптер
- **WebSocket стриминг** - для real-time ответов

## 7. Мониторинг LLM

### Мониторинг инфраструктуры
- **Prometheus + Grafana** - метрики производительности
- **Alertmanager** - алерты и уведомления
- **OpenSearch Stack** - централизованное логирование

### Мониторинг ML компонентов
- **MLFlow** - трекинг экспериментов и моделей
- **KServe метрики** - производительность инференса
- **Vector Search метрики** - эффективность поиска

## 8. Сценарии работы

### Основные сценарии
1. **Диагностика инфраструктуры** - анализ метрик и логов
2. **Troubleshooting** - поиск и решение проблем
3. **Автоматизация задач** - выполнение DevOps операций
4. **Обучение команды** - предоставление знаний и документации

### Типы взаимодействия
- **Чат-интерфейс** - основной способ взаимодействия
- **Action Agent** - выполнение команд и скриптов  
- **Real-time мониторинг** - непрерывный анализ состояния

## 9. Деплой

### Kubernetes деплоймент
1. **ArgoCD** - GitOps управление
2. **Helm Charts** - пакетирование компонентов
3. **KServe** - деплой LLM модели
4. **Ingress** - внешний доступ к UI

### Компоненты для деплоя
- **Open WebUI** - готовый frontend (чат-интерфейс)
- **FastAPI backend** - API и оркестратор
- **RAG/CAG сервисы** - обработка данных
- **LLM model (GPT-OSS-20b/Gemma 3 270M)** - языковая модель в KServe

## 10. Подход к конфигурированию

### Конфигурация через:
- **Kubernetes ConfigMaps** - настройки приложений
- **HashiCorp Vault** - секреты и токены
- **KSOPS** - GitOps управление секретами
- **Helm Values** - параметризация деплоя

## 11. Подход к логгированию

### Централизованное логирование
1. **OpenSearch Stack** - полностью открытая альтернатива ELK:
   - **OpenSearch** - поисковая система (форк Elasticsearch)
   - **Logstash** - сбор и обработка логов
   - **OpenSearch Dashboards** - визуализация (форк Kibana)
2. **Structured logging** - JSON формат логов
3. **Correlation ID** - трекинг запросов через компоненты

### Типы логов
- **Application logs** - логи приложений
- **Infrastructure logs** - логи Kubernetes и системы
- **ML Pipeline logs** - логи ML процессов
- **Audit logs** - логи безопасности и доступа

## 12. CI/CD pipeline RAG/CAG

Автоматизированный pipeline для непрерывной подготовки и обновления данных в RAG (динамические метрики/логи) и CAG (статическая документация) системах. Pipeline включает 5 основных этапов: от сбора данных через Kafka до индексации в OpenSearch и кэширования в Redis. Система обеспечивает обновление RAG данных каждые 30 секунд для real-time мониторинга и еженедельное обновление CAG базы знаний. Весь процесс оркестрируется через Airflow DAGs с полным мониторингом через MLFlow и соблюдением SLA < 200ms для поиска.

📄 **Детальное руководство:** [pipeline-rag-cag.md](additional/pipeline-rag-cag.md)