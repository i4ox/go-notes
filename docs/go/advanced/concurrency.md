# Параллельность в Go

Go предлагает мощные примитивы для параллельного программирования, которые делают написание конкурентного кода проще и безопаснее. 

Основные концепции:

- Горутины(Goroutines)
- Каналы(Channels)
- Примитивы cинхронизации
- WaitGroup

## Горутины

!!! info

    Горутины — это легковесные потоки, управляемые Go runtime. Они используют меньше памяти, чем традиционные потоки ОС, и позволяют эффективно выполнять тысячи параллельных задач.

```go
func main() {
    // Запуск горутины
    go func() {
        fmt.Println("Выполняется в горутине")
    }()
    
    // Даем время горутине завершиться
    time.Sleep(100 * time.Millisecond)
}
```

### Ключевые особенности

- Запускаются с ключевым словом go
- Имеют стек небольшого размера (2KB), который может динамически расти
- Планируются Go runtime, а не ОС
- Дешевле потоков ОС в 100+ раз

### Ограничения

- Горутины не являются потокобезопасными.
- Необходимо использовать каналы или мьютексы для синхронизации.

## Каналы

!!! info

    Каналы — это типизированные каналы связи между горутинами, обеспечивающие синхронизацию.

### Типы каналов

```go
ch := make(chan int)     // Небуферизированный канал
chBuf := make(chan int, 10) // Буферизированный канал
chWrite := make(chan<- int) // Однонаправленный на запись
chRead := make(<-chan int) // Однонаправленный на чтение
```

### Буферизация в каналах

Буферизированные каналы создаются с указанием емкости буфера во втором аргументе make.

```go
ch := make(chan int, 3) // Канал с буфером на 3 элемента
```

#### Как работает буфер

- При отправке (ch <- val) данные сначала помещаются в буфер, если есть свободное место
- При заполнении буфера операция отправки блокируется до освобождения места
- При чтении (<-ch) данные берутся сначала из буфера
- Если буфер пуст, операция чтения блокируется до появления данных

#### Пример поведения

```go
ch := make(chan int, 2) // Буфер на 2 элемента
ch <- 1                 // Отправка - буфер [1]
ch <- 2                 // Отправка - буфер [1,2]
ch <- 3                 // Блокировка - буфер полон
```

### Однонаправленные каналы

!!! info

    Односторонние каналы особенно полезны при передаче в функции.

```go
// Функция принимает канал только для записи
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i // Можно только писать
    }
    close(ch)
}

// Функция принимает канал только для чтения
func consumer(ch <-chan int) {
    for num := range ch {
        fmt.Println(num) // Можно только читать
    }
}

func main() {
    ch := make(chan int)
    go producer(ch) // Неявное приведение к chan<- int
    consumer(ch)    // Неявное приведение к <-chan int
}
```

#### Использование с буферизованными каналами

```go
bufCh := make(chan int, 10)
var writeOnly chan<- int = bufCh
var readOnly <-chan int = bufCh
```

#### Преимущества

1. **Безопасность типов**: Компилятор предотвращает неправильное использование не того вида каналов.
2. **Ясность кода**: Четко показывает намерения разработчика, Упрощает понимание потока данных.
3. **Защита от ошибок**: Предотвращает случайную отправку/получение не в том месте.

#### Особенности

1. **Приведение типов**: Двунаправленный канал может быть неявно приведен к одностороннему, обратное поведение невозможно.
2. **Закрытие каналов**: Закрывать может только канал для записи.

### Операции с каналами

#### Чтение (<-ch)

- Если канал не закрыт и содержит данные - возвращает значение
- Если канал не закрыт, но пуст - блокирует горутину
- Если канал закрыт - немедленно возвращает нулевое значение типа

#### Отправка (ch <- val)

- Если канал не закрыт и есть место в буфере - добавляет значение
- Если канал не закрыт, но буфер полон - блокирует горутину
- Если канал закрыт - вызывает панику

#### Закрытие (close(ch))

- Помечает канал как закрытый
- Все последующие попытки отправки вызовут панику
- Чтение будет возвращать оставшиеся значения, затем нулевые
- Многократное закрытие вызывает панику

### Паттерны с каналами

#### Остановка по сигналу

```go
func worker(stop <-chan struct{}) {
    for {
        select {
        case <-stop:
            return
        default:
            // Полезная работа
        }
    }
}
```

#### Fan-in(много писателей) - объединение результатов

