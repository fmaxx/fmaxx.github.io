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

> Статья предпологает, что вы знакомы с основами разработки под Android. Если это не так, посмотрите [руководство для начинающих в Android](http://www.raywenderlich.com/78574/android-tutorial-for-beginners-part-1){:target="_blank"}.

[Начало](#1-getting-started)


![devices](/images/2021-01-25-android-ble-part-4/1.jpeg)

## Bonding
Некоторые устройства для правильной работы требуют bonding. Технически это обозначает, что генерируются ключи шифрования, обмениваются и хранятся, для безопасного обмена данными. При запуске процедуры bonding, Android может запросить у пользователя согласие, пин-код или кодовую фразу. При следующих подключениях, Android уже знает, что устройство сопряжено и обмен ключами шифрования происходит скрытно без участия пользователя.  Использование bonding делает подключение к устройству более безопасным, так как соединение зашифровано.

Тема bonding плохо описана в документации Google, полностью непонятно, как приложение должно работать с bonding. Первое на что вы обратите внимание это метод `createBond()`. Что интересно, в iOS такого метода нет вообще и фреймворк `CoreBluetooth` делает все за вас! Тогда зачем вызывать `createBond()`? Кажется немного странным, вам заранее надо знать, какие устройства требуют bonding, а какие нет. Протокол Bluetooth был спроектирован так, что обычно устройства явно говорят – им требуется или нет bonding. Я копнул немного глубже и поэкспериментировал. Чтобы разобраться с этим,  ушло некоторое время, но в конце концов, все оказалось просто.

Принципы работы с bonding: 
- **Пусть Android сам работает с bonding.** Android сделает bonding за вас, когда устройство скажет, что нужен bonding, или во время операции чтения/записи зашифрованной характеристики. В большинстве случаев не надо вызывать `createBond()` самостоятельно (_Прим. переводчика: мне пришлось это делать самостоятельно, из-за особенностей прошивки устройства. Кроме того, Samsung работает по-другому, чем другие вендоры_);
- **Нельзя запускать другие операции, в процессе работы bonding.** Если вы будете запускать обнаружение сервисов или читать/писать характеристики, это приведет к ошибками и сбросу соединения. Просто дождитесь пока Android выполнит bonding;
- **Продолжайте очередь операций после завершения bonding.** Как только операция bonding завершилась, продолжайте выполнение операций из очереди;
- **Если вы знаете, что делаете, и это необходимо** вы можете вызвать `createBond()` для запуска bonding с устройством самостоятельно. Но это должно быть исключением.

## Что вызывает bonding?

Есть три причины, по которым запускается процесс bonding:
1. **При соединении с устройством**, оно сигнализирует, что требуется bonding, до любых других операций;
2. **Характеристика может быть «зашифрована» для чтения или записи.** При попытке прочитать или записать такую характеристику, запустится bonding. Если он пройдет удачно чтение/запись также выполнится, в случае ошибки bonding – чтение/запись выполнится с ошибкой `INSUFFICIENT_AUTHENTICATION`. Такая же ошибка есть в iOS.
3. Вы **запускаете процесс bonding самостоятельно** через вызов `createBond()`. Если этого требует ваше устройство, оно вероятно не будет совместимо с iOS, так как там нет аналогичного метода. Но формально в протоколе Bluetooth такое возможно.

Давайте обсудим каждый случай.

### Bonding во время подключения
Если устройство требует bonding сразу после подключения, то при вызове колбека `onConnectionStateChange` состояние bonding будет `BOND_BONDING`. Это означает что идет процесс bonding и **вы не должны ничего делать в этот момент**, например вызывать `discoverServices()`, до тех пор пока процесс bonding не закончится! Иначе возможны неожиданные дисконнекты или ошибки обнаружения сервисов. Поэтому следует специально обрабатывать эту ситуацию в `onConnectionStateChanged`:

{% highlight java %}
// Take action depending on the bond state
if(bondstate == BOND_NONE || bondstate == BOND_BONDED) {
    // Connected to device, now proceed to discover it's services
    ... 
} else if (bondstate == BOND_BONDING) {
    // Bonding process has already started let it complete
    Log.i(TAG, "waiting for bonding to complete");
}
{% endhighlight %}

Чтобы следить, как идет процесс bonding, необходимо зарегистрировать колбек `BroadcastReceiver` для интента `ACTION_BOND_STATE_CHANGED` до вызова `connectGatt`. Этот колбек будет вызываться несколько раз в процессе bonding.


{% highlight java %}
context.registerReceiver(bondStateReceiver, 
                        new IntentFilter(ACTION_BOND_STATE_CHANGED));
private final BroadcastReceiver bondStateReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        final BluetoothDevice device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);

        // Ignore updates for other devices
        if (bluetoothGatt == null || !device.getAddress().equals(bluetoothGatt.getDevice().getAddress()))
            return;

        // Check if action is valid
        if(action == null) return;

        // Take action depending on new bond state
        if (action.equals(ACTION_BOND_STATE_CHANGED)) {
            final int bondState = intent.getIntExtra(EXTRA_BOND_STATE, ERROR);
            final int previousBondState = intent.getIntExtra(BluetoothDevice.EXTRA_PREVIOUS_BOND_STATE, -1);

            switch (bondState) {
                case BOND_BONDING:
                    // Bonding started
                    ...
                    break;
                case BOND_BONDED:
                    // Bonding succeeded
                    ...
                    break;
                case BOND_NONE:
                    // Oh oh
                    ...
                    break;
            }
        }
    }
};
{% endhighlight %}

