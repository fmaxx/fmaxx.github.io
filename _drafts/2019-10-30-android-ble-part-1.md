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


## Почему BLE на Android работает так?

Оглядываясь назад, основные причины это:

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
1. `SCAN_MODE_LOW_POWER`.
2. `SCAN_MODE_BALANCED`.
3. `SCAN_MODE_LOW_LATENCY`.
4. `SCAN_MODE_OPPORTUNISTIC`.

Спасибо!



