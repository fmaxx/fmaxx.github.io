---
layout:     post
title:      Provisioning ios часть-1.
date:       2017-06-06 11:21:29
summary:    Профили, сертификаты и XCode.
categories: ios provisioning
---

Перевод [DEMYSTIFYING IOS PROVISIONING PART 1: PROFILES, CERTIFICATES, AND XCODE (OH MY!)](http://martiancraft.com/blog/2017/05/demystifying-ios-provisioning-part1/?utm_campaign=iOS%2BDev%2BWeekly&utm_medium=email&utm_source=iOS_Dev_Weekly_Issue_312)

Разрушая мифы iOS Provisioning, часть 1: Profiles, Certificates, и Xcode (неужели?!)
==================

Если вы работали достаточно долго с платформой Apple (iOS, watchOS, tvOS, or macOS), то скорее всего вы
столкнулись с этим "загадочным" процессом как *provisioning*. Это процесс разворачивания приложения на устройства напрямую или через AppStore.

В этой серии статей я хочу объяснить что такое *provisioning*, как это работает в больших командах, виды профилей и сертификатов и как все это подключить в XCode.

Первая статья посвящена сертификатами и профилям, а также почему XCode не самый лучший способ управлять профилями. 

## Зачем нужен provisioning?
Provisioning профиль определяет можно ли приложение устанавливать на конкретном устройстве, какие iOS сервисы будут доступны в приложении (iCloud, Keychain, Push Notifications и т.д.) И дополнительные данные, такие как - прямая установка приложения на устройство или через AppStore.

Есть несколько видоп профилей, в зависимости от назначения приложения:
