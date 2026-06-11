---
tags: [system-design, interview, tech-lead, rate-limiter, url-shortener, feed, cache]
level: Senior
---

# System Design — типовые ответы для Tech Lead интервью

## Что это, зачем и когда

### Что такое system design интервью?
**Открытое задание "спроектируйте X"** на 45-60 минут. Без правильного ответа — оценивается процесс мышления: выявление requirements, выбор tradeoffs, capacity planning, выявление bottleneck'ов и решение их.

**Аналогия:** Это не как unit-test где есть expected output. Это как дизайн-ревью: ты архитектор, у которого есть жидкое ТЗ, час времени и senior-инженер на той стороне, который будет проверять обоснованность каждого выбора.

### Что от тебя хотят услышать

1. **Clarifying questions** (5 мин) — что точно нужно? Кто пользователь? Какая нагрузка?
2. **Functional requirements** (5 мин) — что система делает (явно списком)
3. **Non-functional requirements** (5 мин) — RPS, latency, availability, consistency
4. **Capacity estimation** (5 мин) — DAU, QPS, размер данных
5. **High-level design** (10-15 мин) — компоненты на whiteboard
6. **Deep dives** (15-20 мин) — конкретные проблемы (DB schema, hot paths, failure modes)
7. **Wrap-up** — bottleneck'и, что бы добавил с большим временем

---

## Универсальный фреймворк

### Functional vs Non-functional requirements

```
Functional (что система делает):
- Пользователь может X
- Система отображает Y
- Админ настраивает Z

Non-functional (характеристики):
- 1M DAU, 100 QPS
- p99 latency < 200ms
- 99.9% availability (≈ 8.7 часов downtime/год)
- Consistency: eventual для лент, strong для транзакций
- Storage: 100GB новых данных в месяц
```

### Capacity estimation cheatsheet

```
1 day  = 86 400 секунд ≈ 100 000 секунд
1 year = 31.5 миллионов секунд ≈ 3 × 10^7

1KB = 1000 bytes (B)
1MB = 1000 KB
1GB = 1000 MB
1 миллион = 10^6
1 миллиард = 10^9

QPS:
1M DAU * 10 actions/day = 10M actions/day
10M / 100K = 100 QPS average
Peak QPS ≈ 5x average = 500 QPS

Storage:
100M users × 1KB profile = 100GB
1B writes/day × 200B = 200GB/day = 73TB/year
```

### Latency numbers (по памяти)

```
Memory access:        100ns
SSD random read:       100µs (1000x slower)
HDD random seek:        10ms (100x slower again)
Network (datacenter):  500µs
Network (cross-region): 100ms
SQL query (cached):     1ms
SQL query (cold):      50-200ms
HTTP API call:         50-500ms
LLM API call:          500-5000ms
```

---

## Шаблон 1 — Rate Limiter

### Requirements
"Сделать rate limiter для API: 100 запросов/минуту на пользователя, гарантия не пропустить лимит, latency < 5ms"

### Algorithms — что выбрать

| Algorithm | Burst tolerance | Память на user | Сложность |
|-----------|----------------|----------------|-----------|
| **Fixed window counter** | Дёрганый (2x burst на стыке окон) | 1 counter | Простейшая |
| **Sliding window log** | Точный | List of timestamps | Средняя |
| **Sliding window counter** | Точный | 2 counter'а | Средняя |
| **Token bucket** | Burst до bucket size | bucket + tokens count | Средняя |
| **Leaky bucket** | Сглаживание (queue) | queue | Высокая |

**Выбор для RPS API:** Token bucket. Smooth burst tolerance + простой алгоритм.

### Token bucket в Redis

```csharp
public sealed class TokenBucketLimiter(IConnectionMultiplexer redis)
{
    private const string LuaScript = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])  -- tokens per second
        local now = tonumber(ARGV[3])          -- unix timestamp ms
        local cost = tonumber(ARGV[4])

        local data = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(data[1]) or capacity
        local last_refill = tonumber(data[2]) or now

        local elapsed = (now - last_refill) / 1000
        tokens = math.min(capacity, tokens + elapsed * refill_rate)

        if tokens < cost then
            return {0, tokens}  -- denied, current tokens
        end

        tokens = tokens - cost
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)

        return {1, tokens}  -- allowed
    """;

    public async Task<bool> AllowAsync(string key, int capacity, double refillRate, int cost = 1)
    {
        var db = redis.GetDatabase();
        var result = (RedisResult[])await db.ScriptEvaluateAsync(
            LuaScript,
            keys: [$"rl:{key}"],
            values: [capacity, refillRate, DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(), cost]);

        return (int)result[0] == 1;
    }
}
```

