# Версии Go

> [!INFO]
> В этом разделе описываются изменения в Go с версии на версию.
> Затрагивается только сам язык, а не стандартная библиотека, тулинг или компилятор.

## Go 1.0

### Append - добавление в слайс

> [!INFO]
> Это новая функция, поэтому она не требует изменения старых кодов.

Теперь строку можно добавлять напрямую в слайс из байтов.

```go
greeting := []byte{}

greeting = append(greeting, []byte("Hello, ")...) // Раньше
greeting = append(greeting, "World"...)           // Сейчас
```

### Close - закрытие канала

Функция `close` представляет механизм, который позволяет сообщить каналу о том, что вы не будете использовать его дальше.
Также важно, что закрыть канал может только горутина, в которой данные отправляются, однако ранее не было проверки этого ограничения.
Теперь же такой код стал ошибкой компиляции.

```go
var c chan int  
var csend chan<- int = c  
var crecv <-chan int = c  
close(c)     // допустимо  
close(csend) // допустимо  
close(crecv) // ошибка компиляции  
```

### Composite literals - Составные литералы

> [!INFO]
> Утилита `gofmt -s` будет автоматически удалять избыточные указания типов там, где это разрешено.

В Go 1 можно упускать указание типа в составных литералах массивов, срезов и словарей, если их элементы являются указателями.

Ранее были валидными только первые три конструкции.

```go
type Date struct {  
    month string  
    day   int  
}
// Полностью указанный тип — всегда допустимо.  
holiday1 := []Date{  
    Date{"Feb", 14},  
    Date{"Nov", 11},  
    Date{"Dec", 25},  
}
// Опущение имени типа — всегда допустимо.  
holiday2 := []Date{  
    {"Feb", 14},  
    {"Nov", 11},  
    {"Dec", 25},  
}
// Указатели, полностью указанный тип — всегда допустимо.  
holiday3 := []*Date{  
    &Date{"Feb", 14},  
    &Date{"Nov", 11},  
    &Date{"Dec", 25},  
}
// Указатели, тип опущен — допустимо только в Go 1 и выше.  
holiday4 := []*Date{  
    {"Feb", 14},  
    {"Nov", 11},  
    {"Dec", 25},  
}
```

### Goroutines during init - Горутины во время инициализации

Ранее горутины во время инициализации, не начинали выполняться до завершения всех процессов инициализации.

Теперь же горутины в `init` начинают работать сразу.

```go
var PackageGlobal int

func init() {  
    c := make(chan int)  
    go initializationFunction(c)  
    PackageGlobal = <-c  
}
```

### The rune type

В Go 1 появился новый тип `rune`, который представляет собой `int32` и используется для хранения символом Unicode.

```go
delta := 'δ' // delta имеет тип rune  
var DELTA rune  
DELTA = unicode.ToUpper(delta)  
epsilon := unicode.ToLower(DELTA + 1)  
if epsilon != 'δ'+1 {  
    log.Fatal("некорректное приведение регистра для греческого")  
}  
```

### The error type

В Go 1 появился новый встроенный интерфейс `error`, а также начал активно использоваться в стандартной библиотеке.

```go
type error interface {  
    Error() string  
}  
```

### Deleting from maps - Удаление из карт

> [!INFO]
> Утилита `gofix` автоматически преобразует код в соответствие с новым правилами.

Ранее удаление элемента из карты выполнялось так:

```go
m[k] = value, false // False указывал на удаление элемента
```

Теперь это делается так, удаление несуществующего элемента ничего не делает:

```go
delete(m, k) // Удаление элемента начиная с версии Go 1
```

### Iterating in maps - Итерация в картах

Ранее порядок обхода карты при использовании `for range` был неопределенным и зависел от платформы.
Была ситуация, что тесты работали на одной машине, но не на другой.

Теперь же обход карт стал ОФИЦИАЛЬНО НЕПРЕДСКАЗУЕМЫМ и это надо учитывать при разработке.

```go
m := map[string]int{"Sunday": 0, "Monday": 1}  
for name, value := range m {  
    f(name, value) // функция f не должна предполагать что Sunday будет первым значением
}  
```

### Mutiple assignment - Множественное присваивание

В Go 1 уточнены правила порядка вычисления выражений в присваивании.
Все выражения в правой части вычисляются до выполнения присваивания.

