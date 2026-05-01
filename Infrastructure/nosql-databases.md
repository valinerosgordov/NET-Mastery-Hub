---
tags: [infrastructure, nosql, mongodb, redis, cosmos, dynamodb, cassandra, polyglot-persistence, senior]
level: Senior
date: 2026-05-01
---

# NoSQL Databases — обзор и выбор

> **Когда NoSQL > SQL и какой NoSQL для какой задачи.** MongoDB / Redis / Cosmos DB / DynamoDB / Cassandra — реальные сценарии. Closes пробел "знаю SQL, но не понимаю когда брать NoSQL и какой именно".

---

## Что это, зачем и когда

### Зачем нужен NoSQL

SQL великолепен для **реляционных данных** с ACID гарантиями. Но:

- ❌ Schema rigidity — изменение схемы дорого на больших таблицах
- ❌ Vertical scaling предел (1 server can do only so much)
- ❌ JOINs не масштабируются на distributed data
- ❌ Не идеально для unstructured data (logs, metrics, JSON)
- ❌ Очень read-heavy / write-heavy workloads — overhead транзакций

NoSQL **жертвует** ACID/SQL/JOINs ради:
- Horizontal scaling (sharding)
- Flexible schema
- Performance для specific patterns
- Specialized data models (graph / time-series / key-value)

### NoSQL типы

```
Document     → MongoDB, Cosmos, Firebase, CouchDB
Key-Value    → Redis, DynamoDB, Memcached
Column-Family → Cassandra, ScyllaDB, HBase
Graph         → Neo4j, DGraph, Neptune
Time-Series   → InfluxDB, TimescaleDB, ClickHouse
Multi-model   → Cosmos DB (всё сразу), ArangoDB
Search        → Elasticsearch, OpenSearch (часто причисляют)
```

### Главное правило

> **Polyglot persistence** — выбирай DB по задаче, не одну на всё.
>
> Real production app может использовать:
> - **PostgreSQL** для core entities (users, orders)
> - **Redis** для sessions / cache
> - **Elasticsearch** для search
> - **ClickHouse** для analytics
> - **MongoDB** для CMS-like content

См. [[../EFCore/dapper-comparison|Dapper vs EF]] — для SQL контекста.

---

## 1. MongoDB — document database

### Когда использовать

✅ **Хорошо для:**
- Content management (статьи, посты, комментарии)
- Product catalog в e-commerce (разные товары — разные attributes)
- User profiles с переменными полями
- Real-time analytics (агрегации)
- Prototyping (schema может меняться часто)

❌ **Плохо для:**
- Финансовых транзакций (ACID на multi-document — limited)
- Сильно реляционных данных (много JOINs)
- Малых проектов (overkill — Postgres + JSONB достаточно)

### Setup в .NET

```bash
dotnet add package MongoDB.Driver
```

```csharp
public record Product
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string? Id { get; init; }

    public string Name { get; init; } = "";
    public decimal Price { get; init; }
    public Dictionary<string, object>? Attributes { get; init; }
    public List<string> Tags { get; init; } = new();
    public DateTime CreatedAt { get; init; }
}

public class ProductRepository
{
    private readonly IMongoCollection<Product> _collection;

    public ProductRepository(IConfiguration config)
    {
        var client = new MongoClient(config["MongoDB:ConnectionString"]);
        var db = client.GetDatabase("shop");
        _collection = db.GetCollection<Product>("products");
    }

    public async Task<Product?> GetByIdAsync(string id) =>
        await _collection.Find(p => p.Id == id).FirstOrDefaultAsync();

    public async Task<List<Product>> SearchAsync(string text, int skip = 0, int take = 20) =>
        await _collection
            .Find(Builders<Product>.Filter.Text(text))
            .Skip(skip)
            .Limit(take)
            .ToListAsync();

    public async Task CreateAsync(Product p) =>
        await _collection.InsertOneAsync(p);
}
```

### Indexing

```csharp
await _collection.Indexes.CreateOneAsync(
    new CreateIndexModel<Product>(
        Builders<Product>.IndexKeys.Text(p => p.Name).Text(p => p.Description)));

await _collection.Indexes.CreateOneAsync(
    new CreateIndexModel<Product>(
        Builders<Product>.IndexKeys.Ascending(p => p.CreatedAt)));
```

