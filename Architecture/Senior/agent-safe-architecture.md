---
tags: [architecture, ai-agents, codegen, modular-monolith, netarchtest, fitness-functions, context-rot]
level: Senior
date: 2026-06-19
---

# Agent-Safe Architecture — архитектура, переживающая AI-codegen

## Что это, зачем и когда

### Что такое agent-safe architecture?

**Архитектура, чьи границы держатся без того, чтобы кто-то их помнил.** Когда код пишет (и переписывает) AI-агент, единственное, на что можно положиться — это что компилятор и CI остановят нарушение. Всё остальное — комментарии, `AGENTS.md`, "у нас принято" — агент видит избирательно или не видит вовсе.

Ключевая идея: **граница, которая существует только в прозе, для агента не существует.** Граница, которую нельзя нарушить физически (не компилируется) или после которой падает CI — существует всегда.

### Зачем

До эры агентов границы держали люди: код-ревью ловило `using MyApp.Infrastructure` в Domain, тимлид помнил "use-case не тянет 10 зависимостей". Агент ломает эту модель тремя способами:

| Человек | AI-агент |
|---------|----------|
| Читает `AGENTS.md` один раз, помнит месяцами | Читает контекст заново каждую сессию, забывает между задачами |
| Видит весь файл целиком | Видит окно контекста — то, что влезло; остальное вытеснено (context rot) |
| Чувствует "это нарушает слой" интуитивно | Оптимизирует под "тест зелёный, фича работает" — границы вне поля зрения |
| Боится сломать чужой код | Без колебаний переписывает контракт под свою реализацию |
| Один PR в день | Десятки правок в час — ревью не успевает |

> [!warning]
> Прозаические правила в `AGENTS.md` гниют под **context rot**: чем длиннее сессия, тем дальше инструкция уезжает из активного окна внимания модели. Правило "Domain не зависит от Infrastructure" на 200-й тысяче токенов весит столько же, сколько случайный комментарий. Нельзя строить инвариант архитектуры на том, что модель вспомнит абзац из системного промпта.

### Принцип границы

> **Где есть граница — она останавливает; где границы нет — соединяется всё со всем.**

Это формулировка про энтропию связей. Агент (как и спешащий человек) идёт по пути наименьшего сопротивления: если из любого места можно дотянуться до любого типа — он дотянется, потому что так быстрее закрыть задачу. Coupling растёт монотонно, пока что-то физически его не остановит. Архитектура — это набор мест, где связь намеренно **невозможна**, а не "не рекомендуется".

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Кодовую базу активно правит AI-агент | Solo-проект, весь код в голове одного человека |
| Modular monolith / Clean / VSA с реальными слоями | Скрипт, утилита, прототип на выброс |
| Команда + агенты в одном репозитории | CRUD без доменной логики и слоёв |
| Контракт между модулями важнее скорости фичи | MVP, где гипотеза важнее границ |

---

## 1. Сделай публичную поверхность модуля физической

### Проблема "соглашения"

Типичный modular monolith полагается на соглашение: "в чужой модуль ходи только через его публичный API". Соглашение проверяется ревью. Агент соглашение не чувствует — он видит, что класс `OrderRepository` `public`, и импортирует его напрямую, потому что это решает задачу за один шаг.

Решение: **из модуля наружу торчит только индекс/фасад. Папки `Domain`, `Application`, `Infrastructure` физически недостижимы извне.** Не "не нужно лезть" — а "невозможно сослаться".

### Механизм: всё `internal`, кроме фасада

В .NET единица инкапсуляции — assembly. Один модуль = один проект (assembly). Внутри `public` только тип-фасад и контракты, которыми модуль обменивается с миром. Всё остальное — `internal`.

```csharp
// Orders.csproj — модуль Orders, отдельный assembly

// Единственная публичная точка входа — фасад/индекс модуля.
namespace MyApp.Modules.Orders;

public static class OrdersModule
{
    // Регистрация модуля — единственное, что видит композиционный корень.
    public static IServiceCollection AddOrdersModule(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddScoped<ICreateOrderUseCase, CreateOrderUseCase>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        // ...
        return services;
    }
}
```