```go
sa := []int{1, 2, 3}  
i := 0  
i, sa[i] = 1, 2 // теперь i = 1, sa[0] = 2  
```

### Returns and shadowed variables - Возвращение значений и теневые переменные

> [!INFO]
> После изменения, такой код необходимо исправить вручную.

Если в функции с именоваными возвращаемыми значениями используется `return` без аргументов, а одна из переменных была затенена, то компилятор выдаст ошибку.

```go
func Bug() (i, j, k int) {  
    for i = 0; i < 5; i++ {  
        for j := 0; j < 5; j++ { // j переопределяется  
            k += i * j  
            if k > 100 {  
                return // Ошибка компиляции  
            }  
        }  
    }  
    return // Здесь ошибок нет  
}  
```

### Copying structs with unexported fields - Копирование структур с неэкспортируемыми полями

Ранее в Go нельзя было копировать структуры с неэкспортируемыми(непубличными) полями. С рядом исключений:

- Копирование внутри методов структуры (при передаче `receiver`).
- Внутренние реализации функций `copy` и `append` игнорировали это ограничение.

Это было исправлено в Go 1.

Пусть у нас есть пакет `p`:

```go
package p

type Struct struct {
    Public int
    secret int
}

func NewStruct(a int) Struct {
    return Struct{a, computeSecret(a)}
}

func (s Struct) String() string {
    return fmt.Sprintf("{%d (secret %d)}", s.Public, s.secret)
}
```

> [!INFO]
> Важно, что поле `secret` тоже копируется, но не может быть использовано за пределами пакета `p`.

После версии Go 1:

```go
import "p"
import "fmt"

func main() {
    myStruct := p.NewStruct(23)
    copyOfMyStruct := myStruct  // Теперь копирование работает!
    fmt.Println(myStruct, copyOfMyStruct)
}
```

### Equality - Сравнение

В Go 1 разрешено сравнивать структуры и массивы, если их элементы можно сравнить.

Также теперь структуры можно использовать в качестве ключей в картах.

```go
type Day struct {  
    long  string  
    short string  
}  
holiday := map[Day]bool{  
    {"Christmas", "XMas"}: true,  
}  
fmt.Println(holiday[Day{"Christmas", "XMas"}]) // true  
```

## Go 1.1

### Integer division by zero - Целочисленное деление на ноль

Ранее если 0 было константным значением, то код компилировался без ошибок, но приводил к панике во время работы.

Теперь:

- Если 0 контантное значение, то компилятор выдаст ошибку.
- Если же 0 содержится в переменной, по-прежнему вызывается run-time panic.

```go
func f(x int) int {
    return x / 0  // Ошибка во время компиляции
}
```

### Surrogates in Unicode literals - Суррогаты в литералах Unicode

> [!INFO]
> Суррогат - это символ, который содержит в себе последовательность двух символов Unicode.

Теперь строковые (`string`) и рунные (`rune`) литералы не могут содержать суррогаты, так как это некорректные коды Unicode.

```go
// Ошибка компиляции:
s := "\uD800"  // D800-DFFF — это суррогатные половинки в Unicode
```

### Method values - Значения методов

В Go 1.1 добавили возможность использовать методы, привязывая их к конкретным значениям(`receiver`).

```go
package main

import (
    "fmt"
    "bytes"
)

func main() {
    var b bytes.Buffer
    writeToBuffer := b.Write // Метод-значение (method value)

    writeToBuffer([]byte("Hello, "))  // Используем как обычную функцию
    writeToBuffer([]byte("Go!"))

    fmt.Println(b.String())  // "Hello, Go!"
}
```

> [!INFO]
> Не стоит путать с `method expression`.

```go
func main() {
    var b bytes.Buffer
    writeFn := (*bytes.Buffer).Write // method expression

    writeFn(&b, []byte("Hello, "))  // Передаем явный receiver
    writeFn(&b, []byte("Go!"))

    fmt.Println(b.String())  // "Hello, Go!"
}
```

### Return requirements - Новые правила для возвращаемых значений

Раньше любая функция с возвращаемым значением должна была явно завершаться `return` или `panic`, даже если это было очевидно.

```go
func loopForever() int {
    for {
        // Бесконечный цикл, но компилятор требовал return!
    }
    return 0  // Обязательный return, хотя он никогда не выполняется
}
```

