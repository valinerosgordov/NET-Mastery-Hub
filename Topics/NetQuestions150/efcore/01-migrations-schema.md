# Migrations и схема

## Migrations

Версионирование схемы БД. Файлы в Migrations/: класс с Up()/Down(), *_Designer.cs — snapshot. Команды: add, database update, remove.

**Применение**: `dotnet ef database update` или `context.Database.MigrateAsync()`. В production — отдельный шаг CI/CD, backup перед миграцией.

---

## Seed Data

В Migration: InsertData в Up(). В DbContext: HasData в OnModelCreating — данные попадают в миграцию. При EnsureCreated или Migrate применяются.
