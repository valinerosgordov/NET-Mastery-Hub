# Relationships и типы

## Navigation Properties

Свойства, ссылающиеся на связанную сущность или коллекцию. EF использует для JOIN, Include, отслеживания. Без навигаций — только FK.

---

## Конфигурация связей

**One-to-many**: HasOne/WithMany, HasForeignKey. OnDelete: Cascade, Restrict, SetNull.

**Many-to-many** (EF Core 5+): HasMany/WithMany, UsingEntity для junction table.

**Owned types** — без идентичности, принадлежат владельцу. OwnsOne, OwnsMany. Address, Money, Value Objects.

**Value vs Reference types**: value types inline в таблице, reference — отдельная таблица или owned.

---

## Shadow Properties, Enum, Indexes

Shadow properties — не в классе, в модели и БД. Аудит, soft delete, TenantId. Доступ через Entry(entity).Property<T>().

Enum — по умолчанию int. HasConversion<string>() для хранения как string.

Индексы: HasIndex, составные, уникальные, filtered (HasFilter).
