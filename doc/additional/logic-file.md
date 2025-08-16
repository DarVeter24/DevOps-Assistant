# DevOps Assistant - Логика взаимодействия файлов

## 🏗️ Архитектура системы

```
Open WebUI → FastAPI Backend → RAG/CAG → KServe → LLM model
```

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

## 📁 Структура файлов и их назначение

### 🚀 **Основные файлы запуска**

#### `main.py` - Точка входа приложения
**Назначение:**
- Запускает FastAPI сервер
- Инициализирует все компоненты системы
- Регистрирует API роуты

**Взаимодействие:**
- Импортирует `config.py` для настроек
- Подключает роуты из `api/chat.py`
- Инициализирует middleware из `api/middleware.py`
- Запускает uvicorn сервер

```python
# Примерная структура main.py
from fastapi import FastAPI
from config import settings
from api.chat import router as chat_router
from api.middleware import setup_middleware

app = FastAPI()
setup_middleware(app)
app.include_router(chat_router, prefix="/v1")

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### `config.py` - Центр конфигурации
**Назначение:**
- Хранит все настройки приложения
- Подключения к внешним сервисам
- Environment variables

**Взаимодействие:**
- Используется всеми модулями для получения настроек
- Читает `.env` файл
- Предоставляет параметры подключения к OpenSearch, Redis, KServe

```python
# Примерная структура config.py
from pydantic import BaseSettings

class Settings(BaseSettings):
    opensearch_url: str
    redis_url: str
    kserve_endpoint: str
    llm_model_name: str
    
    class Config:
        env_file = ".env"

settings = Settings()
```

---

### 🌐 **API слой (взаимодействие с Open WebUI)**

#### `api/chat.py` - Главный API endpoint
**Назначение:**
- Реализует `/v1/chat/completions` endpoint (OpenAI совместимый)
- Принимает запросы от Open WebUI
- Возвращает ответы в OpenAI формате

**Взаимодействие:**
- Получает HTTP запросы от Open WebUI
- Передает данные в `core/chat_handler.py`
- Форматирует ответы для Open WebUI

```python
# Примерная структура api/chat.py
from fastapi import APIRouter
from core.chat_handler import process_chat_request
from schemas.chat_models import ChatCompletionRequest, ChatCompletionResponse

router = APIRouter()

@router.post("/chat/completions")
async def chat_completions(request: ChatCompletionRequest) -> ChatCompletionResponse:
    response = await process_chat_request(request)
    return response
```

#### `api/middleware.py` - Middleware обработка
**Назначение:**
- CORS настройки для Open WebUI
- Аутентификация (если нужна)
- Логирование запросов

**Взаимодействие:**
- Обрабатывает все входящие HTTP запросы
- Настраивает CORS для фронтенда
- Логирует активность

---

### 📋 **Схемы данных (валидация и типизация)**

#### `schemas/chat_models.py` - Pydantic модели запросов/ответов
**Назначение:**
- Определяет структуру OpenAI-совместимых запросов и ответов
- Обеспечивает валидацию входящих данных
- Типизация для автодополнения и проверки типов

**Взаимодействие:**
- Используется в `api/chat.py` для валидации запросов
- Импортируется в `core/chat_handler.py` для типизации
- Обеспечивает совместимость с OpenAI API форматом

```python
# Примерная структура schemas/chat_models.py
from pydantic import BaseModel
from typing import List, Optional

class ChatMessage(BaseModel):
    role: str  # "system" | "user" | "assistant"
    content: str

class ChatCompletionRequest(BaseModel):
    model: str
    messages: List[ChatMessage]
    temperature: Optional[float] = 0.7
    max_tokens: Optional[int] = 1000
    stream: Optional[bool] = False

class ChatCompletionChoice(BaseModel):
    index: int
    message: ChatMessage
    finish_reason: str

class ChatCompletionResponse(BaseModel):
    id: str
    object: str = "chat.completion"
    model: str
    choices: List[ChatCompletionChoice]
    usage: dict
```

---

### 🧠 **Ядро системы**

#### `core/chat_handler.py` - Центральная логика обработки
**Назначение:**
- **ГЛАВНЫЙ ОРКЕСТРАТОР** всей логики
- Координирует работу RAG, CAG и LLM компонентов
- Реализует основной flow обработки запросов

**Взаимодействие:**
- Получает запросы от `api/chat.py`
- Вызывает `rag/search.py` для поиска контекста
- Вызывает `cag/cache.py` для поиска в кэше
- Использует `llm/openai_adapter.py` для подготовки промпта
- Отправляет в `llm/kserve_client.py` для получения ответа LLM

```python
# Примерная логика core/chat_handler.py
from rag.search import search_context
from cag.cache import get_cached_knowledge
from llm.openai_adapter import prepare_prompt
from llm.kserve_client import call_llm

