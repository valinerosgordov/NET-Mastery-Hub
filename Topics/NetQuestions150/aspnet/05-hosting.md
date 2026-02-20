# Hosting и фоновые задачи

## IHostApplicationLifetime

События: ApplicationStarted, ApplicationStopping (SIGTERM, graceful shutdown), ApplicationStopped. Регистрация callbacks через `lifetime.ApplicationStopping.Register(...)`. CancellationToken для долгих операций при shutdown.

---

## HostedService и BackgroundService

**IHostedService** — контракт для фоновых задач при старте. **BackgroundService** — базовый класс, переопределить `ExecuteAsync(CancellationToken)`. Регистрация: AddHostedService<T>().

Применение: периодическая синхронизация, очереди (RabbitMQ), очистка кэша, warm-up. Остановка — через CancellationToken при shutdown приложения.

**PeriodicTimer** (.NET 6+) — предпочтительнее Task.Delay для периодических задач: интервал между началом итераций, меньше дрейфа.