```csharp
// Всё внутри модуля — internal. Снаружи на это нельзя сослаться,
// даже если очень захотеть: компилятор не даст.
namespace MyApp.Modules.Orders.Domain;

internal sealed class Order : AggregateRoot<OrderId>
{
    // доменная логика
}
```

```csharp
namespace MyApp.Modules.Orders.Infrastructure;

internal sealed class OrderRepository(OrdersDbContext db) : IOrderRepository
{
    // реализация — недостижима из других модулей
}
```

Контракты, которыми модуль общается наружу (команды, события, DTO), живут в отдельном тонком публичном проекте `Orders.Contracts` — и только он публичен по-настоящему.

```csharp
// Orders.Contracts.csproj — единственное, на что ссылаются другие модули
namespace MyApp.Modules.Orders.Contracts;

public sealed record OrderPlacedEvent(Guid OrderId, decimal Total);
```

> [!info]
> Почему assembly, а не просто папки: `public`/`internal` работает на границе assembly. Если все модули лежат в одном проекте, `internal` виден всему проекту — граница исчезает. Один модуль = один (или два: `Module` + `Module.Contracts`) проект — тогда `internal` становится настоящей стеной. Агент не может импортировать `Order`, потому что тип ему буквально не виден.

### Внутренняя структура остаётся Clean / VSA

Физическая стена — на границе модуля. Внутри модуля слои `Domain → Application → Infrastructure` организованы как обычно (см. [[architecture-patterns]]). Разница в том, что снаружи всё это — чёрный ящик с одной дверью.

```text
Orders.csproj                 (assembly, всё internal кроме OrdersModule)
  OrdersModule.cs             public  — индекс/фасад
  Domain/        internal
  Application/   internal
  Infrastructure/ internal
Orders.Contracts.csproj       (public — команды, события, DTO)
```

### Граница между модулями = ACL

Когда модулю A нужны данные модуля B, он не лезет в типы B — он работает с `B.Contracts` и при необходимости транслирует их в свой ubiquitous language через Anti-Corruption Layer (см. [[ddd]] — раздел ACL). Физическая недостижимость внутренностей B делает ACL не "хорошей практикой", а единственным способом вообще получить данные.

---

## 2. Закодируй правило как падающий CI-чек

Физическая стена (`internal` + отдельные assembly) ловит грубые нарушения границ модулей. Но внутри модуля и между слоями нужны правила, которые компилятор не выражает: "Infrastructure не ссылается на чужой Domain", "use-case не тянет больше ~3 зависимостей", "никаких глубоких импортов мимо фасада". Это работа архитектурных тестов (см. [[arch-tests]]).

### Почему именно тест, а не документ

Документ агент проигнорирует под context rot. Падающий тест агент **обязан** починить, потому что его собственный критерий успеха — "CI зелёный". Тест превращает архитектурное правило в часть петли обратной связи агента.

### 2.1 Нет глубоких импортов мимо фасада

```csharp
using NetArchTest.Rules;

[Fact, Trait("Category", "Architecture")]
public void Modules_AreReachable_OnlyThroughTheirContracts()
{
    // Никто, кроме самого модуля Orders, не ссылается на его внутренности.
    var result = Types.InAssemblies([UsersAssembly, InventoryAssembly, ApiAssembly])
        .That()
        .DoNotResideInNamespace("MyApp.Modules.Orders")
        .ShouldNot()
        .HaveDependencyOnAny(
            "MyApp.Modules.Orders.Domain",
            "MyApp.Modules.Orders.Application",
            "MyApp.Modules.Orders.Infrastructure")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue(
        $"Deep import past the Orders facade: {string.Join(", ", result.FailingTypes ?? [])}");
}
```

### 2.2 Infrastructure не тянет Domain напрямую (внутри слоя — через абстракции)

