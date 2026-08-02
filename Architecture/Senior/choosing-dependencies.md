---
tags: [dependencies, licensing, supply-chain, architecture, senior]
level: Senior
date: 2026-08-02
---

# Choosing Dependencies — как выбирать зависимости в 2026: лицензионный разлом .NET OSS

> С 2023 по 2026 коммерциализировалась половина «дефолтного» стека .NET-бэкендера: MediatR, AutoMapper, MassTransit, FluentAssertions, NBomber, EFCore.BulkExtensions. Это не серия случайностей, а структурный сдвиг экономики OSS. Вывод для архитектора: **лицензия зависимости — такой же архитектурный риск, как её производительность или bus factor**, и управлять им нужно теми же инструментами — ADR, тонкие адаптеры, CI-гейты.

---

## 1. Что случилось с .NET OSS 2023-2026

### 1.1. Хронология волны

| Когда | Событие |
|-------|---------|
| 2021 | **Duende IdentityServer** — первый громкий прецедент: v5 коммерческая. IdentityServer4 остался бесплатным, но поддержка кончилась в ноябре 2022 |
| Август 2023 | **Moq / SponsorLink** — в популярнейший mocking-пакет тихо добавили closed-source компонент, собиравший хеши email разработчиков на этапе сборки. Не про деньги, а про доверие: сообщество массово ушло на NSubstitute |
| Январь 2023 | **EFCore.BulkExtensions** — dual license (cFOSS): free только personal / non-profit / выручка < $1M |
| Май 2024 | **NBomber v5** — closed source, «лицензия 2.0»: personal free, организациям платно. v4 остаётся Apache 2.0 |
| 31.12.2024 | **SpecFlow — EOL.** Tricentis не продал и не передал проект — репозитории удалены. Преемник — Reqnroll, форк от создателя SpecFlow |
| Январь 2025 | **FluentAssertions 8** — коммерциализация через партнёрство с Xceed, платная для коммерческого использования. 7.x остаётся Apache 2.0 |
| 2 апреля 2025 | Jimmy Bogard анонсирует коммерциализацию MediatR и AutoMapper (бренд **Lucky Penny Software** появился к запуску 2 июля 2025) |
| 2 июля 2025 | Коммерческий запуск: **MediatR 13+ / AutoMapper 15+** — dual license (RPL-1.5 или commercial). MediatR ≤12.x и AutoMapper ≤14.x — на прежних свободных лицензиях навсегда |
| Q1 2026 | **MassTransit v9** (компания Massient) — коммерческий: ~$400/мес SMB, ~$1200/мес enterprise. v8 остаётся Apache 2.0, но только security-патчи, EOL конец 2026 |

### 1.2. Механизм: почему волна, а не случайности

Экономика соло-мейнтейнера топового пакета сломана с трёх сторон:

1. **Асимметрия ценности.** Пакет с сотнями миллионов загрузок экономит индустрии тысячи человеко-лет, а мейнтейнеру приносит донаты уровня «на кофе». Спонсорство как модель не сработало нигде — ни у Bogard, ни у Moq (SponsorLink был именно попыткой принудить к спонсорству).
2. **Нет институционального якоря.** В Java/CNCF-мире зрелый проект уходит в foundation (Apache, Eclipse, CNCF) и живёт на корпоративные взносы. В .NET-экосистеме сопоставимого механизма нет — .NET Foundation после кризиса доверия 2021 года эту роль не вытянул. Единственный устойчивый исход для автора — компания вокруг проекта.
3. **Прецедент снял табу.** Duende показал, что коммерциализация не убивает проект. После этого каждый следующий переход давался легче: NServiceBus был платным всегда, Duende с 2021, дальше — по хронологии выше.

Следствие для тебя: **любая зависимость с одним владельцем и без foundation — кандидат на смену модели.** Это не осуждение авторов (модель dual-license с бесплатным community-тиром — честная), это входное условие планирования.

### 1.3. Почему это архитектурный риск, а не «проблема юристов»

- Лицензия меняется **односторонне и только для новых версий** — старые релизы неотзывны, но и не поддерживаются вечно. Ты оказываешься перед выбором: платить, мигрировать или копить security-долг.
- Стоимость выхода пропорциональна **глубине проникновения API в код**: `IRequest`-интерфейсы MediatR живут в каждом handler'е, asserts FluentAssertions — в каждом тесте. Решение «взять пакет» дешёвое, решение «убрать пакет» — нет.
- NuGet **не сломает билд** при смене лицензии пакета. Нет механизма license-gate в restore: если сам не проверяешь — узнаешь из новостей.