### Transactions

```csharp
using var session = await _client.StartSessionAsync();
session.StartTransaction();

try
{
    await _orders.InsertOneAsync(session, order);
    await _inventory.UpdateOneAsync(session, ...);
    await session.CommitTransactionAsync();
}
catch
{
    await session.AbortTransactionAsync();
    throw;
}
```

---

## 2. Redis — key-value + structures

### Что Redis может

```
Strings        → SET key value (cache, counters)
Hashes         → HSET key field value (objects)
Lists          → LPUSH/RPUSH (queues, timelines)
Sets           → SADD (unique items, tags)
Sorted Sets    → ZADD (leaderboards, top-N)
Streams        → XADD (event log)
Pub/Sub        → PUBLISH/SUBSCRIBE (real-time)
Bitmaps        → SETBIT (analytics)
Geo            → GEOADD (location queries)
```

### Когда использовать

✅ **Хорошо для:** caching, sessions, rate limiting, leaderboards, pub/sub, distributed locks, job queues

❌ **Плохо для:** primary database для core data, большие объекты (>1 MB), complex queries

### Setup

```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]));

public class CacheService
{
    private readonly IDatabase _db;

    public CacheService(IConnectionMultiplexer redis) => _db = redis.GetDatabase();

    public async Task<T?> GetAsync<T>(string key)
    {
        var value = await _db.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value!) : default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? ttl = null) =>
        await _db.StringSetAsync(key, JsonSerializer.Serialize(value), ttl);

    public async Task<long> IncrementAsync(string key) =>
        await _db.StringIncrementAsync(key);

    public async Task AddScoreAsync(string leaderboard, string user, double score) =>
        await _db.SortedSetAddAsync(leaderboard, user, score);

    public async Task<List<(string User, double Score)>> GetTopAsync(string leaderboard, int n) =>
        (await _db.SortedSetRangeByRankWithScoresAsync(leaderboard, 0, n - 1, Order.Descending))
            .Select(e => (e.Element.ToString(), e.Score))
            .ToList();
}
```

См. [[../AspNetCore/caching|Caching]].

---

## 3. Cosmos DB — Microsoft multi-model

### Особенности

- Multi-model (Document/Key-Value/Graph/Cassandra API)
- Global distribution с multi-region writes
- Tunable consistency (5 levels)
- SLA single-digit ms latency, 99.999% availability
- Auto-scaling через RU/s (Request Units)

### Когда

✅ Multi-region apps, variable workload, уже на Azure, multi-tenant SaaS
❌ Бюджет ограничен, self-hosted, простые apps

### Setup

```csharp
builder.Services.AddSingleton(_ =>
    new CosmosClient(builder.Configuration["Cosmos:ConnectionString"]));

public class OrderRepository
{
    private readonly Container _container;

    public OrderRepository(CosmosClient client)
    {
        var db = client.GetDatabase("shop");
        _container = db.GetContainer("orders");
    }

    public async Task<Order?> GetAsync(string id, string partitionKey)
    {
        try
        {
            var response = await _container.ReadItemAsync<Order>(id, new PartitionKey(partitionKey));
            return response.Resource;
        }
        catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }
    }

    public async Task SaveAsync(Order order) =>
        await _container.UpsertItemAsync(order, new PartitionKey(order.UserId));
}
```

### Critical: partition key choice

```
✅ Хорошие: userId, tenantId, date prefix
❌ Плохие: id (random), timestamp (hot last partition), boolean (только 2 partitions)
```

---

## 4. DynamoDB — AWS managed

### Когда

✅ AWS-native, известные access patterns, serverless (Lambda + DynamoDB), IoT/gaming  
❌ Ad-hoc queries, complex aggregations, joins-heavy

### Setup