Lua-скрипт **атомарен** — Redis выполняет его без interleaving. Без этого race condition между `HMGET` и `HMSET` пропустит burst при высоком QPS.

### Distributed deployment

| | Pros | Cons |
|--|------|------|
| In-memory per-instance | Latency 0ms | За LB у пользователя N counter'ов (один на инстанс) — лимит × N |
| Centralized Redis | Единый счётчик | +1ms на каждый вызов, SPoF (нужен Redis cluster) |
| Hybrid (sticky session + Redis fallback) | Быстро + точно | Сложнее реализация |

Для production — **Redis** с round-trip < 1ms (Redis в том же AZ).

### Built-in ASP.NET Core RateLimiter (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddTokenBucketLimiter("api", opt =>
    {
        opt.TokenLimit = 100;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 0;
        opt.ReplenishmentPeriod = TimeSpan.FromMinutes(1);
        opt.TokensPerPeriod = 100;
    });

    options.RejectionStatusCode = 429;
});

app.MapGet("/api/data", () => "...").RequireRateLimiting("api");
```

In-memory только. Для distributed нужно своё через Redis.

> [!question]- **Интервью: какие edge cases в rate limiting?**
> 1. **Clock skew** — на разных нодах разное время. Token bucket с timestamp ломается. Решение: использовать Redis-time (`TIME` command) как single source of truth.
> 2. **Hot keys** — если 99% запросов от одного user_id, Redis-инстанс с этим ключом перегружен. Решение: shard по user_id, или local cache + periodic Redis-sync.
> 3. **Synchronization после restart** — Redis потерял state (in-memory без AOF) → все пользователи получают full bucket. Решение: persistence (AOF) или принять reset как acceptable.
> 4. **Whitelisting** — internal сервисы не должны попадать под лимит. Скип по auth-claim.
> 5. **Different costs per endpoint** — `/api/cheap` стоит 1 token, `/api/expensive` — 10 tokens.

---

## Шаблон 2 — URL Shortener (bit.ly)

### Requirements
- Создать short URL для long URL (`bit.ly/xyz` → `https://very-long-url.com/...`)
- Redirect on visit
- Custom alias support
- Analytics (click count)
- 100M URLs, 100K creates/day, 100M reads/day

### Capacity
- Reads:writes = 1000:1 (read-heavy)
- 100K writes/day = ~1 QPS write, 1000 QPS read
- Storage: 100M URLs × ~500B/record = 50GB
- Length code: `[a-zA-Z0-9]{7}` = 62^7 = 3.5 трлн variants — хватит надолго

### High-level design

```
                                    ┌────────────┐
                                    │ Redis cache│ (LRU)
                                    │ shortcode →│
                                    │ longurl    │
                                    └─────▲──────┘
                                          │ on miss
                                          │
Browser ──▶ CDN ──▶ Load Balancer ──▶ App ──▶ Postgres
                                          │ (primary
                                          │  + read replicas)
                                          ▼
                                    ┌────────────┐
                                    │ Click events│
                                    │  → Kafka   │
                                    │  → Analytics│
                                    └────────────┘
```

### Schema

```sql
CREATE TABLE urls (
    code        text PRIMARY KEY,                     -- 7-char base62
    long_url    text NOT NULL,
    user_id     uuid,
    expires_at  timestamptz,
    created_at  timestamptz DEFAULT now()
);

-- Optional analytics
CREATE TABLE clicks (
    id          bigserial PRIMARY KEY,
    code        text NOT NULL,
    clicked_at  timestamptz DEFAULT now(),
    referer     text,
    country     text
) PARTITION BY RANGE (clicked_at);
```