> [!question]- Интервью: почему смену лицензии зависимости называют архитектурным риском, а не юридическим?
> Потому что юридическая часть тривиальна (читаешь условия), а последствия — архитектурные: стоимость выхода определяется глубиной coupling'а с API пакета, доступностью альтернатив и наличием тонкого адаптера. Управляется это архитектурными средствами: ADR при добавлении зависимости, изоляция за собственным интерфейсом, exit-план до, а не после смены лицензии.

---

## 2. Карта: что стало платным

### 2.1. Коммерциализированные пакеты

| Библиотека | Платно с | Условия free | Последняя свободная версия | Альтернатива |
|-----------|----------|--------------|---------------------------|--------------|
| **MediatR** | 13+ (июль 2025, Lucky Penny) | Community edition: выручка < $5M **и** привлечённый капитал < $10M; платные тиры по размеру команды (не per-developer); либо RPL-1.5 | ≤12.x — свободная навсегда | Свой dispatcher / Mediator (source-gen) / Wolverine |
| **AutoMapper** | 15+ (июль 2025, Lucky Penny) | Те же условия, что MediatR | ≤14.x — свободная навсегда | Mapperly / ручной маппинг |
| **MassTransit** | v9 (Q1 2026, Massient) | 100%-discount тир для организаций с выручкой < $1M/год; иначе ~$400/мес SMB, ~$1200/мес enterprise | v8 — Apache 2.0, но только security-патчи, EOL конец 2026 | Wolverine / Rebus / Brighter / raw-клиенты брокеров |
| **FluentAssertions** | 8+ (январь 2025, Xceed) | Free для non-commercial | 7.x — Apache 2.0 | Shouldly / AwesomeAssertions (FA-совместимый API, миграция ~0) |
| **NBomber** | v5 (май 2024) | Personal free; организациям платно | v4 — Apache 2.0 | k6 (Grafana) для серьёзного load testing |
| **EFCore.BulkExtensions** | Январь 2023 (cFOSS) | Personal / non-profit / выручка < $1M | MIT-форк `EFCore.BulkExtensions.MIT` на NuGet | Нативный `ExecuteUpdate`/`ExecuteDelete`, PG `COPY`; MIT-форк |
| **Duende IdentityServer** | v5 (2021) | Условия — на сайте Duende; IdentityServer4 без поддержки с ноября 2022 | IdentityServer4 (EOL — не использовать) | OpenIddict / Keycloak; Duende — если нужна поддержка |
| **NServiceBus** | Всегда был коммерческим | Trial / non-production | — | Те же, что у MassTransit |

### 2.2. Смежные случаи (не «платно», но меняет выбор)

| Библиотека | Что случилось | Дефолт теперь |
|-----------|---------------|---------------|
| **Moq** | SponsorLink-инцидент (август 2023): закрытый код в build-пайплайне, сбор данных разработчиков | Новые проекты — NSubstitute |
| **SpecFlow** | EOL 31.12.2024, репозитории удалены Tricentis | Reqnroll (форк создателя SpecFlow) |

---

## 3. Framework выбора зависимости

Чек-лист **перед** добавлением любого пакета. Порядок неслучаен: дешёвые проверки первыми.

### 3.1. Лицензия и её история

Не «какая лицензия сейчас», а «какова динамика»:

- Текущая лицензия + история изменений (git log по файлу LICENSE говорит больше, чем README).
- **Кто владеет копирайтом.** Проект, где автор владеет подавляющей частью кода (или собирает CLA с контрибьюторов), может перелицензировать новые версии в любой момент. Уже выпущенный код неотзывен — но это утешение на год-два, не навсегда.
- Красные флаги: BUSL, «fair source», RPL, custom-лицензии, «free for non-commercial», отдельный LICENSE-файл на «premium»-фичи внутри репо.

### 3.2. Bus factor и governance

- Один мейнтейнер = один инфаркт, один burnout или одна коммерциализация до остановки проекта.
- Foundation-backed (Apache, CNCF, .NET Foundation с оговорками) > компания > соло-автор — по предсказуемости, не по качеству кода.
- Скорость реакции на issues/CVE за последний год — лучший прокси живости, чем звёзды.

### 3.3. Verified publisher и существование пакета

Supply-chain дисциплина (правило этого vault — проверка **до** добавления, не после):

- NuGet: verified publisher, reserved prefix, ссылка на репозиторий, Source Link, подписи.
- Download count и возраст пакета — против typosquatting'а (`Newtonsoft.Json` vs `Newtonsofts.Json`).
- Помнить урок SponsorLink: пакет исполняет код в твоём билде. MSBuild tasks и analyzers из пакета — это исполнение чужого кода на CI с твоими секретами в environment.

### 3.4. Reflection vs source-gen — AOT-готовность

- Reflection-based (AutoMapper, MediatR ≤12 с assembly scanning, Castle DynamicProxy под Moq/NSubstitute) ломается или деградирует под trimming и Native AOT.
- Source-gen (Mapperly, Mediator, `System.Text.Json` source generation) — ошибки в compile-time, ноль runtime-магии, AOT-совместимость.
- Даже если AOT не нужен сегодня — source-gen-пакет проще дебажить: сгенерированный код читается глазами.

### 3.5. Стоимость замены (exit cost)

Ключевой вопрос: **сколько файлов придётся тронуть, чтобы выкинуть пакет?**

- FluentAssertions — в каждом тест-файле, но замена механическая (AwesomeAssertions — drop-in, Shouldly — regex-заменой).
- MediatR — интерфейсы в каждом handler'е, но семантика воспроизводится своими типами за час.
- MassTransit — консьюмеры, саги, топология брокера, retry-политики: выход дорогой. Значит, выбор bus-абстракции — решение уровня ADR, а не «добавил пакет в csproj».
- Правило: **чем шире API-поверхность пакета в твоём коде, тем выше требования к его лицензионной стабильности.** Широкий API + соло-автор = заворачивай в тонкий адаптер сразу.

### 3.6. Lock-файлы и NuGetAudit в CI

Механика, без которой всё выше — благие пожелания:

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  <NuGetAudit>true</NuGetAudit>
  <NuGetAuditMode>all</NuGetAuditMode>   <!-- включая транзитивные -->
  <NuGetAuditLevel>low</NuGetAuditLevel>
</PropertyGroup>
```

```bash
# CI: restore строго по lock-файлу — подмена версии ломает билд
dotnet restore --locked-mode
```

- Lock-файлы фиксируют дерево зависимостей: без них `floating version` или подменённый транзитивный пакет проезжает в прод незамеченным.
- NuGetAudit ловит известные CVE (NU1901-NU1904), `Mode=all` — включая транзитивные.
- Лицензии NuGetAudit **не проверяет** — для license-инвентаризации гонять отдельный шаг (например, `nuget-license` / `dotnet-project-licenses`) и складывать отчёт в артефакты CI.
- Central Package Management (`Directory.Packages.props`) — версии в одном месте, апгрейд осознанный, а не «у каждого проекта своя».

> [!question]- Интервью: как защитить проект от внезапной смены лицензии зависимости?
> Четыре слоя: (1) lock-файлы + `--locked-mode` — новая версия не приедет молча; (2) license-скан в CI — смена лицензии видна как diff отчёта; (3) тонкий адаптер вокруг пакетов с широким API — exit cost контролируем; (4) ADR при добавлении: зафиксированы условия, альтернативы и триггер пересмотра. Тогда «MediatR стал платным» — плановая миграция, а не пожар.

---

## 4. Линия замен этого vault

Позиция vault — «No MediatR» ещё до коммерциализации: стек Vertical Slices + `Result<T>` не нуждается в medium-библиотеке для вызова метода в том же процессе. Лицензионный разлом эту позицию только укрепил.

### 4.1. Дефолт: прямые handler-вызовы или свой dispatcher

Для 90% приложений mediator-библиотека не нужна вовсе — endpoint вызывает handler напрямую через DI:

```csharp
app.MapPost("/orders", async (
    CreateOrderCommand command,
    CreateOrderHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(command, ct);
    return result.ToHttpResult(); // Result -> TypedResults
});
```

Ноль зависимостей, ноль runtime-магии, навигация «go to definition» работает. Cross-cutting (validation, logging) — endpoint filters или decorators через Scrutor.

Если нужен единый seam для cross-cutting поверх всех handler'ов — свой dispatcher, ~50 строк, тот же приём, что внутри MediatR:

```csharp
public interface ICommand<TResponse>;

public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<Result<TResponse>> HandleAsync(TCommand command, CancellationToken ct);
}

public sealed class Dispatcher(IServiceProvider services)
{
    private readonly ConcurrentDictionary<Type, object> _wrappers = new();

    public Task<Result<TResponse>> SendAsync<TResponse>(
        ICommand<TResponse> command, CancellationToken ct)
    {
        var wrapper = (HandlerWrapper<TResponse>)_wrappers.GetOrAdd(
            command.GetType(),
            static type => Activator.CreateInstance(
                typeof(HandlerWrapper<,>).MakeGenericType(type, typeof(TResponse)))!);
        return wrapper.Handle(command, services, ct);
    }