После завершения bonding, мы запускаем обнаружение сервисов (service discovery), если они еще не обнаружены, это можно проверить:

{% highlight java %}
case BOND_BONDED:
    // Bonding succeeded
    Log.d(TAG, "bonded");

    // Check if there are services
    if(bluetoothGatt.getServices().isEmpty()) {
        // No services discovered yet
        bleHandler.post(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG, String.format("discovering services of '%s'", getName()));
                boolean result = bluetoothGatt.discoverServices();
                if (!result) {
                    Log.e(TAG, "discoverServices failed to start");
                }
            }
        });
    }
{% endhighlight %}

Вот и все, что касается особенностей bonding при подключении.

### Bonding при чтении/записи зашифрованных характеристик

Если bonding стартует при чтении/записи зашифрованной характеристики, то самая первая операция чтения/записи окончится с ошибкой `GATT_INSUFFICIENT_AUTHENTICATION`. На версиях Android-6, 7 вы получите эту ошибку в `onCharacteristicRead`/`onCharacteristicWrite`, при этом процесс bonding уже будет запущен внутри Android. С версии Android-8 ошибки не будет и Android самостоятельно повторит операцию после завершения bonding. Получается на Android-6, 7 надо повторить операцию чтения/записи самостоятельно. Итак, вам надо поймать ошибку и сделать повтор операции после bonding.

При получении такой ошибки, не продолжайте запуск операций:

{% highlight java %}
public void onCharacteristicRead(BluetoothGatt gatt, final BluetoothGattCharacteristic characteristic, int status) {
    // Perform some checks on the status field
    if (status != GATT_SUCCESS) {
        if (status == GATT_INSUFFICIENT_AUTHENTICATION ) {
            // Characteristic encrypted and needs bonding,
            // So retry operation after bonding completes
            // This only happens on Android 5/6/7
            Log.w(TAG, "read needs bonding, bonding in progress");
            return;
        } else {
            Log.e(TAG, String.format(Locale.ENGLISH,"ERROR: Read failed for characteristic: %s, status %d", characteristic.getUuid(), status));
            completedCommand();
            return;
        }
    }
...
{% endhighlight %}

После bonding проверяем, есть ли операция в процессе выполнения и повторяем ее:

{% highlight java %}
case BOND_BONDED:
    // Bonding succeeded
    Log.d(TAG, "bonded");

    // Check if there are services
    ...
    // If bonding was triggered by a read/write, we must retry it
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
        if (commandQueueBusy && !manuallyBonding) {
            bleHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    Log.d(TAG, "retrying command after bonding");
                    retryCommand();
                }
            }, 50);
        }
    }
{% endhighlight %}

### Запуск bonding самостоятельно
Как я говорил выше, лучше не вызывать `createBond` самостоятельно, хотя сделать это, конечно можно. Спросите себя, это действительно необходимо? На iOS нет эквивалента метода `createBond()`, если этот метод – единственный способ сделать bonding для вашего устройства, то скорее всего оно несовместимо с iOS. Это прямо указывается в документации iOS. Я перепробовал несколько десятков BLE устройств, и только в единственном случае я вызывал `createBond()` самостоятельно из-за исключительных обстоятельств.

При вызове `createBond` самостоятельно, также нельзя ничего делать, пока bonding не завершится и требуется регистрировать колбек `BroadcastReceiver` для отслеживания процесса. Если устройство уже сопряжено (bonding завершился), то `createBond()` вызовет ошибку, надо проверить состояние bonding перед вызовом.

Еще одна причина запускать `createBond()` самостоятельно – упростить повторное подключение. Объект `BluetoothDevice` можно получить при помощи MAC-адреса, если устройство закешировано или сопряжено (bonding). Таким образом вам не придется снова сканировать устройство... Может пригодиться! (_Прим. переводчика: я как раз работал с таким вариантом подключения, его требовалось сделатьполностью детерминированным, разбитым на подфазы, для точного понимания что происходит._)

## Удаление bonding
Как пользователь Android, я могу увидеть список сопряженных устройств в Bluetooth настройках. Там можно удалить устройство, bonding также будет удален.
> Требуется некоторое время на удаление устройства.

Достаточно странно, что нет официального способа удалить bonding устройства программно. Это можно сделать, используя скрытый метод `removeBond()`, доступный через механизм рефлексии в Java:

{% highlight java %}
try {
    Method method = device.getClass().getMethod("removeBond", (Class[]) null);
    result = (boolean) method.invoke(device, (Object[]) null);
    if (result) {
        Log.i(TAG, "Successfully removed bond");
    }
    return result;
} catch (Exception e) {
    Log.e(TAG, "ERROR: could not remove bond");
    e.printStackTrace();
    return false;
}
{% endhighlight %}

## Потеря bonding
Большинство BLE устройств поддерживают bonding только с одним смартфоном. Типичный сценарий, когда мы теряем bonding такой:
- Смартфон А делает bonding с устройством Х
- Смартфон B делает bonding с устройством Х
- Смартфон А переподключается к устройству Х, и теперь bonding потерян.

При реконнекте смартфон А получит состояние bonding `BOND_NONE` в колбеке `BroadcastReceiver`. Сравнивайте предыдущее состояние bonding, чтобы понять была потеря или нет:

{% highlight java %}
case BOND_NONE:
    if(previousBondState == BOND_BONDING) {
       // Bonding failed
       ...
    } else {
       // Bond lost
       ...
    }
    disconnect();
    break;
{% endhighlight %}

Если случилась потеря bonding, отключаемся от устройства, иначе будут происходить странные вещи и соединение с устройством не будет нормально работать. Когда вы делаете реконнект, Android снова запускает процедуру bonding. Тоже самое происходит и при обрыве связи.

Существует мелкий баг, о котором следует знать. При потере bonding, кажется нужна **одна секунда** для того, чтобы Bluetooth стек обновил свое внутреннее состояние. Если сделать реконнект сразу после потери bonding, Android может сказать, что устройство все еще сопряжено, но на самом деле это будет не так. Сделайте задержку в одну секунду перед переподключением.

## Pairing попап 
_Прим. переводчика: не нашел толковой замены слова «pairing», «спаривание» - звучит неблагозвучно здесь._

Когда Android запускает процесс bonding, может появится всплывающее окно. Я говорю «может», потому что некоторые вендоры используют свою логику показа этого попапа (_Прим. переводчика: на моем Samsung-S9, после обновления до Android-10, это попап стал появляться всегда, при коннекте любого нового устройства, до этого обновления, такого не было_). На смартфонах Google (или других вендоров, где код Android в этой части не изменялся), всплывающий попап появляется только при определенный условиях.

**Pairing попап появляется на переднем фоне если:**
- Устройство недавно было в режиме обнаружения;
- Устройство было обнаружено недавно;
- Устройство недавно было выбрано в «сборщике устройств»;
- Экран настроек Bluetooth виден.

Значение «недавно» означает **в течение последних 60 секунд**. Условия выглядят непонятными, поэтому лучше посмотреть на [исходный код](https://android.googlesource.com/platform/packages/apps/Settings/+/eclair-release/src/com/android/settings/bluetooth/LocalBluetoothManager.java?autodive=0%2F%2F#304){:target="_blank"}. Если все эти условия не выполняются, то вместо попапа появится уведомление, которое большинство пользователей не замечает. Но если они заметят и нажмут на него, всплывающее окно сбивает с толку своей опцией доступа к контактам. Ужасный UI по-моему! Некоторые производители (справедливо) решили исправить такое поведение! На устройствах Samsung всплывающее окно-подтверждение (для подключений в режиме JustWorks) вообще не отображается, а всплывающие окна всегда появляются на переднем плане. При этом всплывающее окно открывается только при вводе PIN-кода или кодовой фразы. Никаких доступов к контактам и всегда передний план. Так намного лучше!

Так что, если вдруг вы захотите, чтобы всплывающее окно всегда отображалось на переднем плане, запускайте обнаружение на одну секунду перед подключением к устройству. Выглядит как хак, но это работает. Код ниже:

{% highlight java %}
public void startPairingPopupHack() {
    String manufacturer = Build.MANUFACTURER;
    if(!manufacturer.equals("samsung")) {
        bluetoothAdapter.startDiscovery();

        callBackHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG, "popup hack completed");
                bluetoothAdapter.cancelDiscovery();
            }
        }, 1000);
    }
}
{% endhighlight %}

Важный момент здесь – вы не должны запускать никакие BLE операции пока попап на экране. Подождите ответа от пользователя.

Если учтете все эти моменты, bonding будет работать как часики!

## Подведение итогов...

На этом мы завершаем цикл статей о BLE в Android (_Прим. переводчика: я готовлю отдельную статью-заключение, где опишу свои подходы к работе с BLE устройствами на Android, небольшие ньюансы и решения для стабильной продолжительной работы с устройствами_). Надеюсь эта информация будет полезной вам и сделает работу с BLE комфортнее. Чем больше знаешь про BLE, тем лучше работает ваше приложение. Успехов!

> Не терпится поработать с BLE? Попробуйте [мою библиотеку Blessed for Android](https://github.com/weliem/blessed-android){:target="_blank"}. Она использует все подходы из этой серии статей и упрощает работу с BLE в вашем приложении.