### Code generation strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Hash(long_url)[:7]** — md5/sha | Determinic, идемпотентно | Collisions нужно обработать (retry с salt) |
| **Random base62** | Простота, быстро | Collisions (но в 7-char пространстве 62^7 — редко) |
| **Counter (auto-increment)** | No collisions | Sequential — guessable, можно сканить (security) |
| **Counter + base62** | No collisions, короткие коды | Same as counter — guessable |
| **Pre-generated batch** | Быстрые writes | Нужен generator service, доп. инфра |

**Production выбор:** random base62 with collision retry. Если коллизия (вероятность ~10^-12 при 100M existing) — генерируем новый.

```csharp
public async Task<string> ShortenAsync(string longUrl, string? customAlias = null, CancellationToken ct = default)
{
    if (customAlias is not null)
    {
        if (!IsValidAlias(customAlias)) throw new InvalidAliasException();
        var existing = await _db.Urls.FindAsync([customAlias], ct);
        if (existing is not null) throw new AliasAlreadyTakenException();
        return await SaveAsync(customAlias, longUrl, ct);
    }

    for (var attempt = 0; attempt < 5; attempt++)
    {
        var code = GenerateBase62(7);
        try
        {
            return await SaveAsync(code, longUrl, ct);
        }
        catch (DbUpdateException ex) when (IsDuplicateKey(ex))
        {
            continue;  // collision, retry
        }
    }
    throw new ShortenFailedException("Too many collisions");
}

private static string GenerateBase62(int length)
{
    const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    return RandomNumberGenerator.GetString(chars, length);
}
```

### Read path

```csharp
public async Task<string?> ResolveAsync(string code, CancellationToken ct)
{
    // 1. Cache (Redis, hot URLs)
    var cached = await _cache.GetAsync<string>($"url:{code}", ct);
    if (cached is not null)
    {
        _ = TrackClickFireAndForget(code);  // sample to Kafka
        return cached;
    }

    // 2. DB (read replica)
    var url = await _db.Urls.AsNoTracking().FirstOrDefaultAsync(u => u.Code == code, ct);
    if (url is null) return null;
    if (url.ExpiresAt < DateTime.UtcNow) return null;

    // 3. Populate cache
    await _cache.SetAsync($"url:{code}", url.LongUrl, TimeSpan.FromHours(1), ct);
    _ = TrackClickFireAndForget(code);

    return url.LongUrl;
}
```

**Click tracking** — fire-and-forget, не должен блокировать redirect (~5ms latency budget). Через Kafka producer с background flush, или Redis stream.

### Scaling decisions

| Bottleneck | Решение |
|-----------|---------|
| 1000 QPS read | Postgres read replicas (3-5 hot, async repl) |
| Hot codes | LRU cache в Redis, 99% hit rate |
| Click writes | Async via Kafka, batch insert в analytics DB |
| Geo latency | CDN с edge cache, регионы Postgres |
| Custom domains | nginx config / Cloudflare, нагрузка на App |

> [!question]- **Интервью: как scale до миллиарда URLs?**
> 1. **Sharding по code prefix** — `code[0]` → shard 0..63 (62 base62 chars). Каждый shard — 1.5B URL.
> 2. **CDN edge caching** — 99% reads попадают в edge, ноль нагрузки на origin.
> 3. **Read replicas** — несколько reader instances per shard.
> 4. **Кэш-only для рекордно-горячих URLs** — топ 1000 в Redis с infinite TTL.
> 5. **Analytics — отдельный pipeline** (Kafka → ClickHouse / BigQuery).

---

## Шаблон 3 — News Feed / Timeline (Twitter-like)

### Requirements
- 100M пользователей, 200 followees average
- 100M tweets/day = ~1000 QPS write
- Reads >> writes — 10K QPS feed reads
- Feed = chronological recent tweets от followees
- p99 < 500ms

### Two strategies

| Strategy | When | Pros | Cons |
|----------|------|------|------|
| **Pull (на запрос)** | Reads << Writes, sparse activity | Простой, мало storage | Slow at read time (fan-in) |
| **Push (fan-out на write)** | Reads >> Writes, активные пользователи | Fast reads | Storage explosion для celebs (1M followers × push) |
| **Hybrid** | Mix | Fast for normal users, manageable | Сложная имплементация |

**Production выбор:** Hybrid. Push для обычных юзеров (≤ 10K followers), pull для celebs.