Теперь разрешено упускать `return`, если компилятор гарантированно знает, что функция не завершится другим способом.

```go
func loopForever() int {
    for { }  // Бесконечный цикл — return не требуется
}
```

```go
func f(x int) int {
    if x > 0 {
        return x
    } else {
        panic("negative number")
    }  // return не требуется
}
```

## Go 1.2

### Use of nil - Использование nil

Ранее `nil`-указатели могли приводить к некорректному доступу к памяти, но не всегда выбрасывали панику.

```go
type T struct {
    X     [1 << 24]byte  // 16MB массив
    Field int32
}

func main() {
    var x *T  // x == nil
    fmt.Println(x.Field)  // Некорректный доступ к памяти!
}
```

В коде выше поле `Field` находится в смещении 16MB от начала структуры `T`, а `x` ссылается на `0x0`, поэтому паника могла не вызваться.

Теперь в Go 1.2 любая операция разыменования `nil`(`nil`-указатели, `nil`-срезы, `nil`-интерфейсы) гарантированно вызывают панику.

```go
func main() {
    var x *T
    fmt.Println(x.Field)  // Теперь 100% panic: runtime error
}
```

### Three-index slices - Трехиндексные нотации для срезов

В Go 1.2 добавили трехиндексные нотации для срезов.

Ранее код работал так:

```go
var array [10]int
slice := array[2:4]  // Элементы с 2 по 3, но ёмкость = 8
```

То есть:

- `slice` содержит элементы [2, 3], но может быть расширен до конца `array`.
- `cap(slice) == 8` (от 2 до конца массива).

Теперь можно явно ограничить емкость среза:

```go
slice := array[2:4:7]  // Элементы [2, 3], но cap(slice) = 5 (7-2)
```

То есть:

- `len(slice) = 4 - 2 = 2`
- `cap(slice) = 7 - 2 = 5`
- Теперь нельзя получить доступ к `array[7:]` через `slice`!

## Go 1.3

В этом релизе не было изменений в языке Go!

## Go 1.4

### For-range loops - Улучшение for-range циклов

> [!INFO]
> Это новая функция, поэтому она не требует изменения старых кодов.

Ранее в Go нельзя было использовать `range` без переменной.

Можно было использовать `_`:

```go
for _ = range x {  // Неудобно!
    fmt.Println("Итерация")
}
```

Теперь можно записывать данный цикл таким образом:

```go
for range x {
    fmt.Println("Итерация")
}
```

### Method calls on **T - Запрет вызова метода через указатель на указатель

Ранее компиляторы `gc` и `gccgo` позволяли ошибочное поведение.

```go
type T int
func (T) M() {}

var x **T
x.M()  // Ошибка в Go 1.4!
```

- По спецификации Go разрешена только одна автоматическая разыменовка(`*T`).
- `x` - это `**T`, и `x.M()` требует двойного разыменования, что неправильно.

## Go 1.5

### Map literals - Литералы для словарей

До Go 1.5 в словарях нельзя было опускать ключи, даже если они были очевидными.

```go
type Point struct {
    Lat, Lon float64
}

m := map[Point]string{
    Point{29.935523, 52.891566}:   "Persepolis",
    Point{-25.352594, 131.034361}: "Uluru",
    Point{37.422455, -122.084306}: "Googleplex",
}
```

Код выше выглядит избыточным.

Теперь можно опустить ключи и получить словарь в виде:

```go
m := map[Point]string{
    {29.935523, 52.891566}:   "Persepolis",
    {-25.352594, 131.034361}: "Uluru",
    {37.422455, -122.084306}: "Googleplex",
}
```

## Go 1.6

В этом релизе не было изменений в языке Go!

## Go 1.7

В данном релизе уточнили, как определить заверщающий оператор в функциях.

Ранее было неясно, считается ли функция корректной, если после `return` есть пустая строка или лишняя `;`.
Теперь точно сказано:

- Go игнорирует пустые строки и `;` в конце.
- Важно только, чтобы последний непустой оператор был завершающим (`return`, `panic`, `for {}` и т. д.).

Код, который работал раньше, продолжит работать. Просто теперь четко определено, что считать завершающим оператором.

```go
func example() int {
    return 42  // Это завершающий оператор
    ;          // Пустой оператор игнорируется
}
```

