# Строки в Go

## Что такое строки?

!!! info

    Строки в Go реализованы как неизменяемые последовательности байт с автоматической поддержкой UTF-8.

Под капотом строка представляет собой структуру:

```go
type stringStruct struct {
    str unsafe.Pointer // указатель на массив байт
    len int            // длина строки в байтах
}
```

### Ключевые особенности

- **Неизменяемость**: Созданную строку нельзя изменить
- **UTF-8 по умолчанию**: Go использует UTF-8 для кодировки текста
- **Лёгкость передачи**: Строки передаются по значению, но фактически копируется только указатель

## Создание и инициализация строк

### Базовые способы создания

```go
// Литеральное объявление
s1 := "Hello, 世界"  // UTF-8 строка
s2 := `Raw\nstring` // Сырая строка (не обрабатывает escape-последовательности)

// Через преобразование
s3 := string([]byte{72, 101, 108, 108, 111}) // "Hello"
```

### Эффективное создание строк

```go
// strings.Builder для многократной конкатенации
var builder strings.Builder
builder.WriteString("Hello")
builder.WriteString(", ")
builder.WriteString("World!")
result := builder.String() // "Hello, World!"

// bytes.Buffer как альтернатива
var buf bytes.Buffer
buf.WriteString("Привет")
buf.WriteByte(' ')
buf.WriteString("мир")
str := buf.String() // "Привет мир"
```

## Работа с элементами и срезами строк

### Получение элемента по индексу

!!! warning

    В Go строки хранятся как байты в UTF-8, поэтому прямое обращение по индексу возвращает байт, а не символ!

```go
s := "Привет"
byteAt2 := s[2]  // Возвращает байт, а не символ!
fmt.Printf("Байт: %v, символ: %c\n", byteAt2, byteAt2)
```

### Для получения символов (рун) по индексу

```go
// Преобразуем в срез рун
runes := []rune(s)
runeAt2 := runes[2]  // Теперь это полноценный символ
fmt.Printf("Символ: %c\n", runeAt2)  // Выведет 'и'
```

### Пример безопасного получения символа

```go
func getRuneAt(s string, pos int) rune {
    for i, r := range s {
        if i == pos {
            return r
        }
    }
    return utf8.RuneError
}
```

### Получение подстрок через срезы

!!! warning

    Срезы строк работают на уровне байтов, что может привести к неожиданным результатам с Unicode:

```go
s := "Привет, 世界!"
sub := s[1:4]  // Срез по байтам, а не символам!
fmt.Println(sub)  // Может вывести мусор для многобайтовых символов
```

#### Правильный способ для Unicode

```go
// Преобразуем в руны и делаем срез
runes := []rune(s)
subRunes := runes[1:4]
properSub := string(subRunes)
fmt.Println(properSub)  // "рив"
```

#### Оптимизированная версия (без полного преобразования)

```go
func substring(s string, start, end int) string {
    counter := 0
    for i := range s {
        if counter >= start {
            for j := i; j < len(s); {
                _, size := utf8.DecodeRuneInString(s[j:])
                if counter >= end {
                    return s[i:j]
                }
                j += size
                counter++
            }
            break
        }
        counter++
    }
    return s
}

fmt.Println(substring("Привет, 世界!", 2, 5))  // "иве"
```

#### Сравнение подходов

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Прямое индексирование | Быстро, O(1) доступ | Работает с байтами, а не символами |
| Преобразование в []rune | Простота, понятность | Дополнительное выделение памяти |
| Итерация с utf8.Decode | Экономия памяти | Более сложная реализация |

1. Для однобайтовых символов (ASCII) - можно использовать прямое индексирование.
2. Для Unicode текста - преобразуйте в []rune если нужно много операций.
3. Для однократных операций - используйте итерацию с utf8.DecodeRuneInString.

## Работа с Unicode и UTF-8

### Правильное получение длины строки

```go
s := "Hello, 世界"

// Длина в байтах
byteLen := len(s) // 13

// Длина в символах (рунах)
runeLen := utf8.RuneCountInString(s) // 9
```

