---
layout:     post
title:      Android, обработка ошибок часть 1, как исключения работают в JVM и приложениях Android.
date:       2022-04-10 12:02:00
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


Сколько раз вы видели это при использовании новенького классного приложения?

![Ошибка в приложении](/images/android-errors/android-app-crash.png)

Ошибка в приложении.

Это первая статья в цикле о том, как работают исключения в Java и Android, как SDK по креш-логам собирают диагностическую информацию для анализа работы приложения в продакшене.

## Как работают исключения в JVM и Android приложениях?
































