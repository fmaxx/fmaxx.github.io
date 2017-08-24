---
layout:     post
title:      Provisioning ios часть-2.
date:       2017-06-06 11:21:29
summary:    Управление профилями и сертификатами.
categories: ios provisioning
---

Перевод статьи [DEMYSTIFYING IOS PROVISIONING PART 2: CREATING AND ASSIGNING CERTIFICATES AND PROFILES](http://martiancraft.com/blog/2017/07/demystifying-provisioning-part2/)

[Перевод первой части здесь]({% post_url 2017-08-20-ios-provisioning-part-1 %})

Разрушая мифы iOS Provisioning, часть 2: управление сертификатами и профилями.
==================

В [первой части]({% post_url 2017-08-20-ios-provisioning-part-1 %}) этой статьи мы узнали что такое сертификаты и профили, как они используются в разработке под платформу Apple. Во второй части статьи мы обсудим как создать все нужное для подписи приложения в личном кабинете разработчика на сайте Apple и как это подключить в Xcode.

Наши задачи:

* создать App ID
* сгенерировать *development* and *App Store* сертификаты
* создать *development* и *distribution* provisioning профили
* создать схему (назовем ее *App Store*) в Xcode проекте
* настроить сертификаты и профили в схеме

Многие задачи выполняются в разделе Provisioning в личном кабинете разработчика на сайте [Apple Developer](http://developer.apple.com).
![image1](/images/2017-08-15-ios-provisioning-part-2/1.png)

Если у вас уже есть оплаченный аккаунт, откройте [Apple Developer website](http://developer.apple.com), выберете "Account" -> "Certificates, Identifiers & Profiles."

## Создание App IDs
App ID это уникальный идентификатор для регистрации вашего приложения. Первым делом, давайте создадим его.
![image1](/images/2017-08-15-ios-provisioning-part-2/2.png)

Шаги для регистрации на сайте apple разрабочика:
1. Выберите "App IDs" под вкладкой "Identifiers" 
2. Нажмите “+” вверху списка
3. Введите имя для App ID (используйте название приложения или название + дополнительное имя)
4. Выберите “Explicit App ID” и введите ваш Bundle ID в текстовое поле
5. Под “App Services” раздела, выберите сервисы нужные для вашего приложения (их можно будет отредактировать позже)
6. Нажимаем “Continue”
7. Если все хорошо - выбираем “Register”

Это все. Bundle ID зарегистрирован и можно генерировать профили на основе этого ID. Если в будущем вы измените Bundle ID, придется также удалить App ID (если он не будет использоваться) и пересоздать новый App ID.

Так, а в чем тогда разница между App ID и Bundle ID? В общем App ID это уникальный идентификатор приложения в экосистеме App Store, не может быть двух приложений там с одинаковыми App ID. Bunlde ID это идентификатор учетной записи разработчика (используется обратная доменная запись - com.mysite.appname).

## Создание сертификатов
Нам нужно создать сертификат для подписи приложений App Store. 



