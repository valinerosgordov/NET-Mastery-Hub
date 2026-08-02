---
tags: [ai, semantic-kernel, agent-framework, vector-search, rag, embeddings, llm]
level: Senior
date: 2026-08-02
---

# Semantic Kernel и Vector Search в .NET

> Semantic Kernel — зрелый AI SDK Microsoft для .NET (LLM, embeddings, vector search, agents), с октября 2025 в maintenance mode: новые агентные проекты Microsoft ведёт в Agent Framework (GA апрель 2026). Здесь — SK для существующих кодовых баз, выбор vector store (pgvector, Qdrant, Pinecone, Redis) под RAG и раздел о миграции на Agent Framework.

## Что это, зачем и когда

> [!warning] Статус 2026: SK в maintenance mode
> С public preview **Microsoft Agent Framework** (октябрь 2025) Semantic Kernel и AutoGen переведены в **maintenance mode** — только bugfix и security-патчи, новых фич не будет. Agent Framework достиг **GA 3 апреля 2026** (.NET и Python) — это конвергенция SK + AutoGen, и для **новых** агентных проектов Microsoft рекомендует именно его. SK-знания не обесценились: Agent Framework вырос из SK (те же `Microsoft.Extensions.AI`-типы, концепции переносятся почти 1:1), а существующие SK-кодовые базы получают critical bugfixes минимум год после GA. Подробности и миграция — [[#Microsoft Agent Framework — куда переехал SK|раздел ниже]].

### Что такое Semantic Kernel?
**AI SDK от Microsoft для .NET.** Абстракция над LLM-провайдерами (OpenAI, Azure OpenAI, Gemini, Ollama), embeddings, vector stores и AI-агентами. Альтернатива LangChain из Python-мира, но native для .NET. С октября 2025 — в maintenance mode (см. callout выше); vector search-часть (`Microsoft.Extensions.VectorData`, коннекторы) живёт своей жизнью и используется и из Agent Framework.

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
await collection.EnsureCollectionExistsAsync();

// Upsert
var article = new Article
{
    Id = 1,
    Title = "EF Core Projections",
    Content = "Используй .Select() чтобы не тащить лишние колонки...",
    Embedding = await embedder.GenerateVectorAsync(
        "EF Core Projections\n\n" +
        "Используй .Select() чтобы не тащить лишние колонки...")
};
await collection.UpsertAsync(article);

// Search
var queryEmbedding = await embedder.GenerateVectorAsync("EF Core performance tips");
var results = collection.SearchAsync(queryEmbedding, top: 5);

await foreach (var r in results)
    Console.WriteLine($"{r.Score:F3} — {r.Record.Title}");
```

### RAG-пайплайн целиком

```csharp
public sealed class RagService(
    IEmbeddingGenerator<string, Embedding<float>> embedder,
    VectorStoreCollection<int, Article> store,
    IChatClient chat)
{
    public async Task<string> AskAsync(string question, CancellationToken ct)
    {
        // 1. Эмбедим вопрос
        var qEmbedding = await embedder.GenerateVectorAsync(question, cancellationToken: ct);

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

---

## Plugins и Functions — extending Kernel

### Что такое Plugin

**Plugin** — collection of functions, доступных kernel'у. Functions могут быть:
- **Native functions** — обычные C# методы с атрибутами
- **Prompt functions** — текстовые prompts (template + LLM call)

Kernel может **выбирать сам** какие functions вызвать в response на user query — это **function calling** / **tools**.

### Native function plugin

```csharp
using Microsoft.SemanticKernel;
using System.ComponentModel;

public sealed class WeatherPlugin
{
    private readonly IWeatherClient _client;

    public WeatherPlugin(IWeatherClient client) => _client = client;

    [KernelFunction]
    [Description("Gets the current weather for a given city")]
    public async Task<string> GetCurrentWeatherAsync(
        [Description("City name, e.g. Berlin")] string city,
        [Description("Units: celsius or fahrenheit")] string units = "celsius",
        CancellationToken ct = default)
    {
        var weather = await _client.GetWeatherAsync(city, units, ct);
        return $"{weather.Temperature}°{units[0]}, {weather.Description}";
    }

    [KernelFunction]
    [Description("Gets weather forecast for next N days")]
    public async Task<string> GetForecastAsync(
        [Description("City name")] string city,
        [Description("Number of days, max 7")] int days = 3,
        CancellationToken ct = default)
    {
        var forecast = await _client.GetForecastAsync(city, days, ct);
        return string.Join("\n", forecast.Days.Select(d => $"{d.Date}: {d.Summary}"));
    }
}
```

### Регистрация plugin

```csharp
var builder = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion("gpt-5-mini", apiKey);   // подставь актуальную модель провайдера

builder.Services.AddSingleton<IWeatherClient, OpenWeatherMapClient>();
builder.Plugins.AddFromType<WeatherPlugin>();   // DI-aware

var kernel = builder.Build();
```

### Prompt function — text template

```csharp
// Inline definition
var summarize = kernel.CreateFunctionFromPrompt(
    promptTemplate: """
        Summarize the following text in {{$max_words}} words or less:
        
        {{$input}}
        """,
    functionName: "Summarize",
    description: "Summarizes input text",
    executionSettings: new OpenAIPromptExecutionSettings
    {
        MaxTokens = 200,
        Temperature = 0.3
    });

// Usage
var result = await kernel.InvokeAsync(summarize, new KernelArguments
{
    ["input"] = longText,
    ["max_words"] = 50
});

Console.WriteLine(result.GetValue<string>());
```

### Plugin from directory

```
plugins/
└── WriterPlugin/
    ├── Summarize/
    │   ├── skprompt.txt        ← prompt template
    │   └── config.json         ← settings
    ├── Translate/
    │   ├── skprompt.txt
    │   └── config.json
    └── Rewrite/
        ├── skprompt.txt
        └── config.json
```

```csharp
kernel.Plugins.AddFromPromptDirectory("plugins/WriterPlugin");

var summarize = kernel.Plugins["WriterPlugin"]["Summarize"];
var result = await kernel.InvokeAsync(summarize, new() { ["input"] = text });
```

`config.json` пример:

```json
{
  "schema": 1,
  "description": "Summarizes input text",
  "execution_settings": {
    "default": {
      "max_tokens": 200,
      "temperature": 0.3
    }
  },
  "input_variables": [
    { "name": "input", "description": "Text to summarize", "required": true }
  ]
}
```

> [!question]- **Интервью: Plugin vs Function в Semantic Kernel?**
> **Plugin** — collection (group) of functions, обычно related по domain (WeatherPlugin = GetCurrent + GetForecast + GetAlerts). **Function** — индивидуальная capability. Два типа: **Native** (C# method с `[KernelFunction]`) — для structured operations (DB query, API call). **Prompt** (text template + LLM) — для text generation (summarization, translation). **Use case**: kernel may **automatically choose** какую function вызвать на user query через function calling. **Аналог LangChain**: tools + agents.

---

## Function Calling — LLM выбирает function автоматически

### Концепция

Все современные LLM (GPT, Claude, Gemini, Llama) поддерживают **function calling**: ты регистрируешь functions, LLM решает какую вызвать на user query, парсирует arguments, ты executes function, returns result обратно LLM.

```
User: "What's the weather in Berlin tomorrow?"
   ↓
LLM analyzes: needs weather forecast
   ↓
LLM returns: function_call("GetForecastAsync", { city: "Berlin", days: 1 })
   ↓
You execute function → "+15°C, partly cloudy"
   ↓
LLM generates final response: "Tomorrow in Berlin will be +15°C with partly cloudy skies"
```

### Auto invocation в SK

Современный SK-API — провайдер-независимый `FunctionChoiceBehavior` (свойство базового `PromptExecutionSettings`); старый `ToolCallBehavior` — OpenAI-специфичный legacy, но всё ещё компилируется.

```csharp
var settings = new OpenAIPromptExecutionSettings
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
};

var chat = kernel.GetRequiredService<IChatCompletionService>();
var history = new ChatHistory("You are a helpful assistant");

history.AddUserMessage("What's the weather in Berlin and Paris?");

var response = await chat.GetChatMessageContentAsync(history, settings, kernel);

Console.WriteLine(response.Content);
// Kernel автоматически:
// 1. Sends user message + available functions to LLM
// 2. LLM returns function calls (GetCurrentWeatherAsync × 2)
// 3. Kernel invokes functions
// 4. Sends results back to LLM
// 5. LLM produces final answer using actual data
```

### Manual control

```csharp
var settings = new OpenAIPromptExecutionSettings
{
    // Современный эквивалент: FunctionChoiceBehavior.Auto(autoInvoke: false)
    ToolCallBehavior = ToolCallBehavior.EnableKernelFunctions   // не auto-invoke
};

var response = await chat.GetChatMessageContentAsync(history, settings, kernel);

// Inspect what LLM wants to call
if (response is OpenAIChatMessageContent openAi)
{
    var toolCalls = openAi.GetToolCalls();
    foreach (var call in toolCalls)
    {
        Console.WriteLine($"LLM wants: {call.FunctionName} with {call.Arguments}");
        
        // Decide whether to allow + invoke
        if (await IsAuthorized(call))
        {
            var result = await call.InvokeAsync(kernel);
            history.Add(new ChatMessageContent(AuthorRole.Tool, result.ToString()));
        }
    }
    
    // Continue conversation
    var final = await chat.GetChatMessageContentAsync(history, settings, kernel);
}
```

### Когда manual control

- **Audit / approval flow** — admin approves before tool calls real action
- **Cost control** — некоторые tools expensive (paid API), gate them
- **Security** — sensitive operations require additional auth
- **Debugging** — log all tool calls for review

### Function description quality

LLM выбирает function по `[Description]` атрибутам. **Качество описаний critical:**

```csharp
// ❌ Плохо — vague
[KernelFunction]
[Description("Gets data")]
public async Task<string> Get(string x) { }

// ✅ Хорошо — specific, includes когда use
[KernelFunction]
[Description("""
    Retrieves the current weather conditions for a specified city.
    Use this when user asks about current weather, temperature, or conditions.
    Do not use for historical weather (use GetHistoricalWeather) or forecast (use GetForecast).
    """)]
public async Task<string> GetCurrentWeatherAsync(
    [Description("Full city name with country if ambiguous, e.g. 'Paris, France' or 'Berlin'")] 
    string city) { }
```

### Common pitfalls function calling

- **Too many functions** (50+) — confuses LLM, может выбрать wrong. Group в logical plugins, expose subset для конкретной session.
- **Side effects без preview** — LLM auto-invoke `DeleteAccount()` → bad. Use manual mode for destructive operations.
- **Returning huge data** — function returns 100K rows → LLM context overflow. Return summary or top-N.
- **No error handling** — function throws → entire chain breaks. Wrap в try-catch, return error string LLM can interpret.

> [!question]- **Интервью: function calling в SK как работает?**
> Современные LLM поддерживают **tool/function calling protocol**: ты declares available functions с descriptions, LLM в response может request function call с arguments вместо text answer. Kernel: 1) Sends user message + function metadata to LLM. 2) LLM returns either text **or** tool_call(name, args). 3) Kernel invokes function. 4) Result sent back to LLM. 5) LLM generates final response using result. **Two modes**: `FunctionChoiceBehavior.Auto()` (kernel handles loop) или `Auto(autoInvoke: false)` (you control). **Production critical**: descriptions quality, function granularity, error handling, manual mode для destructive operations.

