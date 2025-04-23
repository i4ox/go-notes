# Управление потоком в Go

## For

### Стандартный цикл

```go
for i := 0; i < 10; i++ {
    sum += i
}
```

### Важное изменение в Go 1.21

!!! info

    В Go 1.21 было внесено важное изменение в работу циклов for. Теперь для переменной цикла на каждой итерации создается новая ссылка. Это влияет на использование переменных цикла в горутинах и замыканиях.

До Go 1.21:

```go
func main() {
    var funcs []func()
    for i := 0; i < 3; i++ {
        funcs = append(funcs, func() {
            fmt.Println(i) // Всегда печатало 3
        })
    }
    for _, f := range funcs {
        f()
    }
}
```

После Go 1.21:

```go
func main() {
    var funcs []func()
    for i := 0; i < 3; i++ {
        funcs = append(funcs, func() {
            fmt.Println(i) // Теперь печатает 0, 1, 2
        })
    }
    for _, f := range funcs {
        f()
    }
}
```

### Управляющие конструкции в циклах

#### break

!!! info

    Прерывает выполнение цикла и продолжает выполнение программы после цикла.

```go
for i := 0; i < 10; i++ {
    if i == 5 {
        break // Выход из цикла при i == 5
    }
    fmt.Println(i)
}
```

#### continue

!!! info

    Пропускает оставшуюся часть текущей итерации и переходит к следующей.

```go
for i := 0; i < 5; i++ {
    if i == 2 {
        continue // Пропускает печать числа 2
    }
    fmt.Println(i)
}
```

#### break с метками

!!! info

    Вы можете удивить знанием этого факта, его мало кто знает!

    Можно использовать break с метками для выхода из вложенных циклов

```go
OuterLoop:
    for i := 0; i < 5; i++ {
        for j := 0; j < 5; j++ {
            if i*j > 10 {
                break OuterLoop // Выход из обоих циклов
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
```

## Range

!!! info

    `range` - это встроенная конструкция языка Go, предназначенная для итерации (перебора) элементов различных коллекций:

    - массивов (arrays) и срезов (slices)
    - карт (maps)
    - строк (strings)
    - каналов (channels)

### По массиву/срезу

```go
nums := []int{2, 3, 4}
for i, num := range nums {
    fmt.Println(i, num)
}
```

### По map

```go
m := map[string]string{"a": "apple", "b": "banana"}
for k, v := range m {
    fmt.Println(k, v)
}
```

### По строке(итерация по рунам)

```go
for i, c := range "go" {
    fmt.Println(i, c)
}
```

### По каналу

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)
for v := range ch {
    fmt.Println(v)
}
```

### Важные особенности `range`

#### Индексы и значения

```go
arr := [3]int{10, 20, 30}
for i, v := range arr {
    // i - индекс, v - копия значения
    v *= 2 // Не изменяет оригинальный массив
}
```

Если надо изменить значение, можно использовать прием ниже.

```go
type Item struct{ value int }
items := []Item{{1}, {2}, {3}}

// Не работает - v это копия
for _, v := range items {
    v.value *= 2
}

// Рабочий вариант
for i := range items {
    items[i].value *= 2
}
```

#### Пропуск компонентов

```go
for i := range nums {}    // Только индексы
for _, v := range nums {} // Только значения
```

#### Изменения в Go 1.21 (аналогично for)

```go
var funcs []func()
for _, v := range []int{1, 2, 3} {
    funcs = append(funcs, func() {
        fmt.Println(v) // Теперь выводит 1, 2, 3 (до 1.21 выводило 3, 3, 3)
    })
}
```

## If

### Базовый синтаксис

```go
if x < 0 {
    return -x
}
```

### If с коротким оператором

```go
if v := math.Pow(x, n); v < lim {
    return v
}
```

### If-else

```go
if x < 0 {
    return -x
} else {
    return x
}
```

## Switch

### Базовый switch

```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("OS X.")
case "linux":
    fmt.Println("Linux.")
