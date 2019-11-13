---
layout:     post
title:      Android, работа с BLE - часть 1.
date:       2019-10-28 10:00:00
summary:    Разбираемся с Android Bluetooth Low Energy.
categories: Android, BLE, Bluetooth Low Energy
---

Перевод статьи [Making Android BLE work — part 1](https://medium.com/@martijn.van.welie/making-android-ble-work-part-1-a736dcd53b02).
> внимание: в цикле статей используется минимальная версия - Android 6

В последний год я изучил как разрабатывать Bluetooth Low Energy (BLE) приложения под iOS и это оказалось довольно простым. Далее было портирование этого на Android... насколько это будет сложно?

![devices](/images/2019-10-30-android-ble-part-1/1.jpg)

Могу точно сказать это было сложней, чем я представлял, пришлось приложить немало усилий для стабильной работы под Android. Я изучил много статей в свободном доступе, некоторые оказались ошибочными, многие были очень полезными и помогли в деле.
В этой серии статей я хочу описать свои выводы, чтобы вы не тратили уйму времени на поиски, как я.


## Особенности работы BLE под Android:

* **Google документация по BLE очень общая**, в некоторых случаях нет важной информации или она устарела, примеры приложений не показывают как правильно использовать BLE. Я обнаружил лишь несколько источников, как правильно сделать BLE. 
[Презентация Stuart Kent](https://www.stkent.com/2017/09/18/ble-on-android.html){:target="_blank"} дает замечательный материал для старта. Для некоторых продвинутых тем есть хорошая статья [Nordic](https://devzone.nordicsemi.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-00-04-DZ-1046/2604.BLE_5F00_on_5F00_Android_5F00_v1.0.1.pdf){:target="_blank"}.


* **Android BLE API это низкоуровневые операции**, в реальных приложениях нужно использовать несколько слоев абстракции (как например сделано "из коробки" в iOS-CoreBluetooth). Обычно нужно самостоятельно сделать: очередь команд, bonding, обслуживание соединений, обработка ошибок и багов, мультипоточный доступ . Самые известные библиотеки: [SweetBlue](https://github.com/iDevicesInc/SweetBlue){:target="_blank"}, [RxAndroidBle](https://github.com/Polidea/RxAndroidBle){:target="_blank"} и [Nordic](https://github.com/NordicSemiconductor/Android-BLE-Library){:target="_blank"}. На мой взгляд самая легкая для изучения - Nordic, [см. детали тут](https://github.com/weliem/blessed-android){:target="_blank"}.

* **Производители делают изменения в Android BLE стеке** или полностью заменяют на свою реализацию. И надо учитывать разницу поведения для разных устройств в приложении. То что прекрасно работает на одном телефоне, может не работать на других! В целом не все так плохо, например реализация Samsung сделана лучше собственной реализации от Google!

* **В Android есть несколько известных (и неизвестных) багов** которые должны быть обработаны, особенно в версиях 4,5 и 6. Более поздние версии работают намного лучше, но тоже имеют определенные проблемы, такие как случайные сбои соединения с ошибкой 133. Подробнее об этом ниже.

Не претендую на то, что я решил все проблемы, но мне удалось выйти на "приемлемый" уровень. Начнем со сканирования.


## Сканирование устройств
Перед подключением к устройству вам нужно его просканировать. Это делается при помощи класса [`BluetoothLeScanner`](https://developer.android.com/reference/android/bluetooth/le/BluetoothLeScanner){:target="_blank"}:

{% highlight java %}
BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
BluetoothLeScanner scanner = adapter.getBluetoothLeScanner();

if (scanner != null) {
    scanner.startScan(filters, scanSettings, scanCallback);
    Log.d(TAG, "scan started");
}  else {
    Log.e(TAG, "could not get scanner object");
}
{% endhighlight %}

Сканер пытается обнаружить устройства в соответствии с `filters` и `scanSettings` и при нахождении устройства вызывается `scanCallback`:

{% highlight java %}
private final ScanCallback scanCallback = new ScanCallback() {
    @Override
    public void onScanResult(int callbackType, ScanResult result) {
        BluetoothDevice device = result.getDevice();
        // ...do whatever you want with this found device
    }

    @Override
    public void onBatchScanResults(List<ScanResult> results) {
        // Ignore for now
    }

    @Override
    public void onScanFailed(int errorCode) {
        // Ignore for now
    }
};
{% endhighlight %}

В результате сканирования `ScanResult` есть объект `BluetoothDevice`, его используют для подключения к устройству. Прежде чем начать подключаться, давайте поговорим о сканировании подробнее, `ScanResult` содержит несколько полезных сведений об устройстве:
* **Advertisement data** - массив байтов с информацией об устройстве, для большинства устройств это имя и UUIDы сервисов, можно задать в `filters` имя устройства и UUID сервисов для поиска конкретных устройств.
* **RSSI уровень** - уровень сигнала (насколько близко устройство). 
* ... дополнительные данные, см. документацию по `ScanResult` [здесь](https://developer.android.com/reference/android/net/wifi/ScanResult)c

> Не забывайте про жизненный цикл `Activity`, `onScanResult` может вызываться многократно для одних и тех же устройств, при пересоздании `Activity` сканирование может запускаться повторно, вызываю лавину `onScanResult`.

## Настраиваем фильтр для сканирования
Вообще можно передать null вместо фильтров и получить все устройства рядом, иногда это полезно, но чаще требуются устройства с определенным именем или набором UUID сервисов.

# Сканирование устройств по UUID сервиса
Это используется если вам необходимо найти устройства определенной категории, например мониторы артериального давления со стандартным сервисным UUID: 1810. При сканировании устройство может содержать в *Advertisement data* UUID сервис, который характеризует это устройство. На самом деле эти данные не надежные, фактически сервисы могут не поддерживаться, или подделываеться *Advertisement data* данные, в общем тут есть творческий момент.
> Прим. перевочика: одно из моих устройств со специфичной прошивкой, вообще не содержало список UUID сервисов в *Advertisement data*, хотя все остальные прошивки работали ок.

Пример сканирования службы с артериальным давлением:
{% highlight java %}
UUID BLP_SERVICE_UUID = UUID.fromString("00001810-0000-1000-8000-00805f9b34fb");
UUID[] serviceUUIDs = new UUID[]{BLP_SERVICE_UUID};
List<ScanFilter> filters = null;
if(serviceUUIDs != null) {
    filters = new ArrayList<>();
    for (UUID serviceUUID : serviceUUIDs) {
        ScanFilter filter = new ScanFilter.Builder()
                .setServiceUuid(new ParcelUuid(serviceUUID))
                .build();
        filters.add(filter);
    }
}
scanner.startScan(filters, scanSettings, scanCallback);
{% endhighlight %}

Обратите внимание, короткий UUID (например `1810`), называется `16-bit UUID` является частью длинного `128-bit UUID` (в данном случае `00001810-000000-1000-8000-000-00805f9b34fb`). Короткий UUID это BASE_PART длинного UUID, см. спецификацию [здесь](https://www.bluetooth.com/specifications/assigned-numbers/service-discovery){:target="_blank"}

# Сканирование устройств по имени
Поиск устройств использует точное совпадение имени устройства, обычно это применяется в двух случаях:
- поиск конкретного устройства
- поиск конкретной модели устройста
Например мой нагрудный напульсник Polar H7 определяется как "Polar H7 391BBB014", первая часть - "Polar H7" общая для всех таких устройств этой модели, а последняя часть "391BBB014" - уникальный серийный номер. Это очень распространненая практика. Если вы хотите найти все устройства "Polar H7", то фильтр по имени вам не поможет, придется искать подстроку у всех отсканированных устройств в `ScanResult`.
Пример с поиском **точно** по имени:
{% highlight java %}
String[] names = new String[]{"Polar H7 391BB014"};
List<ScanFilter> filters = null;
if(names != null) {
    filters = new ArrayList<>();
    for (String name : names) {
        ScanFilter filter = new ScanFilter.Builder()
                .setDeviceName(name)
                .build();
        filters.add(filter);
    }
}
scanner.startScan(filters, scanSettings, scanCallback);
{% endhighlight %}

# Сканирование устройств по MAC-адресам.
Обычно применяется для **переподключения** к уже известным устройствам. Обычно мы не знаем MAC-адрес девайса, если не сканировали его раньше, иногда адрес печатается на коробке или на корпусе самого устройства, особенно это касается медицинских приборов. Существует другой способ повторного подключения, но в некоторых случаях придется еще раз сканировать устройство, например при очистке кеша Bluetooth.

{% highlight java %}
String[] peripheralAddresses = new String[]{"01:0A:5C:7D:D0:1A"};
// Build filters list
List<ScanFilter> filters = null;
if (peripheralAddresses != null) {
    filters = new ArrayList<>();
    for (String address : peripheralAddresses) {
        ScanFilter filter = new ScanFilter.Builder()
                .setDeviceAddress(address)
                .build();
        filters.add(filter);
    }
}
scanner.startScan(filters, scanSettings, scanByServiceUUIDCallback);
{% endhighlight %}
Вероятно вы уже поняли, что можно комбинировать в фильтре UUID, имя и MAC-адрес устройства. Выглядит неплохо, но на практике я не применял такое. Хотя может быть вам это пригодится?

## Настройка ScanSettings
ScanSettings объясняют Android как он должен сканировать устройства. Там несколько настроек, которые можно задать, ниже полный пример:
{% highlight java %}
ScanSettings scanSettings = new ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_POWER)
        .setCallbackType(ScanSettings.CALLBACK_TYPE_ALL_MATCHES)
        .setMatchMode(ScanSettings.MATCH_MODE_AGGRESSIVE)
        .setNumOfMatches(ScanSettings.MATCH_NUM_ONE_ADVERTISEMENT)
        .setReportDelay(0L)
        .build();
{% endhighlight %}

Посмотрим, что они обозначают:

# **ScanMode**
Безусловно, это самый важный параметр. Определяет метод и время сканирования в Bluetooth стеке. Такая операция требует много энергии и необходим контроль над этим процессом, чтобы не разрядить батарею телефона быстро.
Есть 4 режима работы, в соответствии с руководством [Nordics](https://devzone.nordicsemi.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-00-04-DZ-1046/2604.BLE_5F00_on_5F00_Android_5F00_v1.0.1.pdf){:target="_blank"}:
1. `SCAN_MODE_LOW_POWER`. В этом режиме Android сканирует 0.5с, потом делает паузу на 4.5с. Поиск может занять относительно длительное время, зависит от того насколько часто устройство посылает пакет advertisement данных.
2. `SCAN_MODE_BALANCED`. Время сканирования 2с, время паузы - 3с, "компромисный" режим работы.
3. `SCAN_MODE_LOW_LATENCY`. В этом случае, Android сканирует непрерывно, что очевидно требует больше энергозатрат, при этом получаются лучшие результаты сканирования. Режим подходит если вы хотите найти свое устройство как можно быстрее. Не стоит использовать для длительного сканирования.
4. `SCAN_MODE_OPPORTUNISTIC`. Результаты будут получены если сканирование выполняется другими приложениями! Строго говоря, это вообще не гарантирует, что обнаружится ваше устройство. Стек Android использует этот режим в случае долгого сканирования, для понижения качества результатов. Об этом позже.

# **Callback Type**
Настройка контролирует как будет вызываться callback со `ScanResult` в соответствии с заданными фильтрами, есть 3 возможных вариантов:
1. `CALLBACK_TYPE_ALL_MATCHES`. Callback будет вызывать каждый раз, при получении advertisement пакета от устройств. На практике - каждые 200-500мс будет срабатывать сallback, в зависимости от частоты отправки advertisement пакетов устройствами.

2. `CALLBACK_TYPE_FIRST_MATCH`. Callback сработает один раз для устройства, даже если оно далее будет снова посылать advertisement пакеты.

3. `CALLBACK_TYPE_MATCH_LOST`. Callback будет вызыван если получен первый advertisement пакет от устройства и дальнейшие advertisement пакеты не обнаружены. Немного странное поведение.

В практике обычно используются настройка `CALLBACK_TYPE_ALL_MATCHES` или `CALLBACK_TYPE_FIRST_MATCH`. Правильный тип зависит от конкретного случая. Если не знаете - используйте `CALLBACK_TYPE_ALL_MATCHES`, это дает больше контроля при получении callback, если вы останавливаете сканирование после получения нужных результатов - фактически это `CALLBACK_TYPE_FIRST_MATCH`.

# **Match mode**
Настройка как Android определяет "совпадения".

1. `MATCH_MODE_AGGRESSIVE`. Агрессивность обуславливается к поиску минимального количества advertisement пакетов и устройств даже со слабым сигналом.

2. `MATCH_MODE_STICKY`. В противоположность, этот режим требует большего количества advertisement пакетов и хорошего уровня сигнала от устройств.

Я не тестировал эти настройки подробно, но я в основном использую `MATCH_MODE_AGGRESSIVE`, это помогает быстрее найти устройства.

# **Number of matches**
Параметр определяет сколько advertisement данных необходимо для совпадения.

1. `MATCH_NUM_ONE_ADVERTISEMENT`. Одного пакета достаточно.

2. `MATCH_NUM_FEW_ADVERTISEMENT`. Несколько пакетов нужно для соответствия.

3. `MATCH_NUM_MAX_ADVERTISEMENT`. Максимальное количество advertisement данных, которые устройство может обработать за один временной кадр.

Нет большой необходимости в таком низкоуровнем контроле. Все что вам надо - быстро найти свое устройство, используйте первые 2 варианта.

# **Report delay**
Задержка для вызова сallback в милисекундах. Если она больше нуля, Android будет собирать результаты в течение этого времени и вышлет их сразу все в обработчике `onBatchScanResults`. Важно понимать что `onScanResult` не будет вызываться. Обычно применяется, когда есть несколько устройств одного типа и мы хотим дать пользователю выбрать одно из них. Единственная проблема здесь - предоставить информацию пользователю для выбора, это должно быть больше чем MAC-адрес.

Важно: есть [известный баг](https://devzone.nordicsemi.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-00-04-DZ-1046/2604.BLE_5F00_on_5F00_Android_5F00_v1.0.1.pdf){:target="_blank"} для Samsung S6 / Samsung S6 Edge, когда все результаты сканирования имеют один и тот же RSSI (уровень сигнала) при задержке больше нуля.
> Прим. переводчика: в моему случае требовалось обнаружить 2 однотипных устройства, при этом каждое из них имело свое имя, которое отображалось в UI.

# **Кеширование Android Bluetooth стека**
Процесс сканирования дает вам список BLE устройств и при этом данные устройств "кешируются" в Bluetooth стеке. Там хранится основная информация: имя, MAC-адрес, тип адреса (публичный, случайный), тип устройства (Classic, Dual, BLE) и т.д.  Android нужны эти данные, чтобы подключится к устройству быстрее. Он кеширует все устройства, которые видит при сканировании. Для каждого из них записывается небольшой файл с данными. Когда вы пытаетесь подключиться к устройству, стек Android ищет соответствующий файл, чтобы прочитать данные для подключения. Важный момент - одного MAC-адреса недостаточно для успешного подключения к устройству!

# **Очистка кеша**
BT-ке, как и любой другой, не существует вечно и есть 3 ситуации когда он очищается:
1. Выключение и включение обратно системного переключателя Bluetooth,
2. Перезагрузка вашего телефона,
3. Очистка в ручном режиме в настройка телефона.

Это достаточно сложный момент для разработчиков, потому что телефон часто перезагружается, пользователь может включать-выключать самолетный режим. Есть еще различия между производителями телефонов, например на некоторых телефонах Samsung, кеш не очищался при выключении Bluetooth.

Это значит, что нельзя полагаться на данные об устройстве из BT кеша. Есть небольшой трюк, он поможет узнать закешировано ли устройство или нет:

{% highlight java %}
// Get device object for a mac address
BluetoothDevice device = bluetoothAdapter.getRemoteDevice(peripheralAddress)
// Check if the peripheral is cached or not
int deviceType = device.getType();
if(deviceType == BluetoothDevice.DEVICE_TYPE_UNKNOWN) {
    // The peripheral is not cached
} else {
    // The peripheral is cached
}
{% endhighlight %}
Это важно, если нужно подключиться к устройству позже, не сканируя его. Подробнее об этом позже...

# **Непрерывное сканирование?**
Вообще сканирование не следует делать непрерывно, это очень энергоемкая операция. Пользователи любят, когда батарея их смартфона работает долго. Если вам действительно нужно постоянное сканирование, например при поиске BLE-маячков, выберите настройки сканирования с низким потреблением и ограничивайте время сканирования, например когда приложение находится только на передноем плане (foreground), либо сканируйте с перерывами.

В последнее время Google ограничивает (недокументированно) непрерывное сканирование:
* c Android  8.1 [сканирование без фильтров блокируется при выключенном экране](https://stackoverflow.com/questions/48077690/ble-scan-is-not-working-when-screen-is-off-on-android-8-1-0){:target="_blank"}. Если у вас нет никаких `ScanFilters`, Android приостановит сканирование, когда экран выключен и продолжит, когда экран снова будет включен. [Комментарии от Google.](https://android.googlesource.com/platform/packages/apps/Bluetooth/+/319aeae6f4ebd13678b4f77375d1804978c4a1e1){:target="_blank"} Это очевидно очередной способ энергосбережения от Google.
* c Android 7 вы можете сканировать только в течение 30 минут, после чего Android меняет параметры на [`SCAN_MODE_OPPORTUNISTIC`.](https://blog.classycode.com/undocumented-android-7-ble-behavior-changes-d1a9bd87d983){:target="_blank"} Очевидное решение, перезапускать сканирование с периодом менее, чем [30 мин](https://android-review.googlesource.com/c/platform/packages/apps/Bluetooth/+/231662/2/src/com/android/bluetooth/gatt/ScanManager.java){:target="_blank"}. Посмотрите [commit](https://android-review.googlesource.com/c/platform/packages/apps/Bluetooth/+/215844){:target="_blank"} в исходном коде.
* с Android 7 запуск и останов сканирования более 5 раз за 30 секунд [временно отключает сканирование](https://blog.classycode.com/undocumented-android-7-ble-behavior-changes-d1a9bd87d983){:target="_blank"}.

# **Непрерывное сканирование в фоне**
Google значительно усложнил сканирование на переднем плане. Для фонового режима вы столкнетесь с еще большими трудностями!
Новые версии Android имеют лимиты на работу служб в фоновом режиме, обычно после 10 минут работы, фоновый сервис прекращает свою работу принудительно.
Посмотрите возможные решения этой проблемы:
* Обсуждение на [StackOverflow](https://stackoverflow.com/questions/51371372/beacon-scanning-in-background-android-o){:target="_blank"}
* Статья [David Young](http://www.davidgyoungtech.com/2017/08/07/beacon-detection-with-android-8){:target="_blank"}

> Прим. переводчика: я использовал [`Foreground Service`](https://developer.android.com/guide/components/services){:target="_blank"}, потому что после сканирования, будет длительный обмен данными с устройствами в процессе использованя аппы. Один из плюсов этого решения - работает в [`Doze Mode`](https://developer.android.com/training/monitoring-device-state/doze-standby){:target="_blank"}.

# **Проверка разрешений (permissions)**
Есть еще несколько важных моментов, прежде чем мы закончим статью. Для начала сканирования нужны системные разрешения (permissions):

{% highlight xml %}
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
{% endhighlight %}

Убедитесь что все разрешения одобрены, или запросите их у пользователя. Разрешение `ACCESS_COARSE_LOCATION` Google считает "опасным" и для него требуется обязательное согласие пользователя.

{% highlight java %}
private boolean hasPermissions() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        if (getApplicationContext().checkSelfPermission(Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(new String[] { Manifest.permission.ACCESS_COARSE_LOCATION }, ACCESS_COARSE_LOCATION_REQUEST);
            return false;
        }
    }
    return true;
}
{% endhighlight %}

После получения всех нужный разрешений, нужно проверить включен Bluetooth, если нет - используйте `Intent` для запуска запроса на включение:

{% highlight java %}
BluetoothAdapter bluetoothAdapter = BluetoothAdapter.getDefaultAdapter();
if (!bluetoothAdapter.isEnabled()) {
    Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
    startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
}
{% endhighlight %}

# **Следующая статья: отключение и включение**
На данный момент мы рассмотрели сканирование, в следующей статье мы погрузимся глубже в процесс подключения и отключения устройств.

Спасибо!