```csharp
[DynamoDBTable("Orders")]
public class Order
{
    [DynamoDBHashKey] public string UserId { get; set; } = "";
    [DynamoDBRangeKey] public string OrderId { get; set; } = "";
    public decimal Total { get; set; }
}

public class OrderRepository
{
    private readonly DynamoDBContext _context;

    public OrderRepository(IAmazonDynamoDB client) => _context = new DynamoDBContext(client);

    public Task SaveAsync(Order order) => _context.SaveAsync(order);
    public Task<Order?> GetAsync(string userId, string orderId) =>
        _context.LoadAsync<Order>(userId, orderId);
}
```

### Single-table design

```
PK              SK                Type
USER#42         PROFILE           User
USER#42         ORDER#1001        Order
USER#42         ORDER#1002        Order
PRODUCT#100     INFO              Product
PRODUCT#100     REVIEW#1          Review
```

---

## 5. Cassandra — write-heavy column store

### Когда

✅ Write-heavy (миллионы writes/sec), time-series, logging, IoT, multi-DC replication  
❌ Read-heavy, complex queries, ACID transactions

```csharp
public class TelemetryRepository
{
    private readonly ISession _session;

    public async Task RecordAsync(string deviceId, DateTime ts, double value)
    {
        var stmt = await _session.PrepareAsync(@"
            INSERT INTO telemetry (device_id, day, timestamp, value)
            VALUES (?, ?, ?, ?)");
        await _session.ExecuteAsync(stmt.Bind(deviceId, ts.Date, ts, value));
    }
}
```

---

## 6. Case Study #1 — E-commerce каталог

**Задача:** 1M товаров, разные attributes (книга — author/pages, мебель — material/dimensions). Поиск, фильтры, частые updates цен.

### ❌ SQL — boilerplate

```sql
-- Узкая таблица + EAV (Entity-Attribute-Value)
CREATE TABLE products (id, name, price);
CREATE TABLE product_attributes (product_id, attr_name, attr_value);
-- Каждый product вытаскивается через N JOIN'ов или серию queries
```

### ✅ MongoDB — natural fit

```csharp
public record Product
{
    public string? Id { get; init; }
    public string Category { get; init; } = "";
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
    public Dictionary<string, object> Attributes { get; init; } = new();
    public List<string> Tags { get; init; } = new();
}

// Каждый товар — одно flexible document
new Product
{
    Category = "books",
    Name = "C# in Depth",
    Price = 45m,
    Attributes = new()
    {
        ["author"] = "Jon Skeet",
        ["pages"] = 900,
        ["isbn"] = "978-1617294532"
    }
}

new Product
{
    Category = "furniture",
    Name = "Office Chair",
    Price = 299m,
    Attributes = new()
    {
        ["material"] = "leather",
        ["dimensions"] = new { width = 60, height = 110, depth = 60 },
        ["color"] = "black"
    }
}
```

**Почему лучше:**
- Гибкая schema — новые товарные категории не требуют migrations
- Один query вместо JOIN'ов
- JSONB в Postgres даёт похожее, но Mongo лучше масштабируется на 100M+ items

---

## 7. Case Study #2 — Real-time leaderboard игры

**Задача:** Multiplayer игра, 100K активных, leaderboard top-100, обновляется каждые 5 секунд.

### ❌ SQL

```sql
SELECT user_id, score FROM scores ORDER BY score DESC LIMIT 100;
-- При 100K игроков — медленно даже с индексом
-- Update score каждые 5 сек × 100K = 20K writes/sec
```

### ✅ Redis Sorted Sets

```csharp
// Update score
await _redis.SortedSetAddAsync("leaderboard:global", userId, newScore);
// O(log N), millions of ops/sec

// Top-100
var top = await _redis.SortedSetRangeByRankWithScoresAsync(
    "leaderboard:global", 0, 99, Order.Descending);
// O(log N + 100), микросекунды

// Position user'а
var rank = await _redis.SortedSetRankAsync("leaderboard:global", userId, Order.Descending);
// O(log N)
```

**Production:** Redis для real-time leaderboard, Postgres для permanent records (raw data).

---

## 8. Case Study #3 — Session storage

**Задача:** ASP.NET Core app на 5 instances за load balancer. Sessions нужны, но in-memory не работает (sticky sessions — anti-pattern).

### ❌ In-memory sessions

