# Языковые конструкции

## Generics

Параметризация типов. where T : class, struct, new(). Нет boxing, type safety. Covariance (out), contravariance (in).

---

## delegate, ref, out

Delegate — ссылка на метод. Func, Action, Predicate. Event — обёртка с +=, -=.

ref — вход+выход, инициализация до вызова. out — выход, присвоение в методе обязательно. out с C# 7 — inline объявление.

---

## Exception Handling

try-catch-finally. catch when (condition) — фильтр. throw — сохраняет stack trace. throw ex — сбрасывает (избегать). throw new Exception("msg", ex) — обёртка.

---

## Overloading, Overriding, using

Overloading — одно имя, разные сигнатуры. Overriding — замена virtual метода, override.

using — гарантия Dispose. await using — IAsyncDisposable. using declaration (C# 8) — Dispose при выходе из scope.

---

## Switch, NRT, Primary Constructors, Culture

Switch expression — возвращает значение, все ветки — один тип. _ — обязательная ветка.

Nullable Reference Types — string vs string?, #nullable enable. Статический анализ.

Primary constructors (C# 12) — параметры в объявлении типа. Меньше boilerplate.

Culture: CurrentCulture, CurrentUICulture. TimeZoneInfo для дат с timezone.
