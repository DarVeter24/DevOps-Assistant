# Техническое видение проекта Smart DevSecOps/MLOps Assistant

## Содержание

1. Технологии
2. Принцип разработки  
3. Структура проекта
4. Архитектура проекта
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
Техническое видение для разработки Smart DevSecOps/MLOps Assistant - разговорного помощника для автоматизации DevOps задач через LLM модель и UI chat-интерфейс с использованием технологий дообучения LLM модели через RAG + CAG.

**Текущий статус:** Основная MLOps инфраструктура (K8s, Kafka, Airflow, MLFlow, KServe, Spark) уже развернута на baremetal.

---

## 1. Технологии

### ML/AI компоненты
- **LLM**: GPT-OSS-20b Chat (локальный деплой)
- **Vector Search**: OpenSearch 2.x с Vector Search + Neural Search
- **Embeddings**: sentence-transformers

### Инфраструктура
- **Orchestration**: Kubernetes + ArgoCD (GitOps)
- **Monitoring**: Prometheus + Grafana + Alertmanager
- **Logging**: OpenSearch Stack (OpenSearch + Logstash + OpenSearch Dashboards)
- **Storage**: PostgreSQL + MinIO + Nexus + Redis
- **Security**: HashiCorp Vault + KSOPS

### ML платформа
- **Streaming**: Apache Kafka
- **Processing**: Apache Spark
- **Orchestration**: Apache Airflow
- **Tracking**: MLFlow
- **Serving**: KServe

### Приложение
- **Backend**: FastAPI
- **Frontend**: Open WebUI (готовый чат-интерфейс)
- **Automation**: N8N для workflow'ов

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

## 3. Структура проекта

### Основные компоненты системы
```
Smart DevSecOps/MLOps Assistant
├── RAG Pipeline (векторный поиск в метриках/логах)
├── CAG Pipeline (кэширование runbooks)
├── LLM Integration (LLM model GPT-OSS-20b)
└── MLOps Infrastructure:
    ├── MLFlow (трекинг экспериментов) ✅ развернут
    ├── Airflow (оркестрация пайплайнов) ✅ развернут
    ├── Kafka (стриминг событий) ✅ развернут
    ├── KServe (сервинг моделей) ✅ развернут
    └── Spark (обработка данных) ✅ развернут
```

### Файловая структура проекта (Runtime)
```
/
├── main.py                    # Запуск FastAPI backend
├── config.py                  # Конфигурация (OpenSearch, Redis, KServe)
├── api/                       # FastAPI endpoints для Open WebUI
│   ├── __init__.py
│   ├── chat.py              # /v1/chat/completions endpoint
│   └── middleware.py        # CORS для Open WebUI
├── rag/                       # RAG компоненты (runtime)
│   ├── __init__.py
│   └── search.py            # Поиск в OpenSearch
├── cag/                      # CAG компоненты (runtime)  
│   ├── __init__.py
│   └── cache.py             # Поиск в Redis кэше
├── llm/                      # LLM интеграция
│   ├── __init__.py
│   ├── kserve_client.py     # Клиент для KServe
│   └── openai_adapter.py    # Адаптер OpenAI→KServe
├── core/                     # Основная логика
│   ├── __init__.py
│   └── chat_handler.py      # Обработка запросов (RAG+CAG+LLM)
├── Dockerfile               # Контейнер для деплоя
└── .env                    # Переменные окружения
```

### Pipeline компоненты (отдельно, для Airflow)
```
airflow_dags/
├── rag_pipeline.py          # DAG обновления векторной базы
├── cag_pipeline.py          # DAG обновления кэша  
└── data_processors/
    ├── embedding_generator.py
    ├── opensearch_indexer.py
    └── redis_updater.py
```


## 4. Архитектура проекта

### Архитектурные принципы
1. **Компонентный подход** - четкое разделение ответственности между модулями
2. **Функциональное программирование** - использование функций вместо классов  
3. **Простая схема взаимодействия**: User → Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b)

### 🏗️ Архитектурная концепция

**Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b)**

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
│              LLM model (GPT-OSS-20b)                   │
│              Языковая модель                           │
│    • Генерация ответов                                 │
│    • Обработка промптов                                │
└─────────────────────────────────────────────────────────┘
```

### Поток данных

**User → Open WebUI → FastAPI (RAG/CAG Pipelines, адаптер OpenAI KServe, agent) → KServe → LLM model (GPT-OSS-20b)**

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

**→ LLM model (GPT-OSS-20b)**  
Сама LLM-модель: генерирует ответ по полученному промпту и возвращает его назад по цепочке  

**LLM model (GPT-OSS-20b) → KServe → FastAPI (agent, адаптер OpenAI KServe, RAG/CAG Pipelines) → Open WebUI → User**

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

1. **Локальный деплой**: LLM model (GPT-OSS-20b) через KServe
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
- **LLM model (GPT-OSS-20b)** - языковая модель в KServe

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

### Этапы pipeline подготовки данных

#### 1. Data Ingestion (Сбор данных)
- **Источники**:
  - **Для RAG**: Prometheus метрики, OpenSearch логи, Kubernetes API
  - **Для CAG**: Runbooks, документация, troubleshooting guides
- **Инструменты**: Kafka (стриминг), Airflow (оркестрация сбора)
- **Результат**: Raw данные в staging области

#### 2. Data Processing (Обработка и очистка)
- **Для RAG**: 
  - Парсинг JSON логов и метрик
  - Нормализация временных рядов
  - Фильтрация и агрегация данных
- **Для CAG**:
  - Парсинг Markdown документации
  - Извлечение structured data из runbooks
  - Валидация и форматирование
- **Инструменты**: 
  - **Легкая обработка**: Data Prepper / Logstash
  - **Тяжелая обработка**: Apache Spark (для больших исторических данных)
- **Результат**: Структурированные данные готовые для embedding

#### 3. Embedding Generation (Генерация векторов)
- **Модели embedding**:
  - **sentence-transformers** (уже в стеке)
  - **OpenSearch Neural Search** (встроенные ONNX модели)
  - **Кастомные модели** через KServe при необходимости
- **Процесс**:
  - Batch генерация векторов для новых данных
  - Обновление существующих embeddings при изменениях
- **Результат**: Vector embeddings готовые для поиска

#### 4. Vector Storage & Indexing (Хранение и индексация)
- **OpenSearch Vector Storage**:
  - **kNN индексы** для быстрого поиска
  - **Hybrid search** (текстовый + векторный поиск)
  - **Index templates** для автоматического создания индексов
- **Стратегии обновления**:
  - **Rolling updates** для RAG (новые данные каждые 10-30 сек)
  - **Batch updates** для CAG (еженедельно)
- **Результат**: Searchable vector database

#### 5. Cache Management (Управление кэшем CAG)
- **Redis Cache** для часто запрашиваемых ответов CAG
- **Cache invalidation** при обновлении документации
- **TTL policies** для автоматической очистки
- **Результат**: Быстрый доступ к static knowledge

### Технологический стек RAG/CAG

#### Основные компоненты (все уже развернуты)
- **Vector Database**: OpenSearch 2.x с Vector Search + Neural Search
- **Stream Processing**: Kafka - real-time данные для RAG
- **Orchestration**: Airflow - DAGs для RAG/CAG pipeline
- **Data Processing**: 
  - **Легкая**: Data Prepper / Logstash
  - **Тяжелая**: Apache Spark - большие исторические данные
- **Model Serving**: KServe - LLM model hosting
- **Pipeline Tracking**: MLFlow - мониторинг RAG/CAG метрик

#### Дополнительные компоненты
- **Cache Layer**: Redis для CAG кэширования
- **RAG Framework**: LangChain/LlamaIndex для RAG логики
- **Embedding Models**: sentence-transformers + OpenSearch Neural Search
- **Monitoring**: Prometheus + Grafana (переиспользование)

### Pipeline Automation

#### CI/CD через Airflow DAGs
```python
# Пример структуры DAG для RAG/CAG pipeline
RAG_Pipeline_DAG:
  ├── data_ingestion_task (Kafka → staging)
  ├── light_processing_task (Data Prepper/Logstash)
  ├── heavy_processing_task (Spark - исторические данные) 
  ├── embedding_generation_task (sentence-transformers)
  ├── vector_indexing_task (OpenSearch kNN)
  └── quality_validation_task (поиск тестирование → MLFlow)

CAG_Pipeline_DAG:
  ├── documentation_sync_task (Git → staging)
  ├── markdown_processing_task (парсинг runbooks)
  ├── knowledge_indexing_task (OpenSearch + Redis)
  └── cache_warming_task (предзагрузка популярных запросов)
```

### Мониторинг Pipeline

#### Ключевые метрики
- **Data Freshness**: время от события до индексации
- **Embedding Quality**: semantic similarity scores  
- **Search Performance**: latency и accuracy поиска
- **Cache Hit Ratio**: эффективность CAG кэша
- **RAG Relevance**: релевантность найденных документов

#### Алерты и SLA
- **RAG Pipeline**: обновление данных каждые 30 секунд
- **CAG Pipeline**: обновление документации еженедельно
- **Search Latency**: < 200ms для vector search
- **Cache Response**: < 50ms для CAG запросов