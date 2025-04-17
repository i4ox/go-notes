# Функции в Go

## Объявление функций

### Базовый синтаксис

```go
func name(parameter-list) (result-list) {
    body
}
```

### Примеры объявлений

```go
func add(x int, y int) int {
    return x + y
}

// Сокращенная запись для параметров одного типа
func add(x, y int) int {
    return x + y
}
```

## Возвращаемые значения

### Множественные возвращаемые значения

```go
func swap(x, y string) (string, string) {
    return y, x
}

// Использование
a, b := swap("hello", "world")
```

### Именованные возвращаемые значения

```go
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // "голый" return
}
```

## Функциональные типы

### Функции как значения

```go
func compute(fn func(float64, float64) float64) float64 {
    return fn(3, 4)
}

// Использование
hypot := func(x, y float64) float64 {
    return math.Sqrt(x*x + y*y)
}
fmt.Println(compute(hypot))
```

### Анонимные функции

```go
func main() {
    func() {
        fmt.Println("Анонимная функция")
    }()
}
```

## Замыкания

### Определение и использование

```go
func adder() func(int) int {
    sum := 0
    return func(x int) int {
        sum += x
        return sum
    }
}

// Использование
pos := adder()
fmt.Println(pos(1)) // 1
fmt.Println(pos(2)) // 3
```

## Вариативные функции

### Объявление

```go
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

// Использование
fmt.Println(sum(1, 2))        // 3
fmt.Println(sum(1, 2, 3, 4))  // 10
```

### Передача слайса в вариативную функцию

```go
nums := []int{1, 2, 3, 4}
fmt.Println(sum(nums...))
```

## Часто задаваемые вопросы

### **В**: Когда использовать именованные возвращаемые значения?

- Когда возвращаемые значения имеют очевидные имена
- В коротких функциях для улучшения читаемости
- Когда нужно документировать значение возврата

### **В**: В чем разница между методами и функциями?

- Методы имеют receiver (привязаны к типу)
- Функции независимы от типов
- Методы могут модифицировать состояние receiver

### **В**: Как работает defer?

- Откладывает выполнение до возврата из функции
- Аргументы вычисляются немедленно
- Выполняется в порядке LIFO (последний отложенный - первый выполненный)