## Go 1.8

### Different struct tags - Различающиеся теги структур

Раньше компилятор запрешал преобразовывать одну структуру в другую, если они отличалишь только тегами.
Например, `json:"foo"` и `json:"bar"`.

Теперь такой код работает без ошибок:

```go
func example() {
    type T1 struct {
        X int `json:"foo"`
    }
    type T2 struct {
        X int `json:"bar"`
    }
    
    var v1 T1
    var v2 T2
    v1 = T1(v2) // Теперь компилируется без ошибок
}
```

### Exponential notation - Экспонента

Теперь в языке гарантируется поддержка экспоненты до 16 бит(раньше требовалось 32 бита).

То есть `gc` и `gccgo` все еще работают с экспонентами до 32 бит, но теперь поддерживаются другие реализации, где требуется 16 бит.

## Go 1.9

### Type aliases support - Поддержка типовых псевдонимов

В Go 1.9 введена поддержка типовых псевдонимов.

```go
type T1 = T2
```

## Go 1.10

### Shifts Without Constants - Сдвиги без использования констант

Теперь разрешено использовать такие выражения, как `x[1.0 << s]`, где `s` это беззнаковое целое.

То есть теперь поддерживаются сдвиги без использования констант.

### Anonymous methods receivers - Анонимные получатели методов

Теперь можно использовать любой тип в качестве получателя метода(`receiver`).

Раньше можно было использовать только заранее определенные типы, теперь можно и анонимные.

```go
package main

import (
    "fmt"
    "io"
)

// Пример метода для типа io.Reader
func (s struct{ io.Reader }) Read(p []byte) (n int, err error) {
    return s.Reader.Read(p)
}

func main() {
    // Используем анонимный тип, который включает в себя io.Reader
    var r struct{ io.Reader } = struct{ io.Reader }{io.Stdin}

    // Теперь можно вызвать метод Read на этом анонимном типе
    var p []byte
    _, err := r.Read(p)
    if err != nil {
        fmt.Println("Ошибка при чтении:", err)
    } else {
        fmt.Println("Чтение прошло успешно")
    }
}
```

## Go 1.11

В этом релизе не было изменений в языке Go!

## Go 1.12

В этом релизе не было изменений в языке Go!

## Go 1.13

### Prefixes support for numeric literals - Поддержка префиксов для чисел

Теперь в Go присутствует поддержка префиксов для чисел.

Виды префиксов:

- **Бинарные литералы** - префикс `0b` или `0B`. Пример: `0b1011`.
- **Восьмеричные литералы** - префикс `0o` или `0O`. Пример: `0o660`.
- **Шестнадцатеричные вещественные литералы** - префикс `0x` или `0X`. Пример: `0x1.0p-1021`, через `p` записывается степень.
- **Мнимые литералы** - суффикс `i` теперь может быть использован для любых целых или вещественных чисел, чтобы создать мнимое число.
Пример: `3 + 4i` или `0x2.5p2i`.
- **Разделители цифр** - в числовых литералах теперь можно использовать `_` для разделения групп цифр. Пример: `1_000_000`.

### Delete restriction to count of shifts - Ограничение на количество сдвигов

Убрали ограничение, чтобы количество сдвигов в оператарах `<<` и `>>` всегда были беззнаковыми.

```go
package main

import "fmt"

func main() {
    var x int = 32
    fmt.Println(x << -1) // Нельзя использовать отрицательное значение для сдвига
}
```

Теперь они могут быть знаковыми.

```go
package main

import "fmt"

func main() {
    var x int = 32
    fmt.Println(x << 2)   // Сдвиг на 2 бита влево: 32 * 2^2 = 128
    fmt.Println(x >> 2)   // Сдвиг на 2 бита вправо: 32 / 2^2 = 8
    fmt.Println(x << -1)  // Сдвиг на -1 бит (влево на 1 бит в обратную сторону)
}
```

## Go 1.14

Было добавдено изменение, что в интерфейс могут встраиваться другие интерфейсы с одинаковыми методами.
Они больше не должны быть уникальными.

