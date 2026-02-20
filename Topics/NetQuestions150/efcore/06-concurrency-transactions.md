# Concurrency и Transactions

## Optimistic Concurrency

Проверка при SaveChanges: не изменилась ли строка. [Timestamp] (byte[]), [ConcurrencyCheck] или IsConcurrencyToken(). WHERE RowVersion = @old в UPDATE. 0 affected → DbUpdateConcurrencyException.

---

## Transactions

BeginTransactionAsync, CommitAsync, RollbackAsync. Один DbContext — встроенная транзакция через SaveChanges.

**Кросс-контекстные**: один SqlConnection, BeginTransaction, оба контекста UseTransaction(transaction).

---

## Connection Pooling

ADO.NET пулит соединения. EF использует DbConnection из провайдера. DbContext Scoped — один на запрос. Соединение возвращается в пул при Dispose.
