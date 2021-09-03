---
layout:     post
title:      Android, Жизненый цикл Jetpack компонентов.
date:       2021-08-17 12:02:00
summary:    Руководство по работе с жизненным циклом Android компонентов, базовые понятия, LifecycleObserver, события и состояния жизненного цикла, кастомные LifecycleOwner.
categories: Android Jetpack Kotlin LifecycleObserver LifecycleOwner
---

Перевод статьи [Lifecycle-Aware Components Using Android Jetpack](https://www.raywenderlich.com/22025947-lifecycle-aware-components-using-android-jetpack){:target="_blank"}.
> используется: Kotlin 1.4, Android 10.0, Android Studio 4.2

Android Jetpack - коллекция библиотек для разработчиков, которая улучшает код, уменьшает повторяющийся код и работает единообразно в различный версиях Android. Архитектурные компоненты Jetpack Android дают инструменты 
для создания компонентов в приложении с учетом жизненного цикла Activity или Fragment. 

В этой статье, вы создадите компонент, учитывающий жизненный цикл (lifecycle-aware компонент), в приложении **AwarenessFood**. Компонент будет обрабатывать изменения сетевого подключения. Вы также создадите **lifecycle owner** для обновления состояние сети в Activity.

Само приложение показывает случайны рецепт пользователю и имеет две опции: получить новый случайный рецепт, показать детали связанные с едой. Когда смартфон offline на главном экране приложения появляется SnackBar с сообщением и кнопкой повтора.

В статье вы изучите:
* Жизненные циклы в Android компонентах
* Компоненты учитывающие жизненный цикл (Lifecycle-aware components)
* Наблюдатели жизненного цикла (Lifecycle observers)
* События и состояния
* Владельцы жизненного цикла (Lifecycle owners)
* Как тестировать Lifecycle-aware компоненты
* Компонент LiveData

> Статья предполагает, что вы знакомы с основами разработки под Android. Если это не так, посмотрите [руководство для начинающих в Android](http://www.raywenderlich.com/78574/android-tutorial-for-beginners-part-1){:target="_blank"}.

[Начало](#1-getting-started)

Загрузите материал с оригинальной статьи (кнопка Download вверху страницы [здесь](https://www.raywenderlich.com/22025947-lifecycle-aware-components-using-android-jetpack#toc-anchor-001){:target="_blank"}). Откройте Android Studio версии 4.2.1 или выше и импортируйте проект.

Ниже общая информация, что делает каждый пакет в проекте:
* **analytics** классы для трекинга событий в приложении
* **data** классы модели
* **di** поддержка DI
* **monitor** содержит единственный класс, который смотрит за состоянием подключения к сети
* **network** классы для доступа к внешним API
* **repositories** классы для доступа к постоянному хранилищу
* **viewmodels** бизнес логика
* **view** экраны

## Регистрация в spoonacular API

AwarenessFood приложение получает рецепты через **spoonacular API**. Вам нужно зарегистрироваться там, чтобы приложение работало правильно.

Заходим на сайт [spoonacular](https://spoonacular.com/food-api/console#Dashboard){:target="_blank"} и создаем новый аккаунт. После подтверждения аккаунта, входим в личный кабинет и ищем там API ключ. Копируем его, открываем файл **RecipesModule.kt** внутри пакета **di** и заменяем ключ в следующей строке:

{% highlight kotlin %}
private const val API_KEY = "YOUR_API_KEY_HERE"
{% endhighlight %}

Компилируем и запускаем. Должен появиться экран со случайным рецептом, похожий на такой:

![awareness-food-running](/images/2021-08-17-android-jetpack-lifecycle-tutorial/awareness-food-running-237x500.png)
<br />

Жмем на кнопку **Reload**, чтобы загрузить другой случайный рецепт. Если отключить интернет на устройстве и попробовать получить новый рецепт, то появится SnackBar с ошибкой и кнопкой повторить, как на изображении ниже:

![awareness-food-app-network-error](/images/2021-08-17-android-jetpack-lifecycle-tutorial/awareness-food-app-network-error-237x500.png)
<br />

Чтобы перейти на экран деталей, жмем в меню **More** пункт **Food Trivia**. Вы добавите этот функционал позже. Сейчас это просто экран с кнопкой:

![wareness-food-trivia](/images/2021-08-17-android-jetpack-lifecycle-tutorial/awareness-food-trivia-237x500.png)
<br />

На этом найстройка проекта закончена. Сейчас вы готовы познакомиться с компонентами, учитывающие жизненный цикл (**lifecycle-aware**).

## Жизненные циклы в Android

Важная базовая вещь для Android разработчика - это понимание как работает жизненный цикл Activity и Fragment. Жизненный цикл представляет собой серию вызовов в определенном порядке при изменении состояния Activity, Fragment.

The lifecycle is important because certain actions need to take place when the activity or fragment is in a specific state. For example, setting the activity layout needs to take place in its onCreate().

Жизненный цикл необходимая концепция, потому что нужно понимать что делать в определенных состояниях Activity и Fragment. Например, настройка layout (макета) происходит в `Activity.onCreate()`. Во Fragment за настройку layout отвечает метод `Fragment.onCreateView()`. Другой пример - включение чтения геолокации в методе `onStart()`.

Для освобождения ресурсов, чтение геолокации должно быть отключено в `onStop()`, здесь же освобождаются и другие ресуры и компоненты. Важный момент: не все lifecycle вызовы обязательно выполняются каждый раз. К примеру, операционная система может вызывать, а может и нет метод `onDestory()`.

Жизненный цикл Activity:

![life-cycle activity](/images/2021-08-17-android-jetpack-lifecycle-tutorial/activity_lifecycle-650x205.png)

Для тех, кто хочет узнать побольше о жизненном цикле Activity, взгляните на статью [Introduction to Android Activities With Kotlin](https://www.raywenderlich.com/222232/introduction-to-android-activities-with-kotlin){:target="_blank"}.

Жизненный цикл Fragment:

![life-cycle fragment](/images/2021-08-17-android-jetpack-lifecycle-tutorial/fragment-lifecycle-1-133x500.png)

Подробная статья о жизненном цикле Frament - [Android Fragments Tutorial: An Introduction With Kotlin](https://www.raywenderlich.com/216981/android-fragments-tutorial-an-introduction-with-kotlin){:target="_blank"}.

## Реагируем на изменения жизненного цикла

В большинстве приложений есть несколько компонентов, которым надо реагировать на события жизненного цикла Activity или Fragment. Вам нужно ининиализировать или регистрировать их в методе `onStart()` и освобождать ресурсы в методе `onStop()`.

Следуя такому подходу, ваш код может стать запутанным и подверженным ошибкам. Код внутри методов `onStart()` и `onStop()` будет нарастать бесконечно. При этом, легко забыть отписаться от компонента или вызвать метод компонента в неподходящем событии жизненного цикла, что может привести к багам, утечкам памяти и падениям приложения.

Некоторые из этих проблем есть прямо сейчас в приложении. Открываем файл **NetworkMonitor.kt** в проекте. Этот класс слушает состояние сети и уведомляет Activity об этом.

> Примечание: Подробная информация о мониторинге сетевого соединения находится здесь - [Read Network State ](https://developer.android.com/training/basics/network-ops/reading-network-state){:target="_blank"}.

Экземпляр `NetworkMonitor` должен быть доступен с момента инициализации Activity. Откройте **MainActivity.kt** и метод `onCreate()`, где монитор сети инициализируется вызовом `networkMonitor.init()`.

Затем `NetworkMonitor` регистрирует колбек в методе `onStart()` через вызов `networkMonitor.registerNetworkCallback()`. Наконец, экземпляр Activity отписывается от колбека в методе `onStop`, вызывая `networkMonitor.unregisterNetworkCallback()`.

Инициализация компонента, подписка на события через колбек и отписка от событий добавляет большое количество шаблонного кода в Activity. Кроме того, необходимо использовать только `onStart` и `onStop` для вызова методов `NetworkMonitor`.

В `MainActivity` используется только один компонент, который зависит от жизненного цикла Activity. В более крупных и сложных проектах, множество компонентов требуют такого же подхода и это может привести к полному беспорядку.

## Используем компоненты жизненного цикла (Lifecycle-Aware)

**NetworkMonitor** выполняет различные действия, зависящие от состояния жизненного цикла Activity. Другими словами, **NetworkMonitor** должен быть компонентом учитывающим жизненный цикл (**lifecycle-aware компонент**) и реагировать на изменения в жизненном цикле своего родителия – в нашем случае это `MainActivity`.

Jetpack предоставляет классы и интерфейсы для создания lifecycle-aware компонентов. Используя их, вы можете улучшить работу `NetworkMonitor`. Он будет работать автоматически с учетом текущего состояния жизненного цикла родительской Activity.

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

Каждое значение это эквивалент колбека жизненного цикла. `ON_ANY` отличается тем, что вызывается при любом колбеке.

## Реакция на события жизненного цикла

Сейчас `NetworkMonitor` это **LifecycleObserver**, но он пока не реагирует на жизненный цикл. Нам нужно добавить аннотацию `@OnLifecycleEvent` к методу, чтобы он реагировал на конкретное изменение. Используйте параметр для нужного события.

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

Итак, `NetworkMonitor` теперь может реагировать на изменения жизненного цикла, сейчас вам нужно немного подчистить код. Следующий код больше не нужен в **MainActivity.kt**, потому что эти действия он выполняет сам:

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

В качестве примера, представим, что мы запускаем длительную инициализацию компонента в событии `ON_START` и Activity (или Fragment) уничтожаются до того, как инициализация  закончится. В этом случае, компонент не должен выполнять никаких действий по событию `ON_STOP`, так как он не был инициализирован.

Есть прямая связь между событиями и состояниями жизненного цикла. На диаграмме ниже, показана это взаимосвязь:

![state events](/images/2021-08-17-android-jetpack-lifecycle-tutorial/state-events.png)
<br />

Эти состояния возникают при:
- INITIALIZED: Когда Activity или Fragment уже созданы, но `onCreate()` еще не вызван. Это первоначальное состояние жизненного цикла.
- CREATED: После `ON_CREATE` и после `ON_STOP`.
- STARTED: После `ON_START` and После `ON_PAUSE`.
- RESUMED: только после `ON_RESUME`.
- DESTROYED: после `ON_DESTROY`, но прямо перед вызовом `onDestroy()`. Как только состояние становится `DESTROYED`, Activity или Fragment больше не будут посылать события.

# Используем состояния (State) жизненного цикла

Иногда компононенты должны выполнять код, если их родитель находится, по крайней мере, в определенном состоянии. Нужно быть уверенным, что `NetworkMonitor` выполняет `registerNetworkCallback()` в нужный момент жизненного цикла.

Давайтей добавим `Lifecycle` аргумент в конструктор `NetworkMonitor`: 

```kotlin
private val lifecycle: Lifecycle
```
С ним у `NetworkMonitor` будет доступ к состоянию жизненного цикла родителя.

Сейчас давайте добавим в код `registerNetworkCallback()` такое условие:

```kotlin
fun registerNetworkCallback() {
  if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
    // ...
  }
}
```

С этим условием `NetworkMonitor` будет запускать мониторинг состояния сети только если состояние жизненного цикла не менее, чем `STARTED`.
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

Соберите и запустите приложение снова. После загрузки рецептов включите "Самолетный режим". Вы увидите ошибку сети в SnackBar, также как и раньше:

![awareness-food-app-network-error](/images/2021-08-17-android-jetpack-lifecycle-tutorial/awareness-food-app-network-error-237x500.png)
<br />

Однако внутренння реализация была значительно улучшена.

## Кто владеет жизненным циклом?

Все выглядит хорошо, но... кто владеет жизненным циклом? Почему Activity сама владеет жизненным циклом в этом примере? Существуют ли другие владельцы? 

Владелец жизнненого цикла (**lifecycle owner**) – компонент реализующий интерфейс `LifecycleOwner`. В нем есть единственный метод, который необходимо реализовать: `Lifecycle.getLifecycle()`. Выходит, что любой класс поддерживающий этот интерфейс может быть владельцем жизненного цикла.

Android имеет встроенные компоненты с поддержкой жизненного цикла. Для Activity это `ComponentActivity` (является базовым классом для `AppCompatActivity` и реализует интерфейс `LifecycleOwner`).

Однако, есть и другие классы с поддержкой `LifecycleOwner`. Например, `Fragment` это `LifecycleOwner`. Это значит, что можно переместить код в любой `Fragment`, если это надо и это будет работать точно также, как и в `MainActivity`.

Жизненный цикл Fragment может быть значительно длиннее, чем цикл визуальных компонентов (UI), которых он содержит. Если наблюдатель взаимодействует с UI во Fragment, это может привести к проблемам, поскольку наблюдатель может изменить UI до иниализации или после уничтожения.

Это причина почему есть `viewLifecycleOwner` в `Fragment`. Вы можете использовать  `viewLifecycleOwner` после вызова `onCreateView()` и перед `onDestroyView()`.  Как только жизненный цикл будет уничтожен, он больше не будет отправлять события.

Распространный случай использования `viewLifecycleOwner` это работа с `LiveData` в Fragment. Откройте **FoodTriviaFragment.kt** и добавьте в методе `onViewCreated()`, до `viewModel.getRandomFoodTrivia()`, следующий код:

```kotlin
viewModel.foodTriviaState.observe(viewLifecycleOwner, Observer { 
  handleFoodTriviaApiState(it)
})
```

Также надо будет добавить импорт класса `import androidx.lifecycle.Observer`.

Изпользуя этот код, `FoodTriviaFragment` будет реагировать на события от `foodTriviaState` (является `LiveData`). Так как Fragment является владельцем жизненного цикла (`viewLifecycleOwner`), наблюдатель будет получать обновления данных только, когда Fragment находится в активном состоянии.

Время сделать сборку и запустить приложение. Нажмите **More** в меню и выберите **Food Trivia**. Теперь можно получить несколько забавных и интересных Food Trivia в вашем приложении.

![awareness-food-app-working](/images/2021-08-17-android-jetpack-lifecycle-tutorial/awareness-food-trivia-working-237x500.png)
<br />

# Использование ProcessLifecycleOwner

В некоторых случаях компоненту нужно реагировать на изменения жизненного цикла самого приложения. К примеру, отследить когда приложение уходит в фоновый режим и возвращается на передний план. Для таких ситуаций в Jetpack есть **ProcessLifecycleOwner**, который естественно реализует интерфейс `LifecycleOwner`.

Этот класс представляет жизненный цикл всего процесса приложения. Событие `ON_CREATE` посылается только один раз, когда приложение стартует. При этом событие `ON_DESTROY` не будет посылаться вообще.

`ProcessLifecycleOwner` отправляет события `ON_START` и `ON_RESUME`, когда первая Activity проходит через эти состояния. Наконец, `ProcessLifecycleOwner` отправляет события `ON_PAUSE` и `ON_STOP` после того, последняя видимая Activity приложения проходит через соответствующие состояния.

Важно знать, что эти два последних события произойдут после определенной задержки. `ProcessLifecycleOwner` должен быть уверен в причине этих изменений. Он отправляет события `ON_PAUSE` и `ON_STOP` только если приложение перешло в фоновый режим, а не из-за изменения конфигурации.

При использовании этого владельца жизненного цикла, компонент должен поддерживать интерфейс `LifecycleObserver` как обычно. Откройте **AppGlobalEvents.kt** в пакете **analytics**. Видно, что это `LifecycleObserver`.

Класс отслеживает, когда приложение выходит на передний план или переходит в фоновый режим. Как видно по коду, это происходит когда владелец жизненного цикла посылает события `ON_START` и `ON_STOP`.

Регистрация такого `LifecycleObserver` происходит немного по-другому. Откройте **RecipesApplication.kt** и добавьте следующий код в метод `onCreate()`:

```kotlin
  ProcessLifecycleOwner.get().lifecycle.addObserver(appGlobalEvents)
```

Здесь мы получаем экземпляр `ProcessLifecycleOwner` и добавляем `appGlobalEvents` как слушателя событий.

Соберите и запустите приложение. После старта приложения, сверните его в фон. Если открыть `LogCat` и отфильтровать вывод по тегу *APP_LOGGER*, то вы должны увидеть сообщения:

![logcat-1](/images/2021-08-17-android-jetpack-lifecycle-tutorial/logcat-1.png)
<br />

Итак, вы увидели как Fragment, Activity и Application компоненты реализуют интерфейс `LifecycleOwner`. Но это еще не все. Сейчас вы узнаете, как создавать собственные классы, которые делают тоже самое.

# Создаем собственного владельца жизненного цикла

Как вы помните, любой класс может поддерживать интерфейс `LifecycleOwner`. Это значит, что вы можете создавть кастомного владельца жизненного цикла.

Давайте создадим кастомного владельца, жизненный цикл которого начинается, когда смартфон теряет сетевое соединение и заканчивается при восстановлении соединения.

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

Время собрать и запустить ваше приложение. Все работает также как и раньше, кроме мониторинга сети, где мы используем сейчас кастомный `lifecycleOwner` для обработки сетевого состояния.

В следующей секции, вы узнаете как тестировать компоненты жизненного цикла.

## Тестирование компонентов жизненного цикла

Использование компонента жизненного цикла в нашем `NetworkMonitor` дает еще одно преимущество - мы можем тестировать код со всеми связанными событиями жизненного цикла.

Эти тесты будут проверять, что вызывается нужный метод в `NetworkMonitor` в соответствии с состоянием владельца жизненного цикла. Чтобы создать тест, нужно пройти те же шаги, что и при создании кастомного владельца жизненного цикла.

## Настройка тестов



















































