```go
package main

import "fmt"

// Определим два интерфейса с одинаковыми методами
type InterfaceA interface {
    Method()
}

type InterfaceB interface {
    Method()
}

// Этот интерфейс теперь успешно компилируется, несмотря на пересечение методов.
type InterfaceC interface {
    InterfaceA
    InterfaceB
}

// Реализуем метод для InterfaceC
type Struct struct{}

func (s Struct) Method() {
    fmt.Println("Method implemented.")
}

func main() {
    var c InterfaceC = Struct{}
    c.Method()  // Вывод: "Method implemented."
}
```

## Go 1.15

В этом релизе не было изменений в языке Go!

## Go 1.16

В этом релизе не было изменений в языке Go!

## Go 1.17

### Converting slices to pointers of arrays - Преобразование срезов в указатели на массивы

Теперь возможно преобразовывать срезы `[]T` в указатель на массив типа `*[N]T`.

То есть, если у нас есть срез `s`, то его можно безопасно преобразовать в указатель на массив с размером `N`.

```go
package main

import "fmt"

func main() {
    // Исходный срез
    s := []int{1, 2, 3, 4, 5}
    
    // Преобразуем срез в указатель на массив с длиной 3
    a := *(*[3]int)(s)
    
    // Теперь элементы среза и массива имеют одинаковые индексы
    fmt.Println(a[0]) // 1
    fmt.Println(a[1]) // 2
    fmt.Println(a[2]) // 3
}
```

> [!INFO]
> Преобразование вызовет панику, если длина среза меньше требуемого массива N.

```go
package main

import "fmt"

func main() {
    s := []int{1, 2} // Срез длиной 2
    a := *(*[3]int)(s) // Это приведет к панике, так как срез не содержит достаточно элементов
    fmt.Println(a)
}
```

### unsafe.Add

Функция `unsafe.Add` позволяет прибавлять целое число к указателю.
Это полезно при манипуляции с памятью.

Функция возвращает указатель, увеличенный на заданное количество байтов.

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    arr := [5]int{10, 20, 30, 40, 50}
    ptr := unsafe.Pointer(&arr[0]) // Получаем указатель на первый элемент массива
    
    // Прибавляем 2 к указателю, чтобы получить указатель на третий элемент
    newPtr := unsafe.Add(ptr, 2 * unsafe.Sizeof(arr[0]))
    fmt.Println(*(*int)(newPtr)) // Выводит: 30
}
```

### unsafe.Slice

Функция `unsafe.Slice` позволяет создать срез, начиная с указанного указателя.

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    arr := [5]int{1, 2, 3, 4, 5}
    ptr := unsafe.Pointer(&arr[0]) // Получаем указатель на первый элемент массива
 
    // Создаем срез длиной 3, начиная с указателя на первый элемент
    s := unsafe.Slice((*int)(ptr), 3)
    fmt.Println(s) // Выводит: [1 2 3]
}
```

## Go 1.18

### Generics - Параметризованные типы

1. Функции и структуры теперь могут использовать параметризованные типы.
2. Использование `~` в интерфейсах для указания на базовые типы.
3. Новые встроенные идентификаторы: `any`, `comparable`.

Пример 1 - Дженерик-функция:

```go
package main

import "fmt"

// Обобщенная функция для поиска минимума
func Min[T int | float64](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Min(5, 10))        // 5 (работает с int)
    fmt.Println(Min(3.5, 2.8))     // 2.8 (работает с float64)
}
```

Пример 2 - Дженерик-структура:

```go
package main

import "fmt"

// Дженерик-структура Stack
type Stack[T any] struct {
    items []T
}

// Добавление элемента в стек
func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

// Извлечение элемента из стека
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func main() {
    intStack := Stack[int]{}  // Создаем стек для int
    intStack.Push(10)
    intStack.Push(20)
    fmt.Println(intStack.Pop()) // 20, true

    strStack := Stack[string]{} // Создаем стек для string
    strStack.Push("hello")
    fmt.Println(strStack.Pop()) // "hello", true
}
```

Пример 3 - `comparable`:

```go
package main

import "fmt"

// Функция для поиска элемента в срезе
func Contains[T comparable](slice []T, value T) bool {
    for _, v := range slice {
        if v == value {
            return true
        }
    }
    return false
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}
    fmt.Println(Contains(numbers, 3)) // true

    words := []string{"hello", "world"}
    fmt.Println(Contains(words, "go")) // false
}
```

## Go 1.19

В этом релизе не было изменений в языке Go!

