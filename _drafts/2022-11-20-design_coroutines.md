---
layout:     post
title:      Дизайн Kotlin Coroutines
date:       2022-11-20 12:02:00
summary:    Корутины изнутри.
categories: Android Kotlin Coroutines Error Exception
---

Перевод статьи [Design of Kotlin Coroutines](https://proandroiddev.com/design-of-kotlin-coroutines-879bd35e0f34/){:target="_blank"}

![Как вообще создаются корутины?](/images/design_coroutines/1.jpeg)

Большинство Android разработчиков используют корутины, но кто знает как создаются корутины за сценой? 

Наш примерный план такой:

1. **Определения**
2. **CPS — Continuation Passing Style**
3. **Принцип работы Kotlin корутин**
    * **3.1 Создание корутин**
        * 3.1.1 launch()
        * 3.1.2 start()
        * 3.1.3 invoke()
        * 3.1.4 startCoroutineCancellable()
        * 3.1.5 resumeWithCancellable()
        * 3.1.6 resumeWith()
        * 3.1.7 invokeSuspend()
        * 3.1.8 Выводы по созданию корутин 
    * **3.2 Анализ байткода**

<br/>
## **1. Определения**

# Что такое корутина? 
> Корутина это экземпляр приостанавливаемого блока вычислений. Концептуально похожа на поток (thread), также конкурентно запускается блок кода. Однак корутина не привязана к какому-либо потоку. Кроме того корутина может приостанавить свое выполнение в одном потоке и возобновить в другом.

В асинхронных программах задачи работают параллельно на отдельных потоках, не ожидая, пока другие задачи выполнятся. Неверное использование мультипоточности может приводить к интенсивному использованию CPU, тем самым резко снизить производительность вашего приложения. Потоки являются достаточно дорогим ресурсом. Корутины - легковесная альтернатива нативным потокам.

<br/>
# Что такое suspend функция? 

> Suspend это функция, которая может быть запущена (**_started_**), приостановлена (**_paused_**) и возобновлена (**_resumed_**). Одна из самых важных вещей о suspend функции - она может быть вызвана только из корутины или другой suspend функции.

![suspend функция](/images/design_coroutines/2.jpeg)

Когда корутина приостановлена, текущий поток на котором работала корутина становится свободным для исполнения других корутин. При этом корутина может возобновить свою работу уже на другом потоке. Как вывод, можно запускать множество корутин на небольшом количестве потоков. Ниже мы увидим, как все это работает.

<br/>
# Suspend против обычных функций:
- у suspend функций ноль и больше точек приостановки, тогда как обычные функции не имеют таких точек;
- обычные функции не могут напрямую вызывать suspend функции, так как первые не поддерживают точки приостановки;
- suspend функции могут вызывать обычные функции, потому что обычные функции имеют ноль точек приостановки.

Это простая suspend функция:

```kotlin
suspend fun functionA(): String {
     return  "hello"
}
```

Как мы говорили выше, мы не можем вызвать suspend функцию внутри обычной. Попробуем декомпилировать **functionA()**  (идем **Tools->Kotlin->Show Kotlin Bytecode**) и посмотреть, что там происходит:

```java
@Nullable
public static final Object functionA(@NotNull Continuation $completion) {
  return "hello";
}
```
![что же случилось?](/images/design_coroutines/3.jpeg)

Вам вероятно интересно откуда приходит аргумент Continuation и что он значит. Сейчас мы выясним откуда он появляется, а позже увидим, что он значит. Каждая suspend функция проходит преобразование к **CPS-Continuation Passing Style**.

Можно представить приостановку работы корутины на таком примере. Представим, что мы играем в игру и хотим приостановить (suspend) ее, а затем продолжить с этого же места. В этом случае, мы сохраняем состояние игры и когда захотим продолжить, мы можем восстановить из сохранения последнюю приостановленную точку. Когда процесс приостановлен, он возвращает объект Continuation. 

<br/>

## **2. CPS — Continuation Passing Style**






















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

В обычной функции исключение пробрасывается (_re-thrown_). Это значит, что мы можем перехватить исключения блоком `try-catch` для их обработки в месте вызова такой функции:

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

Как мы увидели в начале, не перехваченное исключение "пробрасывается" выше в обычной функции. В случае корутины это не работает. Иначае, мы бы могли обработать исключение во внешнем `try-catch` блоке и приложение с примером выше не падало.

Тогда, что происходит с не перехваченным исключением в корутине? Вероятно вы знаете, одна из самых крутых штук в корутинах это _Структурная Конкурентность_ ([_Structured Concurrency_](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency){:target="_blank"}). Для работы всех возможностей _Structured Concurrency_, объект `Job` в `CoroutineScope` и объекты `Job` в корутинах и дочерние корутины образуют иерархию вида "родитель-ребенок". Любое не перехваченное исключение, вместо того, чтобы быть проброшенным, распространяется вверх по иерарахии `Job` объектов. Такое распространение приводит к падению родительской `Job` и ведет к отказу всех дочерних `Job`.

Для примера выше иерархия `Job` выглядит так:
![coroutines hierarchy](/images/why-exception-handling-with/2.webp)

Исключение из дочерней корутины распространяется выше до объекта `Job` корутины верхнего уровня (1) и затем дальше до объекта `Job` в `topLevelScope` (2).

![coroutines exceptions propagation](/images/why-exception-handling-with/3.webp)

Исключения могут быть перехвачены через установку обработчика `CoroutineExceptionHandler`. Если не установленного ни одного, вызывается обработчик не перехваченных исключений конкретного потока, который зависит от платформы и скорее всего напечатает исключения в стандартный вывод и затем завершит приложение.

По-моему мнению, у нас фактически два разных механизма для обработки исключений – `try-catch` и `CoroutineExceptionHandlers`, это один из главных факторов, почему обработка исключений в корутинах такая сложная. 

## Ключевая особенность #1
Если корутина не обрабатывает исключения сама блоком `try-catch`, исключение не пробрасывается и таким образом не сможет быть обработана внешним `try-catch`. Вместо этого, исключение "распространяется по иерархии корутин (`Job`)" и может быть обработана специально установленным `CoroutineExceptionHandler`. Если таковых нет, необработанное исключение попадает в обработчик не перехваченных исключений потока.

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
<br/>
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
<br/>

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

Здесь есть еще один аспект - если обрабатывать исключения напрямую в корутине с `try-catch`, то это ломает концепцию отмены корутин в _Structured Concurrency_. К примеру, давайте представим, что мы запустили две корутины параллельно. Они обе как-то зависят друг от друга таким образом, что завершение одной не имеет смысла, если другая падает. При использовании `try-catch` для обработки исключений в каждой корутине, сами исключения не будут распространяться выше к родителю и поэтому другая корутина не будет отменяться. Это тратит ресурсы впустую. В таких ситуациях необходимо использовать `CoroutineExceptionHandler`.

<br/>

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

В этом примере воообще ничего не выводится. Что здесь происходит с `RuntimeException`? Исключение игнорируется? Нет. В корутинах запущенных через `async` , необработанные исключения также немедленно распространяются вверх по иерархии корутин. Но в отличие от `launch`, исключения не обрабатываются установленным `CoroutineExceptionHandler` и не передаются в обработчик потока для необработанных исключений.

Функция `launch` для запуска корутин возвращает экземпляр `Job`, это простое представление корутины которая не возвращает значение. В случае если мы хотим чтобы корутина что-то возвращала, мы должны использовать функцию `async`, которая возвращает объект `Deferred`, специальный подтип `Job` с результатом. Если `async` корутина падает, то исключение оборачивается в возвращаемое значение `Deferred` и пробрасывается, когда мы вызываем `await` у корутины для получения результата. 

Поэтому, мы можем обернуть `.await()` в блок `try-catch`. Так как `.await()` это _suspend_ функция, мы должны запустить новую корутину, чтобы можно было ее вызвать:

```kotlin
fun main() {

    val topLevelScope = CoroutineScope(SupervisorJob())

    val deferredResult = topLevelScope.async {
        throw RuntimeException("RuntimeException in async coroutine")
    }

    topLevelScope.launch {
        try {
            deferredResult.await()
        } catch (exception: Exception) {
            println("Handle $exception in try/catch")
        }
    }

    Thread.sleep(100)
}

// Output: 
// Handle java.lang.RuntimeException: RuntimeException in async coroutine in try/catch
``` 
<br/>
Внимание: исключение оборачивается в `Deferred`, только для `async` корутин верхнего уровня. В противном случае оно немедленно распространяется вверх по иерархии `Job` и перехватывается или `CoroutineExceptionHandler` или передается в обработчик необработанных ошибок в потоке, это происходит даже без вызова метода `.await()`, как в примере ниже:

```kotlin
fun main() {
  
    val coroutineExceptionHandler = CoroutineExceptionHandler { coroutineContext, exception ->
        println("Handle $exception in CoroutineExceptionHandler")
    }
    
    val topLevelScope = CoroutineScope(SupervisorJob() + coroutineExceptionHandler)
    topLevelScope.launch {
        async {
            throw RuntimeException("RuntimeException in async coroutine")
        }
    }
    Thread.sleep(100)
}

// Output
// Handle java.lang.RuntimeException: RuntimeException in async coroutine in CoroutineExceptionHandler
```
<br/>

## Ключевая особенность #4
Необработанные исключения для `launch` и `async` мгновенно распростаняются по иерархии `Job`. Однако, если верхнеуровневая корутина была запущена через `launch`, то исключение обрабатывается `CoroutineExceptionHandler` или передается в обработчик необработанных исключений в потоке. С другой стороны, при запуске верхнеуровневой корутины через `async`, исключение оборачивается в возвращаемый объект `Deferred` и пробрасывается, когда вызывается его метод `.await()`

---

<br/>

## Особенности обработки исключений в coroutineScope{}

Когда в начале статьи мы разговаривали об использовании `try-catch` в корутинах, я сказал вам, что падающая корутина распространяет свое исключение по иерархии `Job` вместо проброса и таким образом, блок `try-catch` не работает.

Однако, когда мы оборачиваем падающую корутину в _scope_ функцию `coroutineScope{}`, происходит нечто интересное:

```kotlin
fun main() {
    
  val topLevelScope = CoroutineScope(Job())
    
  topLevelScope.launch {
        try {
            coroutineScope {
                launch {
                    throw RuntimeException("RuntimeException in nested coroutine")
                }
            }
        } catch (exception: Exception) {
            println("Handle $exception in try/catch")
        }
    }

    Thread.sleep(100)
}

// Output 
// Handle java.lang.RuntimeException: RuntimeException in nested coroutine in try/catch
```
<br/>
Теперь мы можем обрабатывать исключения в `try-catch`. _Scope_ функция `coroutineScope{}` пробрасывается исключения из своих дочерних корутин вместо распространения по иерарахии `Job`.

`coroutineScope{}` используется в основном в _suspend_ функциях для "параллельной декомпозиции". Эти _suspend_ функции будут пробрасывать исключения из своих корутин и таким образом можно будет организовать обработку исключений.

## Ключевая особенность #5

_scope_ функция `coroutineScope{}` пробрасывает исключения от дочерних корутин, вместо распространия по иерархии `Job`. Это дает нам обработку ошибок через `try-catch`.

____
<br/>

## Обработка исключений в supervisorScope{}

Когда мы используем _scope_ функцию `supervisorScope{}`, устанавливается новый, отдельный, вложенный _scope_ с типом `SupervisorJob` в нашей Job иерархии.
Что-то вроде этого...

```kotlin
fun main() {

    val topLevelScope = CoroutineScope(Job())

    topLevelScope.launch {
        val job1 = launch {
            println("starting Coroutine 1")
        }

        supervisorScope {
            val job2 = launch {
                println("starting Coroutine 2")
            }

            val job3 = launch {
                println("starting Coroutine 3")
            }
        }
    }

    Thread.sleep(100)
}
```
… создает иерархию корутин:
![coroutines](/images/why-exception-handling-with/4.webp)


Здесь важно понимать, что `supervisorScope` это новый, отдельный вложенный _scope_, который должен обрабатывать исключения сам. `supervisorScope` не пробрасывает исключения своих внутренних корутин (как это делает `coroutineScope`), и не передает исключения родительской `Job` (в примере это `topLevelScope`).

Еще одна важная вещь - исключения распространяются вверх по иерархии, пока не дойдут до _scope_ верхнего уровня или `SupervisorJob`. Это значит, что в примере выше: "Coroutine 2" и "Coroutine 3", это корутины верхнего уровня.

Также мы можем установить `CoroutineExceptionHandler` отдельно для "Coroutine 2" (или "Coroutine 3"):

```kotlin
fun main() {

    val coroutineExceptionHandler = CoroutineExceptionHandler { coroutineContext, exception ->
        println("Handle $exception in CoroutineExceptionHandler")
    }

    val topLevelScope = CoroutineScope(Job())

    topLevelScope.launch {
        val job1 = launch {
            println("starting Coroutine 1")
        }

        supervisorScope {
            val job2 = launch(coroutineExceptionHandler) {
                println("starting Coroutine 2")
                throw RuntimeException("Exception in Coroutine 2")
            }

            val job3 = launch {
                println("starting Coroutine 3")
            }
        }
    }

    Thread.sleep(100)
}

// Output
// starting Coroutine 1
// starting Coroutine 2
// Handle java.lang.RuntimeException: Exception in Coroutine 2 in CoroutineExceptionHandler
// starting Coroutine 3

```
<br/>
Так как корутины в `supervisorScope` это корутины верхнего уровня, это значит, что `async` корутины теперь оборачивают свои исключения в Deferred объекты...

```kotlin
// ... other code is identical to example above
supervisorScope {
    val job2 = async {
        println("starting Coroutine 2")
        throw RuntimeException("Exception in Coroutine 2")
    }

// ...

// Output: 
// starting Coroutine 1
// starting Coroutine 2
// starting Coroutine 3
```
... и будут проброшены при вызове `.await()`


## Ключевая особенность #6
_scope_ функция `supervisorScope{}` создает новый независимый _scope_ с типом `SupervisorJob` в иерархии `Job`. Этот новый _scope_ не распространяет свои исключения "вверх по иерархии", обработку ошибок он должен выполнять самостоятельно. Корутины запускаемые из `supervisorScope` являются корутинами верхнего уровня. Корутины верхнего уровня ведут себя иначе, чем дочерние корутины, при запуске через `launch()` или `async()`. Кроме того, в корутины верхнего уровня можно установить обработчики исключений `CoroutineExceptionHandlers`.

---

<br/>
## Все! 
Я вывел эти ключевые моменты, пытаясь понять обработку исключений в корутинах.






















