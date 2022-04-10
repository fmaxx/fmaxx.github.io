---
layout:     post
title:      Android, обработка ошибок часть 1, как исключения работают в JVM и приложениях Android.
date:       2022-04-07 12:02:00
summary:    Как исключения работают в JVM и приложениях Android.
categories: Android Kotlin Error Exception
---

Перевод статьи [Error handling on Android part 1: how exceptions work for JVM and Android apps](https://www.bugsnag.com/blog/error-handling-on-android-part-1){:target="_blank"}.

* Часть 1: Как исключения работают в JVM и приложениях Android
* Часть 2: Реализация UncaughtExceptionHandler в приложении JVM
* Часть 3: Отправка креш-отчетов по API
* Часть 4: Обработка нефатальных ошибок в Android
* Часть 5: Особенности обфускации и минимизации в Android креш-отчетов
* Часть 6: Захват сигналов и исключений в Android NDK приложениях

















Как компилятор преобразует _suspend_ код, чтобы корутины можно было приостанавливать и возобновлять?

Корутины в Kotlin представлены ключевым словом **_suspend_**. Интересно, что там происходит внутри? Как компилятор преобразует _suspend_ блоки в код, поддерживающий приостановку и возобновление работы корутины?

Знание этого поможет понимать, почему suspend функция не возвращает управление, пока не завершится вся запущенная работа и как код может приостановить выполнение без блокировки потоков. 

> TL;DR; Компилятор Kotlin создает специальную машину состояний для каждой suspend функции, эта машина берет управление корутиной на себя!

Новенький в Android? Взгляни на эти полезные ресурсы по корутинам:
- [Using coroutines in your Android app](https://codelabs.developers.google.com/codelabs/kotlin-coroutines/#0){:target="_blank"}.
- [Advanced Coroutines with Kotlin Flow and Live Data](https://developer.android.com/codelabs/advanced-kotlin-coroutines#0){:target="_blank"}.

Для тех, кто предпочитает видео:

<iframe width="840" height="473" src="https://www.youtube.com/embed/IQf-vtIC-Uc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<br/>
# Корутины, краткое введение
Говоря по-простому, корутины это асинхронные операции в Android. Как описано в [документации](https://developer.android.com/kotlin/coroutines){:target="_blank"}, мы можем использовать корутины для управления асинхронными задачами, которые иначе могут блокировать основной поток и приводить к зависанию UI приложения.

Также корутины удобно использовать для замены callback-кода на императивный код. Например, посмотрите на этот код с использованием колбеков:
```kotlin
// Simplified code that only considers the happy path
fun loginUser(userId: String, password: String, userResult: Callback<User>) {
  // Async callbacks
  userRemoteDataSource.logUserIn { user ->
    // Successful network request
    userLocalDataSource.logUserIn(user) { userDb ->
      // Result saved in DB
      userResult.success(userDb)
    }
  }
}
```

Заменяем эти колбекина последовательные вызовы функций с использованием корутин:
```kotlin
suspend fun loginUser(userId: String, password: String): UserDb {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}
```
Для функций, которые вызываются в корутинах, мы добавили ключевое слово **suspend**. Так компилятор знает, что эти функции для корутин. С точки зрения разработчика, рассматривайте suspend функцию как обычную, выполнение которой может быть приостановлено и возобновлено в определенный момент.

В отличие от колбеков, корутины предлагают простой способ переключения между потоками и обработки исключений.

Но что в действительности делает компилятор внутри, когда мы отмечаем функцию как _suspend_?

## Suspend под капотом
Давайте вернемся к suspend функции `loginUser`, посмотрите, другие функции которые она вызывает являются также suspend функциями:

```kotlin
suspend fun loginUser(userId: String, password: String): UserDb {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}

// UserRemoteDataSource.kt
suspend fun logUserIn(userId: String, password: String): User

// UserLocalDataSource.kt
suspend fun logUserIn(userId: String): UserDb
```

Кратко говоря, компилятор Kotlin берет suspend функции и преобразовывает их в оптимизированную версию колбеков с использованием [конечной машины состояний](https://en.wikipedia.org/wiki/Finite-state_machine){:target="_blank"} (о которой мы поговорим позже).

# Интерфейс Continuation
Suspend функции взаимодействуют друг с другом с помощью `Continuation` объектов. `Continuation` объект - это простой generic интерфейс с дополнительными данными. Позже мы увидим, что сгенерированная машина состояний для suspend функции будет реализовать этот интерфейс.

Сам интерфейс выглядит так:
```kotlin
interface Continuation<in T> {
  public val context: CoroutineContext
  public fun resumeWith(value: Result<T>)
}
```
- `context` это экземпляр `CoroutineContext`, который будет использоваться при возобновлении.
- `resumeWith` возобновляет выполнение корутины с [Result](https://github.com/Kotlin/kotlinx.coroutines/blob/master/stdlib-stubs/src/Result.kt){:target="_blank"}, он может либо содержать результат вычисления, либо исключение.

> С Kotlin 1.3 и далее, вы можете использовать extensions функции [resume(value: T)](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume.html){:target="_blank"} и [resumeWithException(exception: Throwable)](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume-with-exception.html){:target="_blank"}, это специализированные версии метода `resumeWith`.

Компилятор заменяет ключевое слово suspend на дополнительный аргумент `completion` (тип `Continuation`) в функции, аргуемнт используется для передачи результата suspend функции в вызывающую корутину:


```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resume(userDb)
}
```
Для упрощения, наш пример возвращает `Unit` вместо объекта `User`.

Байткод suspend функций фактически возвращает `Any?` так как это объединение (union) типов `T | COROUTINE_SUSPENDED`. Что позволяет функции возвращать результат синхронно, когда это возможно.

> Если suspend функция не вызывает другие suspend функции, компилятор добавляет аргумент Continuation, но не будет с ним ничего делать, байткод функции будет выглядеть как обычная функция.

Кроме того, интерфейс `Continuation` можно увидеть в:

- При конвертации колбек-API в корутины с использованием [suspendCoroutine](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html){:target="_blank"} или [suspendCancellableCoroutine](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html){:target="_blank"} (предпочтительнее использовать в большинстве случаев). Вы напрямую взаимодействуете с 
экземпляром `Continuation`, чтобы возобновить корутину, приостановленную после выполнения блока кода из аргументов suspend функции.

- Вы можете запустить корутину при помощи [startCoroutine](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html){:target="_blank"} extension функции в suspend методе. Она принимает `Continuation` как аргумент, который будет вызван, когда новая корутина завершится либо с результатом, либо с исключением.

# Используем Dispatchers
Вы можете переключаться между разными диспетчерами для запуска вычислений на разных потоках. Как Kotlin знает, где возобновить suspend вычисления?

Есть подтип `Continuation`, он называется [DispatchedContinuation](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/internal/DispatchedContinuation.kt){:target="_blank"}, где его метод `resume` делает вызов `Dispatcher` доступного в контексте корутины `CoroutineContext`. Все диспетчеры (`Dispatchers`) будут вызывать метод `dispatch`, кроме типа `Dispatchers.Unconfined`, он переопределяет метод `isDispatchNeeded` (он вызывается перед вызовом `dispatch`), который возвращает _false_ в этом случае.

## Сгенрированная машина состояний

> Уточнение: Приведенный код не полностью соответствует байткоду сгенерированному компилятором. Это будет код на Kotlin, достаточно точный, для понимания того, что в действительности происходит внутри. Это представление сгенерировано корутинами версии 1.3.3 и может поменяться в следующих версиях библиотеки.

Компилятор Kotlin определяет, когда функция может остановится внутри. Каждая точка останова представляется как отдельное состояние в конечной машине состояний. Такие состояния компилятор помечает метками:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  
  // Label 0 -> first execution
  val user = userRemoteDataSource.logUserIn(userId, password)
  
  // Label 1 -> resumes from userRemoteDataSource
  val userDb = userLocalDataSource.logUserIn(user)
  
  // Label 2 -> resumes from userLocalDataSource
  completion.resume(userDb)
}
```

Компилятор использует `when` для состояний:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  when(label) {
    0 -> { // Label 0 -> first execution
        userRemoteDataSource.logUserIn(userId, password)
    }
    1 -> { // Label 1 -> resumes from userRemoteDataSource
        userLocalDataSource.logUserIn(user)
    }
    2 -> { // Label 2 -> resumes from userLocalDataSource
        completion.resume(userDb)
    }
    else -> throw IllegalStateException(/* ... */)
  }
}
```

Этот код неполный, так как различные состояния не могут обмениваться информацией. Компилятор использует для обмена тот же самый объект `Continuation`. Вот почему родительски тип в `Continuation` это `Any?` вместо ожидаемого возвращаемого типа `User`.

При этом компилятор создает приватный класс, который:
1. хранит нужные данные
2. вызывает функцию `loginUser` рекурсивно для возобновления вычисления

Ниже представлен примерный вид такого сгенерированного класса:
> Комментарии в коде были добавлены вручную для объяснения действий

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
  
  class LoginUserStateMachine(
    // completion parameter is the callback to the function 
    // that called loginUser
    completion: Continuation<Any?>
  ): CoroutineImpl(completion) {
  
    // Local variables of the suspend function
    var user: User? = null
    var userDb: UserDb? = null
  
    // Common objects for all CoroutineImpls
    var result: Any? = null
    var label: Int = 0
  
    // this function calls the loginUser again to trigger the
    // state machine (label will be already in the next state) and
    // result will be the result of the previous state's computation
    override fun invokeSuspend(result: Any?) {
      this.result = result
      loginUser(null, null, this)
    }
  }
  /* ... */
}
```

Поскольку `invokeSuspend` вызывает `loginUser` только с аргументом `Continuation`, остальные аргументы в функции `loginUser` будут нулевыми. На этом этапе компилятору нужно только добавить информацию как переходить из одного состояния в другое.

Компилятору нужно знать:
1. Функция вызывается первый раз или
2. Функция была возобновлена из предыдущего состояния
Для этого проверяется тип аргумента `Continuation` в функции:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
  /* ... */
  val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)
  /* ... */
}
```

Если функция вызывается первый раз, то создается новый экземпляр `LoginUserStateMachine` и аргумент `completion` передается в этот экземпляр, чтобы возобновить вычисление. Иначе продолжится выполнение машины состояний.

Давайте взглянем на код, который генерирует компилятор для смены состояний и обмена информацией между ними:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
          /* ... leaving out the last state on purpose */
    }
}
```

Обратите внимание на различия между этим и предыдущим примером кода:

- Появилась переменная `label` из `LoginUserStateMachine`, которая передается в `when`.
- Каждый раз при обработке нового состояния проверяется есть ли ошибка.
- Перед вызовом следующей suspend функции (`logUserIn`), `LoginUserStateMachine` обновляет переменную `label`.
- Когда внутри машины состояний вызывается другая suspend функция, экземпляр `Continuation` (с типом `LoginUserStateMachine`) передается как аргумент. Вложенная suspend функция также была преобразована компилятором со своей машиной состояний. Когда эта внутренняя машина состояний завершит свою работу, она возобновит выполнение "родительской" машины состояний.

Последнее состояние должно возобновить выполнение `completion` через вызов `continuation.cont.resume` (очевидно что входной аргумент `completion`, сохраняется в переменной `continuation.cont` экземпляра `LoginUserStateMachine`):

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        /* ... */
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```
<br/>
Компилятор Kotlin делает много работы "под капотом". 
Из _suspend_ функции:
```kotlin
suspend fun loginUser(userId: String, password: String): User {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}
```

<br/>
Генерируется большой кусок кода:
```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {

    class LoginUserStateMachine(
        // completion parameter is the callback to the function that called loginUser
        completion: Continuation<Any?>
    ): CoroutineImpl(completion) {
        // objects to store across the suspend function
        var user: User? = null
        var userDb: UserDb? = null

        // Common objects for all CoroutineImpl
        var result: Any? = null
        var label: Int = 0

        // this function calls the loginUser again to trigger the 
        // state machine (label will be already in the next state) and 
        // result will be the result of the previous state's computation
        override fun invokeSuspend(result: Any?) {
            this.result = result
            loginUser(null, null, this)
        }
    }

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```

<br/>

---

Компилятор Kotlin преобразовывает каждую _suspend_ функцию в машину состояний, с использованием обратных вызовов.

Зная как компилятор работает "под капотом", вы лучше понимаете:
- почему _suspend_ функция не вернет результат пока не завершится вся работа, которая она начала;
- каким образом код приостанавливается не блокируя потоки (вся информация, о том что нужно выполнить при возобновлении работы, хранится в объекте `Continuation`).































