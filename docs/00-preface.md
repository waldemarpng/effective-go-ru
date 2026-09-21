# Предисловие к переводу

Это неофициальный перевод [Effective Go](https://go.dev/doc/effective_go).

## Официальная пометка

В начале оригинала стоит такое предупреждение:

> This document was written for Go's release in 2009 and is not actively updated. While it remains a good guide for using the core language, it does not cover significant changes to the language (generics), ecosystem (modules), or libraries added since. See [issue 28782](https://github.com/golang/go/issues/28782) for context. For a complete list of changes, see the [release notes](https://go.dev/doc/devel/release).

Перевод:

> Этот документ написан к релизу Go 2009 года и активно не обновляется. Он по-прежнему хороший гид по ядру языка, но не покрывает существенные изменения языка (дженерики), экосистемы (модули) и библиотек, появившихся позже. Контекст — [issue 28782](https://github.com/golang/go/issues/28782). Полный список изменений — в [заметках о релизах](https://go.dev/doc/devel/release).

В [issue 28782](https://github.com/golang/go/issues/28782) Rob Pike предложил заморозить Effective Go как «капсулу времени»: не раздувать его современными практиками и библиотеками, а оставить документ в своём голосе. Отдельного официального «нового Effective Go» нет.

## Как устроен этот перевод

Основной текст — верный перевод оригинала. Мы не подменяем старые примеры современными.

Если совет устарел или тема в оригинале отсутствует, рядом стоит пометка:

> **Сейчас.** Краткая суть и ссылка на актуальную страницу на go.dev.

У каждого заголовка есть ссылка **оригинал** на ту же секцию.

Сначала стоит прочитать то, на что указывает и сам оригинал:

- [A Tour of Go](https://go.dev/tour/)
- [How to Write Go Code](https://go.dev/doc/code)
- [The Go Programming Language Specification](https://go.dev/ref/spec)

## Чего в оригинале нет

Эти темы в Effective Go не разобраны. Их не выносим в отдельные главы перевода — только ссылки на официальную документацию.

- **Модули** — [How to Write Go Code](https://go.dev/doc/code), [Using Go Modules](https://go.dev/blog/using-go-modules), [Managing dependencies](https://go.dev/doc/modules/managing-dependencies), [Go Modules Reference](https://go.dev/ref/mod)
- **Дженерики** — [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics), [Tour: Generics](https://go.dev/tour/generics/1)
- **Ошибки после Go 1.13** — [`errors`](https://pkg.go.dev/errors) (`Is`, `As`, `Join`), [Working with Errors in Go 1.13](https://go.dev/blog/go1.13-errors)
- **context** — [`context`](https://pkg.go.dev/context), [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- **Пакеты `slices` и `maps`** — [`slices`](https://pkg.go.dev/slices), [`maps`](https://pkg.go.dev/maps)
- **Fuzzing, coverage, workspaces** — [Fuzzing](https://go.dev/doc/security/fuzz/), [Coverage](https://go.dev/doc/build-cover), [Multi-module workspaces](https://go.dev/doc/tutorial/workspaces)
- **Список изменений языка и стандартной библиотеки** — [Release History](https://go.dev/doc/devel/release)

---

[Введение →](01-introduction.md)
