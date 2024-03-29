---
layout:     post
title:      Kotlin, обрабатываем исключения в корутинах правильно.
date:       2022-05-05 08:02:00
summary:    Узнаем особенности обработки исключений в корутинах.
categories: Android Kotlin Coroutines Suspend Error
---

Перевод статьи [Are You Handling Exceptions in Kotlin Coroutines Properly?](https://www.netguru.com/blog/exceptions-in-kotlin-coroutines){:target="_blank"}

![coding image](/images/2022-05-05-coroutines-errors-handling/1.webp)

## Как Kotlin разработчик, вы скорее всего знаете, что корутины в случае ошибки, выкидывают исключения.

Возможно вы думаете обработка таких исключений происходит как обычно в Kotlin/Java коде. К сожалению, при использовании вложенных корутин, все может работать не так как ожидается.

В этой статье я попробую показать ситуации, в которых требуется осторожность и расскажу про лучшие практики в обработке ошибок.

## Работа вложенных корутин

Давайте начнем с примера, где все вроде бы выглядит нормально.
Пример показывает сценарий, когда нужно обновить View, данные для которого комбинируются из двух разных источников и один из источников падает с ошибкой. Функция-билдер для корутин `async` будет использоваться в слое _Repository_ для выполнения двух запросов паралелльно. Билдер требует `CoroutineScope`, обычно он берется из _ViewModel_ в которой запускается выполнение корутины. 

Метод в _Repository_ будет выглядеть так:

```kotlin
suspend fun getNecessaryData(scope: CoroutineScope): List<DisplayModel> {
    val failingDataDeferred = scope.async { apiService.getFailingData() }
    val successDataDeferred = scope.async { apiService.getData() }
    return failingDataDeferred.await().plus(successDataDeferred.await())
        .map(DisplayModel::fromResponse)
}
```

Запрос с ошибкой просто пробрасывает исключение в своем теле через небольшой таймаут:

```kotlin
suspend fun getFailingData(): List<ResponseModel> {
    delay(100)
    throw RuntimeException("Request Failed")
}
```

Во _ViewModel_ данные запрашиваются:
```kotlin
viewModelScope.launch {
    kotlin.runCatching { repository.getNecessaryData(this) }
        .onSuccess { liveData.postValue(ViewState.Success(it)) }
        .onFailure { liveData.postValue(ViewState.Error(it)) }
}
```
Я использую удобный `kotlin.Result` функционал здесь, `runCatching` оборачивает `try-catch` блок, a `ViewState` – класс-обертка для состояний UI.
При запуске этого кода, приложение **падает** с нашим созданным `RuntimeException`. Это кажется странными, ведь мы используем `try-catch` блок для перехвата любых исключений.
Чтобы понять, что здесь происходит, давайте повторим, как исключения обрабатываются в Kotlin и Java.

## Повторный проброс исключений

Простой пример:
```koltin
fun someMethod() {
   try {
      val failingData = failingMethod()
   } catch (e: Exception) {
      // handle exception
   }
}

fun failingMethod() {
   throw RuntimeException()
}
```

Исключение возникает в `failingMethod`. И в Kotlin и в Java, функции по умолчанию **пробрасывают** все исключения, которые не были обработаны внутри них. Благодаря этому механизму, исключения из функции `failingMethod` можно поймать в родительском коде через блок `try-catch`.

## Распространение исключений

Давайте перенесем логику предыдущего примера во _ViewModel_:

```kotlin
viewModelScope.launch {
     try {
        val failingData = async { throw RuntimeException("Request Failed") }
        val data = async { apiService.getData() }
        val result = failingData.await().plus(data.await()).map(DisplayModel::fromResponse)
        liveData.postValue(ViewState.Success(result))
     } catch (e: Exception) {
        liveData.postValue(ViewState.Error(e))
     }
}
```

Можно заметить некоторое сходство. Первая функция `async` выглядит как `failingMethod` выше, но так как исключение не перехватывается похоже, что этот блок не пробрасывает исключение дальше!

Это первый ключевой момент в этой истории:

И `launch` и `async` функции не пробрасывают исключения, которые возникают внутри. Вместо этого, они РАСПРОСТРАНЯЮТ их вверх по иерархии корутины.
Поведение верхнего уровня `async` объясняется ниже.

## Иерархия корутины и CoroutineExceptionHandler

Наша иерархия корутин выглядит таким образом: 

![Coroutine image](/images/2022-05-05-coroutines-errors-handling/2.png)

На самом верху у нас находится область видимости (scope) от _ViewModel_, где мы создаем корутину **верхнего уровня** при помощи функции `launch`. Внутри этой корутины мы добавляем 2 **дочерние** корутины через `async`.
Когда исключение возникает в любой дочерней корутине, исключение не пробрасывается, а немедленно передается вверх по иерархии пока не достигнет объекта области видимости (scope).

Далее _scope_ передает исключение в обработчик `CoroutineExceptionHandler`. Он может быть установлен или в самом _scope_ через конструктор или в корутине верхнего уровня, как параметр в функциях `async` и `launch`.

Имейте в виду, установка обработчика в любой **дочерней** корутине не будет работать.

Такой механизм распространиения исключений это часть **Структурной Конкурентности (Structured Concurrency)**, принцип проектирования корутин, введеный авторами для правильного выполнения и отмены иерархии корутин без утечек памяти.

[Больше деталей здесь.](https://elizarov.medium.com/structured-concurrency-722d765aa952){:target="_blank"}

Ок, но почему наше приложение падает, когда есть такой механизм? Потому что мы не установили никакого обработчика `CoroutineExceptionHandler`!

Давайте передадим обработчик в функцию `launch` (сделать это через scope здесь невозможно, так как `viewModelScope` создается не нами):

```kotlin
private val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
    liveData.postValue(ViewState.Error(throwable))
}

viewModelScope.launch(exceptionHandler) {
    // content unchanged
}
```
Теперь мы будем правильно получать ошибку в наше _view_.

## Дополнительные возможности

После наших правок, блок `try-catch` больше не нужен, исключения больше не будут перехватываться там.
Мы можем удалить его и полностью полагаться на установленный обработчик ошибок. Однако, это может быть не самым лучшим решением, если нам нужен больший контроль над выполнением корутин, так как этот обработчик собирает все исключения и не дает механизмов повтора или альтернативного исполнения.

Вторая возможность – удалить обработчик исключений, оставить логику блока `try-catch`, но изменить слой репозитория данных (_Repository_) с использованием специальных билдеров корутин для вложенного исполнения: `coroutineScope` и `supervisorScope`. Таким образом у нас будет больше контроля над потоком исполнения и, к примеру, можно использовать метод `recoverCatching` из `kotlin.Result` если требуется операция восстановления после ошибки.

Давайте взглянем, что эти билдеры предлагают.

## coroutineScope билдер

Этот билдер создает дочерний _scope_ в иерархии корутины. Ключевые особенности:

- наследует контекст из вызывающей корутины и поддерживает структурную конкурентность
- не распространяет исключения из дочерних корутин, вместо этого пробрасывает (re-throw) исключения
- отменяет все дочерние корутины, если хотя бы одна из них падает с ошибкой

Теперь больше не надо передавать `viewModelScope` в метод:

```kotlin
suspend fun getNecessaryData(): List<DisplayModel> = coroutineScope {
     val failingDataDeferred = async { apiService.getFailingData() }
     val successDataDeferred = async { apiService.getData() }
     
     failingDataDeferred.await().plus(successDataDeferred.await())
        .map(DisplayModel::fromResponse)
}
```

После этих правок, исключение из первой функции `async` попадет в `try-catch` блок во `ViewModel`, потому что исключение повторно пробрасывается из функции билдера.


## supervisorScope билдер

Билдер создает новый _scope_ с `SupervisorJob`. 

Ключевые особенности:
- если одна из дочерних корутин падает с исключением, другие корутины не отменяются
- дочерние корутины становятся верхнеуровневыми (можно настроить `CoroutineExceptionHandler`)
- наследует контекст из вызывающей корутины и поддерживает структурную конкурентность (также как `coroutineScope`)
- не распространяет исключения из дочерних корутин, вместо этого пробрасывает (re-throw) исключения (также как `coroutineScope`)

Соответственно, если наш первый запрос падает, мы все еще можем получить данные из второго `async` запроса, так как он не будет отменен.

Эта особенность требует обработчика `CoroutineExceptionHandler` в верхнеуровневой корутине, иначе `supervisorScope` все равно упадет.
Причина этого в механизме, который обсуждали выше - _scope_ всегда проверяет установлен ли обработчик ошибок. Если обработчика нет - будет падение.


```kotlin
suspend fun getNecessaryData(): List<DisplayModel> = supervisorScope {
     val failingDataDeferred = async(exceptionHandler) { apiService.getFailingData() }
     val successDataDeferred = async(exceptionHandler) { apiService.getData() }
     
     failingDataDeferred.await().plus(successDataDeferred.await())
        .map(DisplayModel::fromResponse)
}
```

К сожалению, если запустить этот код, `ViewModel` все еще перехватывает исключение.
Почему так?

## Верхнеуровневый async

В соответствии со второй особенностью `supervisorScope`, обе корутины запускаемые через `async` становятся верхнеуровневыми, которые обрабатывают исключения по-другому, чем вложенные `async`:

`async` верхнего уровня скрывает обработку исключения в объекте `Deffered`, который возвращает билдер. Объект выбрасывает нормальное исключение только при вызове метода `await()`

## Обычные исключения в supervisorScope

В документации есть следующее:
"Сбой в _scope_ (исключение выбрасывается в блоке или на этапе отмены) приводит к сбою всего _scope_ со всеми дочерними корутинами".

В нашем сценарии, исключение выбрасывается при вызове `ailingDataDeffered.await()`. Это происходит вне билдера `async`, таким образом это не распространяется в `supervisorScope`, а выбрасывается как обычное исключение. Весь `supervisorScope` немедленно падает и пробрасывает это исключение.

Чтобы избежать этой проблемы, мы можем использовать `launch` решение, которое будет правильно распространять исключение в `supervisorScope` и вторая кортуна будет оставаться живой.

```kotlin
suspend fun getNecessaryData(): List<DisplayModel> = supervisorScope {
   buildList {
        launch(exceptionHandler) { apiService.getFailingData() }.join()
        launch(exceptionHandler) { apiService.getData() }.join()
    }.map(DisplayModel::fromResponse)
}
```

Короткая заметка: вызов `join()` тут нужен, потому что он приостанавливает корутину в которой работает. Благодаря этому метод `getNecessaryData` не вернет управление пока обе `Job` не будут завершены. В противном случае метод вернет управление сразу без всяких данных.

## CancellationException

В последней части статьи, я бы хотел рассказать об исключении `CancellationException`, которое используется в механизме _Structured Concurrency_ как сигнал отмены в корутинах. Это исключение передается всем корутинам внутри _scope_ при его отмене (к примеру, если пользователь ушел с экрана) или если другая корутина падает.

Очень часто мы ненароком ломаем этот механизм, используя такой подход, чтобы обернуть корутины:

```kotlin
private suspend fun fetchData(action: suspend () -> T) =
    try {
       liveData.postValue(ViewState.Success(action()))
    } catch (e: Exception) {
       liveData.postValue(ViewState.Error(e))
    }
```

Это работает ок, если есть такой вызов для каждой корутины (Я также использовал подобный подход выше). 

Однако, если есть несколько корутин внутри другой, у нас могут быть неприятности, потому что мы 
перехватываем `CancellationException` самостоятельно и тем самым ломаем внутренний механизм отмены корутин.

Например, мы вызываем для загрузки данных следующее:

```kotlin
viewModelScope.launch {
    fetchData { someApi.request1() }
    fetchData { someApi.request2() }
}
```

`launch` запускает задачи последовательно по своей природе, это значит, что второй вызов `fetchData` ждет, пока не выполнится первый вызов.

Если пользователь уходит с экрана до того момента, когда `request1` заканчивается, `viewModelScope` будет отменен и первый вызов `fetchData` перехватит и обработает `CancellationException`.

После этого, корутина продолжит свою работу и запустит `request2` как ни в чем не бывало, потому что мы скрыли `CancellationException` от нее.

Это лишняя трата ресурсов, которая может привести также к утечкам памяти или даже падению приложения.

Чтобы это предотвратить, мы можем улучшить метод `fetchData`, чтобы повторно пробрасывать `CancellationException`. Таким образом вся родительская корутина будет отменена правильно:

```kotlin
private suspend fun fetchData(action: suspend () -> T) =
    try {
        liveData.postValue(ViewState.Success(action()))
    } catch (e: Exception) {
        liveData.postValue(ViewState.Error(e))
        if (e is CancellationException) {
            throw e
        }
    }
```

Также помним об удобном операторе `kotlin.Result`: 

```kotlin
private suspend fun runDataFetch(action: suspend () -> T) =
    kotlin.runCatching { action() }
        .onSuccess { liveData.postValue(ViewState.Success(it) }
        .onFailure {
            liveData.postValue(ViewState.Error(it))
            if (it is CancellationException) {
                throw it
            }
        }
```

Это безопасный проброс конкретного исключения - мы можем быть уверены, корутина обработает его правильно.





