## Итерация по строке

```go
// По байтам
for i := 0; i < len(s); i++ {
    fmt.Printf("Byte %d: %x\n", i, s[i])
}

// По символам (рунам)
for idx, runeValue := range s {
    fmt.Printf("Rune %d: %#U\n", idx, runeValue)
}

// Преобразование в []rune
runes := []rune(s)
for i, runeValue := range runes {
    fmt.Printf("Символ %d: %c\n", i, runeValue)
}

// Использование utf8.DecodeRuneInString
for i := 0; i < len(s); {
    runeValue, width := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("Символ: %#U, ширина: %d байт\n", runeValue, width)
    i += width
}
```

### Сравнение подходов для итерации

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| `range` | Простота, читаемость | Нет доступа к позиции байта |
| `utf8.DecodeRuneInString` | Полный контроль | Более сложный синтаксис |
| `[]rune` | Простота, доступ по индексу | Дополнительное выделение памяти |

## Оптимизированные операции по строкам

### Эффективная конкатенация

```go
// Плохо (создаёт промежуточные строки)
s := "a" + "b" + "c" + "d"

// Хорошо (использует strings.Builder)
var builder strings.Builder
builder.WriteString("a")
builder.WriteString("b")
builder.WriteString("c")
builder.WriteString("d")
s := builder.String()
```

### Сравнение строк

```go
// Простое сравнение
if s1 == s2 {
    // ...
}

// Без учёта регистра
if strings.EqualFold(s1, s2) {
    // ...
}
```

## Преобразование и кодировки

### Строки <-> Байты

```go
// Строка → Байты
data := []byte("Hello")

// Байты → Строка
str := string([]byte{72, 101, 108, 108, 111})
```

### Строки <-> Руны

```go
// Строка → Руны
runes := []rune("Привет")

// Руны → Строка
str := string([]rune{1055, 1088, 1080, 1074, 1077, 1090})
```

## Полезные функции пакета `strings`

### Поиск и проверка

```go
// Проверка префикса/суффикса
strings.HasPrefix(s, "http://")
strings.HasSuffix(s, ".txt")

// Поиск подстроки
strings.Contains(s, "abc")
strings.Index(s, "xyz")
```

### Модификация строк

```go
// Замена
strings.Replace(s, "old", "new", -1)

// Разделение и объединение
parts := strings.Split("a,b,c", ",")
joined := strings.Join(parts, "-")
```

### Триминг и обработка

```go
// Удаление пробелов
strings.TrimSpace("  text  ")

// Изменение регистра
strings.ToUpper(s)
strings.ToLower(s)
strings.Title(s)
```

## Производительность и оптимизация

### Ключевые моменты

- Избегайте частой конкатенации с помощью `+` в циклах.
- Используйте `strings.Builder` для построения больших строк.
- Преобразуйте в `[]byte` для частого доступа к отдельным символам.
- Кешируйте результаты дорогих операций (например, преобразований).

### Пример кеширования результатов дорогих строковых операций

```go
var (
    translationCache = make(map[string]string)
    cacheMutex      sync.RWMutex
)

// Функция с кешированием сложного преобразования
func translateText(s string) string {
    // Сначала проверяем кеш
    cacheMutex.RLock()
    if cached, found := translationCache[s]; found {
        cacheMutex.RUnlock()
        return cached
    }
    cacheMutex.RUnlock()

    // Дорогая операция перевода (например, нормализация Unicode)
    result := expensiveTranslationOperation(s)

    // Сохраняем в кеш
    cacheMutex.Lock()
    translationCache[s] = result
    cacheMutex.Unlock()

    return result
}

func expensiveTranslationOperation(s string) string {
    // Имитация дорогой операции
    time.Sleep(100 * time.Millisecond)
    return strings.ToUpper(s)
}
```

#### Когда использовать кеширование?

- При частом вызове функций с одинаковыми аргументами.
- Для тяжелых строковых операций (нормализация, транслитерация).
- Когда память не является критичным ограничением.