async def process_chat_request(request):
    # 1. Поиск контекста через RAG
    rag_context = await search_context(request.messages)
    
    # 2. Поиск в кэше через CAG
    cag_context = await get_cached_knowledge(request.messages)
    
    # 3. Подготовка промпта
    prompt = prepare_prompt(request.messages, rag_context, cag_context)
    
    # 4. Вызов LLM
    response = await call_llm(prompt)
    
    return response
```

---

### 🔍 **RAG компоненты (поиск в динамических данных)**

#### `rag/search.py` - Поиск в метриках и логах
**Назначение:**
- Выполняет векторный поиск в OpenSearch
- Ищет релевантные метрики Prometheus
- Находит связанные логи и события

**Взаимодействие:**
- Вызывается из `core/chat_handler.py`
- Использует настройки из `config.py` для подключения к OpenSearch
- Возвращает найденный контекст обратно в chat_handler

```python
# Примерная логика rag/search.py
from opensearchpy import OpenSearch
from config import settings

async def search_context(messages):
    client = OpenSearch(settings.opensearch_url)
    
    # Векторный поиск по последнему сообщению
    query = messages[-1]["content"]
    
    search_results = client.search(
        index="metrics-logs",
        body={
            "query": {
                "match": {"content": query}
            }
        }
    )
    
    return extract_relevant_context(search_results)
```

---

### 💾 **CAG компоненты (поиск в статических знаниях)**

#### `cag/cache.py` - Поиск в кэше знаний
**Назначение:**
- Ищет готовые ответы в Redis кэше
- Находит runbooks и документацию
- Возвращает инструкции и процедуры

**Взаимодействие:**
- Вызывается из `core/chat_handler.py`
- Использует Redis для быстрого поиска
- Конфигурация из `config.py`

```python
# Примерная логика cag/cache.py
import redis
from config import settings

async def get_cached_knowledge(messages):
    r = redis.from_url(settings.redis_url)
    
    # Поиск по ключевым словам
    query = extract_keywords(messages[-1]["content"])
    
    # Поиск в кэше runbooks
    cached_result = r.get(f"runbook:{query}")
    
    return cached_result if cached_result else None
```

---

### 🤖 **LLM интеграция**

#### `llm/openai_adapter.py` - Адаптер для OpenAI формата
**Назначение:**
- Подготавливает промпты в нужном формате
- Объединяет контекст от RAG и CAG с запросом пользователя
- Форматирует данные для отправки в LLM

**Взаимодействие:**
- Вызывается из `core/chat_handler.py`
- Получает контекст от RAG и CAG компонентов
- Передает готовый промпт в `kserve_client.py`

```python
# Примерная логика llm/openai_adapter.py
def prepare_prompt(messages, rag_context, cag_context):
    system_prompt = "Вы DevOps ассистент..."
    
    # Добавляем контекст от RAG
    if rag_context:
        system_prompt += f"\n\nТекущие метрики: {rag_context}"
    
    # Добавляем контекст от CAG
    if cag_context:
        system_prompt += f"\n\nИнструкции: {cag_context}"
    
    return [
        {"role": "system", "content": system_prompt},
        *messages
    ]
```

#### `llm/kserve_client.py` - Клиент для KServe
**Назначение:**
- Отправляет HTTP запросы в KServe
- Обрабатывает ответы от LLM модели
- Управляет соединением с моделью

**Взаимодействие:**
- Получает промпты от `openai_adapter.py`
- Отправляет запросы в KServe endpoint
- Возвращает ответы обратно в `chat_handler.py`

```python
# Примерная логика llm/kserve_client.py
import httpx
from config import settings

async def call_llm(prompt):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            settings.kserve_endpoint,
            json={
                "messages": prompt,
                "model": settings.llm_model_name
            }
        )
        
    return response.json()
```

---

### 📊 **Управление данными**

#### `data/raw/` - Исходные данные
**Назначение:**
- Хранит исходные файлы документации (md/txt/pdf)
- Содержит runbooks, инструкции, troubleshooting guides
- Источник данных для CAG pipeline

**Взаимодействие:**
- Читается скриптом `data/loader.py`
- Обрабатывается в `dags/index_docs.py`
- Преобразуется в текст для индексации

#### `data/loader.py` - Загрузчик данных
**Назначение:**
- Скрипт для первичной загрузки и индексации данных в OpenSearch
- Конвертация различных форматов (pdf→txt, md→txt)
- Создание embeddings для документов

**Взаимодействие:**
- Читает файлы из `data/raw/`
- Использует `dags/data_processors/` для обработки
- Загружает результат в OpenSearch и Redis

```python
# Примерная структура data/loader.py
from pathlib import Path
from dags.data_processors.embedding_generator import generate_embeddings
from dags.data_processors.opensearch_indexer import index_documents