```go
func fanIn(inputs []<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, in := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for n := range ch {
                out <- n
            }
        }(in)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

#### Fan-out(много читателей) - Разделение работы

```go
func fanOut(input <-chan int, outputs []chan<- int) {
    for val := range input {
        for _, out := range outputs {
            out <- val
        }
    }
}
```

#### Worker Pool(Пул обработчиков)

```go
func workerPool(tasks <-chan Task, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasks {
                results <- process(task)
            }
        }()
    }
    wg.Wait()
    close(results)
}
```

### Закрытие канала

Каналы можно закрывать, чтобы сигнализировать о завершении передачи данных.

```go
close(ch) // Закрытие канала
```

### Итерация по каналу

```go
for v := range ch {
    fmt.Println(v) // Получение значений до закрытия канала
}
```

## Примитивы синхронизации

Go предлагает богатый набор примитивов синхронизации для управления параллельным доступом и координации горутин. Рассмотрим каждый из них подробно.

### sync.Mutex (исключающая блокировка)

!!! info

    Защита общих ресурсов от одновременного доступа.

```go
var mu sync.Mutex
var sharedData int

func updateData() {
    mu.Lock()
    defer mu.Unlock()
    sharedData++
}
```

1. Гарантирует, что только одна горутина может владеть блокировкой.
2. `Lock()` блокирует выполнение горутины, если мьютекс уже захвачен другой горутиной, и ожидает, пока мьютекс не будет освобожден.
3. `Unlock()` освобождает мьютекс, позволяя другим горутинам захватить его.
4. Не рекурсивный (повторный захват в одной горутине приведет к deadlock).

### sync.RWMutex (читающая-записывающая блокировка)

!!! info

    Оптимизация для сценариев "много читателей / редкие писатели".

```go
var rwMu sync.RWMutex
var cache map[string]string

func get(key string) string {
    rwMu.RLock()
    defer rwMu.RUnlock()
    return cache[key]
}

func set(key, value string) {
    rwMu.Lock()
    defer rwMu.Unlock()
    cache[key] = value
}
```

1. Множество горутин может одновременно удерживать `RLock()`, что позволяет нескольким читателям одновременно получать доступ к ресурсу.
2. Только одна горутина может удерживать `Lock()`, и это возможно только при отсутствии активных читателей.
3. Запись (вызов `Lock()`) блокирует как новых читателей, так и писателей, пока текущая запись не будет завершена.


### sync.WaitGroup (ожидание завершения группы горутин)

!!! info

    Ожидание завершения набора горутин.

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        // Работа горутины
    }(i)
}

wg.Wait() // Ожидание всех
```

- `Add(n)` - увеличивает счетчик на n.
- `Done()` - уменьшает счетчик на 1 (лучше использовать с defer).
- `Wait()` - блокирует, пока счетчик не станет 0.

### sync.Once (однократное выполнение)

!!! info

    Гарантированное однократное выполнение кода.
    У `sync.Once` есть единственный метод `Do()`.

```go
var (
    once sync.Once
    config map[string]string
)

func loadConfig() {
    once.Do(func() {
        // Инициализация конфига
        config = readConfig()
    })
}
```

#### Особенности

- Потокобезопасная инициализация
- Гарантирует выполнение ровно один раз, даже при конкурентных вызовах

### sync.Cond (условная переменная)

!!! info

    Ожидание/сигнализация об изменении состояния.

```go
var (
    mu    sync.Mutex
    cond  = sync.NewCond(&mu)
    ready bool
)

// Ожидающая горутина
mu.Lock()
for !ready {
    cond.Wait() // Освобождает мьютекс и ждет
}
// Работа с общим состоянием
mu.Unlock()

// Сигнализирующая горутина
mu.Lock()
ready = true
cond.Broadcast() // или cond.Signal()
mu.Unlock()
```

Методы:

- `Wait()` - освобождает мьютекс и блокирует.
- `Signal()` - пробуждает одну любую горутину.
- `Broadcast()` - пробуждает все горутины.

### sync.Pool (пул объектов)

!!! info

    Переиспользование временных объектов для снижения нагрузки на GC.

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

// Получение
buf := bufPool.Get().(*bytes.Buffer)
defer bufPool.Put(buf)

// Использование
buf.Reset()
buf.WriteString("example")
```

#### Особенности

- Объекты могут быть удалены сборщиком мусора в любой момент
- Не гарантирует сохранение объектов между вызовами Get

#### Преимущества

- Уменьшает количество аллокаций памяти
- Снижает нагрузку на GC
- Повышает производительность при частом создании/удалении объектов

#### Когда использовать sync.Pool

- Когда объекты требуют частого создания/удаления
- Когда инициализация объектов дорогая
- Когда объекты могут быть безопасно переиспользованы

#### Реальный пример использования sync.Pool

!!! info

    Сценарий: У нас есть веб-сервер, который часто обрабатывает JSON-запросы и формирует JSON-ответы. Для этого постоянно создаются и удаляются буферы `bytes.Buffer`.

    Проблема: Частое создание/удаление буферов создает нагрузку на сборщик мусора.

```go
package main

import (
    "bytes"
    "encoding/json"
    "net/http"
    "sync"
)

// Пул буферов
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Получаем буфер из пула
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf) // Возвращаем в пул после использования
    
    // Сбрасываем буфер перед использованием
    buf.Reset()
    
    // Создаем данные для ответа
    user := User{ID: 1, Name: "John Doe", Email: "john@example.com"}
    
    // Кодируем JSON в буфер
    if err := json.NewEncoder(buf).Encode(user); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Копируем из буфера в ResponseWriter
    w.Header().Set("Content-Type", "application/json")
    buf.WriteTo(w)
}

func main() {
    http.HandleFunc("/", handleRequest)
    http.ListenAndServe(":8080", nil)
}
```

### sync.Map (потокобезопасная мапа)

!!! info

    Потокобезопасная альтернатива map + sync.Mutex для специфичных случаев.

```go
var sm sync.Map

// Сохранение
sm.Store("key", "value")

// Загрузка
if val, ok := sm.Load("key"); ok {
    fmt.Println(val)
}

// Удаление
sm.Delete("key")
```

#### Когда использовать

- Когда ключи стабильны и редко меняются
- Для read-heavy workload
- Вместо map + RWMutex в 

## Стандартный пакет `atomic`

!!! info

    Пакет `sync/atomic` предоставляет низкоуровневые атомарные операции для синхронизации.

### Основные типы и операции

```go
import "sync/atomic"

var counter int32

// Атомарное сложение
atomic.AddInt32(&counter, 1)

// Атомарное чтение
val := atomic.LoadInt32(&counter)

// Атомарная запись
atomic.StoreInt32(&counter, 42)

// Compare-And-Swap (CAS)
swapped := atomic.CompareAndSwapInt32(&counter, 42, 43)
// Если counter == 42, устанавливает 43 и возвращает true
// Иначе возвращает false
```

### Поддерживаемые типы

- `int32`
- `int64`
- `uint32`
- `uint64`
- `uintptr`
- `unsafe.Pointer`

### Пример счетчика с `atomic`

```go
type AtomicCounter struct {
    value int32
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt32(&c.value, 1)
}

func (c *AtomicCounter) Decrement() {
    atomic.AddInt32(&c.value, -1)
}

func (c *AtomicCounter) Value() int32 {
    return atomic.LoadInt32(&c.value)
}

func (c *AtomicCounter) CompareAndSwap(old, new int32) bool {
    return atomic.CompareAndSwapInt32(&c.value, old, new)
}
```

### Реальный пример использования `atomic`

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

var (
    requestsHandled uint64
    activeWorkers   int32
)

func worker(id int, wg *sync.WaitGroup) {
    atomic.AddInt32(&activeWorkers, 1)
    defer func() {
        atomic.AddInt32(&activeWorkers, -1)
        wg.Done()
    }()

    // Имитация работы
    time.Sleep(time.Second)
    
    // Увеличиваем счетчик обработанных запросов
    atomic.AddUint64(&requestsHandled, 1)
}

func main() {
    var wg sync.WaitGroup
    
    // Запускаем 100 воркеров
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }
    
    // Мониторинг в фоне
    go func() {
        for {
            fmt.Printf("Active workers: %d, Requests handled: %d\n",
                atomic.LoadInt32(&activeWorkers),
                atomic.LoadUint64(&requestsHandled))
            time.Sleep(200 * time.Millisecond)
        }
    }()
    
    wg.Wait()
    fmt.Println("Final count:", atomic.LoadUint64(&requestsHandled))
}
```

### Когда использовать `atomic`

- Для простых счетчиков и флагов
- Когда нужна максимальная производительность
- Для реализации lock-free алгоритмов
- Для координации между горутинами без блокировок

### Ограничения `atomic`

- Работает только с простыми типами
- Нет гарантий порядка операций между разными переменными
- Сложнее в отладке по сравнению с мьютексами
- Не подходит для сложных структур данных

### Преимущества перед мьютексами

- Нет блокировок (лучшая масштабируемость)
- Меньшие накладные расходы
- Подходит для высоконагруженных участков кода