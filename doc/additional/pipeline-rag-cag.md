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