```csharp
builder.Services.AddSession();  // in-memory by default
// Каждый instance имеет свои sessions
// User → instance1 → login OK
// User → instance2 → "не залогинен"!
// Sticky sessions помогают, но breaks при restart
```

### ✅ Redis-backed sessions

```csharp
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = builder.Configuration["Redis:ConnectionString"];
});

builder.Services.AddSession(opts =>
{
    opts.IdleTimeout = TimeSpan.FromMinutes(30);
    opts.Cookie.HttpOnly = true;
    opts.Cookie.IsEssential = true;
});

app.UseSession();
```

**Все instances share sessions через Redis.**

См. [[../AspNetCore/caching|Caching]] и [[../AspNetCore/auth-security|Auth & Security]].

---

## 9. Case Study #4 — IoT телеметрия

**Задача:** 100K IoT-устройств, каждое шлёт telemetry каждую секунду. **100K writes/sec, 8.6 billion records/day**.

### ❌ Postgres

Невозможно — single-master не выдержит 100K writes/sec.

### ✅ Cassandra (или TimescaleDB / ClickHouse)

```csharp
// Schema partitioned по device + day
CREATE TABLE telemetry (
    device_id text,
    day date,
    timestamp timestamp,
    temperature double,
    humidity double,
    PRIMARY KEY ((device_id, day), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Преимущества:**
- Linear scaling (добавь nodes = больше throughput)
- Write-optimized (LSM tree)
- Auto-replication
- Time-series friendly через clustering

**Альтернативы:**
- TimescaleDB (Postgres extension) — если SQL предпочтительнее, до 1M+ writes/sec
- ClickHouse — для analytics queries поверх IoT data
- InfluxDB — purpose-built TSDB

См. [[messaging|Messaging]] (Kafka для ingestion).

---

## 10. Case Study #5 — Multi-region SaaS

**Задача:** SaaS пользователи в US, EU, Asia. Latency должна быть < 100 ms везде. Данные могут быть consistent eventually.

### ❌ Single-region Postgres

```
US user → US server (10 ms) → US DB (5 ms)         ✓
EU user → US server (100 ms) → US DB (5 ms)         ✗ slow
Asia user → US server (200 ms) → US DB (5 ms)        ✗ very slow
```

### ✅ Cosmos DB multi-region

```csharp
var cosmosOptions = new CosmosClientOptions
{
    ApplicationRegion = Regions.EastUS,  // ближайший region
    ConsistencyLevel = ConsistencyLevel.Session  // good balance
};

var client = new CosmosClient(connStr, cosmosOptions);
```

**Cosmos автоматически:**
- Реплицирует данные в выбранные regions (EU, Asia, US)
- Routes user requests к ближайшему region
- Handles conflicts (last-write-wins or custom merge)

**Trade-off:** Eventual consistency между regions (1-5 sec lag).

См. [[../Architecture/distributed-systems|Distributed Systems]].

---

## 11. Case Study #6 — Migration из MongoDB обратно в Postgres

**Сценарий:** Стартап начал с MongoDB ("schema-less, fast development"). Через 2 года data стало реляционной (orders → users → products → reviews). MongoDB transactions медленные. Команда решает migrate.

### Когда это правильно

✅ **Migration justified когда:**
- Schema стабилизировалась
- Reads = JOIN-heavy
- Нужны строгие ACID
- DBA expertise есть в Postgres
- Postgres + JSONB покрывает remaining flexibility

### Migration strategy

```csharp
// Stage 1: Dual-write
public async Task SaveOrder(Order order)
{
    await _mongo.SaveAsync(order);   // existing
    await _postgres.SaveAsync(order);  // new
    // Если postgres failed — log, eventually sync
}

// Stage 2: Read from postgres, fallback на mongo
public async Task<Order?> GetOrder(int id)
{
    var fromPg = await _postgres.GetAsync(id);
    if (fromPg != null) return fromPg;

    var fromMongo = await _mongo.GetAsync(id);
    if (fromMongo != null)
        await _postgres.SaveAsync(fromMongo);  // backfill

    return fromMongo;
}