    private abstract class HandlerWrapper<TResponse>
    {
        public abstract Task<Result<TResponse>> Handle(
            ICommand<TResponse> command, IServiceProvider sp, CancellationToken ct);
    }

    private sealed class HandlerWrapper<TCommand, TResponse> : HandlerWrapper<TResponse>
        where TCommand : ICommand<TResponse>
    {
        public override Task<Result<TResponse>> Handle(
            ICommand<TResponse> command, IServiceProvider sp, CancellationToken ct)
            => sp.GetRequiredService<ICommandHandler<TCommand, TResponse>>()
                 .HandleAsync((TCommand)command, ct);
    }
}
```

Это весь «mediator»: словарь кешированных wrapper'ов + resolve из DI. Никакой лицензии, никакого assembly scanning.

### 4.2. Когда дефолта мало

| Инструмент | Когда брать | Почему именно он |
|-----------|-------------|------------------|
| **Wolverine** | Нужен durable messaging / saga / outbox / брокер одним инструментом | MIT, ядро OSS (платный только аддон CritterWatch), source-gen runtime, in-process + брокеры единой моделью |
| **Mediator** (martinothamar, source-gen) | Хочется сохранить `IRequest`-семантику MediatR как drop-in | MIT, source generator (AOT-ready), API почти 1:1 с MediatR — миграция минимальная |
| **Mapperly** | Маппинг, который надоело писать руками | Apache 2.0, source-gen: маппинг виден как обычный C#-код, ошибки в compile-time, AOT без оговорок |
| **Ручной маппинг** | DTO ≤ ~10 полей или маппинг с логикой | Ноль зависимостей; extension-метод `ToDto()` читается быстрее любой конфигурации |
| **Shouldly** | Asserts во всех новых тестах | BSD, стабильный API, сообщения об ошибках информативнее голого xUnit Assert |
| **AwesomeAssertions** | Легаси-сьют на FluentAssertions ≤7, переписывать дорого | Apache 2.0 форк с FA-совместимым API — миграция ~0 (сменить пакет и using) |

Обоснование линии целиком: сначала **ноль зависимостей** (прямой вызов, ручной маппинг), затем **source-gen OSS с ясной governance** (Mapperly, Mediator, Wolverine), и только затем — коммерческое, если оно покупает реальную ценность (например, Duende ради поддержки и сертификации).

---

## 5. Common Pitfalls

### 5.1. «Форк — и проблема решена»

Форк наследует код, но не мейнтейнеров. Через год у мёртвого форка нет ни security-патчей, ни совместимости с новым рантаймом — это хуже, чем честно платить. Выживают форки двух типов: с **узкой целью** (AwesomeAssertions — заморозить совместимый API, не развивать фреймворк) и с **институциональной тягой** (Reqnroll — создатель SpecFlow + сообщество, которому некуда отступать). Прежде чем ставить форк, смотри: кто мейнтейнер, какой у него мотив и что в форке происходило последние 6 месяцев.

### 5.2. Pin старой свободной версии = security-долг

«MediatR 12 свободен навсегда» ≠ «MediatR 12 поддерживается навсегда». Старая версия не получает багфиксов под новые версии рантайма, а окно security-патчей конечно (MassTransit v8 — только до конца 2026). Pin — легитимная **тактика** на 12-24 месяца с явной датой пересмотра в ADR; как **стратегия** — это тихое накопление CVE в проде.

### 5.3. RPL-копилефт заразен сильнее GPL

Видишь «MediatR 13 доступен под RPL-1.5, значит бесплатен» — стоп. GPL-обязательства срабатывают при **распространении** (SaaS — лазейка, которую закрывает AGPL). RPL идёт дальше: триггер — **deployment**, включая внутреннее использование. Практический смысл: RPL-опция dual-license пригодна для открытых проектов, а для типичного закрытого коммерческого кода выбор реально между community-тиром (проверь пороги: выручка < $5M и капитал < $10M) и платной лицензией.

### 5.4. Free-тир — это оферта, не контракт

Условия community edition могут пересмотреть в любой релиз, а твоя выручка — перерасти порог. Момент, когда тир слетает, обычно совпадает с моментом, когда миграция дороже всего (кода стало больше). Считай TCO по платному тиру **до** добавления пакета: если цифра неприемлема — пакет неприемлем уже сейчас.

### 5.5. Транзитивная коммерциализация

Пакет X может тянуть коммерческий Y как транзитивную зависимость — сам ты Y не ставил, а обязательства уже твои. `NuGetAuditMode=all` покрывает только CVE; лицензии транзитивного дерева смотри license-сканом (п. 3.6) — он и покажет diff, когда у зависимости зависимостей сменится лицензия.

### 5.6. «NuGet предупредит» — не предупредит

Restore не проверяет лицензии и не сигналит об их смене. Единственный сигнал, который у тебя есть — тот, который ты сам построил в CI. Нет license-скана — узнаешь о разломе из твиттера, в худший момент.

---

## 6. Decision tree

```
Нужен in-process CQRS / mediator?
├─ Просто разделить commands/queries
│    → прямые handler-вызовы из endpoint (zero deps)
├─ Нужен единый seam для cross-cutting (validation/logging/tx)
│  ├─ Достаточно своего → dispatcher ~50 строк (п. 4.1)
│  └─ Хочется IRequest-семантику → Mediator (source-gen, MIT);
│       MediatR 13+ коммерческий
├─ Нужен durable messaging / saga / outbox / брокер
│  ├─ Один инструмент на всё → Wolverine (MIT)
│  └─ Wolverine не лёг → Rebus / Brighter / raw-клиент брокера
│       (MassTransit v9 — только осознанно и платно; v8 EOL конец 2026)
└─ Легаси на MediatR ≤12
     → можно оставаться (лицензия свободная навсегда),
       но фиксируй в ADR дату пересмотра — версия заморожена

