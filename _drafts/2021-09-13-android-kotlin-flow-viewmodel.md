---
layout:     post
title:      Android, Kotlin Flow во ViewModel - все сложно.
date:       2021-09-12 12:02:00
summary:    Как работают Kotlin-Flow во ViewModel.
categories: Android Kotlin-Flow ViewModel
---

Перевод статьи [Kotlin’s Flow in ViewModels: it’s complicated](https://bladecoder.medium.com/kotlins-flow-in-viewmodels-it-s-complicated-556b472e281a){:target="_blank"}.

Загрузка данных для UI в приложениях Android может быть непростой задачей. Нам надо учитывать жизненный цикл компонентов Android и **изменения конфигурации**, потому что все это приводит к уничтожению и восстановлению Activity.

Отдельные экраны приложения постоянно переключаются между активным и скрытым состоянием, когда пользователь ходит вперед-назад по экранам, переключается с одного приложения на другое, блокировка и разблокировка экрана. Каждый компонент должен выполнять активную работу в нужном состоянии экрана.

**Изменения конфигурации** происходят в случаях:
- при изменении ориентации экрана;
- когда приложение переключается в мульти-оконный режим;
- при переключении визуальной темы смартфона;
- при изменении системных настроек - языка, шрифтов и т.д.


# Повышаем эффективность

Для улучшения пользовательского опыта, эффективная загрузка данных во Fragment и Activity должна учитывать следующие правила:

1. **Кеширование**: актуальные загруженные данные, должны быть доставлены немедленно и не загружаться повторно. В частности, когда существующий Fragment или Activity
становятся видимыми снова или Activity пересоздается после изменения конфигурации.
> Прим. переводчика - Кеширование данных (в особенности сетевых запросов), достаточно скользкая тема, где надо работать аккуратно.

2. **Избегать фоновую работу**: когда Activity или Fragment скрываются (состояние изменяется со `STARTED` на `STOPPED`), любая работа по загрузке внешних данных должна вставать на паузу или отменяться для экономии ресурсов. Это особенно важно для бесконченых потоков данных, как например геолокация или периодическое обновление каких-либо данных.

3. **Работа не прерывается при изменении конфигурации**: это исключение из правила #2, Во время смены конфигурации, текущая Activity заменяется новым экземпляром с сохранением состояния, поэтому отменять текущую работу в старом экземпляре Activity и перезапускать работу при создании нового экземпляра Activiti было бы контрпродуктивно.

## Современный подход: ViewModel и LiveData
В 2017 Google зарелизила первый набор библиотек [**Architecture Components**](https://developer.android.com/topic/libraries/architecture){:target="_blank"}, там появились **ViewModel** и **LiveData** компоненты, которые помогают разработчикам эффективно работать с данными, поддерживая все 3 правила выше. 

[**ViewModel**](https://developer.android.com/topic/libraries/architecture/viewmodel){:target="_blank"}, сохраняет данные при изменении конфигурации, используется для достижения правил #1 и #3: операции загрузки выполняются непрерывно во время изменения конфигурации, полученные данные могут кешироваться и совместно использоваться одним или несколькими Fragment или Activity.

[**LiveData**](https://developer.android.com/topic/libraries/architecture/livedata){:target="_blank"}, простой контейнер данных, поддерживающий подписку на изменения и учитывающий жизненный цикл компонентов Android. Новые данные отправляются наблюдателям только когда их жизненный цикл в состоянии не менее `STARTED` (видимый), наблюдатели отписываются автоматически, что избавляет от утечек памяти. `LiveData` используется для достижения правил #1 и #2: кеширует последнее значение данных и это значение автоматически отправляется новым наблюдателям. В дополнение, LiveData уведомляет, когда в состоянии `STARTED` больше нет наблюдателей и можно избежать ненужной фоновой работы.

![view-model-scope](/images/2021-09-13-android-kotlin-flow-viewmodel/view-model-scope.png)
> Опытный разработчик, как правило уже знаком со всем этим. Но важно вспомнить все возможности, чтобы сравнить их с **Flow**.

## LiveData + Coroutines

LiveData довольна ограничена по сравнению с реактивными решениями (например RxJava):

- передает и берет данные только только на главном потоке (main thread). Интересно, что оператор `map` выполняет трансформацию объектов на главном потоке и не может использоваться на I/O потоках или для тяжелых вычислений на CPU. Для этого используется оператор `switchMap` совместно с ручным запуском асинхронной операции в нужном потоке, даже если в основной поток надо отправить единственное значение.
- есть только 3 оператора преобразования для LiveData: `map()`, `switchMap()` и `distinctUntilChanged()`. Если вам нужно больше, вы должны сами это сделать, используя `MediatorLiveData`.

Чтобы преодолеть эти ограничения, библиотеки Jetpack дают специальные "мосты" из LiveData для других технологий, таких как RxJava или Kotlin корутины.

Самый простой и наиболее элегантный из них, по мнению автора, это [**LiveData coroutine builder**](https://developer.android.com/topic/libraries/architecture/coroutines#livedata){:target="_blank"} фунцкция, подключается через `androidx.lifecycle:lifecycle-livedata-ktx` Gradle зависимость. Этот функционал похож на [`flow {} builder function`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html){:target="_blank"} из библиотеки Kotlin Coroutines и позволяет грамотно обернуть корутину в экземпляр LiveData:

```kotlin
val result: LiveData<Result> = liveData {
    val data = someSuspendingFunction()
    emit(data)
}
```

- Вы можете использовать все силу корутин, их контекстов для написания асинхронного кода в синхронной манере без колбеков, автоматически переключаясь между нужными потоками;
- Новые значения отправляются наблюдателям LiveData в главном потоке через _suspending_ методы `emit()` или `emitSource()` из корутины;
- Корутина использует специальную область видимости (scope) и жизненный цикл привязанный к экземпляру LiveData. Когда LiveData становится _неактивной_ (это значит, что больше нет наблюдателей в состоянии `STARTED`), то корутина будет автоматически отменена, работает правило #2;
- В реальности отмена корутины будет **задержана на 5 секунд** после того как LiveData станет неактивной для правильной обработки смены конфигурации: если новая Activity немедленно заменит старую и LiveData становится активной до срабатывания задержки, то отмена корутины не будет и цена перезапуска будет нулевой (правило #3);
- если пользователь вернется назад на экран и LiveData станет активной, то корутина автоматически перезапустится, но только если она была отменена до завершения. Как только корутина завершится, она больше не будет перезапускаться, те же данные не будут загружаться дважды если входные параметры не изменились (правило #1).

Вывод: используйте LiveData coroutines builder, это дает простой код и лучшее поведение по умолчанию.

А что если, репозиторий возвращает _поток значений_ в форме `Flow` (вместо _suspend_ функций с единственным значением)? В этом случае также возможно сконвертировать поток в LiveData и использовать все преимущества перечисленные выше, используя **`asLiveData()`** функцию-расширение.

```kotlin
val result: LiveData<Result> = someFunctionReturningFlow().asLiveData()
```

Внутри `asLiveData()` также использует LiveData coroutines builder для создания простой корутины обрабатывающий Flow пока LiveData активна:

```kotlin
fun <T> Flow<T>.asLiveData(): LiveData<T> = liveData {
    collect {
        emit(it)
    }
}
```

Но давайте остановимся ненадолго – что такое `Flow` и можно ли им полностью заменить LiveData?

## Введение в Kotlin Flow

![Flow is newer and trendy so it must be better, right?](/images/2021-09-13-android-kotlin-flow-viewmodel/kotlin-flow.jpeg)


[**Flow**](https://kotlinlang.org/docs/flow.html){:target="_blank"} это класс из библиотеки [Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines){:target="_blank"} представленной в 2019 году, класс является потоком значений, вычисляемый асинхронно. Концептуально похож на RxJava Observable, но основан на корутинах и имеет более простой API.

Изначально были доступны только **холодные потоки** (cold flows): потоки без состояний, которые создаются по требованию каждый раз, когда наблюдатель начинает собирать значения в области видимости (scope) корутины. Каждый наблюдатель получает собственную последовательность значений, они не общие.

Позже были добавлены новые **горячие потоки** подтипы Flow: [**`SharedFlow`**](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/){:target="_blank"} и [**`StateFlow`**](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/){:target="_blank"}. Они были выпущены со стабильной реализацией API в версии библиотеки корутин #1.4.0.

**`SharedFlow`** публикует данные, которые распространяются всем слушателям. Класс может управлять дополнительным кешем и/или буфером и фактически заменяет все варианты устаревшего `BroadcastChannel` API.

**`StateFlow`** специально оптимизированный подкласс `SharedFlow`, который хранит и воспроизводит только последнее значение. Что-то знакомое, да? 

# StateFlow и LiveData много общего: 

- Эти классы наблюдаемые (observable)
- Они хранят и распространяют последнее значение любому количеству наблюдателей
- Они заставляют перехватывать ошибки на ранней стадии: необработанное исключение (`Exception`) в колбеке LiveData останавливает приложение. Не пойманное исключение в горячем Flow потоке завершает поток без возможности перезапустить его, даже если использовать оператор `.catch()`.

# Но есть и важные отличия:
- `MutableStateFlow` требует начального значения, в отличие от `MutableLiveData`. Замечание: `MutableSharedFlow(replay = 1)`  может эмулировать `MutableStateFlow` без начального значения, но данная реализация менее эффективна.
- `StateFlow` всегда фильтрует повторяющиеся значения с помощью сравнения `Any.equals()`, `LiveData` так не делает, для этого следует подключить оператор `distinctUntilChanged()` (замечание: `SharedFlow` не имеет такого поведения).
- `StateFlow` не учитывает жизненный цикл (**not lifecycle-aware**). Однако, `Flow` может быть использовано в корутине с учетом жизненного цикла, это требует некоторого код для настройки без использования LiveData (детали ниже).
- `LiveData` использует "версионность", чтобы управлять отправкой значений наблюдателям. При помощи этого наблюдатель не получит дважды одного и того же значения при переход обратно в состояние `STARTED`.
`StateFlow` не использует "версионность". Каждый раз, когда корутина собирает данные `Flow`, она рассматривается как **новый наблюдатель** и всегда будет получать сначала последнее значение. Из-за этого может быть дублирование работы, как мы увидим на следующем примере.

## Наблюдение за LiveData против сбора данных в Flow

Организовать наблюдение за экземпляром LiveData довольно просто:

```kotlin
viewModel.results.observe(viewLifecycleOwner) { data ->
    displayResult(data)
}
```

Эта операция однократная и дальше LiveData берет на себя синхронизацию потока данных с жизненным циклом наблюдателей.

Аналогичная операция для Flow называется _сбором_ (collecting) и сбор должен выполняться в корутине. Из-за того, что Flow не знает ничего о жизненном цикле, ответственность за жизненный цикл возлагают на корутину, работающую с Flow.

Чтобы создать корутину для работы с Flow, учитывающую жизненный цикл Activity/Fragment (запускать работу с данными при состоянии `STARTED` и автоматически отменять эту работу при уничтожении):

```kotlin
viewLifecycleOwner.lifecycleScope.launchWhenStarted {
    viewModel.result.collect { data ->
        displayResult(data)
    }
}
```
Но здесь есть серьезное ограничение: **код будет работать правильно только с "холодными" потоками, без поддержки каналом или буффером** Такой Flow управляется только собирающей его корутиной: когда Activity/Fragment перейдет в состояние `STOPPPED`, корутина приостановится, производитель Flow также приостановится с ней, и больше ничего не произойдет пока корутина не возобновится.

Однако, есть другие и другие виды Flow:
- **горячие потоки**, которые всегда активные и посылают результаты всем текущим наблюдателями (включая приостановленные);
- **холодные потоки с колбэком или поддержкой канала**, которые подписываются на активный источник данных, когда сбор данных запускается и останавливает подписку, когда сбор данных отменяется (не приостанавливается).

В этих случаях, **основной производитель Flow будет оставаться активным** даже когда корутина будет приостановлена, сохраняя (в буфер) новые результаты в фоновом режиме. Ресурсы расходуются впустую, правило #2 нарушается.

![Life is like a box of chocolates](/images/2021-09-13-android-kotlin-flow-viewmodel/forest.jpeg)

Нужно создать более безопасный способ сборка Flow любого типа. Корутина работающая с потоком данных, должна быть отменена когда Activity/Fragment становится невидимой и перезапущена снова, так же, как это делает LiveData-Coroutine-Builder. Для этого был представлен новый API в `lifecycle:lifecycle-runtime-ktx:2.4.0` (остается в статусе alpha на момент написания статьи, на момент перевода - перешел в beta).

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.result.collect { data ->
            displayResult(data)
        }
    }
}
```

Или аналогично:

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewModel.result
        .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
        .collect { data ->
            displayResult(data)
        }
}
```

Как видно, эффективно и безопасно работать с данными в Actvivity или Fragment проще с помощью LiveData.

> Можно посмотреть дополнительную информацию о новом API в статье "[**_A safer way to collect flows from Android UIs._**](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda){:target="_blank"}  from Manuel Vivo.

## Заменяем LiveData на StateFlow в ViewModel



















Android Jetpack - коллекция библиотек для разработчиков, которая улучшает код, уменьшает повторяющийся код и работает единообразно в различных версиях Android. Архитектурные компоненты Android Jetpack дают инструменты 
для создания компонентов в приложении с учетом жизненного цикла Activity или Fragment. 

В этой статье, вы создадите компонент, учитывающий жизненный цикл (lifecycle-aware компонент), в приложении **AwarenessFood**. Компонент будет обрабатывать изменения сетевого подключения. Вы также создадите **lifecycle owner** для обновления состояние сети в Activity.

Само приложение показывает случайный рецепт пользователю и имеет две опции: получить новый случайный рецепт, показать детали связанные с едой. При отключении сети, на главном экране приложения появляется SnackBar с сообщением и кнопкой повтора.

В статье вы изучите:
* Жизненные циклы в Android компонентах
* Компоненты учитывающие жизненный цикл (Lifecycle-aware components)
* Наблюдателей жизненного цикла (Lifecycle observers)
* События и состояния
* Владельцев жизненного цикла (Lifecycle owners)
* Как тестировать Lifecycle-aware компоненты
* Компонент LiveData

> Статья предполагает, что вы знакомы с основами разработки под Android. Если это не так, посмотрите [руководство для начинающих в Android](http://www.raywenderlich.com/78574/android-tutorial-for-beginners-part-1){:target="_blank"}.

## Начало

Загрузите материал с оригинальной статьи (кнопка Download вверху страницы [здесь](https://www.raywenderlich.com/22025947-lifecycle-aware-components-using-android-jetpack#toc-anchor-001){:target="_blank"}). Откройте Android Studio версии 4.2.1 или выше и импортируйте проект.

Ниже общая информация, что делает каждый пакет в проекте:
* **analytics** классы для трекинга событий в приложении
* **data** классы модели
* **di** поддержка зависимостей
* **monitor** содержит единственный класс, который смотрит за состоянием подключения к сети
* **network** классы для доступа к внешним API
* **repositories** классы для доступа к постоянному хранилищу
* **viewmodels** бизнес логика
* **view** экраны

## Регистрация в spoonacular API

AwarenessFood приложение получает рецепты через **spoonacular API**. Вам нужно зарегистрироваться там, чтобы приложение работало правильно.

Заходим на сайт [spoonacular.com](https://spoonacular.com/food-api/console#Dashboard){:target="_blank"} и создаем новый аккаунт. После подтверждения аккаунта, входим в личный кабинет и ищем там API ключ. Копируем его, открываем файл **RecipesModule.kt** внутри пакета **di** и заменяем ключ в следующей строке:

{% highlight kotlin %}
private const val API_KEY = "YOUR_API_KEY_HERE"
{% endhighlight %}

Компилируем и запускаем. Должен появиться экран со случайным рецептом, похожий на такой:

![awareness-food-running](/images/2021-09-10-android-jetpack-lifecycle-tutorial/awareness-food-running-237x500.png)
<br />

Жмем на кнопку **Reload**, чтобы загрузить другой случайный рецепт. Если отключить интернет на устройстве и попробовать получить новый рецепт, то появится SnackBar с ошибкой и кнопкой повторить, как на изображении ниже:

![awareness-food-app-network-error](/images/2021-09-10-android-jetpack-lifecycle-tutorial/awareness-food-app-network-error-237x500.png)
<br />

Чтобы перейти на экран деталей, жмем в меню **More** пункт **Food Trivia**. Вы добавите этот функционал позже. Сейчас это просто экран с кнопкой:

![wareness-food-trivia](/images/2021-09-10-android-jetpack-lifecycle-tutorial/awareness-food-trivia-237x500.png)
<br />

На этом настройка проекта закончена. Сейчас вы готовы познакомиться с компонентами, учитывающие жизненный цикл (**lifecycle-aware**).

## Жизненные циклы в Android

Важная базовая вещь для Android разработчика - это понимание, как работает жизненный цикл Activity и Fragment. Жизненный цикл представляет собой серию вызовов в определенном порядке при изменении состояния Activity или Fragment.

Жизненный цикл – необходимая концепция, потому что нужно понимать, что делать в определенных состояниях Activity и Fragment. Например, настройка layout (макета) происходит в `Activity.onCreate()`. Во Fragment за настройку layout отвечает метод `Fragment.onCreateView()`. Другой пример - включение чтения геолокации в методе `onStart()`.

Для освобождения ресурсов, чтение геолокации должно быть отключено в `onStop()`, здесь же освобождаются и другие ресуры и компоненты. Важный момент: не все lifecycle вызовы обязательно выполняются каждый раз. К примеру, операционная система может вызывать, а может и нет метод `onDestory()`.

Жизненный цикл Activity:

![life-cycle activity](/images/2021-09-10-android-jetpack-lifecycle-tutorial/activity_lifecycle-650x205.png)

Для тех, кто хочет узнать побольше о жизненном цикле Activity, взгляните на статью [Introduction to Android Activities With Kotlin](https://www.raywenderlich.com/222232/introduction-to-android-activities-with-kotlin){:target="_blank"}.

Жизненный цикл Fragment:

![life-cycle fragment](/images/2021-09-10-android-jetpack-lifecycle-tutorial/fragment-lifecycle-1-133x500.png)

Подробная статья о жизненном цикле Frament - [Android Fragments Tutorial: An Introduction With Kotlin](https://www.raywenderlich.com/216981/android-fragments-tutorial-an-introduction-with-kotlin){:target="_blank"}.

## Реагируем на изменения жизненного цикла

В большинстве приложений есть несколько компонентов, которым надо реагировать на события жизненного цикла Activity или Fragment. Вам нужно инициализировать или регистрировать их в методе `onStart()` и освобождать ресурсы в методе `onStop()`.

При таком подходе, ваш код может стать запутанным и подверженным ошибкам. Код внутри методов `onStart()` и `onStop()` будет нарастать бесконечно. При этом, легко забыть отписаться от компонента или вызвать метод компонента в неподходящем событии жизненного цикла, что может привести к багам, утечкам памяти и падениям приложения.

Некоторые из этих проблем есть прямо сейчас в приложении. Откройте файл **NetworkMonitor.kt** в проекте. Этот класс слушает состояние сети и уведомляет Activity об этом.

> Примечание: Подробная информация о мониторинге сетевого соединения находится здесь - [Read Network State](https://developer.android.com/training/basics/network-ops/reading-network-state){:target="_blank"}.

Экземпляр `NetworkMonitor` должен быть доступен с момента инициализации Activity. Откройте **MainActivity.kt** и метод `onCreate()`, где монитор сети инициализируется вызовом `networkMonitor.init()`.

Затем `NetworkMonitor` регистрирует колбек в методе `onStart()` через вызов `networkMonitor.registerNetworkCallback()`. Наконец, экземпляр Activity отписывается от колбека в методе `onStop`, вызывая `networkMonitor.unregisterNetworkCallback()`.

Инициализация компонента, подписка на события через колбек и отписка от событий добавляет большое количество шаблонного кода в Activity. Кроме того, необходимо использовать только `onStart` и `onStop` для вызова методов `NetworkMonitor`.

В `MainActivity` используется только один компонент, который зависит от жизненного цикла Activity. В более крупных и сложных проектах таких компонентов гораздо больше и это может привести к полному беспорядку.

## Используем компоненты жизненного цикла (Lifecycle-Aware)

**NetworkMonitor** выполняет различные действия, зависящие от состояния жизненного цикла Activity. Другими словами, **NetworkMonitor** должен быть компонентом учитывающим жизненный цикл (**lifecycle-aware компонент**) и реагировать на изменения в жизненном цикле своего родителя – в нашем случае это `MainActivity`.

Android Jetpack предоставляет классы и интерфейсы для создания lifecycle-aware компонентов. Используя их, вы можете улучшить работу `NetworkMonitor`. Он будет работать автоматически с учетом текущего состояния жизненного цикла родительской Activity.

Владелец жизненного цикла (**lifecycle owner**) – это компонент имеющий жизненный цикл, такой как Activity или Fragment. Владелец жизненного цикла должен знать все компоненты, которые будут слушать события жизненного цикла. Паттерн ["Наблюдатель"](https://www.raywenderlich.com/18409174-common-design-patterns-and-app-architectures-for-android#toc-anchor-014){:target="_blank"} считается лучшим подходом для решения такой задачи.

### Создание наблюдателя жизненного цикла (lifecycle observer)

Наблюдатель жизненного цикла компонент, который способен слушать и реагировать на состояния жизненного цикла своего родителя. У Jetpack есть специальный интерфейс для этого – `LifecycleObserver`.

Ну что, настало время улучшить наш `NetworkMonitor`. Откройте **NetworkMonitor.kt** и добавьте к классу поддержку интерфейса `LifecycleObserver`:

{% highlight kotlin %}
class NetworkMonitor 
@Inject constructor(private val context: Context) : LifecycleObserver {
 // Code to observe changes in the network connection.
}
{% endhighlight %}

Вот и все, теперь это lifecycle наблюдатель. Вы только что сделали первый шаг по превращению `NetworkMonitor` в lifecycle-aware компонент.

# События и состояния жизненного цикла

Класс `Lifecycle` знает состояние (state) жизненного цикла родителя и передает его любому прослушивающему `LifecycleObserver`.

`Lifecycle` использует два перечисления для обмена данными о жизненном цикле: **Event** и **State**.

## Events

`Event` представляет события жизненного цикла, которые отправляет операционная система:

* ON_CREATE
* ON_START
* ON_RESUME
* ON_PAUSE
* ON_STOP
* ON_DESTROY
* ON_ANY

Каждое значение это эквивалент колбека жизненного цикла. `ON_ANY` отличается тем, что отправляется при каждом событии (можно сделать один метод для обработки всех событий).

## Реакция на события жизненного цикла

Сейчас `NetworkMonitor` – это **LifecycleObserver**, но он пока не реагирует на жизненный цикл. Нужно добавить аннотацию `@OnLifecycleEvent` к методу, чтобы он начал реагировать на конкретное событие.

Добавьте  `@OnLifecycleEvent` к нужным методам, как указано ниже:
```kotlin
@OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
fun init() {
// ...
}

@OnLifecycleEvent(Lifecycle.Event.ON_START)
fun registerNetworkCallback() {
// ...
}

@OnLifecycleEvent(Lifecycle.Event.ON_STOP)
fun unregisterNetworkCallback() {
// ...
}
```
В этом случае, метод `init()` реагирует на `ON_CREATE` событие, `registerNetworkCallback()` на `ON_START` и на событие `ON_STOP` вызывается `unregisterNetworkCallback()`.

Итак, `NetworkMonitor` теперь получает изменения жизненного цикла, сейчас нужно немного подчистить код. Следующий код больше не нужен в **MainActivity.kt**, потому что эти действия  `NetworkMonitor` выполняет сам:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
  // ...
  // 1.Network Monitor initialization.
  networkMonitor.init()
  // ...
}

// 2. Register network callback.
override fun onStart() {
  super.onStart()
  networkMonitor.registerNetworkCallback()
}

// 3. Unregister network callback.
override fun onStop() {
  super.onStop()
  networkMonitor.unregisterNetworkCallback()
}

```

Полностью удаляйте `onStart()` и `onStop()`. Кроме того надо удалить `networkMonitor.init()` из метода `onCreate()`.

Сделав эти изменения в коде, вы переместили из Activity ответственность за инициализацию, регистрацию и освобождение ресурсов в сам компонент.

# Состояния (States)
`State` хранит текущее состояние владельца жизненного цикла. Возможные значения:
- INITIALIZED
- CREATED
- STARTED
- RESUMED
- DESTROYED

Состояния жизненного цикла сигнализируют, что конкретное событие случилось.

В качестве примера, представим, что мы запускаем длительную инициализацию компонента в событии `ON_START` и Activity (или Fragment) уничтожаются до того, как инициализация  закончится. В этом случае компонент не должен выполнять никаких действий по событию `ON_STOP`, так как он не был инициализирован.

Есть прямая связь между событиями и состояниями жизненного цикла. На диаграмме ниже, показана это взаимосвязь:

![state events](/images/2021-09-10-android-jetpack-lifecycle-tutorial/state-events.png)
<br />

Эти состояния возникают при:
- INITIALIZED: Когда Activity или Fragment уже созданы, но `onCreate()` еще не вызван. Это первоначальное состояние жизненного цикла.
- CREATED: После `ON_CREATE` и после `ON_STOP`.
- STARTED: После `ON_START` and После `ON_PAUSE`.
- RESUMED: только после `ON_RESUME`.
- DESTROYED: после `ON_DESTROY`, но прямо перед вызовом `onDestroy()`. Как только состояние становится `DESTROYED`, Activity или Fragment больше не будут посылать события.

# Используем состояния (State) жизненного цикла

Иногда компоненты должны выполнять код, если их родитель находится, по крайней мере, в определенном состоянии. Нужно быть уверенным, что `NetworkMonitor` выполняет `registerNetworkCallback()` в нужный момент жизненного цикла.

Добавьте `Lifecycle` аргумент в конструктор `NetworkMonitor`: 

```kotlin
private val lifecycle: Lifecycle
```
С ним у `NetworkMonitor` будет доступ к состоянию жизненного цикла родителя.

После этого добавьте в код `registerNetworkCallback()` условие:

```kotlin
fun registerNetworkCallback() {
  if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
    // ...
  }
}
```

С этим условием `NetworkMonitor` будет запускать мониторинг состояния сети только, если состояние жизненного цикла не менее, чем `STARTED`.
Это достаточно удобно, потому что жизненный цикл может измениться до завершения кода в компоненте. Зная состояние родителя можно избежать крешей, утечек памяти и гонок состояния в компоненте. 

Итак, откройте `MainActivity.kt` и сделайте изменения для `NetworkMonitor`:

```kotlin
networkMonitor = NetworkMonitor(this, lifecycle)
```

У `NetworkMonitor` сейчас есть доступ к родительскому жизненному циклу и прослушивание сети запускается только когда Activity находится в нужном состоянии.

## Подписка на события жизненного цикла

Чтобы `NetworkMonitor` действительно начал реагировать на изменения, его родитель должен зарегистрировать `NetworkMonitor` как слушателя событий. 
В `MainActivity.kt` добавьте следующую линию в методе `onCreate()` после инициализации компонента:

```kotlin
 lifecycle.addObserver(networkMonitor)
```

Теперь владелец жизненного цикла будет уведомлять `NetworkMonitor` об изменениях. Таким же образом можно добавить другие компоненты на прослушивание событий жизненного цикла.

Это отличное улучшение. С помощью одной строчки кода компонент будет получать события жизненного цикла от родителя. Больше не надо писать шаблонный код в Activity или Fragment. Кроме того, компонент содержит весь код инициализации и конфигурации, что делает его самодостаточном и тестируемым.

Соберите и запустите приложение снова. После загрузки рецептов включите "Самолетный режим". Вы увидите ошибку сети в SnackBar, так же как и раньше:

![awareness-food-app-network-error](/images/2021-09-10-android-jetpack-lifecycle-tutorial/awareness-food-app-network-error-237x500.png)
<br />

Однако внутреннняя реализация была значительно улучшена.

## Кто владеет жизненным циклом?

Все выглядит хорошо, но... кто владеет жизненным циклом? Почему Activity сама владеет жизненным циклом в этом примере? Существуют ли другие владельцы? 

Владелец жизненного цикла (**lifecycle owner**) – компонент реализующий интерфейс `LifecycleOwner`. В нем есть единственный метод, который необходимо реализовать: `Lifecycle.getLifecycle()`. Выходит, что любой класс поддерживающий этот интерфейс может быть владельцем жизненного цикла.

Android имеет встроенные компоненты с поддержкой жизненного цикла. Для Activity это `ComponentActivity` (является базовым классом для `AppCompatActivity` и реализует интерфейс `LifecycleOwner`).

Однако, есть и другие классы с поддержкой `LifecycleOwner`. Например, `Fragment` это `LifecycleOwner`. Это значит, что можно переместить код в любой `Fragment`, если это надо и это будет работать точно так же, как и в `MainActivity`.

Жизненный цикл Fragment может быть значительно длиннее, чем цикл визуальных компонентов (UI), которых он содержит. Если наблюдатель взаимодействует с UI во Fragment, это может привести к проблемам, поскольку наблюдатель может изменить UI до инициализации или после уничтожения.

Это причина почему есть `viewLifecycleOwner` в `Fragment`. Вы можете использовать  `viewLifecycleOwner` после вызова `onCreateView()` и перед `onDestroyView()`.  Как только жизненный цикл будет уничтожен, он больше не будет отправлять события.

Распространенный случай использования `viewLifecycleOwner` это работа с `LiveData` в Fragment. Откройте **FoodTriviaFragment.kt** и добавьте в методе `onViewCreated()`, до `viewModel.getRandomFoodTrivia()`, следующий код:

```kotlin
viewModel.foodTriviaState.observe(viewLifecycleOwner, Observer { 
  handleFoodTriviaApiState(it)
})
```

Также надо будет добавить импорт класса `import androidx.lifecycle.Observer`.

Используя этот код, `FoodTriviaFragment` будет реагировать на события от `foodTriviaState` (является `LiveData`). Так как Fragment является владельцем жизненного цикла (`viewLifecycleOwner`), наблюдатель будет получать обновления данных только, когда Fragment находится в активном состоянии.

Время сделать сборку и запустить приложение. Нажмите **More** в меню и выберите **Food Trivia**. Теперь можно получить несколько забавных и интересных Food Trivia в вашем приложении.

![awareness-food-app-working](/images/2021-09-10-android-jetpack-lifecycle-tutorial/awareness-food-trivia-working-237x500.png)
<br />

# Использование ProcessLifecycleOwner

В некоторых случаях компоненту надо реагировать на изменения жизненного цикла самого приложения. К примеру, отследить когда приложение уходит в фоновый режим и возвращается на передний план. Для таких ситуаций в Jetpack есть **ProcessLifecycleOwner**, который естественно реализует интерфейс `LifecycleOwner`.

Этот класс представляет жизненный цикл всего процесса приложения. Событие `ON_CREATE` посылается только один раз, когда приложение стартует. При этом событие `ON_DESTROY` не будет посылаться вообще.

`ProcessLifecycleOwner` отправляет события `ON_START` и `ON_RESUME`, когда первая Activity проходит через эти состояния. Наконец, `ProcessLifecycleOwner` отправляет события `ON_PAUSE` и `ON_STOP` после того, последняя видимая Activity приложения проходит через соответствующие состояния.

Важно знать, что эти два последних события произойдут после определенной задержки. `ProcessLifecycleOwner` должен быть уверен в причине этих изменений. Он отправляет события `ON_PAUSE` и `ON_STOP` только если приложение перешло в фоновый режим, а не из-за изменения конфигурации.

При использовании такого владельца жизненного цикла, компонент должен также поддерживать интерфейс `LifecycleObserver`. Откройте **AppGlobalEvents.kt** в пакете **analytics**. Видно, что это `LifecycleObserver`.

Класс отслеживает, когда приложение выходит на передний план или переходит в фоновый режим. Это происходит когда владелец жизненного цикла посылает события `ON_START` и `ON_STOP`.

Регистрация такого `LifecycleObserver` происходит немного по-другому. Откройте **RecipesApplication.kt** и добавьте следующий код в метод `onCreate()`:

```kotlin
  ProcessLifecycleOwner.get().lifecycle.addObserver(appGlobalEvents)
```

Здесь мы получаем экземпляр `ProcessLifecycleOwner` и добавляем `appGlobalEvents` как слушателя событий.

Соберите и запустите приложение. После старта приложения, сверните его в фон. Если открыть `LogCat` и отфильтровать вывод по тегу *APP_LOGGER*, то вы должны увидеть сообщения:

![logcat-1](/images/2021-09-10-android-jetpack-lifecycle-tutorial/logcat-1.png)
<br />

Итак, вы увидели как Fragment, Activity и Application компоненты реализуют интерфейс `LifecycleOwner`. Но это еще не все. Сейчас вы узнаете, как создавать собственные классы, которые делают то же самое.

# Создаем собственного владельца жизненного цикла

Как вы помните, любой класс может поддерживать интерфейс `LifecycleOwner`. Это значит, что можно создать кастомного владельца жизненного цикла.

Давайте создадим такого владельца, жизненный цикл которого начинается, когда смартфон теряет сетевое соединение и заканчивается при восстановлении соединения.

Откройте **UnavailableConnectionLifecycleOwner.kt** в пакете **monitor** и сделайте изменения, чтобы класс поддерживал интерфейс `LifecycleOwner`:

```kotlin
@Singleton
class UnavailableConnectionLifecycleOwner @Inject constructor() : LifecycleOwner {
// ...
}
```

После этого, добавьте `LifecycleRegistry` в **UnavailableConnectionLifecycleOwner**:
```kotlin
private val lifecycleRegistry: LifecycleRegistry = LifecycleRegistry(this)

override fun getLifecycle() = lifecycleRegistry
```
Класс `LifecycleRegistry` это реализация `Lifecycle`, который может работать с несколькими наблюдателями и уведомлять их о любых изменениях в жизненном цикле.

# Добавляем события

Для уведомления о событиях жизненного цикла можно использовать метод `handleLifecycleEvent()`. Давайте добавим пару методов в наш класс:

```kotlin
fun onConnectionLost() {
  lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START)
}

fun onConnectionAvailable() {
  lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
}
```

В случае, когда смартфон потеряет сетевое соединение, `lifecycleRegistry` пошлет событие `ON_START` всем наблюдателям. Когда соединение снова появится, `lifecycleRegistry` отправит `ON_STOP`.

Наконец, добавьте следующий код:

```kotlin
fun addObserver(lifecycleObserver: LifecycleObserver) {
  lifecycleRegistry.addObserver(lifecycleObserver)
}
```

`addObserver()` регистрирует слушателя жизненного цикла `UnavailableConnectionLifecycleOwner`.


# Реакция на события

Теперь откройте **MainActivity.kt** и добавьте такую строчку кода:

```kotlin
unavailableConnectionLifecycleOwner.addObserver(networkObserver)
```

После этого `networkObserver` будет реагировать на события `unavailableConnectionLifecycleOwner`. `NetworkObserver` покажет SnackBar, когда устройство потеряет сеть и скроет SnackBar при восстановлении сети.

И наконец, замените `handleNetworkState()`:

```kotlin
private fun handleNetworkState(networkState: NetworkState?) {
  when (networkState) {
    NetworkState.Unavailable -> unavailableConnectionLifecycleOwner.onConnectionLost()
    NetworkState.Available -> unavailableConnectionLifecycleOwner.onConnectionAvailable()
  }
}
```

Этот код запускает события `unavailableConnectionLifecycleOwner`.

Время собрать и запустить ваше приложение. Все работает так же как и раньше, кроме мониторинга сети, где мы используем сейчас кастомный `lifecycleOwner` для обработки сетевого состояния.

В следующей секции, вы узнаете как тестировать компоненты жизненного цикла.

## Тестирование компонентов жизненного цикла

Использование компонента жизненного цикла в нашем `NetworkMonitor` дает еще одно преимущество - мы можем тестировать код со всеми связанными событиями жизненного цикла.

Эти тесты будут проверять, что вызывается нужный метод в `NetworkMonitor` в соответствии с состоянием владельца жизненного цикла. Чтобы создать тест, нужно пройти те же шаги, что и при создании кастомного владельца жизненного цикла.

### Настройка тестов
Откройте **NetworkMonitorTest.kt**. Для запуска теста необходимо сделать заглушки (mock) для владельца жизненного цикла и `NetworkMonitor`. Добавьте две заглушки в тестовый класс:

```kotlin
private val lifecycleOwner = mockk<LifecycleOwner>(relaxed = true)
private val networkMonitor = mockk<NetworkMonitor>(relaxed = true)
```

Здесь используется функционал библиотеки [**MockK**](https://mockk.io){:target="_blank"}, она позволяет имитировать реализацию класса. Аргумент `relaxed` говорит, что заглушки могут работать без указания их поведения.

После этого создайте переменную:

```kotlin
private lateinit var lifecycle: LifecycleRegistry
```

Этот код добавляет объект `LifecycleRegistry` в тест, для управления наблюдателями и рассылки им событий жизненного цикла.

И наконец, добавьте следующую строчку кода в метод `setup()`:

```kotlin
lifecycle = LifecycleRegistry(lifecycleOwner)
lifecycle.addObserver(networkMonitor)
```

Здесь инициализируется реестр `LifecycleRegistry`, в него передается заглушка владельца жизненного цикла и добавляется наблюдатель `NetworkMonitor`, чтобы слушать события.

### Добавляем тесты

Чтобы убедиться, что нужный метод вызывается в `NetworkMonitor`, реестр жизненного цикла должен установить правильное состояние и уведомить своих слушателей. Для этого будет использоваться метод `handleLifecycleEvent()`.

Первый тест будет проверять, что при событии `ON_CREATE` вызывается метод `init()`.

Давайте напишем тест:

```kotlin
@Test
fun `When dispatching On Create lifecycle event, call init()`() {
  // 1. Notify observers and set the lifecycle state.
  lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)

  // 2. Verify the execution of the correct method.
  verify { networkMonitor.init() }
}
``` 

В коде выше вы:
1. Сначала устанавливаете состояние `ON_CREATE`
2. После этого, проверяете, что был вызван метод `init()`  в `NetworkMonitor`.

Не забудьте импортировать зависимости:
```kotlin
import androidx.lifecycle.Lifecycle
import io.mockk.verify
import org.junit.Test
```

Запускайте тест, он проходит успешно.

Теперь давайте проверим событие `ON_START`, оно должно вызывать метод `registerNetworkCallback()`:

```kotlin
@Test
fun `When dispatching On Start lifecycle event, call registerNetworkCallback()`() {
  lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_START)

  verify { networkMonitor.registerNetworkCallback() }
}
```

Этот код проверяет `ON_START` событие.

Тест должен пройти успешно.

В конце концов, создадим тест для проверки события `ON_STOP` и вызова метода `unregisterNetworkCallback()`:

```kotlin
@Test
fun `When dispatching On Stop lifecycle event, call unregisterNetworkCallback()`() {
  // 1. Notify observers and set the lifecycle state.
  lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_STOP)

  // 2. Verify the execution of the correct method.
  verify { networkMonitor.unregisterNetworkCallback() }
}
```

Этот код проверяет `ON_STOP` событие.

Запустите тест и ... он падает с ошибкой **Verification failed: call 1 of 1: NetworkMonitor(#2).unregisterNetworkCallback()) was not called.**
Это значит, что `unregisterNetworkCallback()` не выполнился при событии `ON_STOP`? Тестовый код посылает событие `ON_STOP`, но перед ним обязательно должно быть событие `ON_START`.

Добавляем событие `ON_START` в тест:

```kotlin
lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_START)
lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
```  

Теперь тест проходит успешно.

Таким образом, мы проверили все события жизненного цикла и они вызывают нужные методы в `NetworkMonitor`.

## LiveData: компонент жизненного цикла

К этому моменту вы узнали, как создавать собственный компонент жизненного цикла. Но существуют ли такие же готовые компоненты в Android? Конечно, возможно самый известный из них это **LiveData**.

Принцип `LiveData` достаточно прост. Это хранилище данных за которыми можно наблюдать, то есть оно может содержать данные и уведомлять слушателей об изменениях этих данных. Однако `LiveData` это еще и компонент жизненного цикла, то есть он уведомляет своих наблюдателей только тогда, когда жизненный цикл находится в активном состоянии.

Наблюдатель в **активном состоянии** если его жизненный цикл в состоянии `STARTED` или `RESUMED`. Если жизненный цикл в другом состоянии, `LiveData` не будет уведомлять слушателей об изменениях.

# Создание и присваивание переменных LiveData

Откройте **MainViewModel.kt** из пакета **viewmodels**. Добавьте там следующие переменные:

```kotlin
private val _loadingState = MutableLiveData<UiLoadingState>()
val loadingState: LiveData<UiLoadingState>
  get() {
    return _loadingState
  }
```

`_loadingState` это `LiveData` с двумя возможными значениями: `Loading` и `NotLoading`. Эти значения будут говорить UI показывать или скрывать загрузку (`ProgressBar`).

Каждый раз, когда `LiveData` переменная получает новое значение, она уведомляет своих слушателей об изменении. Для установки нового значения используйте поле **value**.

Давайте изменим метод `getRandomRecipe()`:

```kotlin
fun getRandomRecipe() {
  _loadingState.value = UiLoadingState.Loading
  viewModelScope.launch {
    recipeRepository.getRandomRecipe().collect { result ->
      _loadingState.value = UiLoadingState.NotLoading
      _recipeState.value = result
    }
  }
}
```

Теперь обновление `value` будет отправлено всем слушателям `_loadingState`.

# Наблюдения за изменениями LiveData

Откройте **MainActivity.kt** и добавьте код в метод `onCreate()`:
```kotlin
viewModel.loadingState.observe(this, Observer { uiLoadingState ->
  handleLoadingState(uiLoadingState)
})
```

Здесь, `MainActivity` начнет наблюдать за обновлениями от `viewModel.loadingState`. Как вы видите, первый аргумент `observe()` это `this`, то есть текущий экземпляр `MainActivity`. Взгляните на сигнатуру метода `observe()`, там видно, что первый аргумент имеет тип `LifecycleOwner`. Это значит, что наблюдатели `LiveData` будут реагировать на изменения в зависимости от состояния жизненного цикла Activity. Для того чтобы открыть сигнатуру метода, нажмите **Control**- или **Command**-click.

В методе `observe()` есть такой код:
```kotlin
if (owner.getLifecycle().getCurrentState() == DESTROYED) {
  // ignore
  return;
}
```

Видно, если `LifecycleOwner` в состоянии `DESTROYED` и новое значение было установлено для переменной, то наблюдатель не получит обновление. 

Однако, если `LifecycleOwner` становится активным опять, то наблюдатели получат *последнее значение* автоматически.

Соберите и запустите проект. Приложение будет показывать прогресс бар во время загрузки приложения, после окончания прогресс бар скрывается. 

Поздравляю! Вы отрефакторили проект с использованием компонентов жизненного цикла.

## Что посмотреть?

Официальная документация по архитектурным компонентам: [Architecture Components: Official Documentation](https://developer.android.com/topic/libraries/architecture){:target="_blank"}.

Изучаем Jetpack компоненты: [Android Jetpack Architecture Components: Getting Started](https://www.raywenderlich.com/200817/android-jetpack-architecture-components-getting-started){:target="_blank"}.

Дополнительные материалы по тестированию Jetpack компонентов: [Testing Android Architecture Components](https://www.raywenderlich.com/12678525-testing-android-architecture-components){:target="_blank"}.

Глубокое погружение в LiveData: [Testing Android Architecture Components](https://www.raywenderlich.com/10391019-livedata-tutorial-for-android-deep-dive){:target="_blank"}.














































































