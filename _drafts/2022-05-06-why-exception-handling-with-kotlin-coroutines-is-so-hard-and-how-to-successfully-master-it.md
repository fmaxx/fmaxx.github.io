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

... видно, что исключение больше не обрабатывается и приложение падает. Это весьма неожиданно и сбивает с толку. Если основываться на наших знаниях и опыте работы с `try-catch`, мы ожидаем, что каждое исключение обернутое блоком `try-catch` перехватывается и передается в ветку `catch`. Почему здесь это не срабатывает?

Хорошо, если корутина не перехватывает исключение самостоятельно через блок `try-catch`, она "завершается по необработанному исключению" или, простыми словами, она падает. В примере выше, внутренняя корутина запущенная через `launch` не перехватывает `RuntimeException` сама и поэтому падает.

Как мы увидели в начале, неперехваченное исключение "пробрасывается" выше в обычной функции. В случае корутины это не работает. Иначае, мы бы могли обработать исключение во внешнем `try-catch` блоке и приложение с примером выше не падало.

Тогда, что происходит с неперехваченным исключением в корутине? Вероятно вы знаете, одна из самых крутых штук в корутинах это _Структурная Конкурентность_ ([_Structured Concurrency_](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency){:target="_blank"}). Для работы всех возможностей _Structured Concurrency_, объект `Job` в `CoroutineScope` и объекты `Job` в корутинах и дочерние корутины образуют иерархию вида "родитель-ребенок". Любое неперхваченное исключение, вместо того, чтобы быть проброшенным выше, распространяется вверх по иерарахии `Job` объектов. Такое распространение приводит к падению родительской `Job` и ведет к отказу всех дочерних `Job`.

Для примера выше иерархия `Job` выглядит так:
![coroutines hierarchy](/images/why-exception-handling-with/2.webp)

Исключение из дочерней корутины распространяется выше до объекта `Job` корутины верхнего уровня (1) и затем дальше до объекта `Job` в `topLevelScope` (2).

![coroutines exceptions propagation](/images/why-exception-handling-with/3.webp)

Исключения могут быть перехвачены через установку обработчика `CoroutineExceptionHandler`. Если не установленного ни одного, вызывается обработчик неперхваченных исключений конкретного потока, который зависит от платформы и скорее всего напечатает исключения в стандартый вывод и затем завершит приложение.

По-моему мнению, у нас фактически два разных механизма для обработки исключений – `try-catch` и `CoroutineExceptionHandlers`, это один из главных факторов, почему обработка исключений в корутинах такая сложная. 

## Ключевая особенность #1
Если корутина не обрабатывает исключения сама блоком `try-catch`, исключение не пробрасывается и таким образом не сможет быть обработана внешним `try-catch`. Вместо этого, исключение "распространяется по иерархии корутин (`Job`)" и может быть обработана специально установленным `CoroutineExceptionHandler`. Если таковых нет, необработанное исключение попадает в обработчик неперехваченных исключений потока.

## Обработчик CoroutineExceptionHandler

Ладно, сейчас мы знаем, что `try-catch` блок бесполезен если мы стартуем корутину с исключением в `try` ветке. Давайте вместо этого установим `CoroutineExceptionHandler`! Мы можем передать контекст в функцию-билдер корутин `launch`. Так как `CoroutineExceptionHandler` это `ContextElement`, его можно установить как единственный аргумент в `launch`, при запуске нашей дочерней корутины:

```kotlin
fun main() {
  
    val coroutineExceptionHandler = CoroutineExceptionHandler { coroutineContext, exception ->
        println("Handle $exception in CoroutineExceptionHandler")
    }
    
    val topLevelScope = CoroutineScope(Job())

    topLevelScope.launch {
        launch(coroutineExceptionHandler) {
            throw RuntimeException("RuntimeException in nested coroutine")
        }
    }

    Thread.sleep(100)
}

// Output
// Exception in thread "DefaultDispatcher-worker-2" java.lang.RuntimeException: RuntimeException in nested coroutine
``` 
Хм, тем не менее, наше исключение не обрабатывается в `coroutineExceptionHandler` и приложение падает! Это потому что установка `CoroutineExceptionHandler` в дочерние корутины не имеет никакого эффекта. Мы должны установить обработчик в _scope_ или в корутину верхнего уровня, таким образом:

```kotlin
// ...
val topLevelScope = CoroutineScope(Job() + coroutineExceptionHandler)
// ...
```

или что-то похожее на:

```kotlin
// ...
topLevelScope.launch(coroutineExceptionHandler) {
// ...
```

И только тогда, наш обработчик ошибок сработает:

```kotlin
// ..
// Output: 
// Handle java.lang.RuntimeException: RuntimeException in nested coroutine in CoroutineExceptionHandler
```

## Ключевая особенность #2

Чтобы `CoroutineExceptionHandler` сработал, надо его устанавливать или в `CoroutineScope` или в корутинах верхнего уровня.

## try-catch VS CoroutineExceptionHandler

Как вы уже поняли, у нас два варианта для обработки исключений: 
1. оборачиваем код внутри корутины блоком `try-catch`,
2. установка обработчика `CoroutineExceptionHandler`.

Какой вариант следует выбирать? 

У официальной документации [`CoroutineExceptionHandler`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/){:target="_blank"}) есть хорошие ответы на это:

> `CoroutineExceptionHandler` это последнее средство для глобального перехвата всех исключений. Нельзя восстановится после обработки исключения в `CoroutineExceptionHandler`. Корутина к этому времени уже завершилась с соответствующим исключением (когда вызвался обработчик `CoroutineExceptionHandler`). Как правило, такой обработчик используется для логгирования исключений, сообщений об ошибках, рестарах приложения.

> Если вам надо обрабатывать исключение в отдельной части кода, рекомендуется использовать `try-catch` блок вокруг вашего кода внутри корутины. Таким образом, вы можете предотвратить завершение корутины через ошибку (теперь исключение обрабатывается), повторить операцию, и/или предпринять любые другие действия.

Здесь есть еще один аспект - если обрабатывать исключения напрямую в корутине с `try-catch`, то это ломает концепцию отмены корутин в _Structured Concurrency_. К примеру, давайте представим, что мы запустили две корутины параллельно. Они обе как-то зависят друг от друга таким образом, что завершение одной не имеет смысла, если другая падает. При использовании `try-catch` для обработки исключений в каждой корутине, сами исключения не будут распространяться выше к родителю и поэтому другая корутина не будет отменяться. Это тратит ресурсы впустую. В таких сиутациях необходимо использовать `CoroutineExceptionHandler`.

## Ключевая особенность #3

Используйте `try-catch` если вы хотите как-то восстановиться (повтор или другие операции), до того как корутина закончится. Помните, что перехваченное исключение не распространяется выше по иерархии корутин и функционал отмены для _Structured Concurrency_ не работает в этом случае. `CoroutineExceptionHandler` применяйте для логики работающей после того, как корутина завершена.

## launch{} vs async{}

До этого момента, мы использовали только билдер-функцию `launch` для запуска новых корутин. Однако, обработка исключений немного отличается между корутинами запущенными через `launch` и `async`. Посмотрите на следующий пример:

```kotlin
fun main() {

    val topLevelScope = CoroutineScope(SupervisorJob())

    topLevelScope.async {
        throw RuntimeException("RuntimeException in async coroutine")
    }

    Thread.sleep(100)
}

// No output
```

В этом примере воообще ничего не выводится. Что же здесь происходит с `RuntimeException`? Исключение игнорируется? Нет. В корутинах запущенных через `async` , необработанные исключения также немедленно распространяются вверх по иерархии корутин. Но в отличие от `launch`, исключения не обрабатываются установленным `CoroutineExceptionHandler` и не передаются в обработчик потока для необработанных исключений.




The return type of a Coroutine started with launch is Job, which is simply a representation of the Coroutine without a return value. If we need some result from a Coroutine, we have to use async, which returns a Deferred, which is a special kind of Job that additionally holds a result value. If the async Coroutine fails, the exception is encapsulated in the Deferred return type, and is re-thrown when we call the suspend function .await() to retrieve the result value on it.



























