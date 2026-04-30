---
tags: [ai, semantic-kernel, vector-search, rag, embeddings, llm]
level: Senior
---

# Semantic Kernel и Vector Search в .NET

## Что это, зачем и когда

### Что такое Semantic Kernel?
**AI SDK от Microsoft для .NET.** Абстракция над LLM-провайдерами (OpenAI, Azure OpenAI, Gemini, Ollama), embeddings, vector stores и AI-агентами. Альтернатива LangChain из Python-мира, но native для .NET.

**Аналогия:** Entity Framework Core для данных — один API к разным БД. Semantic Kernel — один API к разным LLM и vector stores.

### Что такое Vector Search?
Поиск по **смысловой близости**, а не по совпадению текста. Текст/изображения/аудио кодируются в векторные эмбеддинги (обычно 768-dim или 1536-dim), сравнение через cosine similarity в многомерном пространстве.

### Зачем?

| Full-text search (классика) | Vector search |
|----------------------------|---------------|
| Ищет совпадение слов (LIKE, tsvector, Elastic) | Ищет похожие по смыслу |
| "ef core" не найдёт "entity framework" | Найдёт, потому что векторы близки |
| Синонимы требуют ручной настройки | Работают из коробки через embedding-модель |
| Быстрый, дешёвый | Дороже — генерация embedding платная или медленная |

### Когда использовать?

| Задача | Подход |
|--------|--------|
| Семантический поиск по документам | Vector search |
| RAG — подача контекста в LLM | Vector search + LLM |
| Рекомендации по смыслу / поведению | Vector search |
| Умные ассистенты внутри приложения | Semantic Kernel + Agents |
| Точный поиск по SKU, email, имени | Full-text / LIKE / индекс |
| Гибрид (важны оба) | Hybrid search: BM25 + vector (Qdrant, Weaviate из коробки) |

---

## Vector Stores — выбор под задачу

| Store | Когда | Минусы |
|-------|-------|--------|
| **PostgreSQL + pgvector** | Уже есть Postgres, не хочется новой инфры | Скорость хуже специализированных, ручной hybrid search |
| **Qdrant** | Скорость + rich filtering + гибрид из коробки | Новая инфра, Rust-сервис |
| **Weaviate / Milvus** | Managed / enterprise | Платно, enterprise-ориентированы |
| **Pinecone** | Hosted, zero ops | Vendor lock-in, стоимость на объёме |
| **Redis 8 + RediSearch** | Уже есть Redis, in-memory кэш векторов | Память дорогая на больших корпусах |
| **AWS S3 Vectors** | Дешёвое хранилище для больших корпусов | Медленнее специализированных |
| **SQLite + sqlite-vec** | Embedded, offline, локальные сценарии | Не для production-нагрузки |

---

## Semantic Kernel — Quickstart

```bash
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.SemanticKernel.Connectors.Postgres
dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI
```

### Генерация embedding

```csharp
using Microsoft.SemanticKernel;
using Microsoft.Extensions.AI;

var kernel = Kernel.CreateBuilder()
    .AddOpenAIEmbeddingGenerator(
        modelId: "text-embedding-3-small",
        apiKey: configuration["OpenAI:ApiKey"]!)
    .Build();

var embedder = kernel.GetRequiredService<IEmbeddingGenerator<string, Embedding<float>>>();
var vector = await embedder.GenerateAsync("Как использовать EF Core проекции");
// vector.Vector — ReadOnlyMemory<float> длиной 1536
```

### Модель с векторным полем

```csharp
using Microsoft.Extensions.VectorData;

public sealed class Article
{
    [VectorStoreKey]
    public int Id { get; set; }

    [VectorStoreData(IsFullTextIndexed = true)]
    public string Title { get; set; } = string.Empty;

    [VectorStoreData]
    public string Content { get; set; } = string.Empty;

    [VectorStoreVector(Dimensions: 1536, DistanceFunction = DistanceFunction.CosineSimilarity)]
    public ReadOnlyMemory<float> Embedding { get; set; }
}
```

### Хранение и поиск в Postgres (pgvector)

