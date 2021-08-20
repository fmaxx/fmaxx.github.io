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

