---
layout:     post
title:      Android, работа с BLE - часть 3.
date:       2020-01-18 12:30:00
summary:    Разбираемся с Android Bluetooth Low Energy.
categories: Android BLE BluetoothLowEnergy
---

Перевод статьи [Making Android BLE work — part 3](https://medium.com/@martijn.van.welie/making-android-ble-work-part-3-117d3a8aee23){:target="_blank"}.
> внимание: в цикле статей используется минимальная версия - Android 6

В [предыдущей статье](https://fmaxx.github.io/android/ble/bluetoothlowenergy/2020/01/10/android-ble-part-2.html) мы подробно обсуждали подключение и отключение BLE устройств. Эта статья о **чтении** и **записи** характеристик, а также **включение-выключение уведомлений**.

![devices](/images/2021-01-18-android-ble-part-3/1.jpeg)

## Чтение и запись характеристик
Многие разработчики, которые начинают работать с BLE на Android, сталкиваются с проблемами чтения/записи BLE характеристик. На [Stackoverflow](https://stackoverflow.com/search?q=android+ble+reading+writing+characteristic){:target="_blank"} полно людей, предлагающих просто добавляя задержки между ... Большинство таких советов неверные. Существует 2 основные причины этих проблем:
- **Операции чтения/записи асинхронные**. Это значит, что вызов метода вернется немедленно, но результат вызова вы получите немного позже – в соответствующих колбеках. Например `onCharacteristicRead()` или `onCharacteristicWrite()`.
- **Одновременно может быть запущена только одна операция**. Нужно дождаться выполнения текущей операции, и затем, запускать следующую. В исходном коде `BluetoothGatt` есть блокирующая переменная, которая при запуске операции устанавливается и сбрасывается при вызове колбека. Google про это забыла упомянуть в документации... (_Прим. переводчика: речь идет о `mDeviceBusy` и `mDeviceBusyLock` [здесь](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/bluetooth/BluetoothGatt.java){:target="_blank"}_).

Первая причина, на самом деле, не является проблемой, такова природа BLE. Асинхронное программирование это распространненая штука, используется например при сетевых вызовах. Однако вторая причина очень раздражает и требует специального подхода.

Ниже кусок кода `BluetoothGatt.java` с блокировкой переменной `mDeviceBusy`, перед чтением характеристики:
{% highlight java %}
public boolean readCharacteristic(BluetoothGattCharacteristic characteristic) {
    if ((characteristic.getProperties() 
            & BluetoothGattCharacteristic.PROPERTY_READ) == 0) {
        return false;
    }

    if (VDBG) Log.d(TAG, "readCharacteristic() - uuid: " + characteristic.getUuid());
    if (mService == null || mClientIf == 0) return false;

    BluetoothGattService service = characteristic.getService();
    if (service == null) return false;

    BluetoothDevice device = service.getDevice();
    if (device == null) return false;

    synchronized (mDeviceBusy) {
        if (mDeviceBusy) return false;
        mDeviceBusy = true;
    }

    try {
        mService.readCharacteristic(mClientIf, device.getAddress(),
                characteristic.getInstanceId(), AUTHENTICATION_NONE);
    } catch (RemoteException e) {
        Log.e(TAG, "", e);
        mDeviceBusy = false;
        return false;
    }

    return true;
}
{% endhighlight %}

Когда приходит результат чтения/записи, переменная `mDeviceBusy` сбрасывается в false снова:
{% highlight java %}
public void onCharacteristicRead(String address, 
                                int status, 
                                int handle, 
                                byte[] value) {
    if (VDBG) {
        Log.d(TAG, "onCharacteristicRead() - Device=" + address
                + " handle=" + handle + " Status=" + status);
    }

    if (!address.equals(mDevice.getAddress())) {
        return;
    }

    synchronized (mDeviceBusy) {
        mDeviceBusy = false;
    }
....
{% endhighlight %}

## Используем очередь
Выполнять чтение/запись по одной операции за раз неудобно, но любое сложное приложение должно это учитывать. Решение этой проблемы - использовать **очередь команд**. Все BLE библиотеки, которые я ранее упоминал, так или иначе реализуют очередь. Это одна из лучших практик!
Идея простая – каждая команда сначала добавляется в очередь. Затем команда забирается из очереди на исполнение, после результата, команда помечается как «завершенная» и удаляется из очереди. Запускать команды можно в любое время, но выполняются точно в том порядке, в котором поступают в очередь. Это очень упрощает разработку под BLE. В iOS аналогично работает фреймворк `CoreBluetooth` (_Прим. переводчика: который намного удобнее, чем реализация Bluetooth стека в Android_). 

Очередь создается для каждого объекта `BluetoothGatt`. К счастью, Android сможет обрабатывать очереди от нескольких объектов `BluetoothGatt`, вам не нужно об этом беспокоиться (_Прим. переводчика: у меня это не сработало, я использовал глобальную очередь команд для всех устройств_). Есть много способов создать очередь, мы будем использовать простую очередь `Queue` с `Runnable` для каждой команды и переменной `commandQueueBusy` для отслеживания работы команды:

{% highlight java %}
private Queue<Runnable> commandQueue;
private boolean commandQueueBusy;
{% endhighlight %}

Затем мы добавляем новый экземпляр `Runnable` в очередь при добавлении команды. Ниже пример чтения характеристики (readCharacteristic):

{% highlight java %}
public boolean readCharacteristic(final BluetoothGattCharacteristic characteristic) {
    if(bluetoothGatt == null) {
        Log.e(TAG, "ERROR: Gatt is 'null', ignoring read request");
        return false;
    }

    // Check if characteristic is valid
    if(characteristic == null) {
        Log.e(TAG, "ERROR: Characteristic is 'null', ignoring read request");
        return false;
    }

    // Check if this characteristic actually has READ property
    if((characteristic.getProperties() & PROPERTY_READ) == 0 ) {
        Log.e(TAG, "ERROR: Characteristic cannot be read");
        return false;
    }

    // Enqueue the read command now that all checks have been passed
    boolean result = commandQueue.add(new Runnable() {
        @Override
        public void run() {
            if(!bluetoothGatt.readCharacteristic(characteristic)) {
                Log.e(TAG, String.format("ERROR: readCharacteristic failed for characteristic: %s", characteristic.getUuid()));
                completedCommand();
            } else {
                Log.d(TAG, String.format("reading characteristic <%s>", characteristic.getUuid()));
                nrTries++;
            }
        }
    });

    if(result) {
        nextCommand();
    } else {
        Log.e(TAG, "ERROR: Could not enqueue read characteristic command");
    }
    return result;
}
{% endhighlight %}

В этом методе сначала проверяем все ли готово для выполнения (наличие и тип характеристики), ошибки логгируются. Внутри `Runnable`, фактически вызывается `readCharacteristic()`, который выдает команду на устройство. Мы также отслеживаем сколько было попыток, чтобы сделать повтор в случае ошибки (_Прим. переводчика: это лучшая тактика, чтобы добиться стабильной работы с устройством_). Если чтение характеристики возвращает `false`, мы логгируем ошибку, «завершаем» команду, чтобы можно было запустить следующую. Наконец вызывается `nextCommand()`, чтобы запустить следующую команду из очереди:

{% highlight java %}
private void nextCommand() {
    // If there is still a command being executed then bail out
    if(commandQueueBusy) {
        return;
    }

    // Check if we still have a valid gatt object
    if (bluetoothGatt == null) {
        Log.e(TAG, String.format("ERROR: GATT is 'null' for peripheral '%s', clearing command queue", getAddress()));
        commandQueue.clear();
        commandQueueBusy = false;
        return;
    }

    // Execute the next command in the queue
    if (commandQueue.size() > 0) {
        final Runnable bluetoothCommand = commandQueue.peek();
        commandQueueBusy = true;
        nrTries = 0;

        bleHandler.post(new Runnable() {
            @Override
            public void run() {
                    try {
                        bluetoothCommand.run();
                    } catch (Exception ex) {
                        Log.e(TAG, String.format("ERROR: Command exception for device '%s'", getName()), ex);
                    }
            }
        });
    }
}
{% endhighlight %}

Обратите внимание, мы используем метод `peek()` для получения объекта `Runnable` из очереди, чтобы можно было повторить запуск позже. Этот метод не удаляет объект из очереди. 

Результат чтения будет отправлен в ваш колбек:

{% highlight java %}
@Override
public void onCharacteristicRead(BluetoothGatt gatt, 
                                final BluetoothGattCharacteristic characteristic, 
                                int status) {
    // Perform some checks on the status field
    if (status != GATT_SUCCESS) {
        Log.e(TAG, String.format(Locale.ENGLISH,"ERROR: Read failed for characteristic: %s, status %d", characteristic.getUuid(), status));
        completedCommand();
        return;
    }

    // Characteristic has been read so processes it   
    ...
    // We done, complete the command
    completedCommand();
}
{% endhighlight %}

Чтобы одновременный вызов другой команды и избежать состояния гонки, мы завершаем команду `completedCommand()` после обработки нового значения. 

Теперь мы готовы завершить команду, убираем `Runnable` из очереди через вызов `poll()` и запускаем следующую из очереди:

{% highlight java %}
private void completedCommand() {
    commandQueueBusy = false;
    isRetrying = false;
    commandQueue.poll();
    nextCommand();
}
{% endhighlight %}

В некоторых случаях (ошибка, неожиданное значение), вам нужно будет повторить команду. Сделать это просто, так как объект `Runnable` остается в очереди до вызова `completedCommand()`. Чтобы не уйти в бесконечное повторение – проверям лимит на повторы:
{% highlight java %}
private void retryCommand() {
    commandQueueBusy = false;
    Runnable currentCommand = commandQueue.peek();
    if(currentCommand != null) {
        if (nrTries >= MAX_TRIES) {
            // Max retries reached, give up on this one and proceed
            Log.v(TAG, "Max number of tries reached");
            commandQueue.poll();
        } else {
            isRetrying = true;
        }
    }
    nextCommand();
}
{% endhighlight %}

## Запись характеристик
Чтение характеристики достаточно простая операция, а запись требует допольнительных пояснений. Для выполнения записи нужно предоставить **характеристику**, **массив байтов** и **тип записи**. Существует несколько типов записи, важные для нас это: 
- `WRITE_TYPE_DEFAULT` (вы получите ответ от устройства, например, код завершения);
- `WRITE_TYPE_NO_RESPONSE` (никакого ответа от устройства не будет).

Использовать тот или иной тип зависит от вашего устройства и характеристики (иногда она поддерживает оба типа записи, иногда только один конкретный тип).

В Android каждая характеристика имеет дефолтный тип записи, который определяется при ее создании. Ниже фрагмент кода из исходников Android, где определяется тип:

{% highlight java %}
...
if ((mProperties & PROPERTY_WRITE_NO_RESPONSE) != 0) {
    mWriteType = WRITE_TYPE_NO_RESPONSE;
} else {
    mWriteType = WRITE_TYPE_DEFAULT;
}
...
{% endhighlight %}

Как вы видите, это работает нормально, если характеристика поддерживает только один их двух типов записи. Если характеристика поддерживает оба типа, то значение по умолчанию будет `WRITE_TYPE_NO_RESPONSE`. Имейте это ввиду!

Перед записью можно проверить характеристику, поддерживает ли она нужный тип записи:
{% highlight java %}
// Check if this characteristic actually supports this writeType
int writeProperty;
switch (writeType) {
    case WRITE_TYPE_DEFAULT: writeProperty = PROPERTY_WRITE; break;
    case WRITE_TYPE_NO_RESPONSE : writeProperty = PROPERTY_WRITE_NO_RESPONSE; break;
    case WRITE_TYPE_SIGNED : writeProperty = PROPERTY_SIGNED_WRITE; break;
    default: writeProperty = 0; break;
}
if((characteristic.getProperties() & writeProperty) == 0 ) {
    Log.e(TAG, String.format(Locale.ENGLISH,"ERROR: Characteristic <%s> does not support writeType '%s'", characteristic.getUuid(), writeTypeToString(writeType)));
    return false;
}
{% endhighlight %}

Я рекомендую всегда явно указывать тип записи и не полагаться на дефолтные настройки выбранные Android! 

Итак, запись массива байтов `bytesToWrite` в характеристику выглядит так: 
{% highlight java %}
characteristic.setValue(bytesToWrite);
characteristic.setWriteType(writeType);
if (!bluetoothGatt.writeCharacteristic(characteristic)) {
    Log.e(TAG, String.format("ERROR: writeCharacteristic failed for characteristic: %s", characteristic.getUuid()));
    completedCommand();
} else {
    Log.d(TAG, String.format("writing <%s> to characteristic <%s>", bytes2String(bytesToWrite), characteristic.getUuid()));
    nrTries++;
}
{% endhighlight %}

## Включение/выключение уведомлений
Кроме самостоятельного чтения и записи характеристик, вы можете включить или отключить уведомления от устройств. При включении уведомления, устройство сообщит вам о появлении новых данных и отправит их автоматически.

Для включения уведомлений нужно сделать две вещи в Android:
1. вызвать `setCharacteristicNotification`. Bluetooth стек будет ожидать уведомления для этой характеристики.
2. записать **1** или **2**  как `unsigned int16` в дескриптор конфигурации характеристик (Client Characteristic Configuration, сокращенно - ССС). Дескриптор CCC имеет короткий UUID **2902**.

Почему **1** или **2**? Потому что внутри обмена данными есть Уведомление и Индикация. Полученное Уведомление не подтверждаются стеком Bluetooth, а Индикация наоборот подтверждается стеком. При использовании Индикации, устройство будет точно знать, что данные получены и может их, например, удалить из локального хранилища. С точки зрения Android приложения нет разницы: в обоих случаях вы просто получите массив байтов и Bluetooth стек уведомит устройство об этом, если вы используете Индикацию. Итак, **1** включает уведомления, **2** – индикацию. Чтобы выключить их, записываем **0**. Вы должны самостоятельно определить, что записать в дескриптор CCC.

В iOS метод `setNotify()` делает всю работу за вас. Ниже пример, как сделать тоже самое на Android, там сначала идут проверки входных параметров, определяется что записать в дескриптор и наконец команда отправляется в очередь:

{% highlight java %}
private final String CCC_DESCRIPTOR_UUID = "00002902-0000-1000-8000-00805f9b34fb";
public boolean setNotify(BluetoothGattCharacteristic characteristic, 
                        final boolean enable) {
    // Check if characteristic is valid
    if(characteristic == null) {
        Log.e(TAG, "ERROR: Characteristic is 'null', ignoring setNotify request");
        return false;
    }

    // Get the CCC Descriptor for the characteristic
    final BluetoothGattDescriptor descriptor = characteristic.getDescriptor(UUID.fromString(CCC_DESCRIPTOR_UUID));
    if(descriptor == null) {
        Log.e(TAG, String.format("ERROR: Could not get CCC descriptor for characteristic %s", characteristic.getUuid()));
        return false;
    }

    // Check if characteristic has NOTIFY or INDICATE properties and set the correct byte value to be written
    byte[] value;
    int properties = characteristic.getProperties();
    if ((properties & PROPERTY_NOTIFY) > 0) {
        value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE;
    } else if ((properties & PROPERTY_INDICATE) > 0) {
        value = BluetoothGattDescriptor.ENABLE_INDICATION_VALUE;
    } else {
        Log.e(TAG, String.format("ERROR: Characteristic %s does not have notify or indicate property", characteristic.getUuid()));
        return false;
    }
    final byte[] finalValue = enable ? value : BluetoothGattDescriptor.DISABLE_NOTIFICATION_VALUE;

    // Queue Runnable to turn on/off the notification now that all checks have been passed
    boolean result = commandQueue.add(new Runnable() {
        @Override
        public void run() {
            // First set notification for Gatt object  if(!bluetoothGatt.setCharacteristicNotification(descriptor.getCharacteristic(), enable)) {
                Log.e(TAG, String.format("ERROR: setCharacteristicNotification failed for descriptor: %s", descriptor.getUuid()));
            }

            // Then write to descriptor
            descriptor.setValue(finalValue);
            boolean result;
            result = bluetoothGatt.writeDescriptor(descriptor);
            if(!result) {
                Log.e(TAG, String.format("ERROR: writeDescriptor failed for descriptor: %s", descriptor.getUuid()));
                completedCommand();
            } else {
                nrTries++;
            }
        }
    });

    if(result) {
        nextCommand();
    } else {
        Log.e(TAG, "ERROR: Could not enqueue write command");
    }

    return result;
}
{% endhighlight %}

Результат записи в CCC дескриптор обрабатывается в колбеке `onDescriptorWrite`. Здесь вы должны отличить запись в CCC от записей в другие дескрипторы. Во время обработки колбека, мы также должны хранить какие в данный момент характеристики уведомляются.

{% highlight java %}
@Override
public void onDescriptorWrite(BluetoothGatt gatt, 
                                final BluetoothGattDescriptor descriptor, 
                                final int status) {
    // Do some checks first
    final BluetoothGattCharacteristic parentCharacteristic = descriptor.getCharacteristic();
    if(status!= GATT_SUCCESS) {
        Log.e(TAG, String.format("ERROR: Write descriptor failed value <%s>, device: %s, characteristic: %s", bytes2String(currentWriteBytes), getAddress(), parentCharacteristic.getUuid()));
    }

    // Check if this was the Client Configuration Descriptor  if(descriptor.getUuid().equals(UUID.fromString(CCC_DESCRIPTOR_UUID))) {
        if(status==GATT_SUCCESS) {
            // Check if we were turning notify on or off
            byte[] value = descriptor.getValue();
            if (value != null) {
                if (value[0] != 0) {
                    // Notify set to on, add it to the set of notifying characteristics          notifyingCharacteristics.add(parentCharacteristic.getUuid());
                    }
                } else {
                    // Notify was turned off, so remove it from the set of notifying characteristics               notifyingCharacteristics.remove(parentCharacteristic.getUuid());
                }
            }
        }
        // This was a setNotify operation
        ....
    } else {
        // This was a normal descriptor write....
        ...
        });
    }
    completedCommand();
}
{% endhighlight %}

Чтобы узнать из какой характеристики пришло уведомление – используйте метод `isNotifying()`:

{% highlight java %}
public boolean isNotifying(BluetoothGattCharacteristic characteristic) {
    return notifyingCharacteristics.contains(characteristic.getUuid());
}
{% endhighlight %}

## Лимиты на установку уведомлений


## Подключение к устройству
После удачного сканирования, вы должны подключиться к устройству, вызывая метод `connectGatt()`. В результате мы получаем объект – `BluetoothGatt`, который будет использоваться для всех [GATT операций](https://developer.android.com/reference/android/bluetooth/BluetoothGatt){:target="_blank"}, такие как чтение и запись характеристик. Однако будьте внимательны, есть две версии метода `connectGatt()`. Поздние версии Android имеют еще несколько вариантов, но нам нужна совместимость с Android-6 и мы рассматриваем только эти две:

{% highlight java %}
BluetoothGatt connectGatt(Context context, boolean autoConnect,
        BluetoothGattCallback callback)
BluetoothGatt connectGatt(Context context, boolean autoConnect,
        BluetoothGattCallback callback, int transport)
{% endhighlight %}
 
Внутренняя реализация первой версии – это фактически вызов второй версии с аргументом `transport = TRANSPORT_AUTO`. Для подключения BLE устройств такой вариант не подходит. `TRANSPORT_AUTO` используется для устройств с поддержкой и BLE и классического Bluetooth протоколов. Это значит, что Android будет сам выбирать протокол подключения. Этот момент практически нигде не описан и может привести к непредсказуемым результатам, много людей сталкивались с такой проблемой. Вот почему вы должны использовать вторую версию `connectGatt()` с `transport = TRANSPORT_LE`:

{% highlight java %}
BluetoothGatt gatt = device.connectGatt(context, false, 
    bluetoothGattCallback, TRANSPORT_LE);
{% endhighlight %}

Первый аргумент – `context` приложения.
Второй аргумент – флаг `autoconnect`, говорит подключаться немедленно (`false`) или нет (`true`). При немедленном подключении (`false`) Android будет пытаться соединиться в течение 30 секунд (на большинстве смартфонов), по истечении этого времени придет статус соединения `status_code = 133`. Это не официальная ошибка для таймаута соединения. В исходниках Android код фигурирует как `GATT_ERROR`. К сожалению, эта ошибка появляется и в других случаях. Имейте ввиду, с `autoconnect = false` Android делает соединение только с одним устройством в одно и то же время (это значит если у вас несколько устройств - подключайте их последовательно, а не паралелльно).
Третий аргумент – функция обратного вызова `BluetoothGattCallback` (callback) для конкретного устройства. Этот колбек используется для всех связанных с устройством операциях, такие как чтение и запись. Мы рассмотрим это более детально в следующей статье.

## Autoconnect = true
Если вы установите `autoconnect = true`, Android будет подключаться самостоятельно к устройству всякий раз, когда оно будет обнаружено. Внутри это работает так: Bluetooth стек сканирует сохраненные устройства и когда увидит одно из них – подключается к нему. Это довольно удобно, если вы хотите подключиться к конкретному устройству, когда оно становится доступным. Фактически, это предпочтительный способ для переподключения. Вы просто создаете `BluetoothDevice` объект и вызываете `connectGattwith` с `autoconnect = true`.

{% highlight java %}
BluetoothDevice device = 
bluetoothAdapter.getRemoteDevice("12:34:56:AA:BB:CC");

BluetoothGatt gatt = 
device.connectGatt(context, true, bluetoothGattCallback, TRANSPORT_LE);
{% endhighlight %}

Обратите внимание, этот подход работает только, если устройство есть в Bluetooth кеше или устройство было уже сопряжено (bonding). Посмотрите мою [предыдущую статью](https://fmaxx.github.io/android/ble/bluetoothlowenergy/2019/11/12/android-ble-part-1.html){:target="_blank"}, где подробно объясняется работа с Bluetooth кешем.
При перезагрузке смартфона или выключении/включении Bluetooth (а также Airplane режима) – кеш очистится, это надо проверять перед подключением с `autoconnect = true`, что действительно раздражает.
> Autoconnect работает только с закешированными и сопряженными (bonded) устройствами!

Для того, чтобы узнать, закешировано устройство или нет, можно использовать небольшой трюк. После создания объекта `BluetoothDevice`, вызовите у него `getType`, если результат – `TYPE_UNKNOWN`, значит устройство не закешировано. В этом случае, необходимо просканировать устройство с этим мак-адресом (используя не агрессивный метод сканирования) и после этого можно использовать автоподключение снова.

Android-6 и ниже имеет известный баг, в котором возникает гонка состояний и автоматическое подключение становится обычным (`autoconnect = false`). К счастью, умные ребята из Polidea нашли [решение для этого](https://github.com/Polidea/RxAndroidBle/blob/7663a1ab96605dc26eba378a9e51747ad254b229/rxandroidble/src/main/java/com/polidea/rxandroidble2/internal/util/BleConnectionCompat.java){:target="_blank"}. Настоятельно рекомендуется использовать его, если думаете использовать автоподключение.

Преимущества:
- работает достаточно хорошо на современных версиях Android (прим. переводчика - от Android-8 и выше);
- возможность подключаться к нескольким устройствам одновременно;

Недостатки:
- работает медленнее (Android в этом случае сканирует в режиме `SCAN_MODE_LOW_POWER`, экономя энергию), если сравнивать сканирование в агрессивном режиме + подключение с `autoconnect = false`;

## Изменения статуса подключения
После вызова `connectGatt()`, Bluetooth стек присылает результат в колбек `onConnectionStateChange`, он вызывается при любом изменении соединения. 

Работа с этим колбеком – достаточно нетривиальная вещь. Большинство простых примеров из сети выглядит так (не обольщайтесь):
{% highlight java %}
public void onConnectionStateChange(final BluetoothGatt gatt, 
                                    final int status, 
                                    final int newState) {
    if (newState == BluetoothProfile.STATE_CONNECTED) {
        gatt.discoverServices();
    } else {
        gatt.close();
    }
}
{% endhighlight %}

Этот код обрабатывает только аргумент `newState` и полностью игнорирует `status`. В многих случаях это работает и кажется безошибочным. Действительно, после подключения, следующее что нужно сделать – это вызвать `discoverServices()`. А в случае отключения - необходимо сделать вызов `close()`, чтобы Android освободил все связанные ресурсы в стеке Bluetooth. Эти два момента очень важные для стабильной работы BLE под Android, давайте их обсудим прямо сейчас!

При вызове `connectGatt()`, Bluetooth стек регистрирует внутри себя интерфейс для нового клиента (`client interface: clientIf`). 

Возможно вы заметили такие логи в LogCat:
{% highlight text %}
D/BluetoothGatt: connect() - device: B0:49:5F:01:20:XX, auto: false
D/BluetoothGatt: registerApp()
D/BluetoothGatt: registerApp() — UUID=0e47c0cf-ef13–4afb-9f54–8cf3e9e808d5
D/BluetoothGatt: onClientRegistered() — status=0 clientIf=6
{% endhighlight %}

Здесь видно, что клиент `6` был зарегистрирован после вызова `connectGatt()`. Максимальное количество клиентов (подключения) у Android равно 30 (константа `GATT_MAX_APPS` в исходниках), при достижении которого – Android не будет подключаться к устройствам вообще и вы будете получать постоянно ошибку подключения. Достаточно странно, но сразу после загрузки Android уже имеет 5 или 6 таких подключенных клиентов, предполагаю, что Android использует их для внутренних нужд. Таким образом, если вы не вызываете метод `close()`, то счетчик клиентов увеличивается каждый раз при вызове `connectGatt()`. Когда вы вызываете `close()`, Bluetooth стек удаляет ваш колбек, счетчик клиентов уменьшается на единицу и освобождает ресурсы клиента. 

{% highlight text %}
D/BluetoothGatt: close()
D/BluetoothGatt: unregisterApp() — mClientIf=6
{% endhighlight %}

Важно всегда вызывать `close()` после отключения! А сейчас обсудим основные случаи дисконнекта устройств. 

## Состояние подключения (newState)
Переменная `newState` содержит новое состояние подключения и может иметь 4 значения:
- `STATE_CONNECTED`
- `STATE_DISCONNECTED`
- `STATE_CONNECTING`
- `STATE_DISCONNECTING`

Значения говорят сами за себя. Хотя состояния `STATE_CONNECTING`, `STATE_DISCONNECTING` на практике я их не встречал. Так что, в принципе, можно не обрабатывать их, но для уверенности, я предлагаю их явно учитывать (_прим. переводчика - и это лучше, чем не обрабатывать их._), вызывая `close()` только в том случае если устройство действительно отключено.

## Статус подключения (status)
В примере выше, переменная статуса `status` полностью игнорировалась, но в действительности обрабатывать ее важно. Эта переменная, по сути, является кодом ошибки. Вы можете получить `GATT_SUCCESS` в результате как подключения, так и контролируемого отключения. Таким образом, мы можем по-разному обрабатывать контролируемое или внезапное отключение устройства. 
Если вы получили значение отличное от `GATT_SUCCESS`, значит «что-то пошло не так» и в `status` будет указана причина. К сожалению, объект `BluetoothGatt` дает очень мало кодов ошибок, все они описаны [здесь](https://android.googlesource.com/platform/external/bluetooth/bluedroid/+/5738f83aeb59361a0a2eda2460113f6dc9194271/stack/include/gatt_api.h){:target="_blank"}. Чаще всего вы будете встречаться с кодом 133 (`GATT_ERROR`). Который не имеет точного описания, и просто говорит – "произошла какая-то ошибка". Не очень информативно, подробнее об `GATT_ERROR` позже.

Теперь мы знаем, что обозначают переменные `newState` и `status`, давайте улучшим наш колбек `onConnectionStateChange`:

{% highlight java %}
public void onConnectionStateChange(final BluetoothGatt gatt, 
                                    final int status, 
                                    final int newState) {
if(status == GATT_SUCCESS) {
    if (newState == BluetoothProfile.STATE_CONNECTED) {
        // Мы подключились, можно запускать обнаружение сервисов
        gatt.discoverServices();
    } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
        // Мы успешно отключились (контролируемое отключение)
        gatt.close();
    } else {
        // мы или подключаемся или отключаемся, просто игнорируем эти статусы
    }
} else {
   // Произошла ошибка... разбираемся, что случилось!
   ...
   gatt.close();
} 
{% endhighlight %}
Это не последний вариант, мы еще улучшим колбек в этой статье. В любом случае, теперь у нас есть обработка ошибок и успешных операций.

## Состояние bonding (bondState)
Последний параметр, который необходимо учитывать в колбеке `onConnectionStateChange` – это `bondState`, состояние сопряжения (bonding) с устройством. Мы получаем этот параметр так:
{% highlight java %}
int bondstate = device.getBondState();
{% endhighlight %}
Состояние bonding может иметь одно из трех значений `BOND_NONE`, `BOND_BONDING` or `BOND_BONDED`. Каждое из них влияет на то, как обрабатывать подключение.
- `BOND_NONE`, нет проблем, можно вызывать `discoverServices()`;
- `BOND_BONDING`, устройство в процессе сопряжения, нельзя вызывать `discoverServices()`, так как Bluetooth стек в работе и запуск `discoverServices()` может прервать сопряжение и вызвать ошибку соединения. `discoverServices()` вызываем только после того, как пройдет сопряжение (bonding);
- `BOND_BONDED`, для Android-8 и выше, можно запускать `discoverServices()` без задержки. Для версий 7 и ниже может потребоваться задержка перед вызовом. Если ваше устройство имеет **Service Changed Characteristic**, то Bluetooth стек в этот момент еще обрабатывает их и запуск `discoverServices()` без задержки может вызвать ошибку соединения. Добавьте 1000-1500мс задержки, конкретное значение зависит от количества характеристик на устройстве. Используйте задержку всегда, если вы не знаете сколько **Service Changed Characteristic** имеет устройство.

Теперь мы можем учитывать состояние `bondState` вместе с `status` и `newState`:
{% highlight java %}
if (status == GATT_SUCCESS) {
    if (newState == BluetoothProfile.STATE_CONNECTED) {
        int bondstate = device.getBondState();
        // Обрабатываем bondState
        if(bondstate == BOND_NONE || bondstate == BOND_BONDED) {
            // Подключились к устройству, вызываем discoverServices с задержкой
            int delayWhenBonded = 0;
            if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.N) {
                delayWhenBonded = 1000;
            }
            final int delay = bondstate == BOND_BONDED ? delayWhenBonded : 0;
            discoverServicesRunnable = new Runnable() {
                @Override
                public void run() {
                    Log.d(TAG, String.format(Locale.ENGLISH, "discovering services of '%s' with delay of %d ms", getName(), delay));
                    boolean result = gatt.discoverServices();
                    if (!result) {
                        Log.e(TAG, "discoverServices failed to start");
                    }
                    discoverServicesRunnable = null;
                }
            };
            bleHandler.postDelayed(discoverServicesRunnable, delay);
        } else if (bondstate == BOND_BONDING) {
            // Bonding в процессе, ждем когда закончится
            Log.i(TAG, "waiting for bonding to complete");
        }
....
{% endhighlight %}

## Обработка ошибок
После того как мы разобрались с успешными операциями, давайте взглянем на ошибки. Есть ряд ситуаций, которые на самом деле "нормальные", но выдают себя за ошибки.
- **Устройство отключилось намеренно**. Например, все данные были переданы и больше ему нечего делать. Вы получите статус - 19 (`GATT_CONN_TERMINATE_PEER_USER`); 
- **Истекло время ожидания соединения  и устройство отключилось само**. В этом случае придет статус - 8 (`GATT_CONN_TIMEOUT`);
- **Низкоуровневая ошибка соединения, которая привела к отключению**. Обычно это статус - 133 (`GATT_ERROR`) или более конкретный код, если повезет;
- **Bluetooth стек не смог подключится ни разу**. Здесь также получим статус - 133 (`GATT_ERROR`);
- **Соединение было потеряно в процессе `bonding` или `discoverServices`**. Необходимо выяснить причину и возможно повторить попытку подключения.

Первые два случая абсолютно нормальные явления и все что нужно сделать - это вызывать `close()` и подчистить ссылки на объект `BluetoothGatt`, если необходимо.
В остальных случаях, либо ваш код, либо устройство, что-то делает не так. Вы возможно захотите уведомить UI или другие части приложения о проблеме, повторить подключение или еще каким-то образом отреагировать на ситуацию.
Взгляните как я сделал это в [моей библиотеке](https://github.com/weliem/blessed-android/blob/master/blessed/src/main/java/com/welie/blessed/BluetoothPeripheral.java#L234){:target="_blank"}. 

## Статус 133 при подключении (connecting)
Статус - 133 часто встречается при попытках подключиться к устройству, особенно во время разработки. Этот статус может иметь множество причин, некоторые из них можно контролировать:
- Убедитесь, что вы всегда вызываете `close()` при отключении. Если этого не сделать, в следующий раз при подключении вы точно получите `status=133`; 
- Всегда используйте `TRANSPORT_LE` в вызове `connectGatt()`;
- Перезагрузите смартфон. Возможно Bluetooth стек выбрал лимит по клиентским подключениям или есть внутренняя проблема. (_Прим. переводчика: я сначала выключал/включал Bluetooth, потом Airplane режим и если не помогало - перезагружал_);
- Проверьте что устройство посылает advertising пакеты. Вызов `connectGatt()` с `autoconnect = false` имеет таймаут 30 секунд, после чего присылает ошибку `status=133`;
- Замените/зарядите батарею на устройстве. Обычно устройства работают нестабильно при низком заряде;

Если вы попробовали все способы выше и все еще получаете статус 133, необходимо просто **повторить подключение**! Это одна из Android ошибок, которую мне так и не удалось понять или решить. Иногда вы получаете 133 при подключении к устройству, но если вызывать `close()` и переподключиться, то все работает без проблем! Есть подозрение, что проблема в кеше Android и вызов `close()` сбрасывает его состояние для конкретного устройства. Если кто-нибудь поймет, как решить эту проблему – дайте мне знать! 

## Отключение по запросу (disconnect)
Для отключения устройства вам необходимо сделать шаги:
- вызвать `disconnect()`;
- подождать обновления статуса в `onConnectionStateChange`;
- вызвать `close()`;
- освободить связанные с объектом gatt ресурсы;

Команда `disconnect()` фактически разрывает соединение с устройством и обновляет внутреннее состояние Bluetooth стека. Затем вызывается колбек `onConnectionStateChange` с новым состоянием «disconnected».

Вызов `close()` удаляет ваш  `BluetoothGattCallback`  и освобождает клиента в Bluetooth стеке.

Наконец, удаление `BluetoothGatt` освободит все связанные с подключением ресурсы.

## Отключение «неправильно»
В примерах из сети можно увидеть, разные примеры отключения, например:

- вызвать `disconnect()`
- **сразу** вызвать `close()` 

Это будет работать более-менее. Да устройство отключится, но вы никогда не получите вызов колбека с состоянием «disconnected». Дело в том, что `disconnect()` операция асинхронная (не блокирует поток и имеет свое время выполнения), а `close()` немедленно удаляет коллбек! Получается, когда Android будет готов вызвать колбек, его уже не будет.

Иногда в примерах не вызывают `disconnect()`, а только `close()`. Это приведет к отключению устройства, но это неправильный способ, поскольку `disconnect()` отключает активное соединение и отменяет ожидающее автоматическое подключение (вызов с `autoconnect = true`). Поэтому, если вы вызываете только `close()`, любое ожидающее автоподключение может привести к новому подключению.

## Отмена попытки подключения
Если вы хотите отменить подключение после `connectGatt()`, вам нужно вызвать `disconnect()`. Так как в этому моменту вы еще не подключены, колбек `onConnectionStateChange` не сработает! Просто подождите некоторое время после `disconnect()` и после этого вызывайте `close()` (_прим. переводчика: обычно это 50-100мс_).

При удачной отмене вы увидите примерно такое в логах:
{% highlight text %}
D/BluetoothGatt: cancelOpen() — device: CF:A9:BA:D9:62:9E
{% endhighlight %}

Скорее всего, вы никогда не отмените соединение, для параметра `autoconnect = false`. Часто это делается для подключений с `autoconnect = true`. Например, когда приложение на переднем плане – вы подключаетесь к вашим устройствам и отключаетесь от них, если приложение переходит в фон.

_Прим. переводчика: но это не значит что для `autoconnect = false` не надо проводить такую отмену! Скорее всего вы не увидите этого в логах._

## Обнаружение сервисов (discovering services)
Как только вы подключились к устройству, необходимо запустить обнаружение его сервисов вызовом `discoverServices()`. Bluetooth стек запустит серию низкоуровневых команд для получения **сервисов**, **характеристик** и **дескрипторов**. Это занимает обычно около одной секунды в зависимости от того сколько таких служб, характеристик, дескрипторов имеет ваше устройство. В результате будет вызыван колбек **onServicesDiscovered**.

Первым делом проверим, есть ли какие ошибки после обнаружения сервисов:
{% highlight java %}
// Проверяем есть ли ошибки? Если да - отключаемся
if (status == GATT_INTERNAL_ERROR) {
    Log.e(TAG, "Service discovery failed");
    disconnect();
    return;
}
{% endhighlight %}

Если есть ошибки (обычно это `GATT_INTERNAL_ERROR` со значением 129), делаем отключение устройства, что-то исправить здесь невозможно (нет специальных технических способов для этого). Вы просто отключаете устройство и повторно пробуете подключиться.

Если все прошло удачно, вы получите список сервисов:
{% highlight java %}
final List<BluetoothGattService> services = gatt.getServices();
Log.i(TAG, String.format(Locale.ENGLISH,"discovered %d services for '%s'", services.size(), getName()));
// Работа со списком сервисов (если требуется)
...
{% endhighlight %}

## Кеширование сервисов.
Bluetooth стек кеширует найденные на устройстве сервисы, характеристики и дескрипторы. Первое подключение вызывает реальное обнаружение сервисов, все последующие – возвращаются кешированные версии. Это соответствует стандарту Bluetooth. Обычно это нормально и сокращает время соединения с устройством.
Однако в некоторых случаях, может потребоваться очистить кеш, чтобы снова обнаружить их с устройства при следующем соединении. Типичный сценарий: обновление прошивки, в которой изменяется набор сервисов, характеристик, дескрипторов. Есть  скрытый метод очистки кеша и добраться до него нам поможет рефлексия:

{% highlight java %}
private boolean clearServicesCache() {
    boolean result = false;
    try {
        Method refreshMethod = bluetoothGatt.getClass().getMethod("refresh");
        if(refreshMethod != null) {
            result = (boolean) refreshMethod.invoke(bluetoothGatt);
        }
    } catch (Exception e) {
        Log.e(TAG, "ERROR: Could not invoke refresh method");
    }
    return result;
}
{% endhighlight %}

Этот метод асинхронный, дайте ему некоторое время для завершения!

## Странные штуки в подключении/отключении
Хотя операции подключения и отключения выглядят просто, есть некоторые особенности, которые нужно знать.
- Случайная ошибка 133 при подключении, выше мы разобрались как с ней работать;
- Периодическое зависание подключения, не срабатывает таймаут и не вызывается колбек `onConnectionStateChange`. Это случается не часто, но я видел такие случае при низком уровне батареи или когда устройство находится на границе доступности по расстоянию Bluetooth. Скорее всего общение с устройством происходит, но затем прерывается и зависает. Мой обходной путь – использовать свой таймер подключения и в случае таймаута – закрывать соединение и отключаться;
- Некоторые смартфоны имеют проблему с подключением во время сканирования. Например, Huawei P8 Lite один из таких. Останавливаем сканнер перед любым подключением (_Прим. переводчика: это правило соблюдаем строго!_);
- Все вызовы подключения/отключения асинхронные. То есть неблокирующие, но при этом им нужно время, чтобы выполнится до конца. Избегайте быстрый запуск их друг за другом (_Прим. переводчика: я обычно использую задержку 50-100мс между вызовами_).

### Следующая статья: чтение и запись характеристик.
Теперь мы разобрались с подключением/отключением и обнаружением сервисов, следующая статья – о том, как работать с характеристиками.
> Не терпится поработать с BLE? Попробуйте [мою библиотеку Blessed for Android](https://github.com/weliem/blessed-android){:target="_blank"}. Она использует все подходы из этой серии статей и упрощает работу с BLE в вашем приложении.