### Push (fan-out)

```csharp
public async Task PostTweetAsync(Guid userId, string text, CancellationToken ct)
{
    var tweet = new Tweet { Id = Guid.NewGuid(), UserId = userId, Text = text, CreatedAt = DateTime.UtcNow };
    await _tweets.SaveAsync(tweet, ct);

    if (await _users.IsCelebrityAsync(userId, ct))
    {
        // Skip fan-out — followers будут pull'ить при чтении
        return;
    }

    var followers = await _users.GetFollowersAsync(userId, ct);
    foreach (var batch in followers.Chunk(1000))
    {
        // Async fan-out через очередь
        await _bus.Publish(new FanOutMessage(tweet.Id, batch.ToArray()), ct);
    }
}

// Consumer — пишет в timelines
public class FanOutConsumer : IConsumer<FanOutMessage>
{
    public async Task Consume(ConsumeContext<FanOutMessage> ctx)
    {
        var msg = ctx.Message;
        var tasks = msg.FollowerIds.Select(fid =>
            _redis.ListLeftPushAsync($"timeline:{fid}", msg.TweetId.ToString()));
        await Task.WhenAll(tasks);

        // Trim to last N tweets
        await Task.WhenAll(msg.FollowerIds.Select(fid =>
            _redis.ListTrimAsync($"timeline:{fid}", 0, 999)));
    }
}
```

### Read path (hybrid)

```csharp
public async Task<List<Tweet>> GetTimelineAsync(Guid userId, int limit, CancellationToken ct)
{
    // 1. Pre-computed timeline для regular followees
    var precomputedIds = await _redis.ListRangeAsync($"timeline:{userId}", 0, limit - 1);

    // 2. Pull recent от celebs (которые followed by user)
    var followedCelebs = await _users.GetFollowedCelebsAsync(userId, ct);
    var celebTweets = await Task.WhenAll(followedCelebs.Select(cid =>
        _tweets.GetRecentByUserAsync(cid, limit / 5, ct)));

    // 3. Merge by createdAt, take top N
    var allTweetIds = precomputedIds.Select(s => Guid.Parse(s!))
        .Concat(celebTweets.SelectMany(t => t.Select(x => x.Id)));

    var tweets = await _tweets.GetManyAsync(allTweetIds, ct);
    return tweets.OrderByDescending(t => t.CreatedAt).Take(limit).ToList();
}
```

### Tradeoffs

- **Storage**: timeline cache в Redis = 100M users × 1000 tweets × 16B (Guid) = 1.6TB только на список ID. Реально — ограничить cache до active users (последние 30 дней).
- **Hot keys**: celebrity timeline reads → один Redis-shard перегружен. Лечится через Redis cluster + replicas.
- **Eventual consistency**: новый tweet от celeb появляется в feed через секунды (не миллисекунды). Acceptable для UX.

---

## Шаблон 4 — Distributed cache

### When need
- Read traffic > 10K QPS
- DB не справляется или дорого scale'ить
- Read latency > 50ms критично

### Каркас

```
                    ┌─────────────────┐
                    │  Cache layer    │
                    │  (Redis cluster)│
                    └────────▲────────┘
                             │ miss
                             │
Client ──▶ App ──────────────┴──▶ DB
                             │
                             │ on miss: SET cache
                             ▼
                       Cache populated
```

### Patterns

| Pattern | Описание | Pros | Cons |
|---------|----------|------|------|
| **Cache-Aside (Lazy)** | App читает cache, на miss — DB → SET cache | Простой, robust | Stale data при concurrent updates |
| **Write-Through** | App пишет в cache → cache пишет в DB | Всегда synced | Slow writes (2 hops) |
| **Write-Back (Write-Behind)** | App пишет в cache → eventually flush в DB | Fast writes | Risk потерять данные при cache crash |
| **Read-Through** | Cache как proxy перед DB | App не знает про DB | Нужен cache-server с DB knowledge |

**Default — Cache-Aside.** Самый используемый.

### Cache invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