---

## Agents Framework — autonomous workflows

### Что такое Agent

**Agent** — LLM-powered actor с:
- **Persistent role / instructions** (system prompt)
- **Tools** (functions он может call)
- **Memory** (conversation history)
- **Goal-oriented behavior** (multi-step reasoning)

Несколько agents могут collaborate — **multi-agent systems**.

### Single agent (SK Agents — GA, но заморожены)

SK Agents дошли до GA весной 2025, но с переводом SK в maintenance mode дальше не развиваются — именно этот слой стал основой [[#Microsoft Agent Framework — куда переехал SK|Agent Framework]]. Код ниже актуален для существующих SK-кодовых баз.

```bash
dotnet add package Microsoft.SemanticKernel.Agents.Core
dotnet add package Microsoft.SemanticKernel.Agents.OpenAI
```

```csharp
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Chat;

var researcher = new ChatCompletionAgent
{
    Name = "Researcher",
    Instructions = """
        You are a research assistant. When asked a question:
        1. Use available tools to search for information
        2. Verify facts через multiple sources
        3. Provide structured answer with sources cited
        """,
    Kernel = kernel,
    Arguments = new KernelArguments(new OpenAIPromptExecutionSettings
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    })
};

// Conversation
var thread = new ChatHistoryAgentThread();

await foreach (var response in researcher.InvokeAsync(
    new ChatMessageContent(AuthorRole.User, "What are recent advances in fusion energy?"),
    thread))
{
    Console.WriteLine($"{response.Message.Role}: {response.Message.Content}");
}
```

### Multi-agent collaboration

```csharp
var researcher = new ChatCompletionAgent { Name = "Researcher", Instructions = "...", Kernel = kernel };
var writer = new ChatCompletionAgent { Name = "Writer", Instructions = "...", Kernel = kernel };
var reviewer = new ChatCompletionAgent { Name = "Reviewer", Instructions = "...", Kernel = kernel };

var groupChat = new AgentGroupChat(researcher, writer, reviewer)
{
    ExecutionSettings = new()
    {
        TerminationStrategy = new ApprovalTerminationStrategy()
        {
            Agents = [reviewer],
            MaximumIterations = 10
        },
        SelectionStrategy = new SequentialSelectionStrategy()
    }
};

groupChat.AddChatMessage(new ChatMessageContent(AuthorRole.User, 
    "Write a blog post about AI advances in 2026"));

await foreach (var msg in groupChat.InvokeAsync())
{
    Console.WriteLine($"[{msg.AuthorName}]: {msg.Content}");
}
```

Pipeline:
1. **Researcher** searches и compiles facts
2. **Writer** drafts blog post
3. **Reviewer** проверяет → если "approved" → terminates, иначе sends back

### Agent vs Function calling

| | Function calling | Agent |
|--|------------------|-------|
| Statefulness | Stateless per call | Persistent state, memory |
| Multi-step | Single round-trip | Multi-step reasoning |
| Goal-oriented | No (just answers) | Yes (works toward goal) |
| Collaboration | Single LLM | Multi-agent possible |
| Use case | "Get weather in Berlin" | "Research and write blog post" |

### Когда agents

```
✅ Use agents:
- Multi-step research tasks
- Workflows requiring iteration ("review → revise")
- Specialized roles (researcher + writer)
- Long-running tasks с checkpoint
- Autonomous problem-solving

❌ Не agents:
- Simple Q&A → use function calling
- Single-step transactions → direct call
- Deterministic processes → use code
- Cost-sensitive (agents = more LLM calls)
```

> [!question]- **Интервью: SK Agents framework зачем?**
> **Agent** = LLM с persistent instructions, tools, memory, goal-orientation. Делает multi-step reasoning vs single function call. **Use cases**: 1) Complex tasks (research + summarize + verify). 2) Multi-agent collaboration (researcher → writer → reviewer pipeline). 3) Autonomous workflows. **Termination strategies**: max iterations, agent approval, custom condition. **Selection strategies**: sequential, round-robin, LLM-decided. **Cost**: каждый turn = LLM call → expensive. **Status 2026**: SK Agents — GA, но SK в maintenance mode; новые агентные проекты Microsoft рекомендует строить на Agent Framework (конвергенция SK + AutoGen, GA апрель 2026), где мульти-агентность — это Workflows. **Alternatives вне Microsoft**: LangGraph, CrewAI.

