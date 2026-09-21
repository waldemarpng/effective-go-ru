# Управляющие конструкции

[оригинал](https://go.dev/doc/effective_go#control-structures)

Управляющие конструкции Go родственны C, но отличаются в важных местах. Нет циклов `do` или `while`, только слегка обобщённый `for`; `switch` гибче; `if` и `switch` принимают необязательный оператор инициализации, как `for`; у `break` и `continue` есть необязательная метка, чтобы указать, что прервать или продолжить; есть и новые конструкции, включая type switch и многосторонний мультиплексор обмена `select`. Синтаксис тоже чуть другой: скобок нет, а тела всегда должны быть ограничены фигурными скобками.

## If

[оригинал](https://go.dev/doc/effective_go#if)

Простой `if` в Go выглядит так:

```
if x > 0 {
    return y
}
```

Обязательные скобки подталкивают писать даже простые `if` в несколько строк. Так и стоит делать, особенно когда в теле есть управляющий оператор вроде `return` или `break`.

Поскольку `if` и `switch` принимают оператор инициализации, им часто задают локальную переменную.

```
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

В библиотеках Go вы увидите: если `if` не перетекает в следующий оператор — то есть тело заканчивается на `break`, `continue`, `goto` или `return` — лишний `else` опускают.

```
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

Это типичная ситуация, когда код должен отсечь последовательность ошибочных условий. Он читается хорошо, если успешный поток управления идёт вниз по странице, отбрасывая ошибки по мере появления. Ошибочные ветки обычно заканчиваются `return`, поэтому `else` не нужен.

```
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

## Повторное объявление и присваивание

[оригинал](https://go.dev/doc/effective_go#redeclaration)

Отступление: последний пример предыдущего раздела показывает деталь того, как работает короткая форма объявления `:=`. Объявление, которое вызывает `os.Open`, выглядит так:

```
f, err := os.Open(name)
```

Этот оператор объявляет две переменные, `f` и `err`. Несколькими строками ниже вызов `f.Stat` выглядит так:

```
d, err := f.Stat()
```

кажется, будто он объявляет `d` и `err`. Заметьте, однако, что `err` есть в обоих операторах. Такое дублирование законно: `err` объявляется первым оператором, а во втором только переприсваивается. Значит, вызов `f.Stat` использует уже объявленную выше `err` и просто даёт ей новое значение.

В объявлении `:=` переменная `v` может появиться, даже если она уже объявлена, при условии что:

- это объявление находится в той же области видимости, что и существующее объявление `v` (если `v` уже объявлена во внешней области, объявление создаст новую переменную §);
- соответствующее значение в инициализации присваиваемо `v`;
- объявление создаёт хотя бы ещё одну переменную.

Это необычное свойство — чистый прагматизм: так удобно использовать одно значение `err`, например, в длинной цепочке `if-else`. Вы будете видеть это часто.

§ Здесь стоит отметить, что в Go область видимости параметров функции и возвращаемых значений совпадает с телом функции, хотя лексически они стоят вне скобок, которые это тело ограничивают.

## For

[оригинал](https://go.dev/doc/effective_go#for)

Цикл `for` в Go похож на C, но не совпадает с ним. Он объединяет `for` и `while`, а `do-while` нет. Есть три формы, и только в одной есть точки с запятой.

```
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

Короткие объявления позволяют объявить переменную индекса прямо в цикле.

```
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

Если вы итерируете массив, слайс, строку или map либо читаете из канала, цикл может вести клауза `range`.

```
for key, value := range oldMap {
    newMap[key] = value
}
```

Если нужен только первый элемент range (ключ или индекс), второй опустите:

```
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

Если нужен только второй элемент range (значение), отбросьте первый пустым идентификатором, подчёркиванием:

```
sum := 0
for _, value := range array {
    sum += value
}
```

У пустого идентификатора много применений; о них — в [отдельном разделе](12-blank-identifier.md).

Для строк `range` делает больше работы: разбирает UTF-8 и выдаёт отдельные кодовые точки Unicode. Ошибочные кодировки потребляют один байт и дают заменяющую руну U+FFFD. (Имя `rune` и связанный встроенный тип — термин Go для одной кодовой точки Unicode. Подробности — в [спецификации языка](https://go.dev/ref/spec).) Цикл

```
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

```
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

Наконец, в Go нет оператора запятой, а `++` и `--` — операторы, а не выражения. Поэтому если в `for` нужно вести несколько переменных, используйте параллельное присваивание (хотя тогда нельзя `++` и `--`).

```
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

> **Сейчас.** С Go 1.22 `range` умеет итерировать целые числа; с Go 1.23 — функции‑итераторы. См. [Go 1.22 Release Notes](https://go.dev/doc/go1.22) и [Go 1.23 Release Notes](https://go.dev/doc/go1.23).

## Switch

[оригинал](https://go.dev/doc/effective_go#switch)

`switch` в Go более общий, чем в C. Выражения не обязаны быть константами или даже целыми; ветки вычисляются сверху вниз, пока не найдётся совпадение; если у `switch` нет выражения, он переключается по `true`. Поэтому цепочку `if`-`else`-`if`-`else` можно — и идиоматично — писать как `switch`.

```
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

Автоматического провала в следующую ветку нет, но варианты можно перечислять через запятую.

```
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

Хотя в Go они встречаются куда реже, чем в некоторых других C‑подобных языках, `break` можно использовать, чтобы рано выйти из `switch`. Иногда же нужно выйти из окружающего цикла, а не из switch; в Go это делается меткой на цикле и `break` к этой метке. Пример показывает оба применения.

```
Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break
            }
            size = 1
            update(src[n])

        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

Разумеется, `continue` тоже принимает необязательную метку, но относится только к циклам.

Чтобы закрыть раздел, вот функция сравнения слайсов байт, которая использует два `switch`:

```
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

## Type switch

[оригинал](https://go.dev/doc/effective_go#type_switch)

`switch` можно использовать и чтобы узнать динамический тип переменной интерфейса. Такой type switch берёт синтаксис type assertion с ключевым словом `type` внутри скобок. Если switch объявляет переменную в выражении, в каждой ветке у неё будет соответствующий тип. Идиоматично в таких случаях переиспользовать имя: по сути в каждой ветке объявляется новая переменная с тем же именем, но другим типом.

```
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

> **Сейчас.** Для пустого интерфейса чаще пишут встроенный алиас `any`. См. [Go 1.18 Release Notes](https://go.dev/doc/go1.18).

---

[← Точки с запятой](05-semicolons.md) · [Функции →](07-functions.md)