```csharp
var vectorStore = new PostgresVectorStore(dataSource);
var collection = vectorStore.GetCollection<int, Article>("articles");
await collection.CreateCollectionIfNotExistsAsync();

// Upsert
var article = new Article
{
    Id = 1,
    Title = "EF Core Projections",
    Content = "Используй .Select() чтобы не тащить лишние колонки...",
    Embedding = await embedder.GenerateAsync(
        "EF Core Projections\n\n" +
        "Используй .Select() чтобы не тащить лишние колонки...")
};
await collection.UpsertAsync(article);

// Search
var queryEmbedding = await embedder.GenerateAsync("EF Core performance tips");
var results = collection.SearchAsync(queryEmbedding, top: 5);

await foreach (var r in results)
    Console.WriteLine($"{r.Score:F3} — {r.Record.Title}");
```

### RAG-пайплайн целиком

```csharp
public sealed class RagService(
    IEmbeddingGenerator<string, Embedding<float>> embedder,
    IVectorStoreCollection<int, Article> store,
    IChatClient chat)
{
    public async Task<string> AskAsync(string question, CancellationToken ct)
    {
        // 1. Эмбедим вопрос
        var qEmbedding = await embedder.GenerateAsync(question, cancellationToken: ct);

        // 2. Находим top-5 релевантных статей
        var topArticles = new List<Article>();
        await foreach (var r in store.SearchAsync(qEmbedding, top: 5, cancellationToken: ct))
            topArticles.Add(r.Record);

        // 3. Собираем контекст
        var context = string.Join("\n\n---\n\n",
            topArticles.Select(a => $"# {a.Title}\n{a.Content}"));

        // 4. Спрашиваем LLM с контекстом
        var response = await chat.GetResponseAsync(
            $"""
            Отвечай только на основе предоставленного контекста.
            Если ответа нет в контексте — скажи "не знаю".

            # Контекст
            {context}

            # Вопрос
            {question}
            """,
            cancellationToken: ct);

        return response.Text;
    }
}
```

---

## Chunking — как разбивать документы

Документы редко помещаются в один embedding (и лимит токенов у модели, и смешивание тем в одном векторе). Стратегии:

| Стратегия | Когда | Trade-off |
|-----------|-------|-----------|
| **Fixed-size** (512 tokens + 20% overlap) | Унифицированные тексты | Может резать посреди мысли |
| **Semantic boundaries** (абзац, заголовок) | Структурированные документы (MD, HTML) | Чанки разного размера |
| **Sliding window** | Нужна высокая recall, не боимся дублей | Больше данных в индексе |
| **Parent-child** | Короткий ребёнок для поиска, длинный родитель для LLM | Двойное хранилище |

---

## Локальные модели — Ollama

Для embedding без OpenAI API:

```bash
ollama pull nomic-embed-text
```

```csharp
// Подключение через OpenAI-compatible API
builder.AddOpenAIEmbeddingGenerator(
    modelId: "nomic-embed-text",
    endpoint: new Uri("http://localhost:11434/v1"),
    apiKey: "ollama"); // формальный, Ollama его не проверяет
```

Нулевая стоимость, данные не покидают машину. Качество ниже топовых коммерческих моделей, но достаточно для большинства задач.

---

## Подводные камни

- **Размерность вектора менять потом больно.** Смена модели → reindex всего корпуса. Выбирай под задачу сразу: 384 для быстрого/дешёвого, 768/1024 для баланса, 1536+ для точности.
- **Метрика расстояния должна совпадать при build и query.** CosineSimilarity при индексе → CosineSimilarity при поиске. Смешение = мусорные результаты.
- **Hybrid search обычно лучше чистого vector.** BM25 + vector + reranker (`BAAI/bge-reranker`). Qdrant / Weaviate умеют из коробки, в pgvector — вручную.
- **Стоимость embedding при масштабе.** 100k документов × 1k токенов × $0.02/1M = $2. При 10M документов → $200. Bulk-генерация через batch API дешевле.
- **RLS и multi-tenant.** Vector store должен уметь фильтровать по `TenantId` перед similarity search, а не после — иначе сливается в одну кучу.

## См. также

- [Project Setup](project-setup.md)
- [Observability](observability.md) — трассировка LLM-вызовов через OTel
- [Caching](../AspNetCore/caching.md) — кэш embedding-запросов
- [Reading List](../Meta/reading-list.md)