## Go 1.20

### Converting slices to arrays - Преобразование срезов в массивы

> [!INFO]
> Если длина среза меньше длины массива, то при преобразовании вызовет панику.

Ранее в Go 1.17 можно было преобразовывать срез в указатель на массив, но нельзя было получить сам массив напрямую.

```go
package main

import "fmt"

func main() {
    s := []byte{1, 2, 3, 4}
    p := *(*[4]byte)(s) // Работает, но это указатель на массив
    fmt.Println(p)    // [1 2 3 4]
}
```

Теперь это можно делать напрямую.

```go
func main() {
    s := []byte{1, 2, 3, 4}
    a := [4]byte(s) // Теперь можно преобразовать напрямую!
    fmt.Println(a)  // [1 2 3 4]
}
```

### New unsafe functionality - Новая функциональность unsafe

Добавлены `SliceData`, `String`, `StringData`, которые позволяют глубже работать с низкоуровневыми структурами данных.

Пример использования `unsafe.String` для создания новой строки без копирования:

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    data := []byte("hello")
    str := unsafe.String(&data[0], len(data)) // Конвертируем в строку без копирования
    fmt.Println(str) // "hello"
}
```

### Define order of equalation structure and arrays - Определение порядка сравнения структур и массивов

Теперь в спецификации Go четко прописан порядок сравнения структур и массивов:

- Структуры сравниваются по полям сверху вниз.
- Массивы сравниваются по индексам слева направо.
- Если найдена первая разница, то сравнение завершается.

```go
package main

import "fmt"

type A struct {
    x int
    y map[string]int // map нельзя сравнивать!
}

func main() {
    a1 := A{x: 1, y: nil}
    a2 := A{x: 2, y: nil}

    // Go 1.20 четко говорит, что сравнение остановится на x
    fmt.Println(a1 == a2) // false (сравнение map не выполняется)
}
```

### Upgrade work with `comparable` types - Обновление работы с типами `comparable`

Теперь интерфейсы и составные типы с интерфейсами могут быть ключами в generic-словарях, даже если не все ее значения строго сравнимы.
То есть `==` может паниковать.

Ранее нельзя было использовать `map[K]V`, если `K` имел `comparable` ограничение.
Теперь это разрешено, но сравнение может вызвать панику.

```go
package main

import "fmt"

func HasKey[K comparable, V any](m map[K]V, key K) bool {
    _, exists := m[key]
    return exists
}

func main() {
    m := map[interface{}]int{
        "hello": 1,
        42:      2,
    }

    fmt.Println(HasKey(m, "hello")) // true
    fmt.Println(HasKey(m, 99))      // false
}
```

## Go 1.21

### `min` and `max` functions - Функции `min` и `max`

> [!INFO]
> Работает для `int`, `float64`, `string` и любых типов, которые реализуют интерфейс `comparable`.

Теперь в Go можно находить минимум и максимум без написания лишнего кода.

```go
package main

import "fmt"

func main() {
    fmt.Println(min(3, 1, 4, 2)) // 1
    fmt.Println(max(3, 1, 4, 2)) // 4
}
```

### `clear` for zeroing slices and maps - Удаление элементов из срезов и словарей

> [!INFO]
> `clear(m)` - удаляет все элементы из словаря `m`.
> `clear(s)` - обнуляет все элементы из среза `s`.

Раньше, чтобы очистить карту, приходилось писать следующее:

```go
m := map[string]int{"a": 1, "b": 2}
for k := range m {
    delete(m, k)
}
```

Теперь это можно сделать следующим образом:

```go
clear(m)
```

### Strict order for package initialization - Строгий порядок инициализации пакетов

Теперь Go определяет порядок инициализации пакетов:

1. Все пакеты сортируются по пути импорта.
2. Пока есть неинициализированные пакеты:
    1. Берем первый пакет, у которого все зависимости уже инициализированы.
    2. Инициализируем его и убираем из списка.

### Modified types output - Измененные типы вывода

Go стал умнее в определении типов:

- Теперь можно передавать в generics аргументы, которые сами являются generic-функциями.
- Go лучше понимает методы интерфейсов и использует их для вывода типов.
- Типы для неявных констант теперь выводятся так же, как и в обычных выражениях.
- Ошибки стали более точными — теперь Go отлавливает несовпадения типов раньше.

```go
package main

