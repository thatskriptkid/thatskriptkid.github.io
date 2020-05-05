---
layout: post
title: Обзор декомпилятора GJoy Dex Analysizer
tags: [windows, tool]
category: [ru]
---

![](/assets/images/ru/GDA/main.png)

[Источник картинки](https://btcmanager.com/android-malware-attacking-cryptocurrency-bank-apps/?q=/android-malware-attacking-cryptocurrency-bank-apps/&)

## Intro

Недавно в [твитере увидел новость](https://twitter.com/RamonaHoogeveen/status/1223933604801187841?s=09) о выходе новой версии андроид декомпилятора [GJoy Dex Analysizer](https://github.com/charles2gan/GDA-android-reversing-Tool) (далее просто GDA). Меня заинтересовали его возможности и я никогда о нем ранее не слышал.

Особенности и плюсы вот такие

1. Написан на С++, не на java как другие.
2. Не требует установки, просто один EXE-шник
3. Поддерживает форматы APK, DEX, ODEX, oat
4. Весит всего 2 мегабайта
5. Определяет кросс ссылки для строк, классов, методов, полей
6. Поиск по строкам, классам, методам, полям
7. Деобфускатор
8. Встроенный автоматический анализ малвари
9. Бесплатный

<details>
<summary>И еще много чего еще! Более полный список фич</summary>
<br>
<pre>

Interactive operation:
    1.cross-references for strings, classes, methods and fields;
    2.searching for strings, classes methods and fields;
    3.comments for java code;
    4.rename for methods,fields and classes;
    5.save the analysis results in gda db file.
    ...

Utilities for Assisted Analysis:
    1.extracting DEX from ODEX;
    2.extracting DEX from OAT;
    3.XML Decoder;
    4.algorithm tool;
    5.device memory dump;
    ...
    
New features:
    1.Brand new dalvik decompiler in c++ with friendly GUI;
    2.Support python script
    3.packers Recognition;
    4.Multi-DEX supporting;
    5.making and loading signature of the method 
    6.Malicious Behavior Scanning by API chains;
    7.taint analysis to preview the behavior of variables;
    8.taint analysis to trace the path of variables;
    9.de-obfuscate;
    10.API view with x-ref;
    11.Association of permissions with modules;
    ...
</pre>
</details>

Минусы

1. Только под винду.
2. Только статический анализ

Решил сделать на него обзор, но просто обзор не очень интересно, поэтому я покажу его возможности на примере статического анализа андроид трояна.

_В вики GDA есть свое описание функционала, но я его намеренно не читал, чтобы разобраться самостоятельно. Прочитал после написания статьи и увидел, что описание строится тоже на основе анализа малвари. Вот такое вот совпадение_

Для поиска малвари, как обычно зашел на Virusshare.com, ввел в поиске "apk" и скачал первый попавшийся сэмпл. Сэмпл впервые засветился на вирустотале 28 июля 2019 года. 

![](/assets/images/ru/GDA/1.png)

Что интересно, на [вирустотале](https://www.virustotal.com/gui/file/9d90bf131152d3def8dff175fc1961a5d8feb0c0b0f8c16fad649b681e392e3f/detection) было всего два детекта.

![](/assets/images/ru/GDA/2.png)

Прежде чем провести статически анализ, я решил установить малварь и посмотреть, что это вообще такое. Устанавливается она под названием "Lovely Doll Live Wallpaper". 

![](/assets/images/ru/GDA/3.png)

Запускаем приложение и видим очень милое лицо девушки (какой милый троян!), с единственной кнопкой "Set Wallpaper". 

![](/assets/images/ru/GDA/4.png)

Жмем на эту кнопку и нам еще раз предлагают установить обои

![](/assets/images/ru/GDA/5.png)

После этого обои все таки устанавливаются

![](/assets/images/ru/GDA/6.png)

Теперь перейдем к нашей тулзе GDA. Открываем apk и видим основной интерфейс, в котором нам демонстируют DEX Header и разрешения

![](/assets/images/ru/GDA/7.png)

Можно навести на каждый элемент заголовка DEX и получить описание, что это такое. Про формат DEX и что значат эти байты можно прочитать [тут](https://source.android.com/devices/tech/dalvik/dex-format)

![](/assets/images/ru/GDA/8.png)

Слева в интерфейсе находятся основные пункты

![](/assets/images/ru/GDA/9.png)

Давайте покликаем на каждый. Первым идет пункт `AllStrings`. Как видно из названия, это список всех строк в apk. 

![](/assets/images/ru/GDA/10.png)

Можно произвести поиск по строкам, но не поддерживаются регулярные выражения, что очень плохо. Так как мы анализируем малварь, давайте узнаем, есть ли какая-нибудь работа с БД. Малварь создает таблицу `messages`, уже интересно, почему она так называется, ведь это всего лишь установщик обоев. Причем даже не обоев, а только одного изображения, без права выбора (хоть и милого).

![](/assets/images/ru/GDA/11.png)

Можно кликнуть по ссылке на любую строку и откроется ее расположение в самом файле, как hex view в IDA

![](/assets/images/ru/GDA/12.png)

Пункт `AppString` не очень понятен, в моем случае вывод идентичен `AllString`. Пункт `AllApi` перечисляет все методы, которые юзаются в приложении. Интересно зачем установщику обоев данные нашей сим карты и оператора.

![](/assets/images/ru/GDA/13.png)

Пункт `AndroidManifest` собственно его нам и показывает. Обратите внимание, внизу есть кнопочки, которые покажут отдельно ресиверы, сервисы и т.д.. Это ОЧЕНЬ удобно

![](/assets/images/ru/GDA/14.png)

Кликаем по permission и видим разрешения на чтение и отправку SMS. Это очень странно.

![](/assets/images/ru/GDA/15.png)

Из списка сервисов, видим один мутный, под названием `PaymentStatusReceiver`. Что за оплата непонятно, ведь в приложении нет больше НИЧЕГО, кроме одной кнопки. Также есть сервис `MpSMSService`, которые будет принимать наши смски.

![](/assets/images/ru/GDA/16.png)

`HexView` - просто показывает сырую информацию в хексе

![](/assets/images/ru/GDA/17.png)

Тут мы добрались до двух самых крутых фич GDA! Это `MalScan` и `AccessPermission`. Именно их мне не хватало так долго!

`MalScan` - сканирует приложение на предмет мутных действий.

![](/assets/images/ru/GDA/18.png)

Круто это еще и тем, что нам говорят, в каком конкретно классе это происходит. Например, кликнув по классу в категории `Sending Message`, нам откроется его код. Здесь происходит отправка SMS (`sendTextMessage()`).

![](/assets/images/ru/GDA/19.png)

`AccessPermission` - показывает список разрешений и где в коде они используются. Очень полезная фича. Например, видите что приложение просит доступ к микрофону и тут же просматриваете что она с этим делает

![](/assets/images/ru/GDA/20.png)

В окне с кодом можно переключаться между java/smali

![](/assets/images/ru/GDA/21.png)

Есть встроенный шифровщик/расшифровщик. Очень удобно, если вы нашли в apk ключ и надо быстро проверить его правильность.

![](/assets/images/ru/GDA/22.png)

Можно выбрать любой метод и посмотреть его `Similarity` - список похожих методов во всем коде, в процентном соотношении

![](/assets/images/ru/GDA/23.png)

Можно подключиться к девайсу в рантайме и посмотреть процессы. На этом все функции связанные с рантаймом заканчиваются. На клике по `Open Process` GDA у меня всегда крашился =*( 

![](/assets/images/ru/GDA/24.png)

Работу деобфускатора проверить не удалось. Надеюсь все дело в конкретном сэмлпе.

## Вывод

GDA прекрасно подходит для первоначального статического анализа. Есть даже простенький Malware Scanner, ссылки на использование разрешений и кликабельный интерфейс. 