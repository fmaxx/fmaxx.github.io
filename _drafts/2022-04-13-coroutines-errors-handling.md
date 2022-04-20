---
layout:     post
title:      Kotlin, обрабатываем исключения в корутинах правильно.
date:       2022-04-13 12:02:00
summary:    Узнаем особенности обработки исключений в корутинах.
categories: Android Kotlin Coroutines Suspend Error
---

Перевод статьи [Are You Handling Exceptions in Kotlin Coroutines Properly?](https://www.netguru.com/blog/exceptions-in-kotlin-coroutines){:target="_blank"}

![coding image](/images/2022-04-13-coroutines-errors-handling/1.webp)

## Как Kotlin разработчик, вы скорее всего знаете, что корутины в случае ошибки выкидывают исключения.

Возможно вы думаете, что обработка таких исключений, происходит как обычно в Kotlin/Java код. К сожалению, при использовании вложенных корутин, все может работать не так как ожидается.

В этой статье я попробую показать ситуации, в которых требуется осторожность и расскажу про лучшие практики в обработке ошибок.

## Работа вложенных корутин

Давайте начнем с примера, где вроде бы все выглядит нормально.
Пример показывает сценарий, когда нужно обновить View, данные для которого комбинируются из двух разных источников и один из источников падает с ошибкой. Функция-билдер для корутин `async` будет использоваться в слое _Repository_ для выполнения двух запросов паралелльно. Билдер требует `CoroutineScope` обычно он берется из _ViewModel_ в которой запускается выполнение корутины. Метод _Repository_ будет выглядеть так:

```kotlin
suspend fun getNecessaryData(scope: CoroutineScope): List<DisplayModel> {
    val failingDataDeferred = scope.async { apiService.getFailingData() }
    val successDataDeferred = scope.async { apiService.getData() }
    return failingDataDeferred.await().plus(successDataDeferred.await())
        .map(DisplayModel::fromResponse)
}
```

Запрос с ошибкой просто пробрасывает исключение в своем теле через небольшое время:

```kotlin
suspend fun getFailingData(): List<ResponseModel> {
    delay(100)
    throw RuntimeException("Request Failed")
}
```

Во _ViewModel_ данные запрашиваются так:
```kotlin
viewModelScope.launch {
    kotlin.runCatching { repository.getNecessaryData(this) }
        .onSuccess { liveData.postValue(ViewState.Success(it)) }
        .onFailure { liveData.postValue(ViewState.Error(it)) }
}
```
Я использую удобный `kotlin.Result` функционал здесь, `runCatching` оборачивает `try-catch` блок, a `ViewState` – класс-обертка для состояний UI.
При запуске этого кода, приложение **падает** с нашим созданным `RuntimeException`. Это кажется странными, ведь мы используем `try-catch` блок для перехвата любых исключений.
Чтобы понять, что здесь происходит, давайте повторим как исключения обрабатываются в Kotlin и Java.

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

Можно заметить некоторое сходство. Первая функция `async` выглядит как `failingMethod` выше, но так как исключение не перехватывается, похоже что этот блок не пробрасывает исключение дальше!

Это первый ключевой момент в этой истории:

И `launch` и `async` функции не пробрасывают исключения, которые возникают внутри. Вместо этого, они РАСПРОСТРАНЯЮТ их вверх по иерархии корутины.
Поведение верхнего уровня `async` объясняется ниже.

## Иерархия корутины и CoroutineExceptionHandler

Наша иерархия корутин выглядит таким образом: 

![Coroutine image](/images/2022-04-13-coroutines-errors-handling/2.png)

На самом верху у нас находится область видимости (scope) от _ViewModel_, где мы создаем корутину **верхнего уровня** при помощи функции `launch`. Внутри этой корутины мы добавляем 2 **дочерние** корутины через `async`.
Когда исключение возникает в любой дочерней корутине, исключение не пробрасывается, а немедленно передается вверх по иерархии пока не достигнет объекта области видимости (scope).

Далее _scope_ передает исключение в обработчик `CoroutineExceptionHandler`. Он может быть установлен или в самом _scope_ через конструктор или в корутине верхнего уровня, как параметр в функциях `async` и `launch`.

The scope then passes the exception to the CoroutineExceptionHandler. 
This object can be installed either in the scope itself by passing it in its constructor or in the top-level coroutine by passing it as a parameter to the launch or async methods.

































