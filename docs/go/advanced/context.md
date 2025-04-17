# Контекст в Go

!!! warning

    Запомните главное правило: если операция может блокироваться или выполняться долго — она должна принимать `context.Context` и корректно обрабатывать его отмену.

## Что такое контекст?

!!! info

    Контекст (`context.Context`) — это фундаментальный механизм в Go для управления жизненным циклом операций, особенно в конкурентных сценариях.

Контекст позволяет:

- Передавать значения между компонентами
- Распространять сигналы отмены
- Устанавливать временные ограничения

## Основные функции создания контекста

### Базовые контексты

```go
// Пустой корневой контекст
ctx := context.Background()

// Контекст-заглушка (используется при рефакторинге)
ctx := context.TODO()
```

### Контекст с отменой

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // Важно вызывать cancel для освобождения ресурсов

// В другой горутине или функции:
select {
case <-ctx.Done():
    // Обработка отмены
    return ctx.Err() // Возвращает context.Canceled
default:
    // Продолжение работы
}
```

### Контекст с таймаутом

```go
// Автоматическая отмена через 1 секунду
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()
```

### Контекст с дедлайном

```go
// Автоматическая отмена в конкретное время
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
```

## Передача значений через контекст

```go
// Создание контекста с данными
ctx := context.WithValue(context.Background(), "requestID", "12345")

// Получение значения
if reqID, ok := ctx.Value("requestID").(string); ok {
    fmt.Println("Request ID:", reqID)
}
```

### Рекомендации по использованию значений

- Создавайте типизированные ключи
- Документируйте используемые ключи
- Избегайте частого использования для передачи основных параметров

```go
type ctxKey string

const (
    requestIDKey ctxKey = "requestID"
    userKey      ctxKey = "user"
)

// Установка значения
ctx = context.WithValue(ctx, requestIDKey, "67890")

// Получение значения
if id, ok := ctx.Value(requestIDKey).(string); ok {
    // ...
}
```

## Обработка ошибок контекста

```go
func operation(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        return nil // Успешное завершение
    case <-ctx.Done():
        switch ctx.Err() {
        case context.Canceled:
            log.Println("Operation canceled")
        case context.DeadlineExceeded:
            log.Println("Operation timeout")
        }
        return ctx.Err()
    }
}
```

## Практические примеры

### HTTP-сервер с таймаутом

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()
    
    // Передаем контекст в вызов БД
    result, err := db.QueryContext(ctx, "SELECT...")
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        // Обработка других ошибок
    }
    // ...
}
```

### Параллельные задачи с отменой

```go
func processTasks(ctx context.Context, tasks []Task) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    errCh := make(chan error, len(tasks))
    
    for _, task := range tasks {
        go func(t Task) {
            select {
            case errCh <- t.Process(ctx):
            case <-ctx.Done():
                errCh <- ctx.Err()
            }
        }(task)
    }
    
    // Ожидание завершения
    for range tasks {
        if err := <-errCh; err != nil {
            cancel() // Отменяем другие задачи при первой ошибке
            return err
        }
    }
    return nil
}
```

## Лучшие практики

1. **Всегда передавайте контекст явно**: не храните его в структурах, передавайте как первый аргумент функций.
2. **Проверяйте ctx.Done() в долгих операциях**: особенно в циклах, запросах к БД, HTTP-клиентах.
3. **Освобождайте ресурсы с defer cancel()**: даже если не используете отмену явно.
4. **Не злоупотребляйте context.WithValue**: используйте только для request-scoped данных.
5. **Соблюдайте иерархию контекстов**: производные контексты должны иметь меньший или равный срок жизни.

## Распространенные ошибки

### Игнорирование ctx.Done()

```go
// Плохо
func BadExample(ctx context.Context) {
    time.Sleep(10 * time.Second) // Не реагирует на отмену
}
```

### Утечка горутин

```go
// Плохо
func LeakExample(ctx context.Context) {
    go func() {
        <-ctx.Done() // Горутина может висеть вечно
    }()
}
```

### Переопределение контекста

```go
// Плохо
func OverrideExample(ctx context.Context) {
    ctx = context.Background() // Потеря родительского контекста
}
```