---
tags: [learning-path, overview, navigation]
level: All
date: 2026-06-12
---

# 📚 Learning Path — Overview

> Главная навигация по vault. Гайд "с чего начать" для разных уровней. Все ссылки ведут на актуальные файлы в этой knowledge base.

---

## 🎯 Куда я хочу прийти?

Выбери цель — увидишь свой путь:

| Цель | Roadmap |
|------|---------|
| Системно выучить/повторить сам язык | [[01_language-map\|01 Language Map]] |
| Junior → Middle .NET Backend | [[02_junior-to-middle\|02 Junior → Middle]] |
| Middle → Senior .NET | [[03_middle-to-senior\|03 Middle → Senior]] |
| Готовлюсь к собеседованию | [[04_interview-prep\|04 Interview Prep]] |
| Прокачать конкретную тему | [[05_topics-by-priority\|05 Topics by Priority]] |
| Senior tips & tricks (cheatsheet) | [[09_senior-tips-cheatsheet\|09 Senior Tips]] |
| Behavioral / soft skills для интервью | [[10_interview-behavioral\|10 Interview Behavioral]] |
| Книги и видео | [[99_reading-list\|99 Reading List]] |

---

## 🗺️ Структура vault

Внутри каждой тематической папки файлы разложены по уровням `Junior/` → `Middle/` → `Senior/` (файлы уровня All — в корне папки). Полное оглавление с описанием каждого файла — `INDEX.md` в корне раздела (генерируется скриптом `Scripts/generate_index.ps1`).

```
LearningPath/    10  roadmaps, interview prep, reading list — этот раздел
CSharp/          40  язык: basics → modern features → async → метапрограммирование
Runtime/          9  CLR: GC, JIT, memory model, threading internals, диагностика
AspNetCore/      22  web: pipeline, auth, DI, HttpClient, SignalR, Blazor, AOT
EFCore/          13  ORM: tracking, запросы, миграции, паттерны, Dapper
SQL/              8  реляционные БД: indexes, optimization, PostgreSQL deep
Architecture/    14  SOLID, DDD, CQRS, distributed, decision guides, case studies
Quality/          5  clean code, refactoring, code review, static analysis
Testing/          5  fundamentals → mocking → integration → mutation
Performance/     11  профилирование, кеширование, оптимизация, HFT
Infrastructure/  15  Docker, k8s, CI/CD, messaging, observability, LLM/AI
Snippets/         5  готовые рецепты: CRUD, Result, MediatR, EF, MVVM
```

Ключевые точки входа по темам:

- Язык с нуля → [[csharp-basics|C# Basics]], дальше по [[01_language-map\|Language Map]]
- Async и многопоточность → [[async-threading|Async и Threading]] + [[threading-basics|Threading Basics]]
- Память и GC → [[gc-memory|GC и Memory]] + [[types-and-memory|Types и Memory]]
- Web-фреймворк → [[pipeline-middleware|Pipeline]] + [[aspnet-dependency-injection-deep|DI deep]]
- Выбор архитектуры → [[patterns-decision-guide|Patterns Decision Guide]] + [[real-world-scenarios|Real-World Scenarios]]

---

## 📊 Статистика vault

- **Всего файлов:** 157 контентных заметок (+ README в каждой папке + INDEX)
- **Общий объём:** ~5.5 MB
- **По уровням:** Junior 25 · Middle 29 · Middle→Senior 14 · Senior 73 · остальное — All/paths
- **Покрытие:** все 15 блоков «Complete C# 2026» cheat sheet + architecture/distributed/infra сверху
- **Языки:** русский (основной), технические термины — английские

---

## 🚀 Quick Start

**Если ты новичок в этом vault:**

1. Открой [[02_junior-to-middle\|02 Junior → Middle]] или [[03_middle-to-senior\|03 Middle → Senior]] в зависимости от уровня
2. Открой соответствующий файл в нужной папке
3. После прочтения — пиши код по теме (pet-project!)
4. Возвращайся к [[09_senior-tips-cheatsheet\|09 Senior Tips]] для повторения

**Если готовишься к собесу:**

1. [[04_interview-prep\|04 Interview Prep]] — что повторить
2. [[10_interview-behavioral\|10 Interview Behavioral]] — soft skills
3. [[09_senior-tips-cheatsheet\|09 Senior Tips]] — быстрый review

**Если хочешь в конкретную тему:**

[[05_topics-by-priority\|05 Topics by Priority]] — рейтинг тем по value/effort

---

## 🎓 Уровни

| Уровень | Что значит | Как достичь |
|---------|-----------|-------------|
| **Junior** | Пишет CRUD по заданию | C# fundamentals + ASP.NET basics |
| **Junior+** | Самостоятельно делает простые фичи | + EF Core + auth |
| **Middle** | Полный цикл разработки | + architecture + testing + DevOps |
| **Middle+** | Проектирует фичи, делает code review | + DDD + observability + performance |
| **Senior** | Архитектурные решения, mentoring | + system design + leadership |
| **Senior+** | Cross-functional impact, R&D | + business + cross-team |

См. [[02_junior-to-middle\|02 Junior → Middle]] и [[03_middle-to-senior\|03 Middle → Senior]] для детальных roadmap.
