# Интерфейсы и другие типы

[оригинал](https://go.dev/doc/effective_go#interfaces_and_types)

## Интерфейсы

[оригинал](https://go.dev/doc/effective_go#interfaces)

Интерфейсы в Go задают поведение объекта: если нечто умеет это, его можно использовать здесь. Мы уже видели пару простых примеров: свои принтеры можно реализовать методом `String`, а `Fprintf` может писать куда угодно, у чего есть метод `Write`. Интерфейсы с одним‑двумя методами в коде на Go обычны и обычно получают имя от метода, например `io.Writer` для того, что реализует `Write`.

Тип может реализовывать несколько интерфейсов. Например, коллекцию можно сортировать процедурами пакета `sort`, если она реализует `sort.Interface`, в котором есть `Len()`, `Less(i, j int) bool` и `Swap(i, j int)`, и при этом у неё может быть свой форматтер. В этом надуманном примере `Sequence` удовлетворяет обоим.

```
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy returns a copy of the Sequence.
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    s = s.Copy() // Make a copy; don't overwrite argument.
    sort.Sort(s)
    str := "["
    for i, elem := range s { // Loop is O(N²); will fix that in next example.
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

> **Сейчас.** Для сравнения и упорядочивания произвольных типов есть дженерики и пакеты [`cmp`](https://pkg.go.dev/cmp), [`slices`](https://pkg.go.dev/slices). Введение — в [Tutorial: generics](https://go.dev/doc/tutorial/generics).

## Преобразования

[оригинал](https://go.dev/doc/effective_go#conversions)

Метод `String` у `Sequence` заново делает работу, которую `Sprint` уже умеет для слайсов. (У него ещё сложность O(N²), что плохо.) Можно разделить усилия (и ускорить) если перед вызовом `Sprint` преобразовать `Sequence` в простой `[]int`.

```
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

Этот метод — ещё один пример техники преобразования для безопасного вызова `Sprintf` из метода `String`. Поскольку два типа (`Sequence` и `[]int`) совпадают, если игнорировать имя типа, преобразование между ними законно. Преобразование не создаёт новое значение, оно лишь временно ведёт себя так, будто у существующего значения новый тип. (Есть и другие законные преобразования, например из целого в число с плавающей точкой, которые новое значение создают.)

В программах на Go идиоматично преобразовывать тип выражения, чтобы получить доступ к другому набору методов. Например, можно использовать существующий тип `sort.IntSlice` и свести весь пример к этому:

```
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

Теперь вместо того чтобы `Sequence` реализовывал несколько интерфейсов (сортировка и печать), мы используем способность данного быть преобразованным к нескольким типам (`Sequence`, `sort.IntSlice` и `[]int`), каждый из которых делает свою часть работы. На практике это менее обычно, но может быть эффективно.

## Преобразования интерфейсов и утверждения типа

[оригинал](https://go.dev/doc/effective_go#interface_conversions)

Type switch — форма преобразования: он берёт интерфейс и для каждой ветки в некотором смысле преобразует его к типу этой ветки. Вот упрощённая версия того, как код под `fmt.Printf` превращает значение в строку через type switch. Если это уже строка, нам нужно фактическое строковое значение, которое держит интерфейс; если есть метод `String` — результат вызова метода.

```
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

Первая ветка находит конкретное значение; вторая преобразует интерфейс в другой интерфейс. Смешивать типы так совершенно нормально.

А если важен только один тип? Если мы знаем, что значение держит `string`, и просто хотим его извлечь? Подойдёт type switch с одной веткой, но подойдёт и утверждение типа. Утверждение типа берёт значение интерфейса и извлекает из него значение указанного явного типа. Синтаксис заимствован у клаузы, открывающей type switch, но с явным типом вместо ключевого слова `type`:

```
value.(typeName)
```

и результат — новое значение со статическим типом `typeName`. Этот тип должен быть либо конкретным типом, который держит интерфейс, либо вторым интерфейсным типом, к которому значение можно преобразовать. Чтобы извлечь строку, которая, как мы знаем, есть в значении, можно написать:

```
str := value.(string)
```

Но если окажется, что значения‑строки нет, программа упадёт с ошибкой времени выполнения. Чтобы защититься, используйте идиому «comma, ok» и безопасно проверьте, является ли значение строкой:

```
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

Если утверждение типа не удалось, `str` всё равно будет существовать и иметь тип string, но получит нулевое значение — пустую строку.

Как иллюстрация возможностей, вот оператор `if`-`else`, эквивалентный type switch, которым открывался этот раздел.

```
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## Общность

[оригинал](https://go.dev/doc/effective_go#generality)

Если тип существует только чтобы реализовать интерфейс и никогда не будет иметь экспортируемых методов сверх этого интерфейса, сам тип экспортировать не нужно. Экспорт только интерфейса делает ясным, что у значения нет интересного поведения сверх описанного в интерфейсе. Это также избавляет от нужды повторять документацию на каждом экземпляре общего метода.

В таких случаях конструктор должен возвращать значение интерфейса, а не реализующий тип. Например, в библиотеках хешей и `crc32.NewIEEE`, и `adler32.New` возвращают интерфейсный тип `hash.Hash32`. Чтобы в программе на Go подменить алгоритм CRC-32 на Adler-32, достаточно сменить вызов конструктора; остальной код смена алгоритма не затрагивает.

Похожий подход позволяет потоковым шифрам в различных пакетах `crypto` быть отделёнными от блочных шифров, которые они сцепляют. Интерфейс `Block` в пакете `crypto/cipher` задаёт поведение блочного шифра, который шифрует один блок данных. Затем, по аналогии с пакетом `bufio`, пакеты шифров, реализующие этот интерфейс, можно использовать для построения потоковых шифров, представленных интерфейсом `Stream`, не зная деталей блочного шифрования.

Интерфейсы `crypto/cipher` выглядят так:

```
type Block interface {
    BlockSize() int
    Encrypt(dst, src []byte)
    Decrypt(dst, src []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

Вот определение потокового режима счётчика (CTR), который превращает блочный шифр в потоковый; заметьте, что детали блочного шифра абстрагированы:

```
// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

`NewCTR` применим не к одному конкретному алгоритму и источнику данных, а к любой реализации интерфейса `Block` и любому `Stream`. Поскольку они возвращают значения интерфейса, замена шифрования CTR другими режимами — локальное изменение. Вызовы конструкторов нужно править, но окружающий код должен относиться к результату только как к `Stream` и разницы не заметит.

## Интерфейсы и методы

[оригинал](https://go.dev/doc/effective_go#interface_methods)

Поскольку методы можно повесить почти на что угодно, почти что угодно может удовлетворить интерфейс. Показательный пример — в пакете `http`, который определяет интерфейс `Handler`. Любой объект, реализующий `Handler`, может обслуживать HTTP‑запросы.

```
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`ResponseWriter` сам по себе интерфейс, который даёт доступ к методам, нужным чтобы вернуть ответ клиенту. Среди этих методов — стандартный `Write`, так что `http.ResponseWriter` можно использовать везде, где можно `io.Writer`. `Request` — структура с разобранным представлением запроса от клиента.

Для краткости проигнорируем POST и предположим, что HTTP‑запросы всегда GET; это упрощение не влияет на то, как настраиваются обработчики. Вот тривиальная реализация обработчика, который считает, сколько раз страницу посетили.

```
// Simple counter server.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

(В духе нашей темы заметьте, как `Fprintf` может печатать в `http.ResponseWriter`.) В настоящем сервере доступ к `ctr.n` нужно защитить от одновременного доступа. Подсказки — в пакетах `sync` и `atomic`.

Для справки — как привязать такой сервер к узлу дерева URL.

```
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

Но зачем делать `Counter` структурой? Достаточно целого. (Получатель должен быть указателем, чтобы инкремент был виден вызывающему.)

```
// Simpler counter server.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

А если в программе есть внутреннее состояние, которое нужно уведомить, что страницу посетили? Привяжите канал к веб‑странице.

```
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

Наконец, допустим, мы хотим показать на `/args` аргументы, с которыми вызвали двоичный файл сервера. Легко написать функцию, которая печатает аргументы.

```
func ArgServer() {
    fmt.Println(os.Args)
}
```

Как превратить это в HTTP‑сервер? Можно сделать `ArgServer` методом какого‑нибудь типа, значение которого мы игнорируем, но есть чище способ. Поскольку метод можно определить для любого типа, кроме указателей и интерфейсов, можно написать метод для функции. В пакете `http` есть такой код:

```
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

`HandlerFunc` — тип с методом `ServeHTTP`, поэтому значения этого типа могут обслуживать HTTP‑запросы. Посмотрите на реализацию метода: получатель — функция `f`, и метод вызывает `f`. Это может казаться странным, но не так уж отличается от того, что получатель — канал, а метод в него отправляет.

Чтобы сделать `ArgServer` HTTP‑сервером, сначала изменим его, чтобы была правильная сигнатура.

```
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

Теперь у `ArgServer` та же сигнатура, что у `HandlerFunc`, так что его можно преобразовать к этому типу, чтобы получить доступ к его методам, как мы преобразовывали `Sequence` в `IntSlice`, чтобы получить `IntSlice.Sort`. Код настройки лаконичен:

```
http.Handle("/args", http.HandlerFunc(ArgServer))
```

Когда кто‑то посещает страницу `/args`, у обработчика, установленного на этой странице, значение `ArgServer` и тип `HandlerFunc`. HTTP‑сервер вызовет метод `ServeHTTP` этого типа, с `ArgServer` в качестве получателя, который в свою очередь вызовет `ArgServer` (через вызов `f(w, req)` внутри `HandlerFunc.ServeHTTP`). Затем отобразятся аргументы.

В этом разделе мы сделали HTTP‑сервер из структуры, целого, канала и функции — всё потому, что интерфейсы это просто наборы методов, а методы можно определить для (почти) любого типа.

---

[← Методы](10-methods.md) · [Пустой идентификатор →](12-blank-identifier.md)