```csharp
// 1. TTL (самый простой)
await _cache.SetAsync(key, value, TimeSpan.FromMinutes(5), ct);

// 2. Explicit invalidation на write
public async Task UpdateUserAsync(User user, CancellationToken ct)
{
    await _db.SaveChangesAsync(ct);
    await _cache.RemoveAsync($"user:{user.Id}", ct);
}

// 3. Pub/sub для multi-instance invalidation
public async Task UpdateUserAsync(User user, CancellationToken ct)
{
    await _db.SaveChangesAsync(ct);
    await _redis.PublishAsync("invalidations", $"user:{user.Id}");
    // Все instances подписаны на канал → каждый invalidate свой local cache
}
```

### Hot key problem

Один key (например, `homepage:trending`) запрашивается 10K QPS — один Redis-инстанс перегружен.

**Решения:**

1. **Local in-memory cache** — каждое app-instance держит копию, refresh каждые 30 секунд из Redis. Latency 0, нагрузка на Redis = N instances.
2. **Sharding** — разделить hot key на N подключей: `homepage:trending:0`, `:1`, ... `:9`. Каждый запрос идёт на random shard, получает 1/10 данных, агрегируется на app-side.
3. **Replicated keys** — Redis Cluster с replicas, разные клиенты читают с разных replicas.

### Stampede / cache miss storm

Cache expired, 1000 параллельных запросов — все идут в DB. БД падает.

**Решения:**

```csharp
// 1. Probabilistic early refresh (refresh за X секунд до TTL с шансом)
public async Task<T?> GetOrComputeAsync<T>(string key, Func<Task<T>> compute, TimeSpan ttl, CancellationToken ct)
{
    var (value, expiresAt) = await _cache.GetWithExpiryAsync<T>(key, ct);
    if (value is null) return await ComputeAndCacheAsync(key, compute, ttl, ct);

    var remaining = expiresAt - DateTime.UtcNow;
    if (remaining < TimeSpan.FromSeconds(30))
    {
        // Random chance — один из N запросов делает refresh
        if (Random.Shared.NextDouble() < 0.1)
            _ = ComputeAndCacheAsync(key, compute, ttl, ct);  // background refresh
    }

    return value;
}

// 2. Distributed lock — только один запрос идёт в DB
public async Task<T> GetOrComputeWithLockAsync<T>(string key, Func<Task<T>> compute, TimeSpan ttl, CancellationToken ct)
{
    var cached = await _cache.GetAsync<T>(key, ct);
    if (cached is not null) return cached;

    using var redLock = await _lockFactory.CreateLockAsync($"lock:{key}", TimeSpan.FromSeconds(5), ct);
    if (redLock.IsAcquired)
    {
        // Double-check — кто-то мог уже compute'нуть пока мы ждали lock
        cached = await _cache.GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await compute();
        await _cache.SetAsync(key, value, ttl, ct);
        return value;
    }

    // Lock не получен — другой инстанс computes; ждём и читаем результат
    await Task.Delay(50, ct);
    return await GetOrComputeWithLockAsync(key, compute, ttl, ct);
}
```

---

## Шаблон 5 — Real-time chat / messaging

### Requirements
- 1M concurrent users
- Message delivery < 200ms p99
- Message order preserved per chat
- Offline users — push notifications, sync on reconnect

### Components

```
Client ──▶ WebSocket Gateway ──▶ Connection Service
                                       │ (which user on which gateway)
                                       │
                                       ▼
                                Message Service ──▶ Postgres (history)
                                       │
                                       ▼
                                Pub/Sub (Redis / Kafka)
                                       │
                          fan-out to recipient gateways
```

### Connection Service
Хранит mapping `user_id → gateway_instance`. На new connection — `SET user:123 → gateway:5` в Redis. Сообщения для user идут на тот gateway, гарантированно держащий открытый WebSocket.

### Message ordering

Per-chat ordering:
- Один partition в Kafka per chat (consistent hashing на chat_id)
- В Postgres — `(chat_id, sequence_no)` как PK, sequence_no через DB sequence per chat

```sql
CREATE TABLE messages (
    chat_id     uuid NOT NULL,
    seq_no      bigint NOT NULL,
    sender_id   uuid NOT NULL,
    content     text NOT NULL,
    created_at  timestamptz DEFAULT now(),
    PRIMARY KEY (chat_id, seq_no)
);

CREATE SEQUENCE chat_seq;  -- per-chat counter
-- Or use a separate counter table: SELECT seq_no + 1 FROM chat_counters WHERE chat_id = ?
```

