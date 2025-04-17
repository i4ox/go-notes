# Ошибки в Go

## Что такое ошибки?

!!! info

    Ошибки в Go - это значения, которые реализуют интерфейс `error`.

Они используются для обработки исключительных ситуаций и сигнализируют о том, что что-то пошло не так. Интерфейс `error` определен следующим образом:

```go
type error interface {
    Error() string
}
```

## Создание ошибок

### Стандартный ошибок (пакет errors)

```go
import "errors"

func validate(input string) error {
    if input == "" {
        return errors.New("пустая строка недопустима")
    }
    return nil
}
```

### Форматированные ошибки (пакет fmt)

```go
func processFile(filename string) error {
    if !fileExists(filename) {
        return fmt.Errorf("файл %s не найден", filename)
    }
    return nil
}
```

### Пользовательские ошибки

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("ошибка валидации поля %s: %s", e.Field, e.Message)
}

func validateUser(u User) error {
    if u.Age < 0 {
        return ValidationError{"Age", "возраст не может быть отрицательным"}
    }
    return nil
}
```

## Обработка ошибок

Обработка ошибок в Go осуществляется с помощью проверки возвращаемого значения на nil. Если функция возвращает ошибку, это значение не должно быть nil.

### Базовый паттерн

```go
result, err := someOperation()
if err != nil {
    // Обработка ошибки
    return err // или обработка и продолжение
}
// Использование result
```

### Проверка типа ошибки

```go
if err := process(); err != nil {
    if verr, ok := err.(ValidationError); ok {
        // Обработка ValidationError
        fmt.Println("Ошибка в поле:", verr.Field)
    } else {
        // Другие ошибки
        fmt.Println("Неизвестная ошибка:", err)
    }
}
```

## Продвинутые техники

### Обертывание ошибок

```go
import "errors"

var ErrNotFound = errors.New("не найдено")

func findUser(id int) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("findUser %d: %w", id, err)
    }
    return user, nil
}

// Проверка обернутых ошибок
if errors.Is(err, ErrNotFound) {
    // Обработка случая "не найдено"
}
```

### Распоковка ошибок

```go
if err := process(); err != nil {
    var verr ValidationError
    if errors.As(err, &verr) {
        // Обработка ValidationError
        fmt.Println("Ошибка в поле:", verr.Field)
    }
}
```

### Цепочки ошибок

```go
func processRequest(req Request) error {
    if err := validate(req); err != nil {
        return fmt.Errorf("ошибка валидации: %w", err)
    }
    if err := save(req); err != nil {
        return fmt.Errorf("ошибка сохранения: %w", err)
    }
    return nil
}
```

### Ошибки с контекстом

!!! info

    Иногда полезно передавать контекст ошибки, чтобы предоставить больше информации о том, где произошла ошибка. Это можно сделать, добавляя дополнительную информацию в пользовательские ошибки.

```go
type MyError struct {
    Msg      string
    Context  string
}

func (e MyError) Error() string {
    return fmt.Sprintf("%s: %s", e.Context, e.Msg)
}

func doSomething() error {
    return MyError{"Что-то пошло не так!", "в функции doSomething"}
}

func main() {
    if err := doSomething(); err != nil {
        fmt.Println("Ошибка:", err) // Вывод: Ошибка: в функции doSomething: Что-то пошло не так!
    }
}
```

## Лучшие практики

### Документируйте ошибки

```go
// GetUser возвращает пользователя по ID.
// Возможные ошибки:
// - ErrNotFound если пользователь не существует
// - ErrDatabase при проблемах с БД
func GetUser(id int) (*User, error) {
    // ...
}
```

### Создавайте константы для часто используемых ошибок

```go
var (
    ErrInvalidInput = errors.New("неверный ввод")
    ErrNotFound     = errors.New("не найдено")
)
```

## Работа с паникой(panic)

Хотя в Go предпочтительно использовать ошибки, иногда необходимо использовать panic.    

### Когда использовать panic?

- При невозможности продолжить выполнение программы.
- В случаях, которые указывают на ошибку программиста.

```go
func mustParse(s string) int {
    i, err := strconv.Atoi(s)
    if err != nil {
        panic(fmt.Sprintf("неверное число: %v", s))
    }
    return i
}
```

### Восстановление после panic

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Восстановлено после:", r)
        }
    }()
    
    // Код, который может вызвать panic
    panic("что-то пошло не так")
}
```

## Процесс комплексной обработки ошибок

```go
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e AppError) Unwrap() error {
    return e.Err
}

func processRequest() error {
    if err := validate(); err != nil {
        return AppError{
            Code:    400,
            Message: "неверный запрос",
            Err:     err,
        }
    }
    return nil
}

func main() {
    if err := processRequest(); err != nil {
        var appErr AppError
        if errors.As(err, &appErr) {
            fmt.Printf("Ошибка %d: %s\n", appErr.Code, appErr)
        } else {
            fmt.Println("Неизвестная ошибка:", err)
        }
    }
}
```

## Частые вопросы

### **В**: Как обрабатывать ошибки в Go?

- Проверяйте ошибки сразу после вызова функции, возвращающей ошибку.
- Используйте конструкцию `if err != nil` для обработки ошибок.

### **В**: Как создать пользовательскую ошибку?

- Определите структуру, реализующую интерфейс `error`, и добавьте метод `Error()`.

### **В**: Как использовать ошибки в функциях?

- Возвращайте ошибку как второе значение из функции и проверяйте ее на nil.

### **В**: Как передавать контекст ошибок?

- Добавляйте дополнительную информацию в пользовательские ошибки, чтобы указать, где произошла ошибка.