Выбираю НОВЫЙ пакет любого назначения?
├─ Лицензия + история владения чистые? ── нет → альтернатива / свой код
├─ Verified publisher, живой repo, CVE-реакция? ── нет → отказ
├─ Reflection-магия при живом source-gen аналоге? ── да → бери source-gen
├─ API расползётся по всей кодовой базе?
│    → ADR + тонкий адаптер, иначе exit cost неуправляем
└─ Всё ок → ставим; lock-файл и license-скан уже в CI
```

---

## 7. Cheat sheet

| Нужно | Дефолт vault | Если мало / легаси |
|-------|--------------|--------------------|
| In-process CQRS | Прямые handler-вызовы / свой dispatcher | Mediator (source-gen, MIT) |
| Durable messaging, saga, outbox | Wolverine (MIT) | Rebus / Brighter / raw-клиенты |
| Маппинг | Mapperly (Apache 2.0) или руками | AutoMapper ≤14 в легаси |
| Asserts | Shouldly | AwesomeAssertions (drop-in для FA-сьютов) |
| Mocking | NSubstitute | Moq только в легаси |
| BDD | Reqnroll | SpecFlow мёртв (EOL 31.12.2024) |
| Load testing | k6 (Grafana) | NBomber v4 (Apache 2.0, заморожен) |
| Bulk-операции EF | `ExecuteUpdate` / `ExecuteDelete`, PG `COPY` | `EFCore.BulkExtensions.MIT` |
| OIDC / auth server | OpenIddict / Keycloak | Duende (платно — ради поддержки) |
| CI-гигиена | Lock-файлы + `--locked-mode`, `NuGetAuditMode=all`, license-скан | — |

Мнемоника выбора: **ноль зависимостей → source-gen OSS → коммерческое осознанно.** Каждый шаг вправо — только когда предыдущий реально не закрывает задачу.

---

## См. также

- [[cqrs-mediatr|CQRS + Mediator]] — сам паттерн, pipeline behaviors, структура слайсов
- [[messaging|Messaging]] — брокеры, outbox, где действительно нужен bus
- [[testing|Testing]] — Shouldly, NSubstitute, Testcontainers в деле
- [[object-mapping|Object Mapping]] — Mapperly vs ручной маппинг, подробно
- [[architecture-decisions|ADRs]] — куда записывать license decision по каждой зависимости

## Reading list

- **Jimmy Bogard — AutoMapper and MediatR Going Commercial** — https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/
- **MassTransit** (v9 / Massient, условия) — https://masstransit.io
- **Wolverine docs** — https://wolverine.netlify.app
- **Mediator (Martin Othamar)** — https://github.com/martinothamar/Mediator
- **Mapperly** — https://mapperly.riok.app
- **Shouldly docs** — https://docs.shouldly.org
- **AwesomeAssertions** — https://www.nuget.org/packages/AwesomeAssertions
- **Reqnroll** — https://reqnroll.net
- **k6 (Grafana)** — https://k6.io
- **NuGet: Auditing package dependencies** — https://learn.microsoft.com/nuget/concepts/auditing-packages
- **NuGet: lock files / locked mode** — https://learn.microsoft.com/nuget/consume-packages/package-references-in-project-files
- **RPL-1.5 (текст лицензии, OSI)** — https://opensource.org/license/rpl-1-5
- **Moq SponsorLink issue** — https://github.com/devlooped/moq/issues/1372
- **Duende Software** — https://duendesoftware.com