import (
    "fmt"
    "slices"
)

func main() {
    nums := []int{3, 1, 4, 2}
    idx := slices.IndexFunc(nums, func(n int) bool { return n > 2 })
    fmt.Println(idx) // 0 (индекс первого элемента >2)
}
```

### Armor for `nil`-panics - Защита от panic-ошибок на nil

Теперь Go гарантирует, что `recover()` в отложеннной `defer` функции не вернет `nil`, если паника действительна произошла.

Также теперь `panic(nil)` приведет к `*runtime.PanicNilError`.

```go
package main

import "fmt"

func main() {
    defer func() {
        r := recover()
        fmt.Println(r) // Теперь `r` гарантированно не nil, если была паника
    }()
    
    panic(nil) // Вызовет *runtime.PanicNilError
}
```

## Go 1.22

### Variables inside `for` created again - Переменные внутри `for` создаются заново

Ранее переменные внутри `for` создавались только один раз, а на каждой итерации изменялись.

```go
package main

import "fmt"

func main() {
    funcs := []func(){}

    for i := 0; i < 3; i++ {
        funcs = append(funcs, func() { fmt.Println(i) }) 
    }

    for _, f := range funcs {
        f() // Выведет "3", "3", "3" вместо "0", "1", "2"
    }
}
```

Теперь переменная внутри `for` создается заново при каждой итерации.

```go
package main

import "fmt"

func main() {
    funcs := []func(){}

    for i := 0; i < 3; i++ {
        // i := i // Такой код выполняется под капотом
        funcs = append(funcs, func() { fmt.Println(i) })
    }

    for _, f := range funcs {
        f() // Теперь правильно: "0", "1", "2"
    }
}
```

### `for` can iterate over numbers - `for` может итерировать по числам

Теперь в Go можно итерировать по числам с помощью `for range`.

```go
package main

import "fmt"

func main() {
    for i := range 5 {
        fmt.Println("Итерация:", i)
    }
}
```

## Go 1.23

### `range` now support iteratotion functions - `range` теперь поддерживает итерационные функции

Теперь можно использовать итерационные функции как `range`-выражения в циклах `for-range`.

Функция-итератор может иметь одну из следующих форм:

```go
func(func() bool)      // Только проверка завершения
func(func(K) bool)     // Возвращает один аргумент
func(func(K, V) bool)  // Возвращает два аргумента
```

Пример - простой итератор:

```go
package main

import "fmt"

func counter(n int) func(func(int) bool) {
    return func(yield func(int) bool) {
        for i := 0; i < n; i++ {
            if !yield(i) {
                return
            }
        }
    }
}

func main() {
    for i := range counter(5) {
        fmt.Println(i) // 0, 1, 2, 3, 4
    }
}
```

Пример 2 - итерация по словарю:

```go
package main

import "fmt"

func iterateMap(m map[string]int) func(func(string, int) bool) {
    return func(yield func(string, int) bool) {
        for k, v := range m {
            if !yield(k, v) {
                return
            }
        }
    }
}

func main() {
    data := map[string]int{"a": 1, "b": 2, "c": 3}
    
    for key, value := range iterateMap(data) {
        fmt.Println(key, "=>", value)
    }
}
```

## Go 1.24

### Full support for generic type aliases - Полная поддержка generic-типовых псевдонимов

Теперь `type aliases` может быть параметризован, как обычный определенный тип.

Пример 1 - обобщенный тип:

```go
package main

import "fmt"

// Определяем обобщённый список
type List[T any] []T

// Создаём обобщённый псевдоним
type MyList[T any] = List[T]

func main() {
    var numbers MyList[int] = []int{1, 2, 3}
    var names MyList[string] = []string{"Alice", "Bob"}

    fmt.Println(numbers) // [1 2 3]
    fmt.Println(names)   // ["Alice", "Bob"]
}
```

Пример 2 - обобщённый словарь:

```go
package main

import "fmt"

// Обобщённый словарь
type Dict[K comparable, V any] map[K]V

// Создаём псевдоним с фиксированным ключом `string`
type StringDict[V any] = Dict[string, V]

func main() {
    users := StringDict[int]{"Alice": 25, "Bob": 30}
    
    fmt.Println(users) // map[Alice:25 Bob:30]
}
```