### Delivery guarantees

| | Что | Сложность |
|--|-----|-----------|
| At-most-once | "Сообщение либо доставлено, либо нет" — допустимо для эфемерных (typing indicators) | Тривиально |
| At-least-once | Гарантия доставки, могут быть дубли | Retry до ack |
| Exactly-once **processing** | Доставка хотя бы раз + idempotent handler | Через message_id + dedup на клиенте |

### Sync on reconnect

```csharp
// Client сохраняет last_seq_no per chat
// На reconnect — запрашивает messages > last_seq_no
public async Task<List<Message>> SyncAsync(Guid chatId, long lastSeqNo, CancellationToken ct)
{
    return await _db.Messages
        .Where(m => m.ChatId == chatId && m.SeqNo > lastSeqNo)
        .OrderBy(m => m.SeqNo)
        .Take(1000)
        .ToListAsync(ct);
}
```

### Push notifications

User offline (нет WebSocket) → message доходит до Connection Service → нет mapping → отправляем push через APNS / FCM:
```csharp
if (!_connections.TryGetGateway(userId, out _))
{
    await _push.SendAsync(userId, new PushPayload(message), ct);
}
```

---

## Шаблон 6 — Distributed file storage (Dropbox-like)

### Requirements
- 100M users, 100GB на user → 10PB total
- Versioning, dedup
- Sync between devices

### Design

```
Client ──▶ API ──▶ Metadata Service ──▶ Postgres (file tree, versions)
                          │
                          ▼
                  Chunking Service (split file into 4MB chunks)
                          │
                          ▼
                  Object Storage (S3 / SeaweedFS / MinIO) — content-addressable
                          │
                  Hash(chunk) → object key
```

### Dedup через content-addressable storage

```csharp
public async Task<string> StoreChunkAsync(byte[] chunk, CancellationToken ct)
{
    var hash = Convert.ToHexString(SHA256.HashData(chunk));

    // Check if already stored
    if (await _storage.ExistsAsync($"chunks/{hash}", ct))
        return hash;  // dedup hit

    await _storage.PutAsync($"chunks/{hash}", chunk, ct);
    return hash;
}
```

Если 1000 пользователей загрузили одинаковую картинку — храним один раз. **Огромная экономия** (Dropbox экономит 30%+ через dedup).

### Sync algorithm

Client отслеживает локальные изменения, отправляет на server, получает diff обратно.

```csharp
// Simplified rsync-like
public async Task<SyncResponse> SyncAsync(Guid userId, ClientState state, CancellationToken ct)
{
    var serverState = await _meta.GetTreeAsync(userId, ct);

    var toUpload = state.LocalFiles.Except(serverState.Files);
    var toDownload = serverState.Files.Except(state.LocalFiles);
    var conflicts = state.LocalFiles.Intersect(serverState.Files)
        .Where(f => state.GetMTime(f) != serverState.GetMTime(f));

    return new SyncResponse(toUpload, toDownload, conflicts);
}
```

### Version control

Каждый upload — новая версия. Старые держим N дней / последние M версий:
```sql
CREATE TABLE file_versions (
    file_id     uuid NOT NULL,
    version     int NOT NULL,
    chunks      text[] NOT NULL,
    size        bigint NOT NULL,
    created_at  timestamptz DEFAULT now(),
    PRIMARY KEY (file_id, version)
);
```

---

## Шаблон 7 — Search engine

### Requirements
- 1M документов, full-text + filtering
- Latency < 100ms p99
- Relevance ranking

### Lego

```
Crawler / Ingestion ──▶ Tokenizer ──▶ Indexer ──▶ Inverted Index (Elasticsearch / OpenSearch)
                                                          ▲
                                                          │
Query ──▶ Query Parser ──▶ Search ◀───────────────────────┤
                              │
                              ▼
                          Ranking (BM25 + features)
                              │
                              ▼
                          Results
```

### Inverted index

```
Term → posting list (doc_id, frequency, positions)

"postgres" → [(doc1, 5, [10, 50, 100, 200]), (doc7, 2, [30, 90])]
"index"    → [(doc1, 3, [11, 51, 101]), (doc3, 1, [20])]

Query "postgres index" — find docs containing both → AND of posting lists
```