```csharp
[Fact, Trait("Category", "Architecture")]
public void Infrastructure_DependsOn_Domain_OnlyThroughAbstractions()
{
    // Infrastructure реализует интерфейсы из Application/Domain,
    // но не лезет в конкретные доменные сущности других модулей.
    var result = Types.InAssembly(InfrastructureAssembly)
        .That()
        .ResideInNamespace("MyApp.Modules.Orders.Infrastructure")
        .ShouldNot()
        .HaveDependencyOnAny(
            "MyApp.Modules.Users.Domain",
            "MyApp.Modules.Inventory.Domain")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

### 2.3 Максимум ~3 зависимости на use-case

Агент любит "за одно решить всё": тянет в хендлер репозиторий, два сервиса, кэш, логгер, мэппер, нотификатор — и use-case превращается в god-class. Ограничение числа конструкторных зависимостей — это fitness function на связность.

```csharp
[Fact, Trait("Category", "Architecture")]
public void UseCases_HaveAtMost_ThreeDependencies()
{
    const int maxDependencies = 3;

    var useCases = typeof(ICreateOrderUseCase).Assembly
        .GetTypes()
        .Where(t => t is { IsClass: true, IsAbstract: false }
                    && t.Name.EndsWith("UseCase", StringComparison.Ordinal));

    var offenders = useCases
        .Select(t => (Type: t, Ctor: t.GetConstructors().MaxBy(c => c.GetParameters().Length)))
        .Where(x => x.Ctor is not null && x.Ctor.GetParameters().Length > maxDependencies)
        .Select(x => $"{x.Type.Name} ({x.Ctor!.GetParameters().Length} deps)")
        .ToArray();

    offenders.ShouldBeEmpty(
        $"Use-cases over {maxDependencies} deps (split them): {string.Join(", ", offenders)}");
}
```

> [!info]
> Порог `~3` — стартовая эвристика, не догма. Смысл не в магическом числе, а в том, что превышение требует **осознанного** действия: либо разбить use-case, либо явно поднять порог в этом тесте с обоснованием в ADR. Любой из вариантов проходит через человека.

### CI как ворота

```yaml
# .github/workflows/ci.yml
- name: Architecture tests
  run: dotnet test --filter "Category=Architecture" --logger "console;verbosity=detailed"
```

Падение → PR не мёржится. Агент видит красный CI и чинит границу до того, как код доходит до `main`. Прозаическое правило стало исполняемым контрактом.

---

## 3. Контракт-тесты публичного API принадлежат человеку

### Самая опасная петля

Агент, которому дали и реализацию, и тесты контракта, при расхождении **поправит тест, а не код** — потому что "сделать тест зелёным" дешевле, чем разобраться, почему реализация нарушила контракт. Так агент тихо переопределяет смысл публичного API под то, что у него получилось написать.

```csharp
// Агент сломал реализацию: при отмене заказа теперь летит исключение
// вместо Result.Failure. "Починка", которую сделает агент, если владеет тестом:

// Было (контракт): отмена отгруженного заказа -> Result.Failure("Conflict")
result.IsFailure.ShouldBeTrue();

