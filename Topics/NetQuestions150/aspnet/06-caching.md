# Caching и Rate Limiting

## Output Caching

Кэширование HTTP-ответа целиком. UseOutputCaching(), CacheOutput() на endpoint. Политики: VaryByQuery, VaryByHeader, Tags. Инвалидация: EvictByTagAsync по тегу.

---

## IMemoryCache vs IDistributedCache

| Аспект | IMemoryCache | IDistributedCache |
|--------|--------------|-------------------|
| Хранение | In-memory, один сервер | Redis, SQL Server |
| Масштаб | Нет | Общий для instances |
| Сериализация | Объекты как есть | byte[] |

**HybridCache** (.NET 8) — L1 (memory) + L2 (distributed). Низкая латентность при L1 hit, консистентность через L2.

---

## Паттерны кэширования

Cache-Aside (приложение управляет), Read-Through, Write-Through, Write-Behind, Stampede prevention.

---

## Rate Limiting

Защита от DDoS, fair use. Типы: FixedWindow, SlidingWindow, TokenBucket, Concurrency. AddRateLimiter(), UseRateLimiter().