### BM25 ranking

```
score(doc, query) = Σ IDF(term) × TF_BM25(term, doc)
                    term ∈ query

IDF — log((N - n + 0.5) / (n + 0.5))   N = total docs, n = docs with term
TF_BM25 — saturates at high frequency, normalizes by document length
```

### Hybrid search (BM25 + vectors)

См. подробнее в [LLM/RAG patterns](). Ключ — fusion top-K через RRF (Reciprocal Rank Fusion).

### Когда что

| | Use case |
|--|----------|
| **Postgres FTS** (`tsvector`) | < 10M docs, простая релевантность |
| **Elasticsearch / OpenSearch** | 10M+ docs, сложные queries, аналитика |
| **Meilisearch** | Speed > breadth, simple ranking, typo tolerance |
| **Vector-only** (Qdrant, Pinecone) | Семантика > точность, RAG |
| **Hybrid (Postgres + pgvector)** | RAG, multi-tenant, единая инфра |

---

## Tradeoff анализ — что важнее на интервью

Не существует "правильной архитектуры". Есть **обоснованные tradeoffs**. На интервью обозначай явно:

### Consistency vs Availability
"Я выбираю eventual consistency для feed, потому что задержка 5 секунд приемлема, и нам важнее всегда показывать ленту даже при partition. Для платежей — strong consistency, выбираю CP."

### Latency vs Throughput
"Sync writes в DB дают 50ms latency, async через очередь — 2ms latency, но throughput на ingestion в 100x выше. Для analytics events выбираю async."

### Storage vs Compute
"Pre-computed feed в Redis — 10TB storage, но 1ms reads. On-demand fan-in — 0 storage, 200ms reads. Для 100M reads/day storage tradeoff stoит."

### Simple vs Optimal
"Самый простой работающий вариант — single Postgres + Redis. Достаточно до 10K QPS. Шардинг добавим когда упрёмся, не раньше."

---

## Cheatsheet — что и где использовать

| Компонент | Решение |
|-----------|---------|
| Web framework | ASP.NET Core (Minimal API + DI) |
| Реляционная DB | PostgreSQL |
| NoSQL document | MongoDB / Postgres + JSONB |
| Cache | Redis |
| Search | Elasticsearch / Postgres FTS / Meilisearch |
| Vector | Qdrant / pgvector |
| Message broker | RabbitMQ (low volume) / Kafka (high volume) |
| Object storage | S3 / SeaweedFS / MinIO |
| CDN | Cloudflare / Fastly |
| Load balancer | Nginx / HAProxy / cloud LB |
| Service mesh | Istio (если K8s) |
| Tracing | OpenTelemetry → Jaeger / Tempo |
| Metrics | Prometheus + Grafana |
| Logs | Loki / Elasticsearch / OpenSearch |
| Identity | OAuth2 / OIDC через Keycloak / IdentityServer / Auth0 |
| Background jobs | Hangfire / native BackgroundService / Quartz.NET |
| Workflow / Saga | MassTransit Saga / Temporal |

---

## См. также

- [Distributed Systems Patterns](distributed-systems.md) — Outbox, Idempotency, Saga
- [PostgreSQL Deep]() — RLS, JSONB, partitioning
- [LLM / RAG patterns]() — search, embeddings, hybrid
- [Caching]() — IMemoryCache, IDistributedCache
- [Resilience]() — retry, circuit breaker, timeout
- [Observability]() — OpenTelemetry, metrics, traces
- [Behavioral](../Meta/behavioral.md) — soft skills для tech lead

## Reading list

- **Designing Data-Intensive Applications** — Martin Kleppmann (must-read)
- **System Design Interview** — Alex Xu (Vol 1, Vol 2 — конкретные шаблоны)
- **Awesome System Design** — github.com/madd86/awesome-system-design (links к статьям)
- **Donne Martin — System Design Primer** — github.com/donnemartin/system-design-primer
- **High Scalability blog** — highscalability.com (case studies реальных архитектур)
- **AWS Architecture blog** — aws.amazon.com/blogs/architecture/
- **Engineering blogs** — Netflix Tech Blog, Uber Engineering, Airbnb Tech, Stripe Engineering