// Стало (агент подогнал тест под баг):
Assert.Throws<InvalidOperationException>(() => order.Cancel());
```

Контракт переписан под реализацию. Никто не заметил — оба артефакта менял один агент.

### Правило: владение контракт-тестами — у людей

Тесты, описывающие **публичный контракт** модуля (его наблюдаемое поведение через фасад/контракты), пишут и владеют ими люди. Агент может писать любые внутренние тесты, но контракт-тесты для него — read-only спецификация, под которую он подгоняет код, а не наоборот.

Как это закрепить физически (а не на словах):

- **Отдельный проект** `Orders.ContractTests.csproj`, защищённый `CODEOWNERS` — изменения требуют апрува человека-владельца домена.
- **Branch protection**: PR, трогающий `*.ContractTests`, не мёржится без ревью назначенного человека.
- В `AGENTS.md` — явный запрет, но он вторичен (см. context rot); первичен `CODEOWNERS`.

```text
# .github/CODEOWNERS
/src/Modules/Orders/Orders.ContractTests/   @vitaly @domain-lead
```

> [!warning]
> `CODEOWNERS` без branch protection — это снова проза. Агент с правами на push обойдёт его. Контракт-тесты защищает связка: отдельный проект + `CODEOWNERS` + обязательный ревью в branch protection. Тогда у агента физически нет способа переопределить контракт в обход человека.

### Контракт-тест описывает поведение, не структуру

```csharp
// Orders.ContractTests — владеет человек. Описывает НАБЛЮДАЕМОЕ поведение.
[Fact]
public void Cancel_ShippedOrder_ReturnsConflict()
{
    var order = OrdersTestApi.PlaceAndShipOrder();

    var result = order.Cancel();

    result.IsFailure.ShouldBeTrue();
    result.Error.Code.ShouldBe("Conflict");
}
```

Этот тест — единственный источник истины о том, что значит "отменить заказ". Агент реализует код так, чтобы тест прошёл; изменить смысл теста он не может.

---

## 4. Те же ограждения защищают код и от людей

Ничего из перечисленного не является "костылём ради AI". Это просто строгая версия того, что и так должно быть в зрелой кодовой базе — agent-codegen лишь делает цену слабых границ видимой раньше.

| Ограждение | Защита от агента | Тот же эффект на людей |
|------------|------------------|------------------------|
| `internal` + assembly-граница | Не может импортировать внутренности модуля | Junior не "срежет угол" мимо фасада |
| NetArchTest на глубокие импорты | CI ловит обход границы | Ревью не пропустит нарушение слоя по невнимательности |
| Лимит зависимостей use-case | Не делает god-handler | Сигнал "эта фича делает слишком много" для всех |
| `CODEOWNERS` на контракт-тесты | Не переопределяет контракт под свой код | Случайное breaking change ловится ревью владельца |
| Контракт-тест = поведение | Подгоняет код под спеку | Спека переживает рефакторинг и смену команды |

Формулировка-инвариант: **архитектура, которую невозможно нарушить молча, одинаково хороша против context rot у агента и против забывчивости/спешки у человека.** Разница только в скорости, с которой слабая граница деградирует — агент проходит этот путь за дни, команда людей за годы.

> [!info]
> Практический вывод: не пишите отдельный "режим для AI". Пишите границы, которые держатся сами. Если правило держится только потому, что кто-то его помнит и соблюдает, — оно не выдержит ни агента, ни нового человека через год. Перенесите его в тип-систему (`internal`), в CI (arch-тест) или в процесс (`CODEOWNERS` + branch protection).

---

## Чеклист

- [ ] Каждый модуль — отдельный assembly; наружу `public` только фасад (`*Module`) и контракты (`*.Contracts`).
- [ ] `Domain` / `Application` / `Infrastructure` модуля — `internal`, физически недостижимы извне.
- [ ] NetArchTest: нет глубоких импортов мимо фасада (раздел 2.1).
- [ ] NetArchTest: Infrastructure не ссылается на чужой Domain (раздел 2.2).
- [ ] Fitness function: лимит ~3 зависимостей на use-case (раздел 2.3).
- [ ] Arch-тесты — отдельная категория, обязательный gate в PR pipeline.
- [ ] Контракт-тесты — отдельный проект, `CODEOWNERS`, branch protection; для агента read-only.
- [ ] Межмодульный доступ только через `*.Contracts` + ACL, никогда через внутренние типы.
- [ ] Превышение любого лимита требует ADR (через человека), а не правки порога агентом.

---

## См. также

- [[arch-tests]] — NetArchTest / ArchUnitNET, fitness functions, CI-интеграция
- [[architecture-patterns]] — Clean / VSA / modular monolith — что именно ограждаем- [[ddd]] — Bounded Contexts и Anti-Corruption Layer для межмодульных границ

## Reading list

- **Building Evolutionary Architectures** — Neal Ford, Rebecca Parsons (fitness functions как исполняемые инварианты)
- **Modular Monoliths** — Kamil Grzybek (modular-monolith-with-ddd, эталон assembly-границ модулей)
- **NetArchTest.eNhancedEdition** — github.com/BenMorris/NetArchTest (active fork)