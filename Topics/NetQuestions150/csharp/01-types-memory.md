# Типы и память

## readonly vs const

const — при объявлении, примитивы/enum/string, inline при компиляции. readonly — в конструкторе, любой тип, может быть instance.

---

## class, record, struct

| Аспект | class | record | struct |
|--------|-------|--------|--------|
| Тип | Reference | Reference | Value |
| Равенство | По ссылке | По значению | По значению |
| Назначение | Объекты с идентичностью | DTO, immutable | Маленькие значения |

record — Equals, GetHashCode, ToString, with. record class / record struct. Primary constructors.

---

## ref struct

Только на stack. Span<T>, ReadOnlySpan<T>. Нельзя боксить, в async, в замыканиях. Работа с памятью без аллокаций.

---

## Heap и Stack

Stack — локальные переменные, параметры, value types. Heap — объекты, managed heap, GC. Кратко: stack — вызовы, heap — объекты.

---

## Boxing, string, StringBuilder

Boxing — value type → object, аллокация в heap. Unboxing — извлечение. Избегать: generic коллекции, Span<T>, IEquatable<T>.

string — immutable. StringBuilder — мутабельный буфер для множественных конкатенаций.
