---
layout:     post
title:      Android, работа с BLE - часть 4.
date:       2020-01-25 10:03:00
summary:    Разбираемся с Android Bluetooth Low Energy.
categories: Android BLE BluetoothLowEnergy
---

Перевод статьи [Making Android BLE work — part 4](https://medium.com/@martijn.van.welie/making-android-ble-work-part-4-72a0b85cb442){:target="_blank"}.
> внимание: в цикле статей используется минимальная версия - Android 6

В [предыдущей статье](https://fmaxx.github.io/android/ble/bluetoothlowenergy/2020/01/18/android-ble-part-3.html) мы разобрались с операциями чтения/записи, включения/выключения нотификаций и организации очереди команд. В этой статье мы поговорим о **спряжении устройств** (_Прим. переводчика – далее я буду использовать «bonding»_).


![devices](/images/2021-01-25-android-ble-part-4/1.jpeg)

## Bonding
Некоторые устройства для правильной работы требуют bonding. Технически это обозначает, что генерируются ключи шифрования, обмениваются и хранятся, для более безопасного обмена данными. При запуске процедуры bonding, Android может запросить у пользователя согласие, пин-код или кодовую фразу. При следующих подключениях, Android уже знает, что устройство сопряжено и обмен ключами шифрования происходит скрытно без участия пользователя.  Использование bonding делает подключение к устройству более безопасным, так как соединение зашифровано.

Тема bonding плохо описана в документации Google, полностью непонятно как приложение должно работать с bonding. Метод `createBond()`, это первое на что вы заметите. В iOS такого метода вообще нет и `CoreBluetooth` делает все за вас! Тогда зачем вызывать `createBond()`? Мне показалось это странным, как будто вам заранее надо знать, какие устройства требуют bonding, а какие нет. Протокол Bluetooth был спроектирован так, что обычно устройства явно говорят – им требуется или нет bonding. Я копнул немного глубже и поэкспериментировал. Потребовалось некоторое время, чтобы разобраться с этим, но в конце концов все оказалось просто.

Принципы работы с bonding: 
- **Пусть Android сам работает с bonding** Andriod сделает bonding за вас, когда устройство скажет, что нужен bonding, или во время операции чтения/записи зашифрованной характеристики. В большинстве случаев не надо вызывать `createBond()` самостоятельно (_Прим. переводчика: мне пришлось это делать самостоятельно, из-за особенностей прошивки устройства. Кроме того, Samsung работает по-другом, чем другие вендоры._);
- ;











## Чтение и запись характеристик
Многие разработчики, которые начинают работать с BLE на Android, сталкиваются с проблемами чтения/записи BLE характеристик. На [Stackoverflow](https://stackoverflow.com/search?q=android+ble+reading+writing+characteristic){:target="_blank"} полно людей, предлагающих просто использовать задержки... Большинство таких советов неверные. 

Есть две основные причины проблем:
- **Операции чтения/записи асинхронные**. Это значит, что вызов метода вернется немедленно, но результат вызова вы получите немного позже – в соответствующих колбеках. Например `onCharacteristicRead()` или `onCharacteristicWrite()`.
- **Одновременно может быть запущена только одна операция**. Нужно дождаться выполнения текущей операции, и затем, запускать следующую. В исходном коде `BluetoothGatt` есть блокирующая переменная, которая при запуске операции устанавливается и при вызове колбека сбрасывается. Google забыла про это упомянуть в документации... (_Прим. переводчика: речь идет о `mDeviceBusy` и `mDeviceBusyLock` [здесь](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/bluetooth/BluetoothGatt.java){:target="_blank"}_).

Первая причина, на самом деле, не является проблемой, такова природа BLE. Асинхронное программирование это распространенная штука, используется, например, при сетевых вызовах. Однако вторая причина раздражает и требует специального подхода.

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
Выполнять чтение/запись по одной операции за раз неудобно, но любое сложное приложение должно это учитывать. Решение этой проблемы - использование **очереди команд**. Все BLE библиотеки, которые я ранее упоминал, так или иначе реализуют очередь. Это одна из лучших практик!
Идея простая – каждая команда сначала добавляется в очередь. Затем команда забирается из очереди на исполнение, после результата, команда помечается как «завершенная» и, удаляется из очереди. Запускать команды можно в любое время, но они выполняются точно в том порядке, в котором поступают в очередь. Это очень упрощает разработку под BLE. В iOS аналогично работает фреймворк `CoreBluetooth` (_Прим. переводчика: который намного удобнее, чем реализация Bluetooth стека в Android_). 

Очередь создается для каждого объекта `BluetoothGatt`. К счастью, Android сможет обрабатывать очереди от нескольких объектов `BluetoothGatt`, вам не нужно об этом беспокоиться (_Прим. переводчика: у меня это не сработало, я использовал глобальную очередь команд для всех устройств_). Есть много способов создать очередь, мы будем использовать простую очередь `Queue` с `Runnable` для каждой команды и переменной `commandQueueBusy` для отслеживания работы команды:

{% highlight java %}
private Queue<Runnable> commandQueue;
private boolean commandQueueBusy;
{% endhighlight %}

Мы добавляем новый экземпляр `Runnable` в очередь при выполнении команды. Ниже пример чтения характеристики (readCharacteristic):

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

В этом методе, сначала проверяем все ли готово для выполнения (наличие и тип характеристики) и логгируем ошибки, если они есть. Внутри `Runnable`, фактически вызывается метод `readCharacteristic()`, который выдает команду на устройство. Мы также отслеживаем сколько было попыток, чтобы сделать повтор в случае ошибки (_Прим. переводчика: это лучшая тактика, чтобы добиться стабильной работы с устройством_). Если чтение характеристики возвращает `false`, мы логгируем ошибку, «завершаем» команду, чтобы можно было запустить следующую. Наконец вызывается `nextCommand()`, чтобы запустить следующую команду из очереди:

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


Мы завершаем команду `completedCommand()` после обработки нового значения.  Это помогает избежать одновременный вызов другой команды и состояния гонки.

Теперь мы готовы завершить команду, убираем `Runnable` из очереди через вызов `poll()` и запускаем следующую из очереди:

{% highlight java %}
private void completedCommand() {
    commandQueueBusy = false;
    isRetrying = false;
    commandQueue.poll();
    nextCommand();
}
{% endhighlight %}

В некоторых случаях (ошибка, неожиданное значение), вам нужно будет повторить команду. Сделать это просто, так как объект `Runnable` остается в очереди до вызова `completedCommand()`. Чтобы не уйти в бесконечное повторение – проверяем лимит на повторы:
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
Чтение характеристики достаточно простая операция, а запись требует дополнительных пояснений. Для выполнения записи нужно предоставить **характеристику**, **массив байтов** и **тип записи**. Существует несколько типов записи, важные для нас это: 
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

Почему **1** или **2**? Потому что «под капотом» Bluetooth стека есть Уведомление и Индикация. Полученное Уведомление не подтверждаются стеком Bluetooth, а Индикация наоборот – подтверждается стеком. При использовании Индикации, устройство будет точно знать, что данные получены и может их, например, удалить из локального хранилища. С точки зрения Android приложения нет разницы: в обоих случаях вы просто получите массив байтов и Bluetooth стек уведомит устройство об этом, если вы используете Индикацию. Итак, **1** включает уведомления, **2** – индикацию. Чтобы выключить их, записываем **0**. Вы должны самостоятельно определить, что записать в дескриптор CCC.

В iOS метод `setNotify()` делает всю работу за вас. Ниже пример, как сделать тоже самое на Android, там сначала идут проверки входных параметров, определяется что записать в дескриптор и, наконец команда отправляется в очередь:

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

Результат записи в CCC дескриптор обрабатывается в колбеке `onDescriptorWrite`. Здесь вы должны отличить запись в CCC от записей в другие дескрипторы. Во время обработки колбека, мы также должны хранить, какие в данный момент характеристики уведомляются.

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
К сожалению, нельзя включить столько уведомлений, сколько хочешь. Начиная с Android-5 лимит равен 15. В более старых версиях он был равен 7 или даже 4. Большинство смартфонов поддерживают 15 уведомлений. Не забывайте отключать их, если они вам больше не нужны, чтобы не исчерпать лимит.

## Проблемы с потоками
Итак, мы научились читать/писать характеристики, включать/выключать уведомления, а значит готовы использовать это в реальном проекте. Я думаю, что устройства BLE можно разделить на две категории:
- **Простые устройства**. Например, термометр, который использует официальный Bluetooth Health Thermometer сервис. Такие устройства легко использовать, вы просто включаете уведомления и данные начинают поступать. Здесь мы используем только операции чтения характеристики, запись не нужна;
- **Сложные устройства**. Это может быть любое устройство, но обычно все они используют свой внутренний протокол обмена данными. Часто эти протоколы не спроектированы под BLE, а просто транслируют внутренний последовательный протокол в BLE, где одна характеристика используется для отправки данных, а другая для приема. Сложность в том, что вам требуется знать большое количество команд для работы с устройством: авторизация, обновление пользовательских параметров, параметров самого устройства, получение сохраненных данных и т.д.

Простые устройства обычно не создают проблем с потоками, для сложных – следует работать внимательно. Чтение, запись и уведомления в этом случае будут чередоваться и могут мешать друг другу, особенно если у вас устройство с высокой частотой передачи данных (30Hz или около).

Типичная проблема с потоками выглядит так:
- приходит уведомление
- вы отправляете событие в свою собственную очередь для обработки
- запускается обработку полученных данных
- в это время приходит новое уведомление и перезаписывает предыдущее значение в `BluetoothGattCharacteristic`
- если ваша обработка данных медленная, вы потенциально теряете значение из первого уведомления.

Причины такого поведения:
- **как только сообщение доставлено, Android будет отправлять следующее** (если оно есть). Посколько обработка данных отправляется в другой поток, текущий освобождается и Android продолжит доставку уведомлений;
- Android **переиспользует BluetoothGattCharacteristic объекты внутри**. Они создаются в время обнаружения сервисов (services discovering) и после этого **переспользуются** многократно. Таким образом, когда приходит уведомления Android сохраняет значение в объект `BluetoothGattCharacteristic`. Если характеристика в этот момент обрабатывается в другом потоке мы получим гонку состояний (race condition) и результат будет непредсказуемым.

Очевидно, что **нужно всегда работать с копией массива байтов**. Получили данные, сразу же делаем копию и работаем с ней.

Ниже пример, который использует такую тактику:
{% highlight java %}
@Override
public void onCharacteristicChanged(BluetoothGatt gatt, 
                                    final BluetoothGattCharacteristic characteristic) {
    // Copy the byte array so we have a threadsafe copy
    final byte[] value = new byte[characteristic.getValue().length];
    System.arraycopy(
        characteristic.getValue(), 
        0, value, 0, 
        characteristic.getValue().length);

    // Characteristic has new value so pass it on for processing
    bleHandler.post(new Runnable() {
        @Override
        public void run() {         
            myProcessor.onCharacteristicUpdate(BluetoothPeripheral.this, value, characteristic);
        }
    });
}
{% endhighlight %}

## Другие рекомендации по работе с потоками
Есть несколько дополнительных рекомендаций по работе с BLE на Android. Поскольку стек BLE в основном асинхронный, у нас есть мульти-поточная обработка задач.

Android использует потоки:
- При сканировании (результаты приходят в `main` поток);
- Вызове колбеков `BluetoothGattCallback` (выполняются в потоках `Binder`);

Обработка результатов сканирования на main потоке не будет проблемой. Но с потоками Binder все немного сложнее. При вызове колбека на потоке Binder, Android не будет отправлять новые данные пока не закончится обработка текущих, то есть поток Binder блокируется пока ваш код не завершится. Следует избегать тяжелых операций в колбеках, никаких `sleep()` или что-то подобное. Кроме того, никаких новых вызовов в объекте `BluetoothGatt`, пока вы находитесь в потоке Binder, хотя большинство методов асинхронные.

Я рекомендую следующее:
- Всегда выполняйте вызовы `BluetoothGattCallback` в отдельном потоке,  возможно даже из потока пользовательского интерфейса (_Прим. переводчика: работать на main потоке - плохая идея, если у вас есть активный обмен с устройством, обязательно будут залипания UI, не делайте так)_;
- Освобождайте потоки `Binder` как можно быстрее и **никогда** не блокируйте их;

Самый простой способ выполнить рекомендации выше – создать выделенный `Handler` и использовать его для обработки данных и выдачи новых команд. Обратите внимание, я уже использовал `Handler` на примере кода для колбека `onCharacteristicUpdate`.

Объявление объекта:

{% highlight java %}
Handler bleHandler = new Handler();
{% endhighlight %}

Если хотите запустить `Handler` на `main` потоке:

{% highlight java %}
Handler bleHandler = new Handler(Looper.getMainLooper());
{% endhighlight %}

Прокрутите назад и взгляните на наш метод `nextCommand()`, каждый `Runnable` выполняется в нашем собственном `Handler`, следовательно, мы гарантируем, что все команды выполняются вне потока `Binder`.

## Следующая статья: сопряжение (bonding)
В этой статье мы разобрались с чтением и записью характеристик, включением и выключением уведомлений/нотификаций. В следующей статье, мы детально изучим процесс спряжения с устройством (bonding).

> Не терпится поработать с BLE? Попробуйте [мою библиотеку Blessed for Android](https://github.com/weliem/blessed-android){:target="_blank"}. Она использует все подходы из этой серии статей и упрощает работу с BLE в вашем приложении.