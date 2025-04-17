# Реализация перечислений (enum) в Go

!!! info

    В Go нет встроенной поддержки перечислений (enum) как в некоторых других языках, но есть несколько идиоматических способов реализовать аналогичную функциональность.

## Использование констант с iota (наиболее распространённый способ)

```go
type Weekday int

const (
    Sunday Weekday = iota // 0
    Monday                // 1
    Tuesday               // 2
    Wednesday             // 3
    Thursday              // 4
    Friday                // 5
    Saturday              // 6
)

func (d Weekday) String() string {
    return [...]string{"Sunday", "Monday", "Tuesday", "Wednesday", 
        "Thursday", "Friday", "Saturday"}[d]
}

// Использование
var today Weekday = Wednesday
fmt.Println(today) // Выведет: Wednesday
```

### Преимуществе

- Типобезопасность
- Простота реализации
- Возможность добавления методов

## Использование строковых констант

```go
type Color string

const (
    Red    Color = "RED"
    Green  Color = "GREEN"
    Blue   Color = "BLUE"
)

// Использование
var favorite Color = Blue
fmt.Println(favorite) // Выведет: BLUE
```

### Преимущества

- Читаемые значения
- Простота сериализации/десериализации

## Расширенное использование iota

```go
type Status int

const (
    Pending Status = iota + 1 // 1
    Approved                  // 2
    Rejected                  // 3
)

// Или с пропуском значений
type State int

const (
    Running State = iota + 1 // 1
    Stopped                  // 2
    _                        // Пропуск значения (3)
    Rebooting                // 4
)
```

## Enum с дополнительными свойствами

```go
type Direction int

const (
    North Direction = iota
    East
    South
    West
)

func (d Direction) String() string {
    return [...]string{"North", "East", "South", "West"}[d]
}

func (d Direction) Offset() (x, y int) {
    switch d {
    case North:
        return 0, 1
    case East:
        return 1, 0
    case South:
        return 0, -1
    case West:
        return -1, 0
    }
    return 0, 0
}

// Использование
dir := East
fmt.Println(dir, "offset:", dir.Offset()) // East offset: 1 0
```

## Проверка допустимости значений

```go
func (d Direction) IsValid() bool {
    switch d {
    case North, East, South, West:
        return true
    }
    return false
}

// Или с использованием массива
var validDirections = [...]Direction{North, East, South, West}

func (d Direction) IsValid() bool {
    for _, dir := range validDirections {
        if d == dir {
            return true
        }
    }
    return false
}
```

## Enum с JSON поддержкой

```go
type Size int

const (
    Small Size = iota
    Medium
    Large
)

func (s Size) MarshalText() ([]byte, error) {
    return []byte(s.String()), nil
}

func (s *Size) UnmarshalText(text []byte) error {
    switch string(text) {
    case "Small":
        *s = Small
    case "Medium":
        *s = Medium
    case "Large":
        *s = Large
    default:
        return fmt.Errorf("invalid size: %s", text)
    }
    return nil
}

func (s Size) String() string {
    return [...]string{"Small", "Medium", "Large"}[s]
}

// Использование с JSON
type Product struct {
    Name string `json:"name"`
    Size Size   `json:"size"`
}
```

## Рекомендации

1. Для простых перечислений используйте `iota` с целочисленным типом.
2. Если важна читаемость при сериализации - используйте строковые константы.
3. Добавляйте методы `String()` для удобного вывода.
4. Реализуйте проверку допустимости значений через `IsValid()`.
5. Для веб-API добавляйте поддержку JSON-сериализации.

## Ограничения

1. Нет встроенной проверки на этапе компиляции.
2. Можно присвоить любое значение базового типа.
3. Нет автоматической проверки уникальности значений.