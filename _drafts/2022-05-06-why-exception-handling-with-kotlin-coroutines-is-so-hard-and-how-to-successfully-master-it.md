---
layout:     post
title:      Почему исключения в Kotlin Coroutines это сложно и как с этим жить?
date:       2022-05-06 12:02:00
summary:    Учимся как обрабатывать исключения в корутинах.
categories: Android Kotlin Coroutines Error Exception
---

Перевод статьи [Why exception handling with Kotlin Coroutines is so hard and how to successfully master it!](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/){:target="_blank"}.

![Header](/images/why-exception-handling-with/1.webp)

Обработка исключений вероятно одна из самых сложных частей, когда вы изучаете корутины в Kotlin. В этой статье, я расскажу о причинах сложности и объясню некоторые ключевые моменты для хорошего понимания темы. После этого вы сможете реализовать правильную инфраструктуру для обработки ошибок в своем собственном приложении.

