default:
    fmt.Printf("%s.", os)
}
```

### Switch без условия

```go
switch {
case t.Hour() < 12:
    fmt.Println("Доброе утро!")
case t.Hour() < 17:
    fmt.Println("Добрый день!")
default:
    fmt.Println("Добрый вечер!")
}
```

### Switch с fallthrough

!!! info

    `fallthrough` позволяет перейти к выполнению следующего блока без проверки условия.

```go
// Вывод: 2 3
switch 2 {
case 1:
    fmt.Println("1")
    fallthrough
case 2:
    fmt.Println("2")
    fallthrough
case 3:
    fmt.Println("3")
}
```

#### Особенности при работе с `fallthrough`

1. Можно использовать только в `switch`.
2. Должен быть последним оператором в `case`.
3. Нельзя использовать в последнем `case` блока.
4. Не проверяет условие следующего `case`.

#### Альтернатива для `fallthrough`

```go
// Вместо fallthrough можно перечислять несколько значений
switch x {
case 1, 2, 3: // Эквивалент case 1: fallthrough; case 2: fallthrough; case 3:
    fmt.Println("x is 1, 2 or 3")
}
```

### Switch с типами

```go
var i interface{} = "hello"

switch v := i.(type) {
case int:
    fmt.Printf("int: %v\n", v)
case string:
    fmt.Printf("string: %v\n", v)
default:
    fmt.Printf("unknown: %T\n", v)
}
```

## Defer

### Порядок выполнения defer

!!! info

    Defer следует принципу LIFO (Last In, First Out).

```go
func main() {
    fmt.Println("start")
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    fmt.Println("end")
}
// Выведет:
// start
// end
// 3
// 2
// 1
```

### Оценка аргументов в defer

!!! info

    Аргументы defer вычисляются немедленно, но функция выполняется перед возвратом.

```go
func main() {
    i := 0
    defer fmt.Println(i) // Выведет 0, не 1
    i++
}
```

### Defer и возвращаемые значения

!!! info
    Defer может изменять именованные возвращаемые значения:

```go
func example() (i int) {
    defer func() { i++ }() // Увеличивает i перед возвратом
    return 1 // Фактически вернёт 2
}
```

### Defer в циклах

```go
for _, file := range files {
    f, err := os.Open(file)
    if err != nil {
        continue
    }
    defer f.Close() // Опасность! Все файлы закроются только при выходе из функции
    // Лучше использовать анонимную функцию:
    func() {
        defer f.Close()
        // работа с файлом
    }()
}
```

### Типичные применения defer

#### Закрытие файлов

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close() // Гарантированное закрытие файла
```

#### Освобождение мьютексов

```go
mu.Lock()
defer mu.Unlock() // Гарантированное освобождение блокировки
```

#### Восстановление после паники

```go
defer func() {
    if r := recover(); r != nil {
        fmt.Println("Recovered from:", r)
    }
}()
```

## Часто задаваемые вопросы

### **В**: Как изменения в Go 1.21 влияют на существующий код?

- Старый код может работать по-другому в горутинах и замыканиях
- Переменные цикла теперь имеют отдельную область видимости на каждой итерации
- Может потребоваться пересмотр кода, использующего переменные цикла в замыканиях

### **В**: Когда использовать break с метками?

- При необходимости выхода из вложенных циклов
- Когда нужно прервать выполнение конкретного внешнего цикла
- В сложных алгоритмах с множественными уровнями вложенности

### **В**: В чем особенность fallthrough в Go?

- Выполняется безусловно
- Должен быть последним оператором в case
- Нельзя использовать в последнем case
- Передает управление следующему case независимо от его условия

### **В**: Какие есть особенности работы defer?

- Выполняется в порядке LIFO
- Аргументы вычисляются при объявлении
- Может изменять именованные возвращаемые значения
- Часто используется для освобождения ресурсов
