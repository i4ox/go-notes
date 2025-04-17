# Интерфейсы в Go

## Что такое интерфейсы?

!!! info

    Интерфейсы в Go - это контракты, которые определяют, какое поведение должен реализовать тип. Они не описывают данные, а только методы, которые должны быть у типа.

```go
// Интерфейс - это набор сигнатур методов
type Speaker interface {
    Speak() string
}
```

## Основные встроенные интерфейсы

### Stringer - для строкового представления.

```go
type Stringer interface {
    String() string
}

// Пример реализации
type User struct {
    Name string
    Age  int
}

func (u User) String() string {
    return fmt.Sprintf("%s, %d лет", u.Name, u.Age)
}

func main() {
    user := User{"Алексей", 30}
    fmt.Println(user) // Алексей, 30 лет
}
```

### Error - для обработки ошибок

```go
type Error interface {
    Error() string
}

// Кастомная ошибка
type ValidationError struct {
    Field string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("неверное значение поля %s", e.Field)
}
```

### IO интерфейсы (пакет io)

| Интерфейс | Методы | Назначение
|--------|---------------------------|---------------|
| Reader | Read([]byte) (int, error) | Чтение данных |
| Writer | Write([]byte) (int, error) | Запись данных |
| Closer | Close() error             | Закрытие ресурсов |

```go
// Пример использования Reader
func processInput(r io.Reader) {
    data := make([]byte, 100)
    n, err := r.Read(data)
    // ...
}
```

## Практическое применение интерфейсов

### Полиморфизм

```go
type Animal interface {
    Sound() string
}

type Dog struct{}
func (d Dog) Sound() string { return "Гав!" }

type Cat struct{}
func (c Cat) Sound() string { return "Мяу!" }

func MakeSound(a Animal) {
    fmt.Println(a.Sound())
}
```

### Тестирование (моки)

```go
type Database interface {
    GetUser(id int) (*User, error)
}

// Реальная реализация
type RealDB struct{}

// Тестовая реализация
type MockDB struct{}

func TestUserService(t *testing.T) {
    service := UserService{db: MockDB{}}
    // тестируем с mock-объектом
}
```

## Продвинутые техники

### Композиция интерфейсов

```go
type ReadWriter interface {
    Reader
    Writer
}
```

### Пустой интерфейс

```go
// Может содержать любой тип
func PrintAny(v interface{}) {
    fmt.Println(v)
}
```

### Определение типов в runtime

```go
func checkType(v interface{}) {
    switch v.(type) {
    case int:
        fmt.Println("Это число")
    case string:
        fmt.Println("Это строка")
    default:
        fmt.Println("Неизвестный тип")
    }
}
```

### Проверка реализации интерфейса

```go
var _ MyInterface = (*MyType)(nil) // Проверка на этапе компиляции
```

## Частые вопросы

### **В**: Как проверить, реализует ли тип интерфейс?

```go
var s Shape
if circle, ok := s.(Circle); ok {
    // circle - это Circle
}
```