---

## Microsoft Agent Framework — куда переехал SK

### Что это

**Microsoft Agent Framework** — production SDK Microsoft для агентов и мульти-агентных workflows, **GA 3 апреля 2026** (.NET и Python, open source, MIT). Это конвергенция Semantic Kernel и AutoGen: от SK — enterprise-фундамент (DI, OpenTelemetry, типы `Microsoft.Extensions.AI`), от AutoGen — мульти-агентные оркестрации. Документация: learn.microsoft.com/agent-framework.

```bash
dotnet add package Microsoft.Agents.AI.OpenAI   # тянет core-пакет Microsoft.Agents.AI
```

### Ключевые концепции

| Концепция | Что это |
|-----------|---------|
| **`AIAgent`** | Базовая абстракция агента. Один конкретный тип `ChatClientAgent` работает поверх любого `IChatClient` — вместо зоопарка `ChatCompletionAgent` / `OpenAIAssistantAgent` / `AzureAIAgent` |
| **Session** (`AgentSession`) | Состояние диалога (бывший thread); создаёт сам агент через `CreateSessionAsync()`, бывает локальной и service-backed |
| **Workflows** | Graph-based движок оркестрации: агенты и функции соединяются в детерминированные, воспроизводимые процессы (наследие AutoGen; замена `AgentGroupChat`) |
| **Middleware** | Pipeline перехвата поведения агента: логирование, approval-flow, трансформация tool-вызовов (аналог SK filters) |
| **Tools** | Обычные C# методы через `AIFunctionFactory.Create()` — без `[KernelFunction]`, без Kernel; `[Description]` опционален |

Плюс встроенная поддержка MCP (Model Context Protocol) и A2A (agent-to-agent) из коробки.

### Маппинг SK → Agent Framework

| Semantic Kernel | Agent Framework |
|-----------------|-----------------|
| `Kernel` + plugins | Не нужен — агент строится прямо на `IChatClient` |
| `[KernelFunction]` + `KernelPluginFactory` + Kernel | `AIFunctionFactory.Create(method)` в параметре `tools` |
| `ChatCompletionAgent { Kernel = ... }` | `chatClient.AsAIAgent(instructions: ...)` |
| `new ChatHistoryAgentThread()` — тип выбираешь сам | `await agent.CreateSessionAsync()` — агент сам знает |
| `agent.InvokeAsync(...)` → `IAsyncEnumerable<AgentResponseItem<...>>` | `agent.RunAsync(...)` → единый `AgentResponse` (`.Text`, `.Messages`) |
| `agent.InvokeStreamingAsync(...)` | `agent.RunStreamingAsync(...)` → `AgentResponseUpdate` |
| `OpenAIPromptExecutionSettings` + `KernelArguments` | `ChatClientAgentRunOptions` (`MaxOutputTokens` и т.д.) |
| `AgentGroupChat` + selection/termination strategies | Workflows (graph-based) |
| Filters (function/prompt invocation) | Middleware |

### Минимальный агент на C#

```csharp
using System.ComponentModel;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;

var apiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")!;

// Любой IChatClient (Microsoft.Extensions.AI): OpenAI, Azure OpenAI, Ollama...
IChatClient chatClient = new OpenAIClient(apiKey)
    .GetChatClient("gpt-5-mini")   // подставь актуальную модель провайдера
    .AsIChatClient();

// Tool — обычный метод, без [KernelFunction] и без Kernel
[Description("Gets the current weather for a given city")]
static string GetWeather([Description("City name")] string city)
    => $"{city}: +21°C, sunny";

AIAgent agent = chatClient.AsAIAgent(
    instructions: "You are a helpful weather assistant.",
    tools: [AIFunctionFactory.Create(GetWeather)]);

// Session хранит контекст диалога (аналог AgentThread из SK)
AgentSession session = await agent.CreateSessionAsync();

AgentResponse response = await agent.RunAsync("What's the weather in Berlin?", session);
Console.WriteLine(response.Text);

// Streaming — тот же паттерн
await foreach (AgentResponseUpdate update in agent.RunStreamingAsync("And in Paris?", session))
    Console.Write(update);   // update ToString()-friendly
```

### Когда мигрировать, когда нет

```
✅ Agent Framework:
- Новый агентный проект — по умолчанию, SK новых фич не получит
- Нужны мульти-агентные workflows (в SK оркестрация так и не стабилизировалась)
- AutoGen-проект — мигрировать в течение 6-12 месяцев, экосистема остановлена
- Нужны MCP / A2A / middleware / hosted agents

❌ Оставаться на SK (пока):
- Стабильная SK-система в production без потребности в новых оркестрациях —
  critical bugfixes гарантированы минимум год после GA Agent Framework
- Глубокая завязка на SK-специфику (prompt templates, planners, Memory API) —
  миграция дороже выгоды, пока система не требует новых возможностей
- Vector search на Microsoft.Extensions.VectorData миграции не требует вообще —
  этот слой общий и работает с обоими
```

Сама миграция во многом механическая: официальный guide (learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/) даёт маппинг API почти 1:1.

> [!question]- **Интервью: что случилось с Semantic Kernel в 2025-2026?**
> С public preview Microsoft Agent Framework (октябрь 2025) SK и AutoGen переведены в **maintenance mode** — bugfix/security only. Agent Framework достиг **GA 3 апреля 2026**: конвергенция SK (enterprise-фундамент, `Microsoft.Extensions.AI`-типы) и AutoGen (мульти-агентные паттерны). Ключевые упрощения: не нужен `Kernel` — агент строится прямо на `IChatClient`; tools без `[KernelFunction]`; единый `ChatClientAgent` вместо нескольких типов агентов; `RunAsync` возвращает единый `AgentResponse` вместо `IAsyncEnumerable`. Мульти-агентность — graph-based **Workflows** (замена `AgentGroupChat`). Стратегия: новые проекты — на Agent Framework; стабильный SK-production может не спешить (critical bugfixes минимум год после GA).

---

## Memory Connectors — persistent context

### Что это

**Memory** — хранение и retrieval information across conversations. Не путать с conversation history (in-memory current chat). **Long-term memory** persists между sessions.

> [!warning] Legacy API
> `MemoryBuilder` / `ISemanticTextMemory` (`Microsoft.SemanticKernel.Memory`) объявлены obsolete ещё до maintenance mode — новый код должен использовать `Microsoft.Extensions.VectorData` (Quickstart выше). Раздел оставлен для чтения существующих кодовых баз.

### Volatile memory (in-process)

```csharp
using Microsoft.SemanticKernel.Memory;

var memoryBuilder = new MemoryBuilder()
    .WithLoggerFactory(loggerFactory)
    .WithOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .WithMemoryStore(new VolatileMemoryStore());

var memory = memoryBuilder.Build();

// Store
await memory.SaveInformationAsync(
    collection: "user-preferences",
    text: "User prefers Python over JavaScript for data analysis",
    id: "pref-1");

await memory.SaveInformationAsync(
    collection: "user-preferences",
    text: "User works in healthcare industry",
    id: "pref-2");

// Retrieve relevant
var results = memory.SearchAsync("user-preferences", "what does user prefer for data work?", limit: 3);
await foreach (var result in results)
{
    Console.WriteLine($"{result.Relevance:F2}: {result.Metadata.Text}");
}
```

### Persistent — Postgres pgvector

```bash
dotnet add package Microsoft.SemanticKernel.Connectors.PostgreSQL
```

```csharp
var memoryBuilder = new MemoryBuilder()
    .WithOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .WithPostgresMemoryStore(connectionString, vectorSize: 1536);

var memory = memoryBuilder.Build();

await memory.SaveInformationAsync("knowledge", text: "Sample fact", id: "fact-1");
```

Available connectors (.NET 2026):
- **PostgreSQL (pgvector)** — most popular, mature
- **Qdrant** — high performance, rich filtering
- **Redis (RediSearch)** — fast in-memory
- **Azure AI Search** — managed Azure
- **Pinecone** — managed cloud
- **Weaviate** — managed/self-hosted
- **MongoDB Atlas** — vector search в Mongo
- **DuckDB / SQLite** — embedded

### TextMemoryPlugin — give kernel access to memory

```csharp
kernel.ImportPluginFromObject(new TextMemoryPlugin(memory), "memory");

// Теперь kernel может call memory.recall, memory.save в prompts
var prompt = """
    Previous user context:
    {{memory.recall input=$query collection='user-preferences' limit=3}}
    
    Current question: {{$query}}
    
    Answer using context above.
    """;
```

LLM может query memory automatically через function calling.

### Use cases memory

- **Personalization** — remember user preferences across sessions
- **Long conversations** — summarize и store context older than context window
- **Knowledge base** — RAG over documentation
- **Conversation history** — semantic search через past chats
- **User profiles** — facts about user accumulated over time

### Trade-offs

```
✅ Volatile memory:
- No infrastructure
- Fast
- Good для testing

❌ Lost on restart

✅ Persistent (Postgres/Qdrant):
- Survives restarts
- Shared между instances
- Production-ready

❌ Infrastructure dependency
❌ Network latency
```

---

## Streaming Responses

### Зачем

Default LLM call возвращает full response после complete generation. User waits 5-30 seconds (для long responses) — bad UX.

**Streaming**: tokens flow по мере generation. UI shows ответ progressively (как ChatGPT).

### Streaming в SK

```csharp
var chat = kernel.GetRequiredService<IChatCompletionService>();
var history = new ChatHistory("You are helpful");
history.AddUserMessage("Write a poem about Brno");

await foreach (var update in chat.GetStreamingChatMessageContentsAsync(history, kernel: kernel))
{
    Console.Write(update.Content);   // print as it arrives
}
Console.WriteLine();
```

### ASP.NET Core SSE endpoint

```csharp
app.MapPost("/api/chat/stream", async (
    HttpContext context,
    ChatRequest request,
    Kernel kernel,
    CancellationToken ct) =>
{
    var chat = kernel.GetRequiredService<IChatCompletionService>();
    var history = new ChatHistory(request.SystemPrompt);
    
    foreach (var msg in request.History)
        history.Add(new ChatMessageContent(msg.Role, msg.Content));
    
    history.AddUserMessage(request.Message);
    
    context.Response.ContentType = "text/event-stream";
    context.Response.Headers.CacheControl = "no-cache";
    
    await foreach (var update in chat.GetStreamingChatMessageContentsAsync(
        history, cancellationToken: ct))
    {
        if (!string.IsNullOrEmpty(update.Content))
        {
            await context.Response.WriteAsync(
                $"data: {JsonSerializer.Serialize(new { content = update.Content })}\n\n", ct);
            await context.Response.Body.FlushAsync(ct);
        }
    }
    
    await context.Response.WriteAsync("data: [DONE]\n\n", ct);
});
```

### Frontend (TypeScript)

```typescript
const response = await fetch('/api/chat/stream', {
    method: 'POST',
    body: JSON.stringify(request)
});

const reader = response.body!.getReader();
const decoder = new TextDecoder();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    const chunk = decoder.decode(value);
    const lines = chunk.split('\n');
    
    for (const line of lines) {
        if (line.startsWith('data: ')) {
            const data = line.substring(6);
            if (data === '[DONE]') return;
            
            const { content } = JSON.parse(data);
            appendToUI(content);
        }
    }
}
```

### Streaming + Function Calling

Тонкость: function calls часто блокируют streaming. Pattern: stream text portions, pause при function call, resume после result.

```csharp
await foreach (var update in chat.GetStreamingChatMessageContentsAsync(history, settings, kernel))
{
    if (update.Content != null)
        Console.Write(update.Content);
    
    // Tool calls в streaming — handled внутренне kernel'ом если AutoInvoke
}
```

---

## Multi-modal — vision и audio

### Vision — image understanding

```csharp
using Microsoft.SemanticKernel.ChatCompletion;

var chat = kernel.GetRequiredService<IChatCompletionService>();
var history = new ChatHistory();

// Image as URL
var message = new ChatMessageContent(AuthorRole.User, items: new ChatMessageContentItemCollection
{
    new TextContent("What's in this image?"),
    new ImageContent(new Uri("https://example.com/photo.jpg"))
});
history.Add(message);

// Or local file as bytes
var imageBytes = await File.ReadAllBytesAsync("photo.jpg");
var message2 = new ChatMessageContent(AuthorRole.User, items: new ChatMessageContentItemCollection
{
    new TextContent("Describe this"),
    new ImageContent(imageBytes, "image/jpeg")
});

var response = await chat.GetChatMessageContentAsync(history);
```

### Use cases vision

- OCR / document understanding
- Product image analysis (e-commerce)
- Medical imaging analysis
- Quality control (manufacturing)
- Accessibility (describe images for visually impaired)

### Audio — Whisper для speech-to-text

```bash
dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI
```

```csharp
var builder = Kernel.CreateBuilder()
    .AddOpenAIAudioToText("whisper-1", apiKey);

var kernel = builder.Build();
var audioToText = kernel.GetRequiredService<IAudioToTextService>();

var audioBytes = await File.ReadAllBytesAsync("recording.mp3");
var audioContent = new AudioContent(audioBytes, "audio/mp3");

var transcription = await audioToText.GetTextContentAsync(audioContent);
Console.WriteLine(transcription.Text);
```

### Text-to-speech

```csharp
builder.AddOpenAITextToAudio("tts-1", apiKey);
var textToAudio = kernel.GetRequiredService<ITextToAudioService>();

var audio = await textToAudio.GetAudioContentAsync("Hello, this is a test", new OpenAITextToAudioExecutionSettings("nova"));
await File.WriteAllBytesAsync("output.mp3", audio.Data!.Value.ToArray());
```

---

## Production patterns

### Rate limiting LLM calls

LLM APIs имеют rate limits (RPM, TPM). Production должен respect them.

```csharp
public class RateLimitedChatService(
    IChatCompletionService inner,
    RateLimiter limiter)
{
    public async Task<ChatMessageContent> GetCompletionAsync(
        ChatHistory history, CancellationToken ct)
    {
        using var lease = await limiter.AcquireAsync(permitCount: 1, ct);
        if (!lease.IsAcquired)
            throw new RateLimitExceededException();
        
        return await inner.GetChatMessageContentAsync(history, cancellationToken: ct);
    }
}

// Setup
var limiter = new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
{
    TokenLimit = 100,
    TokensPerPeriod = 60,
    ReplenishmentPeriod = TimeSpan.FromMinutes(1)
});
```

### Retries с exponential backoff

```csharp
using Polly;

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .OrResult<ChatMessageContent>(r => false)   // custom logic
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (ex, time, attempt, ctx) =>
        {
            logger.LogWarning(ex.Exception, "LLM call failed, retry {Attempt} after {Delay}",
                attempt, time);
        });

var result = await retryPolicy.ExecuteAsync(async () =>
    await chat.GetChatMessageContentAsync(history));
```

LLM-specific transient errors:
- **429 Too Many Requests** — retry с backoff
- **500/502/503** — retry с backoff
- **Timeout** — retry once
- **400 Bad Request** — НЕ retry (ваша prompt invalid)
- **401/403** — НЕ retry (auth issue)

### Cost tracking

```csharp
public sealed record ModelPricing(decimal InputPer1M, decimal OutputPer1M);

public class CostTrackingHandler : DelegatingHandler
{
    private readonly IMetrics _metrics;
    private readonly IReadOnlyDictionary<string, ModelPricing> _pricing;   // bind из appsettings
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var response = await base.SendAsync(request, ct);
        
        if (response.IsSuccessStatusCode)
        {
            var content = await response.Content.ReadAsStringAsync(ct);
            var json = JsonDocument.Parse(content);
            
            if (json.RootElement.TryGetProperty("usage", out var usage))
            {
                var promptTokens = usage.GetProperty("prompt_tokens").GetInt32();
                var completionTokens = usage.GetProperty("completion_tokens").GetInt32();
                var model = json.RootElement.GetProperty("model").GetString()!;
                
                // Прайс держи в конфиге, НЕ хардкодь: провайдеры меняют цены
                // и линейки моделей несколько раз в год
                var cost = _pricing.TryGetValue(model, out var p)
                    ? promptTokens * p.InputPer1M / 1_000_000m
                      + completionTokens * p.OutputPer1M / 1_000_000m
                    : 0m;
                
                _metrics.Counter("llm_tokens", new[] { ("model", model), ("type", "prompt") })
                    .Increment(promptTokens);
                _metrics.Counter("llm_cost_usd", new[] { ("model", model) })
                    .Increment((double)cost);
            }
        }
        
        return response;
    }
}
```

### Caching responses

```csharp
public class CachingChatService(
    IChatCompletionService inner,
    IDistributedCache cache,
    IEmbeddingGenerator<string, Embedding<float>> embedder)
{
    public async Task<ChatMessageContent> GetCompletionAsync(
        ChatHistory history, CancellationToken ct)
    {
        // Hash query для exact match cache
        var queryHash = ComputeHash(history.Last().Content);
        var cached = await cache.GetStringAsync($"llm:{queryHash}", ct);
        if (cached != null)
            return new ChatMessageContent(AuthorRole.Assistant, cached);
        
        var response = await inner.GetChatMessageContentAsync(history, cancellationToken: ct);
        
        await cache.SetStringAsync($"llm:{queryHash}", response.Content!,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) }, ct);
        
        return response;
    }
}
```

**Semantic caching** — cache by embedding similarity, не exact match:

```csharp
// Pseudo-code
var queryEmbedding = await embedder.GenerateVectorAsync(query, cancellationToken: ct);
var similar = await vectorStore.SearchAsync(queryEmbedding, top: 1, threshold: 0.95);

if (similar.Any())
    return similar.First().CachedResponse;
// else: real LLM call + cache
```

### Observability — OpenTelemetry для LLM

```csharp
using OpenTelemetry.Trace;

builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddSource("Microsoft.SemanticKernel*")   // SK auto-instruments
        .AddOtlpExporter());
```

Trace spans show:
- LLM request/response (с token counts)
- Function calls
- Embedding generation
- Vector store queries

Critical для debugging: видишь chain of operations, latency breakdown, costs per request.

> [!question]- **Интервью: production patterns LLM apps?**
> 1) **Rate limiting** — respect LLM provider RPM/TPM limits, internal RateLimiter. 2) **Retries с exponential backoff** для 429/5xx (НЕ 4xx). 3) **Cost tracking** — log tokens per request + dollar cost (через DelegatingHandler или metrics). 4) **Caching** — exact match для FAQs, semantic similarity для variations. 5) **Observability** — OpenTelemetry, SK auto-instruments. 6) **Fallback** — secondary model или cached response при primary failure. 7) **Prompt versioning** — track prompt changes как code, A/B testing. **Cost optimization**: 1) cheaper model для simple tasks (mini/nano-tier у OpenAI, Haiku-класс у Anthropic). 2) caching. 3) shorter prompts. 4) max_tokens limits.

---

## Comparison — SK vs LangChain vs LlamaIndex

| | Semantic Kernel | LangChain | LlamaIndex |
|--|-----------------|-----------|------------|
| Language | C# / Python / Java | Python (TS port) | Python (TS port) |
| Microsoft | ✅ | ❌ | ❌ |
| Maturity | Stable, но maintenance mode — новое развитие в Agent Framework | Mature, big ecosystem | Mature, RAG-focused |
| Strengths | .NET native, enterprise, Azure integration | Largest ecosystem, many integrations | Best for RAG, advanced retrieval |
| Weaknesses | Smaller ecosystem, заморожен (bugfix only) | Frequent breaking changes | Less general-purpose |
| Function calling | ✅ Auto + manual | ✅ | ✅ |
| Agents | GA, но заморожены → Agent Framework | LangGraph (mature) | Built-in |
| Memory connectors | Many | Many | Many |
| Streaming | ✅ | ✅ | ✅ |
| Vector stores | 8+ | 50+ | 30+ |

### Когда что

```
Choose Semantic Kernel:
✅ Существующая SK-кодовая база (для НОВЫХ агентных проектов — Agent Framework)
✅ .NET / Azure shop
✅ Enterprise compliance
✅ Strong typing нужен
✅ Microsoft support contract

Choose LangChain:
✅ Python ecosystem
✅ Rapid prototyping
✅ Many third-party integrations needed
✅ Community support

Choose LlamaIndex:
✅ RAG primary use case
✅ Advanced retrieval (hierarchical, hybrid)
✅ Document indexing complexity
✅ Knowledge bases
```

### Hybrid

Возможно use SK для C# orchestration + LlamaIndex (через REST) для retrieval. Или SK + custom retrieval pipeline.

> [!question]- **Интервью: SK vs LangChain — когда что?**
> **SK** — когда .NET shop, enterprise, Azure integration. Strong typing, Microsoft backing. **LangChain** — Python-first, biggest ecosystem (50+ vector stores, 100+ tools, mature agents через LangGraph). **LlamaIndex** — RAG-specialized, лучше для document indexing, advanced retrieval (hierarchical, hybrid search). **Reality check**: для C# проектов SK + custom code часто выигрывает над попытками use Python libraries через REST. **Trade-off**: SK ecosystem меньше, но достаточен для большинства production cases. **2026**: развитие агентной части ушло в Microsoft Agent Framework (Workflows) — именно он конкурирует с LangGraph; сам SK — в maintenance mode, выбирать его для нового агентного проекта уже не стоит.

---

## Подводные камни

### Embedding dimension mismatch
Index built с 1536-dim, query с 768-dim → garbage results. **Match dimensions explicitly**. И помни: смена embedding-модели = полный reindex корпуса — выбирай размерность под задачу сразу (384 для быстрого/дешёвого, 768/1024 для баланса, 1536+ для точности).

### Distance function mismatch  
Index с CosineSimilarity, query с DotProduct → wrong ranking. **Same metric везде**.

### Hybrid search недооценён
Чисто vector search упускает exact matches (SKU, codes). **BM25 + vector + reranker** (`BAAI/bge-reranker`) обычно лучше. Qdrant / Weaviate умеют из коробки, в pgvector — вручную.

### Multi-tenancy без RLS на vector store
Все tenants видят all vectors. **Filter by tenant_id ПЕРЕД similarity**, не после.

### Стоимость embedding при масштабе
100k документов × 1k токенов × $0.02/1M = $2. При 10M документов → $200, плюс re-embedding при смене модели. **Bulk-генерация через batch API дешевле; считай стоимость reindex до выбора модели**.

### Context window overflow
Stuff 50 retrieved chunks в prompt → exceeds 128K context, errors или garbage. **Limit top-K, summarize если нужно более**.

### Prompt injection
User input injected в prompt template → LLM может ignore instructions. **Sanitize, или use structured outputs, или separate user input от system prompt**.

### Cost explosion
Флагманская модель for everything → bill $1000s/month. **Use cheaper models (mini/nano-tier, Haiku-класс) для simple tasks. Cache aggressive. Monitor cost per query**.

### Tool calling без timeouts
Agent endless loop вызывая tools → cost explosion. **Max iterations, max time, circuit breaker**.

### Memory leak в long conversations
History grows unbounded → context overflow + costs grow. **Summarize old messages, sliding window, или RAG over history**.

### Streaming без back-pressure
Server streams faster than client consumes → memory grows. **Implement back-pressure через channel limits**.

---

## Cheat sheet (extended)

```csharp
// === Setup ===
var builder = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion("gpt-5-mini", apiKey)   // подставь актуальную модель
    .AddOpenAIEmbeddingGenerator("text-embedding-3-small", apiKey);

builder.Plugins.AddFromType<MyPlugin>();
var kernel = builder.Build();

// === Function calling auto ===
var settings = new OpenAIPromptExecutionSettings
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
};
var chat = kernel.GetRequiredService<IChatCompletionService>();
var response = await chat.GetChatMessageContentAsync(history, settings, kernel);

// === Streaming ===
await foreach (var update in chat.GetStreamingChatMessageContentsAsync(history, settings, kernel))
{
    Console.Write(update.Content);
}

// === Vision ===
var msg = new ChatMessageContent(AuthorRole.User, items: new ChatMessageContentItemCollection
{
    new TextContent("Describe"),
    new ImageContent(imageBytes, "image/jpeg")
});

// === Memory с pgvector ===
var memory = new MemoryBuilder()
    .WithOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .WithPostgresMemoryStore(connStr, vectorSize: 1536)
    .Build();

await memory.SaveInformationAsync("collection", text, id);
var results = memory.SearchAsync("collection", query, limit: 5);

// === Agent ===
var agent = new ChatCompletionAgent
{
    Name = "Assistant",
    Instructions = "You are helpful",
    Kernel = kernel
};

var thread = new ChatHistoryAgentThread();
await foreach (var resp in agent.InvokeAsync(thread))
    Console.WriteLine(resp.Message.Content);

// === Multi-agent ===
var groupChat = new AgentGroupChat(researcher, writer, reviewer)
{
    ExecutionSettings = new()
    {
        TerminationStrategy = new ApprovalTerminationStrategy
        {
            Agents = [reviewer],
            MaximumIterations = 10
        }
    }
};

// === Production: rate limit + retry + cache ===
services.AddHttpClient<MyLlmClient>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddHttpMessageHandler<CostTrackingHandler>();

// === OpenTelemetry ===
services.AddOpenTelemetry()
    .WithTracing(t => t.AddSource("Microsoft.SemanticKernel*").AddOtlpExporter());
```

---

## Practice exercises (extended)

### 1. Function calling agent
Build assistant с 5 functions: `GetWeather`, `SearchProducts`, `CheckInventory`, `CreateOrder`, `SendEmail`. User says "Order 2 black t-shirts in size M and notify me when shipped". LLM должен chain functions.

### 2. RAG над documentation
Index project docs (markdown files) в pgvector. Build endpoint `/ask?q=...` — semantic search top-5 chunks + LLM answer. Add citation metadata.

### 3. Multi-agent blog writer
3 agents: Researcher (web search tools), Writer (drafts), Reviewer (approves/rejects). Goal: write blog post about given topic. Stop когда reviewer approves или 5 iterations.

### 4. Production observability
Wrap LLM service с OpenTelemetry + cost tracking. Dashboard: tokens/min, cost/hour per model, latency p50/p99, error rate.

### 5. Semantic cache
Cache LLM responses by embedding similarity (threshold 0.95). On cache hit — return stored response. Track hit rate. Compare cost savings.

## См. также

- [[project-setup|Project Setup]]
- [[observability|Observability]] — трассировка LLM-вызовов через OTel
- [[caching|Caching]] — кэш embedding-запросов
- [[99_reading-list|Reading List]]
