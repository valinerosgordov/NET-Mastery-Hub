# OOP

## Access Modifiers

public, internal (по умолчанию для top-level), private, protected, protected internal, private protected.

---

## Interface vs Abstract Class

Interface — контракт, множественное наследование. Abstract class — общая база, конструктор, поля. «Может делать» vs «является разновидностью».

---

## sealed, Inheritance, Composition

sealed — запрет наследования или override. C# — только одиночное наследование классов.

Inheritance — is-a. Composition — has-a, делегирование. Предпочитать composition (DI, тестируемость).

---

## Polymorphism, Encapsulation

Polymorphism — virtual/override, вызов по типу объекта. Интерфейсы — полиморфизм по контракту.

Encapsulation — private/protected поля, доступ через свойства и методы. Свойства — валидация, вычисляемые значения.

---

## Static Constructor, Extension Methods

Static constructor — один раз перед первым обращением к типу. Thread-safe инициализация.

Extension method — статический метод, первый параметр с `this`. Условия: статический класс, статический метод.