def load_documents():
    raw_docs = Path("data/raw/").glob("**/*.md")
    
    for doc in raw_docs:
        # Чтение и обработка
        content = doc.read_text()
        embeddings = generate_embeddings(content)
        
        # Индексация
        index_documents(content, embeddings)
```

---

### 🔄 **Pipeline автоматизации**

#### `dags/index_docs.py` - ETL pipeline для документов
**Назначение:**
- Airflow DAG для полного ETL цикла документов
- Extract → Transform → Embed → Load → Invalidate Cache
- Запускается при изменении документации

**Взаимодействие:**
- Читает данные из `data/raw/`
- Использует процессоры из `dags/data_processors/`
- Обновляет OpenSearch и инвалидирует Redis кэш

#### `dags/rag_pipeline.py` - Обновление векторной БД
**Назначение:**
- Периодическое обновление RAG данных (метрики, логи)
- Генерация новых embeddings для свежих данных
- Поддержание актуальности векторного поиска

**Взаимодействие:**
- Получает данные от Kafka/Prometheus
- Обрабатывает через `data_processors/`
- Обновляет OpenSearch индексы

#### `dags/cag_pipeline.py` - Управление кэшем
**Назначение:**
- Обслуживание и прогрев Redis кэша
- Инвалидация устаревших данных
- Предзагрузка популярных запросов

**Взаимодействие:**
- Анализирует паттерны использования
- Использует `redis_updater.py`
- Оптимизирует производительность CAG

### 🛠️ **Процессоры данных**

#### `dags/data_processors/embedding_generator.py`
**Назначение:**
- Генерация embeddings в batch/mini-batch режиме
- Поддержка различных моделей (sentence-transformers)
- Оптимизация для больших объемов данных

#### `dags/data_processors/opensearch_indexer.py`
**Назначение:**
- Upsert операции с OpenSearch
- Создание и обновление индексов
- Управление mapping'ами и aliases

#### `dags/data_processors/redis_updater.py`
**Назначение:**
- Массовые операции с Redis
- Обновление и инвалидация ключей
- Управление TTL policies

---

## 🔄 **Полный Flow обработки запроса**

```
1. Open WebUI → api/chat.py
   │ (HTTP POST /v1/chat/completions)
   │
2. api/chat.py → schemas/chat_models.py (валидация Pydantic)
   │           → core/chat_handler.py
   │
3. core/chat_handler.py → rag/search.py (поиск в OpenSearch)
   │                    → cag/cache.py (поиск в Redis)
   │
4. core/chat_handler.py → llm/openai_adapter.py (подготовка промпта)
   │
5. llm/openai_adapter.py → llm/kserve_client.py (вызов LLM)
   │
6. Ответ возвращается обратно по цепочке:
   llm/kserve_client.py → llm/openai_adapter.py → 
   core/chat_handler.py → schemas/chat_models.py (форматирование ответа) →
   api/chat.py → Open WebUI
```

### 🔄 **Pipeline Flow обработки данных**

```
Документы в data/raw/ → data/loader.py → 
dags/index_docs.py → data_processors/ → 
OpenSearch + Redis кэш → доступно для RAG/CAG
```

## 🛠️ **Вспомогательные файлы**

### `.env` - Переменные окружения
- Хранит секреты и настройки
- Не попадает в Git репозиторий
- Используется config.py

### `Dockerfile` - Контейнеризация
- Описывает как собрать Docker образ
- Устанавливает зависимости
- Настраивает runtime окружение

### `__init__.py` файлы
- Делают папки Python пакетами
- Позволяют импортировать модули
- Могут содержать инициализацию пакета

---

## 🎯 **Ключевые принципы архитектуры**

1. **Разделение ответственности**: Каждый файл имеет четкую роль
2. **Слабая связанность**: Модули взаимодействуют через четкие интерфейсы
3. **Единая точка входа**: `main.py` координирует все компоненты
4. **Центральная логика**: `core/chat_handler.py` - мозг системы
5. **Модульность**: RAG, CAG, LLM - независимые компоненты

## 🎯 **Обновленные принципы архитектуры**

1. **Разделение ответственности**: Каждый файл имеет четкую роль
2. **Типизация и валидация**: Pydantic модели обеспечивают безопасность типов
3. **Единая точка входа**: `main.py` координирует все runtime компоненты  
4. **Центральная логика**: `core/chat_handler.py` - мозг системы
5. **Модульность**: RAG, CAG, LLM, Pipeline - независимые компоненты
6. **Автоматизация**: Airflow DAGs управляют обновлением данных

Эта архитектура обеспечивает **простоту понимания**, **типобезопасность**, **легкость тестирования** и **возможность масштабирования** каждого компонента независимо.