// Stage 3: Off Mongo (после full backfill)
```

**Lesson:** Choose right database сначала. Migration — expensive. Postgres + JSONB достаточно для большинства "schema-less" needs.

---

## 12. Common Pitfalls

### 1. Использовать NoSQL "потому что modern"

```
❌ "MongoDB лучше Postgres потому что NoSQL"
✅ "MongoDB лучше для нашего use case потому что [конкретная причина]"
```

90% проектов хорошо работают на Postgres. NoSQL — для specific needs.

### 2. Ignore consistency level в distributed NoSQL

```csharp
// Cosmos DB — default consistency = Session
// User создал order → сразу проверяет — может не увидеть!

// ✅ Strong consistency для critical reads
var item = await _container.ReadItemAsync<Order>(id, pk,
    new ItemRequestOptions { ConsistencyLevel = ConsistencyLevel.Strong });
```

### 3. Cosmos partition key — wrong choice

```
❌ partitionKey = id (uniform — нет grouping для queries)
❌ partitionKey = timestamp (hot последняя partition)
✅ partitionKey = userId (queries обычно по user)
```

### 4. MongoDB embedded vs referenced

```csharp
// ❌ Embedded когда subdocuments часто меняются independently
public record User
{
    public List<Order> Orders { get; init; }  // 10K orders в user document!
}

// ✅ Reference
public record Order
{
    public string UserId { get; init; }  // FK-like
}

// User отдельно, queries отдельно
```

Embed когда: subdoc маленький и читается всегда вместе с parent.  
Reference когда: subdoc большой или независимый.

### 5. Redis для primary storage

```
❌ Redis как primary store user data
   - In-memory — power loss → data loss
   - Persistence (RDB/AOF) есть, но slower и не как Postgres
   - Лучше Postgres + Redis cache
```

### 6. Single hot key в Redis

```csharp
// ❌ Все писатели — одна key
await _redis.StringIncrementAsync("global:counter");
// При 100K RPS — single Redis instance bottleneck

// ✅ Sharded counters
var shard = userId % 16;
await _redis.StringIncrementAsync($"counter:{shard}");
// Periodically — sum для total
```

### 7. DynamoDB запросы без index

```csharp
// ❌ Scan вместо Query — медленно и дорого
var all = await _context.ScanAsync<Order>(...).GetRemainingAsync();
// Читает ВСЁ table — стоит O(N) RU

// ✅ Query по PK + SK
var query = _context.QueryAsync<Order>(userId);
// O(matched items) RU
```

### 8. Missing TTL в Redis

```csharp
// ❌
await _redis.StringSetAsync("temp_data", value);
// Memory растёт infinitely

// ✅ TTL ALWAYS для temporary data
await _redis.StringSetAsync("temp_data", value, TimeSpan.FromHours(1));
```

### 9. MongoDB schema validation отключён

```csharp
// MongoDB — schema-less, но schema validation важен
var validator = new BsonDocument
{
    ["$jsonSchema"] = new BsonDocument
    {
        ["bsonType"] = "object",
        ["required"] = new BsonArray { "name", "price" },
        ["properties"] = new BsonDocument
        {
            ["price"] = new BsonDocument { ["bsonType"] = "decimal", ["minimum"] = 0 }
        }
    }
};

await _db.CreateCollectionAsync("products",
    new CreateCollectionOptions<Product> { Validator = validator });
```

### 10. Cassandra с small partitions

```
❌ partition_key = (user_id, message_id)  
   Каждое сообщение — own partition. Миллиарды partitions = плохо

✅ partition_key = (user_id, day), clustering = (timestamp, message_id)
   ~1000 messages per partition — sweet spot
