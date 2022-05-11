---
layout:     post
title:      Почему исключения в Kotlin Coroutines это сложно и как с этим жить?
date:       2022-05-06 12:02:00
summary:    Учимся как обрабатывать исключения в корутинах.
categories: Android Kotlin Coroutines Error Exception
---

Перевод статьи [Why exception handling with Kotlin Coroutines is so hard and how to successfully master it!](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/){:target="_blank"}.

![Header](/images/why-exception-handling-with/1.webp)

Обработка исключений, вероятно одна из самых сложных частей, когда вы изучаете корутины в Kotlin. В этой статье, я расскажу о причинах такой сложности и объясню некоторые ключевые моменты для хорошего понимания темы. После этого вы сможете реализовать правильную инфраструктуру для обработки ошибок в своем собственном приложении.

> Информация: вы можете посмотреть более полное видео об обработке исключений [здесь](https://www.youtube.com/watch?v=Pgek3_3vPU8){:target="_blank"}.


## Обработка исключений в "чистом" Kotlin

В Kotlin без корутин обрабатывать исключения достаточно просто. Для этого используются блоки обработки `try-catch`:

```kotlin
try {
    // some code
    throw RuntimeException("RuntimeException in 'some code'")
} catch (exception: Exception) {
    println("Handle $exception")
}

// Output:
// Handle java.lang.RuntimeException: RuntimeException in 'some code'
```

Если в обычной функции выбросится исключение, оно пробрасывается (_re-thrown_). Это значит, что мы можем перехватить исключения блоком `try-catch` для их обработки в месте вызова такой функции:

```kotlin
fun main() {
    try {
        functionThatThrows()
    } catch (exception: Exception) {
        println("Handle $exception")
    }
}

fun functionThatThrows() {
    // some code
    throw RuntimeException("RuntimeException in regular function")
}

// Output
// Handle java.lang.RuntimeException: RuntimeException in regular function
```

## try-catch в корутинах

Теперь давайте посмотрим, как использовать `try-catch` в котлиновских корутинах. Внутри корутины (которая стартует при помощи функции `launch` в примере ниже) `try-catch` работает штатно, исключение перехватывается:

```kotlin
fun main() {
    val topLevelScope = CoroutineScope(Job())
    topLevelScope.launch {
        try {
            throw RuntimeException("RuntimeException in coroutine")
        } catch (exception: Exception) {
            println("Handle $exception")
        }
    }
    Thread.sleep(100)
}

// Output
// Handle java.lang.RuntimeException: RuntimeException in coroutine
```

Но если запустить другую корутину внутри блока `try-catch`...

```kotlin
fun main() {
    val topLevelScope = CoroutineScope(Job())
    topLevelScope.launch {
        try {
            launch {
                throw RuntimeException("RuntimeException in nested coroutine")
            }
        } catch (exception: Exception) {
            println("Handle $exception")
        }
    }
    Thread.sleep(100)
}

// Output
// Exception in thread "main" java.lang.RuntimeException: RuntimeException in nested coroutine
```

... видно, что исключение больше не обрабатывается и приложение падает. Это весьма неожиданно и сбивает с толку. Если основываться на наших знаниях и опыте работы с `try-catch`, мы ожидаем, что каждое исключение обернутое блоком `try-catch` перехватывается и передается в ветку `catch`. Почему здесь это не происходит?
