```

---

## 13. Best Practices

### Database choice

- **Default — Postgres + JSONB.** Только если конкретный need — NoSQL
- **Polyglot persistence** — multiple DBs для multiple needs
- **Document store (Mongo/Cosmos)** — flexible schema + nested data
- **Key-value (Redis)** — caching, sessions, real-time, rate limit
- **Column-family (Cassandra)** — write-heavy time-series
- **Graph (Neo4j)** — связи > entities (social, fraud)
- **Time-series (Influx/Timescale)** — IoT, metrics, monitoring

### MongoDB

- **Schema validation** даже если "schema-less"
- **Indexes** на каждое поле в queries
- **Embed vs reference** — по access pattern
- **Avoid unbounded arrays** (10K+ items в document)

### Redis

- **TTL ALWAYS** для temporary data
- **Pipelining** для batch operations
- **Avoid hot keys** через sharding
- **Pub/Sub** не для durable messaging (используй Redis Streams или Kafka)

### Cosmos DB

- **Partition key** — самое важное решение, hard to change
- **Use SDK** preferred over REST
- **Provisioned RU/s** vs **Serverless** vs **Autoscale**
- **Indexes** — exclude unused properties (saves RU)

### DynamoDB

- **Single-table design** для known patterns
- **GSI / LSI** для secondary access patterns
- **Query > Scan** всегда
- **DynamoDB Streams** для CDC (change data capture)

---

## 14. Cheat sheet

| Сценарий | DB |
|----------|-----|
| User accounts, orders | **PostgreSQL** (default) |
| Product catalog с гибкими attributes | **MongoDB** или Postgres + JSONB |
| Session storage | **Redis** |
| Cache | **Redis** |
| Real-time leaderboard | **Redis** Sorted Sets |
| Rate limiting | **Redis** counters + TTL |
| Distributed lock | **Redis** SETNX |
| Pub/Sub messaging | **Redis** или **RabbitMQ** |
| Multi-region SaaS | **Cosmos DB** |
| AWS-native serverless | **DynamoDB** |
| IoT telemetry, 100K+ writes/sec | **Cassandra** или **TimescaleDB** |
| Time-series metrics | **InfluxDB** или **TimescaleDB** |
| Analytics over events | **ClickHouse** |
| Social graph, recommendations | **Neo4j** |
| Full-text search | **Elasticsearch** |

---

## 15. Decision tree

```
Какой DB выбрать?
│
├── Реляционные данные с ACID?
│   └── PostgreSQL (default!)
│
├── Flexible schema, nested data?
│   ├── < 1M items, simple → Postgres + JSONB
│   └── > 1M items, complex → MongoDB / Cosmos
│
├── In-memory speed нужен?
│   ├── Cache → Redis
│   ├── Sessions → Redis
│   └── Rate limit / counters → Redis
│
├── Write-heavy (>10K writes/sec)?
│   ├── Time-series → TimescaleDB / Cassandra
│   ├── Analytics → ClickHouse
│   └── General → Cassandra
│
├── Multi-region active-active?
│   └── Cosmos DB (Azure) / DynamoDB Global (AWS)
│
├── Graph relationships?
│   └── Neo4j (или Cosmos Gremlin)
│
└── Search?
    └── Elasticsearch
```

---

## 16. См. также

- [[../EFCore/dapper-comparison|Dapper vs EF Core]] — для SQL контекста
- [[../EFCore/basics-tracking|EF Core Basics]]
- [[../SQL/postgresql-deep|PostgreSQL Deep]] — JSONB как NoSQL alternative
- [[../AspNetCore/caching|Caching]] — Redis-based caching
- [[../Architecture/distributed-systems|Distributed Systems]] — CAP theorem, consistency
- [[messaging|Messaging]] — Kafka для ingestion в NoSQL
- [[elasticsearch|Elasticsearch]] (TBD) — search engine
- [[time-series-dbs|Time-Series DBs]] (TBD)
- [[graph-databases|Graph DBs]] (TBD)

## 17. Reading list

- **Designing Data-Intensive Applications** — Martin Kleppmann (must-read для понимания DBs)
- **NoSQL Distilled** — Pramod Sadalage, Martin Fowler
- **MongoDB: The Definitive Guide** — Kristina Chodorow
- **Redis in Action** — Josiah Carlson
- **Cassandra: The Definitive Guide** — Eben Hewitt
- **Microsoft Docs — Cosmos DB** — learn.microsoft.com/azure/cosmos-db
- **AWS DynamoDB Best Practices** — docs.aws.amazon.com/amazondynamodb
- **High Scalability blog** — highscalability